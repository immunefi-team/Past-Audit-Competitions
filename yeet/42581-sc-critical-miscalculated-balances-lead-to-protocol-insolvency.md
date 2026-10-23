# #42581 \[SC-Critical] Miscalculated Balances Lead to Protocol Insolvency

**Submitted on Mar 24th 2025 at 19:54:56 UTC by @Bani70 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42581
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Users may become unable to claim their unstaked tokens due to a flaw in the StakeV2 contract's logic, which miscalculates excess rewards by including unstaked-but-unclaimed tokens in its calculations. The function `accumulatedDeptRewardsYeet()` incorrectly determines available rewards based on the difference between the contract’s token balance and the `totalSupply`, allowing these funds to be transferred out before users can withdraw them. This incorrect distribution depletes the contract’s reserves, leading to irreversible loss of user funds causing protocol insolvency.

## Vulnerability Details

How the system should work:

1. By calling `stake()`, User can stake an amount of tokens, transferring the specified amount inside the contract.

```solidity
stakingToken.transferFrom(msg.sender, address(this), amount);
```

Then this stake is accounted by incrementing the `balanceOf[User]` and `totalSupply` with the amount the user specified.

```solidity
balanceOf[msg.sender] += amount;
totalSupply += amount;    
```

2. Once users want to unstake they call `startUnstake()` specifying an amount they want to unstake. This function first updates rewards, but we will focus on what it does next.\
   Similar to `stake()` it accounts the unstake by updating the same - mapping `balanceOf[User]` and global variable `totalSupply`.

```solidity
balanceOf[msg.sender] -= unStakeAmount;
totalSupply -= unStakeAmount;
```

3. Ideally users then should call `unstake()` or `rageQuit()` and can claim the full amount of unstaked tokens or a portion of it depending on certain conditions.

Both functions call `_unstake` which transfers the unlocked tokens to the user and burns the locked amount (if any).

```solidity
stakingToken.transfer(msg.sender, unlockedAmount);
stakingToken.transfer(address(0x000000dead), lockedAmount);
```

However there is a problem that leads to insolvency and affects every user that calls `startUnstake`. The problem lies in the `accumulatedDeptRewardsYeet()` that calculates the excess rewards. it incorrectly uses `totalSupply` for this calculation and calculated the difference between the balance of staking tokens in the contract and the amount accounted in `totalSupply`, which also included the tokens that have been started unstaking but have not yet been claimed.

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
        return stakingToken.balanceOf(address(this)) - totalSupply;
    }
```

The result is used in the `executeRewardDistributionYeet()` which transfers the tokens along with any amount that users have started unstaking but haven't claimed yet.

```solidity
 function executeRewardDistributionYeet(
        IZapper.SingleTokenSwap calldata swap,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultParams
    ) external onlyManager nonReentrant {
        uint256 accRevToken0 = accumulatedDeptRewardsYeet();                                        //Incorrectly Calculates Rewards
        require(accRevToken0 > 0, "No rewards to distribute");                  
        require(swap.inputAmount <= accRevToken0, "Insufficient rewards to distribute");

        stakingToken.approve(address(zapper), accRevToken0);                                        //Approves The `Zapper` To Send The Reward Tokens 
        IERC20 token0 = IKodiakVaultV1(stakingParams.kodiakVault).token0();   
        IERC20 token1 = IKodiakVaultV1(stakingParams.kodiakVault).token1();

        uint256 vaultSharesMinted;
        require(
            address(token0) == address(stakingToken) || address(token1) == address(stakingToken),
            "Neither token0 nor token1 match staking token"
        );

        if (address(token0) == address(stakingToken)) {                                             //Swap Staking Tokens for Vault Shares
            (, vaultSharesMinted) = zapper.zapInToken0(swap, stakingParams, vaultParams);
        } else {
            (, vaultSharesMinted) = zapper.zapInToken1(swap, stakingParams, vaultParams);
        }

        _handleVaultShares(vaultSharesMinted);
        emit RewardsDistributedToken0(accRevToken0, rewardIndex);
    }
```

## Impact Details

The vulnerability allows the contract to miscalculate excess rewards, leading to an irreversible loss of user funds and potential protocol insolvency. Specifically, when a user initiates an unstake request, their balance is deducted from the `totalSupply` before they actually claim their tokens. This discrepancy causes the `accumulatedDeptRewardsYeet` function to incorrectly interpret the unstaked-but-unclaimed tokens as excess rewards, which are then distributed via `executeRewardDistributionYeet`. As a result, these funds are permanently removed from the contract before the user can withdraw them.

When the user later attempts to claim their unstaked tokens, the contract no longer holds a sufficient balance, causing a revert and preventing them from recovering their assets.

## References

* **Function `stake`**
  * Users call it to stake, transferring tokens inside the `StakeV2` contract

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L233-L242>

* **Function `startUnstake`**
  * Users call it to start a vesting period to unstake tokens they have been staking.

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L247-L262>

* **Functions `unstake`, `rageQuit` and `_unstake`**\
  (quite compact and can fit in a single snippet)
  * Users can call `unstake`or `rageQuit` depending if they want to wait for the whole vesting period to end to claim the full unlocked amount or if they just want a portion of it and don't want to wait for the whole vesting period. Both functions then call `_unstake`, that sends unlocked amount of tokens to user and burns the locked amount if any.

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L266-L295>

* **Functions `accumulatedDeptRewardsYeet` and `executeRewardDistributionYeet`**
  * `accumulatedDeptRewardsYeet` is a view function that calculates rewards returned by the zapper.
  * `executeRewardDistributionYeet` is a function a `manager` can call to distribute the rewards calculated from the above function.

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L148-L201>

## Proof of Concept

## Proof of Concept

1. User stakes 1000 tokens => calls `stake()` (staking 1000 tokens)\
   Before this function call, the balances were the following:`stakingToken.balanceOf(address(this))` = 100`totalSupply` = 100`balanceOf[User]` = 0;

```solidity
stakingToken.transferFrom(msg.sender, address(this), 1000);

balanceOf[msg.sender] += 1000;
totalSupply += 1000;    
```

Now after this call the balances look like this:`stakingToken.balanceOf(address(this))` = 1100`totalSupply` = 1100`balanceOf[User]` = 1000;

2. User wants to unstake the tokens => calls `startUnstake()` (willing to unstake all 1000 tokens they previously staked)

```solidity
balanceOf[msg.sender] -= 1000;
totalSupply -= 1000;    
```

Now after this call the balances look like this:`stakingToken.balanceOf(address(this))` = 1100`totalSupply` = 100`balanceOf[User]` = 0;

You can see the balances are decremented before the user can claim their tokens.

3. Admin/Manager calls `executeRewardDistributionYeet`, which uses `accumulatedDeptRewardsYeet` to wrongly calculate the difference between the tokens in the contract and `totalSupply`. In this case the difference is exactly the amount our User started unstaking - 1000. The function then approves the zapper the amount of 1000 and the amount is transferred out of the contract.
4. The vesting period for our User ends and he proceeds call `unstake()` to unstake their 1000 tokens. =>`unstake()` calls `_unstake()` which reverts on this line:

```solidity
stakingToken.transfer(msg.sender, unlockedAmount); // unlocked amount is 1000 
```

The function reverts because the function attempts to send the unlocked amount of our user (1000 tokens), but the contract only holds 100 tokens, leaving the protocol insolvent preventing the user from claiming their tokens.
