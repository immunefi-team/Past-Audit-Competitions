# #41633 \[SC-High] Users might lose some of the rewards they’re supposed to get.

**Submitted on Mar 17th 2025 at 06:19:51 UTC by @p0wd3r for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41633
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Users might lose some of the rewards they’re supposed to get.

## Vulnerability Details

During the `zapOut` process, the zapper will perform a swap based on the input `swapData`, and if there’s any leftover, it’ll return it to the `msg.sender` as the comment says.

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L296-L312>

```solidity
    /// @dev Bera is sent to the receiver. Any extra token0 and token1 is sent back to the _msgSender().
    /// @dev integrating contracts must handle any returned token0 and token1
    function zapOutNative(
        address receiver,
        SingleTokenSwap calldata swapData0,
        SingleTokenSwap calldata swapData1,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalNativeOut) {
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
        if (token0Debt == 0 && token1Debt == 0) {
            return (0);
        }

        totalNativeOut = _swapToWBERA(token0, token1, token0Debt, token1Debt, swapData0, swapData1);
        _sendNativeToken(receiver, totalNativeOut);
    }
```

```solidity
    function _swapToWBERA(
        IERC20 token0,
        IERC20 token1,
        uint256 token0Debt,
        uint256 token1Debt,
        SingleTokenSwap calldata swapData0,
        SingleTokenSwap calldata swapData1
    ) internal returns (uint256 wBeraDebt) {
        if (address(token0) == address(wbera)) {
            wBeraDebt += token0Debt;
            token0Debt = 0;
        } else {
            wBeraDebt += _verifyTokenAndSwap(swapData0, address(token0), address(wbera), address(this));
            token0Debt -= swapData0.inputAmount;
        }

        if (address(token1) == address(wbera)) {
            wBeraDebt += token1Debt;
            token1Debt = 0;
        } else {
            wBeraDebt += _verifyTokenAndSwap(swapData1, address(token1), address(wbera), address(this));
            token1Debt -= swapData1.inputAmount;
        }
        // log yeetBalance
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
    }
```

User will call `zapper.zapOut` through `claimRewardsInNative`, meaning the leftover tokens will be returned to the `StakeV2` contract instead of the user.

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L327-L343>

```solidity
    function claimRewardsInNative(
        uint256 amountToWithdraw,
        IZapper.SingleTokenSwap calldata swapData0,
        IZapper.SingleTokenSwap calldata swapData1,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) external nonReentrant {
        _updateRewards(msg.sender);

        IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

        IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
        uint256 receivedAmount =
                            zapper.zapOutNative(msg.sender, swapData0, swapData1, unstakeParams, updatedRedeemParams);

        emit Claimed(msg.sender, receivedAmount);
    }
```

Those tokens should actually belong to the user since they were exchanged using their vault share.

## Impact Details

Users might lose some of the rewards they’re supposed to get and it’s really hard for users to set `swapData.inputAmount` to match the `returned amount` because the price of the share keeps changing.

## References

* <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L296-L312>
* <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L327-L343>

## Proof of Concept

## Proof of Concept

1. The user calls `claimRewardsInNative` and owns 100 vault shares, which at the current market price can be exchanged for 50 token0. So, the user sets `swapData.inputAmount` to 50.
2. When the user’s transaction is executed on-chain, the price changes, and 100 vault shares can now be exchanged for 60 token0.
3. Since the user set `inputAmount` to 50, they only receive the native token equivalent of 50 token0. The remaining 10 token0 are transferred to the `StakeV2` contract, and the user can no longer access them, resulting in the user losing the reward equivalent to those 10 token0.
