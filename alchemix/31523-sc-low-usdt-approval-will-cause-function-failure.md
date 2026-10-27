# 31523 - \[SC - Low] USDT Approval will cause function failure

Submitted on May 21st 2024 at 01:03:13 UTC by @SAAJ for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31523

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardPoolManager.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

USDT approve requires resetting before changing

## Vulnerability Details

There are 2 instances where the `IERC20.approve()` function is called only once without setting the allowance to zero. USDT require first reducing the address' allowance to zero by calling `approve(_spender, 0)` before.

```
File: RewardPoolManager.sol
function depositIntoRewardPool(uint256 _amount) external returns (bool) {
        require(msg.sender == veALCX, "must be veALCX");

        IERC20(poolToken).approve(rewardPool, _amount);
        IRewardPool4626(rewardPool).deposit(_amount, address(this));
        return true;
    }
```

Same issue exist in `claim` of `RevenueHandler` contract

```
File: RevenueHandler.sol
    /// @inheritdoc IRevenueHandler
    function claim(
        uint256 tokenId,
        address token,
        address alchemist,
        uint256 amount,
        address recipient
    ) external override {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, tokenId), "Not approved or owner");

        uint256 amountBurned = 0;

        uint256 amountClaimable = _claimable(tokenId, token);
        require(amount <= amountClaimable, "Not enough claimable");
        require(amount > 0, "Amount must be greater than 0");
        require(amount <= IERC20(token).balanceOf(address(this)), "Not enough revenue to claim");

        userCheckpoints[tokenId][token].lastClaimEpoch = currentEpoch;
        userCheckpoints[tokenId][token].unclaimed = amountClaimable - amount;

        // If the alchemist is defined we know it has an alchemic-token
        if (alchemists[alchemist] != address(0)) {
            require(token == IAlchemistV2(alchemist).debtToken(), "Invalid alchemist/alchemic-token pair");

            (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);
            IERC20(token).approve(alchemist, amount);

            // Only burn if there are deposits
            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;
        }
```

The usage of `approve()` in `claim` function of `RevenueHandler` at L#210 and in `depositIntoRewardPool` function of `RewardPoolManager` at L#87 does not implement the recommended approach for approving USDT tokens as provided by the USDT firm contract.

## Impact Details

The `claim` and `depositIntoRewardPool` function of contract `RevenueHandler` & `RewardPoolManager` respectively are impacted as they will revert when call are made to these function.

Reverting of above mentioned functions, making it impossible to deposit `USDT` and similar tokens into the contract for reward purpose.

## Recommended Mitigation Steps

Use `approve(_spender, 0)` to set the allowance to zero before the line each of the existing approve() calls made. Modify the `claim` function of contract `RevenueHandler` as below:

```diff
    /// @inheritdoc IRevenueHandler
    function claim(
        uint256 tokenId,
        address token,
        address alchemist,
        uint256 amount,
        address recipient
    ) external override {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, tokenId), "Not approved or owner");

        uint256 amountBurned = 0;

        uint256 amountClaimable = _claimable(tokenId, token);
        require(amount <= amountClaimable, "Not enough claimable");
        require(amount > 0, "Amount must be greater than 0");
        require(amount <= IERC20(token).balanceOf(address(this)), "Not enough revenue to claim");

        userCheckpoints[tokenId][token].lastClaimEpoch = currentEpoch;
        userCheckpoints[tokenId][token].unclaimed = amountClaimable - amount;

        // If the alchemist is defined we know it has an alchemic-token
        if (alchemists[alchemist] != address(0)) {
            require(token == IAlchemistV2(alchemist).debtToken(), "Invalid alchemist/alchemic-token pair");

            (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);

+	         IERC20(token).approve(alchemist, 0);
            IERC20(token).approve(alchemist, amount);

            // Only burn if there are deposits
            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;
        }
```

Changes for `depositIntoRewardPool` function of contract `RewardPoolManager` are as below:

```diff
function depositIntoRewardPool(uint256 _amount) external returns (bool) {
        require(msg.sender == veALCX, "must be veALCX");

+	     IERC20(poolToken).approve(rewardPool, 0);
        IERC20(poolToken).approve(rewardPool, _amount);
        IRewardPool4626(rewardPool).deposit(_amount, address(this));
        return true;
    }
```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardPoolManager.sol#L87>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L210>

## Proof of Concept

The `USDT` [contract](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code#L199) clearly mentions `approve` function to be used for changing allowance only after resetting the approved amount to zero.

```
// To change the approve amount you first have to reduce the addresses`
//  allowance to zero by calling `approve(_spender, 0)` if it is not
//  already 0
```
