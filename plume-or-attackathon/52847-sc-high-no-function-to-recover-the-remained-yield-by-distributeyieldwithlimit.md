# 52847 sc high no function to recover the remained yield by distributeyieldwithlimit&#x20;

**Submitted on Aug 13th 2025 at 15:43:57 UTC by @ubl4nk for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52847
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **References:** <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L451>

## Description

### Brief / Intro

This vulnerability causes yield tokens to become permanently locked in the contract due to integer division rounding errors, with no direct mechanism to recover them. The impact is significant when yield tokens have high value (e.g., $1000 per token) or when distributions occur frequently (e.g., daily), leading to accumulating losses that can reach substantial amounts.

### Vulnerability Details

* `distributeYieldWithLimit` computes each holder's share as: (totalAmount \* holderBalance) / effectiveTotalSupply Because Solidity integer division floors results, small rounding losses occur per-holder.
* The non-batched `distributeYield` compensates by assigning the remainder to the last holder (e.g., `lastShare = amount - distributedSum`), preventing rounding loss. `distributeYieldWithLimit` lacks any such final adjustment.
* Consequences:
  1. Locked funds: Cumulative rounding remainders cause `totalAmount - amountDistributed` to remain inside the contract with no direct recovery function.
  2. No recovery mechanism: Admin or other actors cannot reclaim the locked yield, even by changing batch parameters (`totalAmount`, `startIndex`, `maxHolders`).
  3. Misleading event: When `nextIndex == 0`, the function emits `YieldDistributed(totalAmount, yieldTokenAddr)`, implying the full amount was distributed while some tokens may remain locked.
  4. Scalability: Frequent distributions or many holders amplify the problem as rounding errors accumulate.

### Impact Details

A small amount becomes semi-locked in the contract and cannot be recovered directly, accumulating over time and potentially becoming significant.

## Proof of Concept

### Assumptions for PoC

* Contract deployed with a yield token (ERC20-like).
* Holders array: 3 holders (A, B, C) with balances of 10 tokens each.
* Total supply: 30 tokens (`effectiveTotalSupply = 30`).
* Yield amount: `totalAmount = 100` (units of yield token).
* Batch size: single batch with `maxHolders = 3`.
* No restricted holders for simplicity.

{% stepper %}
{% step %}

### Setup

* Holders:
  * A: balance = 10 (allowed)
  * B: balance = 10 (allowed)
  * C: balance = 10 (allowed)
* `totalHolders = 3`
* `effectiveTotalSupply = 10 + 10 + 10 = 30`
* Caller (with YIELD\_DISTRIBUTOR\_ROLE) approves and calls: distributeYieldWithLimit(100, 0, 3)
* Yield token transfer to contract: 100 units
  {% endstep %}

{% step %}

### Execute Batch: distributeYieldWithLimit(100, 0, 3)

Batch holders: A, B, C (all allowed).

Calculations per holder (Solidity integer division floors):

* For A:
  * share = (100 \* 10) / 30 = 1000 / 30 ≈ 33.33 -> 33
  * Transfer 33 to A
* For B:
  * share = (100 \* 10) / 30 = 33
  * Transfer 33 to B
* For C:
  * share = (100 \* 10) / 30 = 33
  * Transfer 33 to C

Distributed sum:

* amountDistributed = 33 + 33 + 33 = 99

Contract state after distribution:

* nextIndex = 0 (endIndex = 3 = totalHolders)
* Event emitted: YieldDistributed(100, yieldTokenAddr) — misleading because only 99 distributed
* Remaining locked in contract: 100 - 99 = 1
  {% endstep %}
  {% endstepper %}

## Key Evidence / Link

* Relevant code: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L451>
