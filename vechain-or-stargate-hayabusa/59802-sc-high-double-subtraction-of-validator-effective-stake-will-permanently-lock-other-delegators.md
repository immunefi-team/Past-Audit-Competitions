# 59802 sc high double subtraction of validator effective stake will permanently lock other delegators staked vet

**Submitted on Nov 16th 2025 at 00:10:54 UTC by @xKeywordx for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59802
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Permanent freezing of funds

## Description

The `Stargate` contract maintains a per-validator checkpointed value of delegators’ effective stake in this mapping:

```solidity
mapping(address validator => Checkpoints.Trace224 amount) delegatorsEffectiveStake;
```

This is updated via `_updatePeriodEffectiveStake` whenever delegations are added, moved, exited, or unstaked.

There is a sequence of interaction between the `Stargate::requestDelegationExit`, `Stargate::unstake` and `Stargate::getDelegationStatus` functions, which can cause the `delegatorsEffectiveStake` of a validator to be subtracted twice, corrupting the `delegatorsEffectiveStake` accounting and leading to a permanent freezing of funds.

The following stepper explains how the bug can occur:

{% stepper %}
{% step %}

### Step

We have two users, Alice and Bob. Both of them staked and have some valid Stargate NFTs. Assume Alice and Bob staked the same amounts for simplicity (50 tokens each).
{% endstep %}

{% step %}

### Step

They both delegate their stakes to the same validator. The validator's `delegatorsEffectiveStake` = 100 right now.
{% endstep %}

{% step %}

### Step

While the validator is active Bob wants to exit, so he calls `requestDelegationExit`, which updates the `delegatorsEffectiveStake` mapping for the validator. Bob's amount will be subtracted from the total because he's exiting, so the `delegatorsEffectiveStake` of the validator will be 50 now.
{% endstep %}

{% step %}

### Step

The validator exits. This enables the bug.
{% endstep %}

{% step %}

### Step

Bob calls `unstake`. Because the validator exited in the meantime, `unstake` will reduce the validator's `delegatorsEffectiveStake` again. This is wrong because Bob's amount was already subtracted when he called `requestDelegationExit`. Excerpt from the code:

```solidity
function unstake(uint256 _tokenId) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    // ..
    // if the delegation is pending or the validator is exited or unknown
    // decrease the effective stake of the previous validator
    if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING) {
        // get the completed periods of the previous validator
        (, , , uint32 oldCompletedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);

        // decrease the effective stake of the previous validator
        _updatePeriodEffectiveStake(
            $,
            delegation.validator,
            _tokenId,
            oldCompletedPeriods + 2,
            false // decrease
        );
    }
    // ..
}
```

{% endstep %}

{% step %}

### Step

Bob will successfully exit, because the validator had an extra 50 tokens from Alice, but Alice's funds will be permanently locked.
{% endstep %}
{% endstepper %}

## Root cause

Double subtraction of delegators’ effective stake for the same delegation, combined with a global (per-validator) aggregation.

{% stepper %}
{% step %}

### Issue

`requestDelegationExit` already removes the token’s effective stake from the `delegatorsEffectiveStake[validator]` mapping via:

```solidity
_updatePeriodEffectiveStake($, validator, tokenId, completedPeriods + 2, false);
```

{% endstep %}

{% step %}

### Issue

If the validator exits before the user gets to `unstake`, but after they have already requested an exit, `unstake` will again call:

```solidity
if (currentValidatorStatus == VALIDATOR_STATUS_EXITED || delegation.status == DelegationStatus.PENDING) {
    //..
    _updatePeriodEffectiveStake($, validator, tokenId, oldCompletedPeriods + 2, false);
}
```

This will remove the same amount twice, although it was already fully removed at the first call.
{% endstep %}

{% step %}

### Issue

Because `delegatorsEffectiveStake` is a global variable stored per validator, the second subtraction is taken out of the aggregated sum, which may now consist only of other delegators’ stake.
{% endstep %}
{% endstepper %}

## Impact

Critical – Permanent freezing of funds

At some point, when other delegators later attempt to `unstake`, `_updatePeriodEffectiveStake(..., false)` runs with `currentValue == 0` and `effectiveStake > 0`, causing a checked arithmetic underflow and revert.

In other words, a user with a staked NFT will reach a state where they have a valid delegation that should be exited, but any call to `unstake(...)` reverts with a panic 0x11, because `delegatorsEffectiveStake[validator]` has been driven to 0 by other delegators’ double subtraction.

## Recommended mitigation

Ensure that each delegation’s effective stake is added and removed exactly once.

## Proof of Concept

Add this test in the following file `Stake.test.ts` and run it:

```javascript
it.only("should prove double subtraction error", async () => {
    // create a second user (Alice == user1 & Bob == user2)
    const levelSpec = await stargateNFTMockContract.getLevel(LEVEL_ID);
    const user2 = otherAccounts[2];

    // initial balance checks
    const bobBalanceBeforeStake = await ethers.provider.getBalance(user2.address);
    const validatorEffectiveStakeBeforeStake = await stargateContract.getDelegatorsEffectiveStake(validator.address, 0);
    console.log("Validator effective delegators balance before stake = ", validatorEffectiveStakeBeforeStake);

    // stake both NFTs
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const tokenId1 = await stargateNFTMockContract.getCurrentTokenId();
    tx = await stargateContract.connect(user2).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const tokenId2 = await stargateNFTMockContract.getCurrentTokenId();

    // delegate the NFTs to the validator
    tx = await stargateContract.connect(user).delegate(tokenId1, validator.address);
    await tx.wait();
    tx = await stargateContract.connect(user2).delegate(tokenId2, validator.address);
    await tx.wait();
    const validatorEffectiveStakeAfterDelegations = await stargateContract.getDelegatorsEffectiveStake(validator.address, 2);
    console.log("Validator effective delegators balance after delegations = ", validatorEffectiveStakeAfterDelegations);

    // set validator completed periods to 10 so is active
    tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 10);
    await tx.wait();

    // Bob requests delegation exit
    tx = await stargateContract.connect(user2).requestDelegationExit(tokenId2);
    await tx.wait();

    const validatorEffectiveStakeAfterBobRequestExit = await stargateContract.getDelegatorsEffectiveStake(validator.address, 12);
    console.log("Validator effective delegators balance after Bob requests exit = ", validatorEffectiveStakeAfterBobRequestExit);

    // advance some periods so is exited
    tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(validator.address, 20);
    await tx.wait();

    // assert the delegation is exited
    expect(await stargateContract.getDelegationStatus(tokenId2)).to.equal(DELEGATION_STATUS_EXITED);
    await tx.wait();

    // change validator status to EXITED, this will enable the bug
    tx = await protocolStakerMockContract.helper__setValidatorStatus(validator.address, VALIDATOR_STATUS_EXITED);
    await tx.wait();

    // unstake the NFT of Bob and expect success
    tx = await stargateContract.connect(user2).unstake(tokenId2);
    await tx.wait();

    // PROVE THE BUG, effective delegations are wrongfully zeroed
    const validatorEffectiveStakeAfterBobUnstake = await stargateContract.getDelegatorsEffectiveStake(validator.address, 22);
    console.log("Validator effective delegators balance after Bob unstake = ", validatorEffectiveStakeAfterBobUnstake);

    const bobBalancePostUnstake = await ethers.provider.getBalance(user2.address);
    expect(bobBalancePostUnstake).to.be.closeTo(
        bobBalanceBeforeStake,
        ethers.parseEther("0.1") // gas tolerance
    );

    // assert the token is burned
    await expect(stargateNFTMockContract.ownerOf(tokenId2)).to.be.revertedWithCustomError(
        stargateNFTMockContract,
        "ERC721NonexistentToken"
    );
    expect(await stargateNFTMockContract.ownerOf(tokenId1)).to.equal(user.address);

    // Alice tries to unstake and it panics due to underflow. Alice's funds are stuck
    await expect(stargateContract.connect(user).unstake(tokenId1)).to.be.revertedWithPanic(0x11);
});
```

Test output:

```javascript
  shard-u1: Stargate: Staking
Validator effective delegators balance before stake =  0n
Validator effective delegators balance after delegations =  3000000000000000000n
Validator effective delegators balance after Bob requests exit =  1500000000000000000n
Validator effective delegators balance after Bob unstake =  0n
    ✔ should prove double subtraction error (80ms)


  1 passing (1s)
```

As shown by the test, two users stake at the same level and delegate to the same validator. If the validator exits between `requestDelegationExit` and `unstake` called by one of the users, the validator's `delegatorsEffectiveStake` can be decreased twice, causing other delegators' funds to become permanently stuck.
