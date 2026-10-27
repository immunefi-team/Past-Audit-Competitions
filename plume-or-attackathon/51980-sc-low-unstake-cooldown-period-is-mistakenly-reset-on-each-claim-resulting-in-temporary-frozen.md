# 51980 sc low unstake cooldown period is mistakenly reset on each claim resulting in temporary frozen funds

* **Submitted on:** Aug 7th 2025 at 00:01:17 UTC by @ZeroXGondar for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51980
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Summary

Each time a user calls `unstake` while there is already a cooling unstake amount, the cooldown end time is reset for that user's slot, effectively re-starting the cooldown for previously submitted unstake claims. This results in temporarily frozen funds.

### Details

`StakingFacet::unstake` has incorrect cooldown handling. Example:

Alice stakes 100e18 tokens. She unstake 20e18 and must wait the cooldown period (e.g., 5 days) to withdraw. Before that cooldown finishes, she calls `unstake` again for 30e18. The implementation resets the cooldown end time for all cooled amounts in that user/validator slot, so the first 20e18's cooldown is extended and will only be withdrawable when the newest cooldown finishes — causing unintended freezing.

The problematic function flow (lines marked with ⇒):

```solidity
function _processCooldownLogic(
    address user,
    uint16 validatorId,
    uint256 amount
) internal returns (uint256 newCooldownEndTime) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.CooldownEntry storage cooldownEntrySlot = $.userValidatorCooldowns[user][validatorId];

    uint256 currentCooledAmountInSlot = cooldownEntrySlot.amount;
    => uint256 currentCooldownEndTimeInSlot = cooldownEntrySlot.cooldownEndTime;

    uint256 finalNewCooledAmountForSlot;
    newCooldownEndTime = block.timestamp + $.cooldownInterval;

    if (currentCooledAmountInSlot > 0 && block.timestamp >= currentCooldownEndTimeInSlot) {
        // Previous cooldown for this slot has matured - move to parked and start new cooldown
        _updateParkedAmounts(user, currentCooledAmountInSlot);
        _removeCoolingAmounts(user, validatorId, currentCooledAmountInSlot);
        _updateCoolingAmounts(user, validatorId, amount);
        finalNewCooledAmountForSlot = amount;
    } else {
        // No matured cooldown - add to existing cooldown
        _updateCoolingAmounts(user, validatorId, amount);
         finalNewCooledAmountForSlot = currentCooledAmountInSlot + amount;
    }

    cooldownEntrySlot.amount = finalNewCooledAmountForSlot;
=>  cooldownEntrySlot.cooldownEndTime = newCooldownEndTime;

    return newCooldownEndTime;
}
```

Because `cooldownEntrySlot.cooldownEndTime` is always overwritten with `newCooldownEndTime`, earlier pending cooldowns in the same slot are extended rather than treated independently.

While funds are not lost, they can be temporarily frozen longer than intended.

### Impact & severity

* User funds can be temporarily frozen when making additional `unstake` calls before a prior cooldown matured.
* The report classifies this as high severity (temporary freeze ≥ 24 hours). The metadata on this report lists severity as Low; the impact described indicates a higher practical severity due to typical multi-day cooldowns.

### Recommendation

{% hint style="info" %}
Do not reset the cooldown end time for the entire slot when adding a new cooling claim. Each claim should be tracked individually (or accumulated in a way that preserves existing matured timers) so previously initiated cooldowns are not extended by subsequent `unstake` calls.
{% endhint %}

## Example flow (intended vs current)

{% stepper %}
{% step %}

### Intended behavior

* Alice unstakes 20e18 on Monday → wait 5 days → those 20e18 are withdrawable on Friday.
* Alice unstakes an additional 30e18 on Thursday (4 days after the first unstake) → wait 5 days → those 30e18 are withdrawable next Monday.
* Result: Alice can withdraw 20e18 on Friday and 30e18 on next Monday.
  {% endstep %}

{% step %}

### Current (buggy) behavior

* Alice unstakes 20e18 on Monday → cooldown set to Friday.
* Alice unstakes 30e18 on Thursday → the slot cooldown end time is reset to Thursday+5 days.
* Result: the first 20e18 cooldown is extended; Alice can only withdraw the combined 50e18 on next Monday (Thursday+5 days), freezing the original 20e18 longer than intended.
  {% endstep %}
  {% endstepper %}

## Proof of Concept

{% stepper %}
{% step %}

1. Alice stakes 100e18 tokens.
   {% endstep %}

{% step %}
2\. Monday: Alice calls `unstake(20e18)`; cooldown = 5 days ⇒ withdrawable Friday.
{% endstep %}

{% step %}
3\. Thursday: Alice calls `unstake(30e18)`; implementation resets slot cooldown ⇒ new cooldown ends next Monday.
{% endstep %}

{% step %}
4\. Result: Alice cannot withdraw her original 20e18 on Friday; it is frozen until next Monday (5 days longer).
{% endstep %}
{% endstepper %}
