# 52870 sc low cooldown extension logic may lead to locked funds

**Submitted on Aug 13th 2025 at 20:11:57 UTC by @EFCCWEB3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52870
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief / Intro

The `_processCooldownLogic` function in the StakingFacet contract aggregates multiple unstaking amounts from the same validator into a single cooldown entry and resets the `cooldownEndTime` to `block.timestamp + cooldownInterval` for the entire aggregated amount. As a result, subsequent unstakes (before prior cooldowns mature) extend the cooldown for previously unstaked funds. In production this can delay user withdrawals beyond the expected cooldown period (for example, 7 days), increase exposure to slashing risk, and cause user confusion because the documentation does not explicitly mention this per-validator aggregation behavior.

### Vulnerability Details

PlumeStaking stores per-user, per-validator cooldown as a single `CooldownEntry` in `PlumeStakingStorage.userValidatorCooldowns[user][validatorId]`, containing `amount` and `cooldownEndTime`. When a user unstakes multiple times from the same validator while a previous cooldown is still active, `_processCooldownLogic`:

* Adds the new unstake `amount` to the existing `amount` in the same cooldown entry, and
* Resets `cooldownEndTime` for the entire aggregated amount to `block.timestamp + cooldownInterval`.

Thus, previously-unstaked funds whose cooldowns were approaching maturity now get extended to the new timestamp. The docs note `cooldownInterval > maxSlashVoteDuration` but do not disclose this aggregation & reset behavior.

Relevant code:

```solidity
function _processCooldownLogic(
    address user,
    uint16 validatorId,
    uint256 amount
) internal returns (uint256 newCooldownEndTime) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.CooldownEntry storage cooldownEntrySlot = $.userValidatorCooldowns[user][validatorId];

    uint256 currentCooledAmountInSlot = cooldownEntrySlot.amount;
    uint256 currentCooldownEndTimeInSlot = cooldownEntrySlot.cooldownEndTime;

    uint256 finalNewCooledAmountForSlot;
    newCooldownEndTime = block.timestamp + $.cooldownInterval;

    if (currentCooledAmountInSlot > 0 && block.timestamp >= currentCooldownEndTimeInSlot) {
        // Previous cooldown for this slot has matured - move to parked and start new cooldown
        _updateParkedAmounts(user, currentCooledAmountInSlot);
        _removeCoolingAmounts(user, validatorId, currentCooledAmountInSlot);
        _updateCoolingAmounts(user, validatorId, amount);
        finalNewCooledAmountForSlot = amount;
    } else {
        // No matured cooldown - add to existing cooldown
        _updateCoolingAmounts(user, validatorId, amount);
        finalNewCooledAmountForSlot = currentCooledAmountInSlot + amount;
    }

    cooldownEntrySlot.amount = finalNewCooledAmountForSlot;
    cooldownEntrySlot.cooldownEndTime = newCooldownEndTime;

    return newCooldownEndTime;
}
```

When a cooldown is active (`block.timestamp < currentCooldownEndTimeInSlot`), the function aggregates the new amount and resets `cooldownEndTime` for the total, thereby extending the cooldown for previously unstaked funds.

Other relevant functions:

```solidity
function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
    PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
    _validateValidatorForUnstaking(validatorId);
    if (amount == 0) {
        revert InvalidAmount(amount);
    }
    if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
        revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
    }
    PlumeRewardLogic.updateRewardsForValidator($s, msg.sender, validatorId);
    _updateUnstakeAmounts(msg.sender, validatorId, amount);
    uint256 newCooldownEndTimestamp = _processCooldownLogic(msg.sender, validatorId, amount);
    _handlePostUnstakeCleanup(msg.sender, validatorId);
    emit CooldownStarted(msg.sender, validatorId, amount, newCooldownEndTimestamp);
    return amount;
}
```

```solidity
function _removeCoolingAmounts(address user, uint16 validatorId, uint256 amount) internal {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    bool isSlashed = $.validators[validatorId].slashed;
    if ($.stakeInfo[user].cooled >= amount) {
        $.stakeInfo[user].cooled -= amount;
    } else {
        $.stakeInfo[user].cooled = 0;
    }
    if (!isSlashed) {
        if ($.totalCooling >= amount) {
            $.totalCooling -= amount;
        } else {
            $.totalCooling = 0;
        }
        if ($.validatorTotalCooling[validatorId] >= amount) {
            $.validatorTotalCooling[validatorId] -= amount;
        } else {
            $.validatorTotalCooling[validatorId] = 0;
        }
    }
    PlumeStakingStorage.CooldownEntry storage entry = $.userValidatorCooldowns[user][validatorId];
    if (entry.amount >= amount) {
        entry.amount -= amount;
        if (entry.amount == 0) {
            entry.cooldownEndTime = 0;
        }
    } else {
        entry.amount = 0;
        entry.cooldownEndTime = 0;
    }
}
```

```solidity
function _processMaturedCooldowns(address user) internal returns (uint256 amountMovedToParked) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    amountMovedToParked = 0;
    uint16[] memory userAssociatedValidators = $.userValidators[user];
    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        PlumeStakingStorage.CooldownEntry memory cooldownEntry = $.userValidatorCooldowns[user][validatorId];
        if (cooldownEntry.amount == 0) {
            continue;
        }
        bool canRecoverFromThisCooldown = _canRecoverFromCooldown(user, validatorId, cooldownEntry);
        if (canRecoverFromThisCooldown) {
            uint256 amountInThisCooldown = cooldownEntry.amount;
            amountMovedToParked += amountInThisCooldown;
            _removeCoolingAmounts(user, validatorId, amountInThisCooldown);
            delete $.userValidatorCooldowns[user][validatorId];
            if ($.userValidatorStakes[user][validatorId].staked == 0) {
                PlumeValidatorLogic.removeStakerFromValidator($, user, validatorId);
            }
        }
    }
    if (amountMovedToParked > 0) {
        _updateParkedAmounts(user, amountMovedToParked);
    }
    return amountMovedToParked;
}
```

```solidity
function _canRecoverFromCooldown(
    address user,
    uint16 validatorId,
    PlumeStakingStorage.CooldownEntry memory cooldownEntry
) internal view returns (bool canRecover) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    if (!$.validatorExists[validatorId]) {
        return false;
    }
    if ($.validators[validatorId].slashed) {
        uint256 slashTs = $.validators[validatorId].slashedAtTimestamp;
        return (cooldownEntry.cooldownEndTime < slashTs && block.timestamp >= cooldownEntry.cooldownEndTime);
    } else {
        return (block.timestamp >= cooldownEntry.cooldownEndTime);
    }
}
```

Documentation states: “unstaking initiates a cooldown period” with `cooldownInterval > maxSlashVoteDuration` to protect cooling funds from slashing, but it does not mention that multiple unstakes aggregate and reset the per-validator cooldown entry.

## Impact Details

Funds already in a cooldown can be locked longer than the expected `cooldownInterval` because a subsequent unstake (before maturity) resets the cooldown for the aggregated amount. For example, an additional unstake 1 hour after the first will extend the cooldown by \~1 hour for the earlier funds; repeated unstakes could extend the cooldown by days. This increases temporary exposure to slashing risk and may confuse users expecting earlier availability. Impact class: Temporary freezing of funds for at least 24 hours (not permanently lost).

## Proof of Concept

{% stepper %}
{% step %}

### Setup State

* `cooldownInterval = 7 days` (604,800 seconds)
* `maxSlashVoteDuration < 7 days` (per docs)
* Alice has `100 PLUME` staked on Validator 1 (`userValidatorStakes[Alice][1].staked = 100`)
* `userValidatorCooldowns[Alice][1]` is empty (`amount = 0`, `cooldownEndTime = 0`)
* `block.timestamp = 1000`
  {% endstep %}

{% step %}

### Assumptions

* Validator 1 is not slashed (`validators[1].slashed = false`)
* `userValidators[Alice] = [1]`
  {% endstep %}

{% step %}

### Execution — Alice Unstakes 100 PLUME

Alice calls: `unstake(validatorId=1, amount=100)`

Validations / actions performed:

* Validator exists & not slashed → `_validateValidatorForUnstaking`
* `amount = 100` is valid and within `userValidatorStakes[Alice][1].staked = 100`
* `_updateUnstakeAmounts(Alice, 1, 100)` reduces staked balances accordingly
* `_processCooldownLogic(Alice, 1, 100)`:
  * `currentCooledAmountInSlot = 0`
  * `currentCooldownEndTimeInSlot = 0`
  * No active cooldown → `finalNewCooledAmountForSlot = 100`
  * `newCooldownEndTime = 1000 + 604800 = 605800`
* `userValidatorCooldowns[Alice][1]` becomes `{ amount: 100, cooldownEndTime: 605800 }`
* `_updateCoolingAmounts(Alice, 1, 100)` increments cooled totals
* Emits: `CooldownStarted(Alice, 1, 100, 605800)`
  {% endstep %}

{% step %}

### Alice Unstakes 1 PLUME Before Maturity

* At `block.timestamp = 6060` (1 hour later) Alice calls: `unstake(validatorId=1, amount=1)` (assuming 1 PLUME was re-staked)

Processing:

* `currentCooledAmountInSlot = 100`
* `currentCooldownEndTimeInSlot = 605800`
* Since `block.timestamp = 6060 < 605800`, cooldown is active
* `_processCooldownLogic`:
  * Adds to existing cooldown: `finalNewCooledAmountForSlot = 100 + 1 = 101`
  * `newCooldownEndTime = 6060 + 604800 = 610860`
* `_updateCoolingAmounts(Alice, 1, 1)` increments cooled totals
* `userValidatorCooldowns[Alice][1]` becomes `{ amount: 101, cooldownEndTime: 610860 }`
* Emits: `CooldownStarted(Alice, 1, 1, 610860)`

Result:

* The original 100 PLUME (expected to mature at `t = 605800`) is now locked until `t = 610860` → \~6 days, 23 hours longer.
* If Alice continues to unstake periodically, the cooldown can be extended further (example: unstaking every hour, or again after 7 days could push maturity further).
  {% endstep %}
  {% endstepper %}

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

(See code excerpts above for the relevant functions.)
