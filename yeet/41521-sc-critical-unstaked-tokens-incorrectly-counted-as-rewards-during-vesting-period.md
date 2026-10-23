# #41521 \[SC-Critical] Unstaked tokens incorrectly counted as rewards during vesting period

**Submitted on Mar 16th 2025 at 07:55:16 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41521
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

**An accounting error in `StakeV2` causes unstaked tokens in vesting to be incorrectly counted as rewards**. This could lead to **unintended loss of funds for stakers** and **potential insolvency for the protocol**.

## Vulnerability Details

The `accumulatedDeptRewardsYeet()` function **does not account for unstaked tokens during the vesting period**:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
@>  return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

Since `totalSupply` is reduced immediately for the `unstakeAmount` at `startUnstake()`, while the unstaked amount remains in the contract during vesting, **`accumulatedDeptRewardsYeet()` mistakenly includes this unstaked amount as part of the rewards**. Consequently, **when `executeRewardDistributionYeet()` is called, these mistakenly counted rewards are distributed to stakers, leading to an unintended loss of funds**.

```solidity
function startUnstake(uint256 unStakeAmount) external {
    --- SNIPPED ---

    balanceOf[msg.sender] -= unStakeAmount;
@>  totalSupply -= unStakeAmount;

    uint256 start = block.timestamp;
    uint256 end = start + VESTING_PERIOD;
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
    stakedTimes[msg.sender]++;  
    emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
}
```

## Impact Details

Consider the following scenario:\
0\. Users A and B each stake 100e18 YEET\
\- Contract State:\
\- `YEET.balance(StakeV2)`: 200e18\
\- `totalSupply`: 200e18\
\- Real rewards: 0

1. B starts unstaking 100e18 YEET
   * Contract State:
     * `YEET.balance(StakeV2)`: 200e18 (unchanged)
     * `totalSupply`: 100e18 (reduced by B's unstake)
     * Tokens in vesting: 100e18 (B's unstaking amount)
2. Manager Distributes "Rewards":
   * `accumulatedDeptRewardsYeet()` returns: 200e18 - 100e18 = 100e18
   * These rewards (actually B's vesting tokens) are distributed
   * Contract State:
     * `YEET.balance(StakeV2)`: 100e18
     * `totalSupply`: 100e18
     * **Tokens distributed as "rewards": 100e18**
3. B finalizes unstake at vesting period ends and withdraws their 100e18 YEET
   * Final Contract State:
     * `YEET.balance(StakeV2)`: 0
     * `totalSupply`: 100e18 (A's stake)
     * **Result: A's stake becomes unbacked by tokens**

This **scenario demonstrates the following impacts**:

1. **User A suffers a complete loss of their 100e18 YEET stake** as they cannot unstake due to insufficient contract balance
2. **The protocol becomes technically insolvent** as it owes User A 100e18 YEET but has 0 balance
3. **The lack of vesting token tracking allows the manager to unknowingly distribute vesting tokens as rewards through `accumulatedDeptRewardsYeet()`, creating a HIGH likelihood**.

### The severity assessment

This issue qualifies as `Direct theft of any user funds` because:

1. Stakers lose their entire principal investment
2. The loss occurs while funds are at-rest in the staking contract
3. The vulnerability allows one user (B) to profit at the expense of another (A)

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L247-L262\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L148-L150>

## Proof of Concept

## Proof of Concept

1. Users A and B each stake 100e18 YEET
   * Contract State:
     * `YEET.balance(StakeV2)`: 200e18
     * `totalSupply`: 200e18
     * Real rewards: 0
2. B starts unstaking 100e18 YEET
   * Contract State:
     * `YEET.balance(StakeV2)`: 200e18 (unchanged)
     * `totalSupply`: 100e18 (reduced by B's unstake)
     * Tokens in vesting: 100e18 (B's unstaking amount)
3. Manager Distributes "Rewards":
   * `accumulatedDeptRewardsYeet()` returns: 200e18 - 100e18 = 100e18
   * These rewards (actually B's vesting tokens) are distributed
   * Contract State:
     * `YEET.balance(StakeV2)`: 100e18
     * `totalSupply`: 100e18
     * **Tokens distributed as "rewards": 100e18**
4. B finalizes unstake at vesting period ends and withdraws their 100e18 YEET
   * Final Contract State:
     * `YEET.balance(StakeV2)`: 0
     * `totalSupply`: 100e18 (A's stake)
     * **Result: A's stake becomes unbacked by tokens**
