# #42532 \[SC-High] Compound function in MoneyBrinter can lead to loss of yield

**Submitted on Mar 24th 2025 at 14:38:16 UTC by @dobrevaleri for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42532
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Partial loss of yield

## Description

## Summary

The `MoneyBrinter::compound()` function can be called by whitelisted managers to compound rewards. When called right before a user withdrawal, it can lead to loss of yield for users, due to the hardcoded swap parameters passed to the claim function.

## Vulnerability Details

The issue is in `StakeV2::claimRewardsIn...()` where users need to simulate and pass exact swap parameters to claim their rewards:

```solidity
    struct VaultRedeemParams {
        address vault;
        address receiver;
        uint256 shares;
        uint256 minAssets; // front-running protection!!
    }

     struct KodiakVaultStakingParams {
        address kodiakVault;
        uint256 amount0Max;
        uint256 amount1Max;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 amountSharesMin;
        address receiver;
    }

    struct SingleTokenSwap {
        uint256 inputAmount;
        uint256 outputQuote;
        uint256 outputMin;
        address executor;
        bytes path;
    }

    function claimRewardsInToken0(
        uint256 amountToWithdraw,
        IZapper.SingleTokenSwap calldata swapData,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) external nonReentrant {
        _updateRewards(msg.sender);

        IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

        IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
@>      uint256 receivedAmount = zapper.zapOutToToken0(msg.sender, swapData, unstakeParams, updatedRedeemParams);

        emit Claimed(msg.sender, receivedAmount);
    }
```

These parameters are used in the `Zapper.sol` to withdraw from the `MoneyBrinter` and `KodiakVault`. Only the `swapData.inputAmount` will be used to swap the received assets into the desired token. So the user will receive only this amount, even though that the `swapData.inputAmount` can be much smaller than the `token1Debt`.

```solidity
// Zapper.sol
function zapOutToToken0(
    address receiver,
    SingleTokenSwap calldata swapData,
    KodiakVaultUnstakingParams calldata unstakeParams,
    VaultRedeemParams calldata redeemParams
) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken0Out) {
    (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
    if (token0Debt == 0 && token1Debt == 0) {
        return (0);
    }
@>  token1Debt -= swapData.inputAmount; // Input Amount could be much less then the token1Debt
    token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
    _sendERC20Token(token0, receiver, token0Debt);
    _sendERC20Token(token1, _msgSender(), token1Debt);
    return (token0Debt);
}
```

The key issue is that a strategy manager can call `MoneyBrinter::compound()` right before the user's withdrawal transaction, which increases the share value. Since the user already simulated and provided exact swap parameters earlier, the newly accrued yield will be claimed, but instead of being swapped and transferred to the user it will be transferred to `StakeV2`.

## Impact

The users can receive less yield, if the `compound()` is executed between the parameters construction (`previewRedeem()` and simulate remove liquidity from Kodiak) and the execution of the claim transaction.

## Proof of Concept

## Proof of Concept

1. User deposits tokens into `StakeV2`.
2. `StakeV2` receives Bera tokens from YeetGame.
3. `StakeV2` executes `executeRewardDistribution()`, providing MoneyBrinter shares to stakers.
4. After some time, Kodiak and Beradrome rewards accrue
5. User simulates withdrawal and calls `claimRewardsInToken0()` with simulated parameters.
6. Manager executes `compound()` call, before the user's transaction is executed.
7. User's transaction executes with outdated parameters.
8. The difference in yield is transferred to `StakeV2` contract and split among all stakers.
