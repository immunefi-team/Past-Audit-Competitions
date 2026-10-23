# 56855 sc medium liquidations fail with arithmetic underflow when forced repayment exhausts collateral

**Submitted on Oct 21st 2025 at 09:21:44 UTC by @InAllHonesty for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56855
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `_forceRepay()` function sends the account's entire collateral balance to the transmuter during liquidation, but the calling function `_liquidate()` then attempts to pay a repayment fee to the liquidator from that same exhausted balance. This causes an arithmetic underflow in the ERC20 transfer, reverting the entire liquidation transaction. Positions where the collateral value approximately equals the earmarked debt cannot be liquidated, leaving bad debt stuck in the protocol.

## Vulnerability Details

Conditions for trigger:

* Account has earmarked debt
* Collateral value approximately equals the earmarked debt
* Liquidation attempts forced repayment
* No excess collateral remains for fees

When liquidating an undercollateralized position with earmarked debt, the protocol calls `_forceRepay()` to repay the debt using the account's collateral. The function calculates a protocol fee but fails to reserve it:

```
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // [...]
    
    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);

    // Repay earmarked debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    // Cap to available collateral
    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    // @audit-issue This check fails when collateralBalance is now 0
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    // @audit-issue Sends FULL creditToYield to transmuter
    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
    }
    return creditToYield;
```

After `_forceRepay()` returns, if the debt was fully repaid, `_liquidate()` calls `_resolveRepaymentFee()` to compensate the liquidator:

```
function _liquidate(uint256 accountId) internal returns (...) {
    // [...]
    
    uint256 repaidAmountInYield = 0;
    if (account.earmarked > 0) {
        repaidAmountInYield = _forceRepay(accountId, account.earmarked);
    }
    
    if (account.debt == 0) {
        feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
        // @audit-issue Tries to transfer fee, but contract balance is 0
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        return (repaidAmountInYield, feeInYield, 0);
    }
    // [...]
}
```

The `_resolveRepaymentFee()` function calculates the fee and attempts to transfer it:

```
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;
    // This correctly handles the accounting (ternary prevents negative balance)
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;
}
```

The issue is that the Alchemist contract's actual token balance is zero after sending everything to the transmuter. When `TokenUtils.safeTransfer()` tries to send the repayment fee to the liquidator, the underlying ERC20 transfer underflows and reverts.

## Impact Details

Positions where `collateralBalance ≈ earmarkedDebt` (in yield token terms) cannot be liquidated. The transaction reverts with an arithmetic underflow before the liquidation completes. This leaves bad debt positions stuck in the protocol.

Even in cases where enough collateral exists to avoid the underflow, the protocol still loses revenue because the fee validation check `if (account.collateralBalance > protocolFeeTotal)` fails after the balance is set to zero. The event emits a `protocolFeeTotal` value that was never actually collected.

## References

`_forceRepay` - <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L782> `_liquidate` - <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L791-L843> `_resolveRepaymentFee` - <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900-L907>

## Proof of Concept

## Proof of Concept

Paste the following at the end of `AlchemistV3.t.sol`:

```solidity
    function testForceRepayUnderflowAndFeeLoss() external {
        // Set protocol fee 2% - repayment fee is already at 1% 
        vm.prank(alOwner);
        alchemist.setProtocolFee(200);

        uint256 depositAmount = 100e18;
        
        // User deposits and borrows
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, 50e18, address(0xbeef));
        vm.stopPrank();
        
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
        console.log("Initial collateral: %e", collateralBefore);
        console.log("Initial debt: %e", debtBefore);
        console.log("Initial earmarked: %e", earmarkedBefore);

        // Create transmuter position to earmark debt
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();
        
        // Fast forward to fully earmark the debt
        vm.roll(block.number + 5_256_000);
        
        (uint256 collateralAfterEarmark, uint256 debtAfterEarmark, uint256 earmarkedAfterEarmark) = alchemist.getCDP(tokenId);
        console.log("\n=== AFTER EARMARKING ===");
        console.log("Collateral: %e", collateralAfterEarmark);
        console.log("Debt: %e", debtAfterEarmark);
        console.log("Earmarked: %e", earmarkedAfterEarmark);

        // Record balances before liquidation
        uint256 alchemistBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 protocolBalanceBefore = IERC20(address(vault)).balanceOf(protocolFeeReceiver);
        uint256 transmuterBalanceBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));
        
        console.log("\n=== BALANCES BEFORE LIQUIDATION ===");
        console.log("Alchemist contract balance: %e", alchemistBalanceBefore);
        console.log("Protocol receiver balance:", protocolBalanceBefore);
        console.log("Transmuter balance:", transmuterBalanceBefore);

        // Manipulate price to make position undercollateralized
        // Double the supply to halve the share price (collateral worth ~50% of original)
        uint256 supply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(supply * 2);
        
        (uint256 collateralAfterPrice, uint256 debtAfterPrice, uint256 earmarkedAfterPrice) = alchemist.getCDP(tokenId);
        uint256 collateralValueInDebt = alchemist.convertYieldTokensToDebt(collateralAfterPrice);
        
        console.log("\n=== AFTER PRICE MANIPULATION ===");
        console.log("Collateral (yield tokens): %e", collateralAfterPrice);
        console.log("Collateral value (debt tokens): %e", collateralValueInDebt);
        console.log("Debt: %e", debtAfterPrice);
        console.log("Earmarked: %e", earmarkedAfterPrice);

        // Calculate what should happen
        uint256 repaymentAmountInYield = alchemist.convertDebtTokensToYield(earmarkedAfterPrice);
        uint256 expectedProtocolFee = repaymentAmountInYield * 200 / 10_000; // 2%
        uint256 expectedRepaymentFee = repaymentAmountInYield * 100 / 10_000; // 1%
        uint256 totalFeesNeeded = expectedProtocolFee + expectedRepaymentFee;
        
        console.log("\n=== EXPECTED FEE CALCULATIONS ===");
        console.log("Repayment amount (yield): %e", repaymentAmountInYield);
        console.log("Expected protocol fee (2%): %e", expectedProtocolFee);
        console.log("Expected repayment fee (1%): %e", expectedRepaymentFee);
        console.log("Total fees needed: %e", totalFeesNeeded);
        console.log("Available collateral: %e", collateralAfterPrice);
        console.log("Shortfall:", totalFeesNeeded > collateralAfterPrice ? totalFeesNeeded - collateralAfterPrice : 0);
        
        // Attempt liquidation - should revert with underflow
        vm.startPrank(externalUser);
        
        vm.expectRevert(); // Expect the transaction to revert
        alchemist.liquidate(tokenId);
        
        vm.stopPrank();
        console.log("Liquidation reverted as expected!");

    }
```

Run using `forge test --mt testForceRepayUnderflowAndFeeLoss -vvvv`
