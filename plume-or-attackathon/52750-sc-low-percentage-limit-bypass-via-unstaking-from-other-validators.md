# 52750 sc low percentage limit bypass via unstaking from other validators

**Submitted on Aug 12th 2025 at 22:50:25 UTC by @EFCCWEB3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52750
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The StakingFacet contract fails to enforce the 33% per-validator stake limit (as specified in the PlumeStaking documentation) during unstaking operations, allowing a validator’s stake percentage to exceed the limit indirectly when users unstake from other validators, reducing totalStaked. If exploited in production, this vulnerability could enable a single validator to control more than 33% of the total stake, undermining the protocol’s decentralization goals and increasing concentration risk, potentially amplifying the impact of slashing events or enabling disproportionate influence in consensus or governance mechanisms, violating the protocol’s design principles.

## Vulnerability Details

The PlumeStaking protocol enforces a maximum validator stake percentage of 33% (3300 basis points) via `_validateValidatorPercentage`, called during staking operations (stake, restake, restakeRewards) through `_performStakeSetup`. However, the unstake flow reduces `totalStaked` without checking whether other validators' percentages now exceed the cap. This asymmetric logic allows a validator whose delegatedAmount remains unchanged to become >33% of the (reduced) totalStaked after others unstake.

Key functions and snippets:

```solidity
function _validateValidatorPercentage(uint16 validatorId, uint256 stakeAmount) internal view {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    uint256 previousTotalStaked = $.totalStaked - stakeAmount;
    // Check if exceeding validator percentage limit
    if (previousTotalStaked > 0 && $.maxValidatorPercentage > 0) {
        uint256 newDelegatedAmount = $.validators[validatorId].delegatedAmount;
        uint256 validatorPercentage = (newDelegatedAmount * 10_000) / $.totalStaked;
        if (validatorPercentage > $.maxValidatorPercentage) {
            revert ValidatorPercentageExceeded();
        }
    }
}
```

Only checks the target validator during staking operations and assumes totalStaked is stable except for the new stake.

```solidity
function unstake(uint16 validatorId, uint256 amount) external nonReentrant returns (uint256) {
    return _unstake(validatorId, amount);
}

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

No validation of other validators’ percentages after reducing `totalStaked.`

```solidity
function _updateUnstakeAmounts(address user, uint16 validatorId, uint256 amount) internal {
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    $.userValidatorStakes[user][validatorId].staked -= amount;
    $.stakeInfo[user].staked -= amount;
    $.validators[validatorId].delegatedAmount -= amount;
    $.validatorTotalStaked[validatorId] -= amount;
    $.totalStaked -= amount;
}
```

Reducing `totalStaked` increases the stake percentage of other validators (`delegatedAmount / newTotalStaked`) without checking if any exceed `maxValidatorPercentage`.

{% stepper %}
{% step %}

### How the vulnerability is triggered — Staking Path

* When staking to validator A, `_validateValidatorPercentage` ensures `(delegatedAmount[A] + stakeAmount) / totalStaked <= 33%`, preventing direct over-concentration.
  {% endstep %}

{% step %}

### How the vulnerability is triggered — Unstaking Path

* When unstaking from validator B, `_updateUnstakeAmounts` reduces `validators[B].delegatedAmount` and `totalStaked`. This increases the percentage for validator A (`delegatedAmount[A] / newTotalStaked`) without any check. If `totalStaked` drops significantly, validator A’s percentage can exceed 33%.
  {% endstep %}
  {% endstepper %}

Documentation Gap: The docs specify a 33% per-validator limit but do not mention enforcement during unstaking or periodic checks to maintain the limit, incorrectly assuming staking-time checks suffice.

## Impact Details

This vulnerability undermines the protocol’s decentralization objective by permitting a validator to exceed the 33% stake limit when others unstake. It does not cause direct theft, freezing, or insolvency, but it allows stake concentration that may increase systemic risk (e.g., slashing impact, governance or consensus influence). The issue is an architectural flaw relative to the stated per-validator capacity limits.

## Proof of Concept

Initial State

* Total Staked: 1000 PLUME
* Validator 1: delegatedAmount = 300 PLUME → 30% = (300 \* 10\_000) / 1000 = 3000 bps
* Validator 2: delegatedAmount = 500 PLUME → 50% = 5000 bps
* Validator 3: delegatedAmount = 200 PLUME → 20% = 2000 bps
* maxValidatorPercentage: 3300 (33%)
* All validators are below the 33% limit.

User Balances

* Alice: 400 PLUME staked on Validator 2

Assumptions

* activeValidators = \[1, 2, 3]
* No validators are slashed (validators\[i].slashed = false)

Execution — Alice Unstakes from Validator 2 Action: unstake(validatorId=2, amount=400)

{% stepper %}
{% step %}

### \_unstake Validations

1. Validator exists & not slashed → \_validateValidatorForUnstaking
2. Amount non-zero and within Alice’s stake: userValidatorStakes\[Alice]\[2].staked >= 400
   {% endstep %}

{% step %}

### Updates during unstake

* Rewards updated: PlumeRewardLogic.updateRewardsForValidator
* \_updateUnstakeAmounts(Alice, 2, 400):
  * userValidatorStakes\[Alice]\[2].staked -= 400 → now 0
  * stakeInfo\[Alice].staked -= 400
  * validators\[2].delegatedAmount -= 400 → now 100 PLUME
  * validatorTotalStaked\[2] -= 400
  * totalStaked -= 400 → now 600 PLUME
* Initiates cooldown: \_processCooldownLogic
* Cleanup: \_handlePostUnstakeCleanup
* Emit: CooldownStarted(Alice, 2, 400, newCooldownEndTimestamp)
  {% endstep %}
  {% endstepper %}

New Validator Percentages

* Validator 1: (300 \* 10\_000) / 600 = 5000 bps (50% > 33%)
* Validator 2: (100 \* 10\_000) / 600 = 1666 bps (16.66%)
* Validator 3: (200 \* 10\_000) / 600 = 3333 bps (33.33% ≈ limit)

Result: Validator 1 exceeds the 33% limit without revert or notification.

Subsequent Actions

* Staking to Validator 1 is blocked by `_validateValidatorPercentage` because it’s already at 50%.
* The concentration persists until users stake elsewhere (increasing `totalStaked`) or Validator 1 is unstaked.

## References

(Original code references provided in the "Target" link above.)
