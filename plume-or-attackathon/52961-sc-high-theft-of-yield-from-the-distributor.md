# 52961 sc high theft of yield from the distributor&#x20;

**Submitted on** Aug 14th 2025 at 13:43:41 UTC by @heeze for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52961
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Theft of unclaimed yield

## Description

Brief/Intro

The `ArcToken.distributeYieldWithLimit` function resets `startIndex` to `0` when `startIndex >= totalHolders`. Because the function also pulls the full `totalAmount` from the distributor whenever `startIndex == 0`, a malicious reordering/shrinking of the holders set between batches can force a fresh pull of `totalAmount` and restart the batch at index `0`. Attackers who position themselves within the first `maxHolders` then receive yield again, potentially with an increased balance. This drains additional funds from the distributor and allows double collection of rewards.

## Vulnerability Details

{% stepper %}
{% step %}

### Silent restart

The function resets the index back to zero when the provided `startIndex` is no longer less than `totalHolders`:

```solidity
if (startIndex >= totalHolders) {
    startIndex = 0;
}
```

If the holders set shrinks or is reordered between batches (for example, addresses transfer out to drop below the prior nextIndex), the function silently restarts from index 0.
{% endstep %}

{% step %}

### Funds pulled again on index 0

Whenever `startIndex == 0`, the contract pulls the full `totalAmount` from the distributor:

```solidity
if (startIndex == 0) {
    yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
}
```

Any call that begins at index 0 will pull `totalAmount` anew, even if a previous batch already pulled and partially distributed funds. These two behaviors together allow re-entrance of a distribution that re-pulls funds and redistributes to the first holders again.
{% endstep %}
{% endstepper %}

This results in two main attack vectors:

* Distributor griefing: Shrinking/reordering the holders array between batches causes a reset and an extra pull of `totalAmount`, draining the distributor.
* Double yield & balance manipulation: Attackers ensure their addresses are within the first `maxHolders` after the reset and can increase balances before restart to collect yield twice (and larger the second time).

## Impact Details

* Attackers can force transfers of the full `totalAmount` for the same yield cycle, draining protocol or distributor funds.
* Addresses within the first `maxHolders` after a reset receive yield twice for the same distribution; attackers can maximize their share by increasing balances before the restart.
* Remaining funds pulled from the distributor may become stuck in the contract.
* Yield accounting becomes unreliable; funds intended for distribution can be stolen or misallocated.
* Over time, this can lead to significant financial losses and loss of trust in the protocol’s yield mechanism.

## References

* [ArcToken.sol - distributeYieldWithLimit function](https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L466)
* ArcToken.sol - \_update function (<https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L692C9-L703C10>)

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Deploy ArcToken and mint tokens to 60 holders: H1, H2, ..., H60, each with 100 tokens.
* Set `maxHolders = 15`, `totalAmount = 60,000` (so each holder is supposed to receive 1,000 tokens per distribution: 60,000 / 60).
  {% endstep %}

{% step %}

### First Batch Distribution

* Distributor calls `distributeYieldWithLimit(totalAmount=60,000, startIndex=0, maxHolders=15)`.
* Contract pulls 60,000 tokens from the distributor.
* Iterates over H1–H15: each receives (60,000 \* 100) / (60 \* 100) = 1,000 tokens.
* Returns `nextIndex = 15`.
  {% endstep %}

{% step %}

### Second Batch Call

* Distributor calls `distributeYieldWithLimit(totalAmount=60,000, startIndex=15, maxHolders=15)`.
* Processes H16–H30: each receives 1,000 tokens.
* Returns `nextIndex = 30`.
  {% endstep %}

{% step %}

### Third Batch Call

* Distributor calls `distributeYieldWithLimit(totalAmount=60,000, startIndex=30, maxHolders=15)`.
* Processes H31–H45: each receives 1,000 tokens.
* Returns `nextIndex = 45`.
  {% endstep %}

{% step %}

### Attack Preparation & Trigger

* Before the next batch, holders H46–H60 transfer all their tokens to H1. H1 now has 100 + (15 \* 100) = 1,600 tokens; H46–H60 have 0.
* The holders array effectively shrinks to H1–H45 (`totalHolders = 45`).
* Since `startIndex (45) >= totalHolders (45)`, the function resets `startIndex` to 0 and pulls another 60,000 tokens from the distributor.
* Iterates over H1–H15 (again, including H1 with the increased balance):
  * H1 receives (60,000 \* 1600) / (6000) = 16,000 tokens.
  * Each of H2–H15 receives 1,000 tokens.
* H1 and others in the first batch have now received yield twice for the same epoch; H1 disproportionately benefits.
  {% endstep %}

{% step %}

### Repeat

* The attacker can repeat this process for subsequent distributions, draining the distributor and double collecting yield as often as practical.
  {% endstep %}
  {% endstepper %}
