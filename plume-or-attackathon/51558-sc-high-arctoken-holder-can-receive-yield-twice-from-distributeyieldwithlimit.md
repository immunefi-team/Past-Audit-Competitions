# 51558 sc high arctoken holder can receive yield twice from distributeyieldwithlimit&#x20;

**Submitted on Aug 3rd 2025 at 23:36:52 UTC by @kaysoft for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51558
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

{% hint style="warning" %}
High-severity issue: a holder can be paid yield twice by manipulating their position in the holders array between batched calls to `distributeYieldWithLimit(...)`.
{% endhint %}

## Description

### Brief / Intro

`distributeYieldWithLimit(...)` is used to distribute yield in batches to holders of ArcToken. It assumes batch calls occur in ordered succession (startIndex from 0 to last index). However, because blockchain transaction ordering is not guaranteed, an address can be removed and re-added to the `holders` array between batch calls — changing its index. This allows the same holder to be included in multiple batches and receive yield twice.

The `holders` are stored in an array-backed Set. When a holder's balance goes to zero they are removed (pop) from the array, and when they receive tokens they are pushed to the end of the array. If a holder is paid in an early batch, then empties and refills their balance before later batches, they can be paid again.

### Relevant README excerpt

From arc/Readme.md (Yield Distribution Mechanism):

> Batched Distribution (distributeYieldWithLimit): For tokens with a larger number of holders, this function allows for paginated distribution. An off-chain script or keeper can call this function repeatedly in batches, using the startIndex and maxHolders parameters to process a subset of holders in each transaction. The function returns the nextIndex to be used in the subsequent call, allowing the process to continue until all holders have been paid. This significantly reduces the gas cost per transaction and avoids hitting block gas limits.

Link: <https://github.com/plumenetwork/contracts/blob/main/arc/README.md?utm\\_source=immunefi#yield-distribution-mechanism>

## Vulnerability Details

<details>

<summary>Click to expand vulnerability details and relevant code</summary>

The `_update(...)` function is called during transfers and may remove an address from the `holders` set when their balance hits zero, then add it back when the address receives tokens:

```solidity
File: ArcToken.sol
function _update(address from, address to, uint256 amount) internal virtual override {
        ArcTokenStorage storage $ = _getArcTokenStorage();

        bool transferAllowed = true;

        address routerAddr = $.restrictionsRouter;
        if (routerAddr == address(0)) {
            revert RouterNotSet();
        }

        address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
        if (specificTransferModule != address(0)) {
            transferAllowed =
                transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
        }

        address globalTransferModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
        if (globalTransferModule != address(0)) {
            try ITransferRestrictions(globalTransferModule).isTransferAllowed(from, to, amount) returns (
                bool globalAllowed
            ) {
                transferAllowed = transferAllowed && globalAllowed;
            } catch {
                transferAllowed = false;
            }
        }

        if (!transferAllowed) {
            revert TransferRestricted();
        }

        if (specificTransferModule != address(0)) {
            ITransferRestrictions(specificTransferModule).beforeTransfer(from, to, amount);
        }
        if (globalTransferModule != address(0)) {
            try ITransferRestrictions(globalTransferModule).beforeTransfer(from, to, amount) { }
                catch { /* Ignore if hook not implemented or fails? */ }
        }

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

        if (specificTransferModule != address(0)) {
            ITransferRestrictions(specificTransferModule).afterTransfer(from, to, amount);
        }
        if (globalTransferModule != address(0)) {
            try ITransferRestrictions(globalTransferModule).afterTransfer(from, to, amount) { }
                catch { /* Ignore if hook not implemented or fails? */ }//@audit no-op catch block
        }
    }
```

Because batched distribution uses indices over this mutable `holders` array, an address removed and re-added between batches can be included twice across different batches.

</details>

## Impact Details

A malicious or colluding user can receive yield twice (double payment) by emptying then refilling their ArcToken balance between batched `distributeYieldWithLimit(...)` calls. This results in theft of yield (overpayment) relative to other holders.

## Recommendation

Consider mechanisms to prevent holders changing their place in the holders list during a distribution run. Options include (but are not limited to):

* Pause ArcToken transfers while a multi-transaction distribution is being executed (e.g., a distribution mode / paused flag) so indices remain stable until distribution completes.
* Or redesign distribution to be resilient against reordering by using a stable snapshot (e.g., snapshot of holders and balances in storage or via ERC-721-like epoched snapshots) and referencing that snapshot during batched distribution.
* Or mark holders as already-paid for the distribution round (e.g., by tracking lastPaidRound for each holder) so re-added addresses cannot be paid again within the same distribution round.

(Do not add any implementation beyond the above high-level mitigations — keep changes minimal and aligned with the project architecture.)

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

* Bob is a holder of 1000 Arc tokens.
* Bob's address is index 0 in the `holders` array of length 100,000.
* The `YIELD_DISTRIBUTOR_ROLE` will run 10 batched `distributeYieldWithLimit(...)` calls to distribute yield (100 USDC per holder).
  {% endstep %}

{% step %}

### First batch

* The distributor executes the first batch; Bob at index 0 receives 100 USDC.
  {% endstep %}

{% step %}

### Manipulation between batches

* Before the final batch executes, Bob performs a transaction that:
  * Transfers out all his Arc tokens to another address (causing removal from `holders`), then
  * Transfers Arc tokens back to his original address (re-adding to `holders` at the end).
* As a result, Bob's index moves from the beginning to the end of the array.
  {% endstep %}

{% step %}

### Final batch

* The distributor executes the last batch. Because Bob was re-added and now appears later in the `holders` array, he is included in another batch and receives another 100 USDC.
* Total Bob received: 200 USDC instead of 100 USDC.
  {% endstep %}
  {% endstepper %}

## References

* README link (unchanged): <https://github.com/plumenetwork/contracts/blob/main/arc/README.md?utm\\_source=immunefi#yield-distribution-mechanism>

***
