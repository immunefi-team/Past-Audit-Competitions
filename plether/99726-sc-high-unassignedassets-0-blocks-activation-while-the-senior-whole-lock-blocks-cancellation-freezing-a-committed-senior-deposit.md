# \#99726 [SC-High] unassignedAssets>0 blocks activation while the senior-whole lock blocks cancellation, freezing a committed Senior deposit

**Submitted on Nov 15th 2026 at 00:13:43 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99726
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

```markdown
## Summary
Reconciliation breaks the stated rule that `only value with no safe claimant path belongs in unassignedAssets` ([ACCOUNTING_SPEC.md#L650](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L650)), a bucket the spec calls `exceptional telemetry, not a normal operating bucket` ([ACCOUNTING_SPEC.md#L652](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L652)): when a mark-to-market gain arrives in the same `settleLpEpoch` as a queued recapitalization of an impaired Senior, the waterfall routes the surplus into the Senior gap (step 4) before it applies the queued recapitalization (step 5, [ACCOUNTING_SPEC.md#L338-L340](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L338-L340)), so the recapitalization — which `never accrues to junior principal` ([HousePoolPendingPreviewLib.sol#L107](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L107)) — meets a near-zero gap and its remainder is parked in `unassignedAssets`, even though the ordinary surplus it displaced is LP value that still has a live Junior claimant path. That persistent ownerless balance then trips the documented `unassignedAssets > 0` activation gate ([ACCOUNTING_SPEC.md#L324](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L324)) and traps a committed Senior deposit that can be neither activated nor cancelled until the owner manually reassigns the bucket.

## Root Cause
The broken invariant is [ACCOUNTING_SPEC.md#L650](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L650): value with a safe claimant path must not sit in `unassignedAssets`. Ordinary trading surplus is value with a claimant path — it is `intended to restore senior impairment, then accrue to junior owners` ([HousePoolPendingPreviewLib.sol#L15](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L15)), and Junior owners exist throughout (the Junior seed and any Junior LP hold shares). Reconciliation nonetheless strands an amount of that value: it distributes the surplus through the waterfall (Senior gap first) and only afterwards applies the queued recapitalization, which can restore Senior but is barred from Junior, so once the surplus has already closed the gap the recapitalization has nowhere to go and its remainder becomes ownerless.

Money is fungible, so the stranded remainder is booked against the recapitalization bucket rather than against the surplus; a purely formal reading therefore sees each bucket obeying its own rule. But the amount that leaves the claimable waterfall equals the surplus, and that surplus had a Junior path that the ordering erased — had the recapitalization been applied while the gap was still open, the surplus would have reached Junior (`applyRevenueIntent` routes it to `juniorPrincipal` once Senior is whole, [HousePoolPendingPreviewLib.sol#L180-L182](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L180-L182)). The displaced value is what ends up ownerless, in violation of [#L650](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L650).

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L95-L99
```solidity
if (plan.revenue) {
    plan.juniorPrincipalBeforeRevenue = plan.state.waterfall.juniorPrincipal;
    plan.state.waterfall =
        HousePoolWaterfallAccountingLib.distributeRevenue(plan.state.waterfall, plan.deltaAssets);
// the surplus fills the Senior gap here, before the queued recapitalization is applied
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1598
```solidity
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
// the queued recap is applied only here, after the surplus above already closed the gap -> remainder is ownerless
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L124-L136
```solidity
uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
    ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal : 0;
// ...
if (remaining > 0) {
    state.unassignedAssets += remaining; // gap already closed by the surplus, so the remainder becomes ownerless
}
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolTrancheGateLib.sol#L28-L29
```solidity
if (!ordinaryDepositsAllowed || paused || unassignedAssets > 0 || !markFreshForReconcile) {
    return false; // any nonzero ownerless balance defers activation of every pending deposit
```

Link:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L851-L858
```solidity
if ((IS_SENIOR || currentLpEpoch() >= requestId) && !queueState.rejected
        && (!IS_SENIOR || !POOL.lpEpochSettlementPaused()) && !_projectedTerminalWipe()
        && !POOL.isSeniorImpairedAfterPendingDepositReconcile()) {
    if (IS_SENIOR) {
        revert TrancheVault__SeniorDepositCommitted(); // Senior now whole -> cancellation blocked
```

The deferral gate at [ACCOUNTING_SPEC.md#L324](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L324) assumes the bucket it keys on is transient; here it is not. The spec never sanctions a committed deposit that can be neither activated nor cancelled, and the recapitalization cannot be applied before the gain lands: `HousePool.reconcile()` is `onlyVault` ([HousePool.sol#L1471](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1471)) and no vault calls it, and `recapitalizePool` cannot reconcile in its own call because the Engine `rejects snapshots while a multi-contract account mutation is in progress` ([ACCOUNTING_SPEC.md#L305-L306](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L305-L306)). The persistence is structural: later gains never reduce `unassignedAssets` (the reconcile only caps it down on a loss, `plan.state.unassignedAssets = min(state.unassignedAssets, distributable)`, [HousePoolReconcilePlanLib.sol#L73-L74](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L73-L74)), so every ordinary settlement preserves the ownerless balance and the deadlock.

## Attack Path
Preconditions (release parameters from `DEPLOYMENT.md` applied through the timelocks: 0.1% / 0.2% margins, 0.04% fee, `vpiFactor` 0.01, 40% skew, 0.25% buffer, 40M USDG / 80%; both tranches seeded). No attacker and no malicious role; every actor is honest:
- A Senior LP's deposit is queued and matured before a market loss.
- A market tail impairs Senior with no terminal deficit and no degraded mode (two-sided book, the shorts exit, 120,000 of longs ride a 1.00 to 0.72 move; the skew limit constrains only opens, [CfdTypes.sol#L92-L94](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTypes.sol#L92-L94)).

1. A routine `settleLpEpoch` books the loss: Junior wiped, Senior impaired by 7,035.45 USDG against its mark, `terminalDeficitAssets = 0`, degraded mode off. The queued deposit is deferred (Senior impaired) but is still cancellable at this point.
2. The owner recapitalizes exactly the 7,035.45 USDG gap with `CfdEngine.recapitalizePool`, the documented recovery action, to restore Senior to its mark. The cash is only queued.
3. Before the next epoch settles, the basket ticks 0.72 to 0.7201 (~0.014%), an ordinary move, and any keeper settles the epoch. The surplus restores Senior to its mark first; the queued recap then meets a zero gap and its remainder becomes `unassignedAssets = 12.00 USDG`.
4. Senior is now whole, so `cancelPendingDeposit` reverts `TrancheVault__SeniorDepositCommitted`, while `unassignedAssets > 0` keeps `trancheDepositsAllowed` false, so the deposit cannot activate. The owner cannot route the bucket back: `assignUnassignedAssets(true, …)` reverts `HousePool__ExceedsSeniorDepositCapacity` (Senior at its mark with Junior wiped gives a senior capacity of zero, [HousePoolSeniorCapacityLib.sol#L102-L112](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L102-L112)), and `assignUnassignedAssets(false, …)` mints new Junior shares to one chosen receiver ([HousePool.sol#L660-L702](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L702)), which never returns the deposit to the Senior LP.

This does not depend on idle keepers. Epochs are hourly ([HousePool.sol#L178](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L178)), so the first settlement after the recapitalization runs at a moved mark; an active keeper settling that epoch is what strands the surplus, not an absent one. The only settlement that would apply the recapitalization cleanly is one at the exact recap mark in the same block, which normal operation does not produce. The trigger is the rational, documented recapitalization of an impaired pool, capitalization of a healthy pool and which trapped no third-party deposit. The victim did have a cancellation window while Senior was impaired (step 1, proved by the PoC control), but committed Senior capital is designed to ride through impairment, and the recap-plus-gain ordering that closes that window is not something the depositor can foresee or control. It is not a known issue: Cecuro LOW-22 only notes that allocation `is deferred to normal reconciliation` (the healthy, cancellable case), whereas here normal reconciliation itself produces the trap, and no `SECURITY.md` Known Limitation covers it. It is in scope as `a legitimate, documented action by a trusted role that triggers a bug harming users` ([scope page](https://immunefi.com/audit-competition/plether/scope/)), and the program assumes `No detection or response deadline`, so the owner's manual `assignUnassignedAssets` cannot be relied on to end the freeze.

## Impact
A committed Senior LP deposit (2,000 USDG in the PoC) is frozen — neither activatable nor cancellable — from the first post-recap settlement until the owner manually reassigns the ownerless bucket, and the same gate defers activation of every other pending tranche deposit, so the frozen amount scales with queued deposit volume. The stranded cash itself (12.00 USDG) is small and recoverable, which is why the loss here is the freezing of the committed deposit, not the dust: the depositor's principal is locked with no self-service exit, not merely a missed return.

**Likelihood:** High — PoC-proven on the real harness with release parameters; it needs only the documented owner recapitalization of an impaired Senior and any positive surplus (a 0.014% tick) before the next of the hourly epoch settlements, both ordinary. It does not require idle keepers, and no actor profits, so there is no profit motive required.
**Impact:** Medium — temporary freezing of funds for at least 1 hour. `unassignedAssets` is `exceptional telemetry, not a normal operating bucket`, so the owner is not watching it, and the program assumes no detection or response deadline, so the freeze cannot be assumed to end quickly; the PoC shows it still in force a day and another settlement later.

## Mitigation
Block cancellation of a committed deposit only when the protocol can actually activate it. `HousePool.canSettleDepositEntries()` ([HousePool.sol#L506-L519](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L506-L519)) already returns false whenever `unassignedAssets > 0` (through the same `_projectedEntriesMaySettle` gate that defers activation), so adding it to the commitment guard in `TrancheVault.cancelPendingDeposit` restores a self-service exit in exactly the trapped state and nowhere else
```

## Proof of Concept

```markdown
## Proof of Concept
Add the PoC below to the existing `packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol`. That file extends the protocol's `AtomicLpEpochSettlementTestFixture` / `BasePerpTest` harness, which deploys the real `CfdEngine`, `HousePool`, both `TrancheVault`s, `PletherOracle` and the production `OrderRouter` through `LegacyOrderRouterHarness` (its only addition is a scalar `commitOrder` adapter onto the production V3 commit). Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocks. The PoC applies the release risk parameters through `CfdEngineAdmin`'s 72-hour timelock and the release Senior limits through `proposePoolConfig`/`finalizePoolConfig`; every trade is committed by the trader and executed by a keeper through `executeOrder`, and every LP settlement goes through `OrderRouter.settleLpEpoch` on a live market. Then run:

```bash
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_QueuedRecapTrapsCommittedSeniorDeposit -vvvv
```

```solidity
// packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol - add these imports:
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {console2} from "forge-std/console2.sol";

// and add these members and methods to contract AtomicLpEpochSettlementTest:
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant SENIOR_LP_2 = address(0x5E2);
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

    function _refreshMark(uint256 price) internal {
        vm.warp(vm.getBlockTimestamp() + 60);
        _setBasket(price, 0, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdateData());
    }

    function _trade(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price)
        internal
    {
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

    function _routerSettle(uint256 epochId, uint256 price) internal {
        _warpToEpoch(epochId);
        _setBasket(price, 0, vm.getBlockTimestamp());
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "live market");
        router.settleLpEpoch(_emptyUpdateData());
    }

    function _seniorTinyRedeem() internal returns (uint256 id) {
        uint256 shares = seniorVault.balanceOf(SENIOR_LP) / 10_000;
        vm.prank(SENIOR_LP);
        id = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
    }

    function _seniorDeposit(address lp, uint256 assets) internal returns (uint256 id) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), assets);
        id = seniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function test_PoC_QueuedRecapTrapsCommittedSeniorDeposit() public {
        _applyReleaseConfig();
        uint256 juniorId = _requestJuniorDeposit(ALICE, 25_000e6);
        _routerSettle(juniorId, 100_000_000);
        _claimJuniorDeposit(juniorId, ALICE);
        uint256 seniorId = _seniorDeposit(SENIOR_LP, 100_000e6);
        _routerSettle(seniorId, 100_000_000);
        vm.prank(SENIOR_LP);
        seniorVault.claimDeposit(seniorId, 100_000e6, SENIOR_LP, SENIOR_LP);

        // A second Senior LP queues a 2,000 deposit while the book is healthy.
        uint256 lp2Id = _seniorDeposit(SENIOR_LP_2, 2_000e6);

        address longB = address(0x1005);
        address shortB = address(0x1006);
        _fundTrader(LONG_TRADER, 20_000e6);
        _fundTrader(SHORT_TRADER, 20_000e6);
        _fundTrader(longB, 20_000e6);
        _fundTrader(shortB, 20_000e6);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 45_000e18, 2_000e6, false, 100_000_000);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 90_000e18, 2_000e6, false, 100_000_000);
        _trade(longB, CfdTypes.Side.LONG, 75_000e18, 2_000e6, false, 100_000_000);
        _trade(shortB, CfdTypes.Side.SHORT, 30_000e18, 2_000e6, false, 100_000_000);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 90_000e18, 0, true, 100_000_000);
        _trade(shortB, CfdTypes.Side.SHORT, 30_000e18, 0, true, 100_000_000);
        _refreshMark(72_000_000);

        // A routine settlement books the loss: Junior wiped, Senior impaired, no deficit, not degraded.
        vm.warp(seniorVault.lastDepositTime(SENIOR_LP) + seniorVault.DEPOSIT_COOLDOWN());
        _routerSettle(_seniorTinyRedeem(), 72_000_000);
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.terminalDeficitAssets(), 0);
        assertFalse(engine.degradedMode());

        // Control: while Senior is impaired the queued deposit is still cancellable.
        uint256 cancelProbe = vm.snapshotState();
        vm.prank(SENIOR_LP_2);
        assertEq(seniorVault.cancelPendingDeposit(lp2Id), 2_000e6, "cancellable while impaired");
        vm.revertToState(cancelProbe);

        // Owner recapitalizes exactly the Senior gap: the documented recovery action. Cash is only queued.
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        uint256 recapTime = vm.getBlockTimestamp();
        console2.log("Senior gap recapitalized by owner", gap);

        // A small, ordinary basket tick (0.72 -> 0.7201, ~0.014%) accrues before the next epoch settles.
        vm.warp(recapTime + 1 hours);
        _refreshMark(72_010_000);
        _routerSettle(_seniorTinyRedeem(), 72_010_000);

        uint256 unassigned = pool.unassignedAssets();
        console2.log("after settle: unassignedAssets / Senior gap now", unassigned, pool.seniorHighWaterMark() - pool.seniorPrincipal());
        console2.log("LP2 pending deposit (frozen principal)", seniorVault.pendingDepositRequest(lp2Id, SENIOR_LP_2));
        assertGt(unassigned, 0);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior now whole");

        // The deposit is trapped: cannot cancel (Senior whole) ...
        vm.prank(SENIOR_LP_2);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(lp2Id);
        // ... and the owner cannot route the ownerless cash back to Senior to clear the gate.
        vm.expectRevert(IHousePool.HousePool__ExceedsSeniorDepositCapacity.selector);
        pool.assignUnassignedAssets(true, SENIOR_LP);

        // One day and another normal settlement later, the 2,000 is still frozen.
        vm.warp(vm.getBlockTimestamp() + 1 days);
        _refreshMark(72_010_000);
        _routerSettle(_seniorTinyRedeem(), 72_010_000);
        console2.log("one day later: LP2 pending / unassignedAssets", seniorVault.pendingDepositRequest(lp2Id, SENIOR_LP_2), pool.unassignedAssets());
        assertEq(seniorVault.pendingDepositRequest(lp2Id, SENIOR_LP_2), 2_000e6, "still frozen");
    }
```

Reproduce on a clean machine (Foundry and the repo's git submodules, no other dependencies):

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# add the imports, members and methods above to packages/perps/test/perps/spec/lp/AtomicLpEpochSettlement.t.sol, then:
cd packages/perps && forge test --match-path test/perps/spec/lp/AtomicLpEpochSettlement.t.sol --match-test test_PoC_QueuedRecapTrapsCommittedSeniorDeposit -vvvv
```

Output:

```bash
Ran 1 test for test/perps/spec/lp/AtomicLpEpochSettlement.t.sol:AtomicLpEpochSettlementTest
[PASS] test_PoC_QueuedRecapTrapsCommittedSeniorDeposit() (gas: 17265868)
Logs:
  Senior gap recapitalized by owner 7035454078
  after settle: unassignedAssets / Senior gap now 12000000 0
  LP2 pending deposit (frozen principal) 2000000000
  one day later: LP2 pending / unassignedAssets 2000000000 12000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Expected vs. observed. While Senior is impaired the control branch cancels the 2,000 deposit for a full 2,000 refund, so the deposit is not stuck by itself. The owner then recapitalizes exactly the 7,035.454078 gap, a 0.014% tick lifts the next settlement's mark, and the surplus closes the gap before the queued recap is applied: `unassignedAssets = 12.00` with Senior exactly at its mark. From that point the 2,000 deposit can neither be cancelled (`TrancheVault__SeniorDepositCommitted`) nor activated (`unassignedAssets > 0`), and the owner's `assignUnassignedAssets(true, …)` reverts `HousePool__ExceedsSeniorDepositCapacity`; the deposit is still pending a day and another settlement later. The `-vvvv` trace shows the real path: `CfdEngine::recapitalizePool(7035454078) -> HousePool::recordClaimantInflow(7035454078, 1, 0)` recording `Recapitalization`/`CashArrived` and emitting `PoolRecapitalized`, every reconcile reached through `LegacyOrderRouterHarness::settleLpEpoch -> HousePool::settleLpEpoch`, no direct `HousePool::reconcile()` call, and the final `cancelPendingDeposit` reverting `TrancheVault__SeniorDepositCommitted`, with only `MockPyth` and `MockSettlementToken` as test doubles.
```
