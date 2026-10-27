# 52998 sc low minor delays from oracle can unfairly reset users streak

**Submitted on Aug 14th 2025 at 15:51:17 UTC by @forgebyola for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52998
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Users can have their spin streaks unfairly reset due to oracle callback delays

## Description

### Brief/Intro

If the supra oracle delays callback, the user streak can get broken, unfairly penalizing the user.

### Vulnerability Details

Users maintain spin streaks by spinning at least once every 24 hours. The greater the user streak the more rewards they can potentially gain, therefore users are economically incentivized to maintain their streak by spinning every day.

When users make a spin, a random number is generated from the supra oracle and the oracle has to call back into the `Spin.sol` contract. The issue is that the user streak is only updated when the callback is received.

Relevant snippet from `handleRandomness`:

```solidity
function handleRandomness(
        uint256 nonce,
        uint256[] memory rngList
    ) external onlyRole(SUPRA_ROLE) nonReentrant {
        address payable user = userNonce[nonce];
        if (user == address(0)) {
            revert InvalidNonce();
        }

        isSpinPending[user] = false;
        delete userNonce[nonce];
        delete pendingNonce[user];

        uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
        uint256 randomness = rngList[0]; // Use full VRF range
        (string memory rewardCategory, uint256 rewardAmount) = determineReward(
            randomness,
            currentSpinStreak
        );

        ...................................................
        // update the streak count after their spin
        userDataStorage.streakCount = currentSpinStreak;

        userDataStorage.lastSpinTimestamp = block.timestamp;
        // ----------  Interactions: transfer Plume last ----------
        if (
            keccak256(bytes(rewardCategory)) == keccak256("Jackpot") ||
            keccak256(bytes(rewardCategory)) == keccak256("Plume Token")
        ) {
            _safeTransferPlume(user, rewardAmount * 1 ether);
        }

        emit SpinCompleted(user, rewardCategory, rewardAmount);
    }
```

The streak computation:

```solidity
function _computeStreak(
        address user,
        uint256 nowTs,
        bool justSpun
    ) internal view returns (uint256) {
        // if a user just spun, we need to increment the streak its a new day or a broken streak
        uint256 streakAdjustment = justSpun ? 1 : 0;

        uint256 lastSpinTs = userData[user].lastSpinTimestamp;

        if (lastSpinTs == 0) {
            return 0 + streakAdjustment;
        }
        uint256 lastDaySpun = lastSpinTs / SECONDS_PER_DAY;
        uint256 today = nowTs / SECONDS_PER_DAY;
        if (today == lastDaySpun) {
            return userData[user].streakCount;
        } // same day
        if (today == lastDaySpun + 1) {
            return userData[user].streakCount + streakAdjustment;
        } // streak not broken yet
        return 0 + streakAdjustment; // broken streak
    }
```

The user streak is reset if the user's `lastSpinTimestamp` surpasses 24 hours (i.e., crosses a day boundary based on integer division by seconds per day).

If, for example, a user spins at 11:58pm, and the supra oracle calls back into the contract at 12:01am, the user streak is calculated at the time when this callback happens, and therefore the user loses their streak (streak is reset) unfairly due to a condition out of their control.

### Impact Details

The user gets their streak reset unfairly, which can lead to economic losses for the user.

### References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L217>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L253>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario: user has an active streak

User has made spins for the last 4 days and has a current streak of 4 days.
{% endstep %}

{% step %}

### User spins late at night

User makes first spin of the day at 11:55pm expecting their streak to be maintained.
{% endstep %}

{% step %}

### Oracle callback delay

Supra oracle calls back at 12:01am.
{% endstep %}

{% step %}

### Streak recalculated on callback

User streak is calculated using this callback time.
{% endstep %}

{% step %}

### Result: streak reset

User's 4 day streak gets reset.
{% endstep %}
{% endstepper %}
