# 50887 sc insight arcotokenpurchase purchasemade event mislabels payment amount as pricepaid instead of unit price

**Submitted on Jul 29th 2025 at 11:15:52 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50887
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>

## Summary

The `PurchaseMade` event in ArcTokenPurchase is defined with a field named `pricePaid`, which by its name suggests a per-token (unit) price. In the implementation, however, the contract emits the total payment amount as `pricePaid`. This naming/value mismatch can mislead integrators, analytics tools, and UIs that consume the event and expect a unit price.

{% hint style="info" %}
This is an informational/insight issue: it does not cause direct loss of funds, but can cause incorrect reporting or UI display of token prices.
{% endhint %}

## Vulnerability Details

The `PurchaseMade` event signature:

* buyer
* tokenContract
* amount — base units of ArcToken bought
* pricePaid — misleadingly named (implies unit price)

In the `buy()` implementation the event is emitted as:

* `arcTokensBaseUnitsToBuy` → correctly used for amount (number of base units bought)
* `_purchaseAmount` → passed into `pricePaid`, but `_purchaseAmount` is the total spent (not a per-token price)

This mismatch means consumers may interpret `pricePaid` as the per-token price when it actually contains total payment.

## Impact Details

* Severity: Low (no direct loss of funds).
* Impact: Misinterpretation of on-chain sales data — faulty price history, misleading UI or analytics.
* Affects: Any integration or analytics system that consumes the `PurchaseMade` event and expects `pricePaid` to be a unit price.

{% hint style="warning" %}
Integrations relying on this event to display unit price may show incorrect prices (e.g., dividing/not dividing by token amount incorrectly). This can mislead users or analytics, but not cause funds loss.
{% endhint %}

## Proof of Concept

<details>

<summary>Relevant faulty code snippet (click to expand)</summary>

{% code title="ArcTokenPurchase.sol (snippet)" %}

```solidity
// Event signature
event PurchaseMade(
    address indexed buyer,
    address indexed tokenContract,
    uint256 amount,      // base units of ArcToken bought
    uint256 pricePaid    // misleadingly named
);

// Emission in buy()
emit PurchaseMade(
    msg.sender,
    _tokenContract,
    arcTokensBaseUnitsToBuy,  // correct: number of base units
    _purchaseAmount           // incorrect: total spent, not unit price
);
```

{% endcode %}

</details>

## Recommendations

* Rename the event field to reflect its true meaning (e.g., `totalPaid`, `amountPaid`, or `paymentAmount`), or
* Emit both values: the unit price and the total paid (ensure correct units and precision), or
* Add documentation/comments clarifying that `pricePaid` is the total payment and not the per-token price.

Choose one of the above approaches to avoid ambiguity for downstream consumers.

## References

* Target repository: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
