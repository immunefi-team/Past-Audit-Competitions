# 51414 sc high attacker can drain yield by transferring tokens to other address in yield batch distributions

**Submitted on Aug 2nd 2025 at 15:11:09 UTC by @TeamJosh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51414
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief / Intro

```sol
function distributeYieldWithLimit(
    uint256 totalAmount,
    uint256 startIndex,
    uint256 maxHolders
) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed)
```

The above function distributes a specified `totalAmount` of a yield token proportionally among token holders in batches defined by `startIndex` and `maxHolders`. The function skips holders for whom `_isYieldAllowed()` returns false. The `totalAmount` is transferred from the distributor only during the first batch (`startIndex == 0`).

## Vulnerability Details

The vulnerability lies in how the `totalAmount` of yield is calculated once at the beginning (first batch), but distributed over multiple batches.

An attacker who receives yield in the first batch (due to having balance and being `yieldAllowed`) can:

* Transfer their entire token balance to another address before the next batch starts.
* The second address will be included in a later batch as a valid holder, and will again receive a proportional share of the same `totalAmount`.
* By repeating this process across multiple addresses, the attacker can collect multiple shares of the same yield.

## Impact Details

* Over-distribution of yield to a single user (or small group).
* Dilution of rewards for honest token holders.
* Draining of the yield pool if unchecked over multiple distributions.

## References

<https://github.com/plumenetwork/contracts/blob/main/arc/src/ArcToken.sol#L310>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Attacker (A) owns 10,000 tokens out of a 100,000 total supply.
{% endstep %}

{% step %}

### First batch

`distributeYieldWithLimit()` is called with `startIndex = 0` and `maxHolders = 50`.

A receives their correct yield (10% share).
{% endstep %}

{% step %}

### Transfer before next batch

Before the next batch, attacker transfers tokens from A to address B.
{% endstep %}

{% step %}

### Subsequent batch

Next batch runs with `startIndex = 50`. B now receives another 10% share of the same `totalAmount`.
{% endstep %}

{% step %}

### Repeat

Repeat the transfer-and-wait process with addresses C, D, E... until the attacker claims yield multiple times from the same fixed total pool.
{% endstep %}
{% endstepper %}
