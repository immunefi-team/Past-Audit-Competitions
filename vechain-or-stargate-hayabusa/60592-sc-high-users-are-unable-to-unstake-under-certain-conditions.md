# 60592 sc high users are unable to unstake under certain conditions

**Submitted on Nov 24th 2025 at 08:32:14 UTC by @Filippo for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60592
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`Stargate.sol` subtracts the effective stake from the delegator effective stake both when a user exits a delegation to that validator and when the validator exits and the user unstakes a token previously delegated to that validator. So the effective stake of the same token may be subtracted twice from the delegator effective stake. The `delegator effective stake` is a `uint256`, so it's possible that it may underflow, rendering users unable to unstake and thus permanently freezing their vet.

## Vulnerability Details

When a user unstakes the function `unstake` checks if the validator associated with the token being unstaken exited, and if so it decreases the delegators effective stake by the token effective stake.

```solidity
function unstake(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    Delegation memory delegation = _getDelegationDetails($, _tokenId);
    DataTypes.Token memory token = $.stargateNFTContract.getToken(_tokenId);

    // ...
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

    // ...
}
```

This is a bug for the following reason: when a user request to exit a delegation, the function `requestDelegationExit` again decreases the delegators effective stake by the token effective stake.

```solidity
function requestDelegationExit(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
    StargateStorage storage $ = _getStargateStorage();
    uint256 delegationId = $.delegationIdByTokenId[_tokenId];

    // ...

    // decrease the effective stake
    _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);

    // ...
}
```

So, if a user both exits the delegation and unstakes after the validator exited, the delegator effective stakes is going to be decreased by the token effective stake twice. This can lead to undreflow, as I've showed in the PoC.

## Impact Details

The only way to unstake a token and get the vet back is through calling the function unstake of Stargate.sol. But this vulnerability may cause some users to be unable to call unstake successfully, rendering them unable to get the vet they used for staking back, thus permanently locking their funds.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/Stargate.sol#L267>

## Proof of Concept

## Proof of Concept

Paste the following test in `packages/contracts/contracts/test/unit/Delegation.test.ts`. Run: `yarn:contracts:test:unit:verbose`. The test should fail with the following output: `@repo/contracts:test:hardhat: 1 failing @repo/contracts:test:hardhat: @repo/contracts:test:hardhat: 1) shard-u2: Stargate: Delegation @repo/contracts:test:hardhat: underflows: @repo/contracts:test:hardhat: Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block) @repo/contracts:test:hardhat: at Stargate._updatePeriodEffectiveStake (contracts/Stargate.sol:1013) @repo/contracts:test:hardhat: at Stargate.unstake (contracts/Stargate.sol:276) @repo/contracts:test:hardhat: at StargateProxy._delegate (@openzeppelin/contracts/proxy/Proxy.sol:31)` The output error clearly shows that there's an underflow when unstaking.

```typescript
it("underflows", async() => {
    const levelSpec = await stargateNFTMock.getLevel(LEVEL_ID);
    // user stakes one token
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const tokenId = await stargateNFTMock.getCurrentTokenId();

    // user stakes another token
    tx = await stargateContract.connect(user).stake(LEVEL_ID, {
        value: levelSpec.vetAmountRequiredToStake,
    });
    await tx.wait();
    const tokenId2 = await stargateNFTMock.getCurrentTokenId();

    // user delegates first token
    tx = await stargateContract.connect(user).delegate(tokenId, validator.address);
    await tx.wait();

    // we get the delegator effective stake after the first delegation
    const delegatorsEffectiveStake = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        12
    );

    // user delegates second token
    tx = await stargateContract.connect(user).delegate(tokenId2, validator.address);
    await tx.wait();

    // we get the delegator effective stake after the second delegation
    const delegatorsEffectiveStake2 = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        12
    );

    // the delegator effective stake has now doubled
    expect(delegatorsEffectiveStake2).to.be.equal(delegatorsEffectiveStake * 2n);

    // we advance the completed periods
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 15);
    await tx.wait();

    // we exit the first delegation
    tx = await stargateContract.connect(user).requestDelegationExit(tokenId);
    await tx.wait();

    const delegatorsEffectiveStake3 = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        25
    );

    // after the first delegation exit, the delegator effective stake went back
    // to what it was after one delegation
    expect(delegatorsEffectiveStake3).to.be.equal(delegatorsEffectiveStake);

    // we exit the second delegation
    tx = await stargateContract.connect(user).requestDelegationExit(tokenId2);
    await tx.wait();

    const delegatorsEffectiveStake4 = await stargateContract.getDelegatorsEffectiveStake(
        validator.address,
        25
    );

    // now the delegator effective stake is back to zero
    expect(delegatorsEffectiveStake4).to.be.equal(0);

    // we fast forward
    tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validator.address, 19);
    await tx.wait();

    // validator exits
    tx = await protocolStakerMock.helper__setValidatorStatus(
        validator.address,
        VALIDATOR_STATUS_EXITED
    );
    await tx.wait();

    await stargateContract.connect(user).unstake(tokenId);
    await tx.wait();
    
});
```
