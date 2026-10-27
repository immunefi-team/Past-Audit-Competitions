# 51391 sc low enabletoken function overwrites amountsold to zero causing permanent loss of sales history

**Submitted on Aug 2nd 2025 at 10:47:27 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51391
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `enableToken` function unconditionally resets the `amountSold` field to zero when called, permanently erasing all records of previous token sales.

This data loss prevents the contract from delivering accurate sales tracking, which is a core functionality of any token sale contract.

### Vulnerability Details

The vulnerability occurs in the `enableToken` function:

```solidity
ps.tokenInfo[_tokenContract] = TokenInfo({ 
    isEnabled: true, 
    tokenPrice: _tokenPrice, 
    totalAmountForSale: _numberOfTokens, 
    amountSold: 0  // ← Always resets to 0, erasing sales history
});
```

The function completely overwrites the `TokenInfo` struct without preserving the existing `amountSold` value. This means:

* Any record of previous sales is permanently lost
* The contract "forgets" how many tokens were sold
* There's no way to recover this data once overwritten

### Impact Details

The contract fails to maintain accurate sales records:

* Permanent Data Loss: If 600 tokens were sold and `enableToken` is called again, the contract permanently loses the record of those 600 sales
* False Reporting: `getTokenInfo()` returns `amountSold: 0` instead of the actual amount, making the contract unreliable for:
  * Revenue tracking
  * Sales analytics
  * Financial reconciliation
  * Audit trails
* Broken Invariants: The fundamental accounting equation is violated:
  * Actual tokens distributed: 600
  * Contract's record: 0
  * This discrepancy cannot be corrected

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L136-L178>

## Proof of Concept

{% stepper %}
{% step %}

### Step: Initial Setup

```solidity
enableToken(tokenAddress, 1000e18, 10e6); // 1000 tokens at $10
// State: {amountSold: 0, totalAmountForSale: 1000e18}
```

{% endstep %}

{% step %}

### Step: Sales Occur

```solidity
// Users buy 600 tokens through multiple transactions
// State: {amountSold: 600e18, totalAmountForSale: 1000e18}
```

{% endstep %}

{% step %}

### Step: Admin Updates Parameters

```solidity
// Admin wants to update price or adjust remaining supply
enableToken(tokenAddress, 400e18, 15e6); // 400 tokens at $15
// State: {amountSold: 0, totalAmountForSale: 400e18}
```

Result:

```solidity
// Before: amountSold = 600e18 (correct)
// After: amountSold = 0 (data lost)
```

The contract now incorrectly reports that zero tokens have been sold, despite 600 tokens actually being distributed. This permanent loss of sales data causes the contract to fail in its core purpose of tracking token sales accurately.
{% endstep %}
{% endstepper %}
