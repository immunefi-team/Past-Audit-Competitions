# 59657 sc high delegators lose first reward period when delegating to pending validators

**Submitted on Nov 14th 2025 at 14:51:14 UTC by @danvinci\_20 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59657
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Summary

When delegating to a validator that is in the pending phase, the protocol initializes:

```
lastClaimedPeriod[tokenId] = completedPeriods + 1;
```

Because the reward calculation always starts from:

```
nextClaimablePeriod = lastClaimedPeriod[tokenId] + 1;
```

the delegator permanently loses rewards for the first active period of the new delegation when the validator was pending. The code assumes delegation starts in the *next* period, but for pending validators the actual delegation `startPeriod` is `completedPeriods + 1`. By shifting `lastClaimedPeriod` forward by one, the protocol skips the first eligible reward period, causing loss of user rewards.

### Vulnerability Details

Relevant code paths during delegation:

```solidity
// Get the latest completed period of the validator
(, , , uint32 completedPeriods) = $.protocolStakerContract.getValidationPeriodDetails(_validator);

// update the last claimed period to the current period of the validator
$.lastClaimedPeriod[_tokenId] = completedPeriods + 1; // current period
```

Reward calculation logic:

```solidity
uint32 nextClaimablePeriod = $.lastClaimedPeriod[_tokenId] + 1;
```

Sequence of effects:

{% stepper %}
{% step %}

### Step: Setting lastClaimedPeriod

Setting `lastClaimedPeriod = completedPeriods + 1`.
{% endstep %}

{% step %}

### Step: nextClaimablePeriod becomes shifted

This makes `nextClaimablePeriod = completedPeriods + 2`. However, if the validator is PENDING, the delegation actually starts at `completedPeriods + 1`. The first active period (`completedPeriods + 1`) is skipped and becomes unclaimable.
{% endstep %}
{% endstepper %}

### Impact

Users lose one full period of rewards when delegating to a validator that is still in the pending phase.

### Recommendation

Safely handle delegations to pending validators so users do not lose rewards for the first active period. (No code changes are provided here; implementors should ensure `lastClaimedPeriod` does not advance past the delegation `startPeriod` for pending validators, or initialize it to the correct previous completed period such that the first active period remains claimable.)

## Proof of Concept

<details>

<summary>Test to reproduce (add to Delegation.test.ts)</summary>

Run:

```
yarn hardhat test --network vechain_solo test/integration/Delegation.test.ts
```

Test:

```javascript
it.only("shoushowing the case where startPeriod < firstClaimabalePeriod", async () => {
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


    // Adding a new validator to the protocol
    const newValidator = (await ethers.getSigners())[5];
    const addValidatorTx = await protocolStakerContract
        .connect(deployer)
        .addValidation(newValidator.address, 12, {
            value: ethers.parseEther("25000000"),
        });
    await addValidatorTx.wait();


    // Assert that the NFT is mature, so it can be delegated
    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.false;

    // Stargate <> ProtocolStaker - delegate the NFT to the validator
    const delegateTx = await stargateContract.connect(user).delegate(tokenId, newValidator.address);
    await delegateTx.wait();
    console.log("\n🎉 Correctly delegated the NFT to validator", newValidator.address);

    const delegation = await stargateContract.getDelegationDetails(tokenId);
    let [start,] = await protocolStakerContract.getDelegationPeriodDetails(delegation.delegationId);

    console.log("start period of delegation: ", start);


    //showing it is in pending phase
    let [, , , , validator2status,] =
        await protocolStakerContract.getValidation(newValidator.address);
    console.log("newValidatorStatus: ", validator2status); 

    
    // Fast-forward to the next period, so that delegation becomes active
    const [period, startBlock, ,] = await protocolStakerContract.getValidationPeriodDetails(
        deployer.address
    );
    const periodsToComplete = 0; // Only fast-forward to the next period
    await fastForwardValidatorPeriods(
        Number(period),
        Number(startBlock),
        periodsToComplete
    );

    let lastClaimedPeriod = await stargateContract.getLastClamiedPeriod(tokenId);
    console.log("lastClaimedPeriod: ", lastClaimedPeriod);
    console.log("nextClaimablePeriod: ", lastClaimedPeriod + BigInt(1));
});
```

Observed output:

```
🎉 Correctly delegated the NFT to validator 0x61E7d0c2B25706bE3485980F39A3a994A8207aCf
start period:  1n
newValidatorStatus:  1n
lastClaimedPeriod:  1n
nextClaimablePeriod:  2n
    ✔ shoushowing the case where startPeriod < firstClaimabalePeriod (1866ms)
```

</details>

## References

* Target contract: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>

{% hint style="info" %}
No code changes are included in this report. Implementers should ensure `lastClaimedPeriod` is initialized or adjusted so it does not skip the delegation `startPeriod` when validators are pending.
{% endhint %}
