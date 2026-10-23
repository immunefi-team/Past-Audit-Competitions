# 57036 sc high unconditional debt reduction before protocol fee check in force repayment&#x20;

**Submitted on Oct 22nd 2025 at 21:41:18 UTC by @w3llyc4de20Ik2nn1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57036
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The fee transfer is conditional, but the debt repayment is not. This creates an inconsistency where debt is reduced without the protocol fee being collected.

## Vulnerability Details

```solidity

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) { 
   if (amount == 0) { 
       return 0; 
   } 
   _checkForValidAccountId(accountId); 
   Account storage account = _accounts[accountId]; 
 
   // Query transmuter and earmark global debt 
   _earmark(); 
 
   // Sync current user debt before deciding how much is available to be repaid 
   _sync(accountId); 
 
   uint256 debt; 
 
   // Burning yieldTokens will pay off all types of debt 
   _checkState((debt = account.debt) > 0); 
 
   uint256 credit = amount > debt ? debt : amount; 
   uint256 creditToYield = convertDebtTokensToYield(credit); 
 @>  _subDebt(accountId, credit);  // Debt reduced here unconditionally 
 
   // Repay debt from earmarked amount of debt first 
   uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit; 
   account.earmarked -= earmarkToRemove; 
 
   creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield; 
 @>  account.collateralBalance -= creditToYield; // <-- Collateral deducted for repayment 

 
 
   @>  uint256 protocolFeeTotal = creditToYield * protocolFee / BPS; // <-- Fee computed on full creditToYield 

  
 
   emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal); 
 
  @> if (account.collateralBalance > protocolFeeTotal) { 
       account.collateralBalance -= protocolFeeTotal; // <-- Fee conditional on *remaining* collateral 

 
       // Transfer the protocol fee to the protocol fee receiver 
       TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); 
   } 
 
   if (creditToYield > 0) { 
       // Transfer the repaid tokens from the account to the transmuter. 
       TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); 
   } 
   return creditToYield; 
} 
```

The \_subDebt(accountId, credit) call reduces the account's debt unconditionally early in the function, but the subsequent protocol fee calculation and transfer are conditional on account.collateralBalance > protocolFeeTotal; when the fee cannot be paid due to insufficient remaining collateral, it is silently skipped without adjusting the already-reduced debt.

## Impact Details

This allows debt to be force-repaid without collecting the intended protocol fee, resulting in uncollected revenue for the protocol during liquidations involving earmarked debt.

## Soln

To fix the uncollected protocol fee issue in \_forceRepay, compute and deduct the protocol fee from collateral before reducing debt (reversing the order), clamp the fee to available collateral, and only proceed with debt reduction and transfers if the full repayment amount (including fee) can be covered, ensuring fees are always collected or the repayment is proportionally adjusted.

```solidity

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) { 
   if (amount == 0) { 
       return 0; 
   } 
   _checkForValidAccountId(accountId); 
   Account storage account = _accounts[accountId]; 
 
   // Query transmuter and earmark global debt 
   _earmark(); 
 
   // Sync current user debt before deciding how much is available to be repaid 
   _sync(accountId); 
 
   uint256 debt; 
 
   // Burning yieldTokens will pay off all types of debt 
   _checkState((debt = account.debt) > 0); 
 
   uint256 credit = amount > debt ? debt : amount; 
   uint256 creditToYield = convertDebtTokensToYield(credit); 
 
   // Repay debt from earmarked amount of debt first 
   uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit; 
   account.earmarked -= earmarkToRemove; 
 
   // Clamp repayment to available collateral 
   creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield; 
   account.collateralBalance -= creditToYield; 
 
   // Compute and clamp protocol fee before debt reduction 
   uint256 protocolFeeTotal = creditToYield * protocolFee / BPS; 
   if (protocolFeeTotal > account.collateralBalance) { 
       // Adjust repayment proportionally if fee can't be fully covered 
       uint256 totalRequired = creditToYield + protocolFeeTotal; 
       uint256 available = creditToYield + account.collateralBalance;  // Repayment + remaining after repayment 
       creditToYield = totalRequired > available ? (credit * available / totalRequired) : creditToYield; 
       credit = convertYieldTokensToDebt(creditToYield);  // Adjust credit proportionally 
       protocolFeeTotal = creditToYield * protocolFee / BPS;  // Recalculate fee on adjusted amount 
   } 
 
   // Now reduce debt after ensuring fee coverage 
   _subDebt(accountId, credit); 
 
   emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal); 
 
   // Transfer protocol fee (now guaranteed) 
   account.collateralBalance -= protocolFeeTotal; 
   TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); 
   _mytSharesDeposited -= protocolFeeTotal;  // Also fix the TVL tracking bug here 
 
   if (creditToYield > 0) { 
       // Transfer the repaid tokens from the account to the transmuter. 
       TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); 
       _mytSharesDeposited -= creditToYield;  // Fix TVL tracking bug 
   } 
   return creditToYield; 
} 
```

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738>

## Proof of Concept

## Proof of Concept

Put the test in the src/test/AlchemistV3.t.sol

Run the test with: forge test --match-test testForceRepaySkipsProtocolFeeWhenInsufficientCollateral -vvvv

```solidity
function testForceRepaySkipsProtocolFeeWhenInsufficientCollateral() external { 

    uint256 protocolFeeBPS = 2000; // 20% 

    uint256 depositAmount = 100e18; 

    vm.startPrank(alOwner); 

    alchemist.setProtocolFee(protocolFeeBPS); 

    vm.stopPrank(); 

 

    address user = address(0xbeef); 

    uint256 tokenId; 

    vm.startPrank(user); 

    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount); 

    alchemist.deposit(depositAmount, user, 0); 

    tokenId = AlchemistNFTHelper.getFirstTokenId(user, address(alchemistNFT)); 

    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId); 

    alchemist.mint(tokenId, maxBorrow, user); 

    vm.stopPrank(); 

 

    uint256 redemptionAmount = maxBorrow; 

    vm.startPrank(anotherExternalUser); 

    deal(address(alToken), anotherExternalUser, redemptionAmount); 

    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount); 

    transmuterLogic.createRedemption(redemptionAmount); 

    vm.stopPrank(); 

 

    vm.roll(block.number + 5_256_000); 

 

    (, uint256 preDebt, ) = alchemist.getCDP(tokenId); 

    assertEq(preDebt, maxBorrow); 

 

    uint256 protocolBefore = IERC20(address(vault)).balanceOf(alchemist.protocolFeeReceiver()); 

 

    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply(); 

    uint256 modifiedSupply = initialSupply * 560 / 10_000 + initialSupply; 

    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedSupply); 

 

    vm.startPrank(externalUser); 

    (uint256 liqYield, uint256 liqFeeYield, uint256 liqFeeUnderlying) = alchemist.liquidate(tokenId); 

    vm.stopPrank(); 

 

    assertGt(liqYield, 0); 

 

    uint256 protocolAfter = IERC20(address(vault)).balanceOf(alchemist.protocolFeeReceiver()); 

    assertEq(protocolAfter, protocolBefore); 

 

    (, uint256 postDebt, ) = alchemist.getCDP(tokenId); 

    assertEq(postDebt, 0); 

} 

```

Proof from Execution Trace

Setup confirms earmarked position at risk: User deposits 100e18 yield shares, mints \~90e18 debt (max borrowable under \~1.11 collateralization). Full redemption created/matured for the debt amount, fully earmarking it (preDebt = 90e18).

Price drop triggers liquidation via forceRepay: \~5.6% yield token devaluation makes position undercollateralized (collateral value drops to \~95.04e18 underlying). Liquidation calls \_forceRepay on full earmarked debt.

Unconditional debt reduction: \_subDebt clears all 90e18 debt (postDebt = 0), as expected.

Conditional fee skip: protocolFeeTotal computed as \~19.008e18 (20% of 95.04e18 creditToYield). But post-repayment collateral (\~4.009e18 remaining) < fee, so the if (account.collateralBalance > protocolFeeTotal) branch skips deduction/transfer. Protocol receiver balance unchanged (0 before/after).

Impact evident: Full debt forgiven (revenue loss to protocol avoided), but no fee collected despite 20% rate. Liquidator gets only \~0.95e18 repayment fee (1% of repaid yield), not protocol fee.

This directly demonstrates the inconsistency: debt reduced unconditionally, fee applied conditionally—skipping revenue in edge cases (devaluation during earmark) where collateral value < (repaid yield + fee). No manipulation; interacts with live setup via standard flows (deposit/mint/redemption/price update/liquidate).
