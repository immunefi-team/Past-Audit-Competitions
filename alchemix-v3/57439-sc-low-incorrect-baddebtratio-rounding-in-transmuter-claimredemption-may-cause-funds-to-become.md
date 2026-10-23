# 57439 sc low incorrect baddebtratio rounding in transmuter claimredemption may cause funds to become permanently stuck

**Submitted on Oct 26th 2025 at 08:54:47 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57439
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `Transmuter::claimRedemption()` function contains a rounding error in the calculation of `badDebtRatio`. When this value is incorrectly rounded down, it can cause `scaledTransmuted` to exceed the actual redeemable amount. As a result, the subsequent redemption call may revert due to insufficient funds. Furthermore, the `AlchemistV3::redeem()` function does not include protocol fees in the `amountToRedeem` calculation, which can also trigger unexpected reverts and lock user funds within the contract.

## Vulnerability Details

As marked by the `@>` tags in the following code snippets:

* @>1 The `badDebtRatio` is calculated using integer division that rounds down, making it smaller than the correct ratio.
* @>2 When 1badDebtRatio > 1e181, the computed scaledTransmuted becomes larger than the actual redeemable amount, leading to an overestimation of available funds.
* @>3 Consequently, amountToRedeem also exceeds the available collateral, causing `claimRedemption() -> redeem()` to revert during redemption. This results in funds being stuck within the contract. Even when the claimer is not the last participant in a redemption pool, rounding errors can still cause an overclaim, indirectly harming other redemption creators.

```js
    // Transmuter::claimRedemption()
    function claimRedemption(uint256 id) external {

        // SNIP...

        // Burn position NFT
        _burn(id);
        
        // Ratio of total synthetics issued by the alchemist / underlingying value of collateral stored in the alchemist
        // If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
@>1        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;


        uint256 scaledTransmuted = amountTransmuted;


@>2        if (badDebtRatio > 1e18) {
@>2            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }


        // If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
@>3        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;


@>3        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);


        // SNIP...
    }
```

Additionally, in the `AlchemistV3::redeem()` function, the `protocolFee` is not included in the `collRedeemed` amount. When `protocolFee > 0`, the safeTransfer of `feeCollateral` may revert, also resulting in stuck funds.

```js
    // AlchemistV3::redeem()
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();


        // SNIP...


        // move only the net collateral + fee
        uint256 collRedeemed  = convertDebtTokensToYield(amount);
@>        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
        uint256 totalOut      = collRedeemed + feeCollateral;


        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
        _totalLocked = totalOut > old ? 0 : old - totalOut;
        _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);


        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
@>        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
        _mytSharesDeposited -= collRedeemed + feeCollateral;


        emit Redemption(redeemedDebtTotal);
    }
```

## Impact Details

* The incorrect rounding of `badDebtRatio` can cause `scaledTransmuted` to exceed the actual redeemable balance.
* As a result, the `claimRedemption()` call may revert due to insufficient funds, permanently locking user funds in the contract.
* Furthermore, when `protocolFee > 0`, the missing fee inclusion in `amountToRedeem` can also cause redemption transactions to fail.
* This issue may impact both individual users and the protocol’s liquidity pool, resulting in frozen redemptions and unclaimable assets.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L220-L232>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L627-L638>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistV3.t.sol` and run it:

```js
    function test_claimRedemption_badDebtRatio() public {
        ////////////////////////////////
        //  0xbeef deposit() + mint() //
        ////////////////////////////////
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        //////////////////////////////
        //  user deposit() + mint() //
        //////////////////////////////
        address user = makeAddr("user");
        vm.prank(address(0xdead));
        whitelist.add(user);
        deal(address(vault), user, accountFunds);
        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, user, 0);
        uint256 tokenIdForUser = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.mint(tokenIdForUser, mintAmount, user);
        vm.stopPrank();

        /////////////////////////////////////////////
        //  anotherExternalUser createRedemption() //
        /////////////////////////////////////////////
        // Need to start a transmutator deposit, to start earmarking debt
        deal(address(alToken), address(anotherExternalUser), accountFunds * 2);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount * 2);
        transmuterLogic.createRedemption(mintAmount * 2);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // decreasing yeild token suppy by 20%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 2000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));

        // Calculate `badDebtRatio` with the same rounding down as the contract
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
        console.log("badDebtRatio round down:", badDebtRatio);

        ////////////////////////////////////////////
        //  anotherExternalUser claimRedemption() //
        ////////////////////////////////////////////

        vm.startPrank(anotherExternalUser);
        // MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 400000000000000000199999 [4e23])
        // [FAIL: ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)] ❌
        vm.expectRevert(); 
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        // Calculate `badDebtRatio` and round it up
        denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        badDebtRatio = (alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator) + ((alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken())) % denominator > 0 ? 1 : 0);

        console.log("badDebtRatio round up:", badDebtRatio);
    }




    // Before running the following tests, please fix `Transmuter::claimRedemption::badDebtRatio` to round up
    function test_redeem_protocolFee() public {
        ////////////////////////////////
        //  0xbeef deposit() + mint() //
        ////////////////////////////////
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        //////////////////////////////
        //  user deposit() + mint() //
        //////////////////////////////
        address user = makeAddr("user");
        vm.prank(address(0xdead));
        whitelist.add(user);
        deal(address(vault), user, accountFunds);
        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, user, 0);
        uint256 tokenIdForUser = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        alchemist.mint(tokenIdForUser, mintAmount, user);
        vm.stopPrank();

        /////////////////////////////////////////////
        //  anotherExternalUser createRedemption() //
        /////////////////////////////////////////////
        // Need to start a transmutator deposit, to start earmarking debt
        deal(address(alToken), address(anotherExternalUser), accountFunds * 2);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount * 2);
        transmuterLogic.createRedemption(mintAmount * 2);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // decreasing yeild token suppy by 20%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 2000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        ////////////////////////////////////////////
        //  anotherExternalUser claimRedemption() //
        ////////////////////////////////////////////

        // setProtocolFee
        vm.prank(alchemist.admin());
        alchemist.setProtocolFee(20);
        vm.startPrank(anotherExternalUser);
        // MockMYTVault::transfer(PointEvaluation: [0x000000000000000000000000000000000000000A], 799999999999999999659 [7.999e20])
        // ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011) ❌
        vm.expectRevert(); 
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();
    }
```
