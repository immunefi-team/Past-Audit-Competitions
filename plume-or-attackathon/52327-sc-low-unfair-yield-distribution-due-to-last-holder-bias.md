# 52327 sc low unfair yield distribution due to last holder bias

**Submitted on Aug 9th 2025 at 21:24:32 UTC by @ZeroXGondar for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52327
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Summary

In `ArcToken::distributeYield` any remaining amount is given to the last holder. This creates unfairness in yield distribution and can also enable a “Zero Balance Dance” (described below) where holders attempt to become the last holder to receive the extra share.

### Details

The following block shows that any remaining amount is transferred to the last holder:

```solidity
      if (holderCount > 0) {
            address lastHolder = $.holders.at(lastProcessedIndex);
            if (_isYieldAllowed(lastHolder)) {
                uint256 lastShare = amount - distributedSum;
                if (lastShare > 0) {
                    yToken.safeTransfer(lastHolder, lastShare);
                    distributedSum += lastShare;
                }
            }
        }
```

This results in unfairness because the last holder may receive more yield than other holders. Two main problems arise:

* Holders can attempt to manipulate the holders array ordering to become the last holder and capture the excess.
* This can create network-level competition ("gas wars") where many users repeatedly transfer tokens to become last in the array.

### Impact details

* Zero Balance Dance Attack: Users can transfer all tokens to another address they control (triggering `holders.remove()`), then transfer back (triggering `holders.add()` at the end of array). The contract's `_update` logic shows this behavior:

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

Because a removed-and-readded address is placed at the end of the holders structure, users can repeatedly perform such transfers (or monitor and frontrun distribute calls) to become the last holder and capture the extra yield. If many users adopt this strategy, it can cause excessive on-chain activity and congestion.

### Objection: transfers can be restricted by admin

Transfers may be restricted by an admin via a whitelisting module, which would mitigate the attack. However, restricting transfers impacts usability and requires trusted management of whitelisted accounts. The underlying unfair distribution logic remains even if transfers are restricted.

Recommendation (from reporter): If there is an unavoidable remainder, consider returning it to the protocol (e.g., collect into a reserve) and include it in the next distribution rather than awarding it to the last holder.

Severity rationale: Low — the contract fails to deliver promised returns fairly, but funds are not lost.

## Proof of Concept

<details>

<summary>PoC: Sequence demonstrating the unfairness and the Zero Balance Dance</summary>

SetupAlice, Bob, and Charlie are token holders. Assume Charlie is currently the last holder in the holders structure.Distribution with remainderYield is distributed. Because there's an excess remainder after per-holder shares are computed, that remainder is transferred to the last holder (Charlie), giving Charlie more yield than Alice or Bob.Incentive to manipulateSeeing this, holders attempt to become the last holder. A holder can:Move all tokens to another address they control (causing their address to be removed from the holders structure).Move tokens back to that address (causing it to be re-added and placed at the end of the holders structure). This pattern (Zero Balance Dance) places them as last holder and eligible for the remainder.Amplification: gas warsMultiple holders may compete to be the last holder, sending many transactions to reorder the holders array. If the excess yield is larger than the gas cost, participants can economically justify these transfers, leading to network congestion and wasted gas.

</details>

## Notes

* Admin transfer restrictions can mitigate the attack surface, but do not remove the unfairness in distribution logic.
* Suggested resolution: do not assign the remainder to the last holder; instead, collect it for future distributions or protocol reserves.
