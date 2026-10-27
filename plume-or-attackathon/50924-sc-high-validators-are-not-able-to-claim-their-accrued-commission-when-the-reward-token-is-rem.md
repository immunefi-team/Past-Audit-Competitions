# 50924 sc high validators are not able to claim their accrued commission when the reward token is removed&#x20;

* Submitted on Jul 29th 2025 at 17:54:04 UTC by @WinSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #50924
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

Validators are not able to claim their accrued commission if the reward token they earned has been removed by the protocol. This is because `requestCommissionClaim` uses the `_validateIsToken` modifier which reverts for removed tokens.

### Vulnerability Details

The relevant part of `requestCommissionClaim` in `ValidatorFacet`:

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

The `_validateIsToken` modifier:

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

Because the modifier requires `isRewardToken[token]` to be true, once an admin calls `removeRewardToken` (which sets `isRewardToken[token]` to false), the validator can no longer call `requestCommissionClaim` for that token — even if they previously accrued commission. The accrued commission would remain stuck (or eventually lost), as `requestCommissionClaim` cannot be executed and therefore a subsequent `finalizeCommissionClaim` cannot be initiated by the validator.

Note: user claims for removed tokens are still supported because `claim` calls `_validateTokenForClaim`, which allows claims for removed tokens if there are stored or calculable rewards:

```solidity
function _validateTokenForClaim(address token, address user) internal view returns (bool isActive) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

    isActive = $.isRewardToken[token];

    if (!isActive) {
        // If token is not active, check if there are previously earned/stored rewards
        // or pending rewards that can still be calculated
        uint16[] memory validatorIds = $.userValidators[user];
        bool hasRewards = false;

        for (uint256 i = 0; i < validatorIds.length; i++) {
            uint16 validatorId = validatorIds[i];

            // Check stored rewards
            if ($.userRewards[user][validatorId][token] > 0) {
                hasRewards = true;
                break;
            }

            // Check pending (calculable) rewards for removed tokens
            uint256 userStakedAmount = $.userValidatorStakes[user][validatorId].staked;
            if (userStakedAmount > 0) {
                (uint256 userRewardDelta,,) = PlumeRewardLogic.calculateRewardsWithCheckpointsView(
                    $, user, validatorId, token, userStakedAmount
                );
                if (userRewardDelta > 0) {
                    hasRewards = true;
                    break;
                }
            }
        }

        if (!hasRewards) {
            revert TokenDoesNotExist(token);
        }
    }
}
```

To mitigate, apply the same relaxed validation for validators so they can still request and finalize commission claims for tokens that have been removed but for which there are accrued commission amounts.

## Impact Details

A validator cannot claim accrued commission for a token after that token has been removed (since `requestCommissionClaim` cannot be called). This can result in permanent loss of validator commissions — classified here as Theft of unclaimed yield — severity: High.

## Proof of Concept

{% stepper %}
{% step %}
A validator sets a commission rate of 30%.
{% endstep %}

{% step %}
A user delegates tokens to this validator.
{% endstep %}

{% step %}
After some time the user accrues 1000 reward tokens. The validator's commission is 300 tokens, user gets 700.
{% endstep %}

{% step %}
The protocol removes this reward token (admin calls `removeRewardToken`).
{% endstep %}

{% step %}
The user calls `claim` and successfully receives their 700 tokens (user claims still work for removed tokens).
{% endstep %}

{% step %}
The validator tries to call `requestCommissionClaim` for the 300 tokens but it reverts due to `_validateIsToken` rejecting removed tokens. The validator cannot request/finalize the commission claim and thus loses those 300 tokens.
{% endstep %}
{% endstepper %}

## References

* Source line reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/ValidatorFacet.sol#L500>
