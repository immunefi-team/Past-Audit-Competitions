# 53063 sc low maxvalidatorpercentage can be used to dos protocol staking&#x20;

**Submitted on Aug 14th 2025 at 18:41:03 UTC by @heeze for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53063
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

The `StakingFacet` contract enforces a maximum validator percentage limit to prevent any single validator from controlling too much of the total stake. However, if the first staker deposits only the minimum stake amount, and the protocol is configured with more than three validators and a `maxValidatorPercentage` such that the sum of all validator percentages equals 100% (e.g., 3 validators at 33% each), then after the first stake, no further staking can occur to any validator. This results in a protocol-wide denial-of-service (DoS), freezing all staking activity and preventing any further participation.

### Vulnerability Details

The `_validateValidatorPercentage` function checks that a validator's delegated amount does not exceed a set percentage of the total staked amount:

```solidity
if (previousTotalStaked > 0 && $.maxValidatorPercentage > 0) {
    uint256 newDelegatedAmount = $.validators[validatorId].delegatedAmount;
    uint256 validatorPercentage = (newDelegatedAmount * 10_000) / $.totalStaked;
    if (validatorPercentage > $.maxValidatorPercentage) {
        revert ValidatorPercentageExceeded();
    }
}
```

When the first user stakes to a validator, `previousTotalStaked` is zero, so the check is skipped. After the first stake, the validator's delegated amount equals the total staked, making their percentage 100%.

Example calculations following the first stake:

* After first stake of 1 ETH to Validator1:
  * `totalStaked = 1 ETH`, `validator1.delegatedAmount = 1 ETH` (100%)
* Second stake of 1 ETH to Validator1:
  * validator's delegated amount becomes 2 ETH, totalStaked becomes 2 ETH → percentage = 100% → reverts if `maxValidatorPercentage = 33%`
* Stake of 1 ETH to Validator2 after first stake:
  * validatorPercentage = (1 ETH \* 10,000) / 2 ETH = 5,000 (50%) → reverts if `maxValidatorPercentage = 33%`
* Stake of 1 ETH to Validator3 after first stake:
  * same as Validator2 → reverts

Any further staking attempt to any validator will revert, as the validator percentage for any new deposit will exceed the per-validator limit.

### Impact Details

* If the first staker deposits only the minimum stake to any validator, no other user can stake to any validator, freezing the growth of the entire protocol.
* The rate of growth for the entire protocol is determined by the size of the first deposit to each validator, which is neither intuitive nor fair.
* Validators and users can be permanently locked out of further participation, halting protocol growth and decentralization.

### References

* [StakingFacet.sol - \_validateValidatorPercentage function](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L147)

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Deploy the protocol and set:

* `minStakeAmount` to 1 ether
* `maxValidatorPercentage` to 33% (3300 basis points)
  {% endstep %}

{% step %}

### Step: First stake

User A stakes 1 ether (the minimum) to Validator1.

* `previousTotalStaked` is 0, so the percentage check is skipped.
* After staking: `delegatedAmount = 1 ether`, `totalStaked = 1 ether` for Validator1.
  {% endstep %}

{% step %}

### Step: Second stake to same validator

User B attempts to stake 1 ether to Validator1.

* Validator percentage = (2 ether \* 10,000) / 2 ether = 10,000 (100%), which exceeds 33%, so the transaction reverts.
  {% endstep %}

{% step %}

### Step: Stake to another validator

User C attempts to stake 1 ether to Validator2.

* Validator percentage = (1 ether \* 10,000) / 2 ether = 5,000 (50%), which exceeds 33%, so the transaction reverts.
  {% endstep %}

{% step %}

### Step: Stake to yet another validator

User D attempts to stake 1 ether to Validator3.

* Validator percentage = (1 ether \* 10,000) / 2 ether = 5,000 (50%), which exceeds 33%, so the transaction reverts.
  {% endstep %}

{% step %}

### Outcome

The protocol is now globally frozen: no additional staking can occur to any validator, and the staking system is effectively DoS'ed.
{% endstep %}
{% endstepper %}
