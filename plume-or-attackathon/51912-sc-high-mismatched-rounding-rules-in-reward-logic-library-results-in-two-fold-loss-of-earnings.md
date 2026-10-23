# 51912 sc high mismatched rounding rules in reward logic library results in two fold loss of earnings

**Submitted on Aug 6th 2025 at 14:55:30 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51912
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Loss of unclaimed yield due to rounding errors

## Description

### Brief/Intro

Every time rewards are settled, the contract silently loses the remainder created by divergent rounding rules. Over weeks of operation—and especially with many delegators per validator—this gap grows into a material shortfall that makes the true APR lower than the correct value.

### Description

The PlumeRewardLogic is utilized to calculate the rewards for stakers and the commission for validators. In this context, `_calculateRewardsCore` charges a ceiled commission value to the stakers, but validators are paid a floored commission value.

Where users are charged (round-up):

```solidity
// PlumeRewardLogic.sol :: _calculateRewardsCore
commissionForThisSegment =
    _ceilDiv(grossRewardForSegment * effectiveCommissionRate, REWARD_PRECISION);
```

File link: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L348>

The ceilDiv function adds b-1 before division, so any fractional part pushes the result up by one wei.

Where validators are paid (double round-down):

```solidity
// PlumeRewardLogic.sol :: updateRewardPerTokenForValidator
grossRewardForValidatorThisSegment =
    (totalStaked * rewardPerTokenIncrease) / REWARD_PRECISION;   // floor #1

commissionDeltaForValidator =
    (grossRewardForValidatorThisSegment * commissionRateForSegment)
    / REWARD_PRECISION;                                          // floor #2
```

File link: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L181C21-L187C62>

Both divisions truncate toward zero, so the validator is under-credited.

The `totalClaimableByToken`, `userRewards`, and `validatorAccruedCommission` functions never receive the missing wei. There is also no “dust sweep” or “fee collector” variable; leading the contract’s ERC-20 balance to drift away from the sum of the bookkeeping variables.

### Impact

* Delegator APR erosion — each settlement skims extra wei from every delegator’s reward.
* Validator revenue loss — commission actually received is systematically lower than the rate advertised on-chain.
* Accounting invariants broken — the true Σ(userRewards) + Σ(validatorAccruedCommission) is less than the supposedly correct value.

## Proof of Concept

For clarity purposes in this proof of concept, we'll consider 1 token = 1e18 units of a token.

{% stepper %}
{% step %}

### Setup

* Deploy protocol with `commissionRate = 10 %` (1e17).
* Two users each stake **1 token** with the same validator.
  {% endstep %}

{% step %}

### Set reward rate

* Global rate yields **1 token + 1 wei** total over the next period.
  {% endstep %}

{% step %}

### Trigger settlement (call any function that invokes `updateRewardsForValidator`)

* Per-user gross reward = 0.5 token
* Per-user commission = `ceil(0.5 × 0.1) = 0.05 000…→ 0.05 000 000 000 000 000 001`
* Total removed from users:\
  `2 × 0.050000…001 wei ≈ 0.100000…002 wei`
  {% endstep %}

{% step %}

### Validator accrual path

* `totalStaked = 2`
* `grossRewardForValidator = floor(1 × 0.1 + 1 wei) = 0.1 token`
* Validator receives: `0.1 token` (no extra wei)
  {% endstep %}

{% step %}

### Observation

* `validatorAccruedCommission` increases by exactly 0.1 token
* Sum removed from users is **0.1 token + 2 wei**
* The extra 2 wei are nowhere in storage yet still reside in the ERC-20 balance.
  {% endstep %}

{% step %}

### Repeat

Every settlement with at least one delegator whose commission share is fractional repeats the leak, widening the gap between bookkeeping and real balances.
{% endstep %}
{% endstepper %}

***
