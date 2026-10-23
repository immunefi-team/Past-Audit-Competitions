# #41894 \[SC-Critical] Incorrect calculation of deposited rewards yeet leads to Staker's not being able to get their staked amount back

**Submitted on Mar 19th 2025 at 08:41:12 UTC by @Oxgritty for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41894
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

* `StakingV2::accumulatedDeptRewardsYeet` is used to calculate the amount of accumulated rewards in yeet that are not distributed yet.
* The problem here is that, this function doesn't account for the fact that when a staker calls `StakingV2::startUnstake` to start unstaking, their balance is still in the contract but subtracted from the `totalSupply`, so `StakingV2::accumulatedDeptRewardsYeet` mistakes the unstake amount as undistributed rewards.

## Vulnerability Details

* When the manager would want to distribute rewards through `StakingV2::executeRewardDistributionYeet`, he would first call the `StakingV2::accumulatedDeptRewardsYeet` to know the amount available for distribution. This function would return an amount which includes the unStakeAmount, which staker unstaked by calling `StakingV2::startUnstake`.
* When the manager will call `StakingV2::executeRewardDistributionYeet`, it will convert (undistributed rewards + unStakeAmount) into `vaultShares`, meaning when the staker would call `StakingV2::unstake` to get his unStakeAmount amount back, this contract won't have enough unStakeAmount of stakingToken and function will revert.

## Impact Details

Stakers not being able to withdraw their staked amount as the contract will not have enough funds.

## References

* When a staker calls `StakingV2::startUnstake` his unstake amount will be [deducted](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L255) from the `totalSupply`.
* This unstake amount is not [deducted](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149) in `StakingV2::accumulatedDeptRewardsYeet`.

## Proof of Concept

## POC: (NOTE: StakingToken is YEET)

### Step 0: Checking Current Balances

* totalSupply = 0
* stakingToken.balanceOf(address(this)) = 0

### Step 1: Staking

1. Alice(a staker) calls `StakingV2::stake` to stake 100e18 Yeet.
2. Bob(another staker) calls `StakingV2::stake` to stake 50e18 Yeet.

Current Balances:

* totalSupply = 150e18
* stakingToken.balanceOf(address(this)) = 150e18

### Step 2: Initiating Unstaking process

Alice initiates unstaking process by calling `StakingV2::startUnstake` where `unStakeAmount=100e18`

Current Balances:

* totalSupply = 50e18
* stakingToken.balanceOf(address(this)) = 150e18

### Step 3: Manager checks amount of stakingToken available for distribution by calling `StakingV2::accumulatedDeptRewardsYeet`. This function will return 100e18(150e18 - 50e18).

### Step 4: Manager calls `StakingV2::executeRewardDistributionYeet` to distribute yeet rewards.

Here the value of (stakingParams.amount0Max + swapData.inputAmount would be equal to 100e18). After the execution of this function, the contract would have 100e18 yeet worth of vault Shares and 50e18 Yeet.

Current Balance:

* totalSupply = 50e18
* stakingToken.balanceOf(address(this)) = 50e18

### Step 5: Finalize unstaking and get unStakeAmount back

Alice calls `StakingV2::unstake` to get her unStakeAmount(100e18) back.\
This function will fail because StakingV2.sol has only 50e18 Yeet.
