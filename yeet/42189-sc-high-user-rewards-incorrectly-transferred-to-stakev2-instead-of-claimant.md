# #42189 \[SC-High] User rewards incorrectly transferred to \`StakeV2\` instead of claimant

**Submitted on Mar 21st 2025 at 15:39:46 UTC by @Ragnarok for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42189
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Users may lose rewards when claiming them due to a portion of the rewards being transferred to the `StakeV2` contract instead of the user.

## Vulnerability Details

When users claim rewards, they can call one of the following functions: `claimRewardsInToken0`, `claimRewardsInToken1`, `claimRewardsInNative`, or `claimRewardsInToken`.

In particular, the `StakeV2::claimRewardsInToken0` function calls `Zapper::zapOutToToken0`, which swaps `token1` for `token0`, then transfers all `token0` to the user (`receiver` parameter). However, any remaining `token1` is sent to the `StakeV2` contract (`msg.sender` in the context of `zapOutToToken0`) instead of the user.

This behavior is incorrect because the remaining `token1` is part of the user's rewards and should be transferred to them. A similar issue occurs in other reward-claiming functions.

`StakeV2::claimRewardsInToken0` function:

```solidity
function claimRewardsInToken0(
    uint256 amountToWithdraw,
    IZapper.SingleTokenSwap calldata swapData,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    IZapper.VaultRedeemParams calldata redeemParams
) external nonReentrant {
    _updateRewards(msg.sender);

    IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

    IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
    // @comment msg.sender receives back token0
    // @comment This contract receives back the remaining token1 (after swapping token1 to token0)
    // @audit The remaining token1 should be transferred to msg.sender
=>  uint256 receivedAmount = zapper.zapOutToToken0(msg.sender, swapData, unstakeParams, updatedRedeemParams);

    emit Claimed(msg.sender, receivedAmount);
}
```

## Impact Details

Users may lose a portion of their rewards when claiming them, as some rewards are inadvertently sent to the `StakeV2` contract instead of the user.

## Proof of Concept

## Proof of Concept

Consider the following scenario:

1. The user calls `StakeV2::claimRewardsInToken0` to claim rewards with `swapData.inputAmount = 10 ether`.
2. After `removeLiquidity`, the actual received amount of `token1` is `11 ether`.
3. The remaining `1 ether` of `token1` is incorrectly transferred to the `StakeV2` contract instead of the user.
