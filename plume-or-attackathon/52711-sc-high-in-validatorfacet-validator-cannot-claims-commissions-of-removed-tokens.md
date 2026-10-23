# 52711 sc high in validatorfacet validator cannot claims commissions of removed tokens

**Submitted on Aug 12th 2025 at 15:17:16 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52711
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

When a reward token is removed by an account with REWARD\_MANAGER\_ROLE, validator admins can no longer call `requestCommissionClaim()` for that token because the function requires the token to still be “active.”

These requirement blocks claiming accrued validator commissions for removed tokens.

### Vulnerability Details

`ValidatorFacet::requestCommissionClaim()` is gated by the `_validateIsToken(token)` modifier, which requires the token to be currently active.

When the reward manager removes a token, `isRewardToken[token]` becomes **false**, blocking the claim path even though commission was already accrued and properly settled at removal time.

There is an asymmetry with stakers: `RewardsFacet::_validateTokenForClaim()` explicitly allows claims for removed tokens if there are stored or calculable rewards. Validator commissions, however, are blocked by `_validateIsToken`. So users (stakers) can claim after removal; validators cannot.

### Impact Details

In production, this causes permanent loss of validator yield (or at least indefinite lock-up), making the issue high severity.

## Recommended fix

1. Allow claims for removed tokens (preferred) by loosening the check so it also accepts “historical” tokens when there is accrued commission.
2. Alternatively, when calling `removeRewardToken()` automatically call `requestCommissionClaim()` for each validator with a non-zero accrued amount.

## Proof of Concept

{% stepper %}
{% step %}

### Step

REWARD\_MANAGER\_ROLE calls `removeRewardToken(tokenX)`, which updates rewards for each validator, creates a checkpoint, then sets `$.isRewardToken[tokenX] = false` and removes `tokenX` from `$.rewardTokens[]`.

Example excerpt:

```solidity
function removeRewardToken(
    address token
) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
  ....
        // Final update to current time to settle all rewards up to this point
        PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId);

        // Create a final checkpoint with a rate of 0 to stop further accrual definitively.
        PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, 0);
    }
     
    // Set rate to 0 to prevent future accrual. This is now redundant but harmless.
    $.rewardRates[token] = 0;
    // DO NOT delete global checkpoints. Historical data is needed for claims.
    // delete $.rewardRateCheckpoints[token];

    // Update the array
    $.rewardTokens[tokenIndex] = $.rewardTokens[$.rewardTokens.length - 1];
    $.rewardTokens.pop();

    // Update the mapping
    $.isRewardToken[token] = false;

    delete $.maxRewardRates[token];
    emit RewardTokenRemoved(token);
}
```

{% endstep %}

{% step %}

### Step

A validator admin attempts to claim by calling `requestCommissionClaim`:

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
{ ... }
```

{% endstep %}

{% step %}

### Step

The `_validateIsToken` modifier reverts the call because `isRewardToken[token]` was set to `false` during removal:

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

There appears to be no alternative path in `ValidatorFacet` to claim validator's accrued commission for removed tokens.
{% endstep %}
{% endstepper %}
