# 52634 sc high batch yield distribution has a mathematical flaw that enables economic manipulation

**Submitted on Aug 12th 2025 at 07:39:34 UTC by @spongebob for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52634
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

The `distributeYieldWithLimit` function in `ArcToken` contains a mathematical flaw that allows attackers to steal disproportionate shares of yield distributions.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466-L555>

The issue stems from the function's design where each call recalculates `effectiveTotalSupply` dynamically but continues using the original `totalAmount` for share calculations.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L540>

This creates a mathematical inconsistency: `effectiveTotalSupply` is recalculated for each batch by iterating through all current holders and checking their eligibility,

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L510-L516>

while `totalAmount` remains the original campaign amount across all batches and never accounts for previously distributed amounts.

Also, the contract stores no campaign-level state between batches to track distributed amounts or snapshot initial conditions.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L49-L60>

An attacker can manipulate the holder set or eligibility status between batches by:

* Transferring tokens to/from restricted addresses to change the `effectiveTotalSupply`
* Manipulating their position in the holders enumerable set to ensure early batch processing
* Coordinating with others to change yield eligibility status between batches

Example scenario demonstrating the inconsistency:

* Initial state: A=50, B=30, C=20 tokens; `totalAmount=100`; `effectiveTotalSupply=100`
* Batch 1 (A): share = (100×50)/100 = 50 tokens distributed
* Between batches: 10 tokens moved from B to restricted address
* New state: A=50, B=20, C=20; `effectiveTotalSupply=40`
* Batch 2 (B): share = (100×20)/40 = 50 tokens distributed (total paid: 100)
* Batch 3 (C): share = (100×20)/40 = 50 tokens needed, but 0 remaining → revert

The yield token transfer only occurs on the first batch (`startIndex == 0`), but subsequent batches continue distributing from the same pool using the flawed calculation.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L526-L528>

## Impact Details

Attackers can capture disproportionate shares of yield distributions that rightfully belong to other token holders. Later batches may revert due to insufficient contract balance, preventing legitimate holders from receiving their entitled yield.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

1. Deploy `ArcToken` with yield distribution capability.
2. Token has 3 holders: Alice (50 tokens), Bob (30 tokens), Charlie (20 tokens).
3. Total supply: 100 tokens, all holders are yield-eligible.
4. Yield distributor prepares 100 yield tokens for distribution.
5. Batch size is set to 1 holder per batch.
   {% endstep %}

{% step %}

### Verify initial state

* `effectiveTotalSupply` = 50 + 30 + 20 = 100 tokens
* All holders are in the enumerable set and yield-eligible
* Contract has 0 yield tokens initially
  {% endstep %}

{% step %}

### Step 1: First Batch Distribution

* Attacker calls `distributeYieldWithLimit(100, 0, 1)`
* Function calculates `effectiveTotalSupply = 100` (all holders eligible)
* Alice (first holder) gets: `(100 * 50) / 100 = 50` yield tokens
* The full 100 yield tokens are transferred to contract on `startIndex == 0`
* Alice receives 50 yield tokens, contract has 50 remaining
  {% endstep %}

{% step %}

### Step 2: Manipulation Between Batches

* Attacker (or accomplice) transfers 10 tokens from Bob to a yield-restricted address (e.g., blacklisted or failing `_isYieldAllowed()`), changing balances to:
  * Alice: 50, Bob: 20, Charlie: 20
* New `effectiveTotalSupply` = 50 + 20 + 20 = 90
  {% endstep %}

{% step %}

### Step 3: Second Batch Distribution

* Attacker calls `distributeYieldWithLimit(100, 1, 1)` for Bob
* Function recalculates `effectiveTotalSupply = 90`
* Bob gets: `(100 * 20) / 90 = 22.22` (rounded down to 22) yield tokens
* Contract now has 28 yield tokens remaining
* Bob received proportionally more than he should relative to the original distribution
  {% endstep %}

{% step %}

### Step 4: Third Batch Distribution

* Attacker calls `distributeYieldWithLimit(100, 2, 1)` for Charlie
* Function still uses `totalAmount = 100` and recalculated `effectiveTotalSupply = 90`
* Charlie should get: `(100 * 20) / 90 = 22.22` yield tokens
* Contract only has 28 tokens left; depending on rounding and previous transfers, Charlie may be underpaid or the transaction may revert
* Total distributed exceeds what should have been fairly allocated, or later batches fail
  {% endstep %}
  {% endstepper %}

## Result

* Alice got 50 tokens (should have been \~55.6 based on final supply if snapshots were used)
* Bob got 22 tokens (timing-dependent)
* Charlie gets 22 tokens or less (may be underpaid)
* Total distributed does not preserve proportionality; eligible holders can be economically manipulated

## References

* Vulnerable code region: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466-L555>

(End of report)
