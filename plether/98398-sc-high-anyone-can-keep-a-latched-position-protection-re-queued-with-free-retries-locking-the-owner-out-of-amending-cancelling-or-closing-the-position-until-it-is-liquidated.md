# \#98398 [SC-High] Anyone can keep a Latched position protection re-queued with free retries, locking the owner out of amending, cancelling, or closing the position until it is liquidated

**Submitted on Nov 7th 2026 at 16:49:21 UTC by @Raiders for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98398
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary
When a triggered stop-loss or take-profit fails because the price gapped through the trader's own close limits, the protection becomes `Latched`. In that state the owner has one way out: amend the limits, which requires `Latched` and zero pending orders. `retryPositionProtectionClose` is permissionless and free. A third party can terminalize each attempt and re-queue a new one in the same transaction, so between transactions the owner never sees `Latched` with an empty queue. Amend, cancel, cancel-and-close, replace and an ordinary close commit all revert, and the position stays open until it is liquidated.

## Root cause
Retry is open to anyone, costs no bounty, and copies the stored (infeasible) limits:

```solidity
// PositionProtectionBook.sol:382-406
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);      // no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    _recordCloseAttempt(protection, linkedOrderId);                             // status -> Triggered (:564-577)
}
```

`router/OrderLiquidationBatchLogic.sol:673` sets `maxExecutionBountyAssets = 0` on the retry. `:663-664` and `:674-677` copy the stored `closeFinancialLimits`. `:670` sets the window to `maxExecutionWindowSeconds`, which defaults to 60 s (`router/OrderExecutionOrchestrator.sol:10`).

Every owner exit needs a state that the retry removes:

```solidity
// PositionProtectionBook.sol:320-325  amendLatchedPositionProtectionCloseLimits
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();

// PositionProtectionBook.sol:346-348  cancelPositionProtectionAndCommitClose: Armed only
// PositionProtectionBook.sol:414-419  _cancel (cancelPositionProtection): PendingOpen or Armed only
// PositionProtectionBook.sol:271-276  replacePositionProtection: PendingOpen or Armed only

// router/OrderLiquidationBatchLogic.sol:313-315  ordinary commitOrder
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

A failed or expired attempt relatches the protection whenever the position is unchanged (`PositionProtectionBook.sol:637-668`, `Latched` at :666). So `Latched` with an empty queue exists only inside whichever transaction terminalizes the attempt. If the attacker sends that transaction, the attacker retries before it ends.

## Attack steps
Precondition (no attacker action). The victim opens a LONG and sets a stop-loss with a gross-debit cap, i.e. a stop-limit. The price gaps through the stop. The keeper's close attempt fails with `ConstraintViolation(GrossAccountDebit)` and the protection relatches with an empty queue (PoC `test_1`). Without an attacker, the victim would widen the cap, retry, and exit at the gap price for a realized loss of 880.52 USDG.

Model E (quiet market, 60 s window), from `test_2`:
1. The attacker back-runs the keeper's failed execution with `retryPositionProtectionClose(pid)`. The protection is now `Triggered` with a zero-bounty child that carries the same infeasible cap.
2. No keeper is paid to execute that child. At `executionDeadline + 1`, the attacker's helper contract calls `router.executeOrder(attempt, [])` (oracle-free expiry), which relatches, then `retryPositionProtectionClose(pid)` in the same transaction.
3. The attacker repeats every 60 s. Between transactions the protection is always `Triggered` with one live child.
4. The victim's amend reverts `ProtectionNotLatched`. Cancel, cancel-and-close and replace revert `ProtectionNotArmed`. `commitOrder(close)` reverts `ProtectionActive`.
5. If the victim expires the attempt themselves, the attacker sees that transaction on the sequencer feed and lands a retry before the victim's follow-up amend.
6. The price keeps drifting against the victim. Once the account crosses maintenance, the attacker liquidates it and collects the keeper share.

Model K (`test_3`). An active keeper executes each attempt about 1 s after commit. The attempt fails and relatches. The attacker back-runs each keeper transaction with a plain retry (0.808M gas).

Model S (`test_4`). The attacker front-runs the keeper every second with its own oracle-path execute+retry (1.614M gas with mock Pyth), so the state is never `Latched` between transactions.

## Impact
Measured in the PoC (no fork; contest harness with production risk params from DEPLOYMENT.md:131-140 and the production 60 s window):
- Lock: 1,418 attacker cycles over 86,498 s (1,153,451 average gas per cycle). The victim's 100,000-unit LONG is still open at the end.
- 96 of 96 victim EOA transactions revert: 16 each of amend, cancel, cancel-and-close, `commitOrder` close, replace, and expire-then-amend.
- End state: the attacker liquidates at 1.0205. Victim wealth goes from 20,000.00 to 17,999.80 USDG, a loss of 2,000.20 (full margin plus the 0.20 trigger bounty). Exiting at the gap would have cost 880.52, so the lock costs the victim an extra 1,119.68 USDG. The attacker receives 50.00 USDG as keeper share.
- Model K: 3,600 of 3,600 cycles held for 1h, and 12 of 12 victim transactions sequenced after the back-run revert.

Who loses: any trader whose triggered protection fails on its own limits. A stop-limit that the price gaps through is the normal way this happens. The MED-9 fix added these trader limits on purpose, so the state is expected, not exotic. The loss is the victim's margin. The victim can still add margin to delay liquidation but cannot reduce exposure.

#### Likelihood and limits
- The victim can escape when it can terminalize the live attempt and amend in one transaction, or in an uninterrupted burst. A 4337 smart account (`executeBatch` is allowed by the protocol's paymaster, perps-aa/src/PletherVerifyingPaymaster.sol:79), an EIP-7702 account, or an EOA that pre-signs execute+amend+retry as a nonce burst escapes Models E and K (`L9_Escape_Refute.t.sol::test_Escape_7702Atomic`, `::test_Escape_EoaBurstMidWindow`). That needs a self-executed attempt with a Pyth payload. Ordinary wallets that wait for each receipt do not do this.
- Model S defeats the atomic escape: a fresh attempt in the same block or publish-second stays pending (`::test_StrongAttacker_SameBlockCycleBlocksAtomicEscape`). It needs the attacker to win every second, which in practice means the Timeboost express lane.
- Model K: a victim transaction that lands between the keeper's transaction and the attacker's back-run succeeds. A randomly timed EOA retry escapes with probability of about attacker latency / keeper period per try (for example 50 ms / 1 s = 5%).
- If the price returns inside the victim's limits, the next attempt fills and the grief ends.

#### Known-issue check
SECURITY.md:547-552 documents that triggering is irreversible, that `Latched` keeps the account trade lock, and that retry is permissionless. The Known Limitations section (SECURITY.md:1356-1359) says failed protection-attempt cleanup is unpaid and that "Liveness depends on keepers processing the terminal attempt and separately appending a retry". That entry covers retries not happening. None of these texts says a third party's retries can block the owner's amendment, and that root cause is not on the program's known-issues list. The acknowledged MED-9 response names the owner's amendment as the remedy for tight limits: "Tight limits may prevent execution until conditions improve or the owner amends them." This bug removes that remedy.

## Recommended fix
Give the owner a path out that third parties cannot close. Any one of these works:

1. Let the owner amend while an attempt is live. In `amendLatchedPositionProtectionCloseLimits`, also accept `Triggered`, and have the Router terminalize the live child (relatch) in the same call before applying the new limits. The owner can then retry in the same transaction. This keeps retries permissionless.
2. Let `cancelPositionProtection` (or cancel-and-close) accept `Triggered`/`Latched` from the owner, terminalizing the live child first. This conflicts with the documented "triggering is irreversible" rule, so option 1 fits the design better.
3. Restrict third-party retries. Record `latchedAt` in `handleFailedProtectionAttempt` and let only `protection.account` retry before `latchedAt + RETRY_GRACE`:

```solidity
// PositionProtectionBook.retryPositionProtectionClose
if (msg.sender != protection.account && block.timestamp < protection.latchedAt + RETRY_GRACE) {
    revert PositionProtectionBook__OwnerRetryWindow();
}
```

The grace window gives the owner an uncontested chance to amend, and keepers still provide liveness after it. A retry bond or bounty paid by non-owner retriers would also raise the attacker's cost, but on its own it does not restore the owner's escape.


## Proof of Concept

## Proof of Concept
Files: `packages/perps/test/poc/H_L9_1_LatchedGriefFinal.t.sol` (4 tests) and optionally `packages/perps/test/poc/L9_Escape_Refute.t.sol` (escape and strong-attacker cases). They extend the repo's `PositionProtectionTest` harness and need no RPC.

Attacker helper (no roles, no funds, no bounty):

```solidity
contract LatchGriefer {
    function cycle(address router, address book, uint64 attempt, uint64 pid, bytes[] calldata data)
        external
        returns (uint64)
    {
        if (
            IPositionProtectionViews(book).getPositionProtection(pid).status
                == PositionProtectionTypes.PositionProtectionStatus.Triggered
        ) {
            IRouterExec(router).executeOrder(attempt, data); // expiry (empty data) or live execution
        }
        return IPositionProtectionActions(book).retryPositionProtectionClose(pid);
    }
}
```

Core of `test_2_ModelE_ExpiryGrief24h_ThenLiquidation` (victim and mark-keeper actions inside each window omitted):

```solidity
(uint64 pid, uint64 attempt) = _naturalPrecondition(false);   // gap 1.0050 -> 1.0080, ConstraintViolation, Latched
(attempt,) = _attackerCycle(attempt, pid, empty);             // back-run the keeper failure
while (vm.getBlockTimestamp() < end) {
    uint256 dl = _deadline(attempt);
    // ... victim tries one exit every 15 min; each is wrapped in vm.expectRevert ...
    _nextBlock(dl + 1);
    (attempt, gc) = _attackerCycle(attempt, pid, empty);      // expire + re-queue atomically
}
assertEq(size, V_SIZE, "victim position still open after 24h");
assertEq(victimFailures, victimTries, "every victim EOA attempt failed");
assertGe(lockSeconds, 24 hours, "locked >= 24h");
uint256 attackerGain = _liquidateAsAttacker(LIQ_PRICE);       // 1.0205
```

Run (from the repo root at a82d4fef):

```
cd packages/perps && FOUNDRY_FFI=false forge test --match-path test/poc/H_L9_1_LatchedGriefFinal.t.sol --match-test "test_[1-4]_" --gas-limit 20000000000 -vv
```

The `--match-test` filter is required. The production risk override breaks five inherited `test_PostLockRisk_*` harness fixtures.

Output:
```
[PASS] test_1_NaturalPrecondition_And_Counterfactual() (gas: 8213722)
  victim wealth before (USDG 1e6) 20000000000
  counterfactual exit at 1.0080: victim wealth after 19119479955
  counterfactual realized loss (USDG 1e6) 880520045
[PASS] test_2_ModelE_ExpiryGrief24h_ThenLiquidation() (gas: 1581653079)
  execution window (s) 60
  lock duration (s) 86498
  attacker cycles 1418
  attacker avg gas per cycle 1153451
  attacker max gas per cycle 1157695
  attacker total gas over 24h 1635593535
  victim attempts (all reverted) 96
   amend / cancel / cancelAndClose 16 16 16
   commitOrder close / replace / expire-then-amend 16 16 16
  victim wealth before (USDG 1e6) 20000000000
  victim wealth after liquidation 17999800000
  victim total loss 2000200000
  attacker liquidation keeper share 50000000
[PASS] test_3_ModelK_ActiveKeeper_BackrunRetry_1h() (gas: 6569070999)
  sample duration (s) 3600
  keeper-terminalized cycles in sample 3600
  attacker avg retry gas (back-run) 807608
  victim attempts (all reverted) 12
[PASS] test_4_ModelS_FrontrunEachSecond_GasSample() (gas: 990748705)
  oracle-path cycles in 10 min 600
  attacker avg oracle-path cycle gas (mock Pyth) 1614296
Suite result: ok. 4 passed; 0 failed; 0 skipped
```
