# 53047 sc high the jackpot eligibility check uses stale storage data instead of the freshly calculated streak&#x20;

* Report ID: #53047
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts: Theft of unclaimed yield

## Description

### Brief / Intro

The `Spin.sol` contract has an incorrect update logic flaw in the `handleRandomness()` function where jackpot eligibility verification uses stale streak data from storage instead of the freshly calculated streak value used for reward determination. This inconsistency causes legitimate users to be wrongfully denied jackpot wins, particularly affecting first-time users and users rebuilding their streaks. The vulnerability undermines the core game mechanics and can result in significant economic losses for users who should have won high-value jackpot prizes.

### Vulnerability Details

The vulnerability exists in the `handleRandomness()` function in `Spin.sol`. The root cause is a temporal mismatch between streak calculation and streak validation:

```solidity
// Line 216: Calculate fresh streak for current spin
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);

// Line 218: Use fresh streak for reward determination
(string memory rewardCategory, uint256 rewardAmount) = determineReward(randomness, currentSpinStreak);

// Lines 225-232: Jackpot eligibility check uses STALE data
if (keccak256(bytes(rewardCategory)) == keccak256("Jackpot")) {
    uint256 currentWeek = getCurrentWeek();
    if (currentWeek == lastJackpotClaimWeek) {
        // ... weekly limit check ...
    } else if (userDataStorage.streakCount < (currentWeek + 2)) {  // ❌ USES OLD STREAK
        userDataStorage.nothingCounts += 1;
        rewardCategory = "Nothing";
        rewardAmount = 0;
        emit NotEnoughStreak("Not enough streak count to claim Jackpot");
    }
}
```

`userDataStorage.streakCount < (currentWeek + 2)` uses the previous spin's streak value stored in `userDataStorage.streakCount`, while the reward determination used `currentSpinStreak` which reflects the updated streak after the current spin.

The `_computeStreak()` function logic:

```solidity
function _computeStreak(address user, uint256 nowTs, bool justSpun) internal view returns (uint256) {
    uint256 streakAdjustment = justSpun ? 1 : 0;
    uint256 lastSpinTs = userData[user].lastSpinTimestamp;
    
    if (lastSpinTs == 0) {
        return 0 + streakAdjustment;  // First-time user returns 1
    }
    // ... consecutive day logic ...
    return 0 + streakAdjustment;  // Broken streak returns 1
}
```

When `justSpun = true`, this function adds 1 to account for the current spin, but the jackpot eligibility check doesn't see this updated value.

### Impact Details

Users who legitimately win jackpots based on correct streak calculations are denied their winnings.

## References

* plume/src/spin/Spin.sol
* Vulnerable function: `handleRandomness()`
* Streak calculation: `_computeStreak()`

## Proof of Concept

{% stepper %}
{% step %}

### Day 1 Setup

```solidity
// New user spins for first time
userData[user].streakCount = 0;
userData[user].lastSpinTimestamp = 0;

// After Day 1 spin completes:
userData[user].streakCount = 1;
userData[user].lastSpinTimestamp = day1_timestamp;
```

{% endstep %}

{% step %}

### Day 2

```solidity
// User spins next day (Week 0 - requires streak >= 2 for jackpot)
uint256 currentSpinStreak = _computeStreak(user, day2_timestamp, true);
// Returns: 1 + 1 = 2 (consecutive day bonus)

// Reward determination uses fresh streak
(rewardCategory, rewardAmount) = determineReward(randomness, 2); // currentSpinStreak = 2
// Returns: ("Jackpot", 5000) - User wins!

// Eligibility check uses stale data (BUG)
if (userDataStorage.streakCount < (currentWeek + 2)) {  // 1 < 2 = TRUE
    // DENIED! Uses old streak (1) instead of current streak (2)
    rewardCategory = "Nothing";
    emit NotEnoughStreak("Not enough streak count to claim Jackpot");
}

// Storage updated AFTER denial
userDataStorage.streakCount = currentSpinStreak; // Finally becomes 2
```

Result: User with legitimate 2-day streak is denied 5,000 PLUME jackpot because eligibility check uses the previous streak value (1) while reward calculation used the current streak (2).
{% endstep %}
{% endstepper %}
