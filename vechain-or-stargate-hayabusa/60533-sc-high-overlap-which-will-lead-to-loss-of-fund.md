# 60533 sc high overlap which will lead to loss of fund

**Submitted on Nov 23rd 2025 at 21:08:09 UTC by @Demelew for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60533
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

if the validator status is changed an Overlap might happen in which it will try to subtract twice from the validator status

## Vulnerability Details

\#. requestDelegationExit (Line 568) When a user calls **requestDelegationExit**, if the validator's status is **ACTIVE or PENDING**, the **\_updatePeriodEffectiveStake** function is called to decrease the effective stake:

* Condition: Validator status is ACTIVE or PENDING
* .Action: Decreases the effective stake immediately upon exit request.
* This is the first decrease. #2. \_delegate (Line 402, triggered by delegation/redelegation) The **\_delegate** function handles new delegations or transfers. It includes logic to handle delegations whose previous delgator has exited
* .Condition: The previously delegated validator's status is VALIDATOR\_STATUS\_EXITED. and the previously delegator status is exited
* Action: Decreases the effective stake associated with the old delegation. -This is the second decrease. #The Overlap Scenario The double decrease occurs in this sequence:A user requests an exit (requestDelegationExit) while the validator is ACTIVE. First Decrease (Line 568) happens. the validator's status remains relevant. The validator updates status to VALIDATOR\_STATUS\_EXITED (due to external protocol rules).The user calls \_delegate (to delegate to a new validator .Since the previous delgator status is now EXITED,\
  and the validator status is VALIDATOR\_STATUS\_EXITED the logic at Line 402 is triggered. Second Decrease (Line 402) happens for the same original stake amount. Result: The effective stake for that delegation amount is decreased twice—once when the exit was requested, and a second time when the validator became exited and a subsequent delegation function was called.

## Impact Details

loss of the stake for the validator

* Permanent freezing of funds

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

add this code in the test folder Delegation.test and what this code do is first delegator then it will see how much the effective stake of the validator is then call requestDelegationExit withdrawal delegator then change the validator status in to VALIDATOR\_STATUS\_EXITED then call the delegator function after that it will see how much of stake was decreased from the previous validator

```
 const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        tx = await stargateContract.connect(user).stake(LEVEL_ID, {
            value: levelSpec.vetAmountRequiredToStake,
        });
        await tx.wait();
        const tokenId = await stargateNFTMock.getCurrentTokenId();
        log("\n🎉 Staked token with id:", tokenId);

        // delegate the NFT to the validator
        tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
        await tx.wait();
        log("\n🎉 Correctly delegated the NFT to validator", validator.address);

        // check the delegation status
        const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
        expect(delegationStatus).to.equal(DELEGATION_STATUS_PENDING);

        // advance 1 period
        // so the delegation is active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 1);
        await tx.wait();
        log("\n Set validator completed periods to 1 so the delegation is active");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );
                 const one =    await stargateContract.$.delegatorsEffectiveStake[validator.address].upperLookup(_period)(validator.address, VALIDATOR_STATUS_EXITED);  

        // request exit
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
        await tx.wait();
        log("\n🎉 Correctly requested to exit the delegation");
        // advance 1 period
        // so the delegation is exited
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        await tx.wait();
        log("\n Set validator completed periods to 2 so the delegation is exited");
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_EXITED
        );
  await  protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);  
        // Delegate again
        const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
        const callTx = stargateContract.connect(user).delegate(tokenId, validator.address);
 await expect(callTx)
            .to.emit(stargateContract, "DelegationWithdrawn")
            .withArgs(
                tokenId,
                validator.address,
                delegationId,
                levelSpec.vetAmountRequiredToStake,
                levelSpec.id
            );
        await expect(callTx).to.not.emit(stargateContract, "DelegationExitRequested");
        await expect(callTx).to.emit(stargateContract, "DelegationRewardsClaimed").withArgs(
            user.address,
            tokenId,
            delegationId,
            ethers.parseEther("0.1"), // fixed rewards in the mock
            2, // first claimable period, it entered in period 1
            2 // last claimable period, it requested exit in period 2
        );
        await expect(callTx)
            .to.emit(stargateContract, "DelegationInitiated")
            .withArgs(
                tokenId,
                validator.address,
                delegationId + 1n, // new delegation id
                levelSpec.vetAmountRequiredToStake,
                LEVEL_ID,
                100 // probability multiplier
            );
        log("\n🎉 Correctly delegated back the NFT to validator", validator.address);
        expect(await stargateContract.getDelegationStatus(tokenId)).to.equal(
            DELEGATION_STATUS_PENDING
        );
```
