# 50402 sc low single rate assumption ignores checkpoints in slashed case&#x20;

**Submitted on Jul 24th 2025 at 09:10:11 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50402
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Theft of unclaimed yield

## Description

In the slashed‑validator branch of `calculateRewardsWithCheckpoints`, the code reads a single rate at the moment of the last global update and applies it across the entire interval without honoring any intermediate checkpoints:

```solidity
PlumeStakingStorage.RateCheckpoint memory effectiveRewardRateChk =
    getEffectiveRewardRateAt($, token, validatorId, validatorLastUpdateTime);
uint256 effectiveRewardRate = effectiveRewardRateChk.rate;
uint256 rewardPerTokenIncrease = timeSinceLastUpdate * effectiveRewardRate;
currentCumulativeRewardPerToken += rewardPerTokenIncrease;
```

Because it never segments the window by the array of `validatorRewardRateCheckpoints`, any rate changes that occurred between `validatorLastUpdateTime` and the slash (or token removal) are completely ignored. In contrast, the unslashed path builds a `distinctTimestamps` array of every checkpoint and correctly applies each piecewise rate.

## Impact

Rewards are under- or over-paid whenever rates changed during the paused interval, leading to incorrect payouts and economic imbalance (theft or loss of unclaimed yield).

## Recommendation

Replace the single‑rate bump with the same segmented loop used in `_calculateRewardsCore`, or invoke a corrected `updateRewardPerTokenForValidator` up to the slash timestamp so that all stored checkpoints are applied and global state advances before per‑user calculations.

## Proof of Concept

<details>

<summary>Detailed PoC and test cases (expand to view)</summary>

```solidity
/**
     * @notice Direct comparison of slashed vs normal checkpoint handling
     * @dev Shows exactly how slashed validators bypass checkpoint segmentation
     */
    function test_CheckpointMechanismComparison() public {
        uint16 slashedValidatorId = DEFAULT_VALIDATOR_ID; // Will be slashed
        uint16 normalValidatorId = 1; // Will remain active
        address token = address(pUSD);
        
        console2.log("=== CHECKPOINT MECHANISM COMPARISON ===");
        
        // Setup both validators with identical conditions
        vm.prank(user1);
        StakingFacet(address(diamondProxy)).stake{value: 50e18}(slashedValidatorId);
        
        vm.prank(user2);
        StakingFacet(address(diamondProxy)).stake{value: 50e18}(normalValidatorId);
        
        uint256 startTime = block.timestamp;
        
        // Create identical rate change pattern for both validators
        uint256[3] memory rates = [uint256(2e15), 6e15, 1e15]; // Various rates
        uint256[3] memory periods = [uint256(1000), 1000, 1000]; // 1000 seconds each
        
        uint256 currentTime = startTime;
        uint256 expectedGrossReward = 0;
        
        for (uint256 i = 0; i < rates.length; i++) {
            // Apply rate change
            vm.prank(admin);
            RewardsFacet(address(diamondProxy)).setRewardRates(_addrArr(token), _uintArr(rates[i]));
            
            console2.log("Phase %s: Rate %s for %s seconds", i + 1, rates[i], periods[i]);
            
            // Calculate expected reward for this period
            expectedGrossReward += (periods[i] * rates[i] * 50e18) / 1e18;
            
            // Advance time
            vm.warp(currentTime + periods[i]);
            currentTime = block.timestamp;
        }
        
        console2.log("Total time: %s seconds", currentTime - startTime);
        console2.log("Expected gross reward per validator: %s", expectedGrossReward);
        
        // NOW: Slash one validator, keep other active
        vm.prank(user2); // user2 votes to slash validator 0
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(slashedValidatorId, currentTime + 1);
        
        // Small additional time for normal validator
        vm.warp(currentTime + 100);
        
        // Calculate rewards for both
        uint256 slashedValidatorReward = RewardsFacet(address(diamondProxy)).earned(user1, token);
        uint256 normalValidatorReward = RewardsFacet(address(diamondProxy)).earned(user2, token);
        
        console2.log("=== COMPARISON RESULTS ===");
        console2.log("Slashed validator reward: %s", slashedValidatorReward);
        console2.log("Normal validator reward: %s", normalValidatorReward);
        
        // Expected reward (after commission) for the 3000-second period
        uint256 expectedNetReward = (expectedGrossReward * 95) / 100;
        console2.log("Expected net reward: %s", expectedNetReward);
        
        // Calculate what single-rate bug would produce for slashed validator
        // Bug: Uses only the FIRST rate (2e15) for entire period
        uint256 singleRateBugReward = (3000 * rates[0] * 50e18 * 95) / (1e18 * 100);
        console2.log("Single-rate bug would give: %s", singleRateBugReward);
        
        console2.log("=== ANALYSIS ===");
        
        // Check if normal validator got proper segmented calculation
        uint256 normalDiff = normalValidatorReward > expectedNetReward ? 
            normalValidatorReward - expectedNetReward : expectedNetReward - normalValidatorReward;
        console2.log("Normal validator difference from expected: %s", normalDiff);
        
        // Check if slashed validator shows single-rate bug
        uint256 slashedDiffFromExpected = slashedValidatorReward > expectedNetReward ?
            slashedValidatorReward - expectedNetReward : expectedNetReward - slashedValidatorReward;
        uint256 slashedDiffFromBug = slashedValidatorReward > singleRateBugReward ?
            slashedValidatorReward - singleRateBugReward : singleRateBugReward - slashedValidatorReward;
            
        console2.log("Slashed validator difference from expected: %s", slashedDiffFromExpected);
        console2.log("Slashed validator difference from single-rate bug: %s", slashedDiffFromBug);
        
        if (slashedDiffFromBug < slashedDiffFromExpected) {
            console2.log("BUG CONFIRMED: Slashed validator used single rate assumption!");
            console2.log("While normal validator properly segmented by checkpoints");
            
            uint256 rewardError = expectedNetReward > slashedValidatorReward ?
                expectedNetReward - slashedValidatorReward : slashedValidatorReward - expectedNetReward;
            console2.log("Reward calculation error: %s tokens", rewardError);
            
            assertTrue(true, "Checkpoint bypass bug confirmed");
        } else {
            console2.log("BOTH VALIDATORS: Proper checkpoint segmentation used");
            assertTrue(false, "No bug - checkpoints properly honored in slashed case");
        }
        
        console2.log("=== TECHNICAL DETAILS ===");
        console2.log("Normal path: updateRewardPerTokenForValidator() + _calculateRewardsCore()");
        console2.log("- Uses getDistinctTimestamps() for proper segmentation");
        console2.log("- Each segment gets its own rate via getEffectiveRewardRateAt()");
        console2.log("");
        console2.log("Slashed path: Direct local calculation");
        console2.log("- Single getEffectiveRewardRateAt() call at validatorLastUpdateTime");
        console2.log("- Rate applied to entire timeSinceLastUpdate period");
        console2.log("- Completely bypasses checkpoint segmentation");
    }
    
    /**
     * @notice Demonstrates the exact vulnerable code pattern
     * @dev Reproduces the specific logic that causes the single rate assumption
     */
    function test_VulnerableCodePattern() public {
        uint16 validatorId = DEFAULT_VALIDATOR_ID;
        address token = address(pUSD);
        
        console2.log("=== VULNERABLE CODE PATTERN DEMO ===");
        
        vm.prank(user1);
        StakingFacet(address(diamondProxy)).stake{value: 100e18}(validatorId);
        
        // Establish initial rate and let time pass
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).setRewardRates(_addrArr(token), _uintArr(1e15));
        
        uint256 initialTime = block.timestamp;
        vm.warp(initialTime + 500); // Some time passes
        
        // Trigger an update to establish validatorLastUpdateTime
        vm.prank(user1);
        RewardsFacet(address(diamondProxy)).earned(user1, token);
        
        uint256 updateTime = block.timestamp;
        console2.log("Validator last update time established: %s", updateTime);
        
        // Change rate significantly
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).setRewardRates(_addrArr(token), _uintArr(10e15));
        console2.log("Rate changed from 1e15 to 10e15");
        
        // More time passes with new rate
        vm.warp(updateTime + 2000);
        
        // Another rate change
        vm.prank(admin);
        RewardsFacet(address(diamondProxy)).setRewardRates(_addrArr(token), _uintArr(5e15));
        console2.log("Rate changed again to 5e15");
        
        // More time passes
        vm.warp(block.timestamp + 1000);
        uint256 slashTime = block.timestamp;
        
        // Slash validator - this triggers the vulnerable code
        vm.prank(user2);
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(validatorId, slashTime + 1);
        
        console2.log("=== VULNERABLE CALCULATION SIMULATION ===");
        console2.log("Slash time: %s", slashTime);
        console2.log("Period to calculate: %s to %s (%s seconds)", updateTime, slashTime, slashTime - updateTime);
        
        // This simulates the vulnerable code pattern:
        // 1. Gets rate at validatorLastUpdateTime (the OLD rate)
        // 2. Applies it to entire period until slash
        console2.log("Vulnerable code would:");
        console2.log("1. Get rate at validatorLastUpdateTime (%s): 1e15", updateTime);
        console2.log("2. Apply this rate to entire %s second period", slashTime - updateTime);
        console2.log("3. Ignore all intermediate rate changes");
        
        uint256 actualReward = RewardsFacet(address(diamondProxy)).earned(user1, token);
        console2.log("Actual calculated reward: %s", actualReward);
        
        // What it SHOULD calculate (proper segmentation):
        // Period 1: updateTime to updateTime+2000 at rate 10e15
        // Period 2: updateTime+2000 to slashTime at rate 5e15
        uint256 period1Reward = (2000 * 10e15 * 100e18) / 1e18;
        uint256 period2Reward = (1000 * 5e15 * 100e18) / 1e18;
        uint256 correctGross = period1Reward + period2Reward;
        uint256 correctNet = (correctGross * 95) / 100;
        
        // What the bug produces (single rate):
        uint256 bugPeriod = slashTime - updateTime; // 3000 seconds
        uint256 bugGross = (bugPeriod * 1e15 * 100e18) / 1e18; // Uses OLD rate!
        uint256 bugNet = (bugGross * 95) / 100;
        
        console2.log("=== CALCULATION COMPARISON ===");
        console2.log("Correct (segmented): %s", correctNet);
        console2.log("Bug (single rate): %s", bugNet);
        console2.log("Actual result: %s", actualReward);
        
        if (actualReward < correctNet * 9 / 10) {
            console2.log("SEVERE UNDERPAYMENT: User lost %s tokens", correctNet - actualReward);
            console2.log("Loss percentage: %s%%", ((correctNet - actualReward) * 100) / correctNet);
        }
    }
```

</details>
