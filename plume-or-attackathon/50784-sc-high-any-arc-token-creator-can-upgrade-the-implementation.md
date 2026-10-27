# 50784 sc high any arc token creator can upgrade the implementation&#x20;

**Submitted on Jul 28th 2025 at 14:40:31 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #50784
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Unauthorized upgrade of Arc Token implementation

{% hint style="danger" %}
High severity: attackers with the described capability can permanently compromise user funds and token behavior.
{% endhint %}

## Description

### Brief / Intro

The DEFAULT\_ADMIN\_ROLE is the (implicit) admin of UPGRADER\_ROLE, so every Arc‑token creator can instantly grant themselves upgrade power, swap in malicious logic, and steal or freeze purchaser funds. Since the Arc-token is portrayed as an official and trusted token implementation for Plume, this behavior is unlikely to be removed from the official UI before mitigations are implemented.

### Vulnerability Details

When ArcTokenFactory initializes a new token it passes msg.sender as DEFAULT\_ADMIN\_ROLE (see: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol#L193>). In OpenZeppelin AccessControlUpgradeable, every custom role’s admin defaults to DEFAULT\_ADMIN\_ROLE unless explicitly overridden. Therefore the creator can immediately call grantRole(UPGRADER\_ROLE, self) and gain unrestricted control over the proxy’s implementation — the correct flow, however, would be to only allow the token factory to retain this role (see: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol#L200>).

This enables the ArcToken creator to grant themselves the UPGRADER\_ROLE.

### Impact Details

Any ArcToken creator is able to modify their created token implementation. This enables exploitative behavior in multiple fronts:

* Modified transfer implementations can permanently freeze user funds.
* Modified balance accounting can enable funds to be removed or added arbitrarily.
* ArcToken purchases can be honeypot: malicious arc token providers can create sales with excessive yield to lure users into purchasing their tokens, receive the earnings from the offering, then rug-pull the token by changing its implementation and claiming the whole yield back to themselves.

## Proof of Concept

{% stepper %}
{% step %}

### Create a token

Someone creates an Arc token.
{% endstep %}

{% step %}

### Grant upgrader

The creator then grants themself the UPGRADER\_ROLE.
{% endstep %}

{% step %}

### Execute malicious upgrade

The creator can decide when to act maliciously. For example, they can wait for volume on the token, execute honeypot actions to attract purchases, then upgrade the implementation to a malicious one that:

* Freezes transfers,
* Reassigns or mints balances to the attacker,
* Or otherwise drains funds.
  {% endstep %}
  {% endstepper %}

## References

* Contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
