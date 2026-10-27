# 49963 sc medium anyone can create an arctoken and block the setpurchasetoken function

**Submitted on Jul 20th 2025 at 20:13:10 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49963
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

`ArcTokenPurchase::setPurchaseToken()` can only be called when there are no active token sales. However, since token sales do not automatically become disabled when all tokens are sold or when the token admin has withdrawn all tokens, any token admin could block `ArcTokenPurchase::setPurchaseToken()` by deliberately not disabling their sale even though there are no tokens left to be sold.

### Vulnerability Details

Since `ArcTokenFactory::createToken()` can be called by anyone, anyone can become a token admin of a token by simply creating one. After that, the token admin can transfer ArcTokens intended for sale to the `ArcTokenPurchase` contract and call `enableToken()` on `ArcTokenPurchase` for the specific ArcToken, setting the price and quantity.

When all of those tokens are sold, the sale for the token will not be disabled automatically. The same applies when the token admin withdraws all of the tokens. As a result, the token will remain enabled until the token admin calls `disableToken()`. Note that `disableToken()` can only be called by the specific token admin.

This leads to issues where any token admin can intentionally or unintentionally block calls to `setPurchaseToken()` by simply not calling `disableToken()`, as this function reverts when there are any active sales. To mitigate this, the sale should automatically become disabled when all tokens are sold or withdrawn.

### Impact Details

`ArcTokenPurchase::setPurchaseToken()` will be permanently blocked, which means that the purchase token can neither be set (if no purchase token has been set before) nor changed. This becomes especially critical when the current purchase token is compromised and needs to be changed.

## Proof of Concept

{% stepper %}
{% step %}

### Step

A token admin transfers ArcTokens intended for sale to the `ArcTokenPurchase` contract.
{% endstep %}

{% step %}

### Step

The token admin calls `enableToken()` for the specific ArcToken, setting the price and quantity. (See: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L142-L178>). The token is added to the `enabledTokens` set.
{% endstep %}

{% step %}

### Step

The token admin calls `withdrawUnsoldArcTokens()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L439-L461>) to withdraw all of the ArcTokens again. Alternatively, all of the ArcTokens could be bought (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L219-L283>).
{% endstep %}

{% step %}

### Step

The token admin intentionally or unintentionally does not call `disableToken()`. The token remains enabled even though there are no tokens left to be bought.
{% endstep %}

{% step %}

### Step

As a result, `setPurchaseToken()` cannot be called anymore (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L293-L295>).
{% endstep %}
{% endstepper %}

## References

Code references are provided throughout the report.
