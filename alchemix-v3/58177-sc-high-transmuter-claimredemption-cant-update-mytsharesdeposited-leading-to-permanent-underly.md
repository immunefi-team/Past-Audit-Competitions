# 58177 sc high transmuter claimredemption cant update mytsharesdeposited leading to permanent underlying value state inside alchemist

**Submitted on Oct 31st 2025 at 06:34:16 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58177
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

the function `Transmuter::claimRedemption` only update `_mytSharesDeposited` if it invokes AlchemistV3 `redeem` function. but the problem is the MYT that is already inside the `Transmuter` contract beforehand that also get sent into the user claiming the redemption would not deducting the MYT share state. Effectively this would inflate the underlying value state that is used by various function, which in return would be inaccurate.

## Vulnerability Details

it is important to understand that, the MYT sent into `Transmuter` contract because liquidation SHOULD be still counted toward `_mytSharesDeposited` which translate into whole `AlchemistV3::getTotalUnderlyingValue` this is an intended behavior.

that’s why when we check `_doLiquidation` when the user collateral is sent to `Transmuter`, there are no reduction in `_mytSharesDeposited` . because this still count toward protocol underlying value. this SHOULD only reduced when the MYT is sent out from Transmuter contract as part of claim redemption and when the alAsset redeemed is burned.

the issue lies in the `Transmuter::claimRedemption` , because this is when the MYT is transferred out from the protocol. When matured MYT would get sent into the user, it would invoke `redeem` function if the balance of `Transmuter` is not enough. However, the current `Transmuter` balance that is get sent other than via `redeem` would not reduced the `_mytSharesDeposited`.

Ultimately, the protocol would left with inflated `_mytSharesDeposited` even if the MYT already get sent out to user for redemption.

```solidity

@>      TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        // Burn remaining synths that were not returned
        TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));
```

the snippet of `claimRedemption` above is calculating how much should be sent into msg.sender. the `claimYield` contains two type of MYT token: 1) the just redeemed from `AlchemistV3::redeem` function and, 2) the holding balance of MYT token via liquidation and in some cases repayment.

there are no `_mytSharesDeposited` update as we can see inside the `claimRedemption` function itself.

worth to note that while repay flow also sent MYT into transmuter, it is not necessary to reduce the MYT shares because the original collateral inside AlchemistV3 still belong to the borrower who just repaid their debt. they can withdraw it if they want. so it is safe to say that this claim redemption issue would occur only from MYT that is gent sent to transmuter because of liquidation.

## Impact Details

1. the core issue is that the `_mytSharesDeposited` variable becomes artificially inflated. this single bad value cascades, causing the protocol to miscalculate its true health factor.
2. `Transmuter` relies on this inflated value to check for bad debt. When the market price of MYT drops, the Transmuter will fail to detect the problem and will not see that the AlchemistV3 is undercollateralized.
3. because the `Transmuter` is blind to the bad debt, it will continue to process redemption claims at a 1:1 ratio. This allows users to redeem assets at full value, effectively draining the protocol of its remaining healthy collateral.
4. this inflated value also breaks other critical functions like liquidation and deposit caps
5. most severe impact is protocol insolvency. the combination of failing to detect bad debt, allowing unsafe redemptions, and failing to liquidate bad positions will drain the protocol of all its value, leading to a complete loss for users.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L250-L266>

## Proof of Concept

## Proof of Concept

add this to `src/test/AlchemistV3.t.sol`

```solidity
    function testLiquidate_getTotalUnderlyingInflated() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // create redemption
        vm.startPrank(address(0xbeef));
        uint256 redemptionAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount);
        transmuterLogic.createRedemption(redemptionAmount);
        vm.stopPrank();

        // maturing the redemption
        vm.roll(block.number + 5_256_000);

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // snapshot alchemist MYT balance before liquidation
        uint256 alchemistUnderlyingValueBeforeLiquidation = alchemist.getTotalUnderlyingValue();

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        assertEq(debt, 0);
        vm.stopPrank();

        // get that the underlying value after liquidation
        uint256 alchemistUnderlyingValueAfterLiquidation = alchemist.getTotalUnderlyingValue();
        // it should not changed after liquidation
        assertEq(alchemistUnderlyingValueAfterLiquidation, alchemistUnderlyingValueBeforeLiquidation);

        // now we redeem the matured redemption
        vm.startPrank(address(0xbeef));
        transmuterLogic.claimRedemption(1);

        // now that all previous alAsset is burned and the MYT get sent to 0xbeef
        // lets check the issue, the getTotalUnderlyingValue would not changed even after the redemption
        uint256 alchemistUnderlyingValueAfterRedemption = alchemist.getTotalUnderlyingValue();
        
        // we check underlying value before liquidation is the same as after redemption, where the MYT sent out into user
        // if this pass, we confirmed the issue
        assertEq(alchemistUnderlyingValueBeforeLiquidation, alchemistUnderlyingValueAfterRedemption);
    }

```

the test would pass, showing that the alchemist underlying value is constant before the liquidation happen, after liquidation happen and after the redemption where the MYT is getting out of protocol.
