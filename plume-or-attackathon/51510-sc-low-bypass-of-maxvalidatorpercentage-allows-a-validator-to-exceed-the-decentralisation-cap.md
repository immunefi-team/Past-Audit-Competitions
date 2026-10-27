# 51510 sc low bypass of maxvalidatorpercentage allows a validator to exceed the decentralisation cap

**Submitted on Aug 3rd 2025 at 14:14:53 UTC by @Rhaydden for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51510
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`StakingFacet::_validateValidatorPercentage` is meant to stop any single validator from controlling more than `maxValidatorPercentage` of total stake. Because the check is calculated against the post-stake `totalStaked` and is only executed on staking, an attacker can momentarily inflate the denominator with a dummy stake to another validator, pass the check, then remove the dummy stake. The target validator’s share subsequently exceeds the cap without any further validation, breaking Plume's decentralisation guarantee.

## Vulnerability Details

Relevant code:

```solidity
uint256 previousTotalStaked = $.totalStaked - stakeAmount;          // correct pre-stake total (unused)

if (previousTotalStaked > 0 && $.maxValidatorPercentage > 0) {
    uint256 newDelegatedAmount = $.validators[validatorId].delegatedAmount;  // already includes stake
    uint256 validatorPercentage = (newDelegatedAmount * 10_000) / $.totalStaked; // **post-stake** total
    if (validatorPercentage > $.maxValidatorPercentage) {
        revert ValidatorPercentageExceeded();
    }
}
```

`validatorPercentage` is computed using the post-stake `$.totalStaked` rather than the intended `previousTotalStaked`. Percentage enforcement runs only in staking / restaking paths. No check is done on `unstake`, `withdraw`, or slashing.

Because unstaking decreases `$.totalStaked` without rechecking percentages, an attacker can:

* stake a large temporary amount to another validator (inflate denominator),
* stake additional funds to the target validator (check passes),
* immediately unstake the temporary amount (denominator shrinks), causing the target validator’s share to silently exceed `maxValidatorPercentage`.

## Impact Details

Low — Contract fails to deliver promised decentralisation guarantees (a single validator can exceed the configured limit), but no direct loss of protocol value. Users delegating to that validator face higher correlated risk (slashing/downtime).

## Fix

Two suggested fixes (either is acceptable):

* Call `_validateValidatorPercentage` whenever `$.totalStaked` decreases (e.g., inside `_updateUnstakeAmounts`, `_removeParkedAmounts`, slash handlers, etc.) to ensure the invariant always holds.

OR

* Make the check monotonic by using the pre-stake total:

```solidity
uint256 validatorPercentage =
    (newDelegatedAmount * 10_000) / previousTotalStaked;  // use pre-stake total
```

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L156-L162>

## Proof of Concept

Assume `maxValidatorPercentage` = 3 000 bp (30 %).

{% stepper %}
{% step %}

### Initial state

* Validator A has 30 PLUME delegated
* `totalStaked` = 100 PLUME
* Validator A share = 30 %
  {% endstep %}

{% step %}

### Inflate denominator

* Attacker stakes 100 PLUME to Validator B.
* `totalStaked` = 200 PLUME
* Validator A share = 15 %
  {% endstep %}

{% step %}

### Stake to target validator

* Attacker stakes 30 PLUME to Validator A.
* The contract check uses post-stake total (230 PLUME) -> Validator A = 26.1 % (< 30 %), the stake passes.
  {% endstep %}

{% step %}

### Remove temporary stake

* Attacker immediately unstakes the 100 PLUME from Validator B.
* `totalStaked` = 130 PLUME
* Validator A now has 60 PLUME -> 46.2 % (> 30 %)
  {% endstep %}

{% step %}

### Result

No validation is triggered by the unstake. The cap is permanently violated until protocol intervention.
{% endstep %}
{% endstepper %}
