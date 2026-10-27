# 51146 sc low getmaxnumberoftokens returns wrong max number of tokens available to buy

**Submitted on Jul 31st 2025 at 15:10:10 UTC by @Oxgritty for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51146
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>

## Description

### Brief / Intro

* `getMaxNumberOfTokens` is used to know the quantity of tokens available to buy.
* It returns a wrong amount because `totalAmountForSale` is not updated when `withdrawUnsoldArcTokens` is used to withdraw unsold ARC tokens.

### Vulnerability Details

* `getMaxNumberOfTokens` relies on `info.totalAmountForSale` to compute the available balance.
* `withdrawUnsoldArcTokens` withdraws ARC tokens but does not update `info.totalAmountForSale`.
* As a result, `getMaxNumberOfTokens` can return an amount greater than the actual token balance held by the contract.

## Impact Details

* A buyer calling `getMaxNumberOfTokens` may see an inflated available amount.
* If the buyer attempts to purchase that inflated amount via `buy`, the `buy` call will revert due to `ContractBalanceInsufficient`.

## References

1. `getMaxNumberOfTokens` function:\
   <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L359>
2. `withdrawUnsoldArcTokens` function:\
   <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L439>
3. `buy` function revert due to insufficient balance:\
   <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L263>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Token Admin calls `enableToken` to register a token for sale with `totalAmountForSale = 500e18`.
{% endstep %}

{% step %}

### Withdraw unsold tokens

Token Admin calls `withdrawUnsoldArcTokens` to withdraw `50e18` tokens. (Contract token balance is now 450e18, but `info.totalAmountForSale` remains 500e18.)
{% endstep %}

{% step %}

### Buyer queries available tokens

Buyer calls `getMaxNumberOfTokens`.\
The function returns `500e18` (based on `info.totalAmountForSale`), which is higher than the actual balance.
{% endstep %}

{% step %}

### Buy attempt fails

Buyer calls `buy` with amount `500e18`.\
The `buy` call reverts due to `ContractBalanceInsufficient`.
{% endstep %}
{% endstepper %}
