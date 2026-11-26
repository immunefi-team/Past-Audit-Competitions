# \#99961 [SC-Low] Recapitalizing a fully wiped pool overwrites the senior high-water mark and strands the recovery inflow as unassigned assets

**Submitted on Nov 15th 2026 at 22:04:29 UTC by @flokir for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99961
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity:** Medium

**Location:** HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent (src/libraries/HousePoolPendingPreviewLib.sol:118-121), reached via HousePool.reconcile (src/HousePool.sol:1471) → HousePool._applyPendingClaimantBucketsLive (src/HousePool.sol:1759-1776)

#### Description

When a loss reconcile wipes both principals, `HousePoolWaterfallAccountingLib.absorbLoss` deliberately leaves `seniorHighWaterMark` at the pre-loss senior entitlement so future value restores the impaired tranche, and `distributeRevenue` uses `seniorHighWaterMark - seniorPrincipal` as the senior restoration gap. `HousePool.reconcile` then applies any queued claimant recapitalization through `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`. That function has a dedicated "empty bootstrap" branch for exactly this post-wipe state (senior shares outstanding, both principals zero), but instead of preserving the stored mark it writes `seniorHighWaterMark = max(recapitalizationTargetAssets, remaining)` — a plain assignment. A small recapitalization (e.g. 1,000e6) therefore overwrites a 100,000e6 senior impairment mark; senior principal is set to the recapitalization amount and the gap the waterfall consumes becomes zero. Every later inflow then follows the "gap == 0" path: a subsequent 500,000e6 recapitalization intended to restore senior LPs is credited wholesale to `unassignedAssets`, and ordinary trading revenue is credited to `juniorPrincipal` while the senior tranche stays stuck at the token recapitalization amount. The `Recapitalization` bucket is created only by `CfdEngine.recapitalizePool` (owner-only) after transferring the tokens to the pool, and any tranche vault can then trigger `HousePool.reconcile`; no attacker is needed — the documented recovery action itself silently destroys the senior claimants' recovery right.

#### Impact

After a pool wipe, the senior tranche's protected recovery entitlement is permanently extinguished as soon as any recapitalization is applied: `seniorHighWaterMark` collapses from the pre-loss senior entitlement to the size of the recapitalization, and `seniorPrincipal` is capped there. Recovery capital then has no safe claimant path — in the traced case a 500,000e6 recapitalization meant to make senior LPs whole is parked in `unassignedAssets` while senior principal stays at the 1,000e6 bootstrap amount, and because `unassignedAssets > 0` blocks ordinary tranche deposits (`HousePoolTrancheGateLib.sol:28`) and LP epoch settlement gates, new LP entry stays closed until governance re-assigns the assets. All subsequent trading revenue is credited to `juniorPrincipal` instead of restoring senior, transferring value from senior shareholders (whose vault share price stays depressed at a small fraction of the wiped 100,000e6 entitlement) to junior shareholders. Triggered by the engine owner funding `CfdEngine.recapitalizePool` — the very action the protocol documents as the post-loss recovery path — followed by any tranche-vault `reconcile`; no unprivileged attacker or owner misconduct is required, and the misrouting persists for all LPs afterwards.

#### Recommendation

Never lower the stored senior mark in the bootstrap branch of `applyClaimantRecapitalizationIntent`. Keep the existing impairment claim by taking a maximum against the stored value, e.g. bootstrap `seniorPrincipal += remaining` and set `seniorHighWaterMark = max(state.waterfall.seniorHighWaterMark, recapitalizationTargetAssets, remaining)`, then route any portion of the inflow above the recapitalization target through the ordinary gap logic (restore `seniorPrincipal` up to the preserved mark, remainder to `unassignedAssets`) instead of zeroing `remaining`. Encode the documented invariant `seniorHighWaterMark` may only decrease through `scaleSeniorOnWithdraw` (proportional senior withdrawal) as a test that runs a loss wipe, a recapitalization and a reconcile, and asserts the mark and the senior restoration gap survive; also assert that a second recapitalization restores `seniorPrincipal` up to the preserved mark rather than landing in `unassignedAssets`.


## Proof of Concept

#### Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Minimal 6-decimal settlement token used as pool collateral.
contract MockSettlementToken {
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
        require(allowed >= amount, "MockSettlementToken: allowance");
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
        require(balanceOf[from] >= amount, "MockSettlementToken: balance");
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
    }
}

/// @notice Replays the recapitalization-bootstrap sequence described by the audit specification.
/// @dev The specification asserts that `HousePool.seniorHighWaterMark` — the protected senior impairment
///      watermark documented by `HousePoolWaterfallAccountingLib.absorbLoss` as deliberately preserved across a
///      loss — survives a claimant recapitalization.
contract Test_31 is Test {

    uint256 internal constant DEPLOY_TIME = 1_700_000_000;

    uint256 internal constant CAP_PRICE = 4_000_000_000;

    /// @dev Settlement token seeded into the senior tranche.
    uint256 internal constant SENIOR_SEED_ASSETS = 100_000e6;
    /// @dev Settlement token seeded into the junior tranche.
    uint256 internal constant JUNIOR_SEED_ASSETS = 50_000e6;
    /// @dev Protected senior claim watermark carried into the pre-attack state (the pre-loss senior entitlement).
    uint256 internal constant SENIOR_IMPAIRMENT_WATERMARK = 100_000e6;
    /// @dev First claimant recapitalization: the inflow that re-bases the watermark.
    uint256 internal constant FIRST_RECAPITALIZATION = 1_000e6;
    /// @dev Second claimant recapitalization: value that should have restored the senior tranche instead goes to unclaimed.
    uint256 internal constant SECOND_RECAPITALIZATION = 500_000e6;

    MockSettlementToken internal token;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    TerminalNavBookV2 internal navBook;
    HousePoolRedemptionMathSidecar internal redemptionMath;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;

    /// @dev Share holder of both permanent seed positions.
    address internal constant SEED_HOLDER = address(0xB0B);
    /// @dev Recipient of the settlement cash leaving the pool when the loss is realised.
    address internal constant SETTLEMENT_PAYEE = address(0x7A11);

    function setUp() public {
        vm.warp(DEPLOY_TIME);

        // ---- Plether perps stack: settlement token, clearinghouse, engine, terminal-NAV book ----
        token = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(token));
        engine = new CfdEngine(address(token), address(clearinghouse), CAP_PRICE, _riskParams(), 100);
        navBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(navBook));

        // ---- HousePool bound to the engine (this harness is governance for both) ----
        redemptionMath = new HousePoolRedemptionMathSidecar();
        pool = new HousePool(address(token), address(engine), address(redemptionMath));
        engine.setPool(address(pool));

        // ---- Two distinct tranche vaults against the same pool, zero maintenance fee ----
        seniorVault =
            new TrancheVault(IERC20(address(token)), address(pool), true, "Plether Senior", "plSEN", 0, address(0));
        juniorVault =
            new TrancheVault(IERC20(address(token)), address(pool), false, "Plether Junior", "plJUN", 0, address(0));
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));

        // ---- spec.setup: finite governed senior limits replace the constructor's unbounded sentinels ----
        // The senior coupon rate keeps its deployed value, so finalization needs no mark-dependent reconcile.
        pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: 800,
                markStalenessLimit: 60,
                seniorFrozenLpFeeBps: 25,
                juniorFrozenLpFeeBps: 75,
                maxSeniorExposureAssets: 1_000_000e6,
                maxSeniorShareBps: 8_000
            })
        );
        vm.warp(DEPLOY_TIME + pool.TIMELOCK_DELAY());
        pool.finalizePoolConfig();

        // Anchor both side carry indexes before any pool depth exists.
        engine.checkpointCarryIndexes();

        // ---- spec.setup: seed both tranche vaults with real settlement token ----
        token.mint(
            address(this),
            JUNIOR_SEED_ASSETS + SENIOR_SEED_ASSETS + FIRST_RECAPITALIZATION + SECOND_RECAPITALIZATION
        );
        token.approve(address(pool), type(uint256).max);
        token.approve(address(engine), type(uint256).max);

        // Junior is seeded first: governed senior admission capacity is derived from `juniorPrincipal`.
        pool.initializeSeedPosition(false, JUNIOR_SEED_ASSETS, SEED_HOLDER);
        pool.initializeSeedPosition(true, SENIOR_SEED_ASSETS, SEED_HOLDER);
        require(pool.seniorHighWaterMark() == SENIOR_IMPAIRMENT_WATERMARK, "setup: watermark capitalised at seeding");
        require(pool.seniorPrincipal() == SENIOR_SEED_ASSETS, "setup: senior principal funded");

        // ---- spec.setup: trading active, so the pool is the live counterparty ----
        pool.activateTrading();
        require(pool.isTradingActive(), "setup: trading must be active");

        // ---- spec.pre_attack: a loss reconcile wipes the claimed equity to settle----
        // Settlement cash leaves the pool (trader payables are met), so canonical pool assets fall to zero while
        // both principal claims are still recorded. The next reconcile absorbs the whole loss junior-first and
        // then senior, saturating both principals at zero while `absorbLoss` leaves the senior
        // high-water mark at its pre-loss entitlement.
        uint256 settlementCashToPay = pool.rawAssets();
        vm.prank(address(engine));
        pool.payOut(SETTLEMENT_PAYEE, settlementCashToPay);

        vm.prank(address(seniorVault));
        pool.reconcile();

        // ---- pre_attack invariants: these MUST hold before the sequence runs ----
        require(pool.accountedAssets() == 0, "pre: canonical asset ledger must be empty");
        require(pool.rawAssets() == 0, "pre: no unaccounted settlement cash");
        require(pool.seniorPrincipal() == 0, "pre: senior claim wiped by the loss reconcile");
        require(pool.juniorPrincipal() == 0, "pre: junior claim wiped by the loss reconcile");
        require(
            pool.seniorHighWaterMark() == SENIOR_IMPAIRMENT_WATERMARK,
            "pre: absorbLoss must preserve the senior impairment watermark"
        );
        require(pool.unassignedAssets() == 0, "pre: no ownerless assets");
        require(pool.pendingRecapitalizationAssets() == 0, "pre: no queued recapitalization");
        require(pool.pendingTradingRevenueAssets() == 0, "pre: no queued trading revenue");
        require(seniorVault.totalSupply() > 0, "pre: senior shares stay outstanding");
        require(juniorVault.totalSupply() > 0, "pre: junior shares stay outstanding");
        require(!pool.isOracleFrozen(), "pre: open market window");
        require(!engine.degradedMode(), "pre: engine not degraded");
    }

    function test_specified_post_state_is_unreachable() public {
        uint256 seniorSupplyBefore = seniorVault.totalSupply();
        uint256 juniorSupplyBefore = juniorVault.totalSupply();

        // ---- Step 1: owner funds a 1_000e6 claimant recapitalization into the pool ----
        engine.recapitalizePool(FIRST_RECAPITALIZATION);
        require(
            pool.pendingRecapitalizationAssets() == FIRST_RECAPITALIZATION, "harness: recapitalization intent queued"
        );
        require(pool.accountedAssets() == FIRST_RECAPITALIZATION, "harness: recapitalization cash recognized");

        // ---- Step 2: a configured tranche vault triggers the canonical reconcile ----
        vm.prank(address(seniorVault));
        pool.reconcile();

        // ---- Step 3: owner funds a second, larger 500_000e6 recapitalization meant to restore senior LPs ----
        engine.recapitalizePool(SECOND_RECAPITALIZATION);
        require(
            pool.pendingRecapitalizationAssets() == SECOND_RECAPITALIZATION,
            "harness: second recapitalization intent queued"
        );

        // ---- Evaluation pass: reconcile and inflow is routed through the waterfall ----
        vm.prank(address(seniorVault));
        pool.reconcile();

        require(pool.pendingRecapitalizationAssets() == 0, "harness: every queued recapitalization was applied");
        require(
            seniorVault.totalSupply() == seniorSupplyBefore && juniorVault.totalSupply() == juniorSupplyBefore,
            "harness: recapitalization moves no shares"
        );

        console.log("seniorPrincipal       ", pool.seniorPrincipal());
        console.log("juniorPrincipal       ", pool.juniorPrincipal());
        console.log("seniorHighWaterMark   ", pool.seniorHighWaterMark());
        console.log("unassignedAssets      ", pool.unassignedAssets());

        // `absorbLoss` promises the senior high-water mark is deliberately unchanged so future revenue restores
        // senior impairment, and `HousePoolWaterfallAccountingLib.distributeRevenue` derives the senior
        // restoration gap from `seniorHighWaterMark - seniorPrincipal`. Both must therefore still record the
        // pre-loss senior entitlement after the recapitalization bootstrap, and the senior bucket must be the
        // first claimant path to receive the inflow.
        bool watermarkPreserved = pool.seniorHighWaterMark() == SENIOR_IMPAIRMENT_WATERMARK;
        bool seniorRestored = pool.seniorPrincipal() == SENIOR_IMPAIRMENT_WATERMARK;
        bool juniorUncredited = pool.juniorPrincipal() == 0;

        require(
            watermarkPreserved && seniorRestored && juniorUncredited,
            "UNWANTED_POST_ATTACK_REACHED: the empty-bootstrap recapitalization branch assigned seniorHighWaterMark = max(recapitalizationTargetAssets, remaining) instead of keeping the stored 100_000e6 senior impairment watermark, so the senior restoration gap that distributeRevenue consumes collapsed to zero, seniorPrincipal stayed at the 1_000e6 recapitalization amount, and the 500_000e6 inflow intended for the impaired senior tranche was parked as unassigned assets"
        );
    }

    function _riskParams() internal pure returns (CfdTypes.RiskParams memory params) {
        params.vpiFactor = 0;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 100;
        params.initMarginBps = 150;
        params.fadMarginBps = 300;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1_000_000;
        params.bountyBps = 10;
        params.keeperShareBps = 5_000;
        params.protocolShareBps = 0;
    }
}

```

### How to Run
install the forge lib and then
forge test --match-path test/Test_31.sol -vvv from the audit root

