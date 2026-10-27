# 52439 sc high dust accumulation in batched yield payouts leaves tokens stranded

**Submitted on Aug 10th 2025 at 17:50:42 UTC by @Afriauditor for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52439
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

`distributeYieldWithLimit` pays yield in batches but never reconciles integer-division remainders (“dust”) across the entire round. Because the function pulls the full `totalAmount` on the first batch and truncates per-holder payouts in each batch, the leftover dust remains in the contract with no sweep or final settlement. Over time this strands yield tokens and underpays holders.

### Vulnerability Details

In the batched path:

```solidity
if (startIndex == 0) {
    yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
}
...
// computed once per call, for all holders
uint256 effectiveTotalSupply = ... // sum of eligible balances

// for each holder in this batch only
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
yToken.safeTransfer(holder, share);
amountDistributed += share;
...
if (nextIndex == 0) {
    emit YieldDistributed(totalAmount, yieldTokenAddr);
}
```

* The full `totalAmount` is deposited on the first batch.
* Payouts use integer division per holder and per batch; truncation leaves a remainder.
* There is no logic at the end of the final batch (`nextIndex == 0`) to transfer the round’s remainder to anyone or to sweep it.
* Unlike the single-shot `distributeYield` (which pays the final remainder to the last holder), the batched version never assigns the cumulative leftover, so the deposited yield minus the sum of all per-holder share transfers (across all batches) sits in the contract.
* Because the contract has no sweep/withdraw for stuck yield and subsequent calls to `distributeYieldWithLimit` deposit new `totalAmount` (rather than using existing balance), the dust accumulates permanently.

### Impact Details

{% hint style="danger" %}
Permanent freezing of funds: The unallocated remainder of each batched distribution stays in the token contract with no recovery path.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup

1. Deploy and initialize `ArcToken` and a mock yield token; set the yield token.
2. Mint `1e18 ARC` to A and `1e18 ARC` to B (both eligible → `effectiveTotalSupply = 2e18`).
3. From the distributor, approve 3 units of the yield token for `ArcToken` to pull.
   {% endstep %}

{% step %}

### First batch

Call:

* `distributeYieldWithLimit(totalAmount=3, startIndex=0, maxHolders=1)`

Behavior:

* Because `startIndex == 0`, the contract pulls 3.
* It pays only A: `floor(3 * 1e18 / 2e18) = 1`.
* Return values: `nextIndex = 1`, `amountDistributed = 1`.
  {% endstep %}

{% step %}

### Second batch

Call:

* `distributeYieldWithLimit(totalAmount=3, startIndex=1, maxHolders=1)`

Behavior:

* It pays only B: `floor(3 * 1e18 / 2e18) = 1`.
* Returns `nextIndex = 0`, `amountDistributed = 1`.
  {% endstep %}

{% step %}

### Result

* Balances: A = 1, B = 1 → total paid = 2.
* But 3 were deposited initially → 1 unit remains in the `ArcToken` contract as dust.
* Repeat across rounds and dust accumulates.
  {% endstep %}
  {% endstepper %}

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
