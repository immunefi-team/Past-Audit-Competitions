# 49817 sc medium inactive validators are prevented to claim to eligible commission rewards

* Report ID: #49817
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* Impacts:
  * Temporary freezing of funds for at least 24 hours

## Description

### Summary

Validators who have initiated the commission claim process via `requestCommissionClaim` and waited the mandatory one-week period cannot execute `finalizeCommissionClaim` to withdraw their earned rewards when an administrator changes their status to inactive.

### Vulnerability Details

While a validator has an active status and participates in the network, they earn commission rewards. The intended flow is:

{% stepper %}
{% step %}

### Step: requestCommissionClaim

Validator calls `requestCommissionClaim` to initiate the commission claim process.

Reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/ValidatorFacet.sol#L500-L539>

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
{
   ...
    $.pendingCommissionClaims[validatorId][token] = PlumeStakingStorage.PendingCommissionClaim({
        amount: amount,
        requestTimestamp: nowTs,
        token: token,
        recipient: recipient
    });
    $.validatorAccruedCommission[validatorId][token] = 0;
    ...
}
```

{% endstep %}

{% step %}

### Step: waiting period

Wait for the mandatory one-week period to pass:

claim.requestTimestamp + PlumeStakingStorage.COMMISSION\_CLAIM\_TIMELOCK
{% endstep %}

{% step %}

### Step: finalizeCommissionClaim

Validator calls `finalizeCommissionClaim` to withdraw their earned rewards.

Reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/ValidatorFacet.sol#L573-L575>

```solidity
function finalizeCommissionClaim(
    uint16 validatorId,
    address token
) external onlyValidatorAdmin(validatorId) nonReentrant returns (uint256) {
    ...
    uint256 readyTimestamp = claim.requestTimestamp + PlumeStakingStorage.COMMISSION_CLAIM_TIMELOCK;

    // First, check if the timelock has passed from the perspective of the current block.
    if (block.timestamp < readyTimestamp) {
        revert ClaimNotReady(validatorId, token, readyTimestamp);
    }

    // --- REVISED SLASHING CHECK ---
    // If the validator is slashed, the claim is only considered valid if its timelock was
    // fully completed BEFORE the slash occurred. This invalidates any pending claims.
    if (validator.slashed && readyTimestamp >= validator.slashedAtTimestamp) {
        revert ValidatorInactive(validatorId);
    }

    // For a non-slashed validator, simply require it to be active to finalize a claim.
    if (!validator.slashed && !validator.active) {
        revert ValidatorInactive(validatorId);
    }
    ...
}
```

{% endstep %}
{% endstepper %}

The problem: Even if the validator was eligible to finalize the claim after the waiting period, if an admin sets the validator to inactive after the claim became eligible, `finalizeCommissionClaim` will revert with `ValidatorInactive`. This prevents the validator from withdrawing funds that were accrued while active and whose claim became eligible before the status change.

This is inconsistent with other behaviors in the codebase (e.g., users can unstake and withdraw from inactive validators and can collect rewards/funds from slashed validators if those funds were in a parked state eligible to be claimed before the slash).

### Impact

Any validator that made a commission claim and became eligible to collect it before being set to inactive can be prevented from collecting those rewards. This results in temporary freezing of funds that can only be resolved if the admin re-activates the validator.

### Recommendation

Modify `finalizeCommissionClaim` so it allows finalization when the claim timelock completed before the validator became inactive. Suggested change:

```diff
function finalizeCommissionClaim(
    uint16 validatorId,
    address token
) external onlyValidatorAdmin(validatorId) nonReentrant returns (uint256) {
    ...

    // For a non-slashed validator, simply require it to be active to finalize a claim.
-    if (!validator.slashed && !validator.active) {
-        revert ValidatorInactive(validatorId);
-    }

+    bool validatorCanClaimRewards = (claim.requestTimestamp + PlumeStakingStorage.COMMISSION_CLAIM_TIMELOCK) < validator.slashedAtTimestamp;
+    if (!validator.slashed && !validatorCanClaimRewards) {
+        revert ValidatorInactive(validatorId);
+    }
    ...
}
```

This ensures a non-slashed validator may finalize a claim if the claim's timelock completed before the validator was marked inactive (or slashed).

## Proof of Concept

### Context

PoC demonstrates:

{% stepper %}
{% step %}
Active validator requests commission claim.
{% endstep %}

{% step %}
Active validator waits the unlock period (7 days) and becomes eligible to claim.
{% endstep %}

{% step %}
Admin sets the validator to inactive.
{% endstep %}

{% step %}
Validator tries to claim commission but `finalizeCommissionClaim` reverts with `ValidatorInactive`.
{% endstep %}
{% endstepper %}

### PoC (test)

Add the following test in `PlumeStakeDiamond.t.sol`:

<details>

<summary>Show test code (click to expand)</summary>

```solidity
function testValidatorCannotClaimEligibleCommission_whenSetToInactive() public {
        // Set a very specific reward rate for predictable results
        uint256 rewardRate = 1e18; // 1 PUSD per second
        vm.startPrank(admin);
        address[] memory tokens = new address[](1);
        tokens[0] = address(pUSD);
        uint256[] memory rates = new uint256[](1);
        rates[0] = rewardRate;
        RewardsFacet(address(diamondProxy)).setRewardRates(tokens, rates);

        // Make sure treasury is properly set
        RewardsFacet(address(diamondProxy)).setTreasury(address(treasury));

        // Ensure treasury has enough PUSD by transferring tokens
        uint256 treasuryAmount = 100 ether;
        pUSD.transfer(address(treasury), treasuryAmount);
        vm.stopPrank();

        // Set a 10% commission rate for the validator
        vm.startPrank(validatorAdmin);
        uint256 newCommission = 10e16;
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            DEFAULT_VALIDATOR_ID,
            newCommission
        );
        vm.stopPrank();

        // Create validator with 10% commission
        uint256 initialStake = 10 ether;
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: initialStake}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // Move time forward to accrue rewards
        vm.roll(block.number + 10);
        vm.warp(block.timestamp + 10);

        // Trigger reward updates by having a user interact with the system
        // This will internally call updateRewardsForValidator
        vm.startPrank(user2);
        StakingFacet(address(diamondProxy)).stake{value: 1 ether}(
            DEFAULT_VALIDATOR_ID
        );
        vm.stopPrank();

        // Move time forward again
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 1);

        // Interact again to update rewards once more
        vm.startPrank(user1);

        // Unstake a minimal amount to trigger reward update
        StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 1); // Unstake 1 wei
        vm.stopPrank();

        // Check that some commission has accrued (positive amount)
        uint256 commission = ValidatorFacet(address(diamondProxy))
            .getAccruedCommission(DEFAULT_VALIDATOR_ID, address(pUSD));
        assertGt(commission, 0, "Commission should be greater than 0");

        // @audit - validator request commission claim
        vm.startPrank(validatorAdmin);
        uint256 balanceBefore = pUSD.balanceOf(validatorAdmin);

        ValidatorFacet(address(diamondProxy)).requestCommissionClaim(
            DEFAULT_VALIDATOR_ID,
            address(pUSD)
        );
        vm.stopPrank();

        // unlock period passes(7 days), now this validator is eligible to claim commission
        uint256 blockTimestamp = block.timestamp;
        vm.warp(blockTimestamp + 7 days + 1 seconds);

        // but admin makes validator inactive first
        vm.startPrank(admin);
           ValidatorFacet(address(diamondProxy)).setValidatorStatus(
            DEFAULT_VALIDATOR_ID,
            false
        );
        vm.stopPrank();

        // @audit - validator try to finalize commission claim but it reverts with ValidatorInactive error
        vm.startPrank(validatorAdmin);
            vm.expectRevert(
                abi.encodeWithSelector(ValidatorInactive.selector, DEFAULT_VALIDATOR_ID)
            );
            ValidatorFacet(address(diamondProxy)).finalizeCommissionClaim(
                DEFAULT_VALIDATOR_ID,
                address(pUSD)
            );
        vm.stopPrank();
    }
```

</details>

Run: forge test --mt testValidatorCannotClaimEligibleCommission\_whenSetToInactive -vv --via-ir

Output:

```
[PASS] testValidatorCannotClaimEligibleCommission_whenSetToInactive() (gas: 1598430)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.36ms (1.18ms CPU time)
Ran 1 test suite in 182.91ms (12.36ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

***

If you want, I can:

* Propose a minimal patched implementation for `finalizeCommissionClaim` that accounts for validator inactive transitions (keeping style and existing checks).
* Create a unit test that validates both scenarios (claim eligible before inactive vs not eligible).
