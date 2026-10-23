# 58129 sc high missing mytsharesdeposited update in forcerepay causes accounting inconsistency which can dos deposit and liquidation

**Submitted on Oct 30th 2025 at 20:37:30 UTC by @gizzy for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58129
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

The `_forceRepay()` function collects and transfers protocol fees and creditToYield but fails to update `_mytSharesDeposited`, unlike `repay()` and `burn()` which properly decrement this accounting variable. This creates a growing discrepancy between actual token balances and recorded deposits, breaking a critical system invariant.

## Vulnerability Details

### The Issue

`_mytSharesDeposited` tracks the total yield tokens deposited in the Alchemist and is used for critical calculations like deposit caps and total value locked (TVL). When protocol fees are collected, tokens leave the contract and must be deducted from this counter.

**Correct implementation in `repay()`:**

```solidity
// Debt is subject to protocol fee similar to redemptions
uint256 feeAmount = creditToYield * protocolFee / BPS;
if (feeAmount > account.collateralBalance) {
    revert("Not enough collateral to pay for debt fee");
} else {
    account.collateralBalance -= creditToYield * protocolFee / BPS;
}

_subDebt(recipientTokenId, credit);

// Transfer the repaid tokens to the transmuter.
TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;  // ✓ Correctly decremented
```

**Buggy implementation in `_forceRepay()`:**

```solidity
uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    // Transfer the protocol fee to the protocol fee receiver
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}

if (creditToYield > 0) {
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
}
return creditToYield;
// ✗ Missing: _mytSharesDeposited -= protocolFeeTotal;
```

### Attack Path

This happens normally from protocol usage.

1. User's position becomes undercollateralized (`collateralizationRatio <= collateralizationLowerBound`)
2. User's account has earmarked debt (`account.earmarked > 0`)
3. Anyone calls `liquidate(accountId)` on the user's position
4. `_liquidate()` calls `_forceRepay()` to repay the earmarked debt
5. Protocol fee and creditToYield transferred out, but `_mytSharesDeposited` is NOT decremented
6. Over time, `_mytSharesDeposited` becomes increasingly inflated

## Impact

### 1. **Broken Deposit Cap Enforcement**

```solidity:369:369:src/alchemistv3.sol
_checkState(_mytSharesDeposited + amount <= depositCap);
```

The inflated `_mytSharesDeposited` will cause deposits to hit the cap prematurely, blocking legitimate deposits even when the actual balance is lower.

### 2. **Incorrect TVL Calculations**

```solidity:1238:1241:src/alchemistv3.sol
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

This inflated TVL affects liquidation calculations, potentially preventing liquidations when the system is actually undercollateralized globally.

### 3. **Compounding Issue**

Each liquidation with earmarked debt increases the discrepancy. Over time with many liquidations, the accounting error grows unbounded.

Reference

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L771C8-L780C10>

## Recommended Mitigation

Update `_forceRepay()` to decrement `_mytSharesDeposited` consistently with `repay()` and `burn()`:

```diff
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
    _subDebt(accountId, credit);

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;

    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;

    uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

    emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
+       _mytSharesDeposited -= protocolFeeTotal;
    }

    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
+       _mytSharesDeposited -= creditToYield;
    }
    return creditToYield;
}
```

**Note:** Both `creditToYield` (sent to transmuter) AND `protocolFeeTotal` (sent to fee receiver) must be decremented since both amounts leave the contract.

## Proof of Concept

## Proof of Concept

Update the AlchemistV3.sol \_mytSharesDeposited to public for test to run

`uint256 public _mytSharesDeposited;`

Copy and paste in AlchemistV3.t

```solidity

    function testBug_ForceRepay_Missing_MytSharesDeposited_Update() external {
        uint256 amount = 200_000e18; // 200,000 yvdai

        // Set protocol fee to 10%
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee); // 100 = 1%

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // Create a healthy position to maintain global collateralization
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount * 2);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Create position for victim (0xbeef)
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);

        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Record _mytSharesDeposited before liquidation (via getTotalDeposited)
        uint256 mytSharesDepositedBefore = alchemist._mytSharesDeposited();
        uint256 actualBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("Before liquidation:");
        console.log("  _mytSharesDeposited:", mytSharesDepositedBefore);
        console.log("  Actual balance:     ", actualBalanceBefore);
        assertEq(mytSharesDepositedBefore, actualBalanceBefore, "Invariant should hold before liquidation");

        // Create transmuter position to enable earmarking
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // Fast forward 60% through transmutation period to earmark debt
        vm.roll(block.number + (5_256_000 * 60 / 100));

        // Verify debt is earmarked
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        assertEq(earmarked, prevDebt * 60 / 100, "60% should be earmarked");

        // Crash the price to make position undercollateralized
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Calculate expected protocol fee
        uint256 creditToYield = alchemist.convertDebtTokensToYield(earmarked);
        uint256 expectedProtocolFee = creditToYield * protocolFee / BPS;

        console.log("\nLiquidation parameters:");
        console.log("  Earmarked debt:       ", earmarked);
        console.log("  CreditToYield:        ", creditToYield);
        console.log("  Expected protocol fee:", expectedProtocolFee);

        // Liquidate the position
        vm.prank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        // Check state after liquidation
        uint256 mytSharesDepositedAfter = alchemist._mytSharesDeposited();
        uint256 actualBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));

        console.log("\nAfter liquidation:");
        console.log("  _mytSharesDeposited:", mytSharesDepositedAfter);
        console.log("  Actual balance:     ", actualBalanceAfter);

        // Calculate the discrepancy
        int256 discrepancy = int256(mytSharesDepositedAfter) - int256(actualBalanceAfter);
        console.log("  Discrepancy:        ", discrepancy > 0 ? uint256(discrepancy) : 0);
    }
```
