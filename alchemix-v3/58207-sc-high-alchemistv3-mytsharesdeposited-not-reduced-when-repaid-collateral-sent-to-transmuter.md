# 58207 sc high alchemistv3 mytsharesdeposited not reduced when repaid collateral sent to transmuter

## #58207 \[SC-High] AlchemistV3 \_mytSharesDeposited Not Reduced When Repaid Collateral Sent to Transmuter

**Submitted on Oct 31st 2025 at 11:13:50 UTC by @Brainiac5 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58207
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Bug Report: AlchemistV3 \_mytSharesDeposited Not Reduced When Repaid Collateral Sent to Transmuter

### Severity

**HIGH**

### Summary

The `AlchemistV3._forceRepay()` function transfers repaid collateral (MYT tokens) to the transmuter but does not decrement `_mytSharesDeposited`. This creates a critical accounting mismatch where `_mytSharesDeposited` shows inflated values compared to actual token balances, incorrectly capping deposits and masking the true protocol state.

### Vulnerable Code

#### Force Repay Transfers Collateral Without Accounting Update

**File**: `src/AlchemistV3.sol`

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    if (amount == 0) {
        return 0;
    }
    _checkForValidAccountId(accountId);
    Account storage account = _accounts[accountId];

    _earmark();
    _sync(accountId);

    uint256 debt;
    _checkState((debt = account.debt) > 0);

    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);

    // Repay earmarked debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    // BUG: Transfers repaid collateral to transmuter but does NOT decrement _mytSharesDeposited
    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        // Missing: _mytSharesDeposited -= creditToYield;
    }
    
    return creditToYield;
}
```

**File**: `src/AlchemistV3.sol`

```solidity
// Correctly incremented on deposit
function deposit(uint256 amount, address recipient) external {
    // ...
    _mytSharesDeposited += amount;
}

// Correctly decremented on withdraw
function withdraw(uint256 amount, address recipient) external {
    // ...
    _mytSharesDeposited -= amount;
}

// BUT: _forceRepay() sends MYT out without decrementing!
```

### Vulnerability Details

#### The Accounting Flow

**When users deposit MYT:**

```solidity
deposit(100e18 MYT)
> _mytSharesDeposited += 100e18
> Contract MYT balance += 100e18
> Both values match 
```

**When force liquidation repays debt:**

```solidity
_forceRepay(accountId, earmarkedAmount)
> creditToYield = 90e18 (converted debt to yield tokens)
> TokenUtils.safeTransfer(myt, address(transmuter), 90e18)
> Contract MYT balance -= 90e18 
> _mytSharesDeposited unchanged (should be -= 90e18)
```

### Impact

#### Severity Justification

**1. Deposit Cap Becomes inconsistent with the true system state**

```solidity
function deposit(uint256 amount, address recipient) external {
    // Uses inflated _mytSharesDeposited for check
    _checkState(_mytSharesDeposited + amount <= depositCap);
    // ...
}
```

* If `depositCap = 1000e18`
* Real deposits = 800e18 (should allow 200e18 more)
* But `_mytSharesDeposited` shows 950e18 due to untracked transfers out
* Result: Only 50e18 deposits allowed instead of 200e18

### Recommended Fix

#### Decrement \_mytSharesDeposited When Sending to Transmuter

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    if (amount == 0) {
        return 0;
    }
    _checkForValidAccountId(accountId);
    Account storage account = _accounts[accountId];

    _earmark();
    _sync(accountId);

    uint256 debt;
    _checkState((debt = account.debt) > 0);

    uint256 credit = amount > debt ? debt : amount;
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);

    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
    }

    if (creditToYield > 0) {
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        
        // FIX: Decrement _mytSharesDeposited to match actual outflow
        _mytSharesDeposited -= creditToYield;
    }
    
    return creditToYield;
}
```

### Proof of Concept

### Proof of Concept

**Test File**: (ADD TO) `src/test/AlchemistV3.t.sol`

Run with:

```bash
forge test --match-test testForceRepayAccountingBugSimple -vv
```

```solidity
    function testForceRepayAccountingBugSimple() external {
        // Setup: Create a position and earmark debt through transmuter
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), 200e18 + 100e18);
        alchemist.deposit(200e18, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, 180e18, address(0xbeef));
        vm.stopPrank();

        // Create redemption to earmark debt  
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 180e18);
        transmuterLogic.createRedemption(180e18);
        vm.stopPrank();

        // Skip to full maturation of redemption
        vm.roll(block.number + 5_256_000);

        // Manipulate price to trigger undercollateralization and force repay
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Liquidate - this triggers _forceRepay() which has the accounting bug
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenId);
        vm.stopPrank();

        // Complete the redemption process
        vm.roll(block.number + 5_256_000);
        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        // NOW DRAIN ALL LIQUIDITY: 0xbeef withdraws remaining collateral
        vm.startPrank(address(0xbeef));
        (uint256 remainingCollateral, uint256 remainingDebt,) = alchemist.getCDP(tokenId);
        console.log("0xbeef remaining collateral before withdraw:", remainingCollateral);
        console.log("0xbeef remaining debt before withdraw:", remainingDebt);
        
        // If there's remaining debt, repay it first
        if (remainingDebt > 0) {
            // Convert debt to yield tokens and repay
            uint256 debtInYield = alchemist.convertDebtTokensToYield(remainingDebt);
            if (vault.balanceOf(address(0xbeef)) >= debtInYield) {
                alchemist.repay(debtInYield, tokenId);
            }
        }
        
        // Withdraw all remaining collateral
        (remainingCollateral, remainingDebt,) = alchemist.getCDP(tokenId);
        if (remainingCollateral > 0) {
            alchemist.withdraw(remainingCollateral, address(0xbeef), tokenId);
        }
        vm.stopPrank();

        // Verify ALL liquidity is drained from the system
        uint256 totalSystemDebt = alchemist.totalDebt();
        uint256 totalSystemCollateral = alchemist.getTotalDeposited();
        
        console.log("=== SYSTEM STATE AFTER ALL WITHDRAWALS ===");
        console.log("Total system debt:", totalSystemDebt);
        console.log("Total system collateral:", totalSystemCollateral);
        
        // BUG DEMONSTRATION: Even after ALL parties have withdrawn everything,
        // _mytSharesDeposited still shows phantom tokens that don't exist
        uint256 accountingBalance = alchemist._mytSharesDeposited();
        uint256 realBalance = IERC20(address(vault)).balanceOf(address(alchemist)) + IERC20(address(vault)).balanceOf(address(transmuterLogic));
        
        console.log("=== FORCE REPAY ACCOUNTING BUG ===");
        console.log("_mytSharesDeposited (accounting):", accountingBalance);
        console.log("Actual vault balance (reality):", realBalance);
        console.log("Phantom tokens:", accountingBalance > realBalance ? accountingBalance - realBalance : 0);
        
        // The bug: _mytSharesDeposited tracks more tokens than actually exist
        // This happens because _forceRepay() transfers tokens out but doesn't update _mytSharesDeposited
        // Even with ZERO system debt and collateral, _mytSharesDeposited > 0
        assertGt(accountingBalance, 0, "BUG: _mytSharesDeposited should be 0 when no liquidity remains, but force repay leaves phantom tokens");
        assertEq(realBalance, 0, "Vault should be empty after all withdrawals");
    }
```
