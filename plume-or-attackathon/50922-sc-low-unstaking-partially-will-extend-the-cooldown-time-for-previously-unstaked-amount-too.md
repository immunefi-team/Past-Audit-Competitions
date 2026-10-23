# 50922 sc low unstaking partially will extend the cooldown time for previously unstaked amount too

**Submitted on Jul 29th 2025 at 17:33:27 UTC by @WinSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50922
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The unstaking mechanism contains a flaw where partial unstaking operations reset the cooldown timer for all previously cooling tokens, forcing users to wait additional time beyond the intended cooldown period. This violates the expected behavior of the cooldown system and potentially prevents users from accessing their funds when legitimately expected.

### Vulnerability Details

When user calls `Unstake()` it internally calls and processes cooldown logic and updates the cooldown end time after which user can withdraw the unstaked amount.

Relevant snippets:

```solidity
function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
    PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
********************SNIP******************

    // Process cooldown logic and cleanup
    uint256 newCooldownEndTimestamp = _processCooldownLogic(msg.sender, validatorId, amount); <@
    _handlePostUnstakeCleanup(msg.sender, validatorId);

    emit CooldownStarted(msg.sender, validatorId, amount, newCooldownEndTimestamp);
    return amount;
}

function _processCooldownLogic(
    address user,
    uint16 validatorId,
    uint256 amount
) internal returns (uint256 newCooldownEndTime) {

********************SNIP******************

    newCooldownEndTime = block.timestamp + $.cooldownInterval;//@audit whenever the cooldown is updated the cooldown end time is updated.

********************SNIP******************

        _updateCoolingAmounts(user, validatorId, amount);
        finalNewCooledAmountForSlot = currentCooledAmountInSlot + amount;
    }
********************SNIP******************

    cooldownEntrySlot.cooldownEndTime = newCooldownEndTime; //@audit the cooldown time is updated

    return newCooldownEndTime;
}
```

When a user unstakes multiple times partially:

* First unstake: Sets `cooldownEndTime = block.timestamp + cooldownInterval`
* Second unstake: RESETS `cooldownEndTime = block.timestamp + cooldownInterval` (ignoring previous timeline)

This means previously cooling tokens lose their accumulated cooldown progress and must restart the entire cooldown period.

Expected execution should be:

* User unstakes 100 tokens → available after 7 days
* User unstakes 50 more tokens after 5 days → original 100 available in 2 more days, new 50 available in 7 more days

Current code executes:

* User unstakes 100 tokens → available after 7 days
* User unstakes 50 more tokens after 5 days → ALL 150 tokens now available in 7 more days (original 100 lost 5 days of progress)

### Impact Details

* Extended Capital Lockup: Users experience longer-than-intended fund lockup periods
* Lost Opportunity Costs: Users cannot access funds for DeFi strategies, arbitrage, or emergency needs

### References

plume/src/facets/StakingFacet.sol - `_processCooldownLogic` function

## Proof of Concept

{% stepper %}
{% step %}

### Initial Setup

Alice stakes 1000 PLUME tokens to Validator A

Current timestamp: Day 0

Cooldown interval: 7 days
{% endstep %}

{% step %}

### Step 1: First Unstake (Day 0)

Alice unstakes 300 PLUME.

System creates cooldown entry:

* amount = 300
* cooldownEndTime = Day 0 + 7 days = Day 7

Alice expects to withdraw 300 PLUME on Day 7.
{% endstep %}

{% step %}

### Step 2: Time Passes (Day 5)

5 days pass (now Day 5).

Alice's 300 PLUME should be withdrawable in 2 more days (Day 7). Alice decides she needs more liquidity.
{% endstep %}

{% step %}

### Step 3: Second Unstake (Bug Trigger, Day 5)

Alice unstakes another 200 PLUME.

BUG: System recalculates cooldown:

* New cooldownEndTime = Day 5 + 7 days = Day 12
* Total cooling amount: 300 + 200 = 500 PLUME

Impact: Original 300 PLUME that should be available on Day 7 now locked until Day 12.
{% endstep %}

{% step %}

### Step 4: Attempted Withdrawal (Day 7) and Further Exploitation

Alice tries to withdraw 300 PLUME on Day 7 — transaction fails (funds still in cooldown).\
Alice must wait until Day 12 for ALL 500 PLUME.

If Alice unstakes again on Day 10, cooldown resets to Day 17, further extending lockup.
{% endstep %}
{% endstepper %}

***

If you want, I can suggest specific code fixes or a patch approach to make cooldown entries preserve previous progress (for example: set per-entry cooldown end times instead of overwriting a shared slot, or set newEntry.cooldownEndTime = max(existingEndTime, block.timestamp + cooldownInterval)).
