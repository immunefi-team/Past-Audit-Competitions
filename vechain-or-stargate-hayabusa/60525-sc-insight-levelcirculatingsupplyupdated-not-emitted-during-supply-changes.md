# 60525 sc insight levelcirculatingsupplyupdated not emitted during supply changes

**Submitted on Nov 23rd 2025 at 19:11:24 UTC by @Rhaydden for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60525
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/libraries/Levels.sol>
* **Impacts:**

## Description

## Finding description and impact

`LevelCirculatingSupplyUpdated` event is [documented](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/interfaces/IStargateNFT.sol#L32) and exposed in `IStargateNFT` as being emitted “when the circulating supply of a token level is updated.” In the current implementation, the event is only emitted on level creation (`addLevel`) and not when circulating supply actually changes thereafter (mint, burn, migrate).

In [Levels.sol](https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/blob/e9c0bc9b0f24dc0c44de273181d9a99aaf2c31b0/packages/contracts/contracts/StargateNFT/libraries/Levels.sol#L123), the event is emitted once inside `addLevel`:

```solidity
emit LevelCirculatingSupplyUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.circulatingSupply);
```

Then all subsequent supply changes go through `_incrementLevelCirculatingSupply` / `_decrementLevelCirculatingSupply`, which call `_checkpointLevelCirculatingSupply`:

```solidity
function _incrementLevelCirculatingSupply(...) internal {
    uint208 circulatingSupply = _getCirculatingSupply($, _levelId);
    _checkpointLevelCirculatingSupply($, _levelId, circulatingSupply + 1);
}

function _decrementLevelCirculatingSupply(...) internal {
    uint208 circulatingSupply = _getCirculatingSupply($, _levelId);
    _checkpointLevelCirculatingSupply($, _levelId, circulatingSupply - 1);
}

function _checkpointLevelCirculatingSupply(...) internal {
    $.circulatingSupply[_levelId].push(Clock._clock(), _circulatingSupply);
}
```

As we see, no event is emitted in these paths.

## Impact

Any offchain indexers and dApps relying on `LevelCirculatingSupplyUpdated` to track per-level supply will get incorrect values after the initial add. They’ll observe one initial “0 → initialSupply” event and then miss all later increments/decrements. The on-chain state is correct and queryable through the checkpoints (`getLevelsCirculatingSupplies` and `getCirculatingSupplyAtBlock`), and consumers could reconstruct supply from `TokenMinted`/`TokenBurned`. Albeit, the public interface and natspec promise an event-based feed which doesnt actually exist post creation.

Severity: Insight. Optimizations and Enhancements

## Recommended mitigation steps

Emit `LevelCirculatingSupplyUpdated` whenever supply changes.

## Proof of Concept

## Proof of concept

Create and add the poc below to a `packages/contracts/test/unit/StargateNFT/CirculatingSupplyEvents.test.ts` file:

```ts
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers } from "hardhat";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import {
  getOrDeployContracts,
} from "../../helpers";
import {
  Stargate,
  StargateNFT,
  TokenAuctionMock,
  TokenAuctionMock__factory,
} from "../../../typechain-types";

// PoC: LevelCirculatingSupplyUpdated is not emitted when supply actually changes (mint/burn/migrate)
// but is emitted on addLevel

describe("poc: LevelCirculatingSupplyUpdated emission behavior", () => {
  let deployer: HardhatEthersSigner;
  let user: HardhatEthersSigner;
  let otherAccounts: HardhatEthersSigner[];
  let stargateNFTContract: StargateNFT;
  let stargateContract: Stargate;

  beforeEach(async () => {
    [deployer, user, ...otherAccounts] = await ethers.getSigners();
  });

  describe("Mint/Burn do not emit LevelCirculatingSupplyUpdated", () => {
    beforeEach(async () => {
      const config = createLocalConfig();
      // Allow calling mint/burn directly by setting stargate to deployer for tests
      config.STARGATE_CONTRACT_ADDRESS = deployer.address;

      const contracts = await getOrDeployContracts({ forceDeploy: true, config });
      stargateNFTContract = contracts.stargateNFTContract;
      stargateContract = contracts.stargateContract;
    });

    it("mint does not emit LevelCirculatingSupplyUpdated", async () => {
      const level = await stargateNFTContract.getLevel(1);

      const txMint = stargateNFTContract
        .connect(deployer)
        .mint(level.id, user.address);

      await expect(txMint)
        .to.emit(stargateNFTContract, "TokenMinted")
        .withArgs(
          user.address,
          level.id,
          false,
          (await stargateNFTContract.getCurrentTokenId()) + 1n,
          level.vetAmountRequiredToStake
        );

      await expect(txMint).to.not.emit(
        stargateNFTContract,
        "LevelCirculatingSupplyUpdated"
      );

      // sanity: circulating supply increased
      const supplyAfter = await stargateNFTContract.getLevelSupply(level.id);
      expect(supplyAfter.circulating).to.equal(1);
    });

    it("burn does not emit LevelCirculatingSupplyUpdated", async () => {
      const level = await stargateNFTContract.getLevel(1);
      const tokenId = (await stargateNFTContract.getCurrentTokenId()) + 1n;
      await (await stargateNFTContract.connect(deployer).mint(level.id, user.address)).wait();

      const txBurn = stargateNFTContract.connect(deployer).burn(tokenId);

      await expect(txBurn)
        .to.emit(stargateNFTContract, "TokenBurned")
        .withArgs(user.address, level.id, tokenId, level.vetAmountRequiredToStake);

      await expect(txBurn).to.not.emit(
        stargateNFTContract,
        "LevelCirculatingSupplyUpdated"
      );

      // sanity: circulating supply decreased
      const supplyAfter = await stargateNFTContract.getLevelSupply(level.id);
      expect(supplyAfter.circulating).to.equal(0);
    });
  });

  describe("Migrate does not emit LevelCirculatingSupplyUpdated", () => {
    let legacyNodesMock: TokenAuctionMock;

    beforeEach(async () => {
      const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
      legacyNodesMock = await legacyNodesMockFactory.deploy();
      await legacyNodesMock.waitForDeployment();

      const config = createLocalConfig();
      config.STARGATE_CONTRACT_ADDRESS = deployer.address;
      // Override legacy nodes address with the mock so we can control metadata
      config.TOKEN_AUCTION_CONTRACT_ADDRESS = await legacyNodesMock.getAddress();

      const contracts = await getOrDeployContracts({ forceDeploy: true, config });
      stargateNFTContract = contracts.stargateNFTContract;
      stargateContract = contracts.stargateContract;
    });

    it("migrate does not emit LevelCirculatingSupplyUpdated", async () => {
      const LEVEL_ID = 4;
      const TOKEN_ID = 500; // <= LEGACY_LAST_TOKEN_ID so it can be preserved

      // Prepare legacy token metadata so migrate can succeed
      await (await legacyNodesMock.helper__setMetadata(TOKEN_ID, {
        owner: user.address,
        strengthLevel: LEVEL_ID,
        onUpgrade: false,
        isOnAuction: false,
        lastTransferTime: 0,
        createdAt: 0,
        updatedAt: 0,
      })).wait();

      const level = await stargateNFTContract.getLevel(LEVEL_ID);

      const txMigrate = stargateNFTContract.connect(deployer).migrate(TOKEN_ID);

      await expect(txMigrate)
        .to.emit(stargateNFTContract, "TokenMinted")
        .withArgs(user.address, LEVEL_ID, true, TOKEN_ID, level.vetAmountRequiredToStake);

      await expect(txMigrate).to.not.emit(
        stargateNFTContract,
        "LevelCirculatingSupplyUpdated"
      );

      const supplyAfter = await stargateNFTContract.getLevelSupply(LEVEL_ID);
      expect(supplyAfter.circulating).to.equal(1);
    });
  });

  describe("Sanity: addLevel does emit LevelCirculatingSupplyUpdated", () => {
    let levelOperator: HardhatEthersSigner;

    beforeEach(async () => {
      const config = createLocalConfig();
      // Any address can be level operator if granted; deployer will grant to user
      const contracts = await getOrDeployContracts({ forceDeploy: true, config });
      stargateNFTContract = contracts.stargateNFTContract;
      stargateContract = contracts.stargateContract;
      levelOperator = otherAccounts[0];

      await (await stargateNFTContract.grantRole(
        await stargateNFTContract.LEVEL_OPERATOR_ROLE(),
        levelOperator.address
      )).wait();
    });

    it("addLevel emits LevelCirculatingSupplyUpdated once", async () => {
      const currentIds = await stargateNFTContract.getLevelIds();
      const expectedNewId = currentIds[currentIds.length - 1] + 1n;

      const tx = stargateNFTContract.connect(levelOperator).addLevel({
        level: {
          id: 123, // ignored
          name: "My New Level",
          isX: false,
          vetAmountRequiredToStake: ethers.parseEther("42"),
          scaledRewardFactor: 150,
          maturityBlocks: 10,
        },
        cap: 5,
        circulatingSupply: 2,
      });

      await expect(tx)
        .to.emit(stargateNFTContract, "LevelCirculatingSupplyUpdated")
        .withArgs(expectedNewId, 0, 2);
    });
  });
});

```
