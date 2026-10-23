# #42725 \[SC-Critical] startUnstake() Reduces Total Supply, but StakingToken Balance in contract Remains Constant, Leading to Inflated accumulatedDeptRewardsYeet()

**Submitted on Mar 25th 2025 at 13:46:17 UTC by @libro9595 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42725
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

* When a user starts `unstaking`, the `totalSupply` variable is reduced, but the actual balance of the staking token remains unchanged.
* As a result, the calculation of `accumulatedDeptRewardsYeet()` becomes inflated compared to the actual amount.
* This can lead to an significantly more rewards distribution in the `executeRewardDistributionYeet()` function .

## Impact Details

* Assume an attacker stakes k amount of tokens in the protocol.
* The protocol’s total supply is X, and the totalSupply of stake token in the vault (denoted as Y) are greater than k (X > k and Y > k).
* When the attacker calls the `startUnstake()` function, their staked amount is deducted from the total supply. As a result, the total supply becomes X - k.
* However, the actual balance of staking tokens in the contract remains Y, as the tokens are not physically withdrawn during the unstaking process.
* Ideally, the balance of rewards should reflect Y - k to account for the pending unstake. However, since the balance remains Y, it artificially inflates the calculated rewards.
* The function `accumulatedDeptRewardsYeet()` calculates rewards as:
* Accumulated Rewards = Y - (X - k)
* This results in an inflated reward amount.
* When the manager calls `executeRewardDistributionYeet()`, the protocol uses this incorrect `accumulatedDeptRewardsYeet()` value to distribute rewards.
* As a result, the attacker can claim significantly more rewards than they are entitled to, causing a substantial loss of funds from the protocol.
* Additionally, because the attacker has initiated the unstake without claiming immediately, they can exploit the inflated reward calculation for an extended period, draining further funds.

## Recommendation:

* Add a new variable `uint256 public pendingUnstake` to track the tokens in the unstaking process. Increment `pendingUnstake` in the startUnstake() function and decrement it when the `unstake` is completed.\
  Update the `accumulatedDeptRewardsYeet()` function to subtract pendingUnstake from the calculation, ensuring rewards are

## Proof of Concept

## Proof of Concept

* totoal suppy = x
* total balance = y

### **Step 1: Staking Tokens**

* The attacker deposits **k tokens** using the `stake()` function.
* After this action:
  * Total supply of staked tokens becomes **X + k**.
  * Contract’s actual token balance becomes **Y + k**.
* The attacker begins earning staking rewards.

### **Step 2: Earning Rewards**

* The attacker waits to accumulate rewards.
* The manager periodically calls `executeRewardDistribution()` to distribute rewards, increasing the `rewardIndex`.

### **Step 3: Initiating Unstake Without Actual Token Transfer**

* The attacker calls `startUnstake(k)` to initiate the unstaking process before the executeRewardDistribution() .
* The `totalSupply` is reduced by **k tokens** (“**totalSupply = X**”).
* However, the actual token balance in the contract remains **Y + k**.
* The staked tokens remain in the contract during the vesting period.

### **Step 4: Executing Reward Distribution**

* The manager calls `executeRewardDistributionYeet()`.
* The function uses `accumulatedDeptRewardsYeet()` to calculate rewards based on the following formula:

  ```
  Accumulated Rewards = Y + k - X
  ```
* Since `totalSupply` (**X**) is now artificially reduced, the function falsely assumes excess rewards.
* The inflated rewards are distributed to stakers, including the attacker.

### **Step 5: Claiming Excessive Rewards**

* The attacker calls `claimRewardsInToken()` to withdraw rewards.
* Due to the inflated reward calculation, the attacker claims a significantly higher amount of rewards than they are entitled to.
* This results in a substantial loss of funds from the protocol.
