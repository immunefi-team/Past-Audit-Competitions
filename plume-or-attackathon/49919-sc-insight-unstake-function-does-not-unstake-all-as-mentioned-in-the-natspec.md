# 49919 sc insight unstake function does not unstake all as mentioned in the natspec

**Submitted on Jul 20th 2025 at 15:04:14 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49919
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Users cannot unstake all funds using the documented zero-amount method.

## Description

### Summary

The `unstake` function in `StakingFacet.sol` contradicts its NatSpec documentation regarding zero-amount behavior. The NatSpec states that passing `amount = 0` should unstake all funds, but the implementation reverts when `amount == 0`, preventing users from unstaking their complete stake through the documented method.

### Vulnerability Details

The NatSpec explicitly documents that passing `0` as the `amount` parameter should trigger an "unstake all" behavior. However, both the public function and the internal implementation revert when `amount == 0`.

Snippet from the repository (link preserved): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L371-L373>

```solidity
   function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
@>        if (amount == 0) {
@>            revert InvalidAmount(0);
@>        }
        return _unstake(validatorId, amount);
    }
    
    
    /**
     * @notice Internal logic for unstaking, handles moving stake to cooling or parked.
     * @param validatorId ID of the validator to unstake from.
@>     * @param amount The amount of PLUME to unstake. If 0, unstakes all.
     * @return amountToUnstake The actual amount that was unstaked.
     */
    function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
        ...
        // Validate unstaking conditions
        _validateValidatorForUnstaking(validatorId);
@>        if (amount == 0) {
@>            revert InvalidAmount(amount);
@>        }
        ...
     }
```

## Impact

Users cannot unstake their complete stake using the documented zero-amount method, because the function reverts with `InvalidAmount(0)` instead of unstaking all the user's funds.

{% hint style="warning" %}
Behavior mismatch between documentation and implementation can lead to user confusion and failed transactions when callers expect the documented "unstake all" shortcut.
{% endhint %}

## Recommendation

Either:

* Update the implementation so that when `amount == 0` the function unstakes the user's full stake (implement the "unstake all" behavior), or
* Update the NatSpec documentation to reflect that passing `0` is invalid and will revert.

## Proof of Concept

{% stepper %}
{% step %}

1. User calls:

```js
unstake(validatorId, 0)
```

{% endstep %}

{% step %}
2\. Expected behavior: function unstakes all user's funds from the validator and the transaction succeeds.
{% endstep %}

{% step %}
3\. Actual behavior: transaction reverts with `InvalidAmount(0)`.
{% endstep %}
{% endstepper %}
