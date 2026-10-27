# 57718 sc low staking tier error using erc4626 shares rather than assets to determine staking tiers leads to long term distortion in fees and commissions&#x20;

**Submitted on Oct 28th 2025 at 12:27:04 UTC by @s8olidity for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57718
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>

## Description

### Brief / Intro

BelongCheckIn uses the "number of staked shares" to apply the "LONG asset thresholds (50k, 250k, 500k, 1M)" logic of `Helper.stakingTiers`. Early entrants receive more shares for the same amount of assets when share prices are low, while later entrants receive fewer shares for the same amount of assets after prices rise. This results in long-term discrepancies in platform fees and promotional commissions, leading to significant economic misalignment.

### Vulnerability Details

* Code locations:
  * contracts/v2/platform/BelongCheckIn.sol:395 (Venue deposit path uses tier)
  * contracts/v2/platform/BelongCheckIn.sol:533 (Promotion settlement path uses tier)
  * contracts/v2/utils/Helper.sol:39 (StakingTiers threshold based on "asset quantity")
* Issues:
  * `staking.balanceOf(addr)` returns ERC4626 shares, not assets.
  * `Helper.stakingTiers` expects the input value to be in LONG assets (50\_000e18 ... 1\_000\_000e18).
  * When the treasury distributes rewards (`distributeRewards`), share prices rise. New entrants receive fewer shares for the same assets, resulting in a lower `balanceOf` and incorrect placement in a lower tier. Early entrants who deposited when price-per-share was lower have more shares for the same asset amount and therefore receive higher-tier benefits.

### Impact

The venue deposit fee rate (`depositFeePercentage`) and promoter settlement fee rate (`PromoterStakingRewardInfo.usdcPercentage/longPercentage`) are calculated using the tier mapping. Using shares instead of assets produces a persistent miscalculation of platform fees and commissions.

* Systemic bias caused by a measurement error; does not require permissions or external conditions.
* Users with the same amount of assets can be misclassified over time due to different deposit timing, causing persistent economic distortion:
  * Underestimation of platform fees (less from higher-tier users) or
  * Overestimation of customer/promoter rewards.
* Recommended Severity: High (persistent economic deviation affects platform fees and profit distributions as TVL/trading volume grow)

## References

* contracts/v2/platform/BelongCheckIn.sol:395, 533
* contracts/v2/utils/Helper.sol:39

## Proof of Concept

```ts
// test/v2/platform/staking-tier-misclassification.test.ts

import { ethers } from 'hardhat';
import { expect } from 'chai';
import { deployLONG, deployStaking } from '../../../helpers/deployFixtures';
import { LONG, Staking } from '../../../typechain-types';

describe('Staking tiers misclassification (shares vs assets)', () => {
  it('Same assets, different deposit timing => different tiers by shares', async () => {
    const [deployer, early, late] = await ethers.getSigners();

    // Deploy LONG + staking(ERC4626)
    const long: LONG = await deployLONG(deployer.address, deployer.address, deployer.address);
    const staking: Staking = await deployStaking(deployer.address, deployer.address, long.address);

    // Mint tokens to users
    const ONE_M = ethers.utils.parseEther('1000000');
    await long.transfer(early.address, ONE_M);
    await long.transfer(late.address, ONE_M);

    // early deposits first: shares == assets initially (p = 1)
    await long.connect(early).approve(staking.address, ONE_M);
    await (await staking.connect(early).deposit(ONE_M, early.address)).wait();

    const earlyShares = await staking.balanceOf(early.address);
    expect(earlyShares).to.equal(ONE_M); // price per share = 1

    // Distribute rewards: raise price per share
    await long.approve(staking.address, ONE_M);
    await (await staking.distributeRewards(ONE_M)).wait();
    // Now totalAssets doubled, totalSupply unchanged => pps > 1

    // late deposits same 1M assets, will get fewer shares
    await long.connect(late).approve(staking.address, ONE_M);
    await (await staking.connect(late).deposit(ONE_M, late.address)).wait();

    const lateShares = await staking.balanceOf(late.address);
    expect(lateShares).to.lt(earlyShares); // fewer shares for same assets

    // Helper.stakingTiers expects ASSETS thresholds (50k..1M)
    // But BelongCheckIn uses balanceOf(shares) to pick the tier.
    // So early qualifies tier by shares >= 1M, late does not, despite same 1M asset.
    expect(earlyShares.gte(ONE_M)).to.be.true;
    expect(lateShares.gte(ONE_M)).to.be.false; // wrong: same assets should be equal-tier

    // This demonstrates tier misclassification purely due to shares/price drift.
  });
});
```
