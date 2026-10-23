# 51001 sc insight inaccurate share calculation in emitted event for non bridge deposits

**Submitted on Jul 30th 2025 at 10:52:15 UTC by @Ambitious\_DyDx for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51001
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

For non-bridge deposits, the `Deposit` event typically emits a "computed share" value rather than the actual shares minted. Different teller implementations can produce different actual minted share values (e.g., due to fees or adjustments), so emitting the computed value may produce discrepancies in off-chain tracking.

### Vulnerability Details

In non-bridge functions such as `depositOneInch` and `depositOkxUniversal`:

* The contract calls `teller.deposit`, which mints and returns the actual shares that were minted and transferred via `ERC20(vaultAddress).safeTransfer`.
* A `Deposit` event is emitted by `_calcSharesAndEmitEvent`, which uses a computed shareAmount like this:

```solidity
shares = supportedAssetAmount.mulDivDown(10 ** teller.vault().decimals(), AccountantWithRateProviders(teller.accountant()).getRateInQuoteSafe(supportedAsset))
```

* That computed share value may differ from the actual shares returned by `teller.deposit` (for example if the vault applies fees or other adjustments).
* Bridge functions use computed shares as an estimate (minting may occur on another chain), but for non-bridge functions the event should use the actual shares for accuracy.

### Impact Details

* Off-chain systems (analytics, UIs, accounting) that rely on emitted share amounts may record incorrect share balances, causing confusion and mismatches between logs and actual holdings.
* No funds are lost, but event logs may not accurately reflect promised returns.
* Matches the in-scope impact: "Contract fails to deliver promised returns, but doesn't lose value."

## Proof of Concept

{% stepper %}
{% step %}

### Deposit flow mismatch — step 1

A user calls `depositOneInch` to deposit assets. The `teller.deposit` call mints (for example) 95 shares (after fees) and returns this value.
{% endstep %}

{% step %}

### Deposit flow mismatch — step 2

`_calcSharesAndEmitEvent` computes 100 shares and emits that value in the `Deposit` event.

Expected Behavior: The event emits the actual 95 shares minted.

Actual Behavior: The event emits the computed 100 shares, causing a discrepancy between emitted logs and actual minted shares.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Recommendation: For non-bridge deposit functions, use the actual shares value returned by `teller.deposit` when emitting the `Deposit` event instead of using the computed estimate.
{% endhint %}

## References

* <http://docs.soliditylang.org/>
* <https://immunefisupport.zendesk.com/hc/en-us/articles/33260632501777-Proof-of-Concept-Rules-for-Audit-Competitions>
* <https://immunefi.com/audit-competition/plume-network-attackathon/scope/#top>
* <https://immunefi.com/audit-competition/plume-network-attackathon/information/#top>

***

PS: This report assumes that the value returned by `teller.deposit` and the value computed by `_calcSharesAndEmitEvent` might not always match.
