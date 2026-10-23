# 31575 - \[SC - Medium] depositIntoRewardPool and withdrawFromRewardPo...

Submitted on May 21st 2024 at 14:08:32 UTC by @Kenzo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31575

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Theft of unclaimed yield

## Description

## Title

`depositIntoRewardPool()` and `withdrawFromRewardPool()` in `RewardPoolManager` are missing slippage control mechanism

## Vulnerability Details

The `RewardPoolManger` is meant to be compatible with ERC4626. The `depositIntoRewardPool()` and `withdrawFromRewardPool()` functions are used to deposit and withdraw funds with Aura Pools and the shares and tokens are minted but as it is pool deposit, there is always fluctuations in the ratio deposit to mint or burn to withdraw. The the issue is these two functions don't implement any slippage mechanism to avoid such ratio drop. At worst case, the MEV attacks such as frontrunning the transaction can make loss of shares while depositing and loss of tokens while withdrawing. `RewardPoolHandler::depositIntoRewardPool` :

````solidity
    function depositIntoRewardPool(uint256 _amount) external returns (bool) {
        require(msg.sender == veALCX, "must be veALCX");

        IERC20(poolToken).approve(rewardPool, _amount);
    
@>      IRewardPool4626(rewardPool).deposit(_amount, address(this));
        return true;
    }
```umm
`RewardPoolHandler::withdrawFromRewardPool` :

```solidity
    function withdrawFromRewardPool(uint256 _amount) external returns (bool) {
        require(msg.sender == veALCX, "must be veALCX");

@>      IRewardPool4626(rewardPool).withdraw(_amount, veALCX, address(this));
        return true;
    }

````

## Impact Details

Users will loss tokens or shares due to not handling the slippage case. At worst, MEV may make user loss a huge amount of funds.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol?utm\\_source=immunefi#L84>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol?utm\\_source=immunefi#L93>

## Recommendation

Implement proper slippage control mechanism in these two functions to avoid the mentioned issue.

## Proof of Concept

Attack is straightforward and POC for Mev attacks are hard to simulate.
