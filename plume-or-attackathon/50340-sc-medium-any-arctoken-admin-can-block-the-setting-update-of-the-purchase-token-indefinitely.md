# 50340 sc medium any arctoken admin can block the setting update of the purchase token indefinitely&#x20;

* Submitted on Jul 23rd 2025 at 19:50:03 UTC by @avoloder for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #50340
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Impact category: Griefing (no direct profit motive for attacker, but damage to users or protocol)

## Description

### Brief / Intro

Any token admin can easily block setting the purchase token in ArcTokenPurchase.sol.

### Vulnerability Details

ArcTokenPurchase.sol allows token admins to enable their token sales by calling `enableToken()`, which adds the token to the `enabledTokens` enumerable set. When the owner of ArcTokenPurchase.sol wants to set or update the purchase token (the token users use to buy ArcTokens), the contract checks whether there are any ongoing sales by verifying if the `enabledTokens` set is empty.

This enables a malicious (or careless) arc token admin to grief the ArcTokenPurchase contract indefinitely by enabling a token sale (even with a very small amount), preventing the owner from setting or updating the purchase token. As a result the contract can become unusable:

* If the purchase token has never been set, users cannot buy ArcToken.
* If the purchase token needs to be updated later (e.g., regulatory reasons), it cannot be updated while any enabled token sale exists.
* There is no emergency mechanism to cancel ongoing sales or otherwise clear `enabledTokens`.

## Impact Details

Any arc token admin can grief ArcTokenPurchase.sol, preventing the contract owner from setting or updating the purchase token and rendering the purchase functionality inoperable.

## Recommendation

{% stepper %}
{% step %}

### Prevent enabling tokens when purchase token is unset

Make it impossible to call `enableToken()` (or otherwise add to `enabledTokens`) if the purchase token has not been set. This prevents token admins from blocking the initial configuration.
{% endstep %}

{% step %}

### Introduce expiration for enabled tokens

Add an expiration time (or sale duration) for enabled token sales so that forgotten or intentionally long-lived listings cannot permanently block updates. For example, when enabling a token record the enable timestamp and enforce an automatic expiry after X days unless renewed.
{% endstep %}

{% step %}

### Add emergency mechanics to cancel sales

Introduce an emergency function (access-controlled to the contract owner or a multisig/governance role) that can cancel all ongoing sales or clear `enabledTokens`, allowing the purchase token to be updated when necessary (e.g., sanctions, regulatory changes).
{% endstep %}
{% endstepper %}

## References

* Vulnerable code reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L293-L295>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario #1

1. ArcTokenPurchase.sol is deployed.
2. An ArcToken admin enables their token with any amount.
3. The contract owner cannot set the purchase token because `enabledTokens` is non-empty.
4. Nobody can buy ArcToken because the purchase token hasn't been set.
5. The contract is effectively dead/unusable.
   {% endstep %}

{% step %}

### Scenario #2

1. ArcTokenPurchase.sol is deployed.
2. Purchase token is set.
3. An ArcToken admin lists their token for sale at a very high price that nobody will buy.
4. Time passes and the sale remains enabled.
5. The purchase token needs to be updated (e.g., for compliance).
6. It is not possible to update the purchase token due to the ongoing sale, blocking the necessary update.
   {% endstep %}
   {% endstepper %}
