# 52948 sc low jackpot reward rejected at exact threshold

**Submitted on Aug 14th 2025 at 13:01:45 UTC by @Am3nh3l for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52948
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief/Intro\
In Spin.sol, the determineReward function uses `<` instead of `<=` for the jackpot threshold check, rejecting the jackpot when probability equals `jackpotThreshold`, unfairly reducing the jackpot probability.

Vulnerability Details

```solidity
        if (probability < jackpotThreshold) {
            return ("Jackpot", jackpotPrizes[weekNumber]);
        } else if (probability <= rewardProbabilities.plumeTokenThreshold) {
            uint256 plumeAmount = plumeAmounts[probability % 3];
            return ("Plume Token", plumeAmount);
        } else if (probability <= rewardProbabilities.raffleTicketThreshold) {
            return ("Raffle Ticket", baseRaffleMultiplier * streakForReward);
        } else if (probability <= rewardProbabilities.ppThreshold) {
            return ("PP", PP_PerSpin);
        }
```

Impact Details\
When `probability == jackpotThreshold`, the jackpot is skipped and the user receives a Plume Token reward instead, reducing the effective jackpot probability (for example, from 0.001% to 0.0009% for `threshold = 1`). Users are unfairly denied jackpots, potentially losing significant rewards (e.g., 5,000–100,000 PLUME).

## References

determineReward uses `<=` for other thresholds, inconsistent with `<` for jackpot.

Code snippet:

```solidity
if (probability < jackpotThreshold) {
    return ("Jackpot", jackpotPrizes[weekNumber]);
} else if (probability <= rewardProbabilities.plumeTokenThreshold) {
    // Plume Token awarded
```

## Proof of Concept

{% stepper %}
{% step %}

### Scenario

* Suppose `jackpotThreshold = 10`
* RNG result: `probability = 10`
  {% endstep %}

{% step %}

### What happens

* `10 < 10` fails → not jackpot
* Falls through to Plume Token condition (`10 <= 200000` succeeds)
* User wins Plume Token instead of jackpot
  {% endstep %}

{% step %}

### Expected behavior

* The jackpot should cover `[0, 10]` (11 values) when using an inclusive threshold, but with `<` it only covers `[0, 9]` (10 values), reducing the effective jackpot chance.
  {% endstep %}
  {% endstepper %}
