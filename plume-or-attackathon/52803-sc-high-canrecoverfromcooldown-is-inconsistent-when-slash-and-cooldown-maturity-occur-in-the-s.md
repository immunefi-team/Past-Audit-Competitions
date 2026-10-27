# 52803 sc high canrecoverfromcooldown is inconsistent when slash and cooldown maturity occur in the same block

**Submitted on Aug 13th 2025 at 09:38:07 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52803
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

## Vulnerability Details

The `StakingFacet::_canRecoverFromCooldown()` function determines if a user can recover their stake after a cooldown, with special handling when the validator has been slashed:

{% code title="vulnerable snippet" %}

```solidity
if ($.validators[validatorId].slashed) {
    // Validator is slashed - check if cooldown ended BEFORE the slash
    uint256 slashTs = $.validators[validatorId].slashedAtTimestamp;
    return (cooldownEntry.cooldownEndTime < slashTs && block.timestamp >= cooldownEntry.cooldownEndTime);
```

{% endcode %}

The condition `cooldownEntry.cooldownEndTime < slashTs` introduces a strict inequality.

This means that when both events (cooldown maturity and slashing) happen in the same block, the withdrawing transaction that is processed before slashing is approved, while the one that goes after is rejected.

## Impact Details

* Yield loss for any user whose cooldown maturity happens in the same block as the slash, but whose transaction is processed after the slash.
* Inconsistent behavior: two users in identical cooldown states may get different results purely due to ordering in the same block.

The function `_canRecoverFromCooldown` is used by `_processMaturedCooldowns`, which is invoked by:

* `withdraw()`
* `restake()`
* `restakeRewards()`

These functions calculate matured cooldowns and allow withdrawing or restaking funds. The bug causes some identical cooldowns to be rejected depending on ordering within the block.

## Recommended fix

Use `<=` instead of `<` in the slash branch to ensure users whose cooldown matures exactly at the slash timestamp are treated consistently:

{% code title="fix suggestion" %}

```solidity
return (cooldownEntry.cooldownEndTime <= slashTs && block.timestamp >= cooldownEntry.cooldownEndTime);
```

{% endcode %}

## Proof of Concept

{% stepper %}
{% step %}

### Step

User A’s transaction is processed before the slash → their cooldown maturity check passes and they can recover.
{% endstep %}

{% step %}

### Step

Slash of the validator is approved.
{% endstep %}

{% step %}

### Step

User B’s transaction is processed after the slash → their cooldown maturity check fails and they permanently lose the ability to recover that cooldown.
{% endstep %}
{% endstepper %}

<details>

<summary>Relevant function snippet</summary>

```solidity
function _canRecoverFromCooldown(
    address user,
    uint16 validatorId,
    PlumeStakingStorage.CooldownEntry memory cooldownEntry
) internal view returns (bool canRecover) {
    ...
    if ($.validators[validatorId].slashed) {
        // Validator is slashed - check if cooldown ended BEFORE the slash
        uint256 slashTs = $.validators[validatorId].slashedAtTimestamp;
        return (cooldownEntry.cooldownEndTime < slashTs && block.timestamp >= cooldownEntry.cooldownEndTime);
    } else {
    ...
    }
}
```

</details>
