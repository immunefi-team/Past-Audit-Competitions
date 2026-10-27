# 52601 sc high in spin handlerandomness jackpot eligibility uses outdated streakcount instead of updated streak

Submitted on Aug 11th 2025 at 21:32:08 UTC by @Paludo0x for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52601
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Vulnerability Details

In Spin.sol, the jackpot eligibility check during `handleRandomness()` compares the older streak `userDataStorage.streakCount` against the required threshold `currentWeek + 2`, instead of the current streak at the moment of the spin `currentSpinStreak`.

The jackpot rule is: a player can get the jackpot only if their consecutive days streak ≥ (currentWeek + 2).

In `handleRandomness`, the contract computes the correct “today” streak `currentSpinStreak` but then checks eligibility against the old stored `streakCount`. This means a player could spin in consecutive days, stop for some days, and still be counted as eligible for the jackpot, even if they should not be because the actual streak is not consecutive to the previous ones.

Moreover it can wrongly reject players who reach the required streak with the current spin. Because the logic uses a value from before the current spin’s streak is incremented, it can produce false positives and false negatives for jackpot eligibility.

### Impact Details

This issue causes:

* False positives: If the stored streak is still high but not consecutive to the current spin, the player might be incorrectly eligible.
* False negatives: A player whose streak reaches the required threshold because of the current spin may be incorrectly denied the jackpot.

Therefore the yield could be attributed in an unfair manner if the jackpot is hit.

## Recommended fix

Use the streak that includes today’s spin when checking the rule streak ≥ (currentWeek + 2):

```solidity
if (currentSpinStreak < (currentWeek + 2)) {
    // deny jackpot
}
```

## Proof of Concept

This is the relevant snippet where the wrong check is performed:

{% code title="Spin.sol (relevant snippet)" %}

```solidity
function handleRandomness(uint256 nonce, uint256[] memory rngList) external onlyRole(SUPRA_ROLE) nonReentrant {
   ....
    // CURRENT STREAK IS COMPUTED
    uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
   ....
    if (keccak256(bytes(rewardCategory)) == keccak256("Jackpot")) {
        uint256 currentWeek = getCurrentWeek();
        if (currentWeek == lastJackpotClaimWeek) {
            userDataStorage.nothingCounts += 1;
            rewardCategory = "Nothing";
            rewardAmount = 0;
            emit JackpotAlreadyClaimed("Jackpot already claimed this week");
        // THE FORMER STREAK COUNT IS USED TO CHECK ELIGIBILITY
        } else if (userDataStorage.streakCount < (currentWeek + 2)) {
            userDataStorage.nothingCounts += 1;
            rewardCategory = "Nothing";
            rewardAmount = 0;
            emit NotEnoughStreak("Not enough streak count to claim Jackpot");
        } else {
            ....
        }
        // update the streak count after their spin
        userDataStorage.streakCount = currentSpinStreak;
        userDataStorage.lastSpinTimestamp = block.timestamp;
      .....     
}
```

{% endcode %}

<details>

<summary>Additional notes / rationale</summary>

Because the contract checks `userDataStorage.streakCount` (the stored value before the current spin) rather than `currentSpinStreak` (the value that accounts for the current spin), eligibility decisions can be incorrect in both directions. Replacing the check with `currentSpinStreak` when evaluating jackpot eligibility aligns the check with the intended rule and resolves both false positives and false negatives.

</details>
