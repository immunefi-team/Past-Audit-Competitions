# #41488 \[SC-Insight] In \`StakeV2.sol\` there exists a critical flaw that allows adversaries to earn more rewards than should be possible for a period of having staked minimal tokens.

**Submitted on Mar 15th 2025 at 20:45:13 UTC by @Oxbakeng for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41488
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Theft of unearned yield/rewards

## Description

## Brief/Intro

In `StakeV2.sol`, there exists a critical flaw in the reward distribution mechanism that allows adversaries to earn disproportionate rewards by frontrunning reward distribution. Adversaries can stake the minimal amount possible during normal operation, then frontrun reward distributions with a much larger stakes to claim rewards they haven't legitimately earned, creating an unfair advantage and effectively stealing yield from honest long-term stakers.

## Vulnerability Details

The vulnerability stems from how the contract calculates and distributes rewards. The core issue is that rewards are calculated based on a user's current balance multiplied by the difference between the current reward index and the user's last recorded index, without considering how long tokens have been staked.

1. The vulnerable code flow involves several functions:

```solidity
function stake(uint256 amount) external {
    require(amount > 0, "Amount must be greater than 0");
    _updateRewards(msg.sender);  // Updates rewards based on previous balance
    
    stakingToken.transferFrom(msg.sender, address(this), amount);
    
    balanceOf[msg.sender] += amount;  // Updates balance for FUTURE rewards
    totalSupply += amount;
    emit Stake(msg.sender, amount);
}
```

2. The \_updateRewards() function updates the user's earned rewards and sets their reward index to the current global index:

```solidity
function _updateRewards(address account) private {
    earned[account] += _calculateRewards(account);
    rewardIndexOf[account] = rewardIndex;
}
```

3. During reward distribution, the global reward index is updated based on the current total supply:

```solidity
function _handleVaultShares(uint256 vaultSharesMinted) internal {
    require(vaultSharesMinted != 0, "No vault shares minted");
    totalVaultShares += vaultSharesMinted;
    
    // Update reward index
    if (totalSupply > 0) {
        rewardIndex += (vaultSharesMinted * MULTIPLIER) / totalSupply;
    }
}
```

4. When calculating rewards, the contract uses the user's current balance:

```solidity
function _calculateRewards(address account) private view returns (uint256) {
    uint256 shares = balanceOf[account];
    return (shares * (rewardIndex - rewardIndexOf[account])) / MULTIPLIER;
}
```

The key vulnerability is that when a user stakes right before a reward distribution, their newly staked tokens are included in:

* The `totalSupply` used to calculate the reward index update (diluting rewards for everyone).
* Their `balanceOf[account]` when they later claim rewards.

This allows them to earn rewards on tokens that weren't staked during the actual reward accumulation period.

## Impact Details

This vulnerability allows malicious users to:

* Stake the most minimal amount allowed (i.e 1 token) during normal operation.
* Monitor for upcoming reward distribution transactions.
* Frontrun these transactions with large stakes.
* Earn rewards on their entire stake as if it had been staked the entire time.
* Potentially unstake shortly after to repeat the process.

The economic impact is significant:

* Honest long term stakers receive fewer rewards than they should.
* Malicious users receive more rewards than they deserve.
* The protocol's reward system is manipulated.
* The vulnerability can be exploited repeatedly further compromising the staking system.

I believe this issue is a HIGH Severity issue as it qualifies as "Theft of unclaimed yield", as it allows adversaries to claim rewards they haven't legitimately earned.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>

## Proof of Concept

## Proof of Concept

1. Attacker stakes the most minimal amount allowed/possible i.e 1 wei token amount.
2. Honest users stake large amount i.e 10k+ tokens.
3. Adversary monitors the berachain for pending reward distribution transactions, alternatively the routine for reward distribution.
4. Adversary frontruns with 10k tokens or stakes these tokens on the period distribution will most likely happen, either or will work.
5. The end result is the adversary gaining more than they should after staking a minimal amount all along vs honest users who staked much more and end up getting way less than they should compared to the adversary getting way more unfairly.
6. Adversary can unstake the large portion to begin the vesting period and unstake the amount, then repeat when rewards are due again, creating an unfair cycle of rewards theft while attacker remains liquid each time entire tokens are unlocked, meanwhile honest stakers have staked indefinitely.
