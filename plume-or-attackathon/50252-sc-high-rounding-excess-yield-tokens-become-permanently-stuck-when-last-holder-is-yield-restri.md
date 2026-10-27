# 50252 sc high rounding excess yield tokens become permanently stuck when last holder is yield restricted

**Submitted on Jul 23rd 2025 at 00:12:16 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50252
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

When yield is distributed to token holders, every holder that is not yield-restricted receives their proportional share. However, any excess amount that occurs due to rounding is always sent to the last holder in the list, regardless of whether they are yield-restricted. If the last holder is yield-restricted, this remaining amount becomes permanently stuck in the contract.

### Vulnerability Details

The `distributeYield()` function iterates through the entire set of token holders and transfers the appropriate yield share to each eligible address. An address is only eligible if it is not yield-restricted.

To account for rounding errors, the function sends the difference between the provided yield token amount and the actual distributed yield token amount to the last holder in the holders array. The issue lies in the fact that this excess is always sent to the last holder, not the last eligible holder. When the last holder is yield-restricted, they cannot receive the excess yield tokens, causing these tokens to remain permanently stuck in the contract.

Suggested mitigation (as described in the report): track the sum of all holderBalances of holders that have already received their share (only those that are not yield-restricted). When this sum equals the `effectiveTotalSupply`, send the excess yield tokens to that holder (the last eligible recipient) instead of the last holder in the array.

### Impact Details

Any leftover amount due to rounding errors remains permanently stuck in the contract when the last holder is yield-restricted, resulting in a permanent loss of yield tokens that should have been distributed to eligible holders.

## References

Code references are provided throughout the report (example locations in the repository):

* distributeYield implementation: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>
* integer division rounding in calculation: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L440>
* leftover transfer to last holder: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L450-L456>

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce step 1

Call `distributeYield()` to distribute yield to all token holders that are not yield-restricted.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>
{% endstep %}

{% step %}

### Reproduce step 2

The calculation of each holder's yield share uses integer division, which rounds down:

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L440>

The leftover amount from rounding is then transferred to the last holder in the array. If the last holder is yield-restricted, these yield tokens cannot be transferred and remain permanently stuck in the contract.

Reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L450-L456>
{% endstep %}
{% endstepper %}
