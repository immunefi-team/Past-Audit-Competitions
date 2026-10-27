# 52517 sc high missing point in time snapshot in batched yield distribution enables double claims and permanent fund lock

**Submitted on Aug 11th 2025 at 11:04:25 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52517
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

The `distributeYieldWithLimit` function calculates yield distribution based on current holder state in each batch call instead of using a fixed point-in-time snapshot.

When token transfers occur between batch calls, this causes distribution calculation inconsistencies that enable two critical exploits:

* Malicious actors can claim yield multiple times for the same tokens by transferring between batches.
* Legitimate holders can have their yield permanently locked in the contract when array repositioning causes them to be skipped.

With no recovery mechanism, locked funds accumulate indefinitely.

### Vulnerability Details

`distributeYieldWithLimit` is intended to distribute yield tokens across multiple batches for gas efficiency. However it recalculates distribution denominators and uses live balances during each batch rather than using a snapshot captured at distribution initiation.

Problematic mechanism (excerpt):

```solidity
 // Lines 510-516: Recalculated in EVERY batch call
uint256 effectiveTotalSupply = 0;
for (uint256 i = 0; i < totalHolders; i++) {
    address holder = $.holders.at(i);
    if (_isYieldAllowed(holder)) {
        effectiveTotalSupply += balanceOf(holder); // Current live balances
    }
}

// Lines 532-544: Distribution uses current balance
uint256 holderBalance = balanceOf(holder);
if (holderBalance > 0) {
    uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
    yToken.safeTransfer(holder, share);
}
```

Critical Issue: The function processes holders by INDEX, not by ADDRESS. That means:

* Batch 1 processes indices 0–999
* Batch 2 processes indices 1000–1999

If holders move positions in the array between batches, they can be processed multiple times or skipped entirely.

State mutations between batches:

* Balance changes: Token transfers modify holder balances used in `effectiveTotalSupply` calculation.
* Holder array mutations: When holders transfer all tokens, they're removed from the array:

```solidity
if (fromBalanceBefore == amount) {
    $.holders.remove(from); // EnumerableSet moves last element to removed position
}
```

Attack vectors:

* Double-Claim Exploit: transferring tokens across indices between batches to be included in multiple batch calculations.
* Permanent Fund Loss: EnumerableSet.remove() moves the last element to the removed position, possibly moving an unprocessed holder into an already processed index so they are never processed.
* DoS via Revert: If balances concentrate and the contract lacks sufficient yield tokens for later batches, ERC20 transfer reverts can halt distribution.

## Impact Details

* Primary: Permanent fund loss — legitimate holders can miss allocations and those yields become locked in the contract.
* Secondary: Double-distribution — attackers can receive more yield than their token share by manipulating positions/balances across batches.
* Tertiary: Distribution DoS — mid-distribution state changes can cause transfers to revert, preventing completion.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L462-L555>

## Proof of Concept

{% stepper %}
{% step %}

### Permanent Fund Lock via Array Repositioning — Setup

Initial State:

* Holders array: \[Alice(0), Bob(1), Charlie(2), Dave(3)]
* Each holds 250 tokens
* 1000 yield tokens to distribute
  {% endstep %}

{% step %}

### Permanent Fund Lock — Execution (Batch 1)

* Batch 1: Process index 0 (Alice)
* `effectiveTotalSupply = 1000`
* Alice receives 250 yield

State mutation:

* Alice transfers all 250 tokens to Dave
* `$.holders.remove(Alice)` triggered — EnumerableSet moves Dave from index 3 to index 0
* New array: \[Dave(0), Bob(1), Charlie(2)]
  {% endstep %}

{% step %}

### Permanent Fund Lock — Execution (Batch 2)

* Batch 2 starts from index 1 and processes Bob and Charlie
* `effectiveTotalSupply` is recalculated (Dave’s 500 tokens included)
* Bob receives 250 yield
* Charlie receives 250 yield

Result:

* Dave (now at index 0) was already past index 0 processing and is skipped for remaining batches
* Dave owns 500 tokens but received 0 yield
* 250 yield tokens become permanently locked in contract (no recovery mechanism)
  {% endstep %}
  {% endstepper %}

<details>

<summary>Demonstration 2: Double-Claim Attack (expand)</summary>

Initial State:

* Holders array: \[Alice(0), Bob(1), Charlie(2), Dave(3)]
* Each holds 250 tokens
* 1000 yield tokens to distribute

Execution:

* Batch 1: Process index 0 (Alice)
  * `effectiveTotalSupply = 1000`
  * Alice receives 250 yield

State mutation:

* Alice transfers 100 tokens to Bob
* Alice now has 150 tokens, Bob has 350 tokens
* Array remains unchanged
* Batch 2: Process index 1 (Bob)
  * `effectiveTotalSupply = 1000` (recalculated using live balances)
  * Bob receives (1000 × 350) / 1000 = 350 yield

Double-Claim outcome:

* Alice + Bob token holdings: 150 + 350 = 500 tokens
* Alice + Bob received yield: 250 + 350 = 600 yield
* They received yield corresponding to 600 tokens despite owning 500

</details>

## Attack Vector — DoS via Revert (example)

* Batch 1 distributes 500 yield tokens.
* Between batches, multiple holders transfer to a single address concentrating expected payouts.
* Batch 2 attempts to distribute 600 tokens to that concentrated holder while the contract only has 500 remaining — ERC20 insufficient balance revert occurs and distribution cannot complete.

## Notes / Summary of Root Cause

* The distribution logic uses live balances and iterates by index across multiple batches without a point-in-time snapshot.
* EnumerableSet removes/moves elements by swapping last element into removed index, which combined with index-based batching leads to holders being processed multiple times or skipped.
* There is no recovery or checkpointing mechanism to ensure a consistent view across batches.
