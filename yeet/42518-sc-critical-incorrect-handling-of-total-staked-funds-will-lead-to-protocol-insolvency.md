# #42518 \[SC-Critical] Incorrect handling of total staked funds will lead to protocol insolvency

**Submitted on Mar 24th 2025 at 13:23:43 UTC by @dobrevaleri for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42518
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `StakeV2::executeRewardDistributionYeet()` function incorrectly distributes pending withdrawals as rewards to remaining stakers, leading to protocol insolvency.

## Vulnerability Details

The root cause lies in how `totalSupply` is decremented when users initiate withdrawals, while those tokens remain in the contract during the vesting period.

```solidity
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
@>      totalSupply -= unStakeAmount;

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }
```

`$YEET` tokens as rewards are distributed as:

```solidity
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
@>      uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

        stakingToken.approve(address(zapper), accRevToken0);
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();  //KodiakVault - KodiakIsland - 
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        uint256 vaultSharesMinted;
        require(
            address(token0) == address(stakingToken) || address(token1) == address(stakingToken),
            "Neither token0 nor token1 match staking token"
        );

        if (address(token0) == address(stakingToken)) {
            (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }
```

The `accumulatedDeptRewardsYeet()` function calculates rewards as:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;  
}
```

When users call `startUnstake()`, their tokens are subtracted from `totalSupply` but remain in the contract until the vesting period ends. This causes `accumulatedDeptRewardsYeet()` to incorrectly count pending withdrawal tokens as distributable rewards.

## Impact

Incorrect handling of the `totalSupply` will lead to distribution of user's stake as rewards, which will lead to insolvency.

## Proof of Concept

## Proof of Concept

1. User A stakes 1000 YEET tokens
2. User A calls `startUnstake(1000)`
   * `totalSupply` decreases by 1000
   * Tokens remain in contract during 10 day vesting
3. During vesting period, manager calls `executeRewardDistributionYeet()`
   * `accumulatedDeptRewardsYeet()` returns 1000 (contract balance - totalSupply)
   * The 1000 tokens are distributed to other stakers as rewards
4. After vesting period, User A tries to withdraw but contract lacks funds
