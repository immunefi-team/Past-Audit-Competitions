# 57883 sc high mytsharesdeposited updates in liquidation functions leads to critical tvl inflation

**Submitted on Oct 29th 2025 at 11:19:37 UTC by @Diavol0 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57883
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Summary

Multiple liquidation-related functions in AlchemistV3 (`_forceRepay`, `_liquidate`, and `_doLiquidation`) fail to update the global `_mytSharesDeposited` variable after transferring MYT tokens out of the contract. This creates a **permanent accounting inflation** where `_mytSharesDeposited` exceeds the actual MYT token balance held by the protocol. This inflation corrupts the Total Value Locked (TVL) calculation used in critical solvency checks, potentially allowing the protocol to bypass emergency liquidation protections during market stress events.

### Root Cause

The protocol maintains `_mytSharesDeposited` as a tracking variable for the total MYT (yield token shares) held by the Alchemist contract. This value is used in `_getTotalUnderlyingValue()` to calculate the protocol's TVL, which feeds into global collateralization checks.

**The invariant that must be maintained:**

```
_mytSharesDeposited == actual MYT token balance held by Alchemist
```

**Correct implementations** (withdraw, repay, burn, redeem):

```solidity
// withdraw() Line 410
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;  // ✓ Correct

// repay() Line 541
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;  // ✓ Correct

// redeem() Line 638
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;  // ✓ Correct
```

**Buggy implementations** (liquidation functions):

1. **\_forceRepay Line 774** (protocol fee transfer):

```solidity
TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
// ❌ MISSING: _mytSharesDeposited -= protocolFeeTotal;
```

2. **\_forceRepay Line 779** (transmuter transfer):

```solidity
TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
// ❌ MISSING: _mytSharesDeposited -= creditToYield;
```

3. **\_liquidate Line 826** (liquidator fee, earmark path):

```solidity
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
// ❌ MISSING: _mytSharesDeposited -= feeInYield;
```

4. **\_liquidate Line 840** (liquidator fee, unearmark path):

```solidity
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
// ❌ MISSING: _mytSharesDeposited -= feeInYield;
```

5. **\_doLiquidation Line 875** (transmuter transfer):

```solidity
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
// ❌ MISSING: _mytSharesDeposited -= (amountLiquidated - feeInYield);
```

6. **\_doLiquidation Line 879** (liquidator fee):

```solidity
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
// ❌ MISSING: _mytSharesDeposited -= feeInYield;
```

### Impact Analysis

**Immediate Impact per Liquidation:**

* MYT tokens transferred out but `_mytSharesDeposited` not decreased
* Protocol's actual MYT balance < `_mytSharesDeposited`
* `_getTotalUnderlyingValue()` returns inflated TVL

**Test Results Demonstrate:**

* Single liquidation: **1.14e18 MYT inflation**
* TVL inflation: **109e18 underlying tokens (40.6% inflation!)**
* Global collateralization artificially boosted by **151%**
* Actual collateralization: 373%
* Reported collateralization: 524%

**Critical Security Implication:**

Line 1258 in `calculateLiquidation()` contains an emergency protection mechanism:

```solidity
if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
    // Force FULL liquidation in emergency low-collateralization state
    return (debt, debt, 0, outsourcedFee);
}
```

This check uses `_getTotalUnderlyingValue()` which depends on the inflated `_mytSharesDeposited`:

```solidity
// Line 862
uint256 alchemistCurrentCollateralization =
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

// Line 1239 (_getTotalUnderlyingValue)
uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
```

**Result**: When protocol is actually under-collateralized, the inflated `_mytSharesDeposited` makes it appear safe, bypassing emergency protections.

**Cumulative Effect:**

* Every liquidation adds more inflation
* No self-healing mechanism exists
* Worsens progressively with protocol usage
* Cannot be corrected without upgrade

### Affected Components

* **Contract**: `AlchemistV3.sol`
* **Functions**:
  * `_forceRepay()` (Lines 738-782)
  * `_liquidate()` (Lines 793-851)
  * `_doLiquidation()` (Lines 853-884)
* **Variable**: `_mytSharesDeposited` (global state)
* **Call Chain**: `liquidate() → _liquidate() → [_forceRepay(), _doLiquidation()]`
* **Affected Calculation**: `_getTotalUnderlyingValue()` → `calculateLiquidation()` Line 862

## Link to Proof of Concept

<https://gist.github.com/6newbie/4b21d92c84a7f1bdcbd5f673a8c872eb>

## Proof of Concept

### Step-by-Step Reproduction

**Prerequisites:**

* AlchemistV3 contract deployed and initialized
* User positions with debt exist
* Transmuter operational for earmarking

**Steps:**

1. **Setup Initial State**

   ```
   - User creates position: 200e18 collateral
   - User borrows max debt: 180e18 (at 90% LTV)
   - Create transmuter redemption to enable earmarking
   - Advance blocks to earmark 60% of debt
   ```
2. **Trigger Liquidation**

   ```
   - Drop collateral price by ~5.9% to trigger undercollateralization
   - Call liquidate(accountId)
   - This internally calls:
     * _liquidate()
       → _forceRepay() for earmarked debt
       → _doLiquidation() for remaining debt
   ```
3. **Observe Bug - Token Accounting**

   ```
   MYT Token Flows:
   - To Liquidator:           1.14e18  (fee)
   - To Transmuter:           0        (in this scenario)
   - To Protocol Fee:         0        (insufficient balance scenario)
   - Total transferred OUT:   1.14e18

   _mytSharesDeposited update: NONE ❌

   Result:
   - Actual MYT balance decreased by ~115.5e18 total
     (114.37e18 used for debt burn + 1.14e18 transferred out)
   - _mytSharesDeposited unchanged
   - Inflation created: 1.14e18
   ```
4. **Observe Bug - TVL Inflation**

   ```
   Reported TVL (using _mytSharesDeposited):  377.7e18
   Actual TVL (using actual balance):         268.6e18
   Artificial Inflation:                      109.0e18 (40.6%!)
   ```
5. **Observe Bug - Collateralization Inflation**

   ```
   Minimum Required:        200% (2.0x)
   Reported (with bug):     524% (5.24x)
   Actual (without bug):    373% (3.73x)
   Artificial Boost:        151% (1.51x)
   ```

### Automated Test Verification

A comprehensive test case is provided as a standalone file for easy integration.

**Test File Location:**

* `testBug_Complete_Liquidation_Path_MytShares_Inflation.sol` (standalone test function in my gist)

**Run test:**

```bash
forge test --match-test testBug_Complete_Liquidation_Path_MytShares_Inflation -vv
```

**Expected output:**

```
[PASS] testBug_Complete_Liquidation_Path_MytShares_Inflation()

Logs:
  === Transfer Analysis ===
  Total transferred OUT:          1143720000000000001210

  === BUG DEMONSTRATION ===
  The following MYT transfers occurred WITHOUT _mytSharesDeposited update:
  2. To Liquidator:               1143720000000000001210
     (_liquidate L826/L840 + _doLiquidation L879)

  TOTAL INFLATION:                1143720000000000001210

  === Protocol-Wide TVL Inflation ===
  Reported TVL (WITH bug):        377714825306893295200000
  Actual TVL (WITHOUT bug):       268634825306893295189092
  Artificial inflation:           109080000000000000010908
  Inflation percentage:           4060 bps  (40.6%)

  === Global Collateralization Impact ===
  Minimum required (200%):        1111111111111111111
  Reported (WITH bug):            5246039240373517988
  Actual (WITHOUT bug):           3731039240373517988
  Artificial boost:               1515000000000000000
```

The test demonstrates:

* ✅ MYT tokens transferred out without `_mytSharesDeposited` update
* ✅ **TVL inflation of 40.6%** from a single liquidation
* ✅ **Global collateralization artificially boosted by 151%**
* ✅ Protocol invariant broken: `_mytSharesDeposited ≠ actual MYT balance`

### Comparison: Complete Call Paths

**Scenario: Liquidation with 60% earmarked debt**

```
liquidate(accountId)
  └─> _liquidate()
        ├─> _forceRepay(earmarked amount)
        │     ├─> Line 774: safeTransfer(myt, protocolFeeReceiver, fee)
        │     │             ❌ Missing: _mytSharesDeposited -= fee
        │     └─> Line 779: safeTransfer(myt, transmuter, creditToYield)
        │                   ❌ Missing: _mytSharesDeposited -= creditToYield
        │
        └─> _doLiquidation(remaining amount)
              ├─> Line 826/840: safeTransfer(myt, msg.sender, feeInYield)
              │                  ❌ Missing: _mytSharesDeposited -= feeInYield
              ├─> Line 875: safeTransfer(myt, transmuter, amountLiquidated - feeInYield)
              │             ❌ Missing: _mytSharesDeposited -= (amountLiquidated - feeInYield)
              └─> Line 879: safeTransfer(myt, msg.sender, feeInYield)
                            ❌ Missing: _mytSharesDeposited -= feeInYield
```
