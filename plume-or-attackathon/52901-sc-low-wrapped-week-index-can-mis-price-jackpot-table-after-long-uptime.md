# 52901 sc low wrapped week index can mis price jackpot table after long uptime

**Submitted on Aug 14th 2025 at 08:01:53 UTC by @spongebob for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52901
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Theft of unclaimed yield

## Description

The `Spin` contract has an underflow/wrap bug in the `determineReward()` function that allows unauthorized distribution of jackpot prizes after week 255.

Relevant code references:

* determineReward location:\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L286>
* casting current week to uint8:\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L197-L199>

`getCurrentWeek()` returns a uint256 current campaign week, but this value is cast to `uint8` when used to index jackpot prizes. Because `uint8` max is 255, any week ≥ 256 wraps (e.g., 256 -> 0, 257 -> 1). The wrapped value is then used to access `jackpotPrizes[weekNumber]`, which can return prizes from early campaign weeks when the campaign should have ended.

The campaign is designed to last only 12 weeks (weeks 0–11), as shown by `getWeeklyJackpot()` returning zero for weeks beyond 11:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L425-L427>

Jackpot prizes are only initialized for weeks 0–11 during contract initialization:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L107-L118>

This creates an inconsistency: jackpot eligibility checks correctly use the full uint256 week value for weekly limits, but prize selection uses the wrapped uint8 value, allowing users to receive jackpot prizes when the campaign should be over.

## Impact

After week 255, users can win jackpot prizes that should remain unclaimed, since the campaign should have ended after week 11. Because the contract tracks `lastJackpotClaimWeek` (using the full uint256 week), this allows one jackpot payout per (wrapped) week, enabling a continuous drain on contract funds over time.

## Proof of Concept

Setup:

* The Spin contract is deployed and initialized with jackpot prizes for weeks 0–11.
* The campaign has been running for 255+ weeks (≈ 5+ years).
* An attacker has sufficient streak count and funds to spin.

{% stepper %}
{% step %}

### Verify Current Week State

* Confirm `getCurrentWeek()` returns a value ≥ 256 (e.g., 256).
* Verify `getWeeklyJackpot()` correctly returns `(256, 0, 0)` (no jackpot should be available).
  {% endstep %}

{% step %}

### User Initiates Spin

* User calls `startSpin()` with correct payment.
* Contract accepts the spin because basic eligibility checks pass.
* An oracle request is generated and pending.
  {% endstep %}

{% step %}

### Oracle Callback Triggers Bug

* Oracle calls `handleRandomness()` with a low random number (e.g., 0).
* `determineReward()` is invoked with this randomness.
  {% endstep %}

{% step %}

### Week Number Wrapping

* `getCurrentWeek()` returns 256.
* Casting to `uint8` wraps 256 to 0.
* `weekNumber` becomes 0 instead of 256.
  {% endstep %}

{% step %}

### Jackpot Prize Access

* Code accesses `jackpotPrizes[0]` (week 0 prize) instead of `jackpotPrizes[256]`.
* Returns week 0's jackpot prize (e.g., 5000 tokens) instead of 0.
  {% endstep %}

{% step %}

### Eligibility Bypass

* The eligibility checks in `handleRandomness()` use the correct `uint256` week value.
* `currentWeek` (256) ≠ `lastJackpotClaimWeek` (likely < 256), so the weekly limit check passes.
* If the user meets the streak requirement, jackpot distribution proceeds.
  {% endstep %}

{% step %}

### Unauthorized Reward Distribution

* User receives the week 0 jackpot (e.g., 5000 PLUME).
* `lastJackpotClaimWeek` is updated to 256.
* Contract transfers tokens that should not have been distributed.
  {% endstep %}

{% step %}

### Repeated Exploitation

* The process can repeat weekly as `getCurrentWeek()` keeps incrementing.
* Week 257 wraps to 1 (returns week 1 prize), week 258 wraps to 2, etc.
* This leads to a continuous unauthorized drain of protocol funds.
  {% endstep %}
  {% endstepper %}

## References

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
