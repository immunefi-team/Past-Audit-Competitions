# 52576 sc high flaw in raffle determinereward in jackpot prize calculation after week 12

**Submitted on Aug 11th 2025 at 18:14:24 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52576
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

There is a logical flaw in the jackpot mechanism where, after week 12, the `jackpotPrizes` mapping will return 0 for any jackpot prize amount.

This results in a situation where a jackpot can be “won” with reward = 0, wasting the winning outcome and reducing the probability for winning other reward types.

### Vulnerability Details

The jackpot mechanism references `jackpotPrizes[week]` to determine the prize amount.

However, `jackpotPrizes` is a mapping, not an array, and only contains defined values for weeks 0 through 11 (twelve entries). Any week index beyond those defined entries will return 0 by default, without reverting.

This means that when `week > 11` (i.e., after week 12 counting from zero), a jackpot “hit” will still be registered and processed, but the prize value will be zero.

### Impact Details

* The jackpot can be “won” after week 12, but the prize is 0.
* This reduces the probability pool for other possible rewards since the jackpot branch consumes RNG outcome space, effectively lowering other reward chances (fairness impact).
* Players may believe they have won a jackpot when in fact they receive nothing.

## Recommended Fix

{% stepper %}
{% step %}

### Keep jackpot prize constant after defined weeks

If week is greater than the last defined jackpot prize week, keep the jackpot prize constant (for example, equal to the last defined prize). This ensures jackpot winners after week 12 still receive the intended prize amount instead of 0.
{% endstep %}

{% step %}

### Revert jackpot attempts beyond defined weeks

Explicitly revert when a jackpot hit occurs for `week > 12` so that no jackpot is awarded and the call reverts instead of registering a zero-value win. This prevents consuming the RNG outcome space with a zero-value jackpot.
{% endstep %}

{% step %}

### Skip jackpot outcome selection for later weeks

Skip selecting jackpot outcomes for `week > 12` entirely: do not increment `jackpotWins`, do not update `lastJackpotClaimWeek`, and treat the RNG outcome as if the jackpot branch did not occur (so other reward branches remain available).
{% endstep %}
{% endstepper %}

## Proof of Concept

In `Raffle::determineReward()` the jackpot prize is determined based on number of weeks since campaign start:

```solidity
function determineReward(
    uint256 randomness,
    uint256 streakForReward
) internal view returns (string memory, uint256) {
   ...
    // Determine the current week in the 12-week campaign
    uint256 daysSinceStart = (block.timestamp - campaignStartDate) / 1 days;
    uint8 weekNumber = uint8(getCurrentWeek());
    uint8 dayOfWeek = uint8(daysSinceStart % 7);

    // Get jackpot threshold for the day of week
    uint256 jackpotThreshold = jackpotProbabilities[dayOfWeek];

    if (probability < jackpotThreshold) {
        return ("Jackpot", jackpotPrizes[weekNumber]);
    } 
    ...
}
```

`getCurrentWeek()` is calculated as follows:

```solidity
function getCurrentWeek() public view returns (uint256) {
    return (block.timestamp - campaignStartDate) / 7 days;
}
```

While `jackpotPrizes` is initialized as follows:

```solidity
    jackpotPrizes[0] = 5000;
    jackpotPrizes[1] = 5000;
    jackpotPrizes[2] = 10_000;
    jackpotPrizes[3] = 10_000;
    jackpotPrizes[4] = 20_000;
    jackpotPrizes[5] = 20_000;
    jackpotPrizes[6] = 30_000;
    jackpotPrizes[7] = 30_000;
    jackpotPrizes[8] = 40_000;
    jackpotPrizes[9] = 40_000;
    jackpotPrizes[10] = 50_000;
    jackpotPrizes[11] = 100_000;
```

In function `handleRandomness` there is no check that the current week falls within the range of defined jackpot weeks; storage is updated as a common jackpot win but no rewards are distributed to the winner when prize resolves to 0:

```solidity
function handleRandomness(uint256 nonce, uint256[] memory rngList) external onlyRole(SUPRA_ROLE) nonReentrant {
   ...

    // ----------  Effects: update storage first  ----------
    if (keccak256(bytes(rewardCategory)) == keccak256("Jackpot")) {
        uint256 currentWeek = getCurrentWeek();
        if (currentWeek == lastJackpotClaimWeek) {
            userDataStorage.nothingCounts += 1;
            rewardCategory = "Nothing";
            rewardAmount = 0;
            emit JackpotAlreadyClaimed("Jackpot already claimed this week");
        } else if (userDataStorage.streakCount < (currentWeek + 2)) {
            userDataStorage.nothingCounts += 1;
            rewardCategory = "Nothing";
            rewardAmount = 0;
            emit NotEnoughStreak("Not enough streak count to claim Jackpot");
        } else {
            userDataStorage.jackpotWins++;
            lastJackpotClaimWeek = currentWeek;
        }
    } 
    ...

    // ----------  Interactions: transfer Plume last ----------
    if (
        keccak256(bytes(rewardCategory)) == keccak256("Jackpot")
            || keccak256(bytes(rewardCategory)) == keccak256("Plume Token")
    ) {
        _safeTransferPlume(user, rewardAmount * 1 ether);
    }

    emit SpinCompleted(user, rewardCategory, rewardAmount);
}
```

Because `jackpotPrizes[weekNumber]` returns 0 for weeks beyond those initialized, the code will:

* Register a jackpot win (increment `jackpotWins`, update `lastJackpotClaimWeek`), but
* Transfer 0 PLUME to the user and emit a SpinCompleted indicating a Jackpot with amount 0.

This both wastes a winning slot and decreases other rewards' effective probabilities.
