# 51530 sc high validators can not claim pending accrued commission when reward tokens have been removed from the isrewardtoken mapping&#x20;

**Submitted on Aug 3rd 2025 at 18:15:58 UTC by @Outliers for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51530
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

When a validator earns rewards, both the validator and their users receive a share. Users are allowed to claim their accrued rewards even if the associated reward tokens are no longer supported. However, validators currently **cannot** claim their share under the same conditions.

This creates an **asymmetric behavior** where users retain full access to their rewards, but validators may be permanently blocked from accessing theirs.

### Vulnerability Details

Relevant code excerpts:

```solidity
 /**
     * @notice Request a commission claim for a validator and token (starts timelock)
     * @dev Only callable by validator admin. Amount is locked at request time.
     */
    function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)

@audit>>>        _validateIsToken(token)                                                                     


    {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();                                      
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

        if (!validator.active || validator.slashed) {                                
            revert ValidatorInactive(validatorId);
        }

        // Settle the commission up to now to ensure an accurate amount
        PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);

@audit>>            uint256 amount = $.validatorAccruedCommission[validatorId][token];

@audit>>            if (amount == 0) {
            revert InvalidAmount(0);
        }
   

     if ($.pendingCommissionClaims[validatorId][token].amount > 0) {
            revert PendingClaimExists(validatorId, token);
        }
```

and the modifier:

```solidity
    modifier _validateIsToken(
        address token
    ) {

@AUDIT>>        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }
```

When users claim this check was removed to prevent this same vulnerability, but this was incorrectly introduced into the Validators Reward claiming action. Causing validators' funds to remain frozen for hours/days.

Because of the time sensitivity of Slashing, Validators should be able to request tokens as long as they have accrued rewards with no pending claim request.

This modifier incorrectly blocks the flow of claiming these tokens, which can be lost forever if:

* Reward tokens are no longer added to the mix.
* If users request at day 2 + wait time, they should get it by day 9, but because of this delay and reward tokens get added again at day 4 validator will have to wait till day 11 to redeem. Validator gets slashed at day 10, hence they will still eventually lose access to these Commission tokens. This example shows the time sensitivity of this critical function as if Validator laid claim at day 2 they will lose these funds.

See Users flow below showing they are not denied:

```solidity
  function claim(
        address token
    ) external nonReentrant returns (uint256) {
        // Validate token
        _validateTokenForClaim(token, msg.sender);

        // Process rewards from all active validators
        uint256 totalReward = _processAllValidatorRewards(msg.sender, token);

        // Finalize claim if there are rewards
        if (totalReward > 0) {
            _finalizeRewardClaim(token, totalReward, msg.sender);
            emit RewardClaimed(msg.sender, token, totalReward);
        }

        // Clear pending flags for all validators
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        uint16[] memory validatorIds = $.userValidators[msg.sender];
        _clearPendingRewardFlags(msg.sender, validatorIds);

        // Clean up validator relationships for validators with no remaining involvement
        PlumeValidatorLogic.removeStakerFromAllValidators($, msg.sender);

        return totalReward;
    }
    function _processValidatorRewards(
        address user,
        uint16 validatorId,
        address token
    ) internal returns (uint256 reward) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        // Settle pending rewards for this specific user/validator/token combination.
        // This updates both $.userRewards and $.totalClaimableByToken consistently.
        PlumeRewardLogic.updateRewardsForValidatorAndToken($, user, validatorId, token);

        // Now that rewards are settled, the full claimable amount is in storage.
        reward = $.userRewards[user][validatorId][token];

        if (reward > 0) {
            // This function will now only *reset* the user's reward to 0, since it's being claimed.
            _updateUserRewardState(user, validatorId, token);
            emit RewardClaimedFromValidator(user, token, validatorId, reward);
        }
    }
```

As seen above user flow ensures users can claim these types of tokens.

## Impact Details

This critical flaw affects the time-sensitive actions that deny Validators their ability to claim pending commissions. Any improper delays or delayed action until slashing actions can result in the total loss of these funds. If Reward tokens were temporarily removed, users would still have full rights to successfully claim their funds, but Validators would be incorrectly denied.

## References

Enable validators to claim all pending rewards unconditionally, or utilize historical reward token mapping to validate tokens instead of the current `isRewardToken` mapping.

## Proof of Concept

{% stepper %}
{% step %}

### PoC path 1: Token permanently removed

1. Rewards are explicitly accrued within the contracts.
2. The reward token is systematically removed from the `isRewardToken` mapping.
3. Users are entitled to claim their accrued rewards linked to these tokens.
4. Validators are unequivocally denied access to the commissions accrued on the claimed rewards.
5. Tokens are never re-added. Commissions are permanently frozen.
   {% endstep %}

{% step %}

### PoC path 2: Token temporarily removed with timing causing slashing loss

1. Rewards are accrued and a validator requests commission at time T (starts timelock).
2. The reward token is removed from `isRewardToken` after T.
3. Validator cannot request/claim due to `_validateIsToken` modifier.
4. Token may be re-added later, but timing causes validator to be slashed before they can claim (time-sensitive).
5. Validator loses ability to claim commissions despite accrual.
   {% endstep %}
   {% endstepper %}
