# 53069 sc low dynamic cooldown interval changes cause unexpected fund lockup extensions

**Submitted on Aug 14th 2025 at 18:51:13 UTC by @Outliers for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53069
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief / Intro

Administrators can unexpectedly extend user cooldown periods by modifying the global `cooldownInterval` parameter between `unstake` transactions. Users who initiate unstaking with the expectation of a specific cooldown period (e.g., 7 days) can have their wait time arbitrarily extended (e.g., to 14 days) if an admin updates the cooldown interval before their subsequent `unstake` transaction executes. This violates user expectations and contradicts the protocol's documentation (<https://docs.plume.org/plume/stake#important-notes-to-consider>) which suggests a fixed 21-day unstake period, creating potential for griefing attacks and unpredictable fund access delays.

## Vulnerability Details

The issue arises from the `_processCooldownLogic` function using the current global `cooldownInterval` value at transaction execution time, rather than preserving the cooldown period that was active when the user first initiated unstaking.

```solidity
function _processCooldownLogic(
    address user,
    uint16 validatorId,
    uint256 amount
) internal returns (uint256 newCooldownEndTime) {
    // ... existing logic ...
    
    // Uses CURRENT cooldownInterval, not the one from when user started unstaking
    newCooldownEndTime = block.timestamp + $.cooldownInterval; // @audit - Admin can change this
    
    if (currentCooledAmountInSlot > 0 && block.timestamp >= currentCooldownEndTimeInSlot) {
        // Previous cooldown matured - start new cooldown with CURRENT interval
        // ...
    } else {
        // Add to existing cooldown - but uses NEW cooldown end time
        // ...
    }
    
    cooldownEntrySlot.cooldownEndTime = newCooldownEndTime; // Overwrites previous end time
}
```

Attack consequences:

* A user who expects a specific cooldown duration can be forced to wait longer if `cooldownInterval` changes in between their unstake transactions.
* This allows griefing or denial-of-service-like scenarios where admins (or an attacker with admin control) can arbitrarily extend access to funds.

## Attack Scenario

{% stepper %}
{% step %}

### Initial state and first unstake

* cooldownInterval = 7 days
* Day 0 (timestamp 1000): User unstakes 100 tokens → expects withdrawal at timestamp 1000 + 7 days
  {% endstep %}

{% step %}

### Second unstake and admin change

* Day 5 (timestamp 1005): User submits second unstake (50 tokens)
* Admin transaction executes before user's second unstake and changes cooldownInterval = 14 days
  {% endstep %}

{% step %}

### Result

* User transaction executes after admin change
* New cooldown computed using CURRENT interval: 1005 + 14 days
* User must wait until 1005 + 14 days instead of expected 1005 + 7 days (or original slot's end)
* Net effect: user's withdrawal delayed by \~7 extra days relative to their expectation
  {% endstep %}
  {% endstepper %}

## Impact Details

* Users committed to shorter periods face unexpected extensions.
* Protocol documentation and user expectations about fixed cooldown durations are violated.
* Enables griefing via policy changes that are effectively retroactive for in-flight unstake flows.

## Mitigation

{% hint style="info" %}
Make cooldown interval truly constant for the purpose of already-started cooldowns. Preserve the cooldown interval (or resulting cooldown end time) that was active when the user first initiated unstaking, and use that stored value when calculating or updating that user's cooldown slot. Avoid reading a mutable global `cooldownInterval` at execution time for in-progress cooldowns.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

```
// Initial setup
$.cooldownInterval = 7 days; // 604800 seconds

// Day 0: User first unstake
unstake(validatorId, 100e18); // at timestamp 1000
// cooldownEndTime = 1000 + 604800 = 1000 + 7 days
// User expects withdrawal at that time
```

{% endstep %}

{% step %}

```
// Day 5: User submits second unstake transaction  
// User calculates: expects new cooldown = 1005 + 7 days
uint256 userSubmissionTime = 1005;

// RACE CONDITION: Admin changes cooldown interval
// Admin transaction executes first:
$.cooldownInterval = 14 days; // 1209600 seconds
```

{% endstep %}

{% step %}

```
// User transaction executes after admin change:
unstake(validatorId, 50e18); // at timestamp 1005

// _processCooldownLogic executes:
newCooldownEndTime = 1005 + $.cooldownInterval; // 1005 + 14 days
cooldownEntrySlot.cooldownEndTime = newCooldownEndTime; // OVERWRITES previous end time

// Result:
// User expected: 1005 + 7 days
// Actual result: 1005 + 14 days
// Additional delay: ~7 days beyond user expectation
```

{% endstep %}
{% endstepper %}
