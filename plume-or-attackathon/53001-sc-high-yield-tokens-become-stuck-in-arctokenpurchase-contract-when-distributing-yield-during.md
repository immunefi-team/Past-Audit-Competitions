# 53001 sc high yield tokens become stuck in arctokenpurchase contract when distributing yield during active sales

**Submitted on Aug 14th 2025 at 16:02:16 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53001
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

When distributing yield while a sale is active and there is no yield restriction, a portion of the yield will be transferred to the `ArcTokenPurchase` contract, where the yield will become stuck. Furthermore, legitimate holders will receive a lower share due to the `effectiveTotalSupply` including the tokens for sale.

### Vulnerability Details

Whenever ArcTokens are transferred, the address to which they are sent will be added to the `holders` set. Before an ArcToken can be enabled for sale, the amount that is going to be offered for sale needs to be sent to the `ArcTokenPurchase` contract. Otherwise, the following code in `ArcTokenPurchase::enableToken()` leads to a revert:

```solidity
if (
    ArcToken(_tokenContract).balanceOf(address(this)) < _numberOfTokens
) {
    revert ContractMissingRequiredTokens();
}
```

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L166-L170>

As a result, the `ArcTokenPurchase` contract is added to the `holders` set before a sale is activated. This means that when yield is distributed while a sale is active, the `effectiveTotalSupply` will include these tokens. Consequently, other holders will receive fewer yield tokens as a share of the whole amount is transferred to the `ArcTokenPurchase` contract, where they will become stuck as long as the yield token is not the same token as the purchase token.

To mitigate this, do not add the `ArcTokenPurchase` contract to the `holders` set.

### Impact Details

A share of the distributed yield is sent to the `ArcTokenPurchase` contract, where the yield tokens will become stuck. Furthermore, other holders will receive fewer yield tokens due to the `effectiveTotalSupply` including the ArcTokens that are for sale.

## References

Code references are provided throughout the report.

## Proof of Concept

{% stepper %}
{% step %}

### Step

ArcTokens are transferred to the `ArcTokenPurchase` contract so that a sale can be enabled. This adds the contract to the `holders` set.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L701-L703>
{% endstep %}

{% step %}

### Step

`ArcToken::distributeYield()` is called, which distributes yield to token holders. The ArcTokens in the `ArcTokenPurchase` contract will be included in the `effectiveTotalSupply` when the contract is not yield-restricted or no yield restriction module is set.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L418-L423>
{% endstep %}

{% step %}

### Step

The `ArcTokenPurchase` contract will receive its share of the yield token amount being distributed. As a result, other holders will receive fewer yield tokens.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L439-L445>
{% endstep %}

{% step %}

### Step

The yield tokens will become stuck in the `ArcTokenPurchase` contract when the yield token is different from the purchase token.
{% endstep %}
{% endstepper %}
