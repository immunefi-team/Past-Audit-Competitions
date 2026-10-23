# 58274 sc high liquidation fee logic in doliquidation strands liquidator rewards when balance is exhausted freezing funds

**Submitted on Oct 31st 2025 at 22:00:27 UTC by @Codexstar for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58274
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

`AlchemistV3._doLiquidation` subtracts the entire liquidation amount—including the liquidator fee—from a borrower’s collateral balance before attempting to transfer the fee. If the seizure zeroes out the balance, the subsequent `>= feeInYield` check fails and the fee is never paid out. Every full-collateral liquidation therefore strands the liquidator reward inside the contract, freezing protocol funds and eroding liquidation incentives.

### Vulnerability Details

* `calculateLiquidation` returns `grossCollateralToSeize = debtToBurn + fee` (see `src/AlchemistV3.sol:1244-1290`). `_doLiquidation` converts this to yield tokens as `amountLiquidated`, while `feeInYield` is calculated from the same `baseFee` (`src/AlchemistV3.sol:858-868`).
* The borrower’s collateral balance is immediately reduced by the full `amountLiquidated`:

  ```solidity
  account.collateralBalance = account.collateralBalance > amountLiquidated
      ? account.collateralBalance - amountLiquidated
      : 0;
  ```

  `src/AlchemistV3.sol:867-872`
* Only after the balance update does the contract try to pay the liquidator:

  ```solidity
  if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
      TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
  }
  ```

  `src/AlchemistV3.sol:877-879`
* Because `amountLiquidated` already includes the fee, any liquidation that seizes the full collateral sets the balance to zero, causing `account.collateralBalance >= feeInYield` to fail. The fee remains trapped inside the Alchemist contract. No other code path compensates the liquidator or removes the frozen funds.

### Impact Details

* Liquidators lose their fee in common scenarios (e.g., badly undercollateralized accounts). The stuck reward removes the incentive to liquidate, threatening protocol solvency.
* The trapped MYT is permanently frozen—there is no redemption mechanism once the transfer fails.
* Example: collateral balance 100 MYT, `amountLiquidated = 100`, `feeInYield = 5`. After the balance is zeroed, the fee transfer condition fails, and 5 MYT remains stranded in the contract.

### References

* Balance update that consumes the entire `amountLiquidated`: `src/AlchemistV3.sol:867-872`
* Fee transfer guarded by depleted balance check: `src/AlchemistV3.sol:877-879`
* Fee included in `grossCollateralToSeize`: `src/AlchemistV3.sol:1244-1290`

## Proof of Concept

## Proof of Concept

1. Open a borrower position with 100 MYT collateral and enough debt for liquidation to seize the full balance (e.g., collateralization well below the lower bound with a 5% liquidator fee).
2. Call `alchemist.liquidate(accountId)`.
3. Observe post-call state:
   * `account.collateralBalance` is set to `0` because `amountLiquidated` equals the full collateral.
   * The transmuter receives `amountLiquidated - feeInYield` (e.g., 95 MYT).
   * The liquidator receives nothing; `account.collateralBalance >= feeInYield` is false.
4. Query `IERC20(myt).balanceOf(address(alchemist))` to confirm the 5 MYT fee remains inside the contract with no exit path, demonstrating permanent fund freezing.

### Runnable Foundry Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {IMockYieldToken} from "../mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "../libraries/AlchemistNFTHelper.sol";
import {SafeERC20} from "../../libraries/SafeERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract LiquidatorFeePoC is AlchemistV3Test {
    using SafeERC20 for IERC20;

    uint256 private constant TARGET_LIQUIDATOR_FEE_BPS = 9_500;
    uint256 private constant BORROW_BPS = 7_000;
    uint256 private constant SUPPLY_BPS = 6_500;

    function test_poc_liquidator_fee_not_paid() external {
        (uint256 feeYield, uint256 baseFeeDebt, uint256 liquidatorDelta, uint256 collateralRemaining) =
            _executeScenario(BORROW_BPS, SUPPLY_BPS);

        emit log_named_uint("Borrow BPS", BORROW_BPS);
        emit log_named_uint("Supply BPS", SUPPLY_BPS);
        emit log_named_uint("Base fee (debt)", baseFeeDebt);
        emit log_named_uint("Fee owed (yield)", feeYield);
        emit log_named_uint("Liquidator received (yield)", liquidatorDelta);
        emit log_named_uint("Collateral remaining (shares)", collateralRemaining);

        assertEq(liquidatorDelta, 0, "fee unexpectedly paid");
        assertGt(feeYield, 0, "fee should be positive");
        assertEq(collateralRemaining, 0, "collateral should be fully seized");
    }

    function _executeScenario(uint256 borrowBps, uint256 supplyFactorBps)
        internal
        returns (uint256 feeInYield, uint256 baseFeeDebt, uint256 liquidatorDelta, uint256 collateralRemaining)
    {
        (uint256 tokenId, uint256 baseFeeDebtLocal, uint256 plannedLiquidation) =
            _preparePosition(borrowBps, supplyFactorBps);

        (uint256 collateralShares,,) = alchemist.getCDP(tokenId);
        require(plannedLiquidation <= collateralShares, "invalid liquidation plan");

        uint256 balanceBefore = IERC20(address(vault)).balanceOf(externalUser);
        vm.startPrank(externalUser);
        (, uint256 feeYield,) = alchemist.liquidate(tokenId);
        vm.stopPrank();
        uint256 balanceAfter = IERC20(address(vault)).balanceOf(externalUser);
        (, , uint256 collateralLeft) = alchemist.getCDP(tokenId);

        return (feeYield, baseFeeDebtLocal, balanceAfter - balanceBefore, collateralLeft);
    }

    function _preparePosition(uint256 borrowBps, uint256 supplyFactorBps)
        internal
        returns (uint256 tokenId, uint256 baseFeeDebt, uint256 amountLiquidated)
    {
        address borrower = yetAnotherExternalUser;
        address helper = anotherExternalUser;

        deal(address(vault), borrower, depositAmount * 2000);
        deal(address(vault), helper, depositAmount * 2000);

        vm.startPrank(borrower);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(depositAmount, borrower, 0);
        tokenId = AlchemistNFTHelper.getFirstTokenId(borrower, address(alchemistNFT));
        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        uint256 borrowAmount = (maxBorrow * borrowBps) / 10_000;
        alchemist.mint(tokenId, borrowAmount, borrower);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount / 10_000);
        alchemist.deposit(depositAmount / 10_000, borrower, tokenId);
        vm.stopPrank();

        vm.startPrank(helper);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 500);
        alchemist.deposit(depositAmount * 500, helper, 0);
        vm.stopPrank();

        vm.startPrank(alOwner);
        alchemist.setGlobalMinimumCollateralization(alchemist.minimumCollateralization());
        alchemist.setLiquidatorFee(TARGET_LIQUIDATOR_FEE_BPS);
        vm.stopPrank();

        uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);

        uint256 strategyBalance = IERC20(mockVaultCollateral).balanceOf(mockStrategyYieldToken);
        uint256 cappedBps = supplyFactorBps > 10_000 ? 10_000 : supplyFactorBps;
        uint256 targetBalance = (strategyBalance * cappedBps) / 10_000;
        if (targetBalance < strategyBalance) {
            IMockYieldToken(mockStrategyYieldToken).siphon(strategyBalance - targetBalance);
        }

        (uint256 collateralShares, uint256 debt,) = alchemist.getCDP(tokenId);

        uint256 globalCollat =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * 1e18 / alchemist.totalDebt();

        (uint256 liquidationAmount,, uint256 baseFeeDebtLocal,) = alchemist.calculateLiquidation(
            alchemist.convertYieldTokensToUnderlying(collateralShares),
            debt,
            alchemist.minimumCollateralization(),
            globalCollat,
            alchemist.globalMinimumCollateralization(),
            alchemist.liquidatorFee()
        );

        baseFeeDebt = baseFeeDebtLocal;
        amountLiquidated = alchemist.convertDebtTokensToYield(liquidationAmount);
    }
}
```

Run with:

```bash
forge test --match-test test_poc_liquidator_fee_not_paid -vv
```
