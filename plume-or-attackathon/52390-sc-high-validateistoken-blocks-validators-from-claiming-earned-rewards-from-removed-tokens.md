# 52390 sc high validateistoken blocks validators from claiming earned rewards from removed tokens&#x20;

**Submitted on Aug 10th 2025 at 12:01:48 UTC by @kaysoft for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52390
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

The `_validateIsToken(...)` modifier reverts for a reward token that has been removed.\
The `requestCommissionClaim(...)` function uses the `_validateIsToken(...)` modifier which will cause unclaimed rewards to revert for a token that has been removed.

## Vulnerability Details

The `requestCommissionClaim(...)` has a `_validateIsToken(...)` that immediately reverts if the reward token has been removed. Pending reward tokens for a removed token cannot be claimed by validators.

```solidity
File: ValidatorFacet.sol
 modifier _validateIsToken(
        address token
    ) {
        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }
```

Instead of reverting immediately in the above logic, check for unclaimed earned commissions like the `RewardsFacet.sol#_validateTokenForClaim(...)`.

```solidity
function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)
        _validateIsToken(token)//@audit-issue validator cannot claim a token that has been removed
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

## Impact Details

Validators cannot claim pending earned reward tokens for a removed reward token. This happens if the reward token was removed before the validator calls the `requestCommissionClaim(...)` function.\
All rewards earned up to the point of reward token removal before a validator calls `requestCommissionClaim(...)` are blocked.

{% hint style="danger" %}
Severity: High — Temporary freezing of funds for at least 24 hours.
{% endhint %}

## Recommendation

Implement a pending reward check for a removed token when `requestCommissionClaim(...)` function is called instead of the reverting that is done in the `_validateIsToken(...)`.

Refer to `RewardsFacet.sol#_validateTokenForClaim(...)` as an example of checking for pending/earned rewards for removed tokens prior to reverting.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

1. Admin deploys PlumeStaking contract and adds 2 reward tokens.
2. Bob has validator 1 on PlumeStaking.
3. Bob has earned 100 Plume reward tokens but has not requested claiming them with the `requestCommissionClaim(...)` function.
   {% endstep %}

{% step %}

### Trigger

4. Admin removes Plume token from reward tokens.
5. Bob calls the `requestCommissionClaim(...)` function to start the claiming process of his earned 100 Plume reward tokens but the transaction reverts due to the `_validateIsToken(...)` modifier.
   {% endstep %}

{% step %}

### Result

6. Bob cannot claim his earned 100 Plume reward tokens.
   {% endstep %}
   {% endstepper %}

## References

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* Suggested reference: RewardsFacet.sol#\_validateTokenForClaim(...) (used as suggested behaviour)
