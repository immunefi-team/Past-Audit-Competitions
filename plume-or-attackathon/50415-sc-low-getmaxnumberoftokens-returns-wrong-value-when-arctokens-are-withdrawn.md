# 50415 sc low getmaxnumberoftokens returns wrong value when arctokens are withdrawn

Submitted on Jul 24th 2025 at 11:34:57 UTC by @KlosMitSoss for Attackathon | Plume Network

* Report ID: #50415
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief/Intro\
The `getMaxNumberOfTokens()` function returns the maximum number of ArcTokens that can be bought. However, this function returns incorrect values when ArcTokens have been withdrawn.

## Vulnerability Details

`getMaxNumberOfTokens()` returns the difference between `totalAmountForSale` and `amountSold`. This does not account for ArcTokens that were withdrawn. When tokens are withdrawn, they are no longer available for purchase, but the function continues to count them as available, leading to an overestimation of purchasable tokens.

The function should subtract withdrawn tokens from the available supply to provide accurate information about how many tokens can actually be purchased.

## Impact Details

`getMaxNumberOfTokens()` returns wrong values.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Call `ArcTokenPurchase::enableToken()` to enable 100e18 ArcTokens for sale.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L142-L178>
{% endstep %}

{% step %}

### Step

Call `ArcTokenPurchase::withdrawUnsoldArcTokens()` to withdraw 10e18 ArcTokens.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L439-L461>
{% endstep %}

{% step %}

### Step

Observe that `ArcTokenPurchase::getMaxNumberOfTokens()` still returns 100e18 even though only 90e18 are actually available for sale.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L359-L364>
{% endstep %}
{% endstepper %}

## References

* enableToken(): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L142-L178>
* withdrawUnsoldArcTokens(): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L439-L461>
* getMaxNumberOfTokens(): <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L359-L364>
