# \#97876 [SC-High] Permissionless retry keeps a gapped stop-loss in `Triggered`, owner's MED-9 amend never lands and traders cannot exit

**Submitted on Nov 3rd 2026 at 17:55:09 UTC by @roulette6041 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97876
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

The Cecuro MED-9 fix stores trader-chosen close ceilings on a take-profit / stop-loss. If a gap pushes the close outside those ceilings, every close attempt fails. The fix's stated recourse is that the owner amends the ceilings while the protection is `Latched`.

Any caller can take that recourse away. A keeper contract calls `executeOrderBatch(attemptId)`, which runs the global FIFO up to and including the failing attempt. In the same transaction it calls `retryPositionProtectionClose`. When the attempt is already the queue head, this is just `executeOrder(attemptId)` followed by the retry. The failed attempt relatches the protection and the retry flips it straight back to `Triggered` with a fresh attempt. So `Latched` never exists at a transaction boundary, and the owner's amend always reverts. Other traders' orders in the queue do not break this.

While this runs, the owner cannot close, cancel or amend. The position stays open and the trader's loss follows the price. In the PoC the trader loses 190.20 USDG instead of 93.74 USDG on a 200 USDG-margin position.

Audited commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`immunefi-team/audit-comp-plether`).

---

## Vulnerability Details

### 1. MED-9 promises an owner amend as the recourse for tight limits

Cecuro MED-9 ("Keepers can retry protection closes without the trader's financial limits", [audit PDF, page 26](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)) is listed as Acknowledged, with a team comment marking it "Addressed" and fix commit [`8a2853f8`](https://github.com/Plether-Fi/plether-core/commit/8a2853f80bce067656c1fd0fcf2dadc5892e2aee). The response reads:

> Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner amends them.

The first half of that last sentence is accepted behaviour, and I am not reporting it. A stop-limit that gaps through its limit does not fill. This report is about the second half: the owner amend is unreachable once any caller chooses to keep the protection cycling. The program accepts a report on a known finding when the fix introduces a new bug, and this is that case.

[`CANONICAL_ENTRYPOINTS.md:145-148`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L148) documents the same recourse: the owner "may amend the close ceilings of a `Latched` protection [...] when the account has no pending Router order".

### 2. Three code facts produce the lockout

**Amend needs `Latched` and no pending order of the owner's.** [`PositionProtectionBook.sol:315-328`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) reverts `OrderRouter__ProtectionNotLatched` unless `status == Latched`. It also reverts `OrderRouter__PendingOrdersExist` if the owner has any pending Router order.

**Retry is permissionless and leaves `Latched` immediately.** [`PositionProtectionBook.sol:382-406`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) loads the protection with `_activeProtection`, not `_ownedActiveProtection`, so there is no caller check. It commits a fresh close attempt and records it, which sets the status back to `Triggered`.

**A failed attempt relatches inside the caller's transaction.** [`PositionProtectionBook.sol:662-667`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667) sets `status = Latched` in the Router's failure callback, during `executeOrder`. Nothing stops the same transaction from calling retry next.

Put together, one transaction of `executeOrderBatch(attempt)` then `retryPositionProtectionClose(id)` goes `Triggered -> Latched -> Triggered`. Every other transaction sees `Triggered`.

The batch form matters when other orders sit ahead of the attempt. `executeOrder(id)` only runs the queue head and reverts `OrderRouter__OrderNotQueueHead` otherwise ([`OrderRouterExecutionSidecar.sol:201-206`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L201-L206)). `executeOrderBatch` walks the FIFO from the head, so the griefer executes those orders on the way to its attempt.

### 3. In `Triggered` the owner has no exit at all

- Amend reverts `ProtectionNotLatched` (above).
- `cancelPositionProtection` reverts `ProtectionNotArmed`: cancel only accepts `PendingOpen` or `Armed` ([`:414-419`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419)).
- `cancelPositionProtectionAndCommitClose` reverts `ProtectionNotArmed` ([`:346`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L346-L348)).
- An ordinary `commitOrder(close)` reverts `ProtectionActive`, because every commit path passes `enforceProtectionLock = true` ([`OrderLiquidationBatchLogic.sol:313-315`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)).

That also means the owner cannot park a pending order of their own to make the griefer's retry revert with `PendingOrdersExist`.

### 4. The owner cannot execute the fresh attempt in the second it was committed

A natural escape is for the owner to run the same trick first: execute the live attempt, amend, and retry in one bundle. That works only in a second the griefer does not get to first.

The reason is the oracle window for order execution. [`PletherOracle.sol:610-614`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614) only accepts a price with `publishTime` in `[commitTime + 1, min(commitTime + orderSettlementWindow, block.timestamp)]`. `orderSettlementWindow` is 15 seconds ([`:87`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L87)). In the second the attempt was committed, `block.timestamp == commitTime`, so the lower bound is above the upper bound and no payload qualifies. The parse failure is caught at [`:629-642`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L629-L642), and the attempt simply stays `Pending` (`HistoricalPriceUnavailable`). The amend in the owner's bundle then reverts `ProtectionNotLatched`, and the whole bundle reverts.

So each second is one race. Whoever lands first with a price for that second decides it. If the griefer is first, it executes the attempt from the previous second (and any orders ahead of it), retries, and commits a new attempt that nobody can touch until the next second. This follows from the in-scope bounds alone. The PoC uses the vendor's `MockPyth`, since Pyth is out of scope.

The owner's account type does not change this. An EOA, an EIP-7702 delegated EOA (available on Arbitrum One since ArbOS 40, [Constitutional AIP: ArbOS 40 Callisto](https://forum.arbitrum.foundation/t/constitutional-aip-arbos-version-40-callisto/28436)) and a smart account all lose a second the griefer wins. The race tests use a contract account, which is the strongest owner.

### 5. The listed Router and keeper limitation describes a different state

The program lists the Router and keeper limitations in SECURITY.md as known. The relevant item is [`SECURITY.md:1356-1359`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359):

> failed protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record. The separate trigger reward is paid once on the original trigger. Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators must alert on prolonged `Latched` state without a live child,

That item covers the case where nobody retries. The protection then sits in `Latched` with no live child, the close never happens, and the operator alert catches it.

The griefed protection is never in that state. At every transaction boundary it is `Triggered` with a freshly committed live child. `Latched` exists only inside the griefer's own transaction. So the documented alert never fires, and keeper liveness is not the issue: the retry happens as fast as the chain allows. The defect is that it happens too fast for the owner. The text's own word "separately" describes the two-transaction pattern that leaves the owner a window. [`CANONICAL_ENTRYPOINTS.md:281-284`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L281-L284) says the same: keepers should "process the terminal attempt, observe `Latched`, confirm the exact position and retry preconditions, and submit the separate retry transaction". Nothing on-chain enforces that separation.

The program treats keepers as untrusted and asks for any contract-permitted behaviour to be considered, including deviations from the reference keepers. An atomic execute-and-retry is such a deviation.

### 6. No pause or admin action frees the owner

This comes from reading the code. I did not test each trusted action against the griefer.

- **Pause does not change the race.** Neither retry ([`PositionProtectionBook.sol:382-406`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)) nor amend ([`:315-328`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)) has a pause gate. The Book's pause check `_validateProtectionCommitAllowed` runs only on create, replace and attach ([`:220`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L220), [`:269`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L269), [`:451`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L451)).
- **Risk-off does not touch the attempt.** The risk-off cutoff applies to opens only ([`OrderRouterExecutionSidecar.sol:900-902`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L900-L902)), and a protection attempt is a close.
- **A config change does not invalidate the attempt.** Protection attempts carry `expectedConfigHash = bytes32(0)` ([`OrderLiquidationBatchLogic.sol:672`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L672)), so a later config finalization does not fail them.
- **Nobody can intervene in the Book itself.** `PositionProtectionBook` inherits no ownership contract ([`:135`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L135)), so there is no admin hook to reset a protection.

The vendor's emergency guide says the same. Its breaker table at [`EMERGENCY_RESPONSE_GUIDE.md:71`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/EMERGENCY_RESPONSE_GUIDE.md#L71) and [`:73`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/EMERGENCY_RESPONSE_GUIDE.md#L73) reads:

> | Commit/execute a close or reduction | Allowed | Allowed | Allowed | No discretionary close-off breaker exists |
>
> | Cancel, trigger, or retry existing protection | Allowed | Allowed | Allowed | Position, oracle, and queue checks still apply |

So retries stay available in all three breaker modes, and pausing does not stop the cycle. One trusted action can end a lock, though not the one demonstrated here: the owner's 72-hour timelocked risk/fee config change ([`CfdEngineAdmin.sol:109-141`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L109-L141), [`TIMELOCK_DELAY` at :20](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L20)) could make an action-charge or fee ceiling fit again. That frees the owner by letting the protection close execute, not by enabling the amend. It does not reach the notional ceiling the PoC exercises, and the action-charge case does not arise at release parameters (`test_H7b_CarryCeiling_DoesNotTripAfterOneYear`: 3.21 USDG of carry after a year against a 10 USDG ceiling).

---

## Impact

The trader asked to exit at a bounded price and cannot exit at any price while the griefer holds the cycle. The trader cannot amend, cancel, or close. The loss follows the price path for as long as the lockout lasts, and on an adverse path it runs to liquidation.

Measured in `test_H7a_GrieferKeepsLatchedClosedUntilLiquidation`, at the DEPLOYMENT.md release parameters (asserted in `test_H7_ProdParams`):

| | Griefed | Counterfactual (owner amends while `Latched`, closes) |
|---|---|---|
| Trader balance start | 255.30 USDG | 255.30 USDG |
| Exit | liquidated at 1.018 after 10 cycles | closed at 1.009 |
| Loss | **190.20 USDG** | **93.74 USDG** |

The extra loss is 96.46 USDG, about 48% of the 200 USDG margin, on a 10,000 USDG notional 50x LONG. The test compresses the adverse drift to 0.1% per cycle to stay short. The number is illustrative: the extra loss scales with notional and with how far price moves during the lockout. The griefer gains nothing from it.

### What the griefer needs, and how the owner gets out

1. **The griefer has to win ordering in every second.** One second won by the owner frees the position (`test_Race2_Control`: the owner escapes the first second it lands first, and an honest keeper closes it next second). On Arbitrum One a griefer would need the Timeboost express lane or a reliable latency edge. I have not measured either on a live sequencer.
2. **The griefer pays for every second.** Measured gas is about 1.46M per cycle cold with `MockPyth`. Real Pyth verification adds more. That is 3,600 cycles an hour. At an assumed 0.01 gwei L2 base fee and $3,000 ETH this is roughly **$160-210 per hour per locked position**, plus two Pyth update fees per cycle ([`PletherOracle.sol:604-607`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L604-L607)), plus any express-lane cost. These are estimates, not measurements.
3. **Frozen-market windows give the owner an escape that does not depend on ordering.** While the oracle is frozen (FX weekends), the same-block guard is off ([`OrderRouterExecutionSidecar.sol:447-452`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L452)), and the owner's bundle succeeds even after the griefer in the same block (`test_Race3c`). A highly levered position may be liquidated before the weekend arrives.
4. **The owner can add margin.** That delays liquidation but does not close the position or change the protection state (`test_Race3b`).
5. **The loss depends on the path.** If price comes back inside the ceiling, the next attempt executes at the trader's own limit and the lockout ends.
6. **The ceiling was reasonable.** The stop's notional ceiling is stop notional plus 0.1%, and it fits when the stop fills at the stop (`test_H7a_CeilingFitsAtStop`). The trader could only have loosened it while `Armed`. After the gap, amend is the only lever left, which is exactly the lever that is blocked. Only the notional ceiling was exercised. I did not run the gross-debit ceiling.
7. **Other traders' orders do not break the lock, but they add costs and a second race.** `H7Bystander.t.sol` runs real opens and closes from 12 bystander accounts at the release parameters. Across two 20-second streams of 1-3 bystander orders per second, the owner escaped 0 times in 120 attempts. It also escaped 0 times in every targeted case (a bystander ahead of the attempt, a stale bystander ahead, a bystander behind it). Three qualifiers:
   - **The griefer has to be FIFO-aware.** An id-targeted `executeOrder` reverts `OrderNotQueueHead` whenever a live order sits ahead of the attempt, and in that second the owner escapes (tested). The working griefer uses `executeOrderBatch` and then retries.
   - **Each order ahead costs the griefer extra gas.** That is about 0.8-1.2M gas per order ahead, and nothing when bystanders sit behind the attempt. Open bounties (at most 0.20 USDG each) partly offset it. Closes pay nothing ([`DEPLOYMENT.md:150`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L150)).
   - **Honest keepers draining the queue create a second ordering race.** When a keeper executes the attempt first, the protection is `Latched` at a transaction boundary. The owner then escapes if its amend lands before the griefer's back-run retry (669k gas). If the back-run lands first, the owner is locked again (`test_Fifo_HonestKeeperFirst_CreatesLatchedBoundary`). I have not measured this race on a live sequencer.

   The owner cannot use the queue against the griefer either. Flooding 10 opens from sub-accounts ahead of the next retry does not break the cycle: the griefer clears them and earns 2.00 USDG in open bounties (`test_Fifo_OwnerFloodAhead_DoesNotBreakCycle`).

---

## Risk Breakdown

- **Affected contract:** `PositionProtectionBook` (`amendLatchedPositionProtectionCloseLimits`, `retryPositionProtectionClose`, `handleFailedProtectionAttempt`).
- **Preconditions:** a protection whose stored ceilings no longer fit after its trigger (for example a stop-loss that gaps through its notional ceiling), and a griefer that lands first in each second and clears any orders ahead of the attempt.
- **Who can exploit:** anyone. No role is needed, and the griefer is an unprivileged contract.
- **Funds at risk:** the trader's margin on the locked position, according to the price path while locked. The attacker gains nothing.
- 
---

## Recommendation

Give the owner a short window after each relatch in which only the owner may retry. A lone amend can then always land between a failed attempt and the next retry.

```diff
@@ PositionProtectionBook.sol:169 (storage)
     mapping(uint64 linkedOrderId => uint64 protectionId) private _attemptProtectionIds;
+    /// @notice Seconds after a relatch during which only the owner may retry, so a lone amend can land.
+    uint256 private constant OWNER_AMEND_WINDOW = 60;
+    mapping(uint64 protectionId => uint256 relatchedAt) private _ownerAmendWindowStart;

@@ PositionProtectionBook.sol:386-389 (retryPositionProtectionClose)
         if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
             revert OrderRouter__ProtectionNotLatched();
         }
+        if (
+            msg.sender != protection.account
+                && block.timestamp < _ownerAmendWindowStart[protectionId] + OWNER_AMEND_WINDOW
+        ) {
+            revert OrderRouter__ProtectionNotLatched();
+        }
         _requireMatchingPosition(protection);

@@ PositionProtectionBook.sol:665-667 (handleFailedProtectionAttempt)
         if (retained) {
             protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
+            _ownerAmendWindowStart[protectionId] = block.timestamp;
             return true;
         }
```

I applied this diff to a working copy at the pin and ran it. The fix check was then re-run in a separate clean copy of the repository at the pin, where the fix tests and the EIP-3860 test both pass.

- The griefer's atomic execute-and-retry now reverts at the retry. Executing alone still relatches. The owner's lone amend lands, the owner retries, and the close executes. After 60 seconds a non-owner retry works again.
- Every griefing, race and FIFO test in the three PoC files now fails with `ProtectionNotLatched` at the first non-owner retry after a relatch, which is the intended effect. The check is on the retry's caller, so it applies the same way to the batch griefer as to the id-targeted one.
- This version does not reset the window on amend, to save bytes. After amending, the owner retries itself, or any keeper can after 60 seconds. The PoC counterfactual, where a keeper retries straight after the owner's amend, fails for that reason.
- The vendor's EIP-3860 initcode test (`test_OrderRouterCreationCodeAndConstructorArgsFitsEip3860`) still passes. `OrderRouter` initcode plus arguments goes from 49,052 to 49,126 bytes against the 49,152 limit. The Book is created by the Router constructor, so headroom is tight. A version with a dedicated custom error, a public constant and a reset on amend went over the limit by 6 to 76 bytes, which is why this version reuses `ProtectionNotLatched`. A dedicated error is cleaner if bytes can be found elsewhere.
- 12 vendor protection tests fail. Each of them has a keeper retry immediately after a relatch (for example `test_RetryPositionProtectionClose_IsPermissionlessFreshAndAppendsFifoTail`). That is exactly the behaviour this changes. Those tests would need a 60-second warp or an owner caller.

The trade-off is that a passive owner's retry waits up to 60 seconds after each failed attempt. That attempt has just failed on the same ceilings, so the delay costs little. The owner can retry at once.

Two alternatives if a time window is unwanted:
- Let the owner amend while `Triggered` by cancelling the live attempt atomically inside `amendLatchedPositionProtectionCloseLimits`, then relatching. This needs a Router hook to cancel the child, so it is a larger change. I have not tested it.
- Reject a retry in the same transaction as the relatch (for example, record the relatch block and require a later block for non-owner retries). This removes the atomic pattern but still leaves the owner racing the next block, so I recommend the window instead.

---

## References

- Amend: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- Retry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- Relatch on failed attempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667
- Cancel status gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419
- Protection lock on commits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- Oracle execution window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L642
- Frozen-mode guard: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L452
- Owner amend and keeper retry docs: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L148 and #L281-L284
- Known limitation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359
- Cecuro MED-9 (page 26): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- MED-9 fix commit: https://github.com/Plether-Fi/plether-core/commit/8a2853f80bce067656c1fd0fcf2dadc5892e2aee
- ArbOS 40 (EIP-7702 on Arbitrum One): https://forum.arbitrum.foundation/t/constitutional-aip-arbos-version-40-callisto/28436


## Link to Proof of Concept

https://gist.github.com/chipsforbeer/62a6437db554a7c5e1720db714ffc913

## Proof of Concept

## Proof of Concept

PoC (secret gist, three Foundry test files plus a README with placement and commands): https://gist.github.com/chipsforbeer/62a6437db554a7c5e1720db714ffc913

Place the three `.t.sol` files at `packages/perps/test/_bounty/` in the repository at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The run commands are below and in the gist README.

- `H7RetryPreemptsAmend.t.sol` (9 tests): core griefing, the counterfactual, the ceiling-fits-at-stop precondition, the release-parameter check, and escape tests in which the owner acts directly after one griefer cycle.
- `H7EscapeRace.t.sol` (7 tests plus the 9 it inherits): signed per-second payloads and explicit block and second control. An always-first griefer, the owner-first control, a multi-tick bundle, the margin deposit, the frozen window, and a forced-failure control that proves the success counter can fire.
- `H7Bystander.t.sol` (9 tests plus the 16 it inherits): a non-empty global FIFO with real opens and closes from 12 bystander accounts. It adds the FIFO-aware batch griefer, a bystander ahead of and behind the attempt, two 20-second streams, the owner's flood counter, a griefer-idle control, and the honest-keeper-first race.

The first two files use an id-targeted griefer (`executeOrder(attemptId)`), which is equivalent to the batch form whenever the attempt is the queue head, as it is in those tests.

They use the vendor's `BasePerpTest` with the real Engine, Router, protection Book, lifecycle Book, Clearinghouse, HousePool and PletherOracle. `MockPyth` stands in for Pyth. No in-scope contract is modified. Positions open through the real Router path (`commitOpenOrderWithProtection` then `executeOrder`).

Place all three files at `packages/perps/test/_bounty/` in the repository at the audited commit (each later file inherits the earlier one), then run from the repository root:

```bash
forge test --root packages/perps --match-path test/_bounty/H7RetryPreemptsAmend.t.sol -vv
forge test --root packages/perps --match-path test/_bounty/H7EscapeRace.t.sol --match-contract H7EscapeRaceTest -vv
forge test --root packages/perps --match-path test/_bounty/H7Bystander.t.sol --match-contract H7BystanderTest -vv
```

Results at `a82d4fef`: 9/9, 16/16 and 25/25 pass.

The FIFO-aware griefer's cycle (`H7BatchGriefer`), and the lockout checked before every cycle:

```solidity
function cycle(uint64 protectionId, uint64 attemptId, bytes[] calldata data)
    external returns (OrderV3Types.BatchResult memory result, uint64 nextAttemptId)
{
    result = ROUTER.executeOrderBatch(attemptId, data);          // clears orders ahead; attempt fails, relatches
    nextAttemptId = BOOK.retryPositionProtectionClose(protectionId); // Latched -> Triggered
}

// _assertOwnerLockedOut, run at every tx boundary:
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
book.cancelPositionProtection(pid);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
book.cancelPositionProtectionAndCommitClose(pid, closeReq);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
router.commitOrder(closeReq);
```

Output excerpts:

```
[PASS] test_H7a_GrieferKeepsLatchedClosedUntilLiquidation()
  cycles: 10
  liquidation price (1e8): 101800000
  equity start (USDG 1e6): 255300999
  equity end, griefed (USDG 1e6): 65100996
  loss, griefed (USDG 1e6): 190200003
[PASS] test_H7a_Counterfactual_OwnerAmendsAndCloses()
  loss, counterfactual (USDG 1e6): 93736999
[PASS] test_Race2_GrieferAlwaysFirst_OwnerNeverEscapes()
  owner attempts: 40
  owner successes (griefer always first): 0
  griefer cycles: 20
[PASS] test_Race2_Control_OwnerFirstInAlternateSeconds_Escapes()
[PASS] test_Race3c_FrozenWindow_OwnerEscapesAfterGriefer()
[PASS] test_Race_ForcedFailure_DetectorFires()
[PASS] test_FifoIII_Stream_HonestKeeper()
  owner attempts: 60
  owner successes: 0
[PASS] test_FifoIII_Stream_NoKeeper_GrieferClearsAll()
  owner attempts: 60
  owner successes: 0
[PASS] test_FifoI_BystanderAheadOfRetry_BatchGrieferStillBlocks()
  griefer extra gas for 1 bystander: 786463
[PASS] test_Fifo_OwnerFloodAhead_DoesNotBreakCycle()
  extra gas per order ahead: 1117495
  griefer bounty earned from owner's flood (USDG 1e6): 2000000
[PASS] test_Fifo_HonestKeeperFirst_CreatesLatchedBoundary()
  griefer back-run retry gas: 669126
```

In `test_Race1`, a fresh attempt probed in its commit second returns `Pending` with `pendingReason 4` (`HistoricalPriceUnavailable`). The owner's adaptive bundle reverts `0x5b54e624` (`OrderRouter__ProtectionNotLatched`). Re-signed with the next second's price and landing first, the same bundle succeeds.

