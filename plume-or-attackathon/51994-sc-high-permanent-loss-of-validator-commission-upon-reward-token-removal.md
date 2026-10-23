# 51994 sc high permanent loss of validator commission upon reward token removal

**Submitted on Aug 7th 2025 at 06:47:24 UTC by @light279 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51994
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

The Plume Staking contract allows validators to earn commission in reward tokens and later request commission claims through `ValidatorFacet::requestCommissionClaim`. However, a vulnerability exists where if a reward token is removed via `RewardsFacet::removeRewardToken` before the validator claims their pending accrued commission, the validator permanently loses access to those unclaimed funds.

### Vulnerability Details

When a validator admin calls `ValidatorFacet::requestCommissionClaim`, the function verifies that the specified token is still a valid reward token using the `ValidatorFacet::_validateIsToken` modifier. This checks `PlumeStakingStorage.layout().isRewardToken[token]`, which must be `true`.

`ValidatorFacet::requestCommissionClaim`:

```javascript
function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)
@>        _validateIsToken(token)
    {
.......................
```

`ValidatorFacet::_validateIsToken`:

```javascript
    modifier _validateIsToken(address token) {
@>        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }
```

However, in `RewardsFacet::removeRewardToken`, once a token is removed:

* Reward checkpoints are finalized correctly.
* The token’s rate is set to 0.
* `isRewardToken[token]` is set to `false`.

`RewardsFacet::removeRewardToken`:

```javascript
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
            PlumeRewardLogic.updateRewardPerTokenForValidator(
                $,
                token,
                validatorId
            );

            // Create a final checkpoint with a rate of 0 to stop further accrual definitively.
            PlumeRewardLogic.createRewardRateCheckpoint(
                $,
                token,
                validatorId,
                0
            );
        }

        // Set rate to 0 to prevent future accrual. This is now redundant but harmless.
        $.rewardRates[token] = 0;
        // DO NOT delete global checkpoints. Historical data is needed for claims.
        // delete $.rewardRateCheckpoints[token];

        // Update the array
        $.rewardTokens[tokenIndex] = $.rewardTokens[$.rewardTokens.length - 1];
        $.rewardTokens.pop();

        // Update the mapping
@>        $.isRewardToken[token] = false;

        delete $.maxRewardRates[token];
        emit RewardTokenRemoved(token);
    }
```

After removal, the token becomes permanently ineligible for claiming via `requestCommissionClaim` because `_validateIsToken` will revert due to the token no longer being recognized as valid.

There is no mechanism to check if any validator has unclaimed commission accrued for the token being removed. As a result, any validator who has not claimed their commission before the token is removed loses access to it forever.

Even if `$.isRewardToken[token]` is set to `false`, the validator admin should still be allowed to claim their accrued commission, since the rewards were earned prior to the token’s removal and they have the appropriate permission to claim them.

## Impact Details

This vulnerability results in permanent loss of validator commission for any validator who failed to claim rewards in a timely manner before token removal.

{% hint style="danger" %}
Permanent freezing of funds: unclaimed commission for removed reward tokens cannot be claimed due to the token no longer being recognized as valid.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}
Assume `TOKEN_A` is an active reward token.
{% endstep %}

{% step %}
Validator `V1` has accrued 100 `TOKEN_A` as commission but has not yet called `requestCommissionClaim`.
{% endstep %}

{% step %}
`removeRewardToken(TOKEN_A)` is called:

* Commission accrual stops.
* `TOKEN_A` is removed from `rewardTokens[]`.
* `isRewardToken[TOKEN_A]` is set to `false`.
  {% endstep %}

{% step %}
Validator `V1` calls `requestCommissionClaim(V1, TOKEN_A)`:

* `_validateIsToken(TOKEN_A)` fails because the token is no longer recognized.
* Transaction reverts with `TokenDoesNotExist(TOKEN_A)`.

Result: validator V1 is permanently unable to claim the 100 `TOKEN_A` commission.
{% endstep %}
{% endstepper %}
