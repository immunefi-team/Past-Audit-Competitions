# 52931 sc high validators can not claim their commissions after the reward token removal&#x20;

**Submitted on Aug 14th 2025 at 11:49:48 UTC by @Slayer for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52931
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The `requestCommissionClaim` function prevents validators from claiming earned commission after a reward token has been removed from the system. While the `removeRewardToken` function correctly updates accrued validator commissions, the commission claim process requires the token to still be active, creating a permanent lock on validator funds.

## Vulnerability Details

When a reward token is removed:

```solidity
// RewardsFacet.sol:removeRewardToken()
for (uint256 i = 0; i < $.validatorIds.length; i++) {
    uint16 validatorId = $.validatorIds[i];

    // Final update to current time to settle all rewards up to this point
    PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId);

    // Create a final checkpoint with a rate of 0 to stop further accrual definitively
    PlumeRewardLogic.createRewardRateCheckpoint($, token, validatorId, 0);
}

// Update the mapping
$.isRewardToken[token] = false;
```

The function correctly:

* Settles all pending validator commission up to the removal timestamp
* Stores commission in `$.validatorAccruedCommission[validatorId][token]`
* Sets `$.isRewardToken[token] = false`

However, when validators try to claim this commission:

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
```

The `_validateIsToken` modifier implementation:

```solidity
modifier _validateIsToken(address token) {
    if (!PlumeStakingStorage.layout().isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }
    _;
}
```

Problem flow:

{% stepper %}
{% step %}

### Token removal and settling

1. Token is removed → `$.isRewardToken[token] = false`.
2. Validator commission is settled and stored in `$.validatorAccruedCommission[validatorId][token]`.
   {% endstep %}

{% step %}

### Claim attempt

3. Validator calls `requestCommissionClaim(validatorId, removedToken)`.
4. `_validateIsToken(removedToken)` checks `$.isRewardToken[removedToken]` → returns `false`.
5. Function reverts with `TokenDoesNotExist(token)`.
   {% endstep %}
   {% endstepper %}

While users can claim rewards from removed tokens because `RewardsFacet._validateTokenForClaim()` includes fallback logic:

```solidity
if (!isActive) {
    // Check if there are previously earned/stored rewards
    // ... fallback logic for removed tokens
    if (!hasRewards) {
        revert TokenDoesNotExist(token);
    }
}
```

validators have no such mechanism.

## Impact Details

Validators lose all accrued commission from removed reward tokens — effectively permanent freezing of those funds.

## Proof of Concept

{% stepper %}
{% step %}

1. Call `removeRewardToken(...)` → `$.isRewardToken[token] = false`.
2. Validator commission is updated and stored in `$.validatorAccruedCommission[validatorId][token]`.
   {% endstep %}

{% step %}
3\. Validator calls `requestCommissionClaim(validatorId, removedToken)`.
4\. `_validateIsToken(removedToken)` checks `$.isRewardToken[removedToken]` → returns `false`.
5\. Function reverts with `TokenDoesNotExist(token)`, blocking the claim.
{% endstep %}
{% endstepper %}

## References

(Add any relevant links to documentation or code)
