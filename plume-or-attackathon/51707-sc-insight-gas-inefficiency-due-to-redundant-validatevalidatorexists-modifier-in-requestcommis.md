# 51707 sc insight gas inefficiency due to redundant validatevalidatorexists modifier in requestcommissionclaim&#x20;

* **Submitted on:** Aug 5th 2025 at 07:06:18 UTC by @rilwan99 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #51707
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

## Description

Brief/Intro

The `requestCommissionClaim()` function in `ValidatorFacet.sol` applies two validators that both check the same `validatorExists[validatorId]` condition: the modifier `onlyValidatorAdmin(validatorId)` and the modifier `_validateValidatorExists(validatorId)`. Because the same storage read and conditional are performed twice, this results in unnecessary gas consumption for every commission claim request.

## Vulnerability Details

The function declares modifiers as follows:

```solidity
function requestCommissionClaim(
    uint16 validatorId,
    address token
)
    external
    onlyValidatorAdmin(validatorId)        // First validation
    nonReentrant
    _validateValidatorExists(validatorId)   // Redundant validation
    _validateIsToken(token)
{
    // Function implementation
}
```

`onlyValidatorAdmin` already performs a validator existence check:

```solidity
modifier onlyValidatorAdmin(uint16 validatorId) {
    if (!PlumeStakingStorage.layout().validatorExists[validatorId]) {
        revert ValidatorDoesNotExist(validatorId);
    }
    if (msg.sender != PlumeStakingStorage.layout().validators[validatorId].l2AdminAddress) {
        revert NotValidatorAdmin(msg.sender);
    }
    _;
}
```

`_validateValidatorExists` performs the identical existence check:

```solidity
modifier _validateValidatorExists(uint16 validatorId) {
    if (!PlumeStakingStorage.layout().validatorExists[validatorId]) {
        revert ValidatorDoesNotExist(validatorId);
    }
    _;
}
```

Because both modifiers execute, the same storage read `PlumeStakingStorage.layout().validatorExists[validatorId]` is performed twice.

## Impact Details

Gas inefficiency impact:

* Each redundant validation costs roughly one additional SLOAD and conditional logic (\~2,100+ gas).
* For frequent commission claim operations, this produces cumulative and avoidable gas waste for validator operators.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

## Proof of Concept

Scenario: Validator wastes gas when requesting for commission

* Setup: Validator is active, not slashed, has accrued rewards and commissions
* Assume: validator ID is 1
* Action: Validator admin calls `requestCommissionClaim(1, 0x123...)`

{% stepper %}
{% step %}

### Modifier Execution: onlyValidatorAdmin(1)

* Storage Read #1: `validatorExists[1]` → Returns `true` (\~2,100 gas)
* Conditional Check: `!validatorExists[1]` → `false`, continues
* Storage Read #2: `validators[1].l2AdminAddress` → Returns `0x123...`
* Authorization Check: `msg.sender == 0x123..` → `true`, continues
  {% endstep %}

{% step %}

### Modifier Execution: \_validateValidatorExists(1) (redundant)

* Storage Read #3: `validatorExists[1]` → Returns `true` (\~2,100 gas) — redundant
  {% endstep %}
  {% endstepper %}

Redundant Gas Consumption: approximately 2,100 gas for the extra SLOAD and associated logic.
