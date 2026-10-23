# 52312 sc low cooldown coalescing bug unintended cooldown extension for prior unstakes

**Submitted on Aug 9th 2025 at 19:03:25 UTC by @Ambitious\_DyDx for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52312
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

In `StakingFacet.sol`, `userValidatorCooldowns[user][validatorId]` usually stores one `{amount, cooldownEndTime}` slot. When a user unstakes again for the same validator before that slot matures, `_processCooldownLogic` (internal function) adds the new amount and overwrites `cooldownEndTime = block.timestamp + cooldownInterval`. This causes earlier-unstaked funds to be re-locked until the new (later) timestamp — multiple unstakes are not independently timed.

### Vulnerability Details

Relevant lines in `_processCooldownLogic` (lines \~818 - 848) show:

```solidity
finalNewCooledAmountForSlot = currentCooledAmountInSlot + amount;
cooldownEntrySlot.amount = finalNewCooledAmountForSlot;
cooldownEntrySlot.cooldownEndTime = newCooldownEndTime; // <-- overwrites prior end
```

* `_processCooldownLogic` coalesces unstakes into a single slot and always sets a fresh `cooldownEndTime` when adding to an unmatured slot.
* As a result, `_processMaturedCooldowns` and `_calculateTotalWithdrawableAmount` use that single timestamp for maturity checks. Overwriting it hides matured portions until the new end, effectively extending their lock.

### Impact Details

* Previously-unstaked funds can be re-locked for up to `cooldownInterval` when a later unstake occurs.
* High user-impact (funds inaccessible for days), even if not stolen — opportunity cost/trust damage.
* Any user doing repeated unstakes to the same validator before the prior cooldown matures won't be able to withdraw prior unstake amounts on time.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol?utm\\_source=immunefi>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario — initial unstake

* t0: call `unstake(validatorId, 100)`
* Resulting slot: `{ amount: 100, cooldownEndTime: t0 + cooldownInterval }`
  {% endstep %}

{% step %}

### Second unstake before first matures

* t1 = t0 + cooldownInterval - 1: call `unstake(validatorId, 50)`
* Code sees unmatured slot and adds amount, then sets `cooldownEndTime = t1 + cooldownInterval`
* Slot becomes `{ amount: 150, cooldownEndTime: t1 + cooldownInterval }`
  {% endstep %}

{% step %}

### Expected vs actual withdrawability

* At t = t0 + cooldownInterval + ε, calling `withdraw()` will **not** include the original 100 — it remains locked until `t1 + cooldownInterval`.
* Observables:
  * `getUserCooldowns(user)` returns a single entry with aggregated amount and the later end.
  * `amountWithdrawable(user)` at `t0 + interval + ε` does not include the original 100.
    {% endstep %}
    {% endstepper %}

## Recommendation

Change storage to allow multiple cooldown entries per `(user, validator)`, and push a new entry on each unstake, so each unstake has its own expiry:

```solidity
struct CooldownEntry {
    uint256 amount;
    uint256 cooldownEndTime;
}

mapping(address => mapping(uint16 => CooldownEntry[])) userCooldownQueues;

// In _processCooldownLogic():
userCooldownQueues[user][validatorId].push(CooldownEntry({
    amount: amount,
    cooldownEndTime: block.timestamp + cooldownInterval
}));
```

This preserves independent timing for each unstake (no overwriting of earlier cooldownEndTime).
