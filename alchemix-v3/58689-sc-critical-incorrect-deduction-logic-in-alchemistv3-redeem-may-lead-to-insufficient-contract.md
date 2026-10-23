# 58689 sc critical incorrect deduction logic in alchemistv3 redeem may lead to insufficient contract collateral

**Submitted on Nov 4th 2025 at 03:54:47 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58689
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `AlchemistV3::redeem()` function contains a logic flaw in how redemption fees are deducted and synchronized. Specifically, `feeCollateral` is subtracted directly from `_mytSharesDeposited` during redemption, expect to perform deduction via `_collateralWeight` when synchronizing. This design can lead to situations where a user’s redemption unintentionally causes contract's collateral to be reduced—especially during periods of sharp price fluctuation.

## Vulnerability Details

The `redeem()` function in `AlchemistV3` is called by the `Transmuter` contract to process a redeemer’s request to convert myt tokens. However, the redemption fee (`feeCollateral`) is deducted directly from `_mytSharesDeposited` (`@>1`), assuming the subsequent `_sync()` call will later update borrower account’s collateral balance using `_collateralWeight` (`@>2`).

```js
    // AlchemistV3::redeem()
    function redeem(uint256 amount) external onlyTransmuter {
        _earmark();


        // SNIP...

        // move only the net collateral + fee
@>        uint256 collRedeemed  = convertDebtTokensToYield(amount);
@>        uint256 feeCollateral = collRedeemed * protocolFee / BPS;
@>        uint256 totalOut      = collRedeemed + feeCollateral;


        // update locked collateral + collateral weight
        uint256 old = _totalLocked;
@>        _totalLocked = totalOut > old ? 0 : old - totalOut;
@>1       _collateralWeight += PositionDecay.WeightIncrement(totalOut > old ? old : totalOut, old);


@>        TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
@>        TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
@>1       _mytSharesDeposited -= collRedeemed + feeCollateral;


        emit Redemption(redeemedDebtTotal);
    }
```

However, if there is significant price volatility during the redemption process, the borrower's collateral balance may be insufficient to cover the deducted collateral. In this case, the shortfall will be borne by other users, preventing them from making proper withdrawals:

```js
    // AlchemistV3::_sync()
    function _sync(uint256 tokenId) internal {
        Account storage account = _accounts[tokenId];


        // Collateral to remove from redemptions and fees
@>2        uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight);
@>2       account.collateralBalance -= collateralToRemove;


        // SNIP...
    }
```

Thus, the deduction model mixes user-level and system-level accounting, creating potential cross-account contamination when redemption fees are calculated under volatile market conditions.

## Impact Details

1. Collateral Misallocation: When the redeeming user’s collateral is insufficient, feeCollateral is effectively sourced from the collective pool, unintentionally reducing other users’ collateral balances.
2. Incorrect Fee Attribution: The protocol fee intended to be covered by the redeemer may instead be paid indirectly by other depositors, leading to unfair loss distribution.
3. Potential Insolvency Risk: Over time, repeated redemptions during volatile conditions could cause discrepancies between reported and actual collateral values, increasing the risk of protocol under-collateralization.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L589-L641>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1042-L1095>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistV3.t.sol` and run it:

```js
    function test_redeem_feeCollateral() public {
        // Set the protocol fee
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(100);

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


        ////////////////////// 
        //  user deposit()  //
        ////////////////////// 
        address user = makeAddr("user");
        vm.prank(address(0xdead));
        whitelist.add(user);
        deal(address(vault), user, accountFunds);

        vm.startPrank(user);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, user, 0);
        uint256 tokenIdForUser = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT));
        vm.stopPrank();

        ////////////////////////////////////////////
        // anotherExternalUser createRedemption() //
        ////////////////////////////////////////////
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        ///////////////////////////////////////////
        // anotherExternalUser claimRedemption() //
        ///////////////////////////////////////////
        vm.prank(anotherExternalUser);
        transmuterLogic.claimRedemption(1);
    


        (uint256 collateralfor0xBeef, ,) = alchemist.getCDP(tokenIdFor0xBeef);
        (uint256 collateralforUser, ,) = alchemist.getCDP(tokenIdForUser);
        assertEq(collateralfor0xBeef, 1);
        
        // There is not enough balance in the contract ❌
        assertEq(collateralforUser, depositAmount);
        assertEq(alchemist.getTotalUnderlyingValue(), 175342857142857142781821);
        vm.prank(user);
        vm.expectRevert(); // User cannot withdraw correctly ❌
        alchemist.withdraw(depositAmount, user, tokenIdForUser);

    }
```
