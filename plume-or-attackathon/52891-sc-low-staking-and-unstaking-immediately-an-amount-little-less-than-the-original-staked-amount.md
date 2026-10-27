# 52891 sc low staking and unstaking immediately an amount little less than the original staked amount leaves dust stake amounts in the system&#x20;

**Submitted on Aug 14th 2025 at 02:21:03 UTC by @WinSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52891
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Staking X amount of tokens and then unstaking immediately an amount equal to X - 1 or X - (minAmount - 1) allows anyone to leave a residual stake amount smaller than the minimum stake. This enables bots to spam with dust amounts, increasing the size of the `validatorStakers` array and complicating accounting. Additionally, the protocol intends that stakes below the minimum amount should not earn rewards, but these dust stakes will continue to earn rewards.

### Vulnerability Details

In `stake`, `restake` and `restakeRewards` functions:

```solidity
    function _validateStakeAmount(
        uint256 amount
    ) internal view {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        if (amount == 0) {
            revert InvalidAmount(0);
        }
        if (amount < $.minStakeAmount) {
            revert StakeAmountTooSmall(amount, $.minStakeAmount);
        }
    }
```

The above check ensures that the amount being staked is greater than or equal to the `minStakeAmount`. But this check can be bypassed by staking and then immediately unstaking an amount slightly smaller than the original stake. That leaves a dust amount (less than `minStakeAmount`) in the system. The `unstake` function lacks a check to prevent leaving such dust. The `unstake` function should ensure that if the amount remaining after unstaking would be less than the minimum stake amount, the function unstakes the whole amount instead of leaving dust.

### Impact Details

* Dust amounts remain in the system, complicating accounting.
* Allows bots to spam with dust stakes, unnecessarily growing `validatorStakers`.
* Dust amounts may continue to earn rewards even though they are below the intended minimum stake.

### References

<https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/StakingFacet.sol#L105>

## Proof of Concept

{% stepper %}
{% step %}

### Step

User stakes X amount in the protocol by calling the `stake` function.
{% endstep %}

{% step %}

### Step

User immediately calls `unstake`.
{% endstep %}

{% step %}

### Step

User unstakes an amount equal to X - 1.
{% endstep %}

{% step %}

### Step

This leaves an amount equal to 1 wei (or another small dust amount) in the system.
{% endstep %}

{% step %}

### Step

The remaining dust amount is less than the min amount required for staking and thus remains as a dust stake while still potentially earning rewards.
{% endstep %}
{% endstepper %}
