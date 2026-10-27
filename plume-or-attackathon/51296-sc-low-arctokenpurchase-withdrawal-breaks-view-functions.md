# 51296 sc low arctokenpurchase withdrawal breaks view functions

**Submitted on Aug 1st 2025 at 14:11:09 UTC by @funkornaut for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51296
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `withdrawUnsoldArcTokens()` function in ArcTokenPurchase.sol contains an accounting logic error that causes multiple view functions (`getMaxNumberOfTokens()` and `getTokenInfo()`) to return incorrect values after admin withdrawals. When token administrators withdraw unsold tokens, the function correctly transfers the tokens but fails to update the internal sale accounting, leading to persistent discrepancies between reported and actual token availability.

### Vulnerability Details

The vulnerability exists because `withdrawUnsoldArcTokens()` only performs the token transfer without updating the `TokenInfo` accounting:

```solidity
function withdrawUnsoldArcTokens(
    address _tokenContract,
    address to,
    uint256 amount
) external onlyTokenAdmin(_tokenContract) {
    // ... validation checks ...
    
    ArcToken token = ArcToken(_tokenContract);
    uint256 contractBalance = token.balanceOf(address(this));
    if (contractBalance < amount) {
        revert InsufficientUnsoldTokens();
    }

    bool success = token.transfer(to, amount);  // Transfers tokens
    if (!success) {
        revert ArcTokenWithdrawalFailed();
    }
    
    //  MISSING: TokenInfo accounting update
    // Should include: info.totalAmountForSale -= amount;
}
```

Meanwhile, both affected view functions rely on the unupdated accounting:

```solidity
function getMaxNumberOfTokens(address _tokenContract) external view returns (uint256) {
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    return info.totalAmountForSale - info.amountSold; // Returns stale data after withdrawals
}

function getTokenInfo(address _tokenContract) external view returns (TokenInfo memory) {
    return _getPurchaseStorage().tokenInfo[_tokenContract]; // Returns struct with wrong totalAmountForSale
}
```

TokenInfo Struct Corruption:

```solidity
struct TokenInfo {
    bool isEnabled;           // ✅ Correct
    uint256 tokenPrice;       // ✅ Correct  
    uint256 totalAmountForSale; // ❌ CORRUPTED - Not reduced by withdrawals
    uint256 amountSold;       // ✅ Correct (updated by purchases)
}
```

### Impact Details

When an admin withdraws Arc tokens and there is an ongoing arc token sale the view functions to get token info and the max number of tokens available for purchase will be incorrect. This may cause issues with user facing front end and overall user experience.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L419-#L429\\>
<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L359-#L363\\>
<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L347-#L350>

## Link to Proof of Concept

<https://gist.github.com/Funkornaut/cd3844097890ece413201eba6ef6ff4a>

## Proof of Concept

{% stepper %}
{% step %}

### Setup: Token Sale Initialized Properly

Admin deposits 500 tokens into `ArcTokenPurchase` contract.

Internal state reflects:

* totalAmountForSale = 500
* amountSold = 0
  {% endstep %}

{% step %}

### Exploit Trigger: Admin Withdraws Unsold Tokens

Admin calls `withdrawUnsoldArcTokens()` and removes 200 tokens from the contract but the function fails to update `totalAmountForSale`.
{% endstep %}

{% step %}

### Mismatched State Emerges

Actual token balance in contract: 300

Reported available tokens via:

* `getMaxNumberOfTokens()` = totalAmountForSale - amountSold = 500 - 0 = 500
* `getTokenInfo().totalAmountForSale` = 500

200 "phantom tokens" are still reported as available, despite no longer existing.
{% endstep %}

{% step %}

### Consequences

UI / contract believes there are 500 tokens for sale.

User tries to buy all 500, but the transaction reverts because only 300 tokens are actually purchasable.
{% endstep %}

{% step %}

### Bug Persists Indefinitely

Additional admin withdrawals deepen the inconsistency.

View functions (`getTokenInfo`, `getMaxNumberOfTokens`) remain broken and mislead users and integrations.
{% endstep %}
{% endstepper %}
