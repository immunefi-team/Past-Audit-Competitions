# 52572 sc high a legitimate arc token holder can be denied his yield&#x20;

**Submitted on Aug 11th 2025 at 17:50:20 UTC by @swarun for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52572
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief/Intro

A legitimate arc token holder can be denied their yield when yield is distributed using the `distributeYieldWithLimit` function.

### Vulnerability Details

When there are restricted token holders (so they are skipped), the `distributeYieldWithLimit` function is used to skip those restricted holders. A non-restricted user can manipulate the holders' ordering (by performing a transfer that removes and re-adds them in the internal holders set) between distribution batches. By doing this after they have already received their share for the current run, the attacker can cause a later distribution batch to double-count balances relative to what the distributor expects, leading to a revert due to insufficient funds and denying yield to legitimate holders.

### Impact Details

Loss of yield for legitimate users.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L653>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario 1 — shifting a holder from earlier index to later index to cause revert

1. Suppose there are 10 holders and there is a restricted holder at index 8 (second last position).
2. The yield distributor calls `distributeYieldWithLimit` with `startIndex = 0` and `maxHolders = 8` to skip the restricted holder at index 8. This distributes to holders at indexes 0..7 (batch size = 8) and leaves index 9 pending.
3. A user at index 5 transfers their entire balance to themselves (calls `transfer` with `from == to`), which triggers the overridden `_update` implementation. Key parts:

```solidity
if (from != address(0)) {
    uint256 fromBalanceBefore = balanceOf(from);
    if (fromBalanceBefore == amount) {
        $.holders.remove(from);
    }
}

super._update(from, to, amount);

if (to != address(0) && balanceOf(to) > 0) {
    $.holders.add(to);
}
```

This remove-then-add changes the user's index (removed from index 5 and added to index 9), shifting the ordering of the holders array without changing its size.

4. The yield distributor now attempts to continue distribution starting at index 8 (to cover the remaining holders). Because of the index shift and prior distributions, the distributor's accounting of effective supply vs. already-transferred amounts can be inconsistent. The call will revert due to lack of funds (one user appears to double-claim), preventing the legitimate holder from receiving their yield.
   {% endstep %}

{% step %}

### Scenario 2 — small transfer to increase balance of a later batched recipient causing a revert

1. Suppose holders at indexes 0–5 are distributed first, index 6 is restricted, and indexes 7–9 are to be distributed next.
2. After the first batch (0–5) is distributed, an attacker transfers a small amount to a holder at index 8 (or otherwise increases their balance), which is allowed because that holder is not restricted.
3. The yield distributor then processes the batch for indexes 7–9. Because the balance of index 8 increased after some portion of the yield was already distributed according to the distributor's effective supply calculation, the later batch calculation may require more funds than are available in the contract (it appears as if some yields would be double-claimed). This causes a revert, denying yield to more than one legitimate holder.
4. The attacker only needs to increase the recipient's balance by a small amount to trigger the mismatch and revert, and can sandwich their transfer between distribution transactions.
   {% endstep %}
   {% endstepper %}

## Relevant Code Snippets

distributeYieldWithLimit:

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
    ArcTokenStorage storage $ = _getArcTokenStorage();
    address yieldTokenAddr = $.yieldToken;
    if (yieldTokenAddr == address(0)) {
        revert YieldTokenNotSet();
    }

    if (totalAmount == 0) {
        revert ZeroAmount();
    }

    uint256 supply = totalSupply();
    if (supply == 0) {
        revert NoTokensInCirculation();
    }

    totalHolders = $.holders.length();
    if (totalHolders == 0) {
        return (0, 0, 0);
    }

    if (startIndex >= totalHolders) {
        startIndex = 0;
    }

    uint256 endIndex = startIndex + maxHolders;
    if (endIndex > totalHolders) {
        endIndex = totalHolders;
    }

    uint256 batchSize = endIndex - startIndex;
    if (batchSize == 0) {
        return (0, totalHolders, 0);
    }

    uint256 effectiveTotalSupply = 0;
    for (uint256 i = 0; i < totalHolders; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) {
            effectiveTotalSupply += balanceOf(holder);
        }
    }

    if (effectiveTotalSupply == 0) {
        nextIndex = endIndex < totalHolders ? endIndex : 0;
        return (nextIndex, totalHolders, 0);
    }

    ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
    amountDistributed = 0;

    if (startIndex == 0) {
        yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
    }

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

    nextIndex = endIndex < totalHolders ? endIndex : 0;

    if (nextIndex == 0) {
        emit YieldDistributed(totalAmount, yieldTokenAddr);
    }

    return (nextIndex, totalHolders, amountDistributed);
}
```

\_update (key behavior that reorders holders):

```solidity
if (from != address(0)) {
    uint256 fromBalanceBefore = balanceOf(from);
    if (fromBalanceBefore == amount) {
        $.holders.remove(from);
    }
}

super._update(from, to, amount);

if (to != address(0) && balanceOf(to) > 0) {
    $.holders.add(to);
}
```

## Notes

* The core issue stems from relying on a mutable ordering of holders (the `$.holders` set/array), combined with batch-based distribution that computes an `effectiveTotalSupply` for the whole set at the start of the call but distributes in slices. If holders can be removed and re-added between batches (or their balances changed), the assumptions used to compute per-batch shares can be invalidated, causing reverts or mis-distribution.
* The attack requires the ability to perform transfers that alter either holder ordering or balances between distribution calls, and the attacker can use minimal amounts to cause the failure.
