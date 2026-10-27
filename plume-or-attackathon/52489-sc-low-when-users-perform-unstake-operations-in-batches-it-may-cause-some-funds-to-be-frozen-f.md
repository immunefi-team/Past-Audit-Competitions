# 52489 sc low when users perform unstake operations in batches it may cause some funds to be frozen for an additional period of time&#x20;

**Submitted on Aug 11th 2025 at 07:57:20 UTC by @Lin511 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52489
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief / Intro

When a user performs multiple unstake operations, if a subsequent unstake operation occurs during the cooling period of funds that were unstaked earlier, the newly unstaked funds are merged into the existing cooling entry and the cooldown timer is reset. This can extend the cooling period of the earlier unstaked funds, causing some of the user's funds to remain frozen for an additional period.

### Vulnerability Details

When the `unstake` function is called, `_processCooldownLogic` is invoked. If the user already has a cooling entry for the same validator that has not yet matured, the newly unstaked amount is added to that existing cooling entry and the cooldown end time is set to `block.timestamp + cooldownInterval`, effectively restarting the cooldown for the combined amount.

Relevant code excerpt:

```solidity
function _processCooldownLogic(
    address user,
    uint16 validatorId,
    uint256 amount
) internal returns (uint256 newCooldownEndTime) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.CooldownEntry storage cooldownEntrySlot = $.userValidatorCooldowns[user][validatorId];

    uint256 currentCooledAmountInSlot = cooldownEntrySlot.amount;
    uint256 currentCooldownEndTimeInSlot = cooldownEntrySlot.cooldownEndTime;

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
    cooldownEntrySlot.cooldownEndTime = newCooldownEndTime;

    return newCooldownEndTime;
}
```

Because the code always sets `cooldownEntrySlot.cooldownEndTime = block.timestamp + cooldownInterval` when adding to an existing cooling slot, a second unstake done during the first cooldown extends the maturity of the funds from the first unstake.

Example scenario (cooldown interval = 14 days):

{% stepper %}
{% step %}

### Step

Alice stakes 10,000 plume.
{% endstep %}

{% step %}

### Step

Alice unstakes 5,000 plume.
{% endstep %}

{% step %}

### Step

13 days pass.
{% endstep %}

{% step %}

### Step

Alice unstakes another 5,000 plume — as a result the cooldown for the earlier 5,000 is updated to now mature 14 days after the second unstake, i.e., \~27 days after the first unstake instead of 14.
{% endstep %}
{% endstepper %}

### Impact Details

Some of the user's funds may be frozen for an additional period. During the cooling period, funds earn no return; the opportunity cost scales with the configured cooldown interval. If the cooldown interval is long (e.g., one month), the user-facing impact and potential loss of returns can be significant.

## Proof of Concept

Proof-of-concept test to include under `PlumeStakingDiamondTest`:

```solidity
function testStakeAndUnstakeTwice() public {
    // In this poc, cooldown interval is set to 7 days.
    uint256 cooldownInterval = ManagementFacet(address(diamondProxy))
        .getCooldownInterval();
    assertEq(cooldownInterval, 604800);
    
    uint256 amount = 100e18;
    vm.startPrank(user1);
    
    // Assume user stake 100 plumes.
    StakingFacet(address(diamondProxy)).stake{value: amount}(
        DEFAULT_VALIDATOR_ID
    );
    assertEq(StakingFacet(address(diamondProxy)).amountStaked(), amount);

    // Unstake half.
    StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, amount/2);
    assertEq(StakingFacet(address(diamondProxy)).amountCooling(), amount/2);

    // Cooldown interval is 7 days, use cheatcode to let 6 days passed, the first unstaked part is still cooling
    vm.warp(vm.getBlockTimestamp() + 6 days);
    assertEq(StakingFacet(address(diamondProxy)).amountWithdrawable(), 0);

    // User unstake again, the mature date is updated to 7 days later.
    // For the first unstaked funds, it's mature period become 13 days.
    StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, amount/4);

    // 12 days passed since the first unstake, and it's mature period is already passed.
    vm.warp(vm.getBlockTimestamp() + 6 days);

    // However the first unstaked part is still in cooling slot.
    assertEq(StakingFacet(address(diamondProxy)).amountWithdrawable(), 0);

    // Users need to wait for the second unstaked funds become matured before they can withdraw the first unstaked funds.
    vm.warp(vm.getBlockTimestamp() + 1 days);
    assertEq(StakingFacet(address(diamondProxy)).amountWithdrawable(), amount * 3 / 4);

    vm.stopPrank();
}
```

## References

None
