# \#99953 [SC-Medium] Mark-fresh reconcile truncates unassignedAssets, permanently forfeiting part of the reserve to Junior LPs

**Submitted on Nov 15th 2026 at 21:12:29 UTC by @flokir for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99953
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity:** Medium

**Location:** HousePoolReconcilePlanLib.planReconcile (src/libraries/HousePoolReconcilePlanLib.sol:73-74), committed to storage by HousePool._reconcile (src/HousePool.sol:1585)

#### Description

`HousePool.unassignedAssets` is the pool's explicit earmark for recognized settlement cash that has no safe LP claimant path; the only supported consumer is `HousePool.assignUnassignedAssets`, which reconciles, then assigns the whole balance to a chosen tranche and a chosen share receiver. It is also treated as a reserve: `_buildWithdrawalSnapshot` subtracts it from LP free assets (src/HousePool.sol:1727-1734 via `HousePoolWithdrawalPreviewLib.reserveAssets`), and `_canAcceptTrancheDeposits` reads it to block new tranche deposits while it is nonzero (src/HousePool.sol:499). Once a recapitalization inflow has been routed into that bucket (zero senior impairment gap in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`), the balance is meant to stay put until the owner assigns it.

`HousePoolReconcilePlanLib.planReconcile` breaks that. On every mark-fresh reconcile it clamps the bucket to the current mark-derived distributable value (`plan.state.unassignedAssets = state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets`, src/libraries/HousePoolReconcilePlanLib.sol:73-74), and `HousePool._reconcile` writes the result straight into storage (src/HousePool.sol:1585). `snapshot.distributable` is not a stable measure of pool cash: `HousePoolAccountingLib.buildReconcileSnapshot` folds in the signed terminal price PnL of open positions, so a fresh mark that puts a trader into profit mechanically reduces it while every unit of pool cash is still physically present.

That makes the clamp destructive . If the pool holds 900 units of earmarked reserve and a fresh waterfall mark prices an open position into profit such that distributable reads 850, the next mark-fresh reconcile silently rewrites `unassignedAssets` from 900 to 850 without any `assignUnassignedAssets` call and without any owner decision — 50 units of still-present, explicitly reserved cash simply stop existing as an earmark. The impairment of principal from that same mark is reversible (`HousePoolWaterfallAccountingLib.distributeRevenue` restores `seniorPrincipal` to `seniorHighWaterMark` and credits the remainder to `juniorPrincipal`, and `absorbLoss` deliberately leaves the high-water mark intact which is the expected chain of actions),  but the truncated reserve is not reversibel: when the mark recovers, `distributableToClaims = snapshot.distributable - plan.state.unassignedAssets` is computed against the reduced reserve (src/libraries/HousePoolReconcilePlanLib.sol:89-91), so the recovered 50 units are classified as revenue and credited to `juniorPrincipal` by `distributeRevenue`.

The result is a one-way ratchet: a repeated, mark-driven excursion moves value out of an owner-directed bucket and into incumbent Junior principal, releases that portion of the withdrawal reserve so the same LPs can redeem it as cash, and quietly reopens the LP-entry gate that a nonzero reserve was documented to hold shut — all without the explicit assignment the design requires.

#### Impact

Any earmarked `unassignedAssets` balance is permanently reduced by the amount of any temporary, mark-driven deficit that a fresh reconcile observes — no owner or privileged action required, and no `assignUnassignedAssets` call involved. In the reproduced path a 900-unit recanalization reserve drops to 850 while all cash remains in the pool; when the mark reverts, the recovered 50 units are booked as revenue and credited to `juniorPrincipal` instead of being restored. Consequences for the client: (1) value the owner could direct with `assignUnassignedAssets` (as an example, minting tranche shares to a chosen receiver) is irrevocably re-attributed to incumbent Junior LPs and cannot be recovered, because `assignUnassignedAssets` reconciles first and then assigns only the already-truncated balance; (2) because `unassignedAssets` is subtracted from LP free assets in `_buildWithdrawalSnapshot`, the truncated 50 units become withdrawable pool cash for the very LPs credited with the recovered revenue; (3) `_canAcceptTrancheDeposits` reads the same variable, so the LP-entry block that a nonzero reserve is supposed to enforce silently lifts, allowing new depositors to enter a pool whose ownerless-value condition has not actually been resolved. Magnitude scales with the mark excursion and is capped by the reserve itself; no settlement-token balance leaves the pool and no one can steal the reserve outright however the value can be sqewed in favour of the incumbent Junior LP which is contrary to the expected contract behavior.



#### Recommendation

Stop keying the earmark to the mark. In `HousePoolReconcilePlanLib.planReconcile`, do not overwrite `state.unassignedAssets` from `snapshot.distributable`; instead treat the stored balance as a fixed reservation and compute `distributableToClaims = snapshot.distributable - pendingBucketAssets - state.unassignedAssets` with the reserve clamped only by mark-independent available cash, so a mark-driven terminal deficit is charged to Senior/Junior principal through `absorbLoss` (where it is reversible) rather than to the earmark. If a solvency cap on the reserve is genuinely required, record the shortfall in a separate field (for example `unassignedAssetsImpairment`) and make the next mark-fresh revenue pass restore that shortfall to `unassignedAssets` before any amount reaches `HousePoolWaterfallAccountingLib.distributeRevenue`, so the sweep is not one-way. In both variants, only `HousePool.assignUnassignedAssets` should be able to clear or reduce the bucket, preserving the documented rule that ownerless value requires explicit assignment and cannot silently become windfall LP ownership. Add an invariant test asserting that a two-step mark excursion (adverse then restored) leaves `unassignedAssets` unchanged.


## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Minimal six-decimal settlement token used as pool collateral and clearinghouse custody asset.
contract Test86SettlementToken {
    string public name = "Settlement USD";
    string public symbol = "sUSD";
    uint8 public decimals = 6;
    uint256 public totalSupply;

    mapping(address account => uint256) public balanceOf;
    mapping(address owner => mapping(address spender => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(address indexed owner, address indexed spender, uint256 amount);

    function approve(
        address spender,
        uint256 amount
    ) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transfer(
        address to,
        uint256 amount
    ) external returns (bool) {
        _move(msg.sender, to, amount);
        return true;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        require(allowed >= amount, "Test86SettlementToken: allowance");
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        _move(from, to, amount);
        return true;
    }

    function mint(
        address to,
        uint256 amount
    ) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    function _move(
        address from,
        address to,
        uint256 amount
    ) internal {
        require(balanceOf[from] >= amount, "Test86SettlementToken: balance");
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
    }
}

/// @notice The engine-reported order router for this scenario. It owns no live orders, so the router-settlement
///         floor lookup used by the clearinghouse resolves to zero (as expected in the production durinf 'On every mark-fresh reconcile' *) while the harness drives the engine's
///         router-gated entrypoints from this address.  
contract Test86RouterStub {
    function accountHeadOrderId(
        address
    ) external pure returns (uint64) {
        return 0;
    }

    function lifecycleBook() external pure returns (address) {
        return address(0);
    }
}

/// @title Test_86   check on test 84(/home/users/bb/ctf/plether/test84/maybe/unassignedAssets.txt)*
/// @notice Deterministic replay of AuditSpecification 86: `HousePoolReconcilePlanLib.planReconcile` silently clamps
///         `unassignedAssets` to the pool's current distributable value on every mark-fresh pass
///         (`plan.state.unassignedAssets = state.unassignedAssets > snapshot.distributable
///         ? snapshot.distributable : state.unassignedAssets`), and `HousePool._reconcile` persists that value with
///         `unassignedAssets = plan.state.unassignedAssets`. `HousePool.assignUnassignedAssets` is the only documented
///         consumer of that bucket ("reserved until explicit assignment"), so a purely temporary, mark-driven deficit
///         permanently truncates it. The corresponding principal loss is reversible — the next revenue reconcile
///         refills senior principal to its high-water mark and credits the remainder to junior principal — but the
///         truncated reserve is never restored: the recovered value is measured against the smaller reserve and is
///         booked as revenue for the incumbent Junior holders instead of returning to the owner-controlled bucket.
contract Test_86 is Test {
    /// @dev Eight-decimal engine price ceiling.
    uint256 internal constant CAP_PRICE = 200_000_000; // 2.00
    /// @dev Eight-decimal entry mark of the LONG position (also the pool's cached mark at entry).
    uint256 internal constant ENTRY_PRICE = 100_000_000; // 1.00
    /// @dev Eight-decimal waterfall mark: a 1% excursion that prices the open position into trader profit.
    uint256 internal constant WATERFALL_PRICE = 99_000_000; // 0.99
    /// @dev Seconds between mark updates (inside the pool's 60-second mark-staleness limit).
    uint64 internal constant MARK_STEP_SECONDS = 10;

    /// @dev Canonical position quantum: one lot is 100 synthetic tokens, so `lots * 8-decimal price` is 6-decimal
    ///      settlement atoms. Five lots at 1.00 is a 500e6 notional, i.e. a 500e6 LONG maximum-profit envelope.
    uint256 internal constant SIZE_QUANTUM = 1e20;
    uint256 internal constant POSITION_LOTS = 5;
    uint256 internal constant POSITION_SIZE = POSITION_LOTS * SIZE_QUANTUM;
    uint256 internal constant POSITION_MARGIN_ASSETS = 100e6;
    uint256 internal constant TRADER_DEPOSIT_ASSETS = 200e6;

    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 100;
    /// @dev Liquidation-charge parameters. 
    uint256 internal constant MIN_BOUNTY_ASSETS = 100_000;
    uint256 internal constant BOUNTY_BPS = 10;
    uint256 internal constant KEEPER_SHARE_BPS = 5_000;
    uint256 internal constant PROTOCOL_SHARE_BPS = 2_000;

    /// @dev Tranche seeds. `distributable` is the pool's physicalassets plus that signed delta.
    uint256 internal constant SENIOR_SEED_ASSETS = 1e6;
    uint256 internal constant JUNIOR_SEED_ASSETS = 1e6;

    /// @dev Claimant recapitalization inflow routed into the owner-designated unassigned bucket the reserve that the mark-fresh clamp truncates.
    uint256 internal constant RECAPITALIZATION_ASSETS = 900e6;

    uint64 internal constant T_DEPLOY = 1_718_020_800;

    address internal constant TRADER = address(0xA11CE);
    address internal constant PROTOCOL_TREASURY = address(0x7B22);
    address internal constant SEED_RECEIVER = address(0x5EED);

    Test86SettlementToken internal token;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    CfdEnginePlanner internal planner;
    CfdEngineSettlementSidecar internal settlementSidecar;
    CfdEngineAdmin internal engineAdmin;
    TerminalNavBookV2 internal navBook;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    Test86RouterStub internal router;

    /// @dev Loss the waterfall mark prices into the open position: `lots * (ENTRY_PRICE - WATERFALL_PRICE)`.
    uint256 internal waterfallMarkDeltaAssets;

    function setUp() public {
        vm.warp(T_DEPLOY);
        vm.roll(1_000);

        // ---- spec.setup.contracts: full stack wiring ----
        token = new Test86SettlementToken();
        clearinghouse = new MarginClearinghouse(address(token));
        engine =
            new CfdEngine(address(token), address(clearinghouse), CAP_PRICE, _riskParams(), FROZEN_CLOSE_SPREAD_BPS);
        clearinghouse.setEngine(address(engine));

        planner = new CfdEnginePlanner();
        settlementSidecar = new CfdEngineSettlementSidecar(address(engine));
        engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlementSidecar), address(engineAdmin));
        navBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(navBook));

        // ---- HousePool bound to the engine (this harness is governance) ----
        pool = new HousePool(address(token), address(engine), address(new HousePoolRedemptionMathSidecar()));
        engine.setPool(address(pool));
        seniorVault =
            new TrancheVault(IERC20(address(token)), address(pool), true, "Plether Senior", "plSEN", 0, address(0));
        juniorVault =
            new TrancheVault(IERC20(address(token)), address(pool), false, "Plether Junior", "plJUN", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));

        // ---- governed config: live mark limit 60s, 8% senior coupon, finite senior capacity ----
        pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: 800,
                markStalenessLimit: 60,
                seniorFrozenLpFeeBps: 25,
                juniorFrozenLpFeeBps: 75,
                maxSeniorExposureAssets: 1_000_000_000e6,
                maxSeniorShareBps: 9_000
            })
        );
        vm.warp(block.timestamp + pool.TIMELOCK_DELAY());
        pool.finalizePoolConfig();

        // ---- sequence steps 1-3: real-cash tranche seeds and trading activation ----

        token.mint(address(this), SENIOR_SEED_ASSETS + JUNIOR_SEED_ASSETS);
        token.approve(address(pool), type(uint256).max);
        pool.initializeSeedPosition(false, JUNIOR_SEED_ASSETS, SEED_RECEIVER);
        pool.initializeSeedPosition(true, SENIOR_SEED_ASSETS, SEED_RECEIVER);
        pool.activateTrading();

        // ---- the router the engine reports for its order entrypoints ----
        router = new Test86RouterStub();
        engine.setOrderRouter(address(router));
        engine.setProtocolTreasury(PROTOCOL_TREASURY);

        // ---- the trader's canonical clearinghouse balance that funds the open position ----
        _deposit(TRADER, TRADER_DEPOSIT_ASSETS);

        // ---- owner funding for the recapitalization inflow (CfdEngine pulls it into the pool) ----
        token.mint(address(this), RECAPITALIZATION_ASSETS);
        token.approve(address(engine), type(uint256).max);

        waterfallMarkDeltaAssets = POSITION_LOTS * (ENTRY_PRICE - WATERFALL_PRICE);

        // ---- spec.setup.states_variables ----
        assertEq(pool.seniorPrincipal(), SENIOR_SEED_ASSETS, "setup: senior principal is seeded");
        assertEq(pool.seniorHighWaterMark(), SENIOR_SEED_ASSETS, "setup: senior is unimpaired at its high-water mark");
        assertEq(pool.juniorPrincipal(), JUNIOR_SEED_ASSETS, "setup: junior principal is seeded");
        assertEq(pool.unassignedAssets(), 0, "setup: no ownerless reserve exists yet");
        assertFalse(pool.lpEpochSettlementPaused(), "setup: LP epoch settlement is live");
        assertTrue(pool.isTradingActive(), "setup: trading is active");
        assertFalse(engine.degradedMode(), "setup: the engine is not degraded");
        assertFalse(engine.isOracleFrozen(), "setup: the market calendar is not frozen");
    }

    /// @notice Replays the specification sequence and checks whether a temporary mark excursion permanently truncates
    ///         the owner-designated unassigned reserve, with the recovered value re-attributed to junior principal.
    function test_specified_post_state_is_unreachable() public {
        uint64 t0 = uint64(block.timestamp);

        // ================= sequence step 4: CfdEngine.recapitalizePool =================
        // The engine owner transfers recapitalization cash to the pool and the engine calls
        // `recordClaimantInflow(Recapitalization, CashArrived)`, so the value is recognized in `accountedAssets` and
        // queued in `pendingRecapitalizationAssets`.
        assertEq(pool.pendingRecapitalizationAssets(), 0, "step4: no claimant bucket before the inflow");
        engine.recapitalizePool(RECAPITALIZATION_ASSETS);

        assertEq(
            pool.pendingRecapitalizationAssets(), RECAPITALIZATION_ASSETS, "step4: the inflow is queued as a claimant bucket"
        );
        assertEq(
            pool.totalAssets(),
            SENIOR_SEED_ASSETS + JUNIOR_SEED_ASSETS + RECAPITALIZATION_ASSETS,
            "step4: arrived cash is recognized in the canonical asset ledger"
        );

        // ---- spec.pre_attack: one open LONG position at the cached engine mark, established here because the pool
        //      must already hold the arrived cash for its depth to cover the position's maximum-profit envelope. ----
        _open(TRADER, POSITION_LOTS, POSITION_MARGIN_ASSETS, ENTRY_PRICE, 1, t0);

        (uint256 sizeAfterOpen,,,, CfdTypes.Side sideAfterOpen,,) = engine.positions(TRADER);
        assertEq(sizeAfterOpen, POSITION_SIZE, "pre: the trader holds one open position");
        assertEq(uint256(sideAfterOpen), uint256(CfdTypes.Side.LONG), "pre: the position is on the LONG side");
        assertEq(engine.lastMarkPrice(), ENTRY_PRICE, "pre: the cached mark is the entry mark");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "pre: no trader claims are outstanding");
        assertGt(pool.seniorPrincipal(), 0, "pre: revenue continuation has a claimed tranche to route into");

        // ================= sequence step 5: HousePool.reconcile =================
        // Mark-fresh reconcile (invoked as the configured senior vault). `applyClaimantRecapitalizationIntent` finds a
        // zero senior impairment gap, so the whole inflow is routed into the owner-designated unassigned bucket.
        uint256 distributableAtEntryAssets = _distributableAssets();
        assertGt(
            distributableAtEntryAssets,
            RECAPITALIZATION_ASSETS,
            "step5: distributable covers the whole recapitalization bucket before it is routed"
        );

        vm.prank(address(seniorVault));
        pool.reconcile();

        uint256 reserveRoutedAssets = pool.unassignedAssets();
        assertEq(
            reserveRoutedAssets,
            RECAPITALIZATION_ASSETS,
            "step5: the zero senior gap routed the entire recapitalization into unassignedAssets"
        );
        assertEq(pool.pendingRecapitalizationAssets(), 0, "step5: the claimant bucket is settled");
        assertEq(pool.lastReconcileTime(), block.timestamp, "step5: the reconcile was mark-fresh");
        uint256 juniorBeforeMarkLossAssets = pool.juniorPrincipal();

        // ================= sequence step 6: CfdEngine.updateMarkPrice (waterfall) =================
        // The router caches a fresh mark 1% below entry. The open LONG is priced into trader profit, which
        // `buildReconcileSnapshot` folds into `distributable` as a signed `terminalLpPriceDeltaAssets`.
        uint64 t1 = t0 + MARK_STEP_SECONDS;
        vm.warp(t1);
        vm.prank(address(router));
        engine.updateMarkPrice(WATERFALL_PRICE, t1);

        assertEq(engine.lastMarkPrice(), WATERFALL_PRICE, "step6: the cached mark is the waterfall mark");
        assertEq(engine.lastMarkTime(), t1, "step6: the cached mark time is the waterfall timestamp");

        int256 terminalDeltaAssets = _terminalLpPriceDeltaAssets();
        assertEq(
            terminalDeltaAssets,
            -int256(waterfallMarkDeltaAssets),
            "step6: the signed terminal delta equals the 1% notional excursion"
        );

        uint256 distributableAtWaterfallAssets = _distributableAssets();
        assertLt(
            distributableAtWaterfallAssets,
            reserveRoutedAssets,
            "step6: the mark-driven deficit pushes distributable below the routed reserve"
        );

        // ================= sequence step 7: HousePool.reconcile =================
        // `planReconcile` overwrites the reserve with the smaller distributable and `_reconcile` persists it. No
        // `assignUnassignedAssets` call consumes the bucket.
        vm.prank(address(juniorVault));
        pool.reconcile();

        uint256 reserveAfterWaterfallAssets = pool.unassignedAssets();
        assertEq(
            reserveAfterWaterfallAssets,
            distributableAtWaterfallAssets,
            "step7: unassignedAssets was overwritten with the mark-driven distributable"
        );
        assertLt(
            reserveAfterWaterfallAssets,
            reserveRoutedAssets,
            "step7: the owner-designated reserve was truncated without an assignUnassignedAssets call"
        );
        assertEq(pool.lastReconcileTime(), block.timestamp, "step7: the reconcile was mark-fresh");
        uint256 juniorAfterMarkLossAssets = pool.juniorPrincipal();
        assertLt(
            juniorAfterMarkLossAssets,
            juniorBeforeMarkLossAssets,
            "step7: the mark-driven deficit was absorbed junior-first"
        );

        // ================= sequence step 8: CfdEngine.updateMarkPrice (revert) =================
        // The temporary excursion reverses: pool value is fully restored at the original mark.
        uint64 t2 = t1 + MARK_STEP_SECONDS;
        vm.warp(t2);
        vm.prank(address(router));
        engine.updateMarkPrice(ENTRY_PRICE, t2);

        assertEq(engine.lastMarkPrice(), ENTRY_PRICE, "step8: the cached mark is back at entry");
        uint256 distributableAfterRevertAssets = _distributableAssets();
        assertGt(
            distributableAfterRevertAssets,
            reserveAfterWaterfallAssets,
            "step8: the recovered pool value exceeds the truncated reserve"
        );

        // ================= sequence step 9: HousePool.reconcile =================
        // `distributableToClaims` is measured against the truncated reserve, so the recovered value is booked as
        // revenue and distributed by `HousePoolWaterfallAccountingLib.distributeRevenue`.
        vm.prank(address(seniorVault));
        pool.reconcile();

        uint256 reserveAfterRevertAssets = pool.unassignedAssets();
        uint256 juniorAfterRevertAssets = pool.juniorPrincipal();
        assertEq(
            reserveAfterRevertAssets,
            reserveAfterWaterfallAssets,
            "step9: the truncated reserve is never restored once the mark reverts"
        );
        assertGt(
            juniorAfterRevertAssets,
            juniorAfterMarkLossAssets,
            "step9: the recovered value was credited to junior principal as revenue"
        );

        // ============================ post_attack evaluation ============================
        // PROMISED: `unassignedAssets` holds canonical value the owner directs with `assignUnassignedAssets`, and only
        // that call may consume or clear it; mark-dependent principal losses are reversible, so the reserve must
        // survive a temporary excursion. REACHED: a single mark-fresh pass keyed the reserve to the mark-driven
        // distributable, permanently destroying part of it and handing the recovery to incumbent Junior holders.
        bool reserveTruncatedWithoutAssignment = reserveAfterWaterfallAssets < reserveRoutedAssets;
        bool reserveNeverRestored = reserveAfterRevertAssets == reserveAfterWaterfallAssets;
        bool recoveryExceedsTruncatedReserve = distributableAfterRevertAssets > reserveAfterRevertAssets;
        bool recoveredValueCreditedToJunior = juniorAfterRevertAssets > juniorAfterMarkLossAssets;

        bool postAttackReached = reserveTruncatedWithoutAssignment && reserveNeverRestored
            && recoveryExceedsTruncatedReserve && recoveredValueCreditedToJunior;

        require(
            !postAttackReached,
            "unwanted_postattak_reached: the mark-fresh reconcile clamped the explicit-assignment reserve from the routed recapitalization down to the mark-driven distributable with no assignUnassignedAssets call, never restored it when the mark reverted, and credited the recovered value to juniorPrincipal as revenue"
        );
    }

    // ==========================================
    // REPLAY HELPERS
    // ==========================================

    /// @dev Mirrors `HousePoolAccountingLib.buildReconcileSnapshot` on the live engine snapshot the pool consumes.
    function _distributableAssets() internal view returns (uint256 distributableAssets) {
        ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = navBook.terminalNavSnapshot();
        uint256 terminalAssets = pool.totalAssets();
        uint256 terminalLiabilities = terminalSnapshot.totalTraderClaimsAssets;
        int256 terminalDelta = terminalSnapshot.terminalLpPriceDeltaAssets;
        if (terminalDelta >= 0) {
            terminalAssets += uint256(terminalDelta);
        } else {
            terminalLiabilities += uint256(-terminalDelta);
        }
        distributableAssets = terminalAssets > terminalLiabilities ? terminalAssets - terminalLiabilities : 0;
    }

    function _terminalLpPriceDeltaAssets() internal view returns (int256 terminalDelta) {
        ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = navBook.terminalNavSnapshot();
        terminalDelta = terminalSnapshot.terminalLpPriceDeltaAssets;
    }

    /// @dev Drives the production open path: the bound router submits the typed order at `price`, the engine plans it
    ///      and the settlement sidecar writes the position and the terminal NAV curve.
    function _open(
        address account,
        uint256 lots,
        uint256 marginAssets,
        uint256 price,
        uint64 orderId,
        uint64 publishTime
    ) internal {
        CfdTypes.Order memory order = CfdTypes.Order({
            account: account,
            sizeDelta: lots * SIZE_QUANTUM,
            marginDelta: marginAssets,
            targetPrice: price,
            commitTime: publishTime,
            commitBlock: uint64(block.number),
            orderId: orderId,
            side: CfdTypes.Side.LONG,
            isClose: false
        });

        uint256 depthAssets = pool.totalAssets();
        vm.prank(address(router));
        engine.processOrderTyped(order, price, depthAssets, publishTime);
    }

    function _deposit(
        address account,
        uint256 amount
    ) internal {
        token.mint(account, amount);
        vm.startPrank(account);
        token.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.depositMargin(amount);
        vm.stopPrank();
    }


    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 0,
            minBountyAssets: MIN_BOUNTY_ASSETS,
            bountyBps: BOUNTY_BPS,
            keeperShareBps: KEEPER_SHARE_BPS,
            protocolShareBps: PROTOCOL_SHARE_BPS
        });
    }
}

```
