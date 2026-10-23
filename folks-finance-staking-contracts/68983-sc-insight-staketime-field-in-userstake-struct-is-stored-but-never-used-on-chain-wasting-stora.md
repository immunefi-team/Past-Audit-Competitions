# 68983 sc insight staketime field in userstake struct is stored but never used on chain wasting storage on every stake

**Submitted on Mar 12th 2026 at 08:12:42 UTC by @hcrlen for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68983
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

### Brief/Intro

`UserStake` struct stores `stakeTime` on every deposit but this field is never referenced in any on-chain logic, resulting in unnecessary storage consumption on every stake operation.

### Vulnerability Details

In `_stake()`, `stakeTime` is written to storage:

```solidity
UserStake({
    ...
    stakeTime: uint64(block.timestamp), // never read again
    ...
})
```

A review of the entire contract shows that `stakeTime` is never read by any function, including:

* withdrawal logic
* reward calculations
* migration logic
* validation checks

As a result, the value is written to storage but never used.

## Impact Details

Writing to a new storage slot requires an SSTORE operation (≈20,000 gas for a cold write). Since `stakeTime` is not used anywhere in the contract, each stake transaction incurs this extra gas cost without providing any functional benefit.

Over time, this increases gas costs for all users interacting with the staking contract.

## Recommendations

Remove `stakeTime` from `UserStake`. The `Staked` event's block timestamp provides equivalent off-chain tracking.

## Proof of Concept

{% stepper %}
{% step %}

#### 1. Call `stake()` or `stakeWithPermit()`.

{% endstep %}

{% step %}

#### 2. `_stake()` stores `stakeTime` in the `UserStake` struct.

```solidity
userStakes[msg.sender].push(
    UserStake({
        amount: amount,
        reward: reward,
        claimedAmount: 0,
        claimedReward: 0,
        aprBps: stakingPeriod.aprBps,
        stakeTime: uint64(block.timestamp), // <--
        unlockTime: uint64(block.timestamp) +  stakingPeriod.stakingDurationSeconds,
        unlockDuration: stakingPeriod.unlockDurationSeconds
    })
);
```

{% endstep %}

{% step %}

#### 3. The variable is never referenced or used again by any on-chain function.

{% endstep %}
{% endstepper %}
