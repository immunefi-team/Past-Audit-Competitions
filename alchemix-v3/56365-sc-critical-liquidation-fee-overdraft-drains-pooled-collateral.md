# 56365 sc critical liquidation fee overdraft drains pooled collateral

**Submitted on Oct 15th 2025 at 06:04:30 UTC by @failsafe\_intern for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56365
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

The `_liquidate` function pays liquidators a repayment fee that exceeds what was deducted from the liquidated account, stealing the shortfall from the protocol's pooled MYT collateral shared by all users.

### Vulnerability Mechanism

When an account with earmarked debt is liquidated and the debt fully clears, `_resolveRepaymentFee` calculates the fee but only deducts `min(fee, collateralBalance)` from the account:

**File:** `v3-poc-b0505da/src/AlchemistV3.sol`\
**Lines:** 515-533

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    return fee;  // Returns FULL fee
}
```

However, `_liquidate` unconditionally transfers the **full returned fee** to the liquidator from the contract's pooled MYT balance:

**File:** `v3-poc-b0505da/src/AlchemistV3.sol`\
**Lines:** 495-499

```solidity
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);  // Transfers FULL fee from pool
    return (repaidAmountInYield, feeInYield, 0);
}
```

### Root Cause

The mismatch occurs because:

1. `_forceRepay` consumes most/all account collateral during earmarked debt repayment
2. `_resolveRepaymentFee` can only deduct remaining collateral but returns the full calculated fee
3. `_liquidate` transfers the full fee amount from `_mytSharesDeposited` (aggregate pool)

**File:** `v3-poc-b0505da/src/AlchemistV3.sol`\
**Lines:** 377-415 (excerpt)

```solidity
creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
account.collateralBalance -= creditToYield;  // Depletes collateral
```

### Attack Scenario

**Setup:**

* Account has 1000 MYT earmarked debt
* Account has only 10 MYT remaining collateral
* `repaymentFee` = 5% (500 BPS)

**Execution:**

1. Attacker calls `liquidate(accountId)`
2. `_forceRepay` clears the 1000 MYT earmarked debt, consuming the 10 MYT collateral
3. Account debt reaches 0, triggering `_resolveRepaymentFee`
4. Calculated fee: `1000 * 5% = 50 MYT`
5. Account deduction: `min(50, 0) = 0 MYT` (collateral already depleted)
6. Liquidator receives: `50 MYT` from contract's pooled balance

**Result:** Protocol loses 50 MYT from pooled collateral, account contributed 0 MYT

### Impact

* **Direct theft** of pooled user funds (`_mytSharesDeposited`)
* **Accounting desynchronization** between tracked collateral and actual MYT balance
* **Cascading insolvency** as repeated liquidations drain the pool
* **Scalable attack** via `batchLiquidate` across multiple accounts
* No special privileges required - any address can call `liquidate()`

***

## Link to Proof of Concept

<https://gist.github.com/Joshua-Medvinsky/9e997ff1c82ae9267b8442a9696b064b>

## Proof of Concept

## Proof of Concept

### Step 1: Identify Vulnerable Account

Query accounts via `getCDP(accountId)` to find:

* `account.earmarked > 0` (has earmarked debt)
* `account.collateralBalance < (account.earmarked * repaymentFee / BPS)` (insufficient collateral for fee)
* Account is liquidatable (collateralization <= `collateralizationLowerBound`)

### Step 2: Execute Liquidation

Call `liquidate(accountId)` from any address

### Step 3: Observe Overdraft

**If** earmarked debt clears the position (`account.debt == 0`):

**Account Side:**

* Collateral deducted: `min(calculatedFee, account.collateralBalance)` (capped)

**Liquidator Side:**

* Fee received: `calculatedFee` (uncapped, from pooled MYT)

**Shortfall:**

```
stolen = calculatedFee - min(calculatedFee, account.collateralBalance)
```

This amount is extracted from `_mytSharesDeposited` (all users' collateral)

### Step 4: Repeat Attack

Target additional vulnerable accounts via `batchLiquidate()` to systematically drain pooled collateral

### Expected Results

**Single liquidation example:**

* Earmarked debt cleared: 1000 MYT
* Repayment fee (5%): 50 MYT
* Account collateral available: 10 MYT
* Account deduction: 10 MYT
* Liquidator receives: **50 MYT**
* **Stolen from pool: 40 MYT**

**Scaled attack (100 similar accounts):**

* Total stolen: 4,000 MYT from pooled collateral
* Protocol becomes insolvent
* Legitimate withdrawals fail

***

## References

**Affected Functions:**

* `liquidate(uint256)` - Entry point at `AlchemistV3.sol:417`
* `_liquidate(uint256)` - Transfers full fee at lines 495-499
* `_resolveRepaymentFee(uint256,uint256)` - Overdraft calculation at lines 515-533
* `_forceRepay(uint256,uint256)` - Depletes collateral at lines 377-415
* `batchLiquidate(uint256[])` - Enables scaled exploitation
