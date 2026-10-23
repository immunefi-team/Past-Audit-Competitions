# #41549 \[SC-Critical] users funds can get lost when the executeRewardDistributionYeet function invoked after users unstake

**Submitted on Mar 16th 2025 at 13:29:42 UTC by @zeroK for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41549
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

when users invokes the startUnstake function they request unstake, invoking this function decrease the balance of user and the totalSupply by the unstake amount, then after 10 days of invoking this function, users are able to transfer their yeet tokens back, however this is not always the case when the executeRewardDistributionYeet function get invoked after unstake function called by users, since the executeRewardDistributionYeet swaps the specified input amount + amount0Max which can be less or equal to the accumulated rewards, this function can transfer the unstake amount of the users by mistake by the manager since the totalSupply decreased but yet the users token not transferred to unstake addresses. more details in Vulnerability Details

## Vulnerability Details

if users want to unstake, they have two way to do so, rageQuit and lose around 50% of the staked amount(not very desired) and the second method is invoking the startUnstake function below:

```solidity
    function startUnstake(uint256 unStakeAmount) external {
        require(unStakeAmount > 0, "Amount must be greater than 0");
        require(stakedTimes[msg.sender] < STAKING_LIMIT, "Amount must be less then the STAKING_LIMIT constant"); // DOS protection https://github.com/Enigma-Dark/Yeet/issues/12
        _updateRewards(msg.sender);
        uint256 amount = balanceOf[msg.sender];
        require(amount >= unStakeAmount, "Insufficient balance");

        balanceOf[msg.sender] -= unStakeAmount;
        totalSupply -= unStakeAmount;

        uint256 start = block.timestamp;
        uint256 end = start + VESTING_PERIOD;
        vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
        stakedTimes[msg.sender]++;
        emit VestingStarted(msg.sender, unStakeAmount, vestings[msg.sender].length - 1);
    }


```

this function updates some important storage, one of them which is our core issue is totalSupply that get decreased by amount, but the amount will not get transferred until the vesting period passed(10 days), this will increase the returned amount by the accumulatedDeptRewardsYeet function:

```solidity
    function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }



```

then, the manager invokes call to the executeRewardDistributionYeet function by setting the `input + amount0Max == accumulatedDeptRewardsYeet`, (keep in mind current logic have issue with verifying and using the yeet token, the issue mentioned [here](https://drive.google.com/file/d/1JK91TgoE_t62RI7lu66V_N517P3AkA9c/view) so i explain this report keeping that issue in mind):

the executeRewardDistributionYeet flow including the swapping steps

```solidity
    function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap, //inputamount, outputQuote,  outputmin, executor, path
        IZapper.KodiakVaultStakingParams calldata stakingParams, // kodiak vault, amount 0/1 max/min, shares min, receiver
        IZapper.VaultDepositParams calldata vaultParams //vault, receiver, minshares 
    ) external onlyManager nonReentrant {
        uint256 accRevToken0 = accumulatedDeptRewardsYeet(); //@audit amount after users unstake
        require(accRevToken0 > 0, "No rewards to distribute");
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

        stakingToken.approve(address(zapper), accRevToken0); //we approve zapper the accRevToken0 but later we use the inputAmount
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0(); //e.g yeet or bera
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1(); //e.g bera or yeet

        uint256 vaultSharesMinted;
        require(address(token0) == address(stakingToken) || address(token1) == address(stakingToken),"Neither token0 nor token1 match staking token");
       //one of them should be yeet
        if (address(token0) == address(stakingToken)) {
            (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams); //@audit change the yeet to shares which used for reward calc/claim later
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }


//zapper.sol zapToken0In 

    function zapInToken0(
        SingleTokenSwap calldata swapData, // inputAmount, outputQuote, min output, executor, receiver
        KodiakVaultStakingParams calldata stakingParams,
        VaultDepositParams calldata vaultParams // vault, receiver, minShare
    ) public nonReentrant onlyWhitelistedKodiakVaults(stakingParams.kodiakVault) returns (uint256, uint256) {
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        token0.safeTransferFrom(_msgSender(), address(this), stakingParams.amount0Max + swapData.inputAmount);//@audit transfer the accumulated reward to zapper
        uint256 token1Debt = _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this)); // @audit -> only token1 can be used extra in the vault staking.
        return _yeetIn(token0, token1, stakingParams.amount0Max, token1Debt, stakingParams, vaultParams);
    }


// swap the yeet to token1

    function _verifyTokenAndSwap(
        SingleTokenSwap calldata swapData,
        address inputToken,
        address outputToken,
        address receiver
    ) internal returns (uint256 amountOut) {
        if (swapData.inputAmount == 0) {
            return 0;
        }
        // check whitelist for input token nd output token
        require(whitelistedTokens[inputToken], "Zapper: input token not supported");
        require(whitelistedTokens[outputToken], "Zapper: output token not supported");

        IOBRouter.swapTokenInfo memory swapTokenInfo = IOBRouter.swapTokenInfo({
            inputToken: inputToken,
            inputAmount: swapData.inputAmount,
            outputToken: outputToken,
            outputQuote: swapData.outputQuote,
            outputMin: swapData.outputMin,
            outputReceiver: receiver
        });
        return _approveRouterAndSwap(swapTokenInfo, swapData.path, swapData.executor);
    }


// yeetIn will deposit the LP tokens into the vault to get share token which later used by users to claim rewards:

    function _yeetIn(
        IERC20 token0,
        IERC20 token1,
        uint256 token0Debt,
        uint256 token1Debt,
        KodiakVaultStakingParams calldata stakingParams,
        VaultDepositParams calldata vaultParams
    ) internal returns (uint256, uint256) {
        (uint256 amount0Used, uint256 amount1Used, uint256 kodiakVaultTokensMinted) =
                    _approveAndAddLiquidityToKodiakVault(stakingParams.kodiakVault, token0, token1, stakingParams);
        // @audit -> reverts if negative. hence user cannot use more than what he has.
        token0Debt -= amount0Used;
        token1Debt -= amount1Used;
        uint256 vaultSharesMinted;
        // if recevier is zapper then deposit into vault, (if this contract received minted LP)
        if (stakingParams.receiver == address(this) && kodiakVaultTokensMinted > 0) {
            vaultSharesMinted = _depositIntoVault(vaultParams, kodiakVaultTokensMinted);
        }
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
        return (kodiakVaultTokensMinted, vaultSharesMinted);
    }


```

the issue in above flow is that the manger can by mistake or in purpose invoke the `executeRewardDistributionYeet` function and setting the swap.inputAmount equal to half of the accumulated amount and the amount0Max equal to the another half of the accumulated amount which includes users unstaked request balance, to make this more clear, expect the scenario below:

* current `balanceOf(address(stakeV2)) == $10k worth of yeet` and the totalSupply is $9k worth of yeet.
* Bob decided to unstake and take his yeet token back($3k worth of yeet) by invoking startUnstake, this will decrease the totalSupply to $6k, but, the balanceOf still at $10k since the tokens not transferred yet and bob should wait 10 days to transfer tokens
* the manager by mistake invoke calls to the executeRewardDistributionYeet function and setting the `swap.input == 2k and amount0max == 2k` so that it can invoke the executeRewardDistributionYeet correctly(due to approve issue in this link <https://drive.google.com/file/d/1JK91TgoE\\_t62RI7lu66V\\_N517P3AkA9c/view)\\>
  this way the executeRewardDistributionYeet will swap the yeet token included bob yeet that added to unstake and change it to share token that benefits other stakers expect bob, this is because `balanceOf` bob became zero and he can accumulate rewards anymore:

```solidity

    function claimRewardsInNative(
        uint256 amountToWithdraw,
        IZapper.SingleTokenSwap calldata swapData0,
        IZapper.SingleTokenSwap calldata swapData1,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams, //kodiak vault, amount0/1 min/max, receiver
        IZapper.VaultRedeemParams calldata redeemParams // vault, receiver, shares, minAssets
    ) external nonReentrant {
        _updateRewards(msg.sender);

        IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

        IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
        uint256 receivedAmount =
                            zapper.zapOutNative(msg.sender, swapData0, swapData1, unstakeParams, updatedRedeemParams);

        emit Claimed(msg.sender, receivedAmount);
    }


    function _calculateRewards(address account) private view returns (uint256) {
        uint256 shares = balanceOf[account]; //@audit zero 
        return (shares * (rewardIndex - rewardIndexOf[account])) / MULTIPLIER;
    }

    /// @notice The function used to update the rewards of an account
    /// @param account The account to update the rewards for
    function _updateRewards(address account) private {
        earned[account] += _calculateRewards(account); //audit zero
        rewardIndexOf[account] = rewardIndex;
    }


```

due to all the flow, there is nothing prevent the manager to invoke call and get shares amount equal to the total of accumulated rewards debt.

```solidity
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute"); //@audit set to half of accRevToken0 since we approve zapper with accRevToken0 which allows setting the amount0/1Max to another half of the accRevToken0

        stakingToken.approve(address(zapper), accRevToken0); 

```

this will lead to direct lose of tokens funds that users unstake by manager, its not possible for manager to keep track and calculate the amount without including the unstake change that happened to totalSupply by unstakers.

## Impact Details

direct lose of funds due to incorrect calculation in the accumulated reward function

## recommend

add a map that track the unstake amount and add its value to the calculation in in the accumulatedDeptRewardsYeet or do not decrease totalSupply until vesting period ends.

## Proof of Concept

## Proof of Concept

the steps for running valid poc:

* first thing stake $10k worth of yeet by different addresses(alice 5k, bob 3k, eve 2k) so that the balanceOf and totalSupply equal to 10k.
* eve rageQuit or unstake first so that the totalSupply became 8k and balanceOf became 8k.
* the manager or owner or any address transfer 2k worh of yeet directly to the vault, this way the accumulatedRewardDebt function return 10k - 8k = 2k worth of yeet.
* bob decide to unstake by invoking the startUnstake, he unstake all his balance which is 3k, the totalSupply became 5k but the yeet still in the stakeV2 contract(not transferred yet)
* manager checks the accumulatedRewardDebt and see that the value to use to get share is 5k worht of yeet and decide to invoke call to the executeRewardDistributionYeet funciton with `swap.input = 2.5k and the amount0max = 2.5k` which later the input amount swapped to token0/token1 and it get used alongside the amount0Max/amount1Max to add liquidity and then adding this liquidity to get shares which later used to calculate rewards for users or stakers.
* bob now can not transfer his tokens back, even if he does these tokens are alice tokens in the stake contract and the lose goes to alice in this case
