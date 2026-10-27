# #41974 \[SC-Critical] Reducing \`totalSupply\` in \`startUnstake\` leads to protocol insolvency

**Submitted on Mar 19th 2025 at 18:04:28 UTC by @Oxl33 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41974
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

**Description:**

When a staker calls `StakeV2::startUnstake` function, `totalSupply` is decreased by `unStakeAmount`:

```solidity
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant");
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

The issue arises when manager calls `StakeV2::executeRewardDistributionYeet` function, which calls `accumulatedDeptRewardsYeet`:

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

As you can see, this function returns the difference between `StakeV2`'s YEET balance and `totalSupply`. In the case where there is at least a single user in the process of unstaking (waiting for vesting to complete), the returned value of this function will be bigger than it should be, due to `totalSupply` being decreased. This will lead to more YEET tokens being approved to `Zapper` and more tokens transferred out (users that started unstaking process did NOT receive their YEET tokens yet).

This issue is only possible if the manager's inputted `stakingParams.amount0Max` and `swapData.inputAmount` parameters do not account for the wrong return value of `accumulatedDeptRewardsYeet` function. I think manager will not account for this when calculating the input parameters, because the only logical way to determine the amount of YEET to transfer out is to use `accumulatedDeptRewardsYeet` function. Because why would the manager use a different way to decide the amount of YEET to transfer, if this is the way used to decide how many tokens to approve? Approving more than actually gets transferred is also a dangerous practice and I strongly believe nobody would do this on purpose.

```solidity
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
@>      uint256 accRevToken0 = accumulatedDeptRewardsYeet();
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

@>      stakingToken.approve(address(zapper), accRevToken0);
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        uint256 vaultSharesMinted;
        require(
            address(token0) == address(stakingToken) || address(token1) == address(stakingToken),
            "Neither token0 nor token1 match staking token"
        );

        if (address(token0) == address(stakingToken)) {
@>          (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }
```

Tokens being transferred here:

```solidity
    function zapInToken0(
        SingleTokenSwap calldata swapData,
        KodiakVaultStakingParams calldata stakingParams,
        VaultDepositParams calldata vaultParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(stakingParams.kodiakVault) returns (uint256, uint256) {
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();
@>      token0.safeTransferFrom(_msgSender(), address(this), stakingParams.amount0Max + swapData.inputAmount);
        uint256 token1Debt = _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));
        return _yeetIn(token0, token1, stakingParams.amount0Max, token1Debt, stakingParams, vaultParams);
    }
```

In the above code snippet you can see that the tokens which are in unstaking process (alongside the actual reward amount) will be sent to `Zapper` and ultimately they will end up being deposited into Beradrome farm and all other stakers will claim them as rewards as time goes by.

There is no way to emergency-withdraw these funds, because all the shares of `MoneyBrinter` vault are held by `StakeV2` contract, and can only be accessed by the stakers, when they claim rewards.

When the user, whose funds got sent to `Zapper`, calls `StakeV2::_unstake` (let's assume after the full vesting period), the outcome will be one of these:

1. The user will receive YEET that other users staked and, if everyone wants to unstake, the last staker's transaction will revert, due to `StakeV2` being out-of-funds
2. If the user is the last staker left, the user's transaction will revert, due to `StakeV2` being out-of-funds
3. If the user had staked a huge amount of YEET (more than all other stakers combined), the user's transaction will revert, due to `StakeV2` being out-of-funds

If this situation were to happen, you would have no good way to fix it. Only thing you could do is to send the amount of YEET that is missing (out of your own pocket) to `StakeV2` contract, and let the user(s) unstake it.

**Impact:**

Protocol insolvency, due to mismanagement of user funds.

**Recommended Mitigation:**

Consider not decreasing `totalSupply` in `startUnstake` function, and instead decrease it in `_unstake` function. This way `accumulatedDeptRewardsYeet` function will return the correct value and the issue will be solved.

Example fix:

```diff
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
-       totalSupply -= unStakeAmount;

...
...
...

    function _unstake(uint256 index) private {
        Vesting memory vesting = vestings[msg.sender][index];

        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
        require(unlockedAmount != 0, "No unlocked amount");
+       totalSupply -= vesting.amount;

        stakingToken.transfer(msg.sender, unlockedAmount);
        stakingToken.transfer(address(0x000000dead), lockedAmount);
        _remove(msg.sender, index);
```

## Proof of Concept

**Proof of Concept:**

Consider this scenario:

1. Alice stakes `10_000_000e18` YEET tokens into `StakeV2`, because she likes the project and wants to receive a big portion of the rewards
2. Some time passes, Alice participates in the project, many other users also stake their YEET tokens
3. Alice calls `StakeV2::startUnstake` function, inputting `unStakeAmount` equal to her initial staked amount
4. Few days (up to `VESTING_PERIOD`) pass and manager decides to call `executeRewardDistributionYeet`, because there are some (e.g. `100_000e18`) YEET tokens ready to be distributed as rewards
5. `accumulatedDeptRewardsYeet` function should return `100_000e18`, but instead returns `10_100_000e18`, so the approved amount and the manager's input are much bigger (for the reasons I described above)
6. `10_100_000e18` YEET tokens get sent to `Zapper` and end up being deposited into Beradrome farm
7. Alice's vesting period ends and she calls `StakeV2::_unstake`, but her transaction reverts, because her staked amount was more than the remaining YEET balance of `StakeV2` contract, so the contract is out-of-funds
8. Alice gets left with nothing and her `10_000_000e18` YEET tokens get claimed as rewards by the remaining stakers
