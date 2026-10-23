# 50409 sc high validator will lose comission

**Submitted on Jul 24th 2025 at 10:46:43 UTC by @shadowHunter for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50409
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

It seems that currently the contract does not provide any way for a Validator to claim commission on removed tokens (historical). This will cause a genuine Validator to lose commissions.

On removing a token, the contract correctly calls `updateRewardPerTokenForValidator` to calculate commission till now but fails to provide a way to claim this.

### Vulnerability Details

See Proof of Concept section.

### Impact Details

Validator will lose commission.

### Recommendation

Instead of using `_validateIsToken(token)` in `requestCommissionClaim`, check that token is not part of both historical and current reward tokens before rejecting.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Reward token `T1` exists.
{% endstep %}

{% step %}

### Step

`REWARD_MANAGER_ROLE` decides to remove it using `removeRewardToken`.
{% endstep %}

{% step %}

### Step

This calls `updateRewardPerTokenForValidator` to ensure commission till now is settled:

```solidity
for (uint256 i = 0; i < $.validatorIds.length; i++) {
    uint16 validatorId = $.validatorIds[i];

    // Final update to current time to settle all rewards up to this point
    PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId);

    // Create a final checkpoint with a rate of 0 to stop further accrual definitively.
    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, 0);
}
```

{% endstep %}

{% step %}

### Step

The implementation calculates and accumulates the validator commission:

```solidity
uint256 commissionDeltaForValidator = (
    grossRewardForValidatorThisSegment * commissionRateForSegment
) / PlumeStakingStorage.REWARD_PRECISION;

if (commissionDeltaForValidator > 0) {
    $.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator;
}
```

{% endstep %}

{% step %}

### Step

Validator `V1` has accrued commission of `X` amount on Token `T1`.
{% endstep %}

{% step %}

### Step

Post removal of `T1`, `V1` wants to claim the commission using `requestCommissionClaim` which calls `_validateIsToken(token)`:

```solidity
function requestCommissionClaim(
    uint16 validatorId,
    address token
)
    external
    onlyValidatorAdmin(validatorId)
    nonReentrant
    _validateValidatorExists(validatorId)
    _validateIsToken(token)
{
    ...
}
```

{% endstep %}

{% step %}

### Step

But `_validateIsToken(token)` reverts since token `T1` is now removed:

```solidity
modifier _validateIsToken(
    address token
) {
    if (!PlumeStakingStorage.layout().isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }
    _;
}
```

This causes the validator to be unable to claim the previously accrued commission.
{% endstep %}
{% endstepper %}
