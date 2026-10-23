# 50490 sc high user loses reward tokens during validator user relationship clearing

**Submitted on Jul 25th 2025 at 10:38:44 UTC by @oxrex for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50490
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Validator-user relationship for reward tokens not yet claimed but accrued from last user interaction timestamp up until validator slash timestamp will be lost once validator-user record is cleared as the user's stake will now be 0 and multiplying pending rewards by 0 would yield 0.

### Vulnerability Details

`userValidatorStakes[user][slashedValidatorId].staked` tracks how much the user stakes to a validator and thus is used for reward calculation for the user based on the staked amount. However:

{% stepper %}
{% step %}

### Step

If a user last claimed rewards at timestamp 1,576,800,000
{% endstep %}

{% step %}

### Step

And the validator became slashed at timestamp 1,576,972,800
{% endstep %}

{% step %}

### Step

And the user-validator relationship was cleared at timestamp 1,577,145,600
{% endstep %}

{% step %}

### Step

The user attempting to now claim rewards accrued between step 1 and timestamp 1,576,972,800 - 1, will fail because the `userValidatorStakes[user][slashedValidatorId].staked` now holds no stakes for the validator in question which is a result of not checking if `userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token]` is lesser than the `validator.slashedAtTimestamp` during `adminClearValidatorRecord()` function call
{% endstep %}
{% endstepper %}

Relevant code excerpt:

```solidity
 function adminClearValidatorRecord(
        address user,
        uint16 slashedValidatorId
    ) external onlyRole(PlumeRoles.ADMIN_ROLE) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        if (user == address(0)) {
            revert ZeroAddress("user");
        }
        if (!$.validatorExists[slashedValidatorId]) {
            revert ValidatorDoesNotExist(slashedValidatorId);
        }
        if (!$.validators[slashedValidatorId].slashed) {
            revert ValidatorNotSlashed(slashedValidatorId);
        }

        uint256 userActiveStakeToClear = $.userValidatorStakes[user][slashedValidatorId].staked;
        PlumeStakingStorage.CooldownEntry storage cooldownEntry = $.userValidatorCooldowns[user][slashedValidatorId];
        uint256 userCooledAmountToClear = cooldownEntry.amount;

        bool recordChanged = false;

        if (userActiveStakeToClear > 0) {
@>            $.userValidatorStakes[user][slashedValidatorId].staked = 0;
            // Decrement user's global stake
            if ($.stakeInfo[user].staked >= userActiveStakeToClear) {
                $.stakeInfo[user].staked -= userActiveStakeToClear;
            } else {
                $.stakeInfo[user].staked = 0; // Should not happen if state is consistent
            }
            emit AdminClearedSlashedStake(user, slashedValidatorId, userActiveStakeToClear);
            recordChanged = true;
        }

        ...

        if ($.userHasStakedWithValidator[user][slashedValidatorId] || recordChanged) {
            PlumeValidatorLogic.removeStakerFromValidator($, user, slashedValidatorId);
        }
    }
```

### Impact Details

Since the validator-user relationship is now cleared, the users will lose reward tokens that were not yet claimed up until the timestamp the slash occurred. Also, the user's `userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] = block.timestamp` will now point to the block.timestamp for an unpaid amount.

The suggested fix is to add an `if` check inside the `adminClearValidatorRecord()` function that verifies whether the user has already had their rewards accounted up to (or beyond) the slash timestamp, for example:

```solidity
 if (userValidatorRewardPerTokenPaidTimestamp[user][validatorId][token] < validator.slashedAtTimestamp) { revert(); }
```

This check guarantees that if the user's reward-per-token paid timestamp is less than the slash timestamp, the user hasn't claimed rewards up until the timestamp of the slash yet. When they do claim rewards, `userValidatorRewardPerTokenPaidTimestamp` will point to either the slash timestamp (if claimed in the same block) or above it (if claimed afterwards).

The PoC provided in the PoC section can be pasted in the PlumeStakingDiamond.t.sol test file and be run with verbosity of 3 (-vvv) to see the resulting reward balance of the users as the rewards were unable to be claimed.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ManagementFacet.sol#L389-L445>

## Proof of Concept

```solidity
function testLostRewardsForUserAfterClearRelationship() public {

        address targetAdmin40 = makeAddr("p_targetAdmin40");
        address voterAdmin41 = makeAddr("p_voterAdmin41");
        vm.label(targetAdmin40, "p_targetAdmin40");
        vm.label(voterAdmin41, "p_voterAdmin41");
        vm.label(targetAdmin40, "targetAdmin40");
        vm.label(voterAdmin41, "voterAdmin41");
        vm.deal(targetAdmin40, 1 ether);
        vm.deal(voterAdmin41, 1 ether);

        vm.startPrank(admin);
        ManagementFacet(payable(address(diamondProxy))).setMaxSlashVoteDuration(
            1 days
        );

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
        ValidatorFacet(address(diamondProxy)).addValidator(
            41,
            5e16,
            voterAdmin41,
            voterAdmin41,
            "voter41",
            "voter41",
            address(0x41),
            1_000_000 ether
        );
        vm.stopPrank();

        address staker1 = makeAddr("staker1");
        address staker2 = makeAddr("staker2");

        vm.deal(staker1, 500_000e18);
        vm.deal(staker2, 500_000e18);

        vm.startPrank(staker1);
        StakingFacet(payable(address(diamondProxy))).stake{
            value: 500_000e18}(40);
        vm.stopPrank();

        vm.startPrank(staker2);
        StakingFacet(payable(address(diamondProxy))).stake{
            value: 500_000e18}(40);
        vm.stopPrank();

        // reward accrue for 2 days before slashing vote begins
        vm.warp(block.timestamp + 172800);
        vm.roll(block.number + 14400);

        // --- Test: Successful Slash ---
        // Cast votes from 3 different active validators
        vm.prank(voterAdmin41);
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
            40,
            block.timestamp + 1 hours
        );
        vm.prank(validatorAdmin); // Global validator admin from setup
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
            40,
            block.timestamp + 1 hours
        );
        vm.prank(user2); // another validator admin from setup
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
            40,
            block.timestamp + 1 hours
        );

        vm.warp(block.timestamp + 86400);
        vm.roll(block.number + 7200);

        vm.startPrank(admin);
        ManagementFacet(payable(address(diamondProxy))).adminClearValidatorRecord(staker1, 40);
        ManagementFacet(payable(address(diamondProxy))).adminClearValidatorRecord(staker2, 40);
        vm.stopPrank();

        vm.warp(block.timestamp + 43200);
        vm.roll(block.number + 3600);

        vm.startPrank(staker1);
        RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE);
        vm.stopPrank();

        vm.startPrank(staker2);
        RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE);
        vm.stopPrank();

        console2.log("Staker 1 balance in PLUME: ", staker1.balance);
        console2.log("Staker 2 balance in PLUME: ", staker2.balance);
    }
```
