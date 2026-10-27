# 52955 sc high a commission rate checkpoint is not created when adding a validator despite the commission rate being set leading to loss of validator commission&#x20;

* **Submitted on:** Aug 14th 2025 at 13:32:59 UTC by @frolic for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52955
* **Report Type:** Smart Contract
* **Severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impact summary:** Permanent freezing/loss of validator commission

## Brief / Intro

When adding a new validator via the `addValidator` function in `ValidatorFacet`, reward rate checkpoints are created for each reward token but no commission rate checkpoint is created even when a nonzero commission is set on the validator. This causes the validator's commission history to be empty from inception, resulting in zero commission being applied until the first explicit commission change — leading to lost commission funds.

## Vulnerability Details

In `addValidator`, reward rate checkpoints are initialized for each reward token using the current global rate:

```solidity
for (uint256 i = 0; i < rewardTokens.length; i++) {
    address token = rewardTokens[i];
    $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
    uint256 currentGlobalRate = $.rewardRates[token];
    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, currentGlobalRate);
}
```

However, there is no corresponding call to `createCommissionRateCheckpoint` to record the validator's starting commission. The validator struct does set the commission value:

```solidity
         validator.validatorId = validatorId;
@>       validator.commission = commission;  // <--- validator commission
         validator.delegatedAmount = 0;
```

Because no commission checkpoint is created at initialization, the checkpoint array for the validator is empty. Commission calculations use checkpointed history; an empty checkpoint history effectively results in a zero commission rate for the initial period. Thus the validator accrues no commission until a later explicit commission update creates a checkpoint.

Additionally, `setValidatorCommission` has an early return when the new commission equals the old commission, preventing creation of a checkpoint if one does not already exist but the desired starting commission equals the stored value:

```solidity
function setValidatorCommission(
        uint16 validatorId,
        uint256 newCommission
    ) external onlyValidatorAdmin(validatorId) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

        //...................
        // If the commission rate is the same, there's nothing to do.
        if (oldCommission == newCommission) {
            return;
        }
```

This means a validator whose commission was set at creation cannot force the creation of the missing checkpoint by calling `setValidatorCommission` if they attempt to set the same value — the function will return early and leave the checkpoint array empty.

## Impact Details

* No commission checkpoint at validator initialization causes the validator's commission rate to be treated as zero for the initial period.
* Validator accrues no commission on rewards distributed before the first commission change that creates a checkpoint.
* Commission that should have been accrued during this period is permanently lost.
* Protocol commission accounting is incorrect from inception for affected validators until corrected.

## Proof of Concept

{% stepper %}
{% step %}

### Step: Deploy and add validator

* Deploy the protocol and call `addValidator` with a nonzero commission rate (e.g., 10%).
* Observation: reward rate checkpoints are created, but no commission rate checkpoint is created for the validator.
  {% endstep %}

{% step %}

### Step: Users stake and earn rewards

* Multiple users stake tokens delegated to the new validator.
* Rewards are distributed over several blocks; stakers accrue rewards as expected.
  {% endstep %}

{% step %}

### Step: Validator attempts to claim commission

* The validator calls `requestCommissionClaim` for a reward token.
* The contract settles commission, but because no commission checkpoint exists for the initial period, the commission rate defaults to zero.
* Result: validator's accrued commission is zero despite user rewards.
  {% endstep %}

{% step %}

### Step: Claim reverts

* The call to `requestCommissionClaim` reverts with `InvalidAmount(0)` because the validator has no accrued commission to claim.
* Outcome: validator receives no commission for the entire period before the first explicit commission rate change — commission funds effectively lost.
  {% endstep %}
  {% endstepper %}

## References

* ValidatorFacet.sol - addValidator function: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/ValidatorFacet.sol#L147>
* PlumeRewardLogic.sol - createCommissionRateCheckpoint: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L757>
