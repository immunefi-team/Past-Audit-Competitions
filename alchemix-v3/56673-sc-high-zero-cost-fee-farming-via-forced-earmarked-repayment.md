# 56673 sc high zero cost fee farming via forced earmarked repayment

**Submitted on Oct 19th 2025 at 08:51:34 UTC by @failsafe\_intern for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56673
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Vulnerability Overview

Any external caller can repeatedly trigger forced earmarked debt repayment on accounts with positive earmarked debt and receive repayment fees directly from victim collateral, even when providing zero capital. This enables continuous fee extraction from all borrowers with earmarked debt.

### Root Cause

**AlchemistV3.sol:585-598** - `_liquidate()` pays repayment fees to callers even for zero-capital forced repayments:

```solidity
// LINE 585-593: Pay fee when debt reaches zero after earmark repayment
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    return (repaidAmountInYield, feeInYield, 0);
}

// LINE 595-598: Pay fee when account becomes healthy after earmark repayment
} else {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    return (repaidAmountInYield, feeInYield, 0);
}
```

**AlchemistV3.sol:548-555** - `_resolveRepaymentFee()` extracts fee from victim collateral:

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    return fee;
}
```

**AlchemistV3.sol:765-797** - `_forceRepay()` uses victim's own collateral for repayment:

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ... validation ...
    uint256 creditToYield = convertDebtTokensToYield(credit);
    _subDebt(accountId, credit);
    earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;
    creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
    account.collateralBalance -= creditToYield;  // Uses victim's collateral
    // ... protocol fee deduction ...
}
```

The code **does not distinguish** between liquidations requiring capital injection and forced earmark repayments using victim's own collateral. Both code paths pay the same repayment fee to `msg.sender`.

### Attack Flow

1. **Precondition**: Target account has `earmarked > 0` (accumulated through transmuter redemptions)
2. **Attacker Action**: Call `liquidate(accountId)` with zero capital
3. **Line 484**: `_liquidate()` calls `_forceRepay()` if `account.earmarked > 0`
4. **Line 765-797**: `_forceRepay()` repays earmarked debt using **victim's collateral**
5. **Line 585-598**: If account becomes healthy or debt reaches zero, pay repayment fee
6. **Line 548-555**: `_resolveRepaymentFee()` deducts `repaymentFee%` from victim's collateral
7. **Line 590/598**: Fee transferred to `msg.sender` (attacker)
8. **Result**: Attacker receives fee without providing any capital
9. **Repeat**: Attack continues as new earmarks accumulate

### Impact

**Economic Loss**: Continuous collateral drain from all borrowers with earmarked debt.

**Attack Specifics**:

* **Capital Required**: 0 (attacker provides no repayment funds)
* **Fee Earned**: `repaymentFee%` of earmarked amount (e.g., 1% of 10 ETH = 0.1 ETH per harvest)
* **Gas Cost**: \~150k-300k per `liquidate()` call
* **Repeatability**: Unlimited, as earmarks continuously accrue via transmuter
* **Scale**: Affects all users with `earmarked > 0`

**Attack Economics**:

* If `repaymentFee = 1%` and typical earmarked debt is 10 ETH worth
* Attacker extracts 0.1 ETH worth of MYT per call
* With gas at 50 gwei, cost is \~0.0075-0.015 ETH
* **Net profit**: 0.085-0.0925 ETH per harvest (\~10-12x gas cost)
* Scales across hundreds of accounts

**System-Wide Impact**:

* Perverse incentive: Attackers harvest from healthy accounts rather than liquidate unhealthy ones
* Legitimate liquidations become less profitable (opportunity cost)
* User collateral drained over time through repeated harvesting
* Protocol reputation damage from unexpected fee extraction

## Link to Proof of Concept

<https://gist.github.com/Joshua-Medvinsky/b10216c271f32985695adf87e20fd78b>

## Proof of Concept

## 3. Proof of Concept

### Step-by-Step Reproduction

**Setup:**

```solidity
// Deploy AlchemistV3 with repaymentFee = 100 BPS (1%)
// User account setup:
accountId = 1
account.collateralBalance = 100 ETH worth of MYT
account.debt = 50 ETH worth
account.earmarked = 10 ETH worth (accumulated from transmuter)
account.collateralizationRatio = 200% (HEALTHY)
```

**Step 1: Attacker Harvests Fee Without Capital**

```solidity
// Attacker calls (providing ZERO capital):
AlchemistV3.liquidate(accountId);

// Execution path:
// 1. liquidate() → _liquidate(accountId)
// 2. _liquidate() checks: account.earmarked > 0 ✅
// 3. _liquidate() calls: _forceRepay(accountId, account.earmarked)
```

**Step 2: Forced Repayment Using Victim's Collateral**

```solidity
// LINE 765-797: _forceRepay() executes:
uint256 creditToYield = convertDebtTokensToYield(10 ETH) = 10 MYT
_subDebt(accountId, 10 ETH)  // debt: 50 → 40 ETH
account.earmarked = 0
account.collateralBalance -= 10 MYT  // collateral: 100 → 90 MYT

// CRITICAL: Repayment used victim's own collateral (not attacker's capital)
```

**Step 3: Fee Extracted from Victim**

```solidity
// LINE 585-598: Check if account is now healthy
account.collateralizationRatio = (90 MYT / 40 debt) = 225% > lowerBound

// Account is healthy, so LINE 595-598 executes:
feeInYield = _resolveRepaymentFee(accountId, 10 MYT)

// LINE 548-555: Calculate fee
fee = 10 MYT * 100 BPS / 10000 = 0.1 MYT (1%)
account.collateralBalance -= 0.1 MYT  // collateral: 90 → 89.9 MYT

// LINE 598: Transfer fee to attacker
TokenUtils.safeTransfer(myt, msg.sender, 0.1 MYT)
```

**Result:**

```solidity
// Attacker balance: +0.1 MYT
// Attacker capital used: 0
// Victim collateral: -10.1 MYT (10 for repayment + 0.1 fee)
// Victim debt: -10 ETH worth
// Net effect: Attacker profited 0.1 MYT with ZERO capital
```

**Step 4: Repeat Attack**

```solidity
// After more earmarks accumulate (e.g., 5 ETH worth):
AlchemistV3.liquidate(accountId);

// Another 0.05 MYT fee extracted (1% of 5 ETH)
// Attack repeats indefinitely as earmarks continuously accrue
```

### Validation Steps

**Expected Behavior**: Repayment fees should only be paid when external party provides capital to liquidate unhealthy positions.

**Actual Behavior**: Fees are paid even when:

1. No capital is provided by caller
2. Repayment uses victim's own collateral
3. Account is healthy (not underwater)
4. Only earmarked debt is being burned

### Code References

**Vulnerable Functions**:

* **Fee payment**: `_liquidate()` at lines 585-598 (AlchemistV3.sol)
* **Fee calculation**: `_resolveRepaymentFee()` at lines 548-555 (AlchemistV3.sol)
* **Forced repayment**: `_forceRepay()` at lines 765-797 (AlchemistV3.sol)
* **Entry point**: `liquidate()` at line 495 (AlchemistV3.sol)

**Critical Logic Flaw**: The code does not check whether the liquidation required external capital. Both code paths (actual liquidation vs forced earmark repayment) execute the same fee payment logic at lines 585-598.

### Proof of Concept (GitHub Gist)

**Complete Foundry POC**: <https://gist.github.com/Joshua-Medvinsky/b10216c271f32985695adf87e20fd78b>

The gist contains:

* `AlchemistV3Harness.sol` - Vulnerable code extracted from AlchemistV3.sol (lines 548-889)
* `MockContracts.sol` - Mock MYT token and Transmuter for isolated testing
* `ECON_C001_POC.t.sol` - 7 comprehensive tests demonstrating the attack
* `foundry.toml` - Forge configuration
* `README.md` - Setup instructions and detailed analysis

**Test Results**: ✅ All tests passing

* Test demonstrates attacker extracting 0.1 MYT fee with ZERO capital
* Victim loses collateral, attacker gains fee
* ROI: Infinite (zero capital investment)

### Mitigation

**Option 1: Require Capital for Fee**

```solidity
// Track whether caller provided capital:
bool callerProvidedCapital = false;
if (collateralizationRatio <= lowerBound) {
    // Actual liquidation path - caller provides capital
    callerProvidedCapital = true;
    // ... liquidation logic ...
}

// Only pay fee if capital was provided:
if (callerProvidedCapital) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

**Option 2: Access Control**

```solidity
// Restrict forced earmark repayment to authorized liquidators:
if (account.earmarked > 0 && !isAuthorizedLiquidator(msg.sender)) {
    revert UnauthorizedForcedRepayment();
}
```
