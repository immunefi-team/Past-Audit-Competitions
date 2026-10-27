# 52770 sc high unbounded gas consumption via stakeonbehalf manipulation

**Submitted on Aug 13th 2025 at 02:21:23 UTC by @bl4ck4non for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52770
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Unbounded gas consumption

## Description

### Brief/Intro

An attacker can exploit the `stakeOnBehalf` function to artificially inflate a victim's validator association list, causing their withdrawal transactions to consume 75x more gas than necessary. This attack targets whale users who legitimately stake across multiple validators, forcing them to pay significantly increased gas fees for routine withdrawals. The root cause is that the withdrawal process iterates through all validators in a user's association list, including those added by attackers through minimal `stakeOnBehalf` calls.

### Vulnerability Details

The withdrawal process iterates through all validators in a user's association list, regardless of whether the user has actual cooldowns with those validators.

A whale user legitimately stakes to multiple validators (e.g., 10 validators) and later unstakes, creating cooldowns with those validators. An attacker uses `stakeOnBehalf` to stake minimal amounts (just above `minStakeAmount`) on behalf of the victim across thousands of additional validators.

Example from the code: `stakeOnBehalf`:

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

    // Emit events
    emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount);
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount);

    return stakeAmount;
}
```

Each `stakeOnBehalf` call triggers `addStakerToValidator`, which appends the validator to the victim's `userValidators` array:

```solidity
function addStakerToValidator(PlumeStakingStorage.Layout storage $, address staker, uint16 validatorId) internal {
    if (!$.userHasStakedWithValidator[staker][validatorId]) {
        $.userValidators[staker].push(validatorId);
        $.userHasStakedWithValidator[staker][validatorId] = true;
    }

    if (!$.isStakerForValidator[validatorId][staker]) {
        // === Store index before pushing ===
        uint256 index = $.validatorStakers[validatorId].length;
        $.validatorStakers[validatorId].push(staker);
        $.isStakerForValidator[validatorId][staker] = true;
        $.userIndexInValidatorStakers[staker][validatorId] = index; // <<< Store the index
    }
}
```

When the victim withdraws, `_processMaturedCooldowns` iterates through the entire (possibly inflated) `userValidators` array, performing expensive storage reads and checks on many validators that have no actual cooldowns:

```solidity
function _processMaturedCooldowns(
    address user
) internal returns (uint256 amountMovedToParked) {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    amountMovedToParked = 0;

    // Make a copy to avoid iteration issues when removeStakerFromValidator is called
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

            // Remove staker if they have no remaining stake with this validator
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

Because the attacker's minimal stakes remain active, the malicious validators are not removed from the victim's `userValidators` list during cleanup, so the attack persists.

### Impact Details

Economic impact on whale users who stake across multiple validators:

* Normal withdrawal: ~~200K gas (~~$6–12 at 20 gwei)
* Post-attack withdrawal: ~~15M gas (~~$450–900 at 20 gwei)
* Attack cost (attacker deposit): \~ $283 (example: 3000 × 0.01 PLUME × $0.09441)

This is around a 75x increase in gas cost and can cost victims thousands during network congestion. The attack is more effective during high gas periods when victims most want to exit.

## References

<details>

<summary>Links</summary>

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol?utm\\_source=immunefi>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeValidatorLogic.sol?utm\\_source=immunefi>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Alice (whale) stakes to 10 validators legitimately.
* Alice unstakes from all 10 validators, creating cooldowns.
* Alice's `userValidators` = \[Val1, Val2, ..., Val10]
  {% endstep %}

{% step %}

### Attacker Execution

Attacker inflates Alice's validator list by calling `stakeOnBehalf`:

```solidity
for (uint16 i = 11; i <= 3000; i++) {
    stakingContract.stakeOnBehalf{value: 0.01 PLUME}(i, alice);
}
```

{% endstep %}

{% step %}

### State After Attack

* Alice's `userValidators.length` = 3000
* Alice has actual cooldowns with validators 1–10 only
* Validators 11–3000 have `cooldownEntry.amount == 0`
  {% endstep %}

{% step %}

### Withdrawal Impact

When Alice calls `withdraw()`, `_processMaturedCooldowns` iterates 3000 validators instead of 10:

```solidity
for (uint256 i = 0; i < 3000; i++) {
    // 2990 iterations waste ~2,500 gas each = ~7.5M wasted gas
    if (cooldownEntry.amount == 0) {
        continue; 
    }
    // only ~10 iterations do real work
}
```

Result: Alice pays \~75x more gas for the same withdrawal operation due to the inflated validator list causing thousands of additional iterations.
{% endstep %}
{% endstepper %}

## Notes / Suggestions (kept minimal)

* The issue arises from appending validators to a user's `userValidators` array on any `addStakerToValidator` call, combined with iterating over that full array when processing cooldowns. Potential mitigations (not exhaustive) include:
  * Only add a validator to `userValidators` when the user has a non-zero stake or a non-zero cooldown amount, or
  * Track and iterate only active cooldown entries (e.g., maintain a separate list/map of validators with non-zero cooldowns), or
  * Add limits/guards to `stakeOnBehalf` (rate-limiting, minimum beneficiary checks, or privileged-only calls) to prevent mass inflation.

(Per instructions: no additional facts beyond the original report were added; suggestions are generic mitigation directions implicit from the vulnerability description.)
