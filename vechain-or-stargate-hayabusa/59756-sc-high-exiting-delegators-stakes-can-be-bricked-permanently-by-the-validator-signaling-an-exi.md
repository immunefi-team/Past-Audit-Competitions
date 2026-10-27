# 59756 sc high exiting delegators stakes can be bricked permanently by the validator signaling an exit after them in the same period

**Submitted on Nov 15th 2025 at 14:51:49 UTC by @flacko for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59756
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Summary

Unstaking and re-delegation incorrectly decrease the effective stake of the old validator when the validator has an `Exited` status and as a result open up a vulnerability where a validator can brick exiting delegators funds by also signaling an exit in the same staking period as them. This can happen both intentionally by a malicious validator or naturally without the knowledge of the validator on the possible outcome.

### Vulnerability Details

The problematic code is in Stargate.sol:

```solidity
    /// @inheritdoc IStargate
    function requestDelegationExit(
        uint256 _tokenId
    ) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {        
    
        // decrease the effective stake
        // Get the latest completed period of the validator
        (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
            delegation.validator
        );
        (, uint32 exitBlock) = $.protocolStakerContract.getDelegationPeriodDetails(delegationId);
        
        // decrease the effective stake
→       _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

Signaling an exit for a delegation exit decreases the effective stake for that validator for period `completedPeriods + 2` which is equal to `currentIteration + 1`.

When a delegation signals an exit it must wait for the `currentIteration` staking period to pass and can withdraw the stake via `unstake()` in the next period (`currentIteration + 1`). The same stands true for validators: when they signal an exit their `CompletedIterations` is set to `currentIteration`.

If both parties have signaled an exit and the next period starts (`currentIteration + 1`) and the delegator calls `unstake()`, `_updatePeriodEffective()` will be called with `oldCompletedPeriods + 2` which will be `currentIteration + 2`. As there's no checkpoint for that period, the contract will do an upper lookup and pick up the most recent checkpoint which is the one that got updated when `requestDelegationExit()` was called whose value already reflects the deducted effective stake of the delegator.

Further, the same effective stake deduction is applied inside `delegate()` when re-delegating away from an exited or pending validator:

```solidity
            // if the delegation is pending or the validator is exited or unknown
            // decrease the effective stake of the previous validator
            if (
                currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
                status == DelegationStatus.PENDING
            ) {
                // get the completed periods of the previous validator
                (, , , uint32 oldCompletedPeriods) = $
                    .protocolStakerContract
                    .getValidationPeriodDetails(currentValidator);

                // decrease the effective stake of the previous validator
→               _updatePeriodEffectiveStake(
                    $,
                    currentValidator,
                    _tokenId,
                    oldCompletedPeriods + 2,
                    false // decrease
                );
            }
```

### Impact Details

* If the exiting delegator's stake is ≥ 50% of all stake delegated to that validator, they would be bricked because `total effective stake - 2 * delegator effective stake` would underflow.
* If multiple delegators have signaled an exit it depends on the size of their stake and the order in which they call `unstake()` but at least some delegators will not be able to withdraw their stake since each delegator's effective stake is subtracted from the total stake twice. Example: 4 delegators each with 25% of total effective stake — if 2 delegators signal an exit, the two that didn't signal an exit may not be able to withdraw their stake because the earlier operations reduced the validator's effective stake twice, leading to underflow and revert.

Note: re-delegation (via `delegate()`) suffers the same double-deduction behavior, so it's not a guaranteed rescue path.

## Proof of Concept

### Test setup addition

Add this snippet to the end of the `beforeEach` function block in `packages/contracts/test/unit/Stargate/Stake.test.ts`. This adds a second validator to show impact on `delegate()` as well.

```solidity
        // Add second validator for re-delegation
        validator2 = contracts.otherAccounts[2];
        tx = await protocolStakerMockContract.addValidation(validator2.address, PERIOD_SIZE);
        await tx.wait();
        tx = await protocolStakerMockContract.helper__setValidatorStatus(
            validator2.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();
```

### Test case (PoC)

Add this test case to the same test unit file.

```solidity
    it.only("[POC] should revert on unstake after validator signaled exit after us", async () => {
        let currentPeriod = 0;

        // 1. Create a stake
        const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
        const userBalanceBeforeStake = await ethers.provider.getBalance(user.address);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();

        const userBalanceAfterStake = await ethers.provider.getBalance(user.address);

        expect(userBalanceAfterStake).to.be.closeTo(
            userBalanceBeforeStake - levelSpec.vetAmountRequiredToStake,
            ethers.parseEther("0.1") // account for gas fees
        );

        // 2. Delegate to the validator
        const tokenId = await stargateNFTMockContract.getCurrentTokenId();
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();

        // 3. Advance validator completed periods by 10
        currentPeriod = 10;
        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, currentPeriod);
        await tx.wait();

        let effectiveStakeBefore =
            await stargateContract.connect(user).getDelegatorsEffectiveStake(validator.address, currentPeriod)

        // 4. Request delegation exit
        // - This will deduce the delegation's effective stake from the validator
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();

        let effectiveStakeAfter =
            await stargateContract.connect(user).getDelegatorsEffectiveStake(validator.address, currentPeriod);

        console.log(`Effective stake updated: ${effectiveStakeBefore.toString()} → ${effectiveStakeAfter.toString()}`)

        // 5. The validator signals an exit in the same staking period after the delegator
        tx = await protocolStakerMockContract.signalExit(validator.address);
        await tx.wait();

        // 6. We are now in the next staking period of the validator.
        //    Update the validator status to exited to simulate housekeeping
        currentPeriod = 11
        tx = await protocolStakerMockContract.helper__setValidatorStatus(validator.address, 3);
        await tx.wait();

        // 7. Assert the delegation is exited
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(DELEGATION_STATUS_EXITED);
        await tx.wait();

        // 8.1. Unstaking now will revert because of double update of effective stake
        const unstakeTx = stargateContract.connect(user).unstake(tokenId);
        await expect(unstakeTx).to.be.revertedWithPanic(17) // arithmetic overflow

        // 8.2. Try redelgating to a different validator to at least save the stake
        const delegateTx = stargateContract.connect(user).delegate(tokenId, validator2.address);
        await expect(delegateTx).to.be.revertedWithPanic(17) // arithmetic overflow
    });
```
