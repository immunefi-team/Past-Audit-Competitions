# 52075 sc medium arctokenpurchase contract is a token holder and may be yield recipient&#x20;

**Submitted on Aug 7th 2025 at 18:18:54 UTC by @Finlooz4 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52075
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Temporary freezing of funds for at least 1 hour

## Description

### Brief/Intro

The `ArcTokenPurchase` contract can receive yield tokens when acting as an ArcToken holder. If the yield token differs from the purchase token, these tokens become permanently stuck since the contract lacks withdrawal functionality for arbitrary ERC20 tokens.

### Vulnerability Details

The `ArcToken` contract's `distributeYield` function transfers yield tokens to holders, including the `ArcTokenPurchase` contract if it holds ArcToken tokens. The yield tokens sent to `ArcTokenPurchase` may be irretrievable because the contract only supports withdrawing the purchase token via `withdrawPurchaseTokens`. If the yield token (set via `setYieldToken`) differs from the purchase token (set via `setPurchaseToken`), there is no function to withdraw yield tokens, potentially locking them in the contract.

Relevant code excerpt:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L389-L462>

```solidity
// ArcToken.sol - Yield Distribution
function distributeYield(uint256 amount) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
    ...
    for (uint256 i = 0; i < lastProcessedIndex; i++) {
        address holder = $.holders.at(i);
        if (!_isYieldAllowed(holder)) continue;
        ...
        yToken.safeTransfer(holder, share); // Tokens sent to ArcTokenPurchase
    }
    ...
}
```

```solidity
// ArcTokenPurchase.sol - Withdrawal Functions (Limited)
function withdrawPurchaseTokens(address to, uint256 amount) external ... {
    // Only withdraws purchaseToken (e.g., DAI)
    ps.purchaseToken.transfer(to, amount); 
}

function withdrawUnsoldArcTokens(...) external ... {
    // Only withdraws ArcTokens
    token.transfer(to, amount);
}
```

### Impact Details

If the yield token (e.g., USDC) ≠ purchase token (e.g., DAI), USDC sent to `ArcTokenPurchase` during distributions becomes permanently inaccessible.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Deploy contracts

Deploy `ArcToken` with a yield token (e.g., USDC) and `ArcTokenPurchase` with a different purchase token (e.g., DAI).
{% endstep %}

{% step %}

### Step 2 — Enable token sale

Enable token sale in `ArcTokenPurchase` using `enableToken`, transferring ArcToken tokens to it, making it a holder.
{% endstep %}

{% step %}

### Step 3 — Distribute yield

Call `distributeYield` in `ArcToken` to distribute yield tokens (USDC) to holders, including `ArcTokenPurchase`.
{% endstep %}

{% step %}

### Step 4 — Attempt withdrawal

Admin tries to withdraw USDC from `ArcTokenPurchase` but fails:

* `withdrawPurchaseTokens` only handles the configured purchase token (e.g., DAI).
* `withdrawUnsoldArcTokens` only handles ArcTokens.
  {% endstep %}

{% step %}

### Step 5 — Result

USDC is permanently stuck in `ArcTokenPurchase`.
{% endstep %}
{% endstepper %}

## References

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
