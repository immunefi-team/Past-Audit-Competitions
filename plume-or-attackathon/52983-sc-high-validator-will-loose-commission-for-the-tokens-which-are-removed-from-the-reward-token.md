# 52983 sc high validator will loose commission for the tokens which are removed from the reward tokens but they still have commission left to be claimed&#x20;

**Submitted on Aug 14th 2025 at 14:54:39 UTC by @swarun for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52983
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

Claiming commission is not allowed for historical tokens because of an incorrect modifier which prevents validators from receiving the unclaimed commission.

### Vulnerability Details

When a reward token is removed its pending commission is calculated and updated for validators (so they should be able to claim it). However, an incorrect modifier applied to the commission claim function prohibits claiming commission for non-reward tokens, thereby preventing claims for historical tokens.

### Impact Details

Validators lose commission they are eligible to claim even though they are not slashed.

### References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L508>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L129>

## Proof of Concept

{% stepper %}
{% step %}

### Step: Removing a reward token updates final checkpoints and validator commission

When a reward token is removed, the contract updates rewards and commissions for validators up to removal time, then removes the token from the reward tokens set:

```solidity
function removeRewardToken(
        address token
    ) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        if (!$.isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }

        // Find the index of the token in the array
        uint256 tokenIndex = _getTokenIndex(token);

        // Store removal timestamp to prevent future accrual
        uint256 removalTimestamp = block.timestamp;
        $.tokenRemovalTimestamps[token] = removalTimestamp;

        // Update validators (bounded by number of validators, not users)
        for (uint256 i = 0; i < $.validatorIds.length; i++) {
            uint16 validatorId = $.validatorIds[i];

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

The loop calls update functions that settle commission for validators for that token. See reward logic references:

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L190>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L135>
  {% endstep %}

{% step %}

### Step: Validator requests commission claim for that token

After removal, a validator should be able to request the commission that was accrued up to the removal timestamp. The following function is used to request commission claims:

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
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

        if (!validator.active || validator.slashed) {
            revert ValidatorInactive(validatorId);
        }

        // Settle commission up to now to ensure accurate amount
        PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);

        uint256 amount = $.validatorAccruedCommission[validatorId][token];
        if (amount == 0) {
            revert InvalidAmount(0);
        }
        if ($.pendingCommissionClaims[validatorId][token].amount > 0) {
            revert PendingClaimExists(validatorId, token);
        }
        address recipient = validator.l2WithdrawAddress;
        uint256 nowTs = block.timestamp;
        $.pendingCommissionClaims[validatorId][token] = PlumeStakingStorage.PendingCommissionClaim({
            amount: amount,
            requestTimestamp: nowTs,
            token: token,
            recipient: recipient
        });
        // Zero out accrued commission immediately
        $.validatorAccruedCommission[validatorId][token] = 0;

        emit CommissionClaimRequested(validatorId, token, recipient, amount, nowTs);
    }
```

This function settles commission and attempts to create a pending claim for the token.
{% endstep %}

{% step %}

### Step: Request fails due to token validation modifier

The request fails because of the \_validateIsToken modifier which requires the token to be an active reward token:

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

Since isRewardToken\[token] was set to false during removal, the modifier reverts, preventing validators from claiming commission for that (historical) token — despite commission having been accrued and settled for them. This results in permanent loss of unclaimed commission.
{% endstep %}
{% endstepper %}
