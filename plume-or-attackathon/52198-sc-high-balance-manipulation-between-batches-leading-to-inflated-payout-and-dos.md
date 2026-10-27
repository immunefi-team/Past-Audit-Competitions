# 52198 sc high balance manipulation between batches leading to inflated payout and dos

**Submitted on Aug 8th 2025 at 16:55:41 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52198
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The `ArcToken::distributeYieldWithLimit` function, which distributes rewards to token holders in batches, is vulnerable if token balances change between batch executions (due to mint, burn, transfers). An attacker can exploit this by transferring tokens from already-paid holders to unpaid holders between batches, inflating payouts and causing later batches to fail, leaving funds stuck.

<details>

<summary>Vulnerability Details (click to expand)</summary>

The `ArcToken::distributeYieldWithLimit` function is designed to distribute a fixed `totalAmount` of rewards among token holders in multiple batches to avoid exceeding gas limits when the number of holders is large.

The problem arises because each batch independently recalculates `effectiveTotalSupply` and reads the current `balanceOf` values directly from state at execution time. This creates a race condition between batches because balances can change in the time gap between executions.

For example, if a holder who has already been paid in batch 1 transfers tokens to a holder scheduled to be paid in batch 2, that second holder’s balance is inflated when batch 2 runs. Since the payout formula uses the updated balances and supply, the recipient’s proportional share is larger than it should be, effectively paying twice for the same tokens. This can deplete the remaining reward pool before all batches complete, causing later transfers to revert and leaving funds permanently stuck.

Relevant code snippet from `ArcToken::distributeYieldWithLimit`:

```solidity
// `ArcToken::distributeYieldWithLimit`
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

Because the function expects the total of all batch payouts to exactly match the original `totalAmount`, any mid-distribution balance change breaks the calculation and enables intentional DoS and freezing of funds.

</details>

## Impact Details

* Denial of Service: Any transfer between batches can cause later batches to revert, halting yield distribution entirely.
* Permanent Freezing of Funds: Rewards earmarked for unpaid holders can become locked and cannot be restored.

## Possible Solution

{% hint style="info" %}
Introduce a global state variable `distributionInProcess`.\
Set this variable to `true` at the start of the batched distribution process, and keep it `true` until the process completes.\
While `distributionInProcess` is `true`, all transfer, mint, and burn operations should be locked.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup and distribution start

1. There are 20 eligible holders. Distribution will be done in 2 batches of 10 each.
2. `distributeYieldWithLimit` is called with a `totalAmount` of rewards and will process holders 0–9 in batch 1 and 10–19 in batch 2.

Function signature:

```solidity
function distributeYieldWithLimit(
    uint256 totalAmount,
    uint256 startIndex,
    uint256 maxHolders
)
    external
    onlyRole(YIELD_DISTRIBUTOR_ROLE)
    nonReentrant
    returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed)
{
```

{% endstep %}

{% step %}

### Exploit between batches

3. Batch 1 completes and pays holders 0–9.
4. Before batch 2 executes, a holder from batch 1 (userA) transfers tokens to a holder in batch 2 (userB).
   {% endstep %}

{% step %}

### Inflated payout and failure

5. When batch 2 runs, the contract recalculates `effectiveTotalSupply` and reads `userB`’s updated `balanceOf(userB)`. The share calculation uses the updated balance and supply:

```solidity
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

6. The inflated share can consume more than the remaining `totalAmount`, causing transfers to revert or the distribution to be incomplete, leaving funds stuck.
   {% endstep %}
   {% endstepper %}
