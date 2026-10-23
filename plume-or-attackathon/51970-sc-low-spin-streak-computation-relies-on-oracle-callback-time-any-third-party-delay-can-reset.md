# 51970 sc low spin streak computation relies on oracle callback time any third party delay can reset the user s streak and block jackpot eligibility&#x20;

**Submitted on Aug 6th 2025 at 22:12:19 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51970
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Users lose value due to a timing edge case

## Description

### Brief / Intro

`Spin.startSpin()` records the spin request but defers streak accounting until the VRF callback. If the Supra DVRF oracle has a partial delay that makes it answer after midnight of the day the spin was requested, `handleRandomness()` treats the spin as belonging to the next day, so `_computeStreak()` declares the previous‑day streak “broken” — even though the user had made a randomness request on the day that would maintain the streak. Users permanently lose their accumulated streak, raffle‑multiplier, and the ability to win the weekly jackpot for a certain period.

## Details

`startSpin()` stores `isSpinPending[user] = true` but does not touch `lastSpinTimestamp`. The function that updates the user streak, `_computeStreak()`, is called only when handling randomness during the Supra DVRF oracle callback. This function considers the streak unbroken only when `today == lastDaySpun + 1`. Because `block.timestamp` is evaluated when SupraRouter calls back and not when the spin request was made, any network / gas‑price congestion, operator throttling, or malicious delay that pushes the callback past the day boundary causes:

{% stepper %}
{% step %}
\_computeStreak() returns 1 instead of prevStreak+1.

(Callback occurs after midnight so the computed "today" is too large relative to lastDaySpun.)
{% endstep %}

{% step %}
userData\[user].streakCount is overwritten with the smaller value.

(The old streak is replaced on-chain when randomness is handled.)
{% endstep %}

{% step %}
Jackpot guard clauses revert the jackpot to "Nothing".

(A missed streak leads to ineligibility for the jackpot and reduced raffle multipliers.)
{% endstep %}
{% endstepper %}

The user's old streak is lost.

## Impact

{% hint style="info" %}

* Loss of value for honest users:
  * Jackpot prizes up to 100 000 PLUME can be lost for the victim.
  * Raffle‑ticket rewards are drastically reduced (`baseRaffleMultiplier × streak`).
* DoS / griefing: An attacker who controls or economically influences the SupraRouter queue can batch many spin requests near midnight and hold the callback until after 00:00, wiping streaks for all pending players.

While no funds are stolen from the contract, the bug causes direct, involuntary loss of user rewards by breaking a spin streak.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}
Setup

* Deploy `Spin` with campaign started.
* User Alice has `streakCount == 6` and `lastSpinTimestamp` set to 23:10 UTC of 14 Aug 2025.
  {% endstep %}

{% step %}
Exploit

* At 23:55 UTC on 15 Aug 2025 Alice calls `startSpin()`; `lastSpinTimestamp` is still 23:10 UTC (14 Aug).
* A network congestion delays the callback until 00:05 UTC on 16 Aug 2025.
  {% endstep %}

{% step %}
Observation

* `handleRandomness()` now computes:\
  `today = 00:05/86400 = 19690` and `lastDaySpun = 23:10/86400 = 19688`, so `today == lastDaySpun + 2`; the streak is treated as broken → returns `1`.
* Alice’s 7‑day streak is lost; she receives zero jackpot if the rng had determined it; and reduced prizes in the other cases.
  {% endstep %}

{% step %}
Result

* Direct monetary loss had she hit the jackpot for that week plus diminished raffle / PP rewards.
  {% endstep %}
  {% endstepper %}
