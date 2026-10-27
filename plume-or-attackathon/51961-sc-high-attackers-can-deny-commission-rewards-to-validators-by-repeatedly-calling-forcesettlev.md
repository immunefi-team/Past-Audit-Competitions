# 51961 sc high attackers can deny commission rewards to validators by repeatedly calling forcesettlevalidatorcommission&#x20;

* Report ID: #51961
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* Submitted on: Aug 6th 2025 at 21:02:21 UTC by @KlosMitSoss
* Context: Attackathon | Plume Network — <https://immunefi.com/audit-competition/plume-network-attackathon>

## #51961 \[SC-High] Attackers can deny commission rewards to validators by repeatedly calling forceSettleValidatorCommission()

### Brief / Intro

An attacker can repeatedly call ValidatorFacet::forceSettleValidatorCommission() with a low timeDelta to cause the commission accrual to always round down to zero in certain scenarios. As a result, the affected validators will never accrue any commission.

### Vulnerability Details

When ValidatorFacet::forceSettleValidatorCommission() is called, it triggers settlement of accrued commission for a specific validator. This updates the validator's cumulative reward-per-token indices (for all reward tokens) and their accrued commission storage. It uses the validator's current commission rate for settlement.

Within this function, PlumeRewardLogic::\_settleCommissionForValidatorUpToNow() is called. This calls PlumeRewardLogic::updateRewardPerTokenForValidator() for every reward token. Here, grossRewardForValidatorThisSegment is calculated using the following formula:

```
totalStaked * timeDelta * effectiveRewardRate / PlumeStakingStorage.REWARD_PRECISION
```

This value is then used for calculating the commissionDeltaForValidator:

```
grossRewardForValidatorThisSegment * commissionRateForSegment / PlumeStakingStorage.REWARD_PRECISION
```

This leads to the following formula for the accrued commission:

```
totalStaked * timeDelta * effectiveRewardRate / PlumeStakingStorage.REWARD_PRECISION 
* commissionRateForSegment / PlumeStakingStorage.REWARD_PRECISION
```

The validatorAccruedCommission\[validatorId]\[token] (the commission that can later be claimed) is then increased by commissionDeltaForValidator.

Example scenario:

* totalStaked = 1e9
* effectiveRewardRate = 1\_587\_301\_587 (value used in tests)
* commissionRateForSegment = 5e16 (5%)

When a user calls ValidatorFacet::forceSettleValidatorCommission() every 12 seconds, the calculation rounds down to zero:

```
1e9 * 12 * 1_587_301_587 / 1e18 * 5e16 / 1e18 = 0.95
```

As a result, anyone can exploit this by repeatedly calling ValidatorFacet::forceSettleValidatorCommission(). This prevents the validator from ever accruing any commission as long as the configuration remains the same. This also works for different rate values and different totalStaked amounts; this is just one example.

### Impact Details

Validators will never accrue any commission if their different rates and totalStaked amounts produce values that round down to zero for a low timeDelta.

### Proof of Concept

{% stepper %}
{% step %}

### Step

Call ValidatorFacet::forceSettleValidatorCommission() to manually trigger settlement of accrued commission for a specific validator.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L745-L756>
{% endstep %}

{% step %}

### Step

Within the function, PlumeRewardLogic::\_settleCommissionForValidatorUpToNow() is called, which calls PlumeRewardLogic::updateRewardPerTokenForValidator().

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L794-L802>
{% endstep %}

{% step %}

### Step

The accrued commission is calculated and rounded down. In certain scenarios (as described above), depending on totalStaked, the reward rate, and commission rate, this rounds down to zero.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L175-L187>
{% endstep %}

{% step %}

### Step

An attacker repeatedly calls ValidatorFacet::forceSettleValidatorCommission() every block (or frequently enough) such that the calculation always rounds down to zero. The validator will never accrue any commission as long as the values remain the same.
{% endstep %}
{% endstepper %}

### References

* ValidatorFacet::forceSettleValidatorCommission(): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L745-L756>
* PlumeRewardLogic::\_settleCommissionForValidatorUpToNow() / updateRewardPerTokenForValidator(): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L794-L802>
* Accrued commission calculation: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L175-L187>
