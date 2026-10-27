# 29047 - \[SC - Insight] Reward is lost when totalSupply

Submitted on Mar 5th 2024 at 18:52:00 UTC by @DuckAstronomer for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29047

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Vulnerability Details

**Affected asset**: governance-main/contracts/voter/gauge/RewardBase.sol

The `notifyRewardAmount()` function within the `GaugeIncentiveController()` contract (derived from `RewardBase.sol`) allows rewards to be sent and distributed to holders of `AToken` based on their eligibility determined by the `EligibilityCriteria` contract.

However, a crucial check is missing in the `notifyRewardAmount()` function. It fails to verify whether `totalSupply == 0` before accepting the reward. This issue could result in the complete loss of the reward or a portion of it, which would then be locked in the `GaugeIncentiveController()` contract's balance indefinitely.

Consider the following scenario:

1. Initially, there was a distribution of **10** `ZeroLend` rewards to `GaugeIncentiveController()`.
2. Subsequently, another **10** `ZeroLend` rewards were distributed after an hour.
3. At this point, `totalSupply` equals **0**.
4. After **13** days, Alice mints **1** `Atoken`, now `totalSupply > 0`. She then waits an additional **14 days** (as defined by the `DURATION` variable in `RewardBase`) and earns **1.4** `ZeroLend`.
5. Consequently, a total of **18.6** `ZeroLend` becomes irreversibly locked in `GaugeIncentiveController()`.

For the mitigation, add a check `require(totalSupply > 0)` to the `notifyRewardAmount()` of `RewardBase`.

## Proof of Concept

To run the Poc put it's code to the `governance-main/test/Gauge.poc.ts` file, generate random private key, and issue the following command:

```
WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Gauge.poc.ts --config hardhat.config.ts --network hardhat
```

```
import { expect } from "chai";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";
import {
  GaugeIncentiveController,
  OmnichainStaking,
  Pool,
  PoolVoter,
  StakingBonus,
  TestnetERC20,
  VestedZeroNFT,
  ZeroLend,
} from "../typechain-types";
import { e18 } from "./fixtures/utils";
import { deployVoters } from "./fixtures/voters";
import { ethers } from "hardhat";
import { time } from "@nomicfoundation/hardhat-network-helpers";

// Put inside governance-main/test/PoolVoter.poc.ts

// Run as:
// WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Gauge.poc.ts --config hardhat.config.ts --network hardhat

describe.only("ZeroLend Immunefi Boost", () => {
  let ant: SignerWithAddress;
  let deployer: SignerWithAddress;
  let now: number;
  let omniStaking: OmnichainStaking;
  let poolVoter: PoolVoter;
  let reserve: TestnetERC20;
  let stakingBonus: StakingBonus;
  let vest: VestedZeroNFT;
  let pool: Pool;
  let aTokenGauge: GaugeIncentiveController;
  let zero: ZeroLend;
  let owner: SignerWithAddress;

  beforeEach(async () => {
    const deployment = await loadFixture(deployVoters);
    ant = deployment.ant;
    now = Math.floor(Date.now() / 1000);
    omniStaking = deployment.governance.omnichainStaking;
    poolVoter = deployment.poolVoter;
    reserve = deployment.lending.erc20;
    stakingBonus = deployment.governance.stakingBonus;
    vest = deployment.governance.vestedZeroNFT;
    zero = deployment.governance.zero;
    pool = deployment.lending.pool;
    aTokenGauge = deployment.aTokenGauge;
    owner = deployment.governance.lending.owner;
    deployer = deployment.governance.deployer;

    // deployer should be able to mint a nft for another user
    await vest.mint(
      ant.address,
      e18 * 20n, // 20 ZERO linear vesting
      0, // 0 ZERO upfront
      1000, // linear duration - 1000 seconds
      0, // cliff duration - 0 seconds
      now + 1000, // unlock date
      true, // penalty -> false
      0
    );

    // stake nft on behalf of the ant
    await vest
      .connect(ant)
      ["safeTransferFrom(address,address,uint256)"](
        ant.address,
        stakingBonus.target,
        1
      );

    // there should now be some voting power for the user to play with
    // ant voting power is ~ 19 ether
    expect(await omniStaking.balanceOf(ant.address)).lessThan(e18 * 20n);
  });

  it("Stuck reward in Gauge", async function () {
    let gaugeToken = await aTokenGauge.aToken();
    let atoken = await ethers.getContractAt("AToken", gaugeToken);

    // Mint 1 WETH to ant
    await reserve.connect(owner)["mint(address,uint256)"](ant.address, 1n * e18);
    expect(await reserve.balanceOf(ant.address)).eq(1n * e18);
    
    // Distribute reward, 10 ZeroLend when totalSupply == 0
    await zero.connect(deployer).approve(aTokenGauge.target, 10n * e18);
    await aTokenGauge.connect(deployer).notifyRewardAmount(zero.target, 10n * e18);

    // + 1 Hour
    await time.increase(3600);

    // Distribute reward, 10 ZeroLend when totalSupply == 0
    await zero.connect(deployer).approve(aTokenGauge.target, 10n * e18);
    await aTokenGauge.connect(deployer).notifyRewardAmount(zero.target, 10n * e18);

    // + 13 Days
    await time.increase(1123200);

    // Mint AToken from WETH for ant
    await reserve.connect(ant).approve(pool.target, 1n * e18);
    await pool.connect(ant).supply(reserve.target, 1n * e18, ant.address, 0n);
    expect(await atoken.balanceOf(ant.address)).greaterThan(0);

    // + 14 Days
    await time.increase(1209600);

    // ant gets reward for 1 day
    let earned = await aTokenGauge.earned(zero.target, ant.address);
    console.log(`${earned}`);

    // The rest of the Reward stuck and not recoverable
    expect(await zero.balanceOf(aTokenGauge.target)).greaterThan(10n * e18);
  });
});
```
