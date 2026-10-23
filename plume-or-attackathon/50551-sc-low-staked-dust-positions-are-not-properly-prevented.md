# 50551 sc low staked dust positions are not properly prevented

**Submitted on Jul 25th 2025 at 23:27:47 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50551
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Unbounded gas consumption

## Description

### Brief / Intro

While the staking mechanism validates that amounts below *minStakeAmount* cannot be directly staked, it is still possible to have less than the minimal amount staked by partially unstaking.

### Vulnerability Details

When calling `stake()`, `restake()` or `stakeOnBehalf()`, the staked amount is eventually compared to *minStakeAmount*, reverting if it’s smaller than that minimum. However, by first calling `stake` for an amount larger than that minimum and then calling `unstake()` for almost all of what was staked, a staked position with a dust value would remain without `handlePostUnstakeCleanup()` being called.

### Impact Details

While there’s no immediate impact (beyond increased gas consumption for anyone iterating over those staked positions), it’s not advisable to allow staked positions to hold amounts smaller than the designated minimum. A malicious user could purposely open many dust positions (albeit needing to wait the cooldown period to retrieve the unstaked amounts), potentially using it as a way to spam, incur greater gas costs or cause DoS if it reaches levels beyond the block gas limit.

### Severity

While this issue could potentially lead to an impact classified as “Medium” (Unbounded gas consumption), practical limitations together with economic disincentives and a lack of any clear motive for the attacker make it an issue that would be better classified as “Low” (or even an “insight”).

## Proof of Concept

{% stepper %}
{% step %}

### Step

User creates many staked positions from many different addresses for a single *validatorId*, staking *minStakeAmount* at a time.
{% endstep %}

{% step %}

### Step

User unstakes `minStakeAmount - 1` from these positions.
{% endstep %}

{% step %}

### Step

User withdraws the funds back after the cooling period.
{% endstep %}

{% step %}

### Step

User repeats the previous steps many times.
{% endstep %}
{% endstepper %}

<details>

<summary>Impact when iterating over stakers list</summary>

While a user iterates over:

address\[] storage stakersList = $.validatorStakers\[validatorId];

the amount of gas consumed can become extremely large, leading to increased costs.

</details>
