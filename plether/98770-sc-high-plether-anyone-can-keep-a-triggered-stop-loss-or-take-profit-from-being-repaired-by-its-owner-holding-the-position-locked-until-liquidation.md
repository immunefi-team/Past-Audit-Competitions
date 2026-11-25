# \#98770 [SC-High] Plether: anyone can keep a triggered stop-loss or take-profit from being repaired by its owner, holding the position locked until liquidation

**Submitted on Nov 10th 2026 at 04:15:34 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98770
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Anyone can keep a triggered stop-loss or take-profit from being repaired by its owner, holding the position locked until liquidation

**Severity:** Low
**Impact:** Contract fails to deliver promised returns, but doesn't lose value
**Asset:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src (`PositionProtectionBook.sol`)
**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Severity note

Filed as Low: the protocol does not deliver the owner-amendment remedy that its documentation and the Cecuro MED-9 fix promise. The same behaviour also matches the Medium row "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": the attacker spends only gas and gains nothing, and the trader's position is held open until liquidation. We leave the final level to triage.

## What is and is not claimed

Claimed: when a triggered protection's close attempt fails on one of the trader's own financial ceilings, the protocol's documented remedy is for the owner to amend those ceilings while the protection is `Latched`. `retryPositionProtectionClose` is callable by anyone and moves the protection out of `Latched`. A third party who calls it, at the cost of gas only, makes the owner's amendment revert and can repeat that every execution window. While this lasts the owner has no way to close, reduce, cancel, replace or re-protect the position. It stays open against the market until it is liquidated.

Not claimed: theft, or a freeze with no possible escape. An account that can execute two calls atomically (a contract wallet) can beat the retry; the PoC includes that case. An ordinary account sending separate transactions cannot rely on doing so.

## Summary

After a trigger, a protection is irreversible. The trader cannot cancel it, replace it, or commit any order of their own; the only route out of the position is the protection's own close attempt. Each attempt carries the financial ceilings the trader chose when creating the protection. If an attempt fails on a ceiling, the protection becomes `Latched`, and the owner may raise the ceilings with `amendLatchedPositionProtectionCloseLimits`, which requires the `Latched` status and an empty order queue.

`retryPositionProtectionClose` has no caller check. It queues a new close attempt with the unchanged ceilings and sets the status to `Triggered`. The attempt is certain to fail again, but while it is pending both conditions of the amendment are false. When it expires or fails, the protection returns to `Latched`, and the same caller can queue the next one in the same transaction.

## Root cause

All references are to `packages/perps/src/PositionProtectionBook.sol` at the pin unless stated.

- `retryPositionProtectionClose` (`:382-406`) loads the protection with `_activeProtection(protectionId)`, not `_ownedActiveProtection(protectionId, msg.sender)`. Any address may call it. It requires only `Latched`, a matching position and an empty queue, then commits a new close attempt.
- `amendLatchedPositionProtectionCloseLimits` (`:315-328`) is owner-only and reverts unless `status == Latched` and `pendingOrderCounts(msg.sender) == 0`.
- `handleFailedProtectionAttempt` (`:637-675`): a failed or expired attempt on an unchanged position sets the status back to `Latched`, so the cycle can repeat without limit.
- The owner's other routes are closed by design once triggered:
  - `cancelPositionProtection` and `replacePositionProtection` require `PendingOpen` or `Armed` (`:412-420`, `:268-277`);
  - `cancelPositionProtectionAndCommitClose` requires `Armed` (`:344-348`);
  - every ordinary commit reverts `OrderRouter__ProtectionActive` while a protection is active (`router/OrderLiquidationBatchLogic.sol:313-315`).

A retried attempt has a zero execution bounty and needs no deposit, so the caller risks nothing.

## How a trader gets here

The trader only has to have set a ceiling that binds at the trigger. This is easy to do without noticing.

In the PoC the trader opens a 100,000-token LONG at 1.00 and sets `maxExplicitFeesAssets = 40 USDG`, which is exactly the 4 bps execution fee on the entry notional. A LONG loses as the basket rises, so its stop-loss is above entry (1.02 here). At the stop the close notional is 102,000 USDG and the fee is 40.80 USDG, above the ceiling. The close attempt fails with `ConstraintViolation` and the protection latches. For a LONG stop-loss, a fee or notional ceiling sized from the entry is always exceeded at the trigger.

The vendor's response to Cecuro MED-9 describes this state and its remedy: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner amends them." `CANONICAL_ENTRYPOINTS.md:145-148` says the same: "The owner may amend the close ceilings of a `Latched` protection [...] when the account has no pending Router order."

## Attack path

No privileged role. The attacker holds no position and no balance in the protocol.

1. The victim's protection triggers and its close attempt fails on a ceiling. Status: `Latched`.
2. The attacker calls `retryPositionProtectionClose(protectionId)`. Status: `Triggered`, one pending order.
3. The victim's `amendLatchedPositionProtectionCloseLimits` reverts `OrderRouter__ProtectionNotLatched`. `cancelPositionProtection`, `replacePositionProtection` and `cancelPositionProtectionAndCommitClose` revert `OrderRouter__ProtectionNotArmed`. A manual close reverts `OrderRouter__ProtectionActive`.
4. When the attempt's 60-second window ends, the attacker calls, in one transaction, `executeOrder` on the expired attempt (no price data needed) and `retryPositionProtectionClose` again. The protection is never `Latched` between transactions.
5. If the victim fails the pending attempt themselves to reach `Latched`, the attacker's retry only has to be sequenced before the victim's separate amendment transaction.

The same effect occurs without malice: any keeper that promptly retries latched protections, which the design relies on for liveness, removes the owner's amendment window in the same way.

## Measurements

Local Foundry tests at the pin with the release risk parameters. All protection, order and liquidation calls are the production entrypoints; the price feed is the repository's mock.

| Test | Result |
|---|---|
| Control: no interference | Owner amends, a retry closes the position, the lock is released |
| One retry, then nothing for 25 hours | Protection still `Triggered`; amend, cancel, replace and manual close all revert |
| Sustained | Owner locked out for 14 continuous hours over 827 renewals; attacker cost about 947,000 gas per renewal and 0 USDG. The 14-hour figure is a test gas limit, not a protocol limit |
| Owner fails the attempt herself, attacker retries before her amend | Amend reverts |
| Price keeps moving, 1.02 to 1.11 | Position is liquidated. Trader's balance 49,909.70 to 39,999.60 USDG: the whole 10,000 USDG margin, where the stop at 1.02 would have limited the loss to about 2,000 |
| Way out: contract wallet batching "fail attempt" and "amend" | Succeeds; the next retry closes the position |

## Impact

A trader whose protection has triggered is exposed to the market with a stop that will not execute and a repair they are not allowed to apply. The attacker gains nothing and spends only gas. The trader's loss is the difference between the stop level and wherever the position is finally closed or liquidated; in the PoC that is about 7,900 USDG on a 10,000 USDG margin. The margin is also unavailable to the trader for as long as the lock is held, which the PoC shows for 14 and 25 hours.

## Limits

- The trader must have set a ceiling that binds at the trigger. A protection with all ceilings at `type(uint256).max` is not affected.
- A contract wallet that fails the pending attempt and amends in one transaction cannot be raced. An externally owned account that gets both transactions sequenced back to back, with no retry between them, also succeeds; it cannot guarantee that ordering.
- Liquidation still works, so the position is not open-ended.

## Why this is not a known issue

- **Cecuro MED-9** (keepers could retry protection closes without the trader's bounds). The fix made retries carry the trader's ceilings and added the owner-only amendment. This report shows the amendment that fix relies on can be denied by the permissionless retry it kept.
- **AI scan, "Protection trigger bounty is paid before a zero-bounty close is executed"** notes that "a Triggered or Latched protection cannot be cancelled to regain ordinary close control". That is the reason the amendment is the only remedy; the scan does not consider a third party blocking it.
- **`SECURITY.md`** states that "Retry is permissionless" and that operators "must alert on prolonged `Latched` state without a live child". Neither covers a live child being used to block the owner.
- Cecuro MED-15, MED-28, MED-30 and LOW-73 concern stuck SHORT closes at a zero price, arming from a stale mark, frozen-market triggers, and degraded mode. Different mechanisms.

## Recommended fix

Let the owner change the ceilings whenever the protection is `Triggered` or `Latched`, and have the next attempt use the stored values. That removes the dependency on an empty queue and on winning a race. Alternatives: give the owner an atomic "expire current attempt and amend" entrypoint, or allow the owner to convert a latched protection into an ordinary close as `cancelPositionProtectionAndCommitClose` does for an armed one.




## Link to Proof of Concept

https://gist.github.com/dexloom/ef3602aa3f3c89653c20c74c9d703406

## Proof of Concept

## Proof of concept

PoC: https://gist.github.com/dexloom/ef3602aa3f3c89653c20c74c9d703406

At the pin, initialise submodules, add the file to `packages/perps/test/perps/`, then:

    git submodule update --init --recursive
    cd packages/perps
    forge test --match-path test/perps/RetryBlocksAmendPoC.t.sol -vv

Expected: 6 tests pass. No file under `src` is changed and no RPC is needed.

```
[PASS] test_Attack_LockedPositionRidesToLiquidation()
  alice_balance_before: 49909699789
  alice_balance_after_liquidation: 39999600000
[PASS] test_Attack_OwnerLatchesButStrangerBackrunsBeforeAmend()
[PASS] test_Attack_SingleRetryFreezesAmendAcrossDormant25h()
[PASS] test_Attack_StrangerRetryKeepsOwnerLockedOut_sustained()
  maxExecutionWindowSeconds: 60
  hours_locked: 14
  attacker_cycles: 827
  attacker_gas_per_cycle: 946744
  attacker_settlement_spent: 0
[PASS] test_Control_LatchedOwnerAmendsThenRetryCloses()
[PASS] test_WayOut_BatchingAccountWinsTheRace()
```

The opening order uses the repository's `LegacyOrderRouterHarness.commitOrder` wrapper, which forwards to the production commit path. Every other call in the tests is the deployed interface.
