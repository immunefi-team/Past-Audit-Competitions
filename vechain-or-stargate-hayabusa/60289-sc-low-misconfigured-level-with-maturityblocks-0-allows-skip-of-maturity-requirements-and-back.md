# 60289 sc low misconfigured level with maturityblocks 0 allows skip of maturity requirements and backrun minting

**Submitted on Nov 21st 2025 at 01:29:33 UTC by @MoZi for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60289
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/libraries/Levels.sol>
* **Impacts:**
  * Unintended economic advantage

## Description

## Brief/Intro

The protocol `Levels::addLevel` function allows the `LEVEL_OPERATOR_ROLE` to introduce new NFT levels. However, the function does not validate the `_levelAndSupply.level.maturityBlocks` input paramater, allowing a new level to be added with `maturityBlocks` = 0.

## Vulnerability Details

Inside `Levels::addLevel`, the function validates only:

* `_levelAndSupply.level.name` length
* `_levelAndSupply.level.vetAmountRequiredToStake` input
* `_levelAndSupply.circulatingSupply`
* `_levelAndSupply.cap`

But no validation is applied to:

* `_levelAndSupply.level.maturityBlocks`
* `_levelAndSupply.level.scaledRewardFactor`

The current`Stargate` and `StargateNFT` contract does not implement the `Levels::updateLevel` from libraries to prevent when this scenario occur. It allowing user to backrun and mint the misconfigured level before protocol pause and upgrade the contract to implement function `Levels::updateLevel`. The updated level is only applies to the next minting and NFTs that have already been minted will not be affected.

This creates an unintended economic advantage for any user who immediately backruns the level addition and mints the NFT.

## Impact Details

when `LEVEL_OPERATOR_ROLE` unintentionally sets maturityBlocks = 0, users can:

* Immediately stake to mint the NFT
* Receive an NFT that is already considered matured
* Delegate instantly without waiting or paying boost

and if the added level is a level that has a high `scaledRewardFactor` then the user who immediately backruns and mints the NFT will get a huge advantage.

## References

<https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/libraries/Levels.sol#L88>

## Proof of Concept

## Proof of Concept

Step-by-step explanations:

1. `LEVEL_OPERATOR_ROLE` call `StargateNFT::addLevel` and mistakenly sets maturityBlocks = 0.
2. A user monitoring mempool detects the misconfiguration.
3. User backruns or immediately call `Stargate::stake(levelId)` to mint the NFT.
4. Minted NFT is exempt from the maturity requirement.
5. User instantly call `Stargate::delegate(TokenId)` without waiting.
6. The protocol cannot fix the misconfigured NFT Level without: pausing the contract, deploying upgrade , adding a new function for `updateLevel`.

Even though the misconfigured NFT level has been updated, the ones that have been minted will not change.

```typescript
   it("Misconfigured Level With maturityBlocks = 0 Allows Permanent Skip of Maturity Requirements and Backrun Minting", async () => {
        // 1. `LEVEL_OPERATOR_ROLE` call `StargateNFT::addLevel` and mistakenly sets maturityBlocks = 0.
        const Level = {
            name: "NOMAD",
            isX: false,
            id: 11,
            maturityBlocks: 0,
            scaledRewardFactor: 2,
            vetAmountRequiredToStake: ethers.parseEther("2")
        }
        tx = await stargateNFTMockContract.connect(deployer).helper__setLevel(Level);
        await tx.wait();
        
        const LevelInfo = await stargateNFTMockContract.getLevel(11);
        console.log("Name: ", LevelInfo.name);
        console.log("Maturity blocks: ", LevelInfo.maturityBlocks);
        console.log("Scaled reward factor: ", LevelInfo.scaledRewardFactor);
        console.log("Stake amount required: ", LevelInfo.vetAmountRequiredToStake);

        console.log("");
        
        // 2. A user monitoring mempool detects the misconfiguration.
        // 3. User backruns or immediately call `Stargate::stake(levelId)` to mint the NFT.
        const levelSpec = await stargateNFTMockContract.getLevel(11);
        tx = await stargateContract.connect(user).stake(11, {
            value: levelSpec.vetAmountRequiredToStake
        });
        await tx.wait();
        
        const TokenId = await stargateNFTMockContract.getCurrentTokenId();
        const Token = {
            tokenId: TokenId,
            levelId: 11,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("2"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0
        }
        await stargateNFTMockContract.helper__setToken(Token);
        const TokenInfo = await stargateNFTMockContract.getToken(TokenId);
        console.log("tokenId: ", TokenInfo.tokenId);
        console.log("levelId: ", TokenInfo.levelId);

        // 4. Minted NFT is exempt from the maturity requirement.
        // 5. User instantly call `Stargate::delegate(TokenId)` without waiting.
        tx = await stargateContract.connect(user).delegate(TokenId, validator.address);
        await tx.wait();

        tx = await protocolStakerMockContract.helper__setValidationCompletedPeriods(
            validator.address,
            10
        );
        await tx.wait();

        // assert the delegation is active
        expect(await stargateContract.getDelegationStatus(TokenId)).to.equal(
            DELEGATION_STATUS_ACTIVE
        );
        await tx.wait();

        await protocolStakerMockContract.helper__setDelegationIsLocked(1, true);
        const Delegation = await stargateContract.getDelegationDetails(TokenId);
        console.log("Delegation Id: ", Delegation.delegationId);
        console.log("Validator: ", Delegation.validator);
        console.log("Delegated Stake: ", Delegation.stake);
        console.log("Probability Multiplier: ", Delegation.probabilityMultiplier);
        console.log("Start Period: ", Delegation.startPeriod);
        console.log("End Period: ", Delegation.endPeriod);
        console.log("is Locked ?: ", Delegation.isLocked);
        console.log("Delegation status: ", Delegation.status);

        // 6. The protocol cannot fix the misconfigured NFT Level without: 
        // pausing the contract, deploying upgrade , adding a new function for `updateLevel`.
    });
```
