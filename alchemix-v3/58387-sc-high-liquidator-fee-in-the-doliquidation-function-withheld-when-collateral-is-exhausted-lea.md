# 58387 sc high liquidator fee in the doliquidation function withheld when collateral is exhausted leading to seized fee trapped in protocol

**Submitted on Nov 1st 2025 at 20:50:07 UTC by @Idealz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58387
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

In `_doLiquidation`, the liquidator fee is computed and included in the amount of collateral to seize. The code deducts the full seized amount from the user, transfers the net seized minus fee to the transmuter, and then tries to transfer the fee to the liquidator only if the victim’s remaining collateral balance is at least the fee. Because the victim’s balance was already reduced by the seized amount (including the fee), the post-seizure remaining balance is often smaller than the fee, so the conditional fails and the liquidator gets nothing. `_doLiquidation` removes the entire seized amount from the account before forwarding the fee. The protocol keeps the fee trapped, liquidators are disincentivized from acting, bad debt can grow unchecked, and trapped fees represent hidden liabilities

## Vulnerability Details

check code excerpt here <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L867-#L880>

```solidity
// compute amounts
amountLiquidated = convertDebtTokensToYield(liquidationAmount);
feeInYield = convertDebtTokensToYield(baseFee);

// update user balance and debt
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
_subDebt(accountId, debtToBurn);

// send liquidation amount - fee to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
// send base fee to liquidator if available
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

**Why this is wrong:**

* `calculateLiquidation(...)` returns `grossCollateralToSeize = debtToBurn + fee` (i.e., the fee is included in the seized collateral).
* The code reduces `account.collateralBalance` by `amountLiquidated` (the full seized amount, which includes the fee). After this reduction, the fee portion of the seized collateral is not represented in the account’s remaining balance.
* The conditional `account.collateralBalance >= feeInYield` asks whether the victim’s remaining balance is at least the fee. That is the wrong invariant: the correct check should not be against the victim’s post-seizure balance because the fee was already extracted as part of the seizure. In nearly all cases where a liquidation consumes most or all of the collateral, the post-seizure balance will be < fee and the fee transfer will be skipped, even though the fee was already taken from the user and sits in the contract.
* **Net effect:** fee is seized but not forwarded, it stays in contract balances as an unforwarded protocol-held token

## Impact Details

* **Permanent freezing of unclaimed yield:** fee amounts taken from user collateral remain in contract and are not forwarded → effectively frozen until manual recovery.
* **Protocol insolvency:** trapped fees accumulate as hidden protocol-held liabilities and reduce available funds; if many liquidations occur with trapped fees, the protocol’s ability to meet obligations deteriorates; under stress, this can cause insolvency

## References

* `_doLiquidation` (transfer & fee logic) — src: `AlchemistV3.sol` lines [852–895](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852-#L894)
* `calculateLiquidation` (fee included in `grossCollateralToSeize`)

## Proof of Concept

## Proof of Concept

Add this function test to the `AlchemistV3.t.sol` test contract

```solidity
function testLiquidatorFeeWithheldWhenCollateralDepleted_PoC() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Maintain global collateralization with a healthy auxiliary position
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Create the victim position
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrow, address(0xbeef));
        vm.stopPrank();

        // Increase liquidation fee significantly
        vm.prank(alOwner);
        alchemist.setLiquidatorFee(9500); // which is 95%

        // Search for the right degradation that triggers the bug
        uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
        
        bool scenarioFound = false;
        uint256 chosenBasisPoints = 0;

        // Iterate this to find exact threshold
        for (uint256 bps = 590; bps <= 1150; bps += 1) {
            uint256 modifiedSupply = initialSupply + (initialSupply * bps) / 10_000;
            IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedSupply);

            (uint256 collateral, uint256 debt,) = alchemist.getCDP(tokenId);
            if (collateral == 0) continue;

            uint256 collateralValue = alchemist.totalValue(tokenId);
            uint256 globalCollatRatio = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();

            (uint256 liquidationAmount,, uint256 baseFee, uint256 outsourcedFee) = alchemist.calculateLiquidation(
                collateralValue,
                debt,
                alchemist.minimumCollateralization(),
                globalCollatRatio,
                alchemist.globalMinimumCollateralization(),
                alchemist.liquidatorFee()
            );

            if (liquidationAmount == 0 || baseFee == 0 || outsourcedFee > 0) {
                continue;
            }

            uint256 amountLiquidatedShares = alchemist.convertDebtTokensToYield(liquidationAmount);
            uint256 feeShares = alchemist.convertDebtTokensToYield(baseFee);
            uint256 remaining = collateral > amountLiquidatedShares ? collateral - amountLiquidatedShares : 0;

            // This is the bug condition
            if (feeShares > 0 && remaining < feeShares) {
                scenarioFound = true;
                chosenBasisPoints = bps;
                break;
            }
        }
        
        if (!scenarioFound) {
        revert("Could not find bug scenario");
       }


        // Apply the chosen degradation
        uint256 chosenSupply = initialSupply + (initialSupply * chosenBasisPoints) / 10_000;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(chosenSupply);

        // Capture state before liquidation
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(externalUser);

        // Execute liquidation
        vm.prank(externalUser);
        (uint256 seizedShares, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);

        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(externalUser);

        // The BUG: fee is reported but liquidator didn't receive it
        assertGt(feeInYield, 0, "Fee should be non-zero");
        assertEq(feeInUnderlying, 0, "No outsourced fee");
        assertEq(
            liquidatorBalanceAfter,
            liquidatorBalanceBefore,
            "BUG CONFIRMED: Liquidator received nothing despite non-zero fee"
        );
    }
```

Run the test using `forge test --match-test testLiquidatorFeeWithheldWhenCollateralDepleted_PoC -vvvv`
