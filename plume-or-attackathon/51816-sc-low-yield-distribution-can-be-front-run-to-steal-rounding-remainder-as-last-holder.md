# 51816 sc low yield distribution can be front run to steal rounding remainder as last holder

**Submitted on Aug 5th 2025 at 23:00:18 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51816
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

When yield is distributed while ArcTokens are for sale, this call can be front-run by purchasing some of these ArcTokens. The last holder receives more yield than any other holder by default, even with the same amount, due to any remainder that occurs from rounding in the share calculation being sent to the last holder. Hence, it makes sense to be the last holder.

### Vulnerability Details

When `ArcToken::distributeYield()` is called, it distributes yield to token holders while skipping restricted accounts. Every token holder receives their share based on the following formula:

```solidity
uint256 share = (amount * holderBalance) / effectiveTotalSupply
```

Any remainder due to rounding is sent to the last token holder, with `distributedSum` being the sum of all previous shares:

```solidity
uint256 lastShare = amount - distributedSum
```

As a result, the last holder will most likely receive more yield than any other holder, even when they own the same amount of ArcTokens.

This opens up a vulnerability when ArcTokens are for sale during yield distribution, where anyone can front-run the call to distribute yield by buying tokens from the sale. The front-runner will end up being the last holder and receiving the remainder.

Furthermore, when the `ArcTokenPurchase` contract is yield-restricted, the `effectiveTotalSupply` will be increased by the front-run since the `effectiveTotalSupply` includes any ArcToken amount held by a holder that is not yield-restricted. As a result, other holders will receive fewer shares due to the front-run.

### Impact Details

* The front-runner steals the remainder that occurs due to rounding from the address that was the last holder before them.
* When the `ArcTokenPurchase` contract is yield-restricted, the `effectiveTotalSupply` will increase due to the front-run. As a result, other holders will receive a lower share than before the front-run.

## Proof of Concept

{% stepper %}
{% step %}

### Front-run scenario setup

An admin calls `ArcToken::distributeYield()` to distribute yield to token holders while a sale is active.

Relevant code:\
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>
{% endstep %}

{% step %}

### Attacker buys tokens (front-run)

Alice front-runs this call by calling `ArcTokenPurchase::buy()` and purchasing some of these ArcTokens. This transfers the `_purchaseAmount` of ArcTokens to the buyer. In the overridden `_update()` function, Alice is added to the holders set, which means that her address will be stored at the last index.

Relevant code:\
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L219-L283\\>
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L701-L703>
{% endstep %}

{% step %}

### Distribute yield and attacker collects remainder

`ArcToken::distributeYield()` is executed. Any remainder that occurs due to rounding errors in this formula:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L440>

is sent to the last holder:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L448-L457>

As a result, being the last holder is profitable. Additionally, when the `ArcTokenPurchase` contract is yield-restricted, other holders will receive fewer yield tokens.
{% endstep %}
{% endstepper %}

## References

Code references are provided above throughout the report.
