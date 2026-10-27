# 50745 sc low single cooldown entry design causes timer reset on multiple unstakes leading to extended lock periods

**Submitted on Jul 28th 2025 at 09:22:28 UTC by @Bluedragon for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50745
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Summary

The `StakingFacet._processCooldownLogic()` function resets the cooldown timer for the entire amount when a user performs multiple unstake operations within the same cooldown period, instead of maintaining separate cooldown tracking for each unstake operation.

### Vulnerability Details

The root cause is that each validator-user pair has only one `CooldownEntry` slot. When a user unstakes multiple times before the first cooldown expires, the system aggregates all amounts into a single cooldown entry and resets the timer to `block.timestamp + $.cooldownInterval`. Thus earlier unstake operations lose their original maturation time and get pushed forward.

Problematic logic in `_processCooldownLogic()`:

{% code title="StakingFacet.sol (relevant snippet)" %}

```
```

{% endcode %}

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

(Original logic kept intact — shows aggregation and reset of cooldown end time.)

## Impact

* Users cannot withdraw their funds when the original cooldown period expires
* Extended waiting periods beyond the intended cooldown duration
* Loss of liquidity for users who perform multiple unstake operations
* Violation of user expectations regarding withdrawal timing

## Recommended Mitigation

Implement a queue-based or array-based cooldown system that tracks individual unstake operations separately, rather than aggregating them into a single cooldown entry. This allows each unstake to mature independently according to its original timeline.

## Proof of Concept

{% stepper %}
{% step %}

### Scenario — Initial state and first unstake (Day 0)

* User has 150 PLUME staked with a validator.
* Day 0: User unstakes 100 PLUME.

Results:

* `cooldownEntry.amount = 100 PLUME`
* `cooldownEntry.cooldownEndTime = block.timestamp + 7 days`
  {% endstep %}

{% step %}

### Second unstake within original cooldown (Day 6)

* Day 6: User unstakes additional 50 PLUME (within the original cooldown period).

Because the code aggregates into the same slot and resets the timer:

* `cooldownEntry.amount = 150 PLUME` (100 + 50)
* `cooldownEntry.cooldownEndTime = block.timestamp + 7 days` (resets to Day 6 + 7 = Day 13)
  {% endstep %}

{% step %}

### Attempted withdrawal after original expected maturation (Day 7)

* Day 7: User attempts to withdraw the original 100 PLUME.

Outcome:

* Withdrawal fails because `block.timestamp < cooldownEntry.cooldownEndTime` (Day 7 < Day 13).
* User must wait until Day 13 to withdraw any funds.
  {% endstep %}
  {% endstepper %}

### Proof Of Code (test case)

Add the following test case to `PlumeStakingDiamond.t.sol` in the `test` directory and run with `forge test --mt testIncorectCoolDownPeriod_Bluedragon -vv --via-ir`:

{% code title="PlumeStakingDiamond.t.sol — testIncorectCoolDownPeriod\_Bluedragon" %}

```
```

{% endcode %}

\`\`\`solidity function testIncorectCoolDownPeriod\_Bluedragon() public { uint256 amount = 150e18; vm.startPrank(user1); // user1 stakes 150 PLUME StakingFacet(address(diamondProxy)).stake{value: amount}( DEFAULT\_VALIDATOR\_ID ); assertEq(StakingFacet(address(diamondProxy)).amountStaked(), amount);

```
    // Unstake
    StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 100e18);
    console2.log("==== UnStaking ==== ");
    console2.log("User Cooling Amount After 1st UnStake: ", StakingFacet(address(diamondProxy)).amountCooling());
    console2.log("After 6 days User UnStakes Remaining PLUME");
    skip(6 days); // Skip 6 days


    // Unstake remaining amount
    StakingFacet(address(diamondProxy)).unstake(DEFAULT_VALIDATOR_ID, 50e18);
    console2.log("User Cooling Amount After 2nd UnStake: ", StakingFacet(address(diamondProxy)).amountCooling());

    skip(1 days); // Skip remaining 1 days to complete the cooldown period
    console2.log("\n==== (7th Day) After Remaining 1 day User Should Be Able To Withdraw 1st Cooling Amount ==== ");

    // Now check the parked funds
    uint256 parked = StakingFacet(address(diamondProxy)).amountWithdrawable();
    console2.log("Withdrawal Funds After Cooldown Interval: ", parked);
    vm.stopPrank();
}
```

```
(Original test code retained.)

<details>
<summary>Test logs</summary>

```

\[PASS] testIncorectCoolDownPeriod\_Bluedragon() (gas: 585956) Logs: ==== UnStaking ==== User Cooling Amount After 1st UnStake: 100000000000000000000 After 6 days User UnStakes Remaining PLUME User Cooling Amount After 2nd UnStake: 150000000000000000000

\==== (7th Day) After Remaining 1 day User Should Be Able To Withdraw 1st Cooling Amount ==== Withdrawal Funds After Cooldown Interval: 0

```
</details>
```
