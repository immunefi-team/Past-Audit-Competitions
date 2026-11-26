# \#99206 [SC-High] Any account can keep a triggered stop-loss owner trapped in their position by consuming the only exit state, `amendLatchedPositionProtectionCloseLimits`

**Submitted on Nov 12th 2026 at 11:50:47 UTC by @Naveen0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99206
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

A TP/SL protection that triggers queues a zero-bounty full-position close attempt. If that attempt fails on the owner's stored close limits, the protection relatches and the position stays open with the account trade-locked. The only remedy left to the owner is `amendLatchedPositionProtectionCloseLimits`, which exists because of the MED-9 fix, and which is owner-only and requires the protection to be `Latched` with zero pending orders for that account.

`retryPositionProtectionClose` requires the identical state and is permissionless, free, uncapped in frequency and indifferent to who calls it. Because a failed attempt relatches inside the executing transaction, a single transaction executes the doomed attempt and immediately queues a fresh one. At every transaction boundary the protection is `Triggered` with a live child, so the owner's amend reverts with `ProtectionNotLatched`, cancel, replace and cancel-and-close revert with `ProtectionNotArmed`, and every ordinary `commitOrder` reverts with `ProtectionActive`. The owner cannot close, reduce or withdraw the position margin, and the position stays exposed to the market throughout.

The attacker can also manufacture the failure that starts this. Close VPI is part of the action charge, and the skew cap constrains opens only, so a fresh account can hold up to 0.4 × pool depth of opposite-side skew. Closing the victim then moves absolute skew further from zero and pushes the victim's close charge above its own `maxActionChargeAssets`. In the PoC an attacker holding 25,000 USDG raises the victim's close charge from 4.13 USDG to 53.29 USDG against a 30.25 USDG stored ceiling — a 25 bp ceiling on a close that normally costs 3.4 bp.

At release parameters this holds an owner for 24 hours with 982.48 USDG of position margin frozen, at one griefer transaction per 60-second execution window and with no sequencing advantage whatsoever. A faster variant also defeats an owner who batches `executeOrder + amend + retry` twice per second, though that one does assume the griefer is sequenced first within each second, which is stated plainly in the limitations.

## Affected code

All references are against `packages/perps/src` at the commit in scope.

- `PositionProtectionBook.sol:315-328` — `amendLatchedPositionProtectionCloseLimits`. The owner's only post-trigger remedy. Requires `status == Latched` and `pendingOrderCounts(msg.sender) == 0`.
- `PositionProtectionBook.sol:382-406` — `retryPositionProtectionClose`. Resolves the record through `_activeProtection`, not `_ownedActiveProtection`, so any address may call it. Gates on the same two conditions and ends in `_recordCloseAttempt`, which sets the status back to `Triggered`. No cooldown, no fee, no owner priority.
- `PositionProtectionBook.sol:637-675` — `handleFailedProtectionAttempt`. On a retryable failure with the protected position unchanged it sets `Latched` (`:663-666`) inside the transaction that executed the failing attempt.
- `OrderRouter.sol:240-247` — `executeOrder`, and `:256-262` `executeOrderBatch`: `external payable nonReentrant`, with no caller restriction. These are what let an arbitrary account drive the failing attempt to its terminal state on demand.
- `OrderRouter.sol:324-329` — the execution path that calls `handleFailedProtectionAttempt` on a terminal, retryable close attempt. `OrderRouter.sol:191-230` is the equivalent expiry path, which relatches the same way.
- `router/OrderLiquidationBatchLogic.sol:313-315` — the commit-time protection lock: any `commitOrder` from an account with an active protection reverts `OrderRouter__ProtectionActive`.
- `router/OrderLiquidationBatchLogic.sol:657-680` — the retry attempt copies the protection's stored close limits, so a persistent failure reproduces identically on every attempt.
- `CloseAccountingLib.sol:107-109` and `CfdEnginePlanLib.sol:1041-1044` — signed VPI is part of the close action charge.
- `CfdOrderPolicyEvaluatorBase.sol:509-513` — the evaluator compares that charge against the stored `maxActionChargeAssets` and fails the order with `ConstraintViolation(ActionCharge, ...)`.
- `interfaces/IPositionProtectionActions.sol:32` — the documented contract the code does not keep: the owner may change close limits "after a failed attempt relatches, before a fresh retry".
- `interfaces/IPositionProtectionActions.sol:69-72` — the retry's own NatSpec, which frames permissionless retry as a safety action that keeps the mandated close alive.

## Root cause

**1. The owner's only post-trigger exit needs a state that a third party can take first.**

`amendLatchedPositionProtectionCloseLimits`:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

`retryPositionProtectionClose` gates on the same two conditions, but resolves the record with `_activeProtection`, so there is no `msg.sender` check at all:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
_requireMatchingPosition(protection);
...
if (router.pendingOrderCounts(protection.account) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
...
_recordCloseAttempt(protection, linkedOrderId); // status back to Triggered
```

The two functions are in a race, with no cooldown, rate limit or grace period reserved for the owner, and the owner can only win it by being sequenced first.

**2. The relatch happens inside the failing transaction, so the window can be closed atomically.** `OrderRouter.sol:324-329` calls `handleFailedProtectionAttempt` while finalizing the terminal attempt, and that sets `Latched` at `PositionProtectionBook.sol:663-666`. A caller that executes the attempt therefore observes `Latched` with zero pending orders inside its own transaction and consumes it immediately. `Triggered → Latched → Triggered` completes within one transaction, so no block boundary ever shows `Latched`. This is the whole mechanism: not a race the owner sometimes loses, but a window that never opens.

**3. Both halves of the attack are permissionless.** `executeOrder` and `executeOrderBatch` carry no caller restriction, so the attacker does not need to wait for an honest keeper to process the doomed attempt — it drives the failure itself and consumes the resulting window in the same transaction. If it skips a window the attempt expires, and the permissionless expiry path relatches the protection in the same way, so the attacker must act once per window to hold the lock. That is the cadence the PoC pays for.

**4. Every retry reuses the stored limits.** The attempt is rebuilt from the protection's retained close limits, so a cause that persists fails identically on attempt 1 and attempt 1,496. Nothing in the retry path escalates, loosens, or gives up.

**5. The other exits being shut is intended, and is not the claim.** `_cancel` requires `PendingOpen` or `Armed`, `cancelPositionProtectionAndCommitClose` and `replacePositionProtection` require `Armed`, and `commitOrder` is blocked by `ProtectionActive`. The project documents all of this: triggering is irreversible by design, and the disposition of MED-30 states explicitly that pending-open, `Triggered` and `Latched` records cannot use the atomic manual exit. I am not asking for any of that to change. The point is the consequence: because every other door is intentionally shut, the amend is not merely the recommended remedy, it is the only one — and it is the one that can be starved.

**6. The attacker can create the failing condition and hold it.** The close action charge includes signed VPI, and the evaluator compares it to the victim's stored `maxActionChargeAssets`. `maxSkewRatio` is enforced on opens only, so an attacker may sit at up to 0.4 × depth of opposite skew indefinitely, and closing the victim's position raises the victim's charge. The attacker also triggers the protection itself and collects the trigger bounty, so the opening move is mildly income-positive. The starvation does not depend on this step: whenever a close fails on the owner's own notional or gross-debit ceilings because the price itself moved, the same lock applies with no attacker position at all.

## Attack path

Victim: a LONG of 100 lots — 12,000 USDG notional at 1.20, 1,000 USDG margin — with a stop-loss at 1.21 and ordinary finite close limits (notional +2%, gross debit = loss at the stop +1%, action charge 25 bp = 30.25 USDG, explicit fees 10 bp). The SDK requires explicit close limits on every protection, so this is the normal shape of a protected position. With no attacker present the stop close at 1.212 costs 4.13 USDG and executes at the first attempt.

1. The basket reaches 1.212 — a 1% move from entry, 0.17% past the stop. In one block a fresh attacker account holding 25,000 USDG deposits, commits a SHORT of 3,350 lots (406,020 USDG notional, inside the skew cap), and calls `triggerPositionProtection` for the victim, collecting the trigger bounty. The attempt is queued behind the attacker's own open in FIFO.
2. One griefer transaction — a contract deployed by a fresh EOA with no relationship to the protocol — calls `executeOrderBatch`, so the attacker's SHORT executes and the victim's close attempt then fails with `ConstraintViolation(ActionCharge, 53.29 USDG > 30.25 USDG)` and relatches; in the same transaction it calls `retryPositionProtectionClose`, queueing a fresh attempt.
3. For each new attempt the griefer repeats `executeOrder(attempt)` + `retryPositionProtectionClose` in one transaction, once per execution window, or at commit + 1 second against an owner actively trying to escape.
4. At every transaction boundary the victim's exits revert: the amend with `ProtectionNotLatched`; cancel, cancel-and-close and replace with `ProtectionNotArmed`; `commitOrder` with `ProtectionActive`. Position margin cannot be withdrawn.
5. The lock lasts as long as the griefer keeps paying gas. If the market keeps moving against the victim the position is liquidated while locked, the protection is marked `Liquidated`, and the griefer can collect the liquidation bounty too.

## Impact

All figures are measured by the PoC at release parameters.

**What is frozen.** For the whole lock the victim cannot close, reduce, cancel, replace or amend anything. Its 982.48 USDG of position margin cannot be withdrawn — the clearinghouse rejects it on the free-equity / open-position guard — and stays exposed to price moves. `addMargin` and withdrawal of free, non-margin settlement still work, which only lets the victim postpone liquidation.

**Measured results.**

- Owner that does not process its own attempt: locked for **86,711 seconds (over 24 hours)** across 1,496 griefer transactions, with every documented exit reverting after every one. After 24 hours the close still fails on the same constraint, 53.30 USDG against the 30.25 USDG ceiling.
- Owner that fights back with a batching contract wallet sending `executeOrder + amend + retry` twice per second: locked for **3,660 seconds**, with all 7,320 escape batches reverting — the same-block one on `OrderNotQueueHead`, the next-block one because the fresh attempt is not yet executable.
- Market drifting against the victim at +0.1% per minute: locked **4,061 seconds**, then liquidated. The victim ends with 509.08 USDG where an executed stop would have left 1,369.95 USDG — **859.87 USDG of damage**, of which **810.72 USDG** is attributable to the starvation rather than to the attacker's skew, measured against a same-state control in which the owner is allowed its one amend. The griefer also collects the 6 USDG liquidation bounty.
- Control without an attacker: the stop executes at 1.212 at the first attempt, well inside every stored limit.
- Control with the attacker's skew but no griefer: the first attempt fails, the owner gets its `Latched` window, amends, and exits on the retry. This isolates the starvation, rather than the skew, as the cause of the lock.

**Attacker cost.** The skew position costs about 334 USDG for 24 hours — a one-off 8 bp round-trip fee plus roughly 9 USDG per day of carry. Griefer transactions run about 1.25M gas each, roughly 3 USD per hour at the 58-second cadence, or about 166 USD per hour at the 1-second cadence used against a responsive owner, assuming 0.01 gwei and 3,000 USD per ETH. There is no attacker revenue beyond the trigger bounty and a possible liquidation bounty, which is why I am filing this as griefing.

## Attacker requirements and limitations

- **No special access.** A fresh account with about 25,000 USDG. No privileged role, no oracle manipulation, no protocol impersonation, no keeper status — and per the program's own scope, keepers hold no special privileges and harm caused by any caller acting as a keeper is in scope.
- **Victim precondition.** The victim's action-charge ceiling must sit below the charge the attacker can induce, roughly the VPI factor times the induced skew ratio plus the execution fee — about 44 bp of close notional in the PoC, 53.29 USDG on 12,120 USDG. Ceilings around 0.5% or higher, and unlimited ceilings, are immune to the skew route.
- **An informed owner can beat the cheap cadence.** An owner who knows to call the permissionless `executeOrder` on its own pending attempt and then amend immediately does recover the window, either in one batched transaction from a contract wallet or in back-to-back transactions. The PoC includes that escape as a passing test rather than hiding it. The 1-second cadence closes that gap only if the griefer is sequenced first in every second, which a local test cannot prove; on an FCFS sequencer this is realistic but it is an assumption, not a result. What keeps the limitation narrow in practice is that the SDK's amend builder is a single call and a UI polling the status always sees `Triggered`, so an ordinary user has no prepared escape and no reason to suspect one is needed.
- **Other orders in the queue.** When other users' orders sit behind the attempt, honest keepers process it in FIFO order and briefly reopen the window without retrying. The griefer then needs only a cheap retry-only backrun after each relatch, about 0.67M gas, which is again an ordering assumption rather than a guarantee.
- **What stays available to the victim.** The position stays live; `addMargin` and free-settlement withdrawal work. The position margin is locked and the position cannot be reduced or closed.

## Relationship to known issues, prior findings and documented limitations

I would rather put all of the overlap in front of triage myself than have it found later. I checked the prior audit report, the repository scan report, the perps `SECURITY.md` limitations and the upstream fix history before writing this.

**MED-9 — this is the fix being shown incomplete, and the starvation is a product of that fix.** MED-9 ("Keepers can retry protection closes without the trader's financial limits") reported that internally generated protection closes carried maxed-out financial bounds. Its disposition says the protection now retains the trader's ceilings and copies them into the initial close and every permissionless retry, that only the owner may amend the retained ceilings while the protection is `Latched`, that keepers cannot relax them, and that tight limits may prevent execution until conditions improve or the owner amends them. Two things follow. First, the failure mode this report exploits — a protection close that fails on the owner's own stored limits and relatches — exists *because* of that fix; before it, the bounds were maxed and such a close did not fail that way. Second, the remedy the fix introduced, named in its own disposition as the thing that makes tight limits survivable, can be denied indefinitely by an unprivileged third party. That is both halves of the program's carve-out for fixed findings: the fix at this commit is incomplete, and it introduced a new reachable state.

**MED-30 (open) — adjacent, and deliberately not what I am claiming.** MED-30 concerns an *armed* protection during frozen markets and was addressed by adding the atomic manual exit for `Armed` records; its disposition states that pending-open, `Triggered` and `Latched` records cannot use that action, and the finding itself notes that `ProtectionActive` blocks any new manual close while a protection is active. So the fact that a triggered owner has no cancel path and no manual close is documented and acknowledged, and I am not reporting it. My claim is specifically that the one remedy `Triggered`/`Latched` records *do* have is consumable by a third party. MED-30 and MED-9 are complementary dispositions — the manual exit for `Armed`, the amend for `Latched` — and this report shows the second one does not hold up.

**MED-15 (fixed) — same shape of outcome, unrelated cause.** MED-15 described a triggered SHORT protection that could never execute because synthesized closes used `targetPrice = 1` against a zero oracle execution price, so every attempt terminalized as `Slippage` and relatched. Its impact paragraph reads much like mine — the close repeatedly fails, the protection stays latched instead of executing, the user stays exposed. The root cause is entirely different, it was fixed, and its accepted residual limitation is temporary unavailability when no positive valid quote exists. Here the close is perfectly executable the moment the owner amends; what is unavailable is the amend, and it is unavailable because someone is actively taking it.

**The repository scan report's trigger-bounty item (low) — keeper inaction, not adversarial pre-emption.** That item observes that the trigger bounty is paid on trigger while the queued close and every retry carry zero execution bounty, so a caller can take payment and abandon the flow, leaving the protection `Triggered` or later `Latched`, not cancellable through the pre-trigger path, with the position open past its threshold and exposed to further loss or liquidation. Its own evidence cites `PositionProtectionBook.sol:297` for the cancellation check, which sits at `:408-419` at this commit, and the report contains no reference to the amend entrypoint at all — it was produced against code that had no amend lane. So it describes the absence of an incentive to finish the close, and better keeper incentives fix it. This report describes an active party pre-empting the owner's remedy, where better keeper incentives make matters worse, because every honest execution hands the griefer a fresh relatch to consume.

**The excluded `SECURITY.md` limitations do not cover it, and the attack inverts their detection assumption.** The Known Limitations section on the router and keepers — which the program excludes — states that failed protection-attempt cleanup is deliberately unpaid, that each retry creates a fresh zero-bounty close record, that liveness depends on keepers processing the terminal attempt and separately appending a retry, and that operators must alert on a prolonged `Latched` state without a live child. Every one of those is about keepers *not* acting. This attack is the opposite: the terminal attempt is always processed, immediately, and a retry is always appended. The acknowledged monitoring signal is a prolonged `Latched` state *without a live child*, and under this attack the record is never `Latched` at a transaction boundary and always has a live child, so the prescribed alert cannot fire for the entire lock. The main protection section separately documents that retry is permissionless and that a recovered price cannot cancel the close mandate; it presents permissionless retry as what keeps the mandated close alive. Used as described here it does the reverse — it guarantees the close never completes while blocking the only action that would let it complete. A mechanism being deliberately permissionless is why this is reachable by anyone; it is not a reason the outcome is intended.

**The spec test does not cover it.** `PositionProtection.t.sol:983-990` checks that one third-party retry made *after* the owner has amended causes a second amend to revert. It never tests repeated pre-emption of the owner's *first* amend, which is the whole bug.

**No prior public disclosure of this root cause.** The MED-9 fix commit `8a2853f80bce067656c1fd0fcf2dadc5892e2aee` and its disposition PR #124 are public in the upstream repository. Neither the PR description, its commits, nor its review discussion mentions a retry pre-empting, racing or starving the owner's amend window, and neither proposes a cooldown or an owner grace period. The upstream artifacts disclose the amend's existence, not the defect in it.

**Parameters are not the cause.** Release parameters only bound how much VPI an attacker can induce, and therefore which victims the skew route reaches. The starvation is a state-machine flaw that holds at any parameter set, and it also reaches victims whose close fails for reasons no attacker created.

## Recommended fix

Give the owner an exit that a third party cannot consume. Any one of these is sufficient:

1. **Let the amend work against a live attempt.** Accept the `Triggered` state when the single pending order is the protection's own close attempt, cancelling that attempt and applying the new limits atomically. This is the smallest change that honours the entrypoint's documented contract, because it stops requiring the owner to win a race.

```solidity
// PositionProtectionBook.amendLatchedPositionProtectionCloseLimits
PositionProtectionTypes.PositionProtectionStatus status = protection.status;
if (status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
    // the only permitted pending order is this protection's own child
    if (protection.linkedOrderId == 0) revert OrderRouter__ProtectionNotLatched();
    IPositionProtectionRouterHost(ROUTER).cancelProtectionCloseAttempt(protection.linkedOrderId);
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
} else if (status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
_setCloseLimits(protection, limits);
```

2. **Reserve the window.** Record the relatch timestamp and let only the owner call `retryPositionProtectionClose` for a grace period after each relatch. One execution window is enough, and it costs honest keepers at most one window of liveness per failure — a cost the project already accepts implicitly, since failed-attempt cleanup is unpaid and retry timing is already nobody's obligation.

```solidity
// retryPositionProtectionClose
if (msg.sender != protection.account && block.timestamp < protection.latchedAt + OWNER_RETRY_GRACE) {
    revert OrderRouter__ProtectionOwnerGrace();
}
```

3. **Let the owner cancel out of `Latched` or `Triggered`**, cancelling any linked attempt, so the owner can always fall back to an ordinary close. This one changes the documented irreversibility of a trigger, so it is the most invasive of the three and I would not choose it first.

Option 2 is the most robust on its own, because it also protects owners whose close fails for reasons no attacker created and who are not watching the chain at all. Option 1 is the most faithful to the documented behaviour of the function. Together they close both the atomic starvation and the sequencing-race variant, and neither changes anything in normal operation: with no griefer present the owner reaches the same amend it reaches today.

Independently of the chosen fix: because `retryPositionProtectionClose` is permissionless, free and unlimited in frequency, nothing currently bounds how often one protection can be re-queued. A per-protection cooldown is cheap and closes the whole class of attacks that depend on re-queuing at machine speed.

## Proof of concept

Two files. `ProtectionAmendStarvation.t.sol` holds the attack, the two controls, the honest-flow and responsive-owner variants, the limitation test, and the two unprivileged helper contracts — the griefer, and the batching owner wallet used only to demonstrate the escape route. `PocBase.sol` is the shared fixture: it reproduces the release deployment on top of the repository's own `BasePerpTest`, applies the release wrapper values through the ordinary owner timelocks at deployment, and asserts every release value through live getters at the end of each test.

On the program's PoC rules: this runs in the repository's own Foundry suite at the commit in scope, no in-scope contract is modified, and the only mocked components are Pyth and USDG. The fixture inherits the repository's `BasePerpTest`, which wires the production `OrderRouter` bytecode through its `LegacyOrderRouterHarness` test wrapper and uses a two-feed mock basket in place of the six FX feeds; both are the repository's own test infrastructure rather than changes to in-scope code, and the attack uses only the production V3 commit path. Every attack step is a public entrypoint called by a fresh address. No protocol contract is impersonated, there is no `vm.store`, no `vm.mockCall` on protocol contracts, and no privileged action after deployment. `vm.snapshotState` and `vm.revertToState` are used solely to compute the no-attack and no-griefer controls from the identical pre-attack state inside the same test.

Place both files in a PoC directory under the perps test tree and run them from the repository root at commit `a82d4fe`:

```bash
mkdir -p packages/perps/test/perps/poc
cp PocBase.sol ProtectionAmendStarvation.t.sol packages/perps/test/perps/poc/
FOUNDRY_PROFILE=quick forge test --root packages/perps \
  --match-path test/perps/poc/ProtectionAmendStarvationPoc.t.sol -vvv
```

Expected output, trimmed to the lines that matter:

```
Ran 7 tests for test/perps/poc/ProtectionAmendStarvation.t.sol:ProtectionAmendStarvationTest
[PASS] test_control_noAttacker_protectionClosesAtTrigger()
  controlA action charge assessed (USDG atoms) 4128002
[PASS] test_control_attackerSkew_noGriefer_ownerAmendsAndExits()
  controlB failing constraint: ActionCharge actual (USDG atoms) 53285479
  controlB failing constraint: ActionCharge limit  (USDG atoms) 30250000
[PASS] test_protlock_attackerInduced_lock24h()
  owner exits all reverted for >= 1h; cycles so far 64
  lock duration (s) 86711
  griefer cycles (txs) 1496
  victim position margin frozen (USDG atoms) after 24h 982480002
  after 24h: ActionCharge actual (USDG atoms) 53301740
[PASS] test_protlock_responsiveOwner_1sCadence_1h()
  fast cadence: lock duration (s) 3660
  fast cadence: owner escape batches tried, all reverted 7320
[PASS] test_protlock_attackerInduced_driftToLiquidation()
  lock duration until liquidation (s) 4061
  victim damage vs control (USDG atoms) 859872778
  griefer-attributable damage vs control B (USDG atoms) 810715301
[PASS] test_protlock_liveOrderFlow_retryBackrun()
[PASS] test_limitation_batchingOwnerWalletEscapesIfFirst()
Suite result: ok. 7 passed; 0 failed; 0 skipped
```

One note on measurement: the 24-hour and 1-hour tests pause gas metering after the first metered hour, or the first 240 transactions, because 1,496 transactions of about 1.25M gas exceed Foundry's per-test gas cap. Gas is therefore metered and reported for a representative prefix and extrapolated. The Router only requires `gasleft()` above a fixed reserve, so paused metering does not change behaviour, and the lock assertions run under both.

## Proof of Concept

### ProtectionAmendStarvationPoc.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Self-contained PoC: permissionless retryPositionProtectionClose starves the owner's only
// post-trigger exit, amendLatchedPositionProtectionCloseLimits. Place at
// packages/perps/test/perps/poc/ProtectionAmendStarvationPoc.t.sol and run:
//   FOUNDRY_PROFILE=quick forge test --root packages/perps \
//     --match-path test/perps/poc/ProtectionAmendStarvationPoc.t.sol -vvv
// Code comments and fixture assertion messages were stripped to fit the 65,536-character
// submission field; each test is described in the report body.

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {
  uint256 internal constant REL_VPI_FACTOR = 0.01e18;
  uint256 internal constant REL_MAX_SKEW_RATIO = 0.4e18;
  uint256 internal constant REL_MAINT_MARGIN_BPS = 10;
  uint256 internal constant REL_INIT_MARGIN_BPS = 20;
  uint256 internal constant REL_FAD_MARGIN_BPS = 300;
  uint256 internal constant REL_BASE_CARRY_BPS = 500;
  uint256 internal constant REL_MIN_BOUNTY_ASSETS = 1e6;
  uint256 internal constant REL_BOUNTY_BPS = 10;
  uint256 internal constant REL_KEEPER_SHARE_BPS = 5000;
  uint256 internal constant REL_PROTOCOL_SHARE_BPS = 0;
  uint256 internal constant REL_EXECUTION_FEE_BPS = 4;
  uint256 internal constant REL_SETTLEMENT_BUFFER_BPS = 25;
  uint256 internal constant REL_FROZEN_CLOSE_SPREAD_BPS = 50;
  uint256 internal constant REL_CAP_PRICE = 2e8;
  uint256 internal constant REL_SIZE_QUANTUM = 1e20;
  uint256 internal constant REL_FAD_RUNWAY_SECONDS = 1 hours;
  uint256 internal constant REL_FAD_MAX_STALENESS = 3 days;
  uint256 internal constant REL_ENGINE_MARK_STALENESS = 60;
  uint256 internal constant REL_BASKET_MAX_CONF_RATIO_BPS = 10;
  uint256 internal constant REL_ADVERSE_CONF_MULTIPLIER_BPS = 2500;
  uint256 internal constant REL_ORDER_STALENESS = 60;
  uint256 internal constant REL_LIQUIDATION_STALENESS = 15;
  uint256 internal constant REL_ORDER_SETTLEMENT_WINDOW = 15;
  uint256 internal constant REL_MAX_COMPONENT_DIVERGENCE = 5;
  uint256 internal constant REL_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
  uint256 internal constant REL_MAX_PENDING_ORDERS = 5;
  uint256 internal constant REL_PROTECTION_TRIGGER_BOUNTY_ASSETS = 200_000;
  uint256 internal constant REL_OPEN_BOUNTY_BPS = 1;
  uint256 internal constant REL_MIN_OPEN_BOUNTY_ASSETS = 10_000;
  uint256 internal constant REL_MAX_OPEN_BOUNTY_ASSETS = 200_000;
  uint256 internal constant REL_MAX_EXECUTION_WINDOW_SECONDS = 60;
  uint256 internal constant REL_MIN_ENGINE_GAS = 600_000;
  uint256 internal constant REL_MAX_PRUNE_ORDERS = 64;
  uint256 internal constant REL_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
  uint256 internal constant REL_MAX_SENIOR_SHARE_BPS = 8000;
  uint256 internal constant REL_SENIOR_RATE_BPS = 800;
  uint256 internal constant REL_POOL_MARK_STALENESS = 60;
  uint256 internal constant REL_SENIOR_FROZEN_LP_FEE_BPS = 25;
  uint256 internal constant REL_JUNIOR_FROZEN_LP_FEE_BPS = 75;
  uint256 internal constant REL_JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
  uint256 internal constant REL_SEED_ASSETS = 1e6;
  uint256 internal constant POC_ALL_EXECUTION_MODES = 7;
  address internal pocKeeper;
  address internal pocJuniorLp;
  address internal pocSeniorLp;
  uint256 internal pocPrice;
  uint256 private _pocClientNonce;
  mapping(address => mapping(bool => uint256)) internal pocLastDepositRequest;
  mapping(address => mapping(bool => uint256)) internal pocLastRedeemRequest;
  function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
    return CfdTypes.RiskParams({
      vpiFactor: REL_VPI_FACTOR,
      maxSkewRatio: REL_MAX_SKEW_RATIO,
      maintMarginBps: REL_MAINT_MARGIN_BPS,
      initMarginBps: REL_INIT_MARGIN_BPS,
      fadMarginBps: REL_FAD_MARGIN_BPS,
      baseCarryBps: REL_BASE_CARRY_BPS,
      minBountyAssets: REL_MIN_BOUNTY_ASSETS,
      bountyBps: REL_BOUNTY_BPS,
      keeperShareBps: REL_KEEPER_SHARE_BPS,
      protocolShareBps: REL_PROTOCOL_SHARE_BPS
    });
  }
  function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
    return 0;
  }
  function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
    return 0;
  }
  function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
    return REL_SEED_ASSETS;
  }
  function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
    return REL_SEED_ASSETS;
  }
  function _pocInitialJuniorLpAssets() internal pure virtual returns (uint256) {
    return 1_000_000e6;
  }
  function _pocInitialSeniorLpAssets() internal pure virtual returns (uint256) {
    return 0;
  }
  function _pocDefaultPrice() internal pure virtual returns (uint256) {
    return 1.2e8;
  }
  function _pocPythFeeWei() internal pure virtual returns (uint256) {
    return 1;
  }
  function setUp() public virtual override {
    super.setUp();
    assertEq(pool.seniorPrincipal(), REL_SEED_ASSETS);
    assertEq(pool.juniorPrincipal(), REL_SEED_ASSETS);
    assertTrue(pool.isTradingActive());
    pocKeeper = _pocActor("pocKeeper");
    pocJuniorLp = _pocActor("pocJuniorLp");
    pocSeniorLp = _pocActor("pocSeniorLp");
    _pocApplyReleaseConfig();
    _pocWarpToWeekdayHourUtc(9);
    baseMockPyth.setFee(_pocPythFeeWei());
    uint256 price = _pocDefaultPrice();
    _setPrice(price);
    _updateMark(pocKeeper, price);
    uint256 juniorAssets = _pocInitialJuniorLpAssets();
    if (juniorAssets > 0) {
      uint256 requestId = _lpRequestDeposit(pocJuniorLp, false, juniorAssets);
      _warpToLpEpoch(requestId);
      _settleLpEpochPublic(pocKeeper);
      assertGt(_lpClaimShares(pocJuniorLp, false), 0);
    }
    uint256 seniorAssets = _pocInitialSeniorLpAssets();
    if (seniorAssets > 0) {
      uint256 requestId = _lpRequestDeposit(pocSeniorLp, true, seniorAssets);
      _warpToLpEpoch(requestId);
      _settleLpEpochPublic(pocKeeper);
      assertGt(_lpClaimShares(pocSeniorLp, true), 0);
    }
    _setPrice(price);
    _updateMark(pocKeeper, price);
    assertFalse(_isFadOrFrozen());
    _assertReleaseParams();
  }
  function _pocApplyReleaseConfig() internal {
    uint256 readyAt = vm.getBlockTimestamp();
    bool poolChange;
    IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
    if (
      poolConfig.maxSeniorExposureAssets != REL_MAX_SENIOR_EXPOSURE_ASSETS
        || poolConfig.maxSeniorShareBps != REL_MAX_SENIOR_SHARE_BPS
        || poolConfig.seniorRateBps != REL_SENIOR_RATE_BPS
        || poolConfig.markStalenessLimit != REL_POOL_MARK_STALENESS
        || poolConfig.seniorFrozenLpFeeBps != REL_SENIOR_FROZEN_LP_FEE_BPS
        || poolConfig.juniorFrozenLpFeeBps != REL_JUNIOR_FROZEN_LP_FEE_BPS
    ) {
      poolConfig.maxSeniorExposureAssets = REL_MAX_SENIOR_EXPOSURE_ASSETS;
      poolConfig.maxSeniorShareBps = REL_MAX_SENIOR_SHARE_BPS;
      poolConfig.seniorRateBps = REL_SENIOR_RATE_BPS;
      poolConfig.markStalenessLimit = REL_POOL_MARK_STALENESS;
      poolConfig.seniorFrozenLpFeeBps = REL_SENIOR_FROZEN_LP_FEE_BPS;
      poolConfig.juniorFrozenLpFeeBps = REL_JUNIOR_FROZEN_LP_FEE_BPS;
      pool.proposePoolConfig(poolConfig);
      readyAt = _pocMax(readyAt, pool.poolConfigActivationTime());
      poolChange = true;
    }
    bool routerChange;
    IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
    if (
      routerConfig.minOpenNotionalAssets != REL_MIN_OPEN_NOTIONAL_ASSETS
        || routerConfig.adverseConfidenceMultiplierBps != REL_ADVERSE_CONF_MULTIPLIER_BPS
        || routerConfig.basketMaxConfidenceRatioBps != REL_BASKET_MAX_CONF_RATIO_BPS
        || routerConfig.maxPendingOrders != REL_MAX_PENDING_ORDERS
        || routerConfig.positionProtectionTriggerBountyAssets != REL_PROTECTION_TRIGGER_BOUNTY_ASSETS
    ) {
      routerConfig.minOpenNotionalAssets = REL_MIN_OPEN_NOTIONAL_ASSETS;
      routerConfig.adverseConfidenceMultiplierBps = REL_ADVERSE_CONF_MULTIPLIER_BPS;
      routerConfig.basketMaxConfidenceRatioBps = REL_BASKET_MAX_CONF_RATIO_BPS;
      routerConfig.maxPendingOrders = REL_MAX_PENDING_ORDERS;
      routerConfig.positionProtectionTriggerBountyAssets = REL_PROTECTION_TRIGGER_BOUNTY_ASSETS;
      routerAdmin.proposeRouterConfig(routerConfig);
      readyAt = _pocMax(readyAt, routerAdmin.routerConfigActivationTime());
      routerChange = true;
    }
    bool riskChange;
    ICfdEngineAdminHost.EngineRiskConfig memory riskConfig = _engineRiskConfig();
    if (
      riskConfig.executionFeeBps != REL_EXECUTION_FEE_BPS
        || riskConfig.settlementBufferBps != REL_SETTLEMENT_BUFFER_BPS
        || riskConfig.frozenCloseSpreadBps != REL_FROZEN_CLOSE_SPREAD_BPS
        || keccak256(abi.encode(riskConfig.riskParams)) != keccak256(abi.encode(_riskParams()))
    ) {
      riskConfig.riskParams = _riskParams();
      riskConfig.executionFeeBps = REL_EXECUTION_FEE_BPS;
      riskConfig.settlementBufferBps = REL_SETTLEMENT_BUFFER_BPS;
      riskConfig.frozenCloseSpreadBps = REL_FROZEN_CLOSE_SPREAD_BPS;
      engineAdmin.proposeRiskConfig(riskConfig);
      readyAt = _pocMax(readyAt, engineAdmin.riskConfigActivationTime());
      riskChange = true;
    }
    bool calendarChange;
    if (engine.fadRunwaySeconds() != REL_FAD_RUNWAY_SECONDS) {
      ICfdEngineAdminHost.EngineCalendarConfig memory calendarConfig;
      calendarConfig.fadDayTimestamps = new uint256[](0);
      calendarConfig.fadRunwaySeconds = REL_FAD_RUNWAY_SECONDS;
      engineAdmin.proposeCalendarConfig(calendarConfig);
      readyAt = _pocMax(readyAt, engineAdmin.calendarConfigActivationTime());
      calendarChange = true;
    }
    bool maintenanceFeeChange;
    if (juniorVault.maintenanceFeeAprBps() != REL_JUNIOR_MAINTENANCE_FEE_APR_BPS) {
      juniorVault.proposeMaintenanceFeeConfig(REL_JUNIOR_MAINTENANCE_FEE_APR_BPS, engine.protocolTreasury());
      readyAt = _pocMax(readyAt, juniorVault.maintenanceFeeConfigActivationTime());
      maintenanceFeeChange = true;
    }
    if (readyAt > vm.getBlockTimestamp()) {
      vm.warp(readyAt + 1);
      vm.roll(vm.getBlockNumber() + 1);
    }
    if (poolChange) {
      pool.finalizePoolConfig();
    }
    if (routerChange) {
      routerAdmin.finalizeRouterConfig();
    }
    if (riskChange) {
      engineAdmin.finalizeRiskConfig();
    }
    if (calendarChange) {
      engineAdmin.finalizeCalendarConfig();
    }
    if (maintenanceFeeChange) {
      juniorVault.finalizeMaintenanceFeeConfig();
    }
  }
  function _pocWarpToWeekdayHourUtc(uint256 hourUtc) internal {
    uint256 nowTs = vm.getBlockTimestamp();
    uint256 day = nowTs / 1 days;
    uint256 target = day * 1 days + hourUtc * 1 hours;
    if (target < nowTs) {
      day += 1;
      target += 1 days;
    }
    while (((day + 4) % 7) == 0 || ((day + 4) % 7) >= 5) {
      day += 1;
      target += 1 days;
    }
    _pocWarpTo(target);
    vm.roll(vm.getBlockNumber() + 1);
  }
  function _assertReleaseParams() internal view {
    CfdTypes.RiskParams memory rp = ICfdEngineRiskParamsView(address(engine)).riskParams();
    assertEq(rp.vpiFactor, REL_VPI_FACTOR);
    assertEq(rp.maxSkewRatio, REL_MAX_SKEW_RATIO);
    assertEq(rp.maintMarginBps, REL_MAINT_MARGIN_BPS);
    assertEq(rp.initMarginBps, REL_INIT_MARGIN_BPS);
    assertEq(rp.fadMarginBps, REL_FAD_MARGIN_BPS);
    assertEq(rp.baseCarryBps, REL_BASE_CARRY_BPS);
    assertEq(rp.minBountyAssets, REL_MIN_BOUNTY_ASSETS);
    assertEq(rp.bountyBps, REL_BOUNTY_BPS);
    assertEq(rp.keeperShareBps, REL_KEEPER_SHARE_BPS);
    assertEq(rp.protocolShareBps, REL_PROTOCOL_SHARE_BPS);
    assertEq(engine.executionFeeBps(), REL_EXECUTION_FEE_BPS);
    assertEq(engine.settlementBufferBps(), REL_SETTLEMENT_BUFFER_BPS);
    assertEq(engine.frozenCloseSpreadBps(), REL_FROZEN_CLOSE_SPREAD_BPS);
    assertEq(engine.fadRunwaySeconds(), REL_FAD_RUNWAY_SECONDS);
    assertEq(engine.fadMaxStaleness(), REL_FAD_MAX_STALENESS);
    assertEq(engine.engineMarkStalenessLimit(), REL_ENGINE_MARK_STALENESS);
    assertEq(engine.CAP_PRICE(), REL_CAP_PRICE);
    assertEq(uint256(terminalNavBook.CAP_PRICE()), REL_CAP_PRICE);
    assertEq(terminalNavBook.SIZE_QUANTUM(), REL_SIZE_QUANTUM);
    assertEq(CfdTypes.SIZE_QUANTUM, REL_SIZE_QUANTUM);
    assertEq(address(engine.orderRouter()), address(router));
    assertEq(address(engine.pool()), address(pool));
    assertEq(router.minOpenNotionalAssets(), REL_MIN_OPEN_NOTIONAL_ASSETS);
    assertEq(router.maxPendingOrders(), REL_MAX_PENDING_ORDERS);
    assertEq(
      router.positionProtectionTriggerBountyAssets(),
      REL_PROTECTION_TRIGGER_BOUNTY_ASSETS,
      "rel: positionProtectionTriggerBountyAssets"
    );
    assertEq(router.openOrderExecutionBountyBps(), REL_OPEN_BOUNTY_BPS);
    assertEq(
      router.minOpenOrderExecutionBountyAssets(), REL_MIN_OPEN_BOUNTY_ASSETS, "minOpenOrderExecutionBounty"
    );
    assertEq(
      router.maxOpenOrderExecutionBountyAssets(), REL_MAX_OPEN_BOUNTY_ASSETS, "maxOpenOrderExecutionBounty"
    );
    assertEq(router.maxExecutionWindowSeconds(), REL_MAX_EXECUTION_WINDOW_SECONDS);
    assertEq(router.minEngineGas(), REL_MIN_ENGINE_GAS);
    assertEq(router.maxPruneOrdersPerCall(), REL_MAX_PRUNE_ORDERS);
    IPletherOracle oracle = router.pletherOracle();
    assertEq(address(oracle), address(pletherOracle));
    assertEq(oracle.adverseConfidenceMultiplierBps(), REL_ADVERSE_CONF_MULTIPLIER_BPS);
    assertEq(oracle.basketMaxConfidenceRatioBps(), REL_BASKET_MAX_CONF_RATIO_BPS);
    assertEq(oracle.orderExecutionStalenessLimit(), REL_ORDER_STALENESS);
    assertEq(oracle.liquidationStalenessLimit(), REL_LIQUIDATION_STALENESS);
    assertEq(oracle.orderSettlementWindow(), REL_ORDER_SETTLEMENT_WINDOW);
    assertEq(
      oracle.maxComponentPublishTimeDivergence(), REL_MAX_COMPONENT_DIVERGENCE, "component divergence"
    );
    assertEq(baseMockPyth.mockFee(), _pocPythFeeWei());
    assertEq(pool.maxSeniorExposureAssets(), REL_MAX_SENIOR_EXPOSURE_ASSETS);
    assertEq(pool.maxSeniorShareBps(), REL_MAX_SENIOR_SHARE_BPS);
    assertEq(pool.seniorRateBps(), REL_SENIOR_RATE_BPS);
    assertEq(pool.markStalenessLimit(), REL_POOL_MARK_STALENESS);
    assertEq(pool.seniorFrozenLpFeeBps(), REL_SENIOR_FROZEN_LP_FEE_BPS);
    assertEq(pool.juniorFrozenLpFeeBps(), REL_JUNIOR_FROZEN_LP_FEE_BPS);
    assertTrue(pool.isTradingActive());
    assertTrue(pool.isSeedLifecycleComplete());
    assertEq(juniorVault.maintenanceFeeAprBps(), REL_JUNIOR_MAINTENANCE_FEE_APR_BPS);
    assertEq(seniorVault.maintenanceFeeAprBps(), 0);
    assertEq(_initialJuniorSeedDeposit(), REL_SEED_ASSETS);
    assertEq(_initialSeniorSeedDeposit(), REL_SEED_ASSETS);
    assertEq(juniorVault.seedReceiver(), address(this));
    assertEq(seniorVault.seedReceiver(), address(this));
    assertGt(juniorVault.seedShareFloor(), 0);
    assertGt(seniorVault.seedShareFloor(), 0);
    assertEq(pool.poolConfigActivationTime(), 0);
    assertEq(routerAdmin.routerConfigActivationTime(), 0);
    assertEq(routerAdmin.oracleConfigActivationTime(), 0);
    assertEq(engineAdmin.riskConfigActivationTime(), 0);
    assertEq(engineAdmin.calendarConfigActivationTime(), 0);
    assertEq(engineAdmin.freshnessConfigActivationTime(), 0);
    assertEq(juniorVault.maintenanceFeeConfigActivationTime(), 0);
    assertFalse(routerAdmin.paused());
    assertEq(uint256(routerAdmin.riskOffOrderCutoff()), 0);
    assertFalse(pool.paused());
    assertFalse(pool.lpEpochSettlementPaused());
  }
  function _pocActor(string memory name) internal returns (address actor) {
    actor = makeAddr(name);
  }
  function _pocRequireActor(address actor) internal view {
    require(actor != address(0), "PocBase: zero actor");
    require(actor != address(this), "poc: owner");
    require(
      actor != address(engine) && actor != address(engineAdmin) && actor != address(engine.settlementSidecar())
        && actor != address(engine.planningReader()) && actor != address(terminalNavBook)
        && actor != address(pool) && actor != address(seniorVault) && actor != address(juniorVault)
        && actor != address(clearinghouse) && actor != address(router) && actor != address(routerAdmin)
        && actor != address(router.lifecycleBook()) && actor != address(router.positionProtectionBook())
        && actor != router.liquidationBatchSidecar() && actor != router.executionSidecar()
        && actor != address(pletherOracle) && actor != address(baseMockPyth)
        && actor != address(settlementToken) && actor != engine.protocolTreasury()
        && actor != address(pool.ENGINE_PROTOCOL_LENS()),
      "poc: protocol addr"
    );
  }
  function _pocEnsureEth(address actor, uint256 weiAmount) internal {
    if (actor.balance < weiAmount) {
      vm.deal(actor, weiAmount);
    }
  }
  function _pocMintUsdg(address actor, uint256 assets) internal {
    settlementToken.mint(actor, assets);
  }
  function _vaultOf(bool senior) internal view returns (TrancheVault) {
    return senior ? seniorVault : juniorVault;
  }
  function _lpRequestDeposit(address lp, bool senior, uint256 assets) internal returns (uint256 requestId) {
    _pocRequireActor(lp);
    TrancheVault vault = _vaultOf(senior);
    _pocMintUsdg(lp, assets);
    vm.startPrank(lp);
    settlementToken.approve(address(vault), assets);
    requestId = vault.requestDeposit(assets, lp, lp);
    vm.stopPrank();
    pocLastDepositRequest[lp][senior] = requestId;
  }
  function _lpClaimShares(address lp, bool senior) internal returns (uint256 shares) {
    return _lpClaimShares(lp, senior, pocLastDepositRequest[lp][senior]);
  }
  function _lpClaimShares(address lp, bool senior, uint256 requestId) internal returns (uint256 shares) {
    _pocRequireActor(lp);
    TrancheVault vault = _vaultOf(senior);
    uint256 assets = vault.claimableDepositRequest(requestId, lp);
    require(assets > 0, "PocBase: no claimable deposit");
    vm.prank(lp);
    shares = vault.claimDeposit(requestId, assets, lp, lp);
  }
  function _warpToLpEpoch(uint256 epochId) internal {
    uint256 start = pool.lpEpochStart(epochId);
    if (vm.getBlockTimestamp() < start) {
      _pocWarpTo(start);
      vm.roll(vm.getBlockNumber() + 1);
    }
  }
  function _settleLpEpochPublic(address keeper) internal returns (bool viaRouter) {
    _pocRequireActor(keeper);
    ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
    if (!snapshot.hasOpenPositions || engine.isOracleFrozen()) {
      vm.prank(keeper);
      pool.settleLpEpoch(0, 0);
      return false;
    }
    _pocTick();
    uint256 nowTs = vm.getBlockTimestamp();
    bytes[] memory data = _pythPayload(pocPrice, 0, nowTs, nowTs - 1);
    uint256 fee = router.pletherOracle().getUpdateFee(data);
    _pocEnsureEth(keeper, fee);
    vm.prank(keeper);
    router.settleLpEpoch{value: fee}(data);
    return true;
  }
  function _traderDeposit(address trader, uint256 assets) internal {
    _pocRequireActor(trader);
    _pocMintUsdg(trader, assets);
    vm.startPrank(trader);
    settlementToken.approve(address(clearinghouse), assets);
    clearinghouse.deposit(trader, assets);
    vm.stopPrank();
  }
  function _traderWithdraw(address trader, uint256 assets) internal {
    _pocRequireActor(trader);
    vm.prank(trader);
    clearinghouse.withdraw(trader, assets);
  }
  function _permissiveTargetPrice(CfdTypes.Side side, bool isClose) internal pure returns (uint256) {
    return ((side == CfdTypes.Side.LONG) == isClose) ? type(uint256).max : 1;
  }
  function _pocOrderRequest(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal returns (OrderV3Types.OrderRequest memory request) {
    uint256 window = router.maxExecutionWindowSeconds();
    request.clientOrderId = bytes32(++_pocClientNonce);
    request.side = side;
    request.sizeDelta = size;
    request.marginDelta = margin;
    request.targetPrice = _permissiveTargetPrice(side, isClose);
    request.isClose = isClose;
    request.bounds = OrderV3Types.ExecutionBounds({
      submitBy: SafeCast.toUint64(vm.getBlockTimestamp() + window),
      executionWindowSeconds: SafeCast.toUint32(window),
      allowedExecutionModes: uint8(POC_ALL_EXECUTION_MODES),
      expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
      maxExecutionBountyAssets: type(uint256).max,
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
  function _commitRequest(address trader, OrderV3Types.OrderRequest memory request) internal returns (uint64 orderId) {
    _pocRequireActor(trader);
    vm.prank(trader);
    orderId = OrderRouter(address(router)).commitOrder(request);
  }
  function _commitOpen(address trader, CfdTypes.Side side, uint256 size, uint256 margin) internal returns (uint64 orderId) {
    return _commitRequest(trader, _pocOrderRequest(side, size, margin, false));
  }
  function _commitClose(address trader, CfdTypes.Side side, uint256 size) internal returns (uint64 orderId) {
    return _commitRequest(trader, _pocOrderRequest(side, size, 0, true));
  }
  function _pocTick() internal {
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(vm.getBlockTimestamp() + 1);
  }
  function _pocWarpTo(uint256 timestamp) internal {
    require(timestamp >= vm.getBlockTimestamp(), "poc: time back");
    vm.warp(timestamp);
  }
  function _pocFeedRawPrice(uint256 i, uint256 basketPrice) internal view returns (uint256) {
    PletherOracle oracle = PletherOracle(address(router.pletherOracle()));
    require(!oracle.inversions(i), "poc: inverted");
    return (basketPrice * oracle.basePrices(i)) / 1e8;
  }
  function _pythPayload(uint256 basketPrice, uint64 basketConfidence, uint256 publishTime, uint256 prevPublishTime) internal view returns (bytes[] memory data) {
    bytes32[] memory ids = _basePythFeedIds();
    MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
    for (uint256 i; i < ids.length; ++i) {
      updates[i] = MockPyth.FeedUpdate(
        ids[i],
        MockPyth.MockPrice(
          SafeCast.toInt64(SafeCast.toInt256(_pocFeedRawPrice(i, basketPrice))),
          SafeCast.toUint64(_pocFeedRawPrice(i, basketConfidence)),
          -8,
          publishTime,
          prevPublishTime
        )
      );
    }
    data = new bytes[](1);
    data[0] = abi.encode(updates);
  }
  function _setPrice(uint256 price) internal {
    pocPrice = price;
    bytes32[] memory ids = _basePythFeedIds();
    uint256 nowTs = vm.getBlockTimestamp();
    for (uint256 i; i < ids.length; ++i) {
      baseMockPyth.setPrice(
        ids[i], SafeCast.toInt64(SafeCast.toInt256(_pocFeedRawPrice(i, price))), uint64(0), int32(-8), nowTs
      );
    }
  }
  function _updateMark(address keeper, uint256 price) internal {
    _updateMark(keeper, price, 0);
  }
  function _updateMark(address keeper, uint256 price, uint64 confidence) internal {
    _pocRequireActor(keeper);
    _pocTick();
    uint256 nowTs = vm.getBlockTimestamp();
    bytes[] memory data = _pythPayload(price, confidence, nowTs, nowTs - 1);
    uint256 fee = router.pletherOracle().getUpdateFee(data);
    _pocEnsureEth(keeper, fee);
    vm.prank(keeper);
    router.updateMarkPrice{value: fee}(data);
    pocPrice = price;
  }
  function _executeNext(address keeper, uint256 price) internal returns (OrderV3Types.ExecutionResult memory result) {
    require(router.globalTailOrderId() != 0, "PocBase: order queue empty");
    return _executeOrderPublic(keeper, router.nextExecuteId(), price, 0);
  }
  function _executeOrderPublic(address keeper, uint64 orderId, uint256 price, uint64 confidence) internal returns (OrderV3Types.ExecutionResult memory result) {
    _pocRequireActor(keeper);
    (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
    require(pending.orderId == orderId && orderId != 0, "PocBase: order is not pending");
    _pocTick();
    if (vm.getBlockNumber() <= pending.commitBlock) {
      vm.roll(uint256(pending.commitBlock) + 1);
    }
    if (vm.getBlockTimestamp() <= pending.commitTime) {
      vm.warp(uint256(pending.commitTime) + 1);
    }
    bytes[] memory data = _historicalPayload(price, confidence, pending.commitTime, pending.commitTime);
    uint256 fee = router.pletherOracle().getOrderExecutionFee(data);
    _pocEnsureEth(keeper, fee);
    vm.prank(keeper);
    result = router.executeOrder{value: fee}(orderId, data);
    pocPrice = price;
  }
  function _executeBatch(address keeper, uint64 maxOrderId, uint256 price) internal returns (OrderV3Types.BatchResult memory result) {
    _pocRequireActor(keeper);
    uint64 head = router.nextExecuteId();
    require(router.globalTailOrderId() != 0 && head != 0 && head <= maxOrderId, "PocBase: bad batch range");
    (IOrderRouterAccounting.PendingOrderView memory first,) = router.getPendingOrderView(head);
    (IOrderRouterAccounting.PendingOrderView memory last,) = router.getPendingOrderView(maxOrderId);
    require(first.orderId == head && last.orderId == maxOrderId, "PocBase: batch bounds not pending");
    _pocTick();
    if (vm.getBlockNumber() <= last.commitBlock) {
      vm.roll(uint256(last.commitBlock) + 1);
    }
    if (vm.getBlockTimestamp() <= last.commitTime) {
      vm.warp(uint256(last.commitTime) + 1);
    }
    bytes[] memory data = _historicalPayload(price, 0, first.commitTime, first.commitTime);
    uint256 fee = router.pletherOracle().getOrderExecutionFee(data) * (uint256(maxOrderId - head) + 1);
    _pocEnsureEth(keeper, fee);
    vm.prank(keeper);
    result = router.executeOrderBatch{value: fee}(maxOrderId, data);
    pocPrice = price;
  }
  function _historicalPayload(uint256 price, uint64 confidence, uint64 windowStartCommitTime, uint64 prevPublishTime) internal view returns (bytes[] memory) {
    uint256 nowTs = vm.getBlockTimestamp();
    uint256 deadline = uint256(windowStartCommitTime) + router.pletherOracle().orderSettlementWindow();
    uint256 publishTime = nowTs < deadline ? nowTs : deadline;
    return _pythPayload(price, confidence, publishTime, prevPublishTime);
  }
  function _liquidate(address keeper, address account, uint256 price) internal {
    _liquidate(keeper, account, price, 0);
  }
  function _liquidate(address keeper, address account, uint256 price, uint64 confidence) internal {
    _pocRequireActor(keeper);
    _pocTick();
    uint256 nowTs = vm.getBlockTimestamp();
    bytes[] memory data = _pythPayload(price, confidence, nowTs, nowTs - 1);
    uint256 fee = router.pletherOracle().getUpdateFee(data);
    _pocEnsureEth(keeper, fee);
    vm.prank(keeper);
    router.executeLiquidation{value: fee}(account, data);
    pocPrice = price;
  }
  function _freeSettlement(address account) internal view returns (uint256) {
    return clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets;
  }
  function _settlementOf(address account) internal view returns (uint256) {
    return clearinghouse.balanceAssets(account);
  }
  function _positionOf(address account) internal view returns (uint256 size, uint256 margin, CfdTypes.Side side) {
    (size, margin,,, side,,) = engine.positions(account);
  }
  function _positionSizeOf(address account) internal view returns (uint256 size) {
    (size,,,,,,) = engine.positions(account);
  }
  function _isFadOrFrozen() internal view returns (bool) {
    return engine.isFadWindow() || engine.isOracleFrozen();
  }
  function _pocMax(uint256 a, uint256 b) internal pure returns (uint256) {
    return a > b ? a : b;
  }
}

contract ProtLockGriefer {
  OrderRouter internal immutable ROUTER;
  IPositionProtectionActions internal immutable BOOK;
  address internal immutable OPERATOR;
  constructor(OrderRouter router_, IPositionProtectionActions book_, address operator_) {
    ROUTER = router_;
    BOOK = book_;
    OPERATOR = operator_;
  }
  function cycle(uint64 attemptId, uint64 protectionId, bytes[] calldata data) external payable returns (uint8 status, uint8 reason, uint64 nextAttemptId) {
    require(msg.sender == OPERATOR, "operator");
    OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder{value: msg.value}(attemptId, data);
    status = uint8(r.status);
    reason = uint8(r.terminalReason);
    nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
  }
  function cycleBatch(uint64 maxOrderId, uint64 protectionId, bytes[] calldata data) external payable returns (uint32 terminalCount, uint64 nextAttemptId) {
    require(msg.sender == OPERATOR, "operator");
    OrderV3Types.BatchResult memory r = ROUTER.executeOrderBatch{value: msg.value}(maxOrderId, data);
    terminalCount = r.terminalCount;
    nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
  }
  receive() external payable {}
}
contract OwnerBatchWallet {
  address internal immutable OWNER;
  constructor(address owner_) {
    OWNER = owner_;
  }
  function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata datas) external payable {
    require(msg.sender == OWNER, "owner");
    for (uint256 i; i < targets.length; ++i) {
      (bool ok, bytes memory ret) = targets[i].call{value: values[i]}(datas[i]);
      if (!ok) {
        assembly ("memory-safe") {
          revert(add(ret, 32), mload(ret))
        }
      }
    }
  }
  receive() external payable {}
}
contract ProtectionAmendStarvationTest is PocBase {
  IPositionProtectionActions internal book;
  IPositionProtectionViews internal views;
  IOrderLifecycleBook internal lifecycle;
  address internal victim;
  address internal attacker;
  address internal griefEoa;
  address internal keeper;
  ProtLockGriefer internal griefer;
  uint256 internal constant ENTRY = 1.2e8;
  uint256 internal constant SL = 1.21e8;
  uint256 internal constant TRIGGER_PRICE = 1.212e8;
  uint256 internal constant V_LOTS = 100;
  uint256 internal constant V_SIZE = V_LOTS * 1e20;
  uint256 internal constant V_DEPOSIT = 1_500e6;
  uint256 internal constant V_MARGIN = 1_000e6;
  uint256 internal constant A_LOTS = 3_350;
  uint256 internal constant A_SIZE = A_LOTS * 1e20;
  uint256 internal constant A_DEPOSIT = 25_000e6;
  uint256 internal constant A_MARGIN = 20_000e6;
  uint256 internal constant CYCLE_SECONDS = 58;
  function setUp() public override {
    super.setUp();
    book = IPositionProtectionActions(address(router.positionProtectionBook()));
    views = IPositionProtectionViews(address(router.positionProtectionBook()));
    lifecycle = IOrderLifecycleBook(address(router.lifecycleBook()));
    victim = _pocActor("victim");
    attacker = _pocActor("attacker");
    griefEoa = _pocActor("griefEoa");
    keeper = _pocActor("keeper");
    vm.prank(griefEoa);
    griefer = new ProtLockGriefer(OrderRouter(address(router)), book, griefEoa);
  }
  function _victimLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
    uint256 slNotional = V_LOTS * SL;
    l.maxExecutionNotionalAssets = (slNotional * 10_200) / 10_000;
    l.maxGrossAccountDebitAssets = (SL - ENTRY) * V_LOTS + slNotional / 100;
    l.maxActionChargeAssets = (slNotional * 25) / 10_000;
    l.maxExplicitFeesAssets = (slNotional * 10) / 10_000;
  }
  function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
    l.maxExecutionNotionalAssets = type(uint256).max;
    l.maxGrossAccountDebitAssets = type(uint256).max;
    l.maxActionChargeAssets = type(uint256).max;
    l.maxExplicitFeesAssets = type(uint256).max;
  }
  function _openAndProtect(address owner) internal returns (uint64 pid) {
    _traderDeposit(owner, V_DEPOSIT);
    _commitOpen(owner, CfdTypes.Side.LONG, V_SIZE, V_MARGIN);
    OrderV3Types.ExecutionResult memory r = _executeNext(keeper, ENTRY);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    _setPrice(ENTRY);
    _updateMark(keeper, ENTRY);
    PositionProtectionTypes.PositionProtectionParams memory params;
    params.stopLossTriggerPrice = SL;
    params.closeLimits = _victimLimits();
    vm.prank(owner);
    pid = book.createPositionProtection(params);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
      "armed"
    );
  }
  function _controlCloseAtTrigger(address owner, uint64 pid) internal returns (uint256 settlementAfter) {
    uint64 attemptId = _trigger(keeper, pid, TRIGGER_PRICE);
    OrderV3Types.ExecutionResult memory r = _executeOrderPublic(keeper, attemptId, TRIGGER_PRICE, 0);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    assertEq(_positionSizeOf(owner), 0);
    settlementAfter = _settlementOf(owner);
  }
  function _trigger(address caller, uint64 pid, uint256 price) internal returns (uint64 attemptId) {
    _pocTick();
    uint256 nowTs = vm.getBlockTimestamp();
    bytes[] memory live = _pythPayload(price, 0, nowTs, nowTs - 1);
    uint256 fee = router.pletherOracle().getUpdateFee(live);
    _pocEnsureEth(caller, fee);
    vm.prank(caller);
    attemptId = book.triggerPositionProtection{value: fee}(pid, live);
    pocPrice = price;
  }
  function _attackerSkewAndTrigger(uint64 pid) internal returns (uint64 openId, uint64 attemptId) {
    _traderDeposit(attacker, A_DEPOSIT);
    _pocTick();
    openId = _commitOpen(attacker, CfdTypes.Side.SHORT, A_SIZE, A_MARGIN);
    uint256 nowTs = vm.getBlockTimestamp();
    bytes[] memory live = _pythPayload(TRIGGER_PRICE, 0, nowTs, nowTs - 1);
    uint256 fee = router.pletherOracle().getUpdateFee(live);
    _pocEnsureEth(attacker, fee);
    vm.prank(attacker);
    attemptId = book.triggerPositionProtection{value: fee}(pid, live);
    pocPrice = TRIGGER_PRICE;
    assertGt(attemptId, openId);
  }
  function _griefFirstBatch(uint64 openId, uint64 attemptId, uint64 pid) internal returns (uint64 next, uint256 gasUsed, uint256 feeWei) {
    (IOrderRouterAccounting.PendingOrderView memory first,) = router.getPendingOrderView(openId);
    _pocTick();
    bytes[] memory data = _historicalPayload(TRIGGER_PRICE, 0, first.commitTime, first.commitTime);
    feeWei = router.pletherOracle().getOrderExecutionFee(data) * 2;
    _pocEnsureEth(griefEoa, feeWei);
    uint256 g0 = gasleft();
    vm.prank(griefEoa);
    (uint32 terminalCount, uint64 next_) = griefer.cycleBatch{value: feeWei}(attemptId, pid, data);
    gasUsed = g0 - gasleft();
    next = next_;
    assertEq(terminalCount, 2);
    assertEq(_positionSizeOf(attacker), A_SIZE);
    assertEq(
      uint8(lifecycle.lifecycleStatus(attemptId)), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt #1 failed"
    );
    assertEq(
      uint8(lifecycle.terminalOutcome(attemptId).reason),
      uint8(OrderV3Types.TerminalReason.ConstraintViolation),
      "attempt #1 failed on the victim's own close limits"
    );
  }
  function _griefCycle(uint64 attemptId, uint64 pid, uint256 price) internal returns (uint64 next, uint256 gasUsed, uint256 feeWei) {
    return _griefCycleAt(attemptId, pid, price, CYCLE_SECONDS);
  }
  function _griefCycleAt(uint64 attemptId, uint64 pid, uint256 price, uint256 delaySeconds) internal returns (uint64 next, uint256 gasUsed, uint256 feeWei) {
    (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(attemptId);
    assertEq(pending.orderId, attemptId);
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(uint256(pending.commitTime) + delaySeconds);
    assertFalse(_isFadOrFrozen());
    bytes[] memory data = _historicalPayload(price, 0, pending.commitTime, pending.commitTime);
    feeWei = router.pletherOracle().getOrderExecutionFee(data);
    _pocEnsureEth(griefEoa, feeWei);
    uint256 g0 = gasleft();
    vm.prank(griefEoa);
    (uint8 st, uint8 reason, uint64 next_) = griefer.cycle{value: feeWei}(attemptId, pid, data);
    gasUsed = g0 - gasleft();
    next = next_;
    assertEq(st, uint8(OrderV3Types.LifecycleStatus.Failed));
    assertEq(reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation));
    assertGt(next, attemptId);
    pocPrice = price;
  }
  function _assertOwnerLocked(address owner, uint64 pid) internal {
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(vm.getBlockTimestamp() + 1);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
      "a live attempt exists at the tx boundary"
    );
    assertEq(router.pendingOrderCounts(owner), 1);
    vm.prank(owner);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
    book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
    vm.prank(owner);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
    book.cancelPositionProtection(pid);
    OrderV3Types.OrderRequest memory closeReq = _pocOrderRequest(CfdTypes.Side.LONG, V_SIZE, 0, true);
    vm.prank(owner);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
    book.cancelPositionProtectionAndCommitClose(pid, closeReq);
    vm.prank(owner);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
    OrderRouter(address(router)).commitOrder(closeReq);
    PositionProtectionTypes.PositionProtectionParams memory p;
    p.stopLossTriggerPrice = 1.9e8;
    p.closeLimits = _unlimited();
    vm.prank(owner);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
    book.replacePositionProtection(pid, p);
  }
  function _receiptFor(Vm.Log[] memory logs, uint64 orderId) internal view returns (OrderV3Types.OrderReceipt memory receipt) {
    bool found;
    for (uint256 i; i < logs.length; ++i) {
      if (
        logs[i].emitter == address(lifecycle) && logs[i].topics.length > 1
          && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
          && uint256(logs[i].topics[1]) == uint256(orderId)
      ) {
        (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
        found = true;
      }
    }
    require(found, "receipt not found");
  }
  function test_control_noAttacker_protectionClosesAtTrigger() public {
    uint64 pid = _openAndProtect(victim);
    uint64 attemptId = _trigger(keeper, pid, TRIGGER_PRICE);
    vm.recordLogs();
    OrderV3Types.ExecutionResult memory r = _executeOrderPublic(keeper, attemptId, TRIGGER_PRICE, 0);
    OrderV3Types.OrderReceipt memory receipt = _receiptFor(vm.getRecordedLogs(), attemptId);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    assertEq(_positionSizeOf(victim), 0);
    PositionProtectionTypes.CloseFinancialLimits memory l = _victimLimits();
    assertLe(receipt.economics.actionChargeAssessedAssets, l.maxActionChargeAssets);
    assertLe(receipt.economics.grossAccountDebitAssets, l.maxGrossAccountDebitAssets);
    console2.log("controlA action charge assessed (USDG atoms)", receipt.economics.actionChargeAssessedAssets);
    console2.log("controlA vpi (signed, USDG atoms)", receipt.economics.vpiAssets);
    console2.log("controlA gross account debit (USDG atoms)", receipt.economics.grossAccountDebitAssets);
    console2.log("controlA victim settlement after close", _settlementOf(victim));
    _assertReleaseParams();
  }
  function test_control_attackerSkew_noGriefer_ownerAmendsAndExits() public {
    uint64 pid = _openAndProtect(victim);
    (, uint64 attemptId) = _attackerSkewAndTrigger(pid);
    vm.recordLogs();
    _executeBatch(keeper, attemptId, TRIGGER_PRICE);
    OrderV3Types.OrderReceipt memory receipt = _receiptFor(vm.getRecordedLogs(), attemptId);
    assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
    assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge));
    assertGt(receipt.failure.actual, receipt.failure.limit);
    assertEq(receipt.failure.limit, _victimLimits().maxActionChargeAssets);
    console2.log("controlB failing constraint: ActionCharge actual (USDG atoms)", receipt.failure.actual);
    console2.log("controlB failing constraint: ActionCharge limit (USDG atoms)", receipt.failure.limit);
    console2.log("controlB attacked close: receipt vpi (signed, USDG atoms)", receipt.economics.vpiAssets);
    console2.log("controlB attacked close: receipt execution fee (USDG atoms)", receipt.economics.executionFeeAssets);
    console2.log("controlB attacked close: receipt pool depth (USDG atoms)", receipt.poolDepthAssets);
    (uint256 aSize,,) = _positionOf(attacker);
    console2.log("controlB attacker SHORT notional at 1.212 (USDG atoms)", (aSize / 1e20) * TRIGGER_PRICE);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
      "latched"
    );
    assertEq(router.pendingOrderCounts(victim), 0);
    PositionProtectionTypes.CloseFinancialLimits memory amended = _victimLimits();
    amended.maxActionChargeAssets = 80e6;
    vm.prank(victim);
    book.amendLatchedPositionProtectionCloseLimits(pid, amended);
    vm.prank(keeper);
    uint64 retryId = book.retryPositionProtectionClose(pid);
    OrderV3Types.ExecutionResult memory r = _executeOrderPublic(keeper, retryId, TRIGGER_PRICE, 0);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    assertEq(_positionSizeOf(victim), 0);
    console2.log("controlB victim settlement after amend + exit", _settlementOf(victim));
    _assertReleaseParams();
  }
  function test_protlock_attackerInduced_lock24h() public {
    uint64 pid = _openAndProtect(victim);
    (, uint256 vMargin,) = _positionOf(victim);
    uint256 vSettlementBefore = _settlementOf(victim);
    uint256 snap = vm.snapshotState();
    uint256 victimControl = _controlCloseAtTrigger(victim, pid);
    vm.revertToState(snap);
    (uint64 openId, uint64 attempt) = _attackerSkewAndTrigger(pid);
    uint256 attackerStart = A_DEPOSIT;
    console2.log("attacker settlement after commit + trigger (deposit + trigger bounty)", _settlementOf(attacker));
    uint256 totalGas;
    uint256 totalFeeWei;
    uint256 cycles;
    uint256 maxGas;
    {
      (uint64 next, uint256 g, uint256 f) = _griefFirstBatch(openId, attempt, pid);
      attempt = next;
      totalGas += g;
      totalFeeWei += f;
      ++cycles;
      maxGas = g;
    }
    uint256 lockStart = vm.getBlockTimestamp();
    _assertOwnerLocked(victim, pid);
    bool passed1h;
    uint256 meteredCycles = cycles;
    while (vm.getBlockTimestamp() < lockStart + 24 hours + 5 minutes) {
      (uint64 next, uint256 g, uint256 f) = _griefCycle(attempt, pid, TRIGGER_PRICE);
      attempt = next;
      totalFeeWei += f;
      ++cycles;
      if (!passed1h) {
        totalGas += g;
        ++meteredCycles;
        if (g > maxGas) {
          maxGas = g;
        }
      }
      _assertOwnerLocked(victim, pid);
      if (!passed1h && vm.getBlockTimestamp() >= lockStart + 1 hours) {
        passed1h = true;
        console2.log("owner exits all reverted for >= 1h; cycles so far", cycles);
        console2.log("metered: avg gas per griefer tx over the first hour", totalGas / meteredCycles);
        vm.pauseGasMetering();
      }
    }
    vm.resumeGasMetering();
    uint256 lockedFor = vm.getBlockTimestamp() - lockStart;
    assertGe(lockedFor, 24 hours);
    assertEq(_positionSizeOf(victim), V_SIZE);
    (, uint256 vMarginNow,) = _positionOf(victim);
    console2.log("lock duration (s)", lockedFor);
    console2.log("griefer cycles (txs)", cycles);
    console2.log("griefer metered cycles (first hour)", meteredCycles);
    console2.log("griefer avg gas per cycle (metered)", totalGas / meteredCycles);
    console2.log("griefer max gas per cycle (metered)", maxGas);
    console2.log("griefer gas extrapolated to the whole lock", (totalGas / meteredCycles) * cycles);
    console2.log("griefer Pyth fees paid (wei, MockPyth 1 wei/update)", totalFeeWei);
    console2.log("victim position margin frozen (USDG atoms) at open", vMargin);
    console2.log("victim position margin frozen (USDG atoms) after 24h", vMarginNow);
    console2.log("victim total settlement custody before attack", vSettlementBefore);
    uint64 closeId = _commitClose(attacker, CfdTypes.Side.SHORT, A_SIZE);
    vm.recordLogs();
    _executeBatch(keeper, closeId, TRIGGER_PRICE);
    OrderV3Types.OrderReceipt memory last = _receiptFor(vm.getRecordedLogs(), attempt);
    assertEq(uint8(last.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge));
    console2.log("after 24h: ActionCharge actual (USDG atoms)", last.failure.actual);
    console2.log("after 24h: ActionCharge limit (USDG atoms)", last.failure.limit);
    assertEq(_positionSizeOf(attacker), 0);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
      "latched, no griefer"
    );
    vm.prank(keeper);
    uint64 retryId = book.retryPositionProtectionClose(pid);
    OrderV3Types.ExecutionResult memory r = _executeOrderPublic(keeper, retryId, TRIGGER_PRICE, 0);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    uint256 victimFinal = _settlementOf(victim);
    uint256 attackerEnd = _settlementOf(attacker);
    uint256 attackerCost = attackerStart > attackerEnd ? attackerStart - attackerEnd : 0;
    uint256 victimLoss = victimControl > victimFinal ? victimControl - victimFinal : 0;
    console2.log("victim settlement, control (no attack)", victimControl);
    console2.log("victim settlement, after 24h lock", victimFinal);
    console2.log("victim extra loss vs control (USDG atoms)", victimLoss);
    console2.log("attacker settlement start", attackerStart);
    console2.log("attacker settlement end (flat price, includes trigger bounty)", attackerEnd);
    console2.log("attacker position cost: fees + carry + VPI net (USDG atoms)", attackerCost);
    assertGt(attackerCost, 0);
    assertLt(attackerCost, 500e6);
    assertGe(vMarginNow, vMargin - 5e6);
    _assertReleaseParams();
  }
  function _controlBExitValue(uint64 pid, uint64 attempt) internal returns (uint256 victimControlB) {
    uint256 snapB = vm.snapshotState();
    _executeBatch(keeper, attempt, TRIGGER_PRICE);
    PositionProtectionTypes.CloseFinancialLimits memory amended = _victimLimits();
    amended.maxActionChargeAssets = 80e6;
    vm.prank(victim);
    book.amendLatchedPositionProtectionCloseLimits(pid, amended);
    vm.prank(keeper);
    uint64 retryId = book.retryPositionProtectionClose(pid);
    OrderV3Types.ExecutionResult memory rb = _executeOrderPublic(keeper, retryId, TRIGGER_PRICE, 0);
    assertEq(uint8(rb.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    victimControlB = _settlementOf(victim);
    vm.revertToState(snapB);
  }
  function _victimOptionsWhileLocked() internal {
    uint256 freeBefore = _freeSettlement(victim);
    vm.prank(victim);
    engine.addMargin(victim, 1e6);
    _traderWithdraw(victim, 1e6);
    uint256 freeNow = _freeSettlement(victim);
    assertLe(freeNow, freeBefore - 2e6);
    assertGe(freeNow + 2e6 + 0.5e6, freeBefore);
    vm.prank(victim);
    try clearinghouse.withdraw(victim, freeNow + 1) {
      revert("margin withdraw unexpectedly succeeded");
    } catch (bytes memory err) {
      bytes4 sel = bytes4(err);
      assertTrue(
        sel == IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector
          || sel == ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition.selector,
        "margin withdraw blocked by the free-equity / open-position guard"
      );
      console2.log(
        "margin withdraw revert selector (InsufficientFreeEquity=1, WithdrawBlocked=2)",
        sel == IMarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector ? 1 : 2
      );
    }
    OrderV3Types.OrderRequest memory inc = _pocOrderRequest(CfdTypes.Side.LONG, 100 * 1e20, 100e6, false);
    vm.prank(victim);
    vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
    OrderRouter(address(router)).commitOrder(inc);
    console2.log("victim: addMargin ok, free withdraw ok, margin withdraw/any order blocked");
  }
  struct DriftCtx {
    uint64 pid;
    uint64 attempt;
    uint256 totalGas;
    uint256 cycles;
    uint256 lockStart;
    uint256 price;
    uint256 victimControl;
    uint256 victimControlB;
  }
  function _driftLoop(DriftCtx memory d) internal {
    while (true) {
      if (d.cycles == 20) {
        _victimOptionsWhileLocked();
      }
      uint256 nextPrice = d.price + 0.0012e8;
      d.price = nextPrice;
      if (engineLens.previewLiquidation(victim, nextPrice).liquidatable) {
        break;
      }
      (uint64 next, uint256 g,) = _griefCycle(d.attempt, d.pid, nextPrice);
      d.attempt = next;
      d.totalGas += g;
      ++d.cycles;
      _assertOwnerLocked(victim, d.pid);
    }
  }
  function test_protlock_attackerInduced_driftToLiquidation() public {
    DriftCtx memory d;
    d.pid = _openAndProtect(victim);
    {
      uint256 snap = vm.snapshotState();
      d.victimControl = _controlCloseAtTrigger(victim, d.pid);
      vm.revertToState(snap);
    }
    {
      (uint64 openId, uint64 attempt) = _attackerSkewAndTrigger(d.pid);
      d.victimControlB = _controlBExitValue(d.pid, attempt);
      (uint64 next, uint256 g,) = _griefFirstBatch(openId, attempt, d.pid);
      d.attempt = next;
      d.totalGas += g;
      ++d.cycles;
    }
    d.lockStart = vm.getBlockTimestamp();
    _assertOwnerLocked(victim, d.pid);
    d.price = TRIGGER_PRICE;
    _driftLoop(d);
    uint256 lockedFor = vm.getBlockTimestamp() - d.lockStart;
    assertGe(lockedFor, 1 hours);
    assertEq(_positionSizeOf(victim), V_SIZE);
    uint256 griefSettlementBefore = _settlementOf(griefEoa);
    _liquidate(griefEoa, victim, d.price);
    assertEq(_positionSizeOf(victim), 0);
    assertEq(
      uint8(views.getPositionProtection(d.pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated),
      "protection forfeited"
    );
    uint256 victimFinal = _settlementOf(victim);
    uint256 damage = d.victimControl - victimFinal - 1e6;
    console2.log("lock duration until liquidation (s)", lockedFor);
    console2.log("griefer cycles", d.cycles);
    console2.log("griefer avg gas per cycle", d.totalGas / d.cycles);
    console2.log("liquidation price", d.price);
    console2.log("victim settlement, control (stop executed at 1.212)", d.victimControl);
    console2.log("victim settlement, after lock + liquidation", victimFinal);
    console2.log("victim damage vs control (USDG atoms)", damage);
    console2.log("victim settlement, control B (attacker skew, no griefer, owner amends)", d.victimControlB);
    console2.log("griefer-attributable damage vs control B (USDG atoms)", d.victimControlB - victimFinal - 1e6);
    console2.log("griefer liquidation bounty (USDG atoms)", _settlementOf(griefEoa) - griefSettlementBefore);
    assertGt(damage, 700e6);
    _assertReleaseParams();
  }
  function _walletExec(OwnerBatchWallet wallet, address ownerEoa, address target, uint256 value, bytes memory data) internal {
    address[] memory targets = new address[](1);
    uint256[] memory values = new uint256[](1);
    bytes[] memory datas = new bytes[](1);
    targets[0] = target;
    values[0] = value;
    datas[0] = data;
    _pocEnsureEth(ownerEoa, value);
    vm.prank(ownerEoa);
    wallet.execute{value: value}(targets, values, datas);
  }
  function _openAndProtectWallet(OwnerBatchWallet wallet, address ownerEoa) internal returns (uint64 pid) {
    address w = address(wallet);
    _pocMintUsdg(w, V_DEPOSIT);
    _walletExec(wallet, ownerEoa, address(settlementToken), 0, abi.encodeWithSignature("approve(address,uint256)", address(clearinghouse), V_DEPOSIT));
    _walletExec(wallet, ownerEoa, address(clearinghouse), 0, abi.encodeWithSignature("deposit(address,uint256)", w, V_DEPOSIT));
    OrderV3Types.OrderRequest memory req = _pocOrderRequest(CfdTypes.Side.LONG, V_SIZE, V_MARGIN, false);
    _walletExec(wallet, ownerEoa, address(router), 0, abi.encodeCall(OrderRouter.commitOrder, (req)));
    OrderV3Types.ExecutionResult memory r = _executeNext(keeper, ENTRY);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    _setPrice(ENTRY);
    _updateMark(keeper, ENTRY);
    PositionProtectionTypes.PositionProtectionParams memory params;
    params.stopLossTriggerPrice = SL;
    params.closeLimits = _victimLimits();
    _walletExec(wallet, ownerEoa, address(book), 0, abi.encodeCall(IPositionProtectionActions.createPositionProtection, (params)));
    pid = views.activePositionProtectionId(w);
    assertGt(pid, 0);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
      "wallet protection armed"
    );
  }
  function _escapeBatch(uint64 attemptId, uint64 pid, uint64 attemptCommitTime) internal view returns (address[] memory targets, uint256[] memory values, bytes[] memory datas, uint256 fee) {
    bytes[] memory data = _historicalPayload(TRIGGER_PRICE, 0, attemptCommitTime, attemptCommitTime);
    fee = router.pletherOracle().getOrderExecutionFee(data);
    PositionProtectionTypes.CloseFinancialLimits memory amended = _victimLimits();
    amended.maxActionChargeAssets = 80e6;
    targets = new address[](3);
    values = new uint256[](3);
    datas = new bytes[](3);
    targets[0] = address(router);
    values[0] = fee;
    datas[0] = abi.encodeWithSignature("executeOrder(uint64,bytes[])", attemptId, data);
    targets[1] = address(book);
    datas[1] = abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (pid, amended));
    targets[2] = address(book);
    datas[2] = abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (pid));
  }
  function _tryEscape(OwnerBatchWallet wallet, address ownerEoa, uint64 attemptId, uint64 pid, uint64 attemptCommitTime) internal returns (bytes4 sel) {
    (address[] memory targets, uint256[] memory values, bytes[] memory datas, uint256 fee) =
      _escapeBatch(attemptId, pid, attemptCommitTime);
    _pocEnsureEth(ownerEoa, fee);
    vm.prank(ownerEoa);
    try wallet.execute{value: fee}(targets, values, datas) {
      sel = bytes4(0);
    } catch (bytes memory err) {
      sel = bytes4(err);
    }
  }
  function test_limitation_batchingOwnerWalletEscapesIfFirst() public {
    address ownerEoa = _pocActor("walletOwner");
    vm.prank(ownerEoa);
    OwnerBatchWallet wallet = new OwnerBatchWallet(ownerEoa);
    address w = address(wallet);
    uint64 pid = _openAndProtectWallet(wallet, ownerEoa);
    (uint64 openId, uint64 attempt) = _attackerSkewAndTrigger(pid);
    (attempt,,) = _griefFirstBatch(openId, attempt, pid);
    (attempt,,) = _griefCycle(attempt, pid, TRIGGER_PRICE);
    assertEq(
      uint8(views.getPositionProtection(pid).status),
      uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
      "griefer tx boundary: Triggered"
    );
    (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(attempt);
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(uint256(pending.commitTime) + 5);
    bytes4 sel = _tryEscape(wallet, ownerEoa, attempt, pid, pending.commitTime);
    assertEq(sel, bytes4(0));
    uint64 escapeAttempt = views.getPositionProtection(pid).linkedOrderId;
    OrderV3Types.ExecutionResult memory r = _executeOrderPublic(keeper, escapeAttempt, TRIGGER_PRICE, 0);
    assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    assertEq(_positionSizeOf(w), 0);
    console2.log("limitation: batching wallet escaped at attempt commit + 5 s (58-s griefer cadence)");
    _assertReleaseParams();
  }
  struct FastCtx {
    OwnerBatchWallet wallet;
    address ownerEoa;
    uint64 pid;
    uint64 attempt;
    uint256 totalGas;
    uint256 meteredCycles;
    uint256 cycles;
    uint256 fails;
    bytes4 selA;
    bytes4 selB;
  }
  uint256 internal constant METERED_FAST = 240;
  function _fastCycle(FastCtx memory c) internal {
    (IOrderRouterAccounting.PendingOrderView memory pendingN,) = router.getPendingOrderView(c.attempt);
    (uint64 next, uint256 g,) = _griefCycleAt(c.attempt, c.pid, TRIGGER_PRICE, 1);
    ++c.cycles;
    if (c.meteredCycles < METERED_FAST) {
      c.totalGas += g;
      ++c.meteredCycles;
      if (c.meteredCycles == METERED_FAST) {
        vm.pauseGasMetering();
      }
    }
    c.selA = _tryEscape(c.wallet, c.ownerEoa, c.attempt, c.pid, pendingN.commitTime);
    assertTrue(c.selA != bytes4(0));
    (IOrderRouterAccounting.PendingOrderView memory pendingN1,) = router.getPendingOrderView(next);
    vm.roll(vm.getBlockNumber() + 1);
    c.selB = _tryEscape(c.wallet, c.ownerEoa, next, c.pid, pendingN1.commitTime);
    assertTrue(c.selB != bytes4(0));
    c.fails += 2;
    assertEq(views.getPositionProtection(c.pid).linkedOrderId, next);
    c.attempt = next;
  }
  function test_protlock_responsiveOwner_1sCadence_1h() public {
    FastCtx memory c;
    c.ownerEoa = _pocActor("walletOwner");
    vm.prank(c.ownerEoa);
    c.wallet = new OwnerBatchWallet(c.ownerEoa);
    c.pid = _openAndProtectWallet(c.wallet, c.ownerEoa);
    {
      (uint64 openId, uint64 attempt) = _attackerSkewAndTrigger(c.pid);
      (uint64 next, uint256 g,) = _griefFirstBatch(openId, attempt, c.pid);
      c.attempt = next;
      c.totalGas += g;
      ++c.meteredCycles;
      ++c.cycles;
    }
    uint256 lockStart = vm.getBlockTimestamp();
    while (vm.getBlockTimestamp() < lockStart + 1 hours + 1 minutes) {
      _fastCycle(c);
    }
    vm.resumeGasMetering();
    uint256 lockedFor = vm.getBlockTimestamp() - lockStart;
    assertGe(lockedFor, 1 hours);
    assertEq(_positionSizeOf(address(c.wallet)), V_SIZE);
    assertEq(c.selA, IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector);
    console2.log("fast cadence: lock duration (s)", lockedFor);
    console2.log("fast cadence: griefer txs", c.cycles);
    console2.log("fast cadence: owner escape batches tried, all reverted", c.fails);
    console2.log("fast cadence: metered griefer txs", c.meteredCycles);
    console2.log("fast cadence: avg gas per griefer tx (metered)", c.totalGas / c.meteredCycles);
    console2.log("fast cadence: griefer gas extrapolated to the lock", (c.totalGas / c.meteredCycles) * c.cycles);
    console2.log("fast cadence: (a) revert selector (OrderNotQueueHead)");
    console2.logBytes4(c.selA);
    console2.log("fast cadence: (b) revert selector");
    console2.logBytes4(c.selB);
    console2.log("  ProtectionNotLatched selector for reference");
    console2.logBytes4(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
    (IOrderRouterAccounting.PendingOrderView memory last,) = router.getPendingOrderView(c.attempt);
    vm.roll(vm.getBlockNumber() + 1);
    vm.warp(uint256(last.commitTime) + 2);
    assertEq(_tryEscape(c.wallet, c.ownerEoa, c.attempt, c.pid, last.commitTime), bytes4(0));
    _assertReleaseParams();
  }
  function test_protlock_liveOrderFlow_retryBackrun() public {
    uint64 pid = _openAndProtect(victim);
    (uint64 openId, uint64 attempt) = _attackerSkewAndTrigger(pid);
    (attempt,,) = _griefFirstBatch(openId, attempt, pid);
    address trader = _pocActor("honestTrader");
    _traderDeposit(trader, 2_000e6);
    uint256 retryGasTotal;
    for (uint256 i; i < 3; ++i) {
      (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(attempt);
      vm.roll(vm.getBlockNumber() + 1);
      vm.warp(uint256(pending.commitTime) + 1);
      uint64 tid = _commitOpen(trader, CfdTypes.Side.LONG, 10 * 1e20, 200e6);
      assertGt(tid, attempt);
      OrderV3Types.BatchResult memory b = _executeBatch(keeper, tid, TRIGGER_PRICE);
      assertEq(b.terminalCount, 2);
      assertEq(
        uint8(views.getPositionProtection(pid).status),
        uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
        "window opened by honest flow"
      );
      assertEq(router.pendingOrderCounts(victim), 0);
      uint256 g0 = gasleft();
      vm.prank(griefEoa);
      attempt = book.retryPositionProtectionClose(pid);
      retryGasTotal += g0 - gasleft();
      vm.prank(victim);
      vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
      book.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
    }
    _assertOwnerLocked(victim, pid);
    console2.log("live flow: griefer retry-only backrun avg gas", retryGasTotal / 3);
    console2.log("live flow: honest trader LONG lots executed (reduces attacker skew)", _positionSizeOf(trader) / 1e20);
    _assertReleaseParams();
  }
}
```
