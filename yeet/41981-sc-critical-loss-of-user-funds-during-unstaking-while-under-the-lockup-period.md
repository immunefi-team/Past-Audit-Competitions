# #41981 \[SC-Critical] Loss of user funds during unstaking, while under the lockup period

**Submitted on Mar 19th 2025 at 18:59:12 UTC by @Oxodus for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41981
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Users can lose their funds when they are unstaking and their funds are in lockup because the funds will appear as accumulated rewards and therefore can be distributed as rewards to other users, causing the users who have unstaked their tokens to lose their funds.

## Vulnerability Details

In the `StakeV2` contract, users are allowed to stake their yeet tokens and receive rewards. Users can also unstake their tokens by calling the `startUnstake` function, which prepares the funds for unstaking by locking the funds for the vesting period. However, while preparing for the unstaking, the function also has the following line of code `totalSupply -= unStakeAmount` with removes the amount to be unstaked from the contract's token supply of the staking token but doesn't send them to the user since they are still under lockup period. In the same contract, we also have a function `StakeV2::accumulatedDeptRewardsYeet` which according to the docs is meant to:

> The function used to calculate the accumulated rewards that gets return by the zapper since swaps are not 100% efficient

and:

> @return The accumulated rewards in YEET tokens

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

The function above instead of only accounting for the staking tokens that are returned by the zapper, it will also include the user funds that are in lockup and are awaiting the vesting period to end. The issue with this function is that the return value(Which will include the users funds), is called by the `executeRewardDistributionYeet` which distributes the rewards to the users

```solidity
uint256 accRevToken0 = accumulatedDeptRewardsYeet();
```

This will effectively distribute the users funds as rewards to other users who have staked in the contract. Although the `executeRewardDistributionYeet` is a privileged function, this is still an issue because the function will always behave as not expected and cause the user to lose their funds.

## Impact Details

The impact of this issue is critical since users with their stake awaiting the vesting period to end so that they can regain their funds will end up losing their funds when the manager calls the `executeRewardDistributionYeet` intending to distribute any rewards in the contract due to the incorrect accounting of the `accumulatedDeptRewardsYeet()` function.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L148\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L158\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L255>

## Proof of Concept

## Proof of Concept

Lets assume :

1. John has `100e18` yeet tokens
2. Alice has `80e18` yeet tokens too
3. The `StakeV2` has 0 yeet token balance

* John and Alice both deposit `100e18` and `80e18` yeet tokens respectively into the contract by calling the `stake` function. Their new balances are `0` each, and the `stakeV2` yeet balance is `180e18` tokens ie `accumulatedDeptRewardsYeet()` returns `0` and `totalsupply` is `180e18`
* Currently the `rewardIndex` in `StakeV2` is 0
* As users yeet via the `Yeet` contract, a portion of their funds is transfered to the `StakeV2` contract as reward for the stakers (Won't focus on the amounts)
* The manager calls `executeRewardDistribution` function to distribute the rewards to stakers, the function zaps Into the zapper and then shares minted by the vault to the `StakeV2` contract.
* In the zapping process:

> yeet tokens gets returned by the zapper since swaps are not 100% efficient

* As users continue to yeet, the `executeRewardDistribution` is called by the manager to distribute the rewards and more yeet tokens continue to accumulate in the `StakeV2` contract and the `accumulatedDeptRewardsYeet` function returns `> 0` say maybe `500e18`since there are yeet tokens accumulated in the contract. Now the manager want to call the `executeRewardDistributionYeet` to distribute the accumulated yeet token rewards but,
* John unstakes their `100e18` yeet tokens by calling the `startUnstake` which subtracts the amount to be unstaked from the `totalsupply` and the funds are locked for the `VESTING_PERIOD` which is currently set to `10 days`.
* So now the `accumulatedDeptRewardsYeet` which initially was returning `500e18` now returns `600e18` since John's tokens were subtracted from the `totalSupply` but are still in the contract.
* The manager calls the `accumulatedDeptRewardsYeet` to see how many tokens have accumulated, and the function returns `600e18` .
* The manager proceeds to call `executeRewardDistributionYeet` to distribute the accumulated yeet tokens to the stakers with the `swap.input` amount as `600e18`. When the transaction finishes executing, the new yeet balance in the contract is `80e18`, the `totalSupply` is 80e18 ie Alices tokens . The `accumulatedDeptRewardsYeet` returns 0 since

```
stakingToken.balanceOf(address(this)) - totalSupply
```

is 0.

* John vesting time is over and heads over to `unstake` to finish the unstaking, however the transfer fails since the contract doesn't have john's funds. They were distributed as rewards to the users who have staked since the contract doesn't have a mechanism of accounting for the unstaked tokens that are under their vesting period. Therefore John loses their funds.
* This accounting mismatch will always cause users in their vesting period to lose funds when the manager calls `accumulatedDeptRewardsYeet` since they will always accounted as `rewards form the zapper`. Consider adding a mechanism to keep track of the total yeet tokens that are locked while awaiting their vesting period to elapse to avoid this issue
