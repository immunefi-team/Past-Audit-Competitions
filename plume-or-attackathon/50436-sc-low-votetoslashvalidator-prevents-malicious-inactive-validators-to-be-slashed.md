# 50436 sc low votetoslashvalidator prevents malicious inactive validators to be slashed&#x20;

**Submitted on Jul 24th 2025 at 15:50:57 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50436
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

A malicious validator cannot be slashed if their status is `inactive`.

### Vulnerability Details

The `voteToSlashValidator` function only allows active validators to be slashed.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L662-L664>

Relevant snippet:

```solidity
function voteToSlashValidator(uint16 maliciousValidatorId, uint256 voteExpiration) external nonReentrant {
    ...
    if (!targetValidator.active) {
        revert ValidatorInactive(maliciousValidatorId);
    }
    ...
```

Problem: a malicious validator can prevent other malicious validators from being slashed by skipping the vote process, since slashing requires unanimity of votes. To stop this, governance might set the skipping malicious validator to `inactive` so they cannot break the next round, but once inactive that validator becomes ineligible to be slashed — effectively shielded. They can then unstake/withdraw (after timelock) and exit the protocol with no consequences despite having behaved maliciously.

### Impact

Malicious validators can escape slashing with 100% of their funds while still acting maliciously at the network and protocol level (by skipping votes). This can lead to protocol insolvency because the protocol cannot punish malicious validators.

### Recommendation

Allow `voteToSlashValidator` to also target inactive validators. For example, remove the `active` check:

```diff
function voteToSlashValidator(uint16 maliciousValidatorId, uint256 voteExpiration) external nonReentrant {
    ...
-   if (!targetValidator.active) {
-       revert ValidatorInactive(maliciousValidatorId);
-   }
    ...
```

## Proof of Concept

{% stepper %}
{% step %}

### Context (step 1)

1. A malicious validator intentionally skips the voting process to protect another malicious validator from being slashed.
2. Governance sets this skipping malicious validator to `inactive` (to avoid them breaking future slashing rounds).
3. Once `inactive`, that validator cannot be slashed anymore due to the `active` check in `voteToSlashValidator` and can exit the protocol later with no consequences.
   {% endstep %}

{% step %}

### PoC test (step 2)

Add the following test in `PlumeStakingDiamond.t.sol`:

```solidity
function testMaliciousValidator_cannotBeSlashed_onceInactive() public {
    // Setup validators
    testSlash_Setup();

    // Set max slash vote duration
    vm.startPrank(admin);
    ManagementFacet(address(diamondProxy)).setMaxSlashVoteDuration(1 days);
    vm.stopPrank();

    // Create users and only stake funds with validator 0
    address user1_slash = makeAddr("user1_slash");
    address user2_slash = makeAddr("user2_slash");
    vm.deal(user1_slash, 100 ether);
    vm.deal(user2_slash, 100 ether);

    // user1 stakes with validator 0
    vm.startPrank(user1_slash);
    StakingFacet(address(diamondProxy)).stake{value: 10 ether}(
        DEFAULT_VALIDATOR_ID
    );
    vm.stopPrank();

    // user2 stakes with validator 1
    vm.startPrank(user2_slash);
    StakingFacet(address(diamondProxy)).stake{value: 10 ether}(
        1
    );
    vm.stopPrank();

    // Target validator to slash
    address voter1Admin = user2; // user2 is admin for validator1
    uint16 voter0ValidatorId = 0;
    address voter0Admin = validatorAdmin; // validatorAdmin is admin for validator0

    vm.startPrank(voter1Admin);
    uint256 voteExpiration = block.timestamp + 1 hours; // Set vote expiration 1 hour from now
    ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
        2,
        voteExpiration
    );
    vm.stopPrank();

    // 1 hour passes and validator 0 intentionally skips voting. 
    // Votes expires and the malicious validator 2 cannot be slashed 
    vm.warp(block.timestamp + 1 hours + 1 minutes);

    // Governance set the validator 0 to inactive so he does not break the slashing process 
    // in the next round
    vm.startPrank(admin); // Assuming admin has ADMIN_ROLE needed for setValidatorStatus
    ValidatorFacet(address(diamondProxy)).setValidatorStatus(
        voter0ValidatorId,
        false
    );
    vm.stopPrank();

    // Validator identifies that validator 0 is behaving maliciously
    // he votes to slash validator 0
    // but the transaction reverts with ValidatorInactive(0)
    vm.startPrank(voter1Admin);
    voteExpiration = block.timestamp + 1 hours; // Set vote expiration 1 hour from now
    ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
        voter0ValidatorId,
        voteExpiration
    );
    vm.stopPrank();
}
```

Run: orge test --mt testMaliciousValidator\_cannotBeSlashed\_onceInactive --via-ir

Expected failing output:

```
Failing tests:
Encountered 1 failing test in test/PlumeStakingDiamond.t.sol:PlumeStakingDiamondTest
[FAIL: ValidatorInactive(0)] testMaliciousValidator_cannotBeSlashed_onceInactive() (gas: 1633834)
```

{% endstep %}
{% endstepper %}
