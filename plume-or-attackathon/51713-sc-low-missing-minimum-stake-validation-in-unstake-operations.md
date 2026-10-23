# 51713 sc low missing minimum stake validation in unstake operations

**Submitted on Aug 5th 2025 at 08:11:58 UTC by @rilwan99 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51713
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `PlumeStaking` contract allows users to perform partial unstaking without validating that the remaining stake meets the `minStakeAmount` requirement. This creates dust stakes below the minimum threshold and a loophole for users to bypass the minimum stake amount restriction.

### Vulnerability Details

The `unstake()` function validates the unstake amount but fails to check if the remaining stake meets minimum requirements:

```solidity
function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
    // Validates unstake amount only
    _validateValidatorForUnstaking(validatorId);
    if (amount == 0) {
        revert InvalidAmount(amount);
    }
    if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
        revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
    }
    // Missing: validation of remaining stake amount
}
```

This vulnerability is also present in `restake()`, as the `restake()` function moves funds from cooldown/parked to active stake on a different validator without ensuring the source validator retains minimum stake:

```solidity
function restake(uint16 validatorId, uint256 amount) external nonReentrant {
    // ... validation logic
    _performStakeSetup(user, validatorId, amount); // Only validates new stake amount
    // Missing: validation that source validator stake remains above minimum
}
```

## Impact Details

### Economic Impact

* Gas Inefficiency: Dust stakes increase gas costs for yield distribution operations as the system must process many sub-minimum positions
* Validator Overhead: Validators experience increased operational costs managing numerous small stakes
* System Degradation: Accumulation of dust positions leads to bloated storage and reduced performance

### Severity

This is a **MEDIUM** severity issue as it:

* Causes significant system inefficiency and poor user experience
* Creates technical debt that compounds over time
* Affects **core staking mechanics** used by all participants

## References

Add any relevant links to documentation or code

## Proof of Concept

{% stepper %}
{% step %}

### Dust Stakes via Partial Unstaking

Setup:

* User stakes 100 PLUME with Validator A, `minStakeAmount = 10 PLUME`

Action:

* User calls `unstake(validatorA, 95)` to unstake 95 PLUME

Validation:

* Function validates 95 PLUME > minStakeAmount ✓

Result:

* 5 PLUME remains staked (below 10 PLUME minimum)

Impact:

* User has economically unviable dust stake position
  {% endstep %}

{% step %}

### Dust Stakes via Restaking

Setup:

* User has 100 PLUME staked with Validator A, 50 PLUME in cooldown

Action:

* User calls `restake(validatorB, 95)`

Validation:

* Function validates 95 PLUME restake amount ✓

Execution:

* 50 PLUME from cooldown + 45 PLUME from Validator A stake moved to Validator B

Result:

* 5 PLUME remains with Validator A (below minimum)

Impact:

* User now has fragmented positions with dust stake
  {% endstep %}
  {% endstepper %}
