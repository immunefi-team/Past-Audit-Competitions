# 53011 sc critical uncleaned partial approval consumption in dex aggregator integration leads to permanent dos

**Submitted on Aug 14th 2025 at 16:38:29 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53011
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The DexAggregatorWrapperWithPredicateProxy contract fails to clean up token approvals after DEX swaps, creating a critical vulnerability when aggregators consume less than the approved amount.

Since 1inch and OKX aggregators support partial fills as a documented feature, lingering approvals will cause safeApprove to revert on all subsequent transactions for affected tokens, permanently disabling the contract.

## Vulnerability Details

The vulnerability exists in the approval pattern used in both `_oneInchHelper` and `_okxHelper` functions.

When the contract approves tokens for swapping, it does not verify that the full approved amount was consumed, nor does it reset approvals to zero after the swap.

Vulnerable Code in `_oneInchHelper` (lines 267-271):

{% code title="DexAggregatorWrapperWithPredicateProxy.sol (excerpt)" %}

```solidity
// Approve aggregator for full amount
depositAsset.safeApprove(address(aggregator), depositAmount);

// Perform swap - ignores spentAmount return value!
(supportedAssetAmount,) = aggregator.swap(executor, desc, data);
// No cleanup of remaining approval
```

{% endcode %}

Vulnerable Code in `_okxHelper` (lines 321-333):

{% code title="DexAggregatorWrapperWithPredicateProxy.sol (excerpt)" %}

```solidity
// Approve OKX for full amount
depositAsset.safeApprove(okxApprover, fromTokenAmount);

// Execute swap
(bool success, bytes memory result) = address(okxRouter).call(okxCallData);
// No verification of actual consumption or cleanup
```

{% endcode %}

The 1inch AggregationRouterV6 explicitly supports partial fills through the `_PARTIAL_FILL` flag, returning a `spentAmount` that can be less than the approved amount. This is a documented feature for handling low liquidity scenarios.

When a swap consumes only 80% of the approved tokens (common during slippage optimization), 20% approval remains active.

The critical issue arises because `SafeERC20.safeApprove()` reverts when attempting to change a non-zero approval to another non-zero value:

```solidity
// Next user attempting to swap the same token
depositAsset.safeApprove(address(aggregator), newAmount); // REVERTS!
// Error: "SafeERC20: approve from non-zero to non-zero allowance"
```

## Impact Details

* Once triggered, the contract becomes permanently unusable for affected tokens.
* All users attempting to swap the same token will have transactions revert.
* No recovery mechanism exists — the contract lacks owner functions to reset approvals.
* Can be triggered accidentally through normal usage or deliberately by an attacker.

{% hint style="danger" %}
Severity: Critical — leads to permanent DoS for affected tokens.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Initial State

* DexAggregatorWrapperWithPredicateProxy is deployed
* Users are swapping tokens normally through the wrapper
  {% endstep %}

{% step %}

### Trigger Partial Consumption

* User A initiates swap of 1000 USDC for ETH via 1inch
* Contract approves 1000 USDC to aggregator (line 267)
* 1inch aggregator encounters slippage/optimization and only consumes 800 USDC
* Swap completes successfully, but 200 USDC approval remains
  {% endstep %}

{% step %}

### DoS Triggered

* User B attempts to swap USDC for any token
* Contract calls `depositAsset.safeApprove(aggregator, amount)` (line 267)
* Transaction reverts with "SafeERC20: approve from non-zero to non-zero allowance"
* All subsequent USDC swaps fail permanently
  {% endstep %}
  {% endstepper %}
