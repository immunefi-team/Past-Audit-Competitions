# 50914 sc low bypass of minimum stake enforcement via partial unstake

**Submitted on Jul 29th 2025 at 17:00:55 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50914
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Unbounded gas consumption
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The Plume staking contract enforces a minimum stake amount `$.minStakeAmount` during the staking process to ensure that users contribute a meaningful amount of tokens. However, the same constraint is not enforced during partial unstakes. As a result, users can bypass the minimum stake requirement and remain staked with less than the required minimum.

## Vulnerability Details

The `StakingFacet::stake` function correctly enforces a check ensuring that the amount being staked is not less than `$.minStakeAmount`:

```javascript
function _validateStakeAmount(uint256 amount) internal view {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        if (amount == 0) {
            revert InvalidAmount(0);
        }
@>        if (amount < $.minStakeAmount) {
            revert StakeAmountTooSmall(amount, $.minStakeAmount);
        }
    }
```

{% stepper %}
{% step %}

### How the bypass works

* Stake exactly the minimum amount (passes validation).
* Call `StakingFacet::unstake(uint16 validatorId, uint256 amount)` with a partial unstake less than the staked amount.
* The remaining stake falls below the minimum threshold because the unstake flow does not re-validate the user's remaining stake against `$.minStakeAmount`.
  {% endstep %}
  {% endstepper %}

There is no constraint such as:

```javascript
if (newStakeAmount != 0 && newStakeAmount < $.minStakeAmount) {
    revert RemainingStakeBelowMinimum();
}
```

Thus, users can keep an active stake that is less than what the protocol originally intended to allow.

## Impact Details

This bug undermines the protocol's invariant that users should always have at least the minimum stake amount. Possible consequences include:

* Inconsistent enforcement of staking rules across staking vs unstaking flows.
* Degraded validator staking metrics, since total staked values and user counts may reflect sub-minimum stake participants.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L353>

## Proof of Concept

{% stepper %}
{% step %}

1. Assume $.minStakeAmount = 1 ether.
2. Alice calls `stake(validatorId)` with exactly 1 ether. This passes validation.
3. Alice calls `unstake(validatorId, 0.1 ether)`.
4. Now, Alice has 0.9 ether staked (below the minStakeAmount).
5. Alice remains in the system with an invalid partial stake that could:
   * Still earn rewards,
   * Be counted in validator's staker list,
   * Trigger edge cases elsewhere in the protocol.
     {% endstep %}
     {% endstepper %}
