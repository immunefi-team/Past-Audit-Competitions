# #41487 \[SC-Critical] Updates totalSupply before transferring the tokens which causes calculating more reward tokens

**Submitted on Mar 15th 2025 at 20:18:10 UTC by @Yaneca\_b for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41487
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `totalSupply` variable is reduced immediately during `startUnstake()` — before tokens are fully unlocked from vesting. This leads to inflated reward calculations, allowing malicious users to initiate vesting, claim an unfairly high proportion of rewards, and exit without penalty. If exploited in production, the staking pool could be drained, leaving honest stakers with little to no rewards.

## Vulnerability Details

The core issue lies within the `startUnstake()` function:

```solidity
balanceOf[msg.sender] -= unStakeAmount;
totalSupply -= unStakeAmount;
```

Here, `totalSupply` is decremented immediately when the unstake process starts, even though the tokens are still technically held within the contract under vesting. This results in a misleadingly lower `totalSupply`, which artificially increases the share of rewards each remaining staker receives — including the attacker, who still has their full reward eligibility.

## Impact Details

When the `totalSupply` is reduced but the corresponding rewards are not properly transferred, the `accumulatedDeptRewardsYeet()` function returns an inflated rewards amount. This results in an over-distribution of staked funds as rewards, potentially draining the contract’s balance. Over time, this imbalance can lead to insolvency, leaving the protocol unable to meet user withdrawals or honor legitimate reward distributions.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L149\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L255>

## Proof of Concept

## Proof of Concept

```
// there is a staked amount and rewards in the stakingContract contract
function testAccumulatedDeptRewardsYeet() public {
    // Step 1: Save the current accumulatedDeptRewardsYeet value to memory before unstaking
    uint256 initialDeptRewards = stakingContract.accumulatedDeptRewardsYeet();

    // Step 2: Call startUnstake() to trigger the unstake process
    uint256 amount = 1 ether;
    stakingContract.startUnstake(amount);

    // Step 3: Capture the new accumulatedDeptRewardsYeet value after unstaking
    uint256 newDeptRewards = stakingContract.accumulatedDeptRewardsYeet();

    // Step 4: Assert that the rewards have changed (not equal before and after)
    // The assumption is that the unstaking operation will affect the rewards
    assert(newDeptRewards == initialDeptRewards + amount);
}
```
