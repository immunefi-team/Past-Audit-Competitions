# 51041 sc high streak count misuse in jackpot eligibility allows theft of user funds

**Submitted on Jul 30th 2025 at 16:18:47 UTC by @Ambitious\_DyDx for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51041
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds

## Description

### Brief/Intro

I have observed that in the `handleRandomness` function, the contract checks or verifies jackpot eligibility against the established pre-spin streak (`userDataStrorage.streakCount`) instead of the established newly computed streak (`currentSpinStreak`). A user who has sufficient streak on that spin is incorrectly denied their jackpot payout, diverting their reward into the contract permanently.

### Vulnerability Details

In `handleRandomness` function, where the jackpot branch resides, the code does:

```solidity
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
// …
if (keccak256(bytes(rewardCategory)) == keccak256("Jackpot")) {
    uint256 currentWeek = getCurrentWeek();
    //  I believe the contract uses old streakCount, not currentSpinStreak:
    if (userDataStorage.streakCount < (currentWeek + 2)) {
        userDataStorage.nothingCounts += 1;
        rewardCategory = "Nothing";
        rewardAmount   = 0;
        emit NotEnoughStreak("Not enough streak count to claim Jackpot");
    } else {
        userDataStorage.jackpotWins++;
        lastJackpotClaimWeek = currentWeek;
    }
}
// …
// only _after_ the check do we write:
userDataStorage.streakCount = currentSpinStreak;
```

The vulnerability arises because the on-chain streak is not updated until after the eligibility test. A user who reaches the required streak on the current spin still has the prior stored streak and is treated as ineligible.

### Impact Details

* The vulnerability can result in the permanent denial of jackpot payouts (e.g., weekly jackpot up to 50,000 PLUME or more) to eligible users.
* The denied payout remains locked in the contract and is irrecoverable.
* This results in direct loss of user funds.

### References

* <https://immunefisupport.zendesk.com/hc/en-us/articles/33260632501777-Proof-of-Concept-Rules-for-Audit-Competitions>
* <https://immunefi.com/audit-competition/plume-network-attackathon/scope/#top>
* <https://immunefi.com/audit-competition/plume-network-attackathon/information/#top>
* <http://docs.soliditylang.org/>

## Proof of Concept

The following outlines the proof of concept of the vulnerability identified.

{% stepper %}
{% step %}

### Initiate the Setup

* Deploy `spin` with a known `campaignStartDate` so that `getCurrentWeek()` returns the expected value (>=0) and required streak (= currentWeek + 2).
* Whitelist the attacker address or fund the contract with sufficient PLUME.
  {% endstep %}

{% step %}

### Spin until threshold

* Call `startSpin()` and simulate the VRF callback with a value that triggers `"Jackpot"` on the final qualifying day.
* Example: for week 0 required streak = 2. Spin Day 1 (streak -> 1), Spin Day 2 (streak -> 2).
  {% endstep %}

{% step %}

### Observe the denial

* On the qualifying day (Day 2), `determineReward` returns `("jackpot", prize)` but `userDataStorage.streakCount` is still 1.
* The contract takes the "Not enough streak" branch:
  * No `SpinCompleted` with `"jackpot"` emitted.
  * User's PLUME balance unchanged.
  * Contract's balance increases by the jackpot amount.
    {% endstep %}

{% step %}

### Conclusion

* Although the user's streak logically reached the threshold this spin, the contract uses the old stored streak and permanently denies the payout.
  {% endstep %}
  {% endstepper %}

## Recommendation

Replace the check that uses the stored streak with the freshly computed streak before the stored value is updated. For example:

```diff
-    if (userDataStorage.streakCount < (currentWeek + 2)) {...
+    if (currentSpinStreak       < (currentWeek + 2)) {...
```

This ensures eligibility is evaluated using the up-to-date streak value for the current spin.

PS: The proof of concept is based on my understanding of the project's workflow. Thank you.
