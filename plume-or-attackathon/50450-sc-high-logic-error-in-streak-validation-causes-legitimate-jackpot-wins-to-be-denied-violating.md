# 50450 sc high logic error in streak validation causes legitimate jackpot wins to be denied violating reward contract expectations

**Submitted on Jul 24th 2025 at 19:18:58 UTC by @Bug82427 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50450
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

In the Spin contract, jackpot eligibility is determined during `handleRandomness()` by checking whether `userDataStorage.streakCount` is greater than or equal to `(currentWeek + 2)`. However, this streak count reflects the user’s state before the current spin. Since `updateUserData()` is called after this check, users spinning their way into jackpot eligibility are denied their rightful rewards.

This is a classic logic flaw in state-dependent access control, where stale state is used to make a reward decision. The impact is that users who legitimately earn jackpot eligibility during their current spin will have their win silently discarded if they land on the jackpot, despite meeting all conditions.

While no funds are stolen or transferred, the jackpot — which should have been won — is withheld. The spin fee is still consumed and non-refundable. This results in broken reward mechanics and undermines user trust. The jackpot funds remain in the contract but the user’s reward is lost forever due to a flawed conditional check.

## Proof of Concept

Assumptions:

* `currentWeek = 3`
* Jackpot eligibility requires `streakCount ≥ currentWeek + 2 = 5`
* Player Alice has `streakCount = 4` before spinning
* Alice is about to spin and successfully lands on a jackpot

Execution trace (stepper):

{% stepper %}
{% step %}

### Alice starts the spin

* Alice calls `startSpin()`.
* She pays the 2 PLUME fee.
* A random request is sent to the Supra VRF.
* Her request ID is stored.
  {% endstep %}

{% step %}

### Oracle returns randomness

* The Supra oracle later calls `handleRandomness(...)` with a result corresponding to `rewardCategory = "Jackpot"`.
* Inside `handleRandomness()`, the contract retrieves Alice’s `userDataStorage`:
  * `streakCount == 4`
  * `lastSpinWeek == currentWeek == 3`
* During this spin, `streakCount` would be incremented to 5 by `updateUserData()`, but that update has not yet run.
  {% endstep %}

{% step %}

### Eligibility check uses stale state

* Before `updateUserData()` is called, the contract checks eligibility:

```solidity
if (userDataStorage.streakCount < (currentWeek + 2)) {
    revert InvalidSpin("Not enough streak");
}
```

* Since `streakCount == 4`, this check fails, even though the user would reach 5 during this spin.
  {% endstep %}

{% step %}

### Outcome

* Jackpot reward logic is skipped entirely, and the spin ends with a revert or lower reward.
* `updateUserData()` is only called after the reward condition is checked, so Alice is wrongly denied the jackpot.
  {% endstep %}
  {% endstepper %}

## Conclusion

State-dependent reward checks use a stale `streakCount` (pre-spin) before `updateUserData()` runs, which causes legitimate jackpot wins earned during the current spin to be denied. This breaks the reward mechanics: users pay the spin fee but may be incorrectly prevented from receiving a jackpot they earned that same spin.

### References

* Target contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
