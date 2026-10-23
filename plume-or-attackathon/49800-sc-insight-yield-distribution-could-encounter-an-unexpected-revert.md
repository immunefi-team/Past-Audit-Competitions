# 49800 sc insight yield distribution could encounter an unexpected revert

**Submitted on Jul 19th 2025 at 14:19:28 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49800
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Function call could be DoS

## Description

### Brief/Intro

When calling `distributeYield()` or `distributeYieldWithLimit()`, the transaction might experience an unexpected revert as a result of a yield-token blacklisted token holder.

### Vulnerability Details

As mentioned in `ArcTokenFactory.sol`, the yield token could be USDC:

param yieldToken Address of the yield token (e.g., USDC)

USDC is one example of a token with a blacklist, reverting whenever a blacklisted address is involved in a transaction. However, when `distributeYield()` or `distributeYieldWithLimit()` are being called, the function iterates over *holders*, transferring yield tokens to multiple addresses at the same time. Since `yToken.safeTransfer()` is not called using a try-catch scheme, any blacklisted user would revert the entire transaction.

### Impact Details

Any blacklisted user would cause an unexpected revert without a clear event indicating the problem (and more importantly, the index of the problematic holder). While this holder could be skipped using the `distributeYieldWithLimit()` function, the lack of clear revert reason might cause a temporary DoS until the issue is identified.

{% hint style="info" %}
Suggestion: Consider using a try-catch block around transfers of the yield-token, skipping holders that failed to receive the yield-token, and emitting an event describing the failure and the index (or address) of the problematic holder.
{% endhint %}

## Proof of Concept

<details>

<summary>Proof of Concept</summary>

As per Attackathon rules, no PoC is necessary.

</details>
