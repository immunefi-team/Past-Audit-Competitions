# 60259 sc low malicious user can bypass maturity period for newly added levels

* **Submitted on:** Nov 20th 2025 at 15:41:53 UTC by @danvinci\_20
* **Report ID:** #60259
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:** Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Summary

The system allows creation of new staking levels via `addLevel()`, and each level is expected to require a boost fee to skip maturity before delegation. During the upgrade, `initializeV3()` sets the `boostPricePerBlock` values only for pre-existing levels. However, new levels added after the upgrade do not store or initialize any `boostPricePerBlock` value. Because the boost amount is computed as:

```
boostAmount = maturityBlocksRemaining * boostPricePerBlock
```

new levels default to `boostPricePerBlock = 0`, resulting in `boostAmount = 0`. As a result, users can stake and instantly delegate without paying the intended VTHO boost fee, effectively bypassing the maturity period entirely.

This breaks the economic model of the system and violates the expected requirement that skipping maturity must incur cost.

### Vulnerability Details

Relevant code paths:

During upgrade:

```solidity
function initializeV3(
    address stargate,
    uint8[] memory levelIds,
    uint256[] memory boostPricesPerBlock
) external onlyRole(UPGRADER_ROLE) reinitializer(3) {
    ...
    for (uint256 i; i < levelIds.length; i++) {
        Levels.updateLevelBoostPricePerBlock($, levelIds[i], boostPricesPerBlock[i]);
    }
}
```

Only provided `levelIds` receive boost price updates.

Adding new levels:

```solidity
function addLevel(
    DataTypes.LevelAndSupply memory _levelAndSupply
) external onlyRole(LEVEL_OPERATOR_ROLE) {
    Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
}
```

There is no assignment to set the `boostPricePerBlock` for the new level, so new levels implicitly use the storage default value of `0`.

Because boosting costs 0 for new levels, any user can perform the following sequence and bypass maturity:

{% stepper %}
{% step %}

### Stake into a newly added level

User stakes to a level that was added after the upgrade and therefore has `boostPricePerBlock = 0`.
{% endstep %}

{% step %}

### Automatically boost for free

Because `boostAmount = maturityBlocksRemaining * boostPricePerBlock` and `boostPricePerBlock == 0`, the computed boosting cost is zero.
{% endstep %}

{% step %}

### Instantly delegate

The user delegates immediately, effectively bypassing the intended maturity waiting period without incurring the expected VTHO cost.
{% endstep %}
{% endstepper %}

### Impact

This is a broken system behavior. The core issue is that the logic does not enforce boost price requirements for new levels and there is no functionality to set the `boostPricePerBlock` in a separate transaction for a specific level at creation time. This allows users to bypass maturity without payment and undermines the protocol staking economics.

### Recommendation

Modify `addLevel` to allow setting the `boostPricePerBlock` when creating a new level (for example, include a parameter in `addLevel` or ensure `addLevel` initializes `boostPricePerBlock` to a non-zero value, or require an explicit setter callable only by level operators immediately after creation). Ensure upgrade and initialization paths cover future-added levels or add validation preventing zero-priced boosts for levels where maturity/boosting is expected.

## Proof of Concept

The reporter provided integration tests demonstrating the issue. Run with: yarn hardhat test --network vechain\_solo test/integration/Delegation.test.ts

Two relevant test cases are summarized below.

<details>

<summary>1) Test showing boost is charged for previous levels</summary>

Test excerpt:

```ts
it.only("previous levels are not missing the boostprice.", async () => {
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

    let balanceBefore = await mockedVthoToken.balanceOf(user);
    console.log("Balance before stake and delegate: ", balanceBefore);

    let approveTx = await mockedVthoToken.connect(user).approve(await stargateNFTContract.getAddress(), balanceBefore);
    await approveTx.wait();

    // staking the token here 
    const levelId = 1;
    const levelSpec = await stargateNFTContract.getLevel(levelId);
    const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;

    // staking to get the NFT
    let stakeTx = await stargateContract
        .connect(user)
        .stakeAndDelegate(levelId, deployer.address, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Correctly staked an NFT of level", levelId);

    let balanceAfter = await mockedVthoToken.balanceOf(user);
    console.log("Balance before stake and delegate: ", balanceAfter);
});
```

Test output shows VTHO balance decreased as expected for old levels:

```
Balance before stake and delegate:  87112285931760246646623899502532662132735n
Balance before stake and delegate:  87112285931760246646615555003917976460036n
✔ Newly added levels are missing boostPrice. (188ms)
```

</details>

<details>

<summary>2) Test showing newly added level has boostAmt == 0</summary>

Test excerpt:

```ts
it.only("Newly added levels are missing boostPrice.", async () => {
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

    const levelOperator =(await ethers.getSigners())[5];

    const grantTx = await stargateNFTContract.grantRole(
        await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
        levelOperator.address
    );
    await grantTx.wait();

    const currentLevelIds = await stargateNFTContract.getLevelIds();

    const newLevelAndSupply = {
        level: {
            id: 25,
            name: "My New Level",
            isX: false,
            vetAmountRequiredToStake: ethers.parseEther("1000000"),
            scaledRewardFactor: 150,
            maturityBlocks: 30,
        },
        cap: 872,
        circulatingSupply: 0,
    };

    const expectedLevelId = currentLevelIds[currentLevelIds.length - 1] + 1n;

    // Add new level
    const addLevelTx = await stargateNFTContract
        .connect(levelOperator)
        .addLevel(newLevelAndSupply);
    await addLevelTx.wait();

    await mineBlocks(1);

    // staking the token here 
    const levelId = expectedLevelId;
    const levelSpec = await stargateNFTContract.getLevel(levelId);
    const levelVetAmountRequired = levelSpec.vetAmountRequiredToStake;        

    // staking to get the NFT
    let stakeTx = await stargateContract
        .connect(user)
        .stake(levelId, { value: levelVetAmountRequired });
    await stakeTx.wait();
    log("\n🎉 Correctly staked an NFT of level", levelId);

    const tokenId = await stargateNFTContract.getCurrentTokenId();
    let boostAmt = await stargateNFTContract.boostAmount(tokenId);
    console.log("boostAmt: ", boostAmt);
});
```

Test output shows boost amount is zero for newly added level:

```
app env:  local
boostAmt:  0n
✔ Newly added levels are missing boostPrice. (153ms)
```

</details>

***

If you want, I can:

* Propose a small code patch (diff) to initialize `boostPricePerBlock` in `addLevel` or add a setter restricted to the appropriate role.
* Draft unit tests to prevent regression.
