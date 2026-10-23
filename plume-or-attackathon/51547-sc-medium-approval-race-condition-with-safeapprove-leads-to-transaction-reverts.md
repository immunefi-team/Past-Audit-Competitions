# 51547 sc medium approval race condition with safeapprove leads to transaction reverts

**Submitted on Aug 3rd 2025 at 21:11:00 UTC by @Tomioka for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51547
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Descriptive Summary

The contract uses `safeApprove` without first resetting token allowances to zero. For tokens that do not follow the ERC20 `approve` semantics (for example, USDT), attempting to change a non-zero allowance directly to another non-zero value will revert. This causes deposit transactions to fail, creating a denial of service for affected tokens.

### Brief/Intro

Multiple `safeApprove` calls in the contract do not reset the allowance to zero before setting it to the desired amount. Tokens such as USDT implement protections that revert when changing a non-zero allowance to a different non-zero value. The correct approach for such tokens is a two-step process: set allowance to `0` and then set it to the target amount.

## Vulnerability Details

The unsafe pattern appears at multiple locations where `safeApprove` (or `approve`) is called without first resetting allowance to zero:

* Line 338: `depositAsset.safeApprove(address(aggregator), depositAmount);`
* Line 353: `canonicalWrapToken.approve(address(aggregator), nativeValueToWrap);`
* Line 377: `depositAsset.safeApprove(okxApprover, fromTokenAmount);`
* Line 399: `supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);`
* Line 400: `supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);`

Tokens like USDT prevent changing an existing non-zero allowance to another non-zero value. Without the two-step approve (first `0`, then desired amount), these calls can revert.

## Impact Details

* Users attempting to deposit tokens with non-standard `approve` behavior (e.g., USDT) will experience transaction reverts.
* This results in a denial of service for deposits of those tokens: funds remain in users' wallets and deposits cannot be completed.
* No direct fund theft is reported, but core functionality is disrupted, harming user trust and operations. Recovery requires contract changes or user workarounds.

## Proof of Concept

<details>

<summary>Proof of Concept (details)</summary>

The functions `depositOneInch` and `depositOkxUniversal` (and other paths) call `safeApprove` (or `approve`) directly with the desired non-zero allowance. If the token already has a non-zero allowance set for the same spender, tokens like USDT will revert the `approve` call, causing the whole transaction to revert.

</details>

## Attack Scenario

{% stepper %}
{% step %}
A user attempts to deposit USDT tokens via `depositOneInch`.
{% endstep %}

{% step %}
The contract executes `depositAsset.safeApprove(address(aggregator), depositAmount)` (line 338).
{% endstep %}

{% step %}
If a previous non-zero allowance for the aggregator exists, USDT's non-standard `approve` reverts the transaction.
{% endstep %}

{% step %}
The user cannot complete the deposit; funds remain in their wallet.
{% endstep %}

{% step %}
An analogous failure can occur in `depositOkxUniversal` when calling `supportedAsset.safeApprove(vaultAddress, supportedAssetAmount)` (line 400).
{% endstep %}
{% endstepper %}

## References

* Code locations: Lines 338, 353, 377, 399, 400 in DexAggregatorWrapperWithPredicateProxy.sol (target link above).

## Remediation (high-level)

* Use the two-step allowance pattern when interacting with tokens that may implement non-standard `approve`: first set allowance to `0`, then set allowance to the desired amount.
* Alternatively, use `safeIncreaseAllowance` / `safeDecreaseAllowance` when supported, or detect token-specific behavior and handle allowances accordingly.
* Consider using permit/EIP-2612 flows where available to avoid on-chain approve races.
