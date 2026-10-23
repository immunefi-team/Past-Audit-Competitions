# 60593 sc low no mechanism to set boostpriceperblock for levels added after initialization

**Submitted on Nov 24th 2025 at 08:42:24 UTC by @Brainiac5 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60593
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Location:**\
StargateNFT.sol\
Function: `addLevel()`

**Description:**\
When new NFT levels are added using the `addLevel()` function, there is no mechanism to set the `boostPricePerBlock` value for these levels. The `boostPricePerBlock` is only set for levels added during contract initialization (via `initializeV3`). As a result, any level added after deployment will have an unset boost price.

**Relevant Code Snippet:**

```solidity
function addLevel(
    DataTypes.LevelAndSupply memory _levelAndSupply
) public onlyRole(LEVEL_OPERATOR_ROLE) {
    Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
}
```

**Impact:**

* Users can use the boost feature for levels added after contract initialization for free.

**Suggested Fix:**

* Add a function to set or update `boostPricePerBlock` for any level, not just those initialized in `initializeV3`.
* Alternatively, extend `addLevel()` to accept a `boostPricePerBlock` parameter and set it during level addition.

```diff
   function addLevel(
        DataTypes.LevelAndSupply memory _levelAndSupply
+        uint256 boostPricePerBlock
    ) public onlyRole(LEVEL_OPERATOR_ROLE) {
+        DataTypes.StargateNFTStorage storage $ = _getStargateNFTStorage();

-        Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
+        Levels.addLevel($, _levelAndSupply);
+        Levels.updateLevelBoostPricePerBlock($, $.MAX_LEVEL_ID, boostPricePerBlock);

    }
```

## Proof of Concept

## Proof of Concept

> Add the unit test below to `/test/unit/StargateNFT/Levels.test.ts`

```
        it.only("should be able to add level, but not boostPricePerBlock", async () => {
            const levelOperator = otherAccounts[2];

            const grantTx = await stargateNFTContract.grantRole(
                await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
                levelOperator.address
            );
            await grantTx.wait();

            const currentLevelIds = await stargateNFTContract.getLevelIds();

            const newLevelAndSupply = {
                level: {
                    id: 25, // This id does not matter since it will be replaced by the real one
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

            const boostPricePerBlock = await stargateNFTContract.boostPricePerBlock(expectedLevelId);
            expect(boostPricePerBlock).to.equal(0);
        });
```
