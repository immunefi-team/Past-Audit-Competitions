# 51658 sc high yield distribution in batches let the same tokens collect rewards in multiple batches stealing yield from other users

**Submitted on Aug 4th 2025 at 17:20:11 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51658
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

By opportunistically (e.g., front-running yield distribution batches) transferring ARC tokens between controlled addresses after each partial distribution in `ArcToken.distributeYieldWithLimit`, an attacker can claim an outsized share of yield without increasing net ownership of the underlying ARC token, depleting the pool that honest users rely on.

### Vulnerability Details

`ArcToken.distributeYieldWithLimit` iterates over the `holders` array in fixed-size batches. Because balance snapshots are not taken, an attacker can move tokens between addresses between successive batches so that the same underlying ARC token is counted for reward multiple times.

Notice the following snippet that showcases how the instantly held balance determines the distributed amount of yield:

```solidity
// arc/ArcToken.sol L530-L546
for (uint256 i = 0; i < batchSize; i++) {
            uint256 holderIndex = startIndex + i;
            address holder = $.holders.at(holderIndex);

            if (!_isYieldAllowed(holder)) {
                continue;
            }

            uint256 holderBalance = balanceOf(holder);
            if (holderBalance > 0) {
                uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
                if (share > 0) {
                    yToken.safeTransfer(holder, share);
                    amountDistributed += share;
                }
            }
        }
```

Effectively this means that the balances utilized for distribution of yield are the instant balances of ARC tokens held at the moment the distribution call is made for each batch. They are used with no snapshot taken at the beginning of the yield distribution nor a cooldown or checkpoints on transfers between batches — so the value may change throughout the whole distribution.

### Impact Details

This enables balance manipulation to siphon yield that should have gone to later-batched users. Users can transfer balances between `ArcToken.distributeYieldWithLimit` calls to earn an improper amount of yield.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Attacker controls a number of addresses that hold at least 1 wei of the ARCToken: A₁, A₂, …, Aₙ (n ≥ number of batches).
{% endstep %}

{% step %}

### Initial state

Before round 1, all ARC is in A₁.
{% endstep %}

{% step %}

### Batch 0 distribution

The distributor calls `distributeYieldWithLimit()`; batch 0 pays A₁.
{% endstep %}

{% step %}

### Transfer to next controlled address

The attacker immediately moves all ARC from A₁ → A₂.
{% endstep %}

{% step %}

### Batch 1 distribution

The distributor calls the function again (batch 1); A₂ now earns again.
{% endstep %}

{% step %}

### Repeat

Repeat the transfer between batches until all batches are processed.
{% endstep %}

{% step %}

### Result

Honest users receive a much smaller amount than they should; the attacker captures an improperly high amount of yield.
{% endstep %}
{% endstepper %}
