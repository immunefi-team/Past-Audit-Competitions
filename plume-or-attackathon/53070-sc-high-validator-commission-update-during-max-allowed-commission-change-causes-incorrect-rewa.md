# 53070 sc high validator commission update during max allowed commission change causes incorrect reward calculations

**Submitted on Aug 14th 2025 at 18:52:05 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53070
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

### Brief / Intro

When the `ManagementFacet::setMaxAllowedValidatorCommission` function is called, it enforces the new maximum commission rate for all existing validators whose commission is above the new limit. However, due to the way the update is applied, delegator rewards after this change are calculated incorrectly for their staking period before the commission change, leading to inaccurate reward distribution.

```javascript
function setMaxAllowedValidatorCommission(
        uint256 newMaxRate
    ) external onlyRole(PlumeRoles.TIMELOCK_ROLE) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        // Max rate cannot be more than 50% (REWARD_PRECISION / 2)
        if (newMaxRate > PlumeStakingStorage.REWARD_PRECISION / 2) {
            revert InvalidMaxCommissionRate(newMaxRate, PlumeStakingStorage.REWARD_PRECISION / 2);
        }

        uint256 oldMaxRate = $.maxAllowedValidatorCommission;
        $.maxAllowedValidatorCommission = newMaxRate;

        emit MaxAllowedValidatorCommissionSet(oldMaxRate, newMaxRate);

        // Enforce the new max commission on all existing validators
        uint16[] memory validatorIds = $.validatorIds;
        for (uint256 i = 0; i < validatorIds.length; i++) {
            uint16 validatorId = validatorIds[i];
            PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

            if (validator.commission > newMaxRate) {
                uint256 oldCommission = validator.commission;

                // Settle commissions accrued with the old rate up to this point.
@>                PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);

                // Update the validator's commission rate to the new max rate.
@>                validator.commission = newMaxRate;

                // Create a checkpoint for the new commission rate.
                PlumeRewardLogic.createCommissionRateCheckpoint($, validatorId, newMaxRate);

                emit ValidatorCommissionSet(validatorId, oldCommission, newMaxRate);
            }
        }
    }
```

Note: This report completely differs from report #52315, as it concerns an entirely separate function `ManagementFacet::setMaxAllowedValidatorCommission`. Resolving #52315 will not address this issue.

## Vulnerability Details

{% stepper %}
{% step %}

### Step

The function iterates through all validators and, if their current commission exceeds the new max, it:

* Settles accrued commission with the old rate.
* Directly updates the validator’s commission to the new max rate.
* Creates a commission rate checkpoint.
  {% endstep %}

{% step %}

### Step

Reward calculations for delegators are based on their stake start time, not the actual time the commission rate was changed. Delegators who staked before the change should have rewards apportioned across the old and new commission rates, but the current logic may not achieve that.
{% endstep %}

{% step %}

### Step

When a delegator/user claims rewards for a time period ending at T, the claim flow is:

RewardsFacet::claim(address token,uint16 validatorId) → RewardsFacet::\_processValidatorRewards → PlumeRewardLogic.updateRewardsForValidatorAndToken → PlumeRewardLogic::calculateRewardsWithCheckpoints → PlumeRewardLogic::\_calculateRewardsCore

Inside `_calculateRewardsCore`, `getEffectiveCommissionRateAt` attempts to fetch the rate from checkpoints for the relevant period. Because the new checkpoint timestamp is at T and the claim’s reward period also ends at T, the lookup can fail to find an earlier applicable checkpoint. The code then falls back to using the current `validator.commission` stored in `ValidatorInfo`, which was already updated to the new reduced rate — not the historical rate that applied during part of the staking period.
{% endstep %}
{% endstepper %}

The relevant fallback in `PlumeRewardLogic::getEffectiveCommissionRateAt`:

```javascript
function getEffectiveCommissionRateAt(
        PlumeStakingStorage.Layout storage $,
        uint16 validatorId,
        uint256 timestamp
    ) internal view returns (uint256) {
        PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $
            .validatorCommissionCheckpoints[validatorId];
        uint256 chkCount = checkpoints.length;

        if (chkCount > 0) {
            uint256 idx = findCommissionCheckpointIndexAtOrBefore(
                $,
                validatorId,
                timestamp
            );
            if (idx < chkCount && checkpoints[idx].timestamp <= timestamp) {
                // Similar to above, ensure this is the latest one.
                return checkpoints[idx].rate;
            }
        }
        // Fallback to the current commission rate stored directly in ValidatorInfo
        // This is important if no checkpoints exist or all are in the future.
@>        uint256 fallbackComm = $.validators[validatorId].commission;
        return fallbackComm;
    }
```

This causes delegators to receive more rewards than they should when a validator's commission is reduced, because rewards for periods before the commission change are effectively computed with the new lower rate.

Note: This occurs when a validator already had delegators and then the Admin lowers the max allowed commission so that the validator's commission is reduced.

## Impact Details

{% hint style="warning" %}
This results in delegators receiving more rewards than they should, due to the validator’s commission being reduced. Impacts include theft of unclaimed yield and potential protocol insolvency.
{% endhint %}

## Proof of Concept

<details>

<summary>PoC steps (expand to view)</summary>

Initial StateMax allowed commission = 50%.Validator A’s commission = 45%.Delegator D stakes 100 tokens to Validator A at block T1.Rewards AccumulationBetween T1 and T2, rewards accrue under the 45% commission rate. Delegator D does not claim rewards yet.Commission Reduction via Max Allowed UpdateAdmin calls setMaxAllowedValidatorCommission(30%) at block T2.Since 45% > 30%, the contract:Settles validator’s commission up to T2 (accrued using 45%).Directly updates commission to 30%.Creates checkpoint at T2.Incorrect Reward CalculationWhen Delegator D later claims rewards, the system applies the new 30% commission rate for their original stake from time T1 → T2, so rewards for T1 → T2 are incorrectly overpaid because the commission was reduced.

</details>

## References / Affected Code

* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol>

(Keep all links and query parameters as in the original report.)

***

If you want, I can:

* Propose specific code fixes or patches to ensure historical periods use the correct commission rate (e.g., ensure checkpoints are created with timestamps strictly before the moment a commission update takes effect, avoid updating `validator.commission` until checkpoints guarantee historical lookups, or adjust the lookup logic to consider checkpoints with equal timestamps correctly).
* Generate a minimal test demonstrating the incorrect behavior.
