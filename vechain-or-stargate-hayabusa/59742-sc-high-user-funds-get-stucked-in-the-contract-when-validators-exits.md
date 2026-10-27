# 59742 sc high user funds get stucked in the contract when validators exits&#x20;

**Submitted on Nov 15th 2025 at 12:18:00 UTC by @danvinci\_20 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59742
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Summary

The system decreases a user’s effective stake when they request a delegation exit and then decreases it again when they later call `unstake()` if the validator has already exited.

Because the validator is already in an exited state, the second decrease is performed using a later period index, causing `_updatePeriodEffectiveStake()` to underflow during:

```solidity
updatedValue = currentValue - effectiveStake;
```

This results in a panic(0x11) arithmetic underflow, reverting `unstake()`. Since `unstake()` is the only mechanism for the user to retrieve their staked VET, the user’s funds become permanently stuck in the Stargate contract, with no recovery path.

### Vulnerability Details

Relevant Code Paths

* When `requestDelegationExit()` is called, the contract immediately reduces the user's effective stake:

```solidity
_updatePeriodEffectiveStake(
    $, 
    delegation.validator, 
    _tokenId, 
    completedPeriods + 2, 
    false
);
```

* Later, if the validator exits before the user calls `unstake()`, the `unstake()` function performs another decrease:

```solidity
if (
    currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
    delegation.status == DelegationStatus.PENDING
) {
    (, , , uint32 oldCompletedPeriods) =
        $.protocolStakerContract.getValidationPeriodDetails(delegation.validator);

    _updatePeriodEffectiveStake(
        $,
        delegation.validator,
        _tokenId,
        oldCompletedPeriods + 2,
        false // decrease again
    );
}
```

* Inside `_updatePeriodEffectiveStake` the underflow occurs:

```solidity
uint256 updatedValue = _isIncrease
    ? currentValue + effectiveStake
    : currentValue - effectiveStake;  // underflows on second decrease
```

Because the second decrease subtracts the same `effectiveStake` from a smaller `currentValue`, the result becomes negative, causing a panic(0x11). This reverts the `unstake()` call entirely.

As a result, the user can never unstake or retrieve their VET, leaving their funds permanently stuck inside the Stargate contract.

## Recommendation

{% hint style="warning" %}
Guard to ensure effective stake is only decreased once per delegation lifecycle and avoid trying to reduce effective stake for periods when there are no stakers.
{% endhint %}

## Proof of Concept

<details>

<summary>Test showing the revert (add this test to Delegation.test.ts and run: yarn hardhat test --network vechain_solo test/integration/Delegation.test.ts)</summary>

```javascript
it.only("user funds get stucked in the contract", async () => {
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


    //staking the token here 
    const levelId = 1;
    const levelSpec = await stargateNFTContract.getLevel(levelId);
    const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;

    // Stake an NFT of level 1
    const stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Correctly staked an NFT of level", levelId);

    // Assert that user1 is the owner of the NFT, and the NFT is under the maturity period
    const tokenId = await stargateNFTContract.getCurrentTokenId();
    expect(await stargateNFTContract.ownerOf(tokenId)).to.equal(user.address);
    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.true;

    // Fast-forward until the NFT is mature
    await mineBlocks(Number(levelSpec.maturityBlocks));
    log("\n🚀 Fast-forwarded", Number(levelSpec.maturityBlocks), "blocks to mature the NFT");

    // Assert that the NFT is mature, so it can be delegated
    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.false;



    //new validator enters the staker
    const newValidator = (await ethers.getSigners())[5];
    const addValidatorTx = await protocolStakerContract.connect(newValidator).addValidation(newValidator.address, 12, {
            value: ethers.parseEther("25000000"),
        });
    await addValidatorTx.wait();
    log("\n🎉 Correctly added a new validator to the protocol");



    // Stargate <> ProtocolStaker - delegate the NFT to the validator
    const delegateTx = await stargateContract.connect(user).delegate(tokenId, newValidator.address);
    await delegateTx.wait();
    console.log("\n🎉 Correctly delegated the NFT to validator", newValidator.address);

    let delegation = await stargateContract.getDelegationDetails(tokenId);
    let [start,] = await protocolStakerContract.getDelegationPeriodDetails(delegation.delegationId);
    console.log("start period of delegation: ", start);

    let [period,startBlock,,completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(newValidator.address);
    console.log("completedPeriods Before: ", completedPeriods);

    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        5
    );

    [,,,completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(newValidator.address);
    console.log("completedPeriods Now: ", completedPeriods);


    const requestExitTx = await stargateContract.connect(user).requestDelegationExit(tokenId);
    await requestExitTx.wait();
    console.log("\n🎉 correctly request exit");

    delegation = await stargateContract.getDelegationDetails(tokenId);
    console.log("end period of delegation: ", delegation.endPeriod)

    await protocolStakerContract.connect(newValidator).signalExit(newValidator.address);


    let [,,exitBlock,] = await protocolStakerContract.getValidationPeriodDetails(newValidator.address);
    console.log("endPeriod of Validator: ", exitBlock);

    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        3
    );

     let [,,,,status,] = await protocolStakerContract.getValidation(newValidator.address);
    console.log("status is 3 showing exit: ", status);


    //now the user who delegated wants to exit unstake but gets a revert
    await expect(
        stargateContract.connect(user).unstake(tokenId)
    ).to.be.revertedWithPanic(0x11);
});
```

Output observed:

```
🎉 Correctly delegated the NFT to validator 0x61E7d0c2B25706bE3485980F39A3a994A8207aCf
start period of delegation:  1n
completedPeriods Before:  0n
completedPeriods Now:  5n

🎉 correctly request exit
end period of delegation:  6n
endPeriod of Validator:  120n
status is 3 showing exit:  3n
    ✔ user funds get stucked in the contract (6211ms)
```

</details>
