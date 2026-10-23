# #41644 \[SC-High] \`\_clearUserDebt\` in zapOut function sends the remaining tokens to \`msg.sender\` instead of receiver.

**Submitted on Mar 17th 2025 at 08:22:59 UTC by @OxAnmol for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41644
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `Zapper:zapOut` function uses `_clearUserDebt` to send remaining tokens that were not swapped back to the user. However, `_msgSender()` is incorrectly used as the destination address, which sends funds to the StakerV2 contract instead of the actual user.

## Vulnerability Details

When users call `claimRewardsInToken`, `claimRewardInToken1`, or claimRewardInToken0 in the `StakerV2` contract, they provide a `SingleTokenSwap` struct as a parameter. This includes an `amountIn` for the token the user wants to swap from.

```solidity
function claimRewardsInToken(
    uint256 amountToWithdraw,
    address outputToken,
    IZapper.SingleTokenSwap calldata swap0,
    IZapper.SingleTokenSwap calldata swap1,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    IZapper.VaultRedeemParams calldata redeemParams
) external nonReentrant {
    _updateRewards(msg.sender);

    IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

    IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);
    uint256 receivedAmount =
                        zapper.zapOut(outputToken, msg.sender, swap0, swap1, unstakeParams, updatedRedeemParams);

    emit Claimed(msg.sender, receivedAmount);
}
```

The `Zapper:zapOut` function performs several operations:

1. Redeems token1 and token0/YEET and WBERA from Kodiak
2. Swaps token1 and token0 to the desired output token based on the amountIn provided by the user
3. Sends the output amount to the user/receiver
4. Checks if there are any remaining token1 and token0 that were not used for swapping
5. In step 4, it uses \_msgSender() (which is the StakerV2 contract) as the destination for unused tokens instead of the receiver

```solidity
function zapOut(
    address outputToken,
    address receiver,
    SingleTokenSwap calldata swap0,
    SingleTokenSwap calldata swap1,
    KodiakVaultUnstakingParams calldata unstakeParams,
    VaultRedeemParams calldata redeemParams
)
    public
    override
    nonReentrant
    onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault)
    returns (uint256 totalAmountOut)
{
    (
        IERC20 token0,
        IERC20 token1,
        uint256 token0Debt,
        uint256 token1Debt
    ) = _yeetOut(redeemParams, unstakeParams);
    if (token0Debt == 0 && token1Debt == 0) {
        return totalAmountOut;
    }
    // @note -> Do I need to check this, what happens if I don't.
    if (outputToken == address(token0)) {
        revert("Zapper: Invalid output token");
    } else if (outputToken == address(token1)) {
        revert("Zapper: Invalid output token");
    } else {
        // sent directly to receiver. What is receiver is zapper?
        totalAmountOut += _verifyTokenAndSwap(
            swap0,
            address(token0),
            outputToken,
            receiver
        );
        token0Debt -= swap0.inputAmount;
        totalAmountOut += _verifyTokenAndSwap(
            swap1,
            address(token1),
            outputToken,
            receiver
        );
        token1Debt -= swap1.inputAmount;
    }
    //@audit should I send token0 and token1 to receiver/user ?
->>  _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
}
```

## Impact Details

Users will lose all tokens that were not used for swapping. According to the impact scope, this constitutes "Permanent freezing of funds," which is a critical severity bug.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L352>

## Proof of Concept

### Flow

1. A user calls `StakerV2:claimRewardsInToken` to claim a reward in USDC. They want to swap 100 token0 and 100 token1 to USDC.
2. `Zapper:zapOut` is called and calculates the amounts. Based on the user's stakes, they're eligible for 200 token0 and 200 token1.
3. `_verifyTokenAndSwap` swaps 100 token0 and 100 token1 to USDC as specified by the user and sends that to the receiver/user.
4. Only 100 tokens of each type are utilized, and the remaining 100 of each should be returned to the receiver.

However, in \_clearUserDebt, \_msgSender() is used as the destination for remaining funds, which is actually the StakerV2 contract, not the user
