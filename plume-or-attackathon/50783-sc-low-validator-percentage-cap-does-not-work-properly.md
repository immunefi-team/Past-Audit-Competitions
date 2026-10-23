# 50783 sc low validator percentage cap does not work properly

**Submitted on Jul 28th 2025 at 14:39:50 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #50783
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The `maxValidatorPercentage` enforcement is unfair because it doesn't rebalance existing validators who exceeded the cap before it was introduced. Additionally, if the cap is raised to 100%, a single validator can hold 99.9% of the stake, since the cap is calculated based on `totalStaked`.

### Vulnerability Details

Validation of the validator percentage in `StakingFacet.sol:_validateValidatorPercentage()`:

```solidity
    function _validateValidatorPercentage(
        uint16 validatorId, uint256 stakeAmount
    ) internal view {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        uint256 previousTotalStaked = $.totalStaked - stakeAmount;

        // Check if exceeding validator percentage limit
        if (previousTotalStaked > 0 && $.maxValidatorPercentage > 0) {
            uint256 newDelegatedAmount = $.validators[validatorId].delegatedAmount;
@>            uint256 validatorPercentage = (newDelegatedAmount * 10_000) / $.totalStaked;
@>            if (validatorPercentage > $.maxValidatorPercentage) {
                revert ValidatorPercentageExceeded();
            }
        }
    }
```

* The check only prevents new stakes that would exceed the limit, but doesn't address existing validators who already exceed the percentage cap.
* Example scenario: System starts with `maxValidatorPercentage = 0`, Validator A accumulates 70% of total stake, then admin sets `maxValidatorPercentage = 50%`. Validator A remains at 70% while new validators can grow up to 50%, creating asymmetric percentages (70 + 50 = 120%).

## Impact Details

* Validators that exceeded the cap before enforcement can keep their oversized stake, giving them an unfair long-term advantage.
* Validators below the cap can still grow up to the cap, resulting in an asymmetric total where multiple validators can exceed 100% combined when some were grandfathered above the cap.

## Proof of Concept

{% stepper %}
{% step %}

### Initial state

* System starts with `maxValidatorPercentage = 0`.
* Validator A accumulates 70% of total stake.
  {% endstep %}

{% step %}

### After admin changes cap

* Admin sets `maxValidatorPercentage = 50%`.
* Validator A remains at 70% (no rebalance).
* Validator B (at 10%) can grow to 50%.
* Resulting effective percentages can sum to more than 100% (e.g., 70% + 50% = 120%).
  {% endstep %}
  {% endstepper %}

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L147-L162>
