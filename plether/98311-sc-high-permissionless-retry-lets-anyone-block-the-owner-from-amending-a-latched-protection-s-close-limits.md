# \#98311 [SC-High] Permissionless retry lets anyone block the owner from amending a latched protection's close limits

**Submitted on Nov 6th 2026 at 22:23:04 UTC by @blackgrease for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98311
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a triggered protection close fails on the trader's close limits, the trader's only fix is `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits()`. Anyone acting as a keeper can call `retryPositionProtectionClose()` before the owner does, which re-queues the same failing close and makes the owner's subsequent amend revert. 

By repeating this, a malicious keeper keeps the trader locked out of their position - they can't amend, cancel or close it. The position stays open until the price moves back inside the limits or, in the worst case, it gets liquidated. As per the Immunefi scope, anyone can act as a keeper, so keepers are untrusted.

Although amend and retry share the same preconditions and their interaction is tested, the protocol fails to identify the adversarial side of the preconditions that malicious keepers can exploit, which also exposes an incomplete fix for a previous finding (Cecuro audit: MED-9).

## Vulnerability Details

Amending and retrying a protection close share the same precondition - the protection must be `Latched`. While only the owner can amend the close limits, anyone can retry a failed protection close.

Once a protection triggers, the trader is locked out of the position and cannot create other orders. This is enforced by:
1. `commitOrder`, which reverts with `ProtectionActive`
2. and `cancelPositionProtection()`, which calls `_cancel()`, which only accepts a protection that is `PendingOpen`/`Armed`.

If the protection's close then fails due to a close limit (e.g. the price gaps past the trigger and the notional exceeds `maxExecutionNotionalAssets`), the protection relatches and the owner's only way to get a successful close executed is to amend the limits.

```solidity
function amendLatchedPositionProtectionCloseLimits(...) external {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    if (router.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
    ...
}
```

The problem arises from two things involving `retryPositionProtectionClose()`:
1. The functions execution is permissionless
2. Retrying moves the protection back to `Triggered` with a new queued close, which makes the owner's amend revert

which creates a griefing scenario that malicious keepers can exploit.

```solidity
function retryPositionProtectionClose(uint64 protectionId) external returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId); // no owner check
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    ...
    _recordCloseAttempt(protection, linkedOrderId); // status -> Triggered
}
```

When a protection close fails, `retryPositionProtectionClose()` doesn't check why it failed. Any following retry without an amend re-queues the close with the same close limits, and it fails the same way unless conditions improve.

As per the Immunefi scope, anyone can act as a keeper, so keepers are untrusted users. 

Consider the below example using a stop-loss protection that a malicious keeper prevents from being amended, resulting in the worst-case scenario - liquidation.

1. Alice, using an EOA, holds a 10,000-unit LONG with a stop-loss at 1.04 and a $10,500 notional cap.
2. The price gaps from 1.03 to 1.06. The stop-loss triggers, but the close executes at 1.06 for $10,600, which breaks the cap and the protection close fails.
3. Alice tries to raise the cap to $11,000. An attacker calls `retryPositionProtectionClose()` first, re-queuing the close with the old cap, and Alice's subsequent amend reverts. (Any attempt to submit an ordinary close or cancel the protection will revert.)
4. Each time the close fails, the attacker clears it and retries atomically, keeping the position from being amended and closed.
5. Because Alice needs two separate transactions (clear, then amend), the attacker can retry in between them and maintain the attack.
6. The price keeps rising. Each rise pushes the notional further over the cap, so the close never passes.
7. At 1.19, Alice's losses push her below the maintenance margin and the position becomes liquidatable. The attacker, acting as a keeper, liquidates it and collects the liquidation reward.
8. Instead of closing near 1.04, Alice loses most of her remaining margin and pays the liquidation charge.

In this scenario, there are three things to note:
- Liquidation is a secondary consequence that is not guaranteed but included for thoroughness as a worst case scenario. The primary malicious act was preventing a protected close from being ammended.
- Alice can delay liquidation by adding margin - if she had the funds - but she still cannot exit. Any added margin will also be exposed.
- If Alice were a smart contract with multicall support or a smart account, she could batch the "clear and amend" into one transaction which would stop the attack. In this case, whether the attack continues depends on whose transaction executes first - the attacker's or Alice's.

**Protocol documentation, test and incomplete previous fixes**

1. In `DEPLOYMENT.md`, the protocol gives keepers the guideline - _"do not continuously queue the same failing attempt."_ However, this assumes all keepers will behave honestly, and nothing in the code enforces honest behaviour while the scope treats keepers as untrusted.

2. The only test that calls amend, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, has the owner amend before the third-party retry. No test covers a retry before the owner's first amend, or repeated retries that leave the owner unable to amend.

3. Additionally, this issue shows that the MED-9 fix from the Cecuro audit is incomplete. The response there says tight limits block the close "until conditions improve or the owner amends them". This issue prevents the owner from amending and, when conditions do not improve, can result in worse consequences - such as the position being liquidated.

## Impact

**Griefing:** A malicious keeper can keep a trader from fixing their close limits for as long as they keep retrying. During that time the trader can't amend, cancel or close manually, and their position stays exposed to the market.

For a stop-loss protection, this defeats its purpose, especially when the price move the trader was trying to exit keeps going. This keeps the trader trapped in a position they are trying to exit.

**Potential for liquidation as a secondary consequence:**  For a trader with an exploited stop-loss protection, this can result in their position being liquidated. 

In either case, the trader takes losses beyond the stop-loss level, and in the event of a liquidation, additionally loses most of their remaining margin and pays the liquidation charge.

In the PoC, the attacker retries 13 times while the price rises from 1.06 to 1.19, where Alice is liquidated. She ends with $86.00 instead of the $1,391.76 she keeps when able to amend - a $1,305.76 loss.

The attack costs only gas, about one transaction per 60 second execution window. The attacker gains nothing by default, but in the case of adverse price movements, can also be the keeper who collects the liquidation reward as an additional bonus of exploiting this issue.

Following Immunefi's griefing definition - _"when the attacker calls certain functions of the smart contract that would put it in a suboptimal state, thus blocking normal function execution for any user."_ - this issue demonstrates exactly that.

## Recommendation

Consider allowing the owner to amend the stored close limits while the protection is `Triggered`, not only when `Latched`.

In this case, any maliciously queued close keeps its original limits and fails as expected, while every later retry - including ones attempted by the attacker - will use the amended limits. If conditions do improve and the existing queued close successfully executes, the amended limits change nothing as no position exists.

```diff
    function amendLatchedPositionProtectionCloseLimits(...) external nonReentrant {
        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-       if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
-       if (router.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
+       if (protection.status != Latched && protection.status != Triggered) revert OrderRouter__ProtectionNotLatched();
        _setCloseLimits(protection, limits);
        _emitCloseLimitsUpdated(protection);
    }
```

## References

- `retryPositionProtectionClose()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405
- `amendLatchedPositionProtectionCloseLimits()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L329
- `cancelPositionProtection()` / `_cancel()`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419
- Keeper guidance: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L421
- Existing test: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L954
- Cecuro AI audit (2026-10-26), MED-9: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf



## Link to Proof of Concept

https://gist.github.com/blackgrease/d87c7fc0c2cd0d8d63b4ba683b4d57de

## Proof of Concept

The provided PoC in the private gist link - https://gist.github.com/blackgrease/d87c7fc0c2cd0d8d63b4ba683b4d57de - reproduces the stop-loss scenario above. Alice (an EOA) holds a 10,000-unit LONG at 1.00 with a stop-loss at 1.04 and a $10,500 notional cap. The price gaps from 1.03 to 1.06 and the stop-loss triggers.

**Understanding the PoC**

The attacker uses a helper contract that clears the failing close and retries in one transaction. Alice's amend, manual close and cancel all revert. She also tries clearing the close herself and then amending, but the attacker retries in between. The attacker repeats the clear-and-retry as the price rises 0.01 every 30 seconds until Alice is liquidated, with the attacker as the liquidator.

**Instructions to run**

1. Place the code below in `packages/perps/test/perps/spec/trader/PoC_StopLossRetryGriefingLiquidation.t.sol`
2. Run: `forge test --mt testPoC_StopLossRetryGriefingLiquidation --root packages/perps --threads 1 -vvv`

**Console output**

```bash
[PASS] testPoC_StopLossRetryGriefingLiquidation() (gas: 17428986)
Logs:
  [CONTROL] Alice balance after amended stop-loss close at 1.06 (USDC 6dp): 1391760000
  [ATTACK]  retry cycles while Alice was locked out: 13
  [ATTACK]  liquidation price (1e8): 119000000
  [ATTACK]  Alice balance after liquidation (USDC 6dp): 85999966
  [ATTACK]  Alice loss vs control (USDC 6dp): 1305760034
  [ATTACK]  attacker liquidation reward (USDC 6dp): 5000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 131.65ms (88.93ms CPU time)

```
