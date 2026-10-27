# #42723 \[SC-Critical] Unstaked Tokens Included in Excess Reward Calculation Can Cause DoS for Unstaking Users

**Submitted on Mar 25th 2025 at 13:45:07 UTC by @x0bserver for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42723
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Description

The `StakeV2` contract's `accumulatedDeptRewardsYeet` function is designed to return the undistributed rewards, i.e., the excess rewards. However, the function does not properly account for users who have initiated unstaking. When a user calls the `startUnstake` function, their tokens are not immediately transferred out of the `StakeV2` contract due to the vesting period. Instead, these tokens are only deducted from `totalSupply`:

```solidity
    function startUnstake(uint256 unStakeAmount) external {  
        require(unStakeAmount > 0, "Amount must be greater than 0");  
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less than the STAKING_LIMIT constant"); // DOS protection  
        _updateRewards(msg.sender);  
        uint256 amount = balanceOf[msg.sender];  
        require(amount >= unStakeAmount, "Insufficient balance");  

        balanceOf[msg.sender] -= unStakeAmount;  
@>      totalSupply -= unStakeAmount;  
```

This means the user's unstaked tokens remain in the contract until the end of the vesting period. After this period, the user can withdraw their tokens through a separate transaction using the `_unstake` function:

```solidity
    function _unstake(uint256 index) private {  
        Vesting memory vesting = vestings[msg.sender][index];  

        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);  
        require(unlockedAmount != 0, "No unlocked amount");  

@>      stakingToken.transfer(msg.sender, unlockedAmount);  
```

This process creates a time gap between when the tokens are deducted from `totalSupply` and when they are actually withdrawn from the contract. During this gap, the unstaked tokens are still present in the contract. While this alone isn't problematic, the `accumulatedDeptRewardsYeet` function fails to account for these tokens.

This function is called during the `executeRewardDistributionYeet` process, which distributes excess rewards to the vault. The function calculates the undistributed rewards using the following logic:

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {  
        return stakingToken.balanceOf(address(this)) - totalSupply;  
    }  
```

Due to the aforementioned gap, this function incorrectly considers tokens in the vesting period as excess rewards. As a result, these tokens can be transferred to the vault via the `executeRewardDistributionYeet` function. This miscalculation DoSes the unstaking process, preventing users from withdrawing their tokens after the vesting period ends.

## Impact

Affected users will be unable to withdraw their staked tokens after the vesting period, causing denial of service (DoS). Additionally, excess tokens meant for users will be misdirected to the vault.

## Mitigation

Update the `accumulatedDeptRewardsYeet` function to exclude tokens in the vesting period from the excess reward calculation. This can be achieved by maintaining a separate record of unstaked tokens currently in vesting and subtracting them from the total contract balance:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {  
    return stakingToken.balanceOf(address(this)) - totalSupply - totalVestingAmount;  
}  
```

Here, `totalVestingAmount` should track the sum of all tokens currently in the vesting process. This ensures that only actual excess rewards are distributed to the vault, preventing DoS during unstaking.

## Proof of Concept

## Attack Scenario

1. User A stakes tokens in the `StakeV2` contract and decides to unstake.
2. User A calls the `startUnstake` function. The tokens are deducted from `totalSupply`, but they remain locked in the contract during the vesting period.
3. The `executeRewardDistributionYeet` function is later called, which uses the `accumulatedDeptRewardsYeet` function to calculate excess rewards.
4. The function mistakenly considers User A's unstaked but still-locked tokens as excess and transfers them to the vault.
5. After the vesting period, User A tries to withdraw their tokens via `_unstake` but finds that the contract lacks sufficient balance due to the previous incorrect reward distribution.
