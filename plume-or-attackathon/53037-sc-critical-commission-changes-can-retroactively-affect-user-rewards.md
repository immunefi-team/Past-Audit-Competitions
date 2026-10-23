# 53037 sc critical commission changes can retroactively affect user rewards

**Submitted on Aug 14th 2025 at 17:50:07 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #53037
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Theft of unclaimed yield

## Description

### Brief/Intro

When a Validator is being added, no commission checkpoint is actually being pushed. This creates a situation where validator commission changes can retroactively affect user rewards for intervals that occurred before that change, possibly incentivizing validators and users to work together to increase gains.

### Vulnerability Details

Instead of adding a commission checkpoint when a validator is being added, only the `validator.commission` variable is actually being set, and the code relies on that variable being returned if the correct commission is not found when `getEffectiveCommissionRateAt()` is being called (and it won't be found for that initial interval, because no commission checkpoint exists).

```
fallbackComm = $.validators[validatorId].commission;
```

However, when a validator's commission is being changed using `setValidatorCommission()`, the commission for previous intervals is only settled from the validator's perspective:

```solidity
// Settle commissions accrued with the old rate up to this point.
PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
```

Not from the user's perspective, whose rewards are still determined based on the checkpoint mechanism that calls `getEffectiveCommissionRateAt()`, which would return the current commission for the initial interval, not the one that actually applied at the time.

### Impact Details

While this could just result in inaccurate accounting where users receive less/more (depending on if the validator commission was increased/decreased) than what they were entitled to, this issue could also incentivize validators and users to work together and game the system.

Suppose the initial commission when the validator is added is very high. A user can stake to that validator and not claim his rewards until enough time has passed and a validator purposefully decreases the commission to the minimum. While the validator's commission is not retroactively affected (as the code settles the state up until the change), the user would receive much more than he should have had, as it is that new (and smaller) commission that would be applied to his previous earnings, not the original one.

### Suggestions

Actually add checkpoints when a validator is added.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Validator \_v is added using `addValidator()` with `maxAllowedValidatorCommission` as the commission. Only `_v.commission` is actually set.
{% endstep %}

{% step %}

### Step

User stakes a large amount to Validator \_v.
{% endstep %}

{% step %}

### Step

After a while, Validator admin calls `setValidatorCommission()` and changes `_v.commission` to 0. From the validator's perspective, commission for past intervals are settled correctly before the commission actually changes.
{% endstep %}

{% step %}

### Step

User calls `claim()` to calculate the accrued reward from the moment he staked. As no commission checkpoint actually exists for the first interval, `_v.commission`, which is now 0, is used.

Result: The user ended up not paying any commission for the interval between steps 2 and 3, but the validator still received the full commission for that same interval.
{% endstep %}
{% endstepper %}
