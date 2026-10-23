# #41365 \[SC-Critical] Vested tokens are counted as accumulated revenue

**Submitted on Mar 14th 2025 at 11:42:08 UTC by @armormadeofwoe for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41365
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`accumulatedDeptRewardsYeet` incorrectly counts all vested staking tokens as revenue, allowing them to be swapped for vault shares and distributed as rewards to all stakers.

## Vulnerability Details

`accumulatedDeptRewardsYeet` considers the difference between the live `stakingToken.balanceOf(address(this))` and the user-deposited `totalSupply` as revenue.

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

Afterwards this amount can be swapped for additional rewards via `executeRewardDistributionYeet`

```solidity
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
        @> uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        @> require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");
```

The issue here lies within `startUnstake`, due to it subtracting the unstaked tokens from the `totalSupply` without transferring or accounting the `stakingToken` they should receive.

```solidity
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); 
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        @> totalSupply -= unStakeAmount;
        @> // MISSING TRANSFER OR INTERNAL ACCOUNTING
        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
    }
```

So what happened is:

1. User started and unstake which vests their token for 10 days
2. `totalSupply` went down, however no token transfers took place
3. Calling `accumulatedDeptRewardsYeet` will return value equal to the vested amount

Let's see a practical example.\
Assume there is no revenue, `balanceOf(address(StakingV2)) == totalSupply == 100 tokens`

1. Bob unstakes his 60 tokens by calling `startUnstake`
2. `totalSupply = 100 - 60 = 40`, 10 day vesting period starts for Bob
3. Admin calls `executeRewardDistributionYeet` with input amount == `accumulatedDeptRewardsYeet == 60`
4. 60 staking tokens are swapped through the Zapper for vault shares
5. `_handleVaultShares` handles reward distribution by updating the reward index.
6. 10 days pass
7. Bob calls `unstake`, however the method reverts since `balanceOf(address(StakingV2)) = 40` while `unlockedAmount = 60`

NB! This example is severely simplified for illustrative purposes. One might argue that the admin would not perform the call, knowing that there is no current revenue. However under normal working conditions, the protocol will have hundreds/thousands of actors and it is not possible to track which funds are part of the vestings and which are accumulated revenue.

## Impact Details

Protocol becomes insolvent since `balanceOf(address(StakingV2))` < owed amount to unstaking parties.

Last person to withdraw their funds will be unable to due to insufficient balance.

Rewards had been stolen and distributed to other Yeet participants.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L148-L150>

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L158-L160>

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L255>

## Proof of Concept

## Proof of Concept

Attaching step-by-step from main body

Let's see a practical example.\
Assume there is no revenue, `balanceOf(address(StakingV2)) == totalSupply == 100 tokens`

1. Bob unstakes his 60 tokens by calling `startUnstake`
2. `totalSupply = 100 - 60 = 40`, 10 day vesting period starts for Bob
3. Admin calls `executeRewardDistributionYeet` with input amount == `accumulatedDeptRewardsYeet == 60`
4. 60 staking tokens are swapped through the Zapper for vault shares
5. `_handleVaultShares` handles reward distribution by updating the reward index.
6. 10 days pass
7. Bob calls `unstake`, however the method reverts since `balanceOf(address(StakingV2)) = 40` while `unlockedAmount = 60`
