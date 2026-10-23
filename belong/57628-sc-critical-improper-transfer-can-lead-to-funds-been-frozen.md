# 57628 sc critical improper transfer can lead to funds been frozen

**Submitted on Oct 27th 2025 at 17:37:10 UTC by @bugdaddy96 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57628
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Transferring sLONG shares to another address creates a permanent desync between ERC20 balances and internal stake tracking, preventing recipients from withdrawing funds without paying a 10% penalty.

### Vulnerability Details

The staking contract records stakes in an internal mapping but does not override \_update() on transfers to sync internal stake tracking with ERC20 balance changes:

```solidity
mapping(address staker => Stake[] times) public stakes;
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

Because transfers of the ERC20 shares do not update the stakes mapping, transferred shares remain attributed to the original staker's stakes array. This creates a desync between the ERC20 balanceOf and the internal stake timestamps.

### Impact Details

* Users who receive transferred shares cannot withdraw them without paying the emergency-withdraw penalty (10%) until the min stake period requirement is satisfied according to the original owner's stake timestamp.
* Breaks expected ERC4626 behavior for transferred shares.
* Affects composability and DeFi integrations that assume transfers preserve withdrawability and stake timestamps.

## Proof of Concept

```javascript
import { ethers } from 'hardhat';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { expect } from 'chai';
import { LONG, Staking } from '../../../typechain-types';
import { deployLONG, deployStaking } from '../../../helpers/deployFixtures';

describe('Staking Transfer Desync Bug POC', () => {
  async function fixture() {
    const [admin, treasury, pauser, alice, bob, charlie] = await ethers.getSigners();

    const long: LONG = await deployLONG(admin.address, admin.address, pauser.address);
    const staking: Staking = await deployStaking(admin.address, treasury.address, long.address);

    return {
      admin,
      treasury,
      pauser,
      alice,
      bob,
      charlie,
      long,
      staking,
    };
  }

  describe('Bug: Transfer breaks stake tracking', () => {
    it('POC: Transferred shares cannot be withdrawn without penalty', async () => {
      const { staking, long, admin, treasury, alice, bob } = await loadFixture(fixture);

      const depositAmount = ethers.utils.parseEther('1000');

      await long.connect(admin).transfer(alice.address, depositAmount);
      await long.connect(alice).approve(staking.address, depositAmount);
      await staking.connect(alice).deposit(depositAmount, alice.address);
      
      expect(await staking.balanceOf(alice.address)).to.equal(depositAmount);
      expect((await staking.stakes(alice.address, 0)).shares).to.equal(depositAmount);

      await staking.connect(alice).transfer(bob.address, depositAmount);

      expect(await staking.balanceOf(alice.address)).to.equal(0);
      expect(await staking.balanceOf(bob.address)).to.equal(depositAmount);
      expect((await staking.stakes(alice.address, 0)).shares).to.equal(depositAmount);

      await staking.connect(admin).setMinStakePeriod(1);
      await ethers.provider.send('evm_mine', []);

      await expect(
        staking.connect(bob).withdraw(depositAmount, bob.address, bob.address)
      ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

      const balanceBefore = await long.balanceOf(bob.address);
      await staking.connect(bob).emergencyWithdraw(depositAmount, bob.address, bob.address);
      const balanceAfter = await long.balanceOf(bob.address);

      const penalty = depositAmount.mul(1000).div(10000);
      const expectedPayout = depositAmount.sub(penalty);

      expect(balanceAfter).to.equal(expectedPayout);
    });

    it('POC: Multiple transfers compound the issue', async () => {
      const { staking, long, admin, alice, bob, charlie } = await loadFixture(fixture);

      const depositAmount = ethers.utils.parseEther('1000');
      
      await long.connect(admin).transfer(alice.address, depositAmount);
      await long.connect(alice).approve(staking.address, depositAmount);
      await staking.connect(alice).deposit(depositAmount, alice.address);

      await staking.connect(alice).transfer(bob.address, depositAmount.div(2));

      await staking.connect(bob).transfer(charlie.address, depositAmount.div(2));

      await staking.connect(admin).setMinStakePeriod(1);
      await ethers.provider.send('evm_mine', []);
      
      await expect(
        staking.connect(bob).withdraw(0, bob.address, bob.address)
      ).to.not.be.reverted;

      await expect(
        staking.connect(charlie).withdraw(depositAmount.div(2), charlie.address, charlie.address)
      ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
    });

    it('Root cause: Missing _update() override', async () => {
      const { staking, long, admin, alice, bob } = await loadFixture(fixture);

      const depositAmount = ethers.utils.parseEther('1000');

      await long.connect(admin).transfer(alice.address, depositAmount);
      await long.connect(alice).approve(staking.address, depositAmount);
      await staking.connect(alice).deposit(depositAmount, alice.address);

      const aliceStakesBeforeTransfer = (await staking.stakes(alice.address, 0)).shares;
      
      await staking.connect(alice).transfer(bob.address, depositAmount);

      const aliceStakesAfterTransfer = (await staking.stakes(alice.address, 0)).shares;
      
      expect(aliceStakesBeforeTransfer).to.equal(aliceStakesAfterTransfer);
    });
  });
});
```
