# #41528 \[SC-High] When claiming rewards in native Bera via \`StakeV2.claimRewardsInNative\`, excess \`token0Debt\` or/and \`token1Debt\` is not returned to the kodiak vault but stuck in \`StakeV2\` contract.

**Submitted on Mar 16th 2025 at 09:46:33 UTC by @X0sauce for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41528
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Vulnerability Details

When claiming rewards in the form of native `Bera` tokens through `StakeV2.claimRewardsInNative`, any surplus unused `token0Debt` and `token1Debt` is not sent back to the kodiak vault. Instead, it is returned to the `StakeV2` contract, where it becomes stuck and cannot be utilized for future reward claims.

In `StakeV2.claimRewardsInNative`

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
@>        uint256 receivedAmount =
                            zapper.zapOutNative(msg.sender, swapData0, swapData1, unstakeParams, updatedRedeemParams); //@audit zaps out of ERC4626 vaultto native Bera Token to be given to staker as rewards

        emit Claimed(msg.sender, receivedAmount);
    }
```

In `Zapper.zapOutNative`

```solidity
    function zapOutNative(
        address receiver,
        SingleTokenSwap calldata swapData0,
        SingleTokenSwap calldata swapData1,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalNativeOut) {
@>        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams); //@audit withdraws island token from ERC4626 vault and retrieves underlying token0 and token1 from kodiak vault
        if (token0Debt == 0 && token1Debt == 0) {
            return (0);
        }

        totalNativeOut = _swapToWBERA(token0, token1, token0Debt, token1Debt, swapData0, swapData1);
        _sendNativeToken(receiver, totalNativeOut);
    }
```

In `Zapper._yeetOut`

```solidity
    function _yeetOut(
        IZapper.VaultRedeemParams calldata redeemParams,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams
    ) internal returns (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) {
@>        uint256 islandTokensReceived = _withdrawFromVault(redeemParams); //@audit withdraws island token from ERC4626 vault and retrieves underlying token0 and token1 from kodiak vault
        if (redeemParams.receiver == address(this)) {
            (token0, token1, token0Debt, token1Debt) =
@>            _approveAndUnstakeFromKodiakVault(unstakeParams, islandTokensReceived); // @audit removes token0 and token1 from kodiak vault by transferring islandToken
            if (unstakeParams.receiver != address(this)) {
                return (IERC20(address(0)), IERC20(address(0)), 0, 0);
            }
        }
    }
```

In `Zapper_approveAndUnstakeFromKodiakVault`

```solidity
    function _approveAndUnstakeFromKodiakVault(
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        uint256 islandTokenDebt
    ) internal returns (IERC20, IERC20, uint256, uint256) {
        // unstake from destination Island
        IERC20 _token0 = IKodiakVaultV1(unstakeParams.kodiakVault).token0();
        IERC20 _token1 = IKodiakVaultV1(unstakeParams.kodiakVault).token1();
        require(unstakeParams.receiver != address(0), "Zapper: zero address beneficiary");
        IERC20(address(unstakeParams.kodiakVault)).safeIncreaseAllowance(address(kodiakStakingRouter), islandTokenDebt);
        (uint256 _amount0, uint256 _amount1,) = kodiakStakingRouter.removeLiquidity(
            IKodiakVaultV1(unstakeParams.kodiakVault),
            islandTokenDebt,
            unstakeParams.amount0Min,
            unstakeParams.amount1Min,
            unstakeParams.receiver
        );

        // require(islandTokenDebt == _liqBurned, "Invalid island token burn amount");
@>        return (_token0, _token1, _amount0, _amount1); //@audit actual token0 and token1 amounts (amount0 and amount1) used for swap for Wbera that will be converted back to native Bera to be rewarded to staker
    }
```

Going back to `Zapper.zapOutNative`

```solidity
    function zapOutNative(
        address receiver,
        SingleTokenSwap calldata swapData0,
        SingleTokenSwap calldata swapData1,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalNativeOut) {
@>        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams); //@audit withdraws island token from ERC4626 vault and retrieves underlying token0 and token1 from kodiak vault
        if (token0Debt == 0 && token1Debt == 0) {
            return (0);
        }

@>        totalNativeOut = _swapToWBERA(token0, token1, token0Debt, token1Debt, swapData0, swapData1); //@audit Swap token0 and token1 for Wbera
        _sendNativeToken(receiver, totalNativeOut);
    }
```

In `Zapper._swapToWBERA`

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
@>        _clearUserDebt(token0, token1, token0Debt, token1Debt, _msgSender()); //@audit unused token0 or token1 amount is returned back to StakeV2 and stuck
    }
```

## Root Cause

Lack of mechanism in place to return any unused surplus of `token0` or `token1` to the Kodiak vault.

## Impact

Any surplus of `token0` or `token1` remains stuck in `StakeV2` and cannot be utilized for subsequent reward claims.

## Mitigation

Implement a mechasim to return the surplus `token0Debt` or `token1Debt` back to the Kodiak vaults for future reward claims.

## Proof of Concept

## POC

Consider the below simplistic scenario

1. Bob, a staker, initiates a call to `StakeV2.claimRewardsInNative` to claim rewards equivalent to 100 vault shares.
2. He proceeds to call `zapper.zapOutToToken0`, which leads him to `zapper._yeetOut`.
3. During this process, he calls `zapper._withdrawFromVault` and receives 10 `islandTokens` in exchange for the 100 vault shares redeemed from the ERC4626 vault.
4. The `redeemParams.receiver` is designated as the `Zapper` address, allowing Bob to enter `Zapper._approveAndUnstakeFromKodiakVault` to redeem the underlying `token0` and `token1` in exchange for the island tokens. In this process, he receives:
   * `_token0` = Wbera
   * `_token1` = USDC (the specific token type is not critical)
   * `_amount0` = 10
   * `_amount1` = 10
5. Afterwards, he returns to `Zapper.zapOutNative` and calls `Zapper._swapToWBERA` with the following parameters:
   * `token0` = `WBERA`
   * `token1` = `USDC`
   * `token0Debt` = `_token0` = 10
   * `token1Debt` = `_token1` = 10
6. Suppose the `Zapper._swapToWBERA` function is executed with a vault where `token0` is `WBERA`, and the subsequent call result in:
   * Only 5 out of the 10 `token1` (USDC) utilized to swap for 0.5 `WBERA`.
   * The remaining `token1Debt` = 10 - 5 = 5 `USDC` will be sent to `StakeV2` and become stuck instead of being returned to the Kodiak vault, where it could be used for future reward claims.
   * The total `wberaDebt` = 10 + 0.5 = 10.5 `WBERA`.
7. Ultimately, the total `wberaDebt` of 10.5 `WBERA` is transferred to Bob, while 5 `USDC` remains trapped in `StakeV2`.

A similar scenario can happen if `token1` is the `WBERA` token
