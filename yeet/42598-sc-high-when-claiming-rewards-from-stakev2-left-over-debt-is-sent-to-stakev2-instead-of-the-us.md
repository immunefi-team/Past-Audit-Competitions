# #42598 \[SC-High] When claiming rewards from \`StakeV2\` left-over debt is sent to \`StakeV2\` instead of the user

**Submitted on Mar 24th 2025 at 22:08:01 UTC by @kmm for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42598
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When claiming rewards from `StakeV2`, vault tokens are zapped out via the zapper contract. However, if a user specifies a smaller `inputAmount` for the token swap, any excess tokens are erroneously sent to `StakeV2` and become permanently locked.

## Vulnerability Details

Users claim their rewards using the following functions:

* `claimRewardsInNative`
* `claimRewardsInToken0`
* `claimRewardsInToken1`
* `claimRewardsInToken`

The vulnerability applies to all of them, but we will focus on `claimRewardsInToken1`:

```solidity
function claimRewardsInToken1(
    uint256 amountToWithdraw,
    IZapper.SingleTokenSwap calldata swapData,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    IZapper.VaultRedeemParams calldata redeemParams
) external nonReentrant {
    _updateRewards(msg.sender); // Update the rewards

    IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

    IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);

    uint256 receivedAmount = zapper.zapOutToToken1(msg.sender, swapData, unstakeParams, updatedRedeemParams);

    emit Claimed(msg.sender, receivedAmount);
}
```

The function validates the withdrawal amount and delegates to the zapper via `zapOutToToken1`:

```solidity
function zapOutToToken1(
    address receiver,
    SingleTokenSwap calldata swapData,
    KodiakVaultUnstakingParams calldata unstakeParams,
    VaultRedeemParams calldata redeemParams
) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken1Out) {
    (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
    
    if (token0Debt == 0 && token1Debt == 0) {
        return 0;
    }

    token0Debt -= swapData.inputAmount;
    token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this));

    _sendERC20Token(token0, _msgSender(), token0Debt);
    _sendERC20Token(token1, receiver, token1Debt);

    return token1Debt;
}
```

Here, `_yeetOut` returns amounts in `token0Debt` and `token1Debt`. The user provides a swap amount (`swapData.inputAmount`) for converting `token0` to `token1`. Any leftover `token0Debt` (i.e., `token0Debt - inputAmount`) is sent to `_msgSender()`, which is the `StakeV2` contract.

This leads to a scenario where if a user intends to convert only part of their `token0Debt` (e.g., 50%), the remaining 50% is sent to `StakeV2` and becomes inaccessible to the user. This results in a **permanent partial loss**.

## Impact Details

Permanent, irrecoverable loss of user funds occurs when the user specifies a `swapData.inputAmount` smaller than the full `token0Debt`. The excess `token0` is sent to `StakeV2` and becomes trapped.

## References

* [StakeV2.sol#L372](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L372)
* [Zapper.sol#L284](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L284)

## Proof of Concept

## Proof of Concept

For simplicity assume that the rate of token0/token1 is always 1:1.

1. User has 5 compounding vault shares that equal (500 token0 and 500 token1).
2. User call claim `claimRewardsInToken1` and passes `swapData.inputAmount = 250`
3. 250 token0 is swapped for 250 token1 tokens
4. 750 token1 tokens are sent out to the user, while 250 token0 tokens are sent to `StakeV2`, causing a net loss of 250 token0 tokens for the user.
