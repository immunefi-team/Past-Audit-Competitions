# 59866 sc high the delegator s rewards in period 1 cannot be claimed

**Submitted on Nov 16th 2025 at 14:41:35 UTC by @shaflow1 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59866
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of unclaimed yield

## Description

### Brief / Intro

When a validator is in the `Queue` rather than `Active`, `completedPeriods` will return 0, and `CurrentIteration` will also return 0. Adding a delegate to a validator in this state should take effect in period 1 (`current + 1 = 0 + 1 = 1`) and start accruing rewards.

However, the `_delegate` function in the Stargate contract does not account for this special case and records the delegate as taking effect in period 2 (`completedPeriods + 2 = 0 + 2`) for reward accrual. This causes the delegator to be unable to claim the delegate rewards for period 1, and the rewards get stuck in the StargateNFT contract.

***

## Vulnerability Details

First, the intended logic in the protocol staker contract (Go code excerpt) for adding a delegation:

<https://github.com/vechain/thor/blob/bffc3378d5d8aa346433ae911495bcc3952b265d/builtin/staker/staker.go#L535>

```go
func (s *Staker) AddDelegation(...) (*big.Int, error) {
    //...
	// add delegation on the next iteration - val.CurrentIteration() + 1,
	current, err := val.CurrentIteration(currentBlock)
	if err != nil {
		return nil, err
	}
	delegationID, err := s.delegationService.Add(validator, current+1, stake, multiplier)
    //...
}

func (v *Validation) CurrentIteration(currentBlock uint32) (uint32, error) {
	// Unknown, Queued return 0
	if v.Status == StatusUnknown || v.Status == StatusQueued {
		return 0, nil
	}

	// Exited, from active or queued
	if v.Status == StatusExit {
		return v.CompletedPeriods, nil
	}

	// Active(signaled exit)
	// Once signaled exit, complete iterations is set to the current
	// iteration of the time that exit is signaled
	if v.CompletedPeriods > 0 {
		return v.CompletedPeriods, nil
	}

	// Active
	if currentBlock < v.StartBlock {
		return 0, errors.New("curren block cannot be less than start block")
	}
	if v.Period == 0 {
		return 0, errors.New("period cannot be zero")
	}
	elapsedBlocks := currentBlock - v.StartBlock
	completedPeriods := elapsedBlocks / v.Period
	return completedPeriods + 1, nil
}
```

Under normal circumstances, `completedPeriods + 1 = currentPeriods`, and a delegation takes effect in the next period, so the effective period is `completedPeriods + 2`.

However, there is a special case: if the validator is in the Queue state rather than Active, `completedPeriods = currentPeriods = 0`. In this case, the effective period of the delegation should be `completedPeriods + 1`.

In the Stargate contract `_delegate` function, the special case is not considered and the effective period is always recorded as `completedPeriods + 2`:

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L461>

```solidity
    function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
        //...
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
                _updatePeriodEffectiveStake(
                    $,
                    currentValidator,
                    _tokenId,
                    oldCompletedPeriods + 2,
                    false // decrease
                );
            }
        //...
        // Get the latest completed period of the validator
        (, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(
            _validator
        );

        // update the mappings regarding the delegation
        $.delegationIdByTokenId[_tokenId] = delegationId;
        // update the last claimed period to the current period of the validator
        // (aka: claimable periods will be from the next period)
        $.lastClaimedPeriod[_tokenId] = completedPeriods + 1; // current period

        // Increase the delegators effective stake in the next period
        _updatePeriodEffectiveStake($, _validator, _tokenId, completedPeriods + 2, true);
        //...
    }
```

Because of this, when delegating to a validator in the QUEUED state, the Stargate contract sets the effective stake starting in period 2, while `lastClaimedPeriod[_tokenId]` is set to period 1. This mismatch leaves the rewards for period 1 permanently unclaimable.

***

## Impact Details

If a delegation is made to a validator that is in the `Queue`:

* `_updatePeriodEffectiveStake` will set the effective period to period 2,
* `lastClaimedPeriod[_tokenId]` is set to period 1.

This results in the user's staking delegation rewards for period 1 being unclaimable. The delegation rewards will become stuck in the Stargate contract (StargateNFT), effectively freezing yield.

***

## Proof of Concept

The following integration test demonstrates the issue by delegating to a queued validator and checking the effective stake periods:

```javascript
it("Delegation created to a queued validator should result in pending delegation with periodEffectiveStake = 2", async () => {
    // --- Stake NFT (Level 1) ---
    const levelId = 1;
    const levelSpec = await stargateNFTContract.getLevel(levelId);
    const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;

    const stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Staked NFT of level", levelId);

    const tokenId = await stargateNFTContract.getCurrentTokenId();

    // fast-forward to maturity
    await mineBlocks(Number(levelSpec.maturityBlocks));
    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.false;

    const validatorQueued = (await ethers.getSigners())[5];

    // --- Add validator with very small stake so that it remains QUEUED ---
    const tx = await protocolStakerContract
        .connect(validatorQueued)
        .addValidation(
            validatorQueued.address,
            12,
            { value: ethers.parseEther("25000000") }  // insufficient to become Active
        );
    await tx.wait();

    const validationInfo = await protocolStakerContract.getValidation(
        validatorQueued.address
    );
    const status = validationInfo[4];
    expect(status).to.equal(1); // 1 = QUEUED
    log("\n🎯 Validator is in QUEUED state");

    // --- delegate to queued validator ---
    const delegateTx = await stargateContract
        .connect(user)
        .delegate(tokenId, validatorQueued.address);
    await delegateTx.wait();

    log("\n🎉 Delegated NFT to queued validator", validatorQueued.address);

    // fetch delegationId
    const delegationId = await stargateContract.getDelegationIdOfToken(tokenId);
    expect(delegationId).to.not.equal(0);

    // --- Check delegation status is pending ---
    const delegationStatus = await stargateContract.getDelegationStatus(tokenId);
    expect(delegationStatus).to.equal(1n); // 1 = pending

    const EffectiveStake1 = await stargateContract.getDelegatorsEffectiveStake(validatorQueued.address ,1);
    const EffectiveStake2 = await stargateContract.getDelegatorsEffectiveStake(validatorQueued.address ,2);
    console.log("EffectiveStake1:" + EffectiveStake1);
    console.log("EffectiveStake2:" + EffectiveStake2);

    const [startPeriod, endPeriod] = await protocolStakerContract.getDelegationPeriodDetails(delegationId);
    console.log("StartPeriod in StakingContract:" + startPeriod);
});
```

To run the POC, add the above code to `packages/contracts/test/integration/Delegation.test.ts` and run the integration tests.

Based on test logs (attachments), when delegation is performed while the validator is in the QUEUED state, the rewards begin to take effect from period 2 (`EffectiveStake = 2`) in the Stargate contract, whereas in the `protocolStakerContract` the delegation is effective from period 1. This discrepancy causes rewards from period 1 to be unclaimable.

***

## References

* Stargate.sol (target): <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* Protocol staker AddDelegation & CurrentIteration reference: <https://github.com/vechain/thor/blob/bffc3378d5d8aa346433ae911495bcc3952b265d/builtin/staker/staker.go#L535>
