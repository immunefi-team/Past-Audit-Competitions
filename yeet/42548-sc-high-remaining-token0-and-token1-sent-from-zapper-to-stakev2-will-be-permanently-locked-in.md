# #42548 \[SC-High] Remaining token0 and token1 sent from Zapper to StakeV2 will be permanently locked in StakeV2 forever.

**Submitted on Mar 24th 2025 at 15:42:08 UTC by @kaysoft for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42548
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

In StakeV2.sol, every function to claim reward integrates the Zapper.sol.

The zapOut(...), zapOutNative(...), zapOutToToken1(...) and zapOutToToken0(...) all returned unused token0Debt and token1Debt to msg.sender but the StakeV2 does not send them to the user.

For example, one of the functions integrated by StakeV2, ZapOutNative(...) function warns in its last comment which I quote:

> @dev Bera is sent to the receiver. Any extra token0 and token1 is sent back to the \_msgSender().

> @dev integrating contracts must handle any returned token0 and token1

Truly this extra token0 and token1 are sent back to the StakeV2.sol but StakeV2.sol does not send this extra token0 and token1 to the user that calls the `claimRewardsInNative(...)` function causing the user to lose the extra token0 and token1.

```solidity
File: StakeV2.sol
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
                            zapper.zapOutNative(msg.sender, swapData0, swapData1, unstakeParams, updatedRedeemParams);//@audit issue token0 and token1 that are not swapped are locked here

        emit Claimed(msg.sender, receivedAmount);
    }
```

## Vulnerability Details

The major issue is that after the following functions in Zapper.sol does all the redeeming, removing Liquidity, and swapping, the `_clearUserDebt(...)` sends back the remaining `token0Debt` and `tokenDebt` to `_msgSender()`.\
This is because `_msgSender()` is passed to the `_clearUserDebt(...)` function as the receiver.

the `claimRewardsInNative(...)` function of StakeV2.sol calls the zap.zapOutNative(...). So the `_clearUserDebt(...)` in `zapOutNative(...)` will send the remaining `token0Debt` and token1Debt\` to StakeV2.sol.

However this remaining `token0Debt` and token1Debt`sent to the StakeV2.sol are not sent to the user but locked in the StakeV2.sol contract because after calling the`zap.zapOutNative(...)`function,`claimRewardsInNative(...)\` only emitted an event.

This can even cause the user to lose all their rewards to StakeV2.sol if they pass `swap0.inputAmount` and `swap1.inputAmount` both as zero.

The zapOut(...), zapOutNative(...), zapOutToToken1(...) and zapOutToToken0(...) all return the remaining tokens to StakeV2.sol but those returned tokens are not sent to the user but locked in StakeV2.sol forever.

1. Let's look at the zapOut(...) function where token0Debt and token1Debt is sent to msg.sender in this case StakeV2.sol is the msg.sender.

```solidity
File: Zapper.sol
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
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
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
            totalAmountOut += _verifyTokenAndSwap(swap0, address(token0), outputToken, receiver);
            token0Debt -= swap0.inputAmount;
            totalAmountOut += _verifyTokenAndSwap(swap1, address(token1), outputToken, receiver);
            token1Debt -= swap1.inputAmount;
        }
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());
    }
```

2. Let's look at another function where remaining token is sent to `msg.sender`(StakeV2.sol).

```solidity
File: Zapper.sol
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
        token1Debt -= swapData.inputAmount;
        token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
        _sendERC20Token(token0, receiver, token0Debt);
        _sendERC20Token(token1, _msgSender(), token1Debt);//@audit here StakeV2 receives token1Debt but does not send to user of `claimRewardsInToken1()`
        return (token0Debt);
    }
```

Zapper.sol handles redemption and removing liquidity to get the `token0` and `token1` and also swap it to the desired output tokens.

During this swap By Zapper.sol, `swap0.inputAmount` is deducted from `token0Debt` amount and the remaining token0Debt is sent back to the `msg.sender` which is the StakeV2.sol. The issue is that the StakeV2 does not send the remaining `token0Debt` and `token1Debt`to the user.

## Impact Details

* Loss of token0 and token1 that are not swapped to the StakeV2 contract
* The above token0 and token1 that are not swapped will be locked in StakeV2 contract forever.
* This loss is highly likely to occur for all users because users are not sure of the exact output amounts of token0 and token1 from the removeLiquidity operation that's why users supply `unstakeParams.amount0Min` and `unstakeParams.amount0Min`.
* Users are allowed to pass in `swap0.inputAmount` that is zero and in this case all rewards will be lost because when `swap0.inputAmount` is zero, `verifyAndSwapToken(...)` returns zero and execution continues with the token0Debt being the full amount of the reward.

```solidity
function _verifyTokenAndSwap(
        SingleTokenSwap calldata swapData,
        address inputToken,
        address outputToken,
        address receiver
    ) internal returns (uint256 amountOut) {
        if (swapData.inputAmount == 0) {
            return 0;
        }
...
}
```

## Recommendations

Consider sending the remaining `token0Debt` and `token1Debt` returned to StakeV2.sol from Zapper to the user that is claiming their reward.

## Proof of Concept

## Proof of Concept

1. Bob deposits to StakeV2.sol
2. Bob earns reward
3. Bob calls `claimRewardsInToken(...)` function to claim all his reward while passing `swap0.inputAmount` and `swap1.inputAmount` both as 80 as input to the `claimRewardsInToken(...)` function function call.
4. 100 of both token0 and token1 where received by the Zapper during removeLiquidity.
5. Bob receives 80 token0 and token1 worth of `output` token from the swap during the transaction.
6. The remaining 20 token0 and 20 token1 are sent back to StakeV2.sol
7. Bob losts 20 token0 and 20 token1 amounts to the StakeV2.sol which is locked in the contract forever.

The Zapper.sol#zapOut(...) function below with `_clearDebt(...)` sending the remaining token to StakeV2.sol

```solidity
File: Zapper.sol
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
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
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
            totalAmountOut += _verifyTokenAndSwap(swap0, address(token0), outputToken, receiver);
            token0Debt -= swap0.inputAmount;
            totalAmountOut += _verifyTokenAndSwap(swap1, address(token1), outputToken, receiver);
            token1Debt -= swap1.inputAmount;
        }
        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender());//@audit token0Debt and token1Debt are locked in StakeV2
    }
```
