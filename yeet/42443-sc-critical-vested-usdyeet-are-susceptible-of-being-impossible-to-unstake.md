# #42443 \[SC-Critical] Vested \`$YEET\` are susceptible of being impossible to unstake

**Submitted on Mar 23rd 2025 at 23:28:54 UTC by @greed for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42443
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Summary

Users have the ability to stake `$YEET` tokens in the `StakeV2.sol` contract in order to get rewards.

At any point, a user is allowed to unstake his tokens which will be submitted to a vesting period of 10 days. The vested amount of tokens will be claimable and effectively unstaked after this period has passed.

Contract managers can execute the distribution of `$YEET` rewards which will distribute the excess of tokens across all stakers.

This excess is wrongly calculated and includes the vested tokens users are trying to unstake.

### Vulnerability details

The number of tokens users have staked in the contract is accounted in the `totalSupply` variable. This number is increased everytime a user stakes and everytime a user starts the unstake process.

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L240><https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L255>

```solidity
function stake(uint256 amount) external {
    require(amount > 0, "Amount must be greater than 0");
    _updateRewards(msg.sender);

    stakingToken.transferFrom(msg.sender, address(this), amount);

    balanceOf[msg.sender] += amount;
@>  totalSupply += amount;
    emit Stake(msg.sender, amount);
}

function startUnstake(uint256 unStakeAmount) external {
    require(unStakeAmount > 0, "Amount must be greater than 0");
    require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
    _updateRewards(msg.sender);
    uint256 amount = balanceOf[msg.sender];
    require(amount >= unStakeAmount, "Insufficient balance");

    balanceOf[msg.sender] -= unStakeAmount;
@>  totalSupply -= unStakeAmount;

    uint256 start = block.timestamp;
    uint256 end = start + VESTING_PERIOD;
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
    stakedTimes[msg.sender]++;
    emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
}
```

The function involved when calculating the rewards to distribute subtracts the `$YEET` token balance of the contract with the `totalSupply`.

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L149>

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
@>  return stakingToken.balanceOf(address(this)) - totalSupply;
}

/// @notice The function used to distribute excess rewards to the vault.
function executeRewardDistributionYeet(
    IZapper.SingleTokenSwap calldata swap,
    IZapper.KodiakVaultStakingParams calldata stakingParams,
    IZapper.VaultDepositParams calldata vaultParams
) external onlyManager nonReentrant {
@>  uint256 accRevToken0 = accumulatedDeptRewardsYeet();
    require(accRevToken0 > 0, "No rewards to distribute");
    require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

    stakingToken.approve(address(zapper), accRevToken0);
    IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
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

Thus, the resulting amount of rewards fail to account for the vested amount of tokens waiting to be unstaked.

This means when the vesting period has ended, a user might not be able to claim his tokens because the contract does not have enough liquidity.

### Impact

Users might be prevented from withdrawing their tokens after unstaking them.

## Proof of Concept

### Proof of concept

1. Alice stakes 800 tokens: `totalSupply == 800`
2. The Yeet game is being played and 1000 tokens are accrued as rewards: `stakingToken.balanceOf(address(this)) == 1800`
3. Alice wants to unstake her 800 tokens so she starts the vesting period to claim them 10 days later: `totalSupply == 800 - 800 == 0`
4. Manager distributes the maximum amount of rewards possible: `stakingToken.balanceOf(address(this)) - totalSupply` meaning `1800 - 0 == 1800`
5. When Alice attempts to unstake her 800 tokens, the transaction will fail because the contract

### Recommendation

1. Maintain a storage variable `vestedTokens` that accounts for the vested amount of `$YEET` and increases everytime a user `startUnstake()` and decreases when a user `unstake()`.
2. Modify the `accumulatedDeptRewardsYeet()` to account for the vested tokens.

```diff
function accumulatedDeptRewardsYeet() public view returns (uint256) {
-   return stakingToken.balanceOf(address(this)) - totalSupply;
+   return stakingToken.balanceOf(address(this)) - totalSupply + vestedTokens;
}
```
