# 50425 sc high active non slashed validators cannot claim rewards when a reward token is disabled

**Submitted on Jul 24th 2025 at 13:30:05 UTC by @oxrex for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50425
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

When a reward token is disabled, the intent is to allow validators as well as users to claim the accrued rewards up until the timestamp the disabled token became disabled. However, for validators there exists an edge case that is not handled correctly.

### Vulnerability Details

When a reward token is disabled, no further `requestCommissionClaim()` function call will be possible for all validators because the function uses the modifier `_validateIsToken` which ensures the `isRewardToken[token]` bool for the reward token in question must be true, otherwise it reverts. Since, during deactivation of a reward token, the contract:

{% stepper %}
{% step %}

### Step: Accrue final rewards for validators

Accrue all rewards up until block timestamp for all validators and save it in each validator's `validatorAccruedCommission[validatorId][token]`.
{% endstep %}

{% step %}

### Step: Disable the reward token

Set the `isRewardToken[token]` bool to false for the said token.
{% endstep %}
{% endstepper %}

It will now not be possible for validators to request commission claim for the accrued commissions because `requestCommissionClaim` will revert with the message: `TokenDoesNotExist`.

Relevant snippet from ValidatorFacet.sol:

```solidity
modifier _validateIsToken(
    address token
) {
    if (!PlumeStakingStorage.layout().isRewardToken[token]) {
        revert TokenDoesNotExist(token);
    }
    _;
}

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

Relevant snippet from RewardsFacet.sol showing token removal flow:

```solidity
function removeRewardToken(
    address token
) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
    ...

    // Update validators (bounded by number of validators, not users)
    for (uint256 i = 0; i < $.validatorIds.length; i++) {
        uint16 validatorId = $.validatorIds[i];

        // Final update to current time to settle all rewards up to this point
        PlumeRewardLogic.updateRewardPerTokenForValidator($, token, validatorId); // updates validator rewards into `validatorAccruedCommission[validatorId][token]`

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
    $.isRewardToken[token] = false; // marks the token status as false

    delete $.maxRewardRates[token];
    emit RewardTokenRemoved(token);
}
```

## Impact Details

When the reward token is deactivated, accrued rewards that were calculated and updated for each of the validators cannot be requested, and calling `forceSettleValidatorCommission()` does not fix it because that function only accrues rewards for the validators without actually sending out the rewards.

Users are able to claim accrued rewards for the tokens, but validators cannot — even though they are still active / not slashed and should be able to claim the accrued commission for the period up to deactivation.

Suggested fix (described by reporter): allow validators to request claim for accrued rewards up to the timestamp the token was deactivated. Concretely, modify the request logic to allow requests when the validator is active, not slashed, the reward token is deactivated, and `validatorAccruedCommission[validatorId][token] != 0`.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol?utm\\_source=immunefi#L126-L133>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol?utm\\_source=immunefi#L508>

## Proof of Concept

Add the PoC test below in the `PlumeStakingDiamondTest` test file and run the test with verbosity of 3 (`-vvv`). You can comment in the `vm.expectRevert();` to actually see the test failing in the console if desired.

```solidity
function testAfterRewardTokenRemovedValidatorCommissionLoss() public {

    address targetAdmin40 = makeAddr("p_targetAdmin40");
    vm.label(targetAdmin40, "p_targetAdmin40");
    vm.deal(targetAdmin40, 0.000001 ether); // for paying gas by validator admin

    vm.startPrank(admin);

    // Add validators for testing
    ValidatorFacet(address(diamondProxy)).addValidator(
        40,
        5e16,
        targetAdmin40,
        targetAdmin40,
        "target40",
        "target40",
        address(0x40),
        1_000_000 ether
    );
    vm.stopPrank();

    address staker1 = makeAddr("staker1");
    address staker2 = makeAddr("staker2");

    vm.deal(staker1, 501_000e18);
    vm.deal(staker2, 501_000e18);

    vm.startPrank(staker1);
    StakingFacet(payable(address(diamondProxy))).stake{
        value: 500_000e18}(40);
    vm.stopPrank();

    vm.startPrank(staker2);
    StakingFacet(payable(address(diamondProxy))).stake{
        value: 500_000e18}(40);
    vm.stopPrank();

    vm.warp(block.timestamp + 86400);
    vm.roll(block.number + 7200);
    console2.log("Time: ", block.timestamp);

    vm.startPrank(admin);
    RewardsFacet(address(diamondProxy)).removeRewardToken(PLUME_NATIVE);
    vm.stopPrank();

    vm.warp(block.timestamp + 86400);
    vm.roll(block.number + 7200);
    console2.log("Time: ", block.timestamp);

    vm.startPrank(targetAdmin40);
    vm.expectRevert(); // the function will revert here in this next call
    ValidatorFacet(address(diamondProxy)).requestCommissionClaim(40, PLUME_NATIVE);
    vm.stopPrank();

    vm.startPrank(targetAdmin40);
    ValidatorFacet(address(diamondProxy)).forceSettleValidatorCommission(40);
    vm.stopPrank();

    console2.log("Validator balance in PLUME: ", targetAdmin40.balance); // only the gas amount the validator admin/receiver had before is still there. no new PLUME received
}
```
