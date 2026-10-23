# #41647 \[SC-High] Unused tokens after zapping can be stuck and not entitled to users

**Submitted on Mar 17th 2025 at 08:38:58 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41647
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The leftover tokens after a zapping action is sent from `Zapper` to the contract `StakeV2`. These tokens are unhandled by `StakeV2`, causing the tokens to be permanently stuck in the contract

## Vulnerability Details

Users can claim staking rewards in either token0, token1, native token or whitelisted token through functions `StakeV2#claimRewardsInNative()`, `StakeV2#claimRewardsInToken0()`, `StakeV2#claimRewardsInToken1()`, `StakeV2#claimRewardsInToken()`

In the above functions, the user will spend vault shares and call `Zapper` contract to handle zap out the position to the wanted token.

For example with the function `StakeV2#claimRewardsInToken1()`, the contract `StakeV2` calls `Zapper#zapOutToToken1()` to redeem from Vault to receive `islandTokens` which is then used to remove liquidity from KodiakVault to finally receive pool's `token0` and `token1`. After received `token0` and `token1`, the `Zapper` contract calls OB SwapRouter to swap `token0` --> `token1` (here user claims rewards in token1). After swap, all unused `token0` are sent back to `StakeV2` contract, and all `token1` are sent to the user. Here, the function `StakeV2#claimRewardsInToken1()` does not handle the unused/leftover `token0` sent from `Zapper` contract, which can cause the tokens to stuck in the contract

```solidity
    // StakeV2 contract
    function claimRewardsInToken1(
        uint256 amountToWithdraw,
        IZapper.SingleTokenSwap calldata swapData,
        IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
        IZapper.VaultRedeemParams calldata redeemParams
    ) external nonReentrant {
        _updateRewards(msg.sender);

        IZapper.VaultRedeemParams memory updatedRedeemParams = _verifyAndPrepareClaim(amountToWithdraw, redeemParams);

        IERC20(redeemParams.vault).approve(address(zapper), amountToWithdraw);

        uint256 receivedAmount = zapper.zapOutToToken1(msg.sender, swapData, unstakeParams, updatedRedeemParams); // <<<<< call Zapper to handle zap out to token1

        /// HERE lack the logics to handle the leftover tokens sent from Zapper contract

        emit Claimed(msg.sender, receivedAmount);
    }

////////////

// Zapper contract
    function zapOutToToken1(
        address receiver,
        SingleTokenSwap calldata swapData,
        KodiakVaultUnstakingParams calldata unstakeParams,
        VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken1Out) {
        (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams); // << handle redeem and remove liquidity to get token0, token1
        if (token0Debt == 0 && token1Debt == 0) {
            return (0);
        }
        token0Debt -= swapData.inputAmount;
        token1Debt += _verifyTokenAndSwap(swapData, address(token0), address(token1), address(this)); // << handle swap token0 to token1

        _sendERC20Token(token0, _msgSender(), token0Debt); // << refund token0 to msg.sender, which is Stake contract
        _sendERC20Token(token1, receiver, token1Debt); // << send token1 to user as rewards
        return (token1Debt);
    }
```

Indeed, if the liquidity pair is BERA-YEET, then the tokens can be used for the next reward distributions. However, these tokens should be entitled to the user, not for the reward distributions because it is redeemed from user shares amount

## Impact Details

* Users can not get the tokens that should be entitled to receive and the tokens are distributed among other stakers

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L361-L375\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L272-L287>

## Proof of Concept

## Proof of Concept

1.Update the test `test_successful_zap_out_yeet` at file `test/zapper/ZapOut.t.sol`

```solidity
    function test_successful_zap_out_yeet() public {
        // 1% slippage during LP redeem
        IZapper.SingleTokenSwap memory swapInfo = prepareSwapInfo(
            amount1,
            97842998386015666176, // assuming 2% slippage
            95886138418295352852, // assuming 2% slippage
            0xDa547d8ce09e23E9e8053dd187B58841B5fB8D5d,
            vm.parseBytes(
                "0x1740F679325ef3686B2f574e392007A92e4BeD417bC98B68bCBb16cEC81EdDcEa1A3746Fdc5025A401017507c1dc16935B82698e4C63f2746A2fCf994dF802e79e01B6a43bc17680fb67fD8371977d264E047f47c67500Da547d8ce09e23E9e8053dd187B58841B5fB8D5dffff00692bB44820568223798f2577D092BAf0d696dd4701Da547d8ce09e23E9e8053dd187B58841B5fB8D5d000bb801d6D83aF58a19Cd14eF3CF6fe848C9A4d21e5727c01ffff0193439A0E080805b5a147019410B3CeAf12bF4AE900Da547d8ce09e23E9e8053dd187B58841B5fB8D5d01277aaDBd9ea3dB8Fe9eA40eA6E09F6203724BdaE01ffff01EA2e981f185e6A4B53eb1B72792CF02e2EBCbDcB00Da547d8ce09e23E9e8053dd187B58841B5fB8D5d010E4aaF1351de4c0264C5c7056Ef3777b41BD8e030280150027882A7D759355842294A846F039c8e29EC0e3d501Da547d8ce09e23E9e8053dd187B58841B5fB8D5d000bb8ffff01246c12D7F176B93e32015015dAB8329977de981B011E55c4C69acAeb49b2834FF5Bc5D8De5d716B39004f5AFCF50006944d17226978e594D4D25f4f92B40001E55c4C69acAeb49b2834FF5Bc5D8De5d716B39000Da547d8ce09e23E9e8053dd187B58841B5fB8D5d000bb8"
            )
        );
        IZapper.SingleTokenSwap memory noSwap;
        (IZapper.VaultRedeemParams memory vaultParams, IZapper.KodiakVaultUnstakingParams memory islandUnstakingParams)
        = prepareVaultRedeemAndUnstakeParams(moneyBrinter, vaultShares, 100, amount0, amount1, 100, zapper, zapper);

        vm.prank(alice);
        address staking = makeAddr('staking');
        IERC20(moneyBrinter).transfer(staking, vaultParams.shares);

        approveVaultTokens(staking, moneyBrinter, vaultParams.shares);
        
        vm.prank(staking);
        uint256 amountOut = contracts.zapper.zapOutToToken0(alice, swapInfo, islandUnstakingParams, vaultParams);

        console.log("Token Out: ", amountOut);
        verifyMinOutputTokens(
            yeet, islandUnstakingParams.amount0Min, islandUnstakingParams.amount1Min, noSwap, swapInfo, amountOut
        );
        verifyNoBalanceInZapper();

        vm.assertEq(IERC20(yeet).balanceOf(staking), 0, "yeet balance in staking is not 0, should belong to alice");
        vm.assertEq(IERC20(Wbera).balanceOf(staking), 0, "Wbera balance in staking is not 0, should belong to alice");
    }
```

2. Run the test and console shows

```bash
Failing tests:
Encountered 1 failing test in test/zapper/ZapOut.t.sol:ZapOut
[FAIL: Wbera balance in staking is not 0, should belong to alice: 440897924579083 != 0] test_successful_zap_out_yeet() (gas: 1621201)
```

The result shows that YEET is sent to Alice successfully, but WBERA is not
