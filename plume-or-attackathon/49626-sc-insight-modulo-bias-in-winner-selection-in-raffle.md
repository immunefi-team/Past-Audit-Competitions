# 49626 sc insight modulo bias in winner selection in raffle

**Submitted on Jul 17th 2025 at 19:12:26 UTC by @Opzteam for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49626
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Vulnerability Description

The Raffle contract contains a modulo bias vulnerability in the winner selection mechanism. The contract uses a simple modulo operation without bias protection, which can favor certain ticket indices when the total number of tickets doesn't evenly divide the VRF random number space (1,000,000), leading to unfair raffle outcomes.

### Vulnerable Code Snippet

```solidity
// Line 247 in handleWinnerSelection function
uint256 winningTicketIndex = (rng[0] % totalTickets[prizeId]) + 1;
```

When `totalTickets[prizeId]` doesn't evenly divide 1,000,000, some ticket indices get more chances than others.

Example: If totalTickets = 7, then 1,000,000 % 7 = 6, meaning tickets 1-6 get 142,858 chances each while ticket 7 only gets 142,852 chances.

## Proof of Concept

{% stepper %}
{% step %}
Create a raffle with 100,001 total tickets
{% endstep %}

{% step %}
Calculate bias: 1,000,000 % 100,001 = 999,999 (significant remainder)
{% endstep %}

{% step %}
Exploit: Tickets 1-999,999 get 10 chances each, ticket 1,000,000 gets 9 chances, ticket 100,001 gets 1 chance
{% endstep %}

{% step %}
Result: Users with tickets in positions 1-999,999 have higher winning probability than those in positions 1,000,000 and 100,001
{% endstep %}

{% step %}
Manipulation: Attacker can strategically time entries or manipulate ticket purchases to favor advantageous positions
{% endstep %}
{% endstepper %}

## Remediation

{% hint style="info" %}
Implement rejection sampling to ensure uniform distribution.
{% endhint %}

Recommended code pattern:

```solidity
uint256 maxValidValue = (type(uint256).max / totalTickets[prizeId]) * totalTickets[prizeId];
uint256 randomValue;
do {
    randomValue = rng[0];
} while (randomValue >= maxValidValue);
uint256 winningTicketIndex = (randomValue % totalTickets[prizeId]) + 1;
```

This ensures all ticket indices have exactly equal probability of being selected.
