# 51276 sc low arctokenpurchase re enabling active token sales causes accounting corruption and token loss

**Submitted on Aug 1st 2025 at 11:08:35 UTC by @rilwan99 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51276
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

The `ArcTokenPurchase.enableToken()` function lacks validation to prevent re-enabling already enabled tokens, which leads to accounting inconsistencies and Arc Tokens being permanently locked in the contract.

<details>

<summary>Vulnerability Details</summary>

The vulnerability exists in the `enableToken()` function in ArcTokenPurchase.sol:

```solidity
function enableToken(
    address _tokenContract,
    uint256 _numberOfTokens,
    uint256 _tokenPrice
) external onlyTokenAdmin(_tokenContract) {
    // ... validation checks ...
    
    // No check if token is already enabled
    ps.tokenInfo[_tokenContract] = TokenInfo({
        isEnabled: true,
        tokenPrice: _tokenPrice,
        totalAmountForSale: _numberOfTokens,  // Overwrites previous value
        amountSold: 0  // Resets to 0, losing previous sales data
    });
    
    ps.enabledTokens.add(_tokenContract);
    emit TokenSaleEnabled(_tokenContract, _numberOfTokens, _tokenPrice);
}
```

No validation checks if the token is already enabled. The assignment completely replaces the existing `TokenInfo` struct, losing all previous state including sales history and actual token availability.

</details>

## Impact Details

* Token Loss: Unsold tokens become permanently locked when `amountSold` resets to 0.
* Accounting Mismatch: Mismatch between actual contract balance and recorded availability.

<details>

<summary>References</summary>

<https://github.com/plumenetwork/contracts/blob/main/arc/src/ArcTokenPurchase.sol>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Initial Setup

* Admin calls `enableToken(tokenAddr, 100e18, 10e6)` // 100 tokens at $10 each
* Contract receives 100 Arc Tokens
* State: `totalAmountForSale = 100e18, amountSold = 0, isEnabled = true`
  {% endstep %}

{% step %}

### Users Purchase Tokens

* User1 calls `buy(tokenAddr, 200e6, 20e18)` // Buys 200 tokens
* User2 calls `buy(tokenAddr, 200e6, 20e18)` // Buys 200 tokens
* State after sales: `totalAmountForSale = 100e18, amountSold = 40e18, isEnabled = true`
* Remaining for sale: `60e18` Arc tokens
  {% endstep %}

{% step %}

### Token Admin Re-enables (Bug Trigger)

* Admin calls `enableToken(tokenAddr, 50e18, 20e6)` // Admin wants to change price
* State becomes: `totalAmountForSale = 50e18, amountSold = 0, isEnabled = true` // OVERWRITTEN
* Contract still holds 60 Arc Tokens, but accounting shows only 50 available

Impact: 10 Arc Tokens (60e18 - 50e18) are left permanently stuck in the contract and not accounted for. There is no way to withdraw/recover these funds.
{% endstep %}
{% endstepper %}
