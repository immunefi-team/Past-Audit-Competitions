# 59841 sc low the newly added level cannot have its boost price set because the updatelevelboostpriceperblock function is not exposed

**Submitted on Nov 16th 2025 at 10:01:13 UTC by @shaflow1 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59841
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>

## Description

### Brief/Intro

In the `StargateNFT` contract, the `LEVEL_OPERATOR_ROLE` is allowed to call `addLevel` to add new levels. However, since the `updateLevelBoostPricePerBlock` function is not exposed, the newly added level cannot have its `BoostPricePerBlock` set. As a result, users can bypass the maturity period for free and start delegating immediately.

### Vulnerability Details

The `addLevel` function is implemented as follows:

```solidity
/// @inheritdoc IStargateNFT
function addLevel(
    DataTypes.LevelAndSupply memory _levelAndSupply
) public onlyRole(LEVEL_OPERATOR_ROLE) {
    Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
}
```

`LEVEL_OPERATOR_ROLE` can call `addLevel` to add a new level. This function sets the level’s cap, level data, and circulating supply, but it does **not** set the `boostPricePerBlock`.

In the library, the `updateLevelBoostPricePerBlock` function used to update `boostPricePerBlock` is only called during the V3 contract initialization. After that, it cannot be invoked. As a result, no one is able to set the `boostPricePerBlock` for newly added levels.

Reference lines:

* StargateNFT addLevel: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/StargateNFT.sol#L302>
* Levels library reference: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/libraries/Levels.sol#L180>

## Impact Details

{% hint style="warning" %}
For newly added levels, `boostPricePerBlock` is always zero, so users can call the `boost` function to skip the maturity period for free.
{% endhint %}

## Proof of Concept

<details>

<summary>Test PoC (click to expand)</summary>

```solidity
it("add new level", async () => {
    const levelOperator = otherAccounts[3];

    // grant role
    const grantTx = await stargateNFTContract.grantRole(
        await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
        levelOperator.address
    );
    await grantTx.wait();

    // current level ids
    const currentLevelIds = await stargateNFTContract.getLevelIds();
    const expectedNewLevelId = currentLevelIds[currentLevelIds.length - 1] + 1n;

    // new level data
    const newLevel = {
        level: {
            id: 26, // placeholder
            name: "Boost Test Level",
            isX: false,
            vetAmountRequiredToStake: ethers.parseEther("1234"),
            scaledRewardFactor: 200,
            maturityBlocks: 50,
        },
        cap: 1000,
        circulatingSupply: 0,
    };

    // add level
    const tx = await stargateNFTContract.connect(levelOperator).addLevel(newLevel);
    await tx.wait();

    // query boost price
    const price = await stargateNFTContract.boostPricePerBlock(expectedNewLevelId);

    console.log("Boost price =", price);
    // Mint a level-26 NFT
    const staker = otherAccounts[4];
    const stakeTx = await stargateContract.connect(staker).stake(expectedNewLevelId, {
        value: newLevel.level.vetAmountRequiredToStake,
    });
    await stakeTx.wait();

    const tokenId = await stargateNFTContract.tokenOfOwnerByIndex(staker.address, 0);

    const boostBefore = await stargateNFTContract.boostAmount(tokenId);
    expect(boostBefore).to.equal(0n);

    const maturityBefore = await stargateNFTContract.maturityPeriodEndBlock(tokenId);
    console.log("Maturity period end before boost:", maturityBefore.toString());

    await stargateNFTContract.connect(staker).boost(tokenId);

    const maturityAfter = await stargateNFTContract.maturityPeriodEndBlock(tokenId);
    console.log("Maturity period end after boost:", maturityAfter.toString());
});
```

To run the test, add the code to the end of `describe("Add level", () => {` in the file `packages/contracts/test/unit/StargateNFT/Levels.test.ts`.

The test verifies that the newly added level cannot have its `boostPricePerBlock` set, allowing users to skip the maturity period for free.

</details>
