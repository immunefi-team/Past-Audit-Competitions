# 52960 sc insight incosistent withdrawable amount calculations

**Submitted on Aug 14th 2025 at 13:42:08 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52960
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In `StakingFacet`, the `amountWithdrawable()` function returns **matured cooldowns** that haven't been processed, while `totalAmountWithdrawable()` only returns **processed parked funds**, creating inconsistent balance reporting.

### Vulnerability Details

The two conflicting functions for the withdrawable returns are `totalAmountWithdrawable` and `amountWithdrawable`.

Notice the global calculation returns directly the `totalWithdrawable` variable:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L553-L555>

```solidity
function totalAmountWithdrawable() external view returns (uint256 amount) {
    return PlumeStakingStorage.layout().totalWithdrawable;
}
```

Now the `amountWithdrawable` function considers the matured cooldowns for the user:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L520-L522>

```solidity
function amountWithdrawable() external view returns (uint256 totalWithdrawableAmount) {
    return _calculateTotalWithdrawableAmount(msg.sender);
}

function _calculateTotalWithdrawableAmount(
    address user
) internal view returns (uint256 totalWithdrawableAmount) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint16[] storage userAssociatedValidators = $.userValidators[user];

    totalWithdrawableAmount = $.stakeInfo[user].parked; // Same as global method

    // BUT ALSO includes matured cooldowns that haven't been processed yet:
    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        PlumeStakingStorage.CooldownEntry storage cooldownEntry = $.userValidatorCooldowns[user][validatorId];

        if (cooldownEntry.amount > 0 && block.timestamp >= cooldownEntry.cooldownEndTime) {
            if ($.validatorExists[validatorId] && $.validators[validatorId].slashed) {
                if (cooldownEntry.cooldownEndTime < $.validators[validatorId].slashedAtTimestamp) {
                    totalWithdrawableAmount += cooldownEntry.amount; // NOT in global state
                }
            } else if ($.validatorExists[validatorId] && !$.validators[validatorId].slashed) {
                totalWithdrawableAmount += cooldownEntry.amount; // NOT in global state
            }
        }
    }

    return totalWithdrawableAmount;
}
```

The global `totalWithdrawable` state variable is only updated when funds are explicitly moved to parked state via these functions:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L731>

```solidity
// @audit increased when unstaking and restaking(matured cooldowns)
function _updateParkedAmounts(address user, uint256 amount) internal {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    $.stakeInfo[user].parked += amount;
    $.totalWithdrawable += amount; // Global state updated here
}
```

Problem is cooldowns become withdrawable immediately when they mature (`block.timestamp >= cooldownEndTime`), but the global state is not updated until:

* User calls `withdraw()` → triggers `_processMaturedCooldowns()`
* User calls `unstake()` → triggers `_processCooldownLogic()`
* User calls `restake()` → triggers `_processMaturedCooldowns()`

This causes the amount to withdraw to be inconsistent when calling the withdrawable functions.

## Example

{% stepper %}
{% step %}
User stakes 100 PLUME to Validator A
{% endstep %}

{% step %}
User unstakes 100 PLUME → enters cooldown period
{% endstep %}

{% step %}
Time passes → cooldown matures
{% endstep %}

{% step %}
Before the user calls any processing function, current state:

```solidity
// User's actual state:
$.stakeInfo[user].parked = 0                    // No processed funds
$.userValidatorCooldowns[user][validatorA] = {
    amount: 100,
    cooldownEndTime: past_timestamp             // Matured
}
$.totalWithdrawable = 0                         // Global state not updated
```

{% endstep %}

{% step %}
When the user calls `totalAmountWithdrawable`, the result is 0.
{% endstep %}

{% step %}
When the same user calls `amountWithdrawable`, it includes matured cooldown amounts and returns 100.
{% endstep %}
{% endstepper %}

This discrepancy happens because `totalAmountWithdrawable` does not account for recently matured cooldowns, so it will always be outdated compared to `amountWithdrawable`.

## Impact Details

Two view functions return different withdrawable amounts for the same user, breaking integrations that rely on consistent balance data.

## Recommendation

`totalAmountWithdrawable` never reflects the actual up-to-date withdrawable amount, it only includes parked amounts. Either update it to reflect the real total withdrawable amount or rename it to something more accurate, such as `totalParkedAmount`.

## Proof of Concept

{% stepper %}
{% step %}
User stakes 100 PLUME to Validator A
{% endstep %}

{% step %}
User unstakes 100 PLUME → enters cooldown period
{% endstep %}

{% step %}
Time passes → cooldown matures
{% endstep %}

{% step %}
Before the user calls any processing function, current state:

```solidity
// User's actual state:
$.stakeInfo[user].parked = 0                    // No processed funds
$.userValidatorCooldowns[user][validatorA] = {
    amount: 100,
    cooldownEndTime: past_timestamp             // Matured
}
$.totalWithdrawable = 0                         // Global state not updated
```

{% endstep %}

{% step %}
When the user calls `totalAmountWithdrawable`, the result is 0.
{% endstep %}

{% step %}
When the same user calls `amountWithdrawable`, it includes matured cooldown amounts and returns 100.
{% endstep %}
{% endstepper %}
