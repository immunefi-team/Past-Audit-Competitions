# 50428 sc medium reverting on callback increases chances of winning

Submitted on Jul 24th 2025 at 14:12:28 UTC by @Oppi992 for Attackathon | Plume Network

* Report ID: #50428
* Report Type: Smart Contract
* Report severity: Medium
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Theft of gas
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

`handleRandomness()` pays out rewards immediately by calling `_safeTransferPlume()`. If the recipient is a contract that reverts on receiving the assets, the whole callback reverts. Since Supra dVRF re‑tries failed callback, and because `determineReward()` depends on the current `block.timestamp`, an attacker can delay the successful callback until a more favourable timestamp, or drain the sponsor’s gas budget.

### Vulnerability Details

On a successful spin, `handleRandomness()` calls `_safeTransferPlume()` to send the winning user the rewards via a low-level value transfer. Since this low-level call can be purposefully rejected, and since `_safeTransferPlume` requires the low-level call to succeed, a malicious user can decide to revert the entire callback (by spending all the available gas). In Supra dVRF V3, "Failed callback transactions (due to insufficient gas) are automatically retried every 6 hours, for up to 48 hours." (see Ref #1) Therefore, the user would get another "spin of the wheel" at a later time. Though the randomness would not change, the `block.timestamp` would. Apart from the obvious gas theft, this degree of freedom allows the user to improve their situation, on behalf of other users, as detailed below.

### Impact Details

{% stepper %}
{% step %}

### Wait until a better day of the week

A draw that only wins a Plume token on day D can win the jackpot on day D+1 (because the `jackpotProbabilities` are monotonically increasing). As winning Plume tokens still guarantees a call to `_safeTransferPlume()`, one can revert the callback and wait for a day that would yield the jackpot.
{% endstep %}

{% step %}

### Wait until next week

A draw that wins a small jackpot on week W can win a larger jackpot on week W+1 by delaying the callback until the later week.
{% endstep %}

{% step %}

### Avoid losing the jackpot

If in a week W the jackpot was already taken, naively leaving the user empty-handed, it is worthwhile to postpone the "spin" to next week and try to win next week's unclaimed jackpot.
{% endstep %}

{% step %}

### Gas drain

Always revert the transaction (if Plume tokens have been won) to repeatedly cause failed callbacks and steal gas from the sponsor who provided the gas funds for Supra.
{% endstep %}
{% endstepper %}

## References

<details>

<summary>Ref #1 — Supra dVRF V3 documentation</summary>

<https://docs.supra.com/dvrf/migration-to-dvrf-3.0#whats-new-in-dvrf-3.0>

</details>

## Proof of Concept

We give concrete demonstrations for each attack vector from "Impact Details". Assumptions: `jackpotProbabilities`, `jackpotPrizes`, and `plumeAmounts` are at their initial values; campaignStartDate is Monday at midnight.

{% stepper %}
{% step %}

### Wait until a better day of the week (Concrete)

Bob starts a spin on Tuesday, and he gets `randomness` of 4. This is not enough for the jackpot on Tuesday (`jackpotThreshold=3`), but Bob does get a Plume token. He purposefully reverts the entire transaction and waits until the callback is invoked on Thursday. Then the same `randomness` of 4 is below the Thursday `jackpotThreshold` (=7), and so Bob gets the jackpot.
{% endstep %}

{% step %}

### Wait until next week (Concrete)

Bob starts a spin on Sunday in the second week, and gets a `randomness` of 0. This is below the `jackpotThreshold` and so Bob is entitled to 5,000 Plume tokens. However, he can revert the callback and wait until Monday (beginning of the third week), where this `randomness` would net him 10,000 tokens.
{% endstep %}

{% step %}

### Avoid losing the jackpot (Concrete)

Bob starts a spin on Sunday in the first week, and gets a `randomness` of 0. This is below the `jackpotThreshold`, but the jackpot was already claimed, so Bob is entitled to 0 tokens. `handleRandomness()` would still call `_safeTransferPlume()` with 0 tokens, so Bob can revert the callback and wait until Monday (beginning of the second week), where this `randomness` would net him 5,000 tokens.
{% endstep %}

{% step %}

### Gas drain (Concrete)

Bob starts a spin on Tuesday, and he gets `randomness` of 1000, which nets him one Plume token. He decides to revert this transaction to waste gas for the sponsor. After six hours, the callback is retried, and Bob reverts again. This can repeat up to 8 times, wasting a considerable amount of gas fees paid by the sponsor.
{% endstep %}
{% endstepper %}
