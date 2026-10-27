# 50507 sc high non atomic yield distribution may lead to theft of yield

**Submitted on Jul 25th 2025 at 15:04:50 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50507
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The function `distributeYieldWithLimit()` allows the Yield Distributor to distribute the yield in a non-atomic way. This allows for users to transfer Arc Tokens to other addresses between the calls, doubling their yield for the same token balance.

### Vulnerability Details

`distributeYieldWithLimit()` rewards Arc Token holders based on their Arc Token balance, distributing yield proportional to their shares.\
When yield distribution is divided into separate transactions using `distributeYieldWithLimit()`, incorrect yield distribution can occur if Arc Token balances change between calls. This can happen due to benign transfers or malicious actions.

A malicious actor who already received their fair share during an earlier `distributeYieldWithLimit()` call can transfer their Arc Tokens to a different address that has not yet been processed; when that later address is processed, it will be credited again for the transferred balance. Additionally, if the Yield Token has an intrinsic callback (e.g., ERC-777), the same problem might occur even for an atomic `distributeYield()` call, because transfers lack a `nonReentrant` modifier.

### Impact Details

Malicious users can multiply their yields by a factor proportional to the number of `distributeYieldWithLimit()` calls (or distribution chunks), effectively stealing yield that should have gone to others.

## Proof of Concept

Consider this scenario:

* Address A (attacker) is among the first entries in the `holders` array and has 100 Arc Tokens.
* Address B (also attacker-controlled) appears later in the `holders` array (for example, in the second third if there are 1000 addresses) and has 1 Wei of Arc Tokens.

{% stepper %}
{% step %}

### First step

The Yield Distributor calls `distributeYieldWithLimit()` for the first third of `holders`. It iterates over address A and transfers yield proportional to A's 100 Arc Token balance.
{% endstep %}

{% step %}

### Second step

The attacker immediately transfers 100 Arc Tokens from address A to address B.
{% endstep %}

{% step %}

### Third step

The Yield Distributor calls `distributeYieldWithLimit()` for the second third of `holders`. It iterates over address B and transfers yield proportional to B's current 100 Arc Token balance. The transaction does not revert.
{% endstep %}

{% step %}

### Fourth step

The Yield Distributor calls `distributeYieldWithLimit()` for the final third of `holders`. This transaction reverts (or at minimum results in an incorrect final distribution) because addresses A and B combined have received more yield than they were entitled to.
{% endstep %}
{% endstepper %}

***

References:

* Target contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
