# 53043 sc high handlerandomness doesn t properly account for current streak which could result in the user spinning losing a jackpot

**Submitted on Aug 14th 2025 at 17:57:08 UTC by @valkvalue for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53043
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

`handleRandomness` doesn't properly account for the current streak which could result in the User spinning losing funds from the Jackpot.

### Vulnerability Details

Root cause: Wrong check, which is not inclusive of the current streak.

The flow for Spin.sol is described here: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/SPIN.md>

When the SupraOracle callback processes `handleRandomness`, the contract tracks the user's streak (consecutive days they called `startSpin()`). Per docs and implementation, the user must have a streak of `currentWeek + 2` to win the Jackpot:

Quote:

> Streak Requirement: The user's streakCount must be greater than or equal to currentWeek + 2. This means the required streak to be eligible for the jackpot increases as the campaign progresses. If this check fails, the reward also defaults to "Nothing".

Relevant code excerpts:

```solidity
function handleRandomness(uint256 nonce, uint256[] memory rngList) external onlyRole(SUPRA_ROLE) nonReentrant {
    address payable user = userNonce[nonce];
    if (user == address(0)) {
        revert InvalidNonce();
    }

    isSpinPending[user] = false;
    delete userNonce[nonce];
    delete pendingNonce[user];

    uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
    uint256 randomness = rngList[0]; // Use full VRF range
    (string memory rewardCategory, uint256 rewardAmount) = determineReward(randomness, currentSpinStreak);

    UserData storage userDataStorage = userData[user];
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
    } else if (keccak256(bytes(rewardCategory)) == keccak256("Raffle Ticket")) {
        userDataStorage.raffleTicketsGained += rewardAmount;
        userDataStorage.raffleTicketsBalance += rewardAmount;
    } else if (keccak256(bytes(rewardCategory)) == keccak256("PP")) {
        userDataStorage.PPGained += rewardAmount;
    } else if (keccak256(bytes(rewardCategory)) == keccak256("Plume Token")) {
        userDataStorage.plumeTokens += rewardAmount;
    } else {
        userDataStorage.nothingCounts += 1;
    }
    // update the streak count after their spin
    userDataStorage.streakCount = currentSpinStreak;
    ...
}
```

When processing a request, the contract computes the current streak via `_computeStreak`:

```solidity
function _computeStreak(address user, uint256 nowTs, bool justSpun) internal view returns (uint256) {
    // if a user just spun, we need to increment the streak its a new day or a broken streak
    uint256 streakAdjustment = justSpun ? 1 : 0;
    ...
    if (today == lastDaySpun + 1) {
        return userData[user].streakCount + streakAdjustment;
    } // streak not broken yet
    ...
}
```

Notes:

* `_computeStreak()` factors the new streak correctly (it uses `justSpun ? 1 : 0`) and the returned `currentSpinStreak` is used in `determineReward` to pick the reward category (see A.1 and A.2).
* However, the Jackpot eligibility check uses the stored `userDataStorage.streakCount` (pre-update) in the `else if (userDataStorage.streakCount < (currentWeek + 2))` branch (A.3).
* The stored `streakCount` is only updated later with `userDataStorage.streakCount = currentSpinStreak;` (A.4).

Because `determineReward` can return "Jackpot" based on the new streak but the subsequent eligibility check uses the old streak, the flow may fall into the "NotEnoughStreak" branch and convert a Jackpot to "Nothing" even when the user actually met the streak requirement for that spin.

### Impact Details

Jackpot would be lost even if the user meets the criteria for streak. This loss is irreversible for that spin because the RNG outcome is consumed; future spins receive new RNG and there's no guarantee of hitting the Jackpot again. Thus unclaimed Jackpot value could be lost.

### References

* <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/README.md>
* <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/SPIN.md>

## Proof of Concept

Detailed step-by-step explanation is provided in the Vulnerability Details. Summary flow:

{% stepper %}
{% step %}

### Step 1

Assume required streak (currentWeek + 2) equals 4 (so currentWeek is 2).
{% endstep %}

{% step %}

### Step 2

User currently has streak = 3.
{% endstep %}

{% step %}

### Step 3

User calls `startSpin()` on the 4th day. `_computeStreak` will return 4 (it accounts for the current spin). `determineReward(randomness, currentSpinStreak)` uses this corrected streak and may return `("Jackpot", amount)`.
{% endstep %}

{% step %}

### Step 4

But the Jackpot eligibility check uses `userDataStorage.streakCount` (still 3) and compares to `currentWeek + 2` (4). The check fails, the reward is downgraded to "Nothing", and later `userDataStorage.streakCount` is updated to 4. The user thus loses the Jackpot they should have received.
{% endstep %}
{% endstepper %}
