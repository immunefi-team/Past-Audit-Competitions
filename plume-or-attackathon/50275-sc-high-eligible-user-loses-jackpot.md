# 50275 sc high eligible user loses jackpot

**Submitted on Jul 23rd 2025 at 09:04:01 UTC by @shadowHunter for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #50275
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

Any spin initiated before the start of `Week 12` is eligible for the jackpot. However, if a user spins `X` seconds before the `Week 12` cutoff but the Supra oracle callback arrives at—or after—the start of `Week 12`, the user does not receive the jackpot, even when the random result would otherwise qualify.

### Vulnerability Details

See the Proof of Concept section below.

### Impact Details

User will lose jackpot even when they were qualified.

### Recommendation

Week should be calculated from the spin time and not the callback time.

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

* User A calls `startSpin()` just before the end of **Week 11**, specifically at:

```
time = start of Week 12 - X seconds
```

* This makes User A eligible for a jackpot, since jackpot prizes are configured as:

```solidity
jackpotPrizes[0] ... jackpotPrizes[11]
```

{% endstep %}

{% step %}

### Oracle delay

* The Supra oracle processes the random number after some delay `Y`, where:

```
Y > X
```

* The callback therefore happens during Week 12.
  {% endstep %}

{% step %}

### Outcome and conditions

* Assume:
  * The returned random number qualifies for a jackpot.
  * The user has the required streak.
* Still, the user won't receive a jackpot because `weekNumber` is now 12 and:

```solidity
jackpotPrizes[12] // does not exist
```

{% endstep %}

{% step %}

### Relevant code snippet

```solidity
if (probability < jackpotThreshold) {
    return ("Jackpot", jackpotPrizes[weekNumber]);
}
```

Problem: Even though the spin was valid during Week 11, the reward is denied due to the delayed callback pushing `weekNumber` into Week 12, where no jackpot prize is configured.
{% endstep %}
{% endstepper %}
