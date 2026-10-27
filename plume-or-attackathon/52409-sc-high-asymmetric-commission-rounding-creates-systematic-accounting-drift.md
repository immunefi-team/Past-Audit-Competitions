# 52409 sc high asymmetric commission rounding creates systematic accounting drift

**Submitted on Aug 10th 2025 at 13:41:05 UTC by @spongebob for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52409
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

PlumeStaking calculates commission two different ways: users get charged with ceiling division, while validators accrue commission with floor division.

The mismatch creates a small rounding gap on every calc. That gap isn’t tracked anywhere, so over time the system’s books drift: sum(user\_payouts) + sum(validator\_commission) < sum(emitted\_rewards).

No one can steal funds and individual balances look fine, but the totals won’t reconcile and value effectively leaks from accounting.

## Vulnerability Details

**User commission (ceil)**

When users claim, `_calculateRewardsCore()` uses ceiling division to compute commission:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L346-L348>

The helper `_ceilDiv` explicitly rounds up so users always pay at least as much commission as validators should get:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L421-L433>

**Validator commission (floor)**

Validator accrual in `updateRewardPerTokenForValidator()` uses normal (floor) division. `_ceilDiv`’s comment even calls out this asymmetry: it’s designed so “sum of user commissions ≥ validator accrued commission.”

The extra few wei created by ceiling division during user claims never gets recorded as anyone’s balance. Users are paid net (gross − ceiling commission), validators later claim what they accrued via floor division, and the difference disappears.

Because rewards are processed across time segments between checkpoints, each segment can introduce another tiny delta.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L340-L356>

Lots of micro-claims or frequent rate changes mean lots of segments—and more drift. This is very subtle because each piece works alone.

Users don’t underpay, validators don’t over-accrue, and there are no negative balances.

The problem only shows up when you add everything across the system and compare to total emitted rewards.

## Impact Details

I consider a low severity as the contract delivers per-user/per-validator amounts, but total distribution won’t match emissions.

## Proof of Concept

{% stepper %}
{% step %}

### Deploy and setup

* Deploy the PlumeStaking system with a test validator and reward token.
* Set a commission rate for the validator (e.g., 33% to maximize rounding effects).
* Add a reward token with a specific rate that will create fractional commission amounts.
* Fund the treasury with sufficient reward tokens.
  {% endstep %}

{% step %}

### Create a minimal-stake scenario

* Stake a small amount (e.g., 1 wei) to create minimal rewards that amplify rounding effects.
* Set reward rate to a value that creates fractional commission (e.g., a rate that results in commission like 2.7 tokens).
* Wait for a short time period (e.g., 1 second) to accrue minimal rewards.
  {% endstep %}

{% step %}

### Record initial state

Record:

* Total reward tokens in treasury
* User's claimable rewards
* Validator's accrued commission
  {% endstep %}

{% step %}

### Manual expected calculation

Calculate expected values manually:

* Gross reward = stake × rate × time
* Commission (ceiling) = ceil(gross\_reward × commission\_rate)
* Commission (floor) = floor(gross\_reward × commission\_rate)
* Rounding delta = ceiling\_commission - floor\_commission
* Expected net user reward = gross\_reward - ceiling\_commission
  {% endstep %}

{% step %}

### Execute claim and observe

* Execute user claim and record actual claimed amount.
* Check validator's accrued commission using `getAccruedCommission()`.
  {% endstep %}

{% step %}

### Compute discrepancy

* Total distributed = user\_claimed + validator\_commission
* Expected total = gross\_reward
* Missing amount = expected\_total - total\_distributed
  {% endstep %}

{% step %}

### Repeat to accumulate drift

* Repeat the minimal accrual & claim cycle multiple times with different small stake amounts to accumulate the rounding delta.
* Each iteration should show the same small rounding delta being lost.
* Sum all missing amounts to demonstrate cumulative accounting drift.
  {% endstep %}

{% step %}

### Final observation

* Confirm treasury balance shows more tokens than the sum of all distributed rewards.

Conclusion: Each claim cycle will show a small amount (the rounding delta) that disappears from accounting. The cumulative effect grows with each transaction. Treasury will retain slightly more tokens than the sum of user payouts + validator commissions and the equation emitted\_rewards ≠ Σ(user\_payouts) + Σ(validator\_commission) will be violated.
{% endstep %}
{% endstepper %}
