# 52560 sc high incorrect current streak used when calculating whether the jackpot should be awarded or not

**Submitted on Aug 11th 2025 at 15:47:59 UTC by @swarun for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52560
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

When deciding whether the jackpot should be awarded, the function does not take into account the user's current spin count correctly. This can lead to denial of the jackpot prize to an otherwise eligible user.

### Vulnerability Details

When the determineReward function indicates a user is eligible for the jackpot, the contract checks the user's streak against the required threshold. However, that check compares the stored (old) streakCount instead of the computed currentSpinStreak (which already includes the current spin). Because the stored streakCount is updated only after the jackpot eligibility check, the check can use a stale value (one less than the current streak), causing an eligible user to be denied the jackpot.

### Impact Details

A user who has satisfied the streak requirement (including the current spin) can be incorrectly prevented from receiving the jackpot prize. Since jackpot wins are rare and valuable, this results in a significant loss for affected users.

### Reference

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L232>

## Proof of Concept

{% stepper %}
{% step %}

### Step — startSpin

User with streak equal to current week + 1 calls startSpin and pays the spin price:

```solidity
function startSpin() external payable whenNotPaused canSpin {
    if (!enableSpin) {
        revert CampaignNotStarted();
    }
    require(msg.value == spinPrice, "Incorrect spin price sent");

    if (isSpinPending[msg.sender]) {
        revert SpinRequestPending(msg.sender);
    }
    isSpinPending[msg.sender] = true;

    string memory callbackSignature = "handleRandomness(uint256,uint256[])";
    uint8 rngCount = 1;
    uint256 numConfirmations = 1;
    uint256 clientSeed = uint256(keccak256(abi.encodePacked(admin, block.timestamp)));

    uint256 nonce = supraRouter.generateRequest(callbackSignature, rngCount, numConfirmations, clientSeed, admin);
    userNonce[nonce] = payable(msg.sender);
    pendingNonce[msg.sender] = nonce;

    emit SpinRequested(nonce, msg.sender);
}
```

{% endstep %}

{% step %}

### Step — randomness callback (handleRandomness)

The supra router calls handleRandomness:

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

    // Apply reward logic
    UserData storage userDataStorage = userData[user];

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

    userDataStorage.lastSpinTimestamp = block.timestamp;
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

Note: currentSpinStreak is computed and passed to determineReward.
{% endstep %}

{% step %}

### Step — \_computeStreak

The computed streak logic:

```solidity
function _computeStreak(address user, uint256 nowTs, bool justSpun) internal view returns (uint256) {
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

This function returns the updated streak (including the current spin when justSpun is true).
{% endstep %}

{% step %}

### Step — where the incorrect check happens

determineReward may return ("Jackpot", amount) based on currentSpinStreak. However, the jackpot eligibility guard uses the stored userDataStorage.streakCount (old value) instead of currentSpinStreak:

```solidity
} else if (userDataStorage.streakCount < (currentWeek + 2)) {
    userDataStorage.nothingCounts += 1;
    rewardCategory = "Nothing";
    rewardAmount = 0;
    emit NotEnoughStreak("Not enough streak count to claim Jackpot");
}
```

Since userDataStorage.streakCount is only updated after this check:

```solidity
// update the streak count after their spin
userDataStorage.streakCount = currentSpinStreak;
```

the contract may treat a legitimately eligible user as ineligible (stale value), denying the jackpot.
{% endstep %}
{% endstepper %}

## Summary of the bug

* determineReward is called with currentSpinStreak (which correctly accounts for the current spin).
* The subsequent jackpot eligibility check uses userData.storage.streakCount (old/stale value).
* The stored streak is updated only after the check, so users can be incorrectly denied the jackpot even when currentSpinStreak meets the requirement.

## Suggested remediation (high level)

* Use the computed currentSpinStreak when checking jackpot eligibility (i.e., compare currentSpinStreak to the threshold), or update userDataStorage.streakCount before performing the jackpot eligibility check.
* Ensure effect-ordering follows checks that depend on computed ephemeral state.

(End of report)
