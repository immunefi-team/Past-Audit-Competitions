# 52303 sc insight incorrect yield distribution event emission

**Submitted on Aug 9th 2025 at 16:03:03 UTC by @TheCarrot for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52303
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In `distributeYieldWithLimit` the contract emits `YieldDistributed(totalAmount, yieldTokenAddr)` when `nextIndex == 0`. However, the function actually transfers `amountDistributed` (which can be < `totalAmount` due to integer-division rounding or restricted holders). The event therefore can report an amount that was not actually sent.

### Vulnerability Details

Inside `distributeYieldWithLimit` the code accumulates `amountDistributed` as it transfers per-holder shares:

```solidity
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
if (share > 0) {
    yToken.safeTransfer(holder, share);
    amountDistributed += share;
}
...
if (nextIndex == 0) {
    emit YieldDistributed(totalAmount, yieldTokenAddr); // <-- incorrect
}
```

The bug: the emitted event uses `totalAmount` (the amount taken from the distributor at the start of a distribution run) rather than `amountDistributed` (the sum of actual transfers performed). When rounding (integer division) or restricted accounts reduce or zero-out per-holder shares, `amountDistributed` can be strictly less than `totalAmount`, producing a discrepancy between the event and on-chain token movements.

### Impact Details

* Emits incorrect distributed amount in final batch
* Creates discrepancy between actual transfers and event logs
* May mislead off-chain monitoring systems

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L551>

## Proof of Concept

A step-by-step reproducible scenario showing the mismatch between event and actual transfers:

{% stepper %}
{% step %}

### Setup test tokens

* Deploy a simple ERC20 test token `YieldToken` and mint 1 unit to the distributor address `D`.
* Ensure `D` has approved the ArcToken contract to spend 1 `YieldToken` (approve `ArcToken` for 1).
  {% endstep %}

{% step %}

### Setup ArcToken holders

* Ensure the ArcToken contract has two holders `H1` and `H2` recorded in its `holders` set, each with ArcToken balances of `1` token (so `balanceOf(H1) = 1` and `balanceOf(H2) = 1`).
* Ensure `H1` and `H2` are eligible for yield calculation (i.e., not both restricted) so that `effectiveTotalSupply = 2`.
  {% endstep %}

{% step %}

### Ensure distributor role

* Give `D` the `YIELD_DISTRIBUTOR_ROLE` on the ArcToken contract.
  {% endstep %}

{% step %}

### Call the function

From `D`, call:

```js
distributeYieldWithLimit(1 /* totalAmount */, 0 /* startIndex */, 2 /* maxHolders */)
```

{% endstep %}

{% step %}

### Observe execution behavior

* At function start (because `startIndex == 0`) the contract executes `yToken.safeTransferFrom(D, address(this), 1)`, contract now holds 1 `YieldToken`.
* For each holder: compute `share = (totalAmount * holderBalance) / effectiveTotalSupply`. Here `share = (1 * 1) / 2 = 0` due to integer division truncation. Therefore neither holder receives any `YieldToken` and `amountDistributed` remains 0.
  {% endstep %}

{% step %}

### Event vs reality

* Because `nextIndex == 0` (we processed all holders in the single batch), the contract emits:

```solidity
YieldDistributed(1, yieldTokenAddr)
```

which claims `1` was distributed.

* But in reality `amountDistributed == 0` and no `safeTransfer` calls were made to `H1` or `H2`. The contract balance remains 1 `YieldToken`.
  {% endstep %}
  {% endstepper %}
