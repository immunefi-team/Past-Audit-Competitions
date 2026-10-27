# 53038 sc low distributeyield can be frontrun to sandwich rewards we can force ourselves to be the last holder and get unfairly big bonuses

**Submitted on Aug 14th 2025 at 17:50:12 UTC by @valkvalue for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53038
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

`distributeYield` can be frontrun to sandwich rewards; an adversary can force themselves to be the last holder which would give them an unfairly large bonus.

### Vulnerability Details

The current `distributeYield` implementation for `ArcToken` allows an adversary to sandwich rewards by becoming a holder of the token immediately before yield is distributed. The adversary can also aim to be the last holder, since the last holder receives any leftover funds that were not distributed due to some other holders being ineligible for yield.

Relevant excerpt:

```solidity
    function distributeYield(
        uint256 amount
    ) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {

......
        ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
        yToken.safeTransferFrom(msg.sender, address(this), amount);

        uint256 distributedSum = 0;
        uint256 holderCount = $.holders.length();
        if (holderCount == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return;
        }

        uint256 effectiveTotalSupply = 0;
        for (uint256 i = 0; i < holderCount; i++) {
            address holder = $.holders.at(i);
            if (_isYieldAllowed(holder)) {
                effectiveTotalSupply += balanceOf(holder);
            }
        }

        if (effectiveTotalSupply == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return;
        }

        uint256 lastProcessedIndex = holderCount > 0 ? holderCount - 1 : 0;
        for (uint256 i = 0; i < lastProcessedIndex; i++) {
            address holder = $.holders.at(i);

            if (!_isYieldAllowed(holder)) {
                continue;
            }

            uint256 holderBalance = balanceOf(holder);
            if (holderBalance > 0) {
                uint256 share = (amount * holderBalance) / effectiveTotalSupply;
                if (share > 0) {
                    yToken.safeTransfer(holder, share);
                    distributedSum += share;
                }
            }
        }

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

    }
```

Because the final share is computed as the remainder (amount - distributedSum) and sent to the last holder in the holders list, an adversary who ensures they are the last holder when `distributeYield` runs will receive any rounding leftovers plus any undistributed portions caused by ineligible holders.

To become the last holder:

* Acquire tokens from the market or transfer tokens to a fresh address you control, which will be appended to the holders set/list by the contract's `_update()` logic.

## Impact Details

{% hint style="warning" %}
The vulnerability enables theft of unclaimed yield by front-running or sandwiching the `distributeYield` operation. The attacker may receive an outsized share by ensuring they are the last processed holder.
{% endhint %}

## References

* Implementation link: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L448-L457>
* Target in report header: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Observe distribution

An oracle or mempool watcher sees a pending `distributeYield` transaction submitted by an account with `YIELD_DISTRIBUTOR_ROLE`.
{% endstep %}

{% step %}

### Step 2 — Become a holder (front-run)

Before the `distributeYield` transaction is mined, acquire some ArcToken (from market or by transferring to a fresh address you control). The contract will add the new address to the holders list via `_update()`.
{% endstep %}

{% step %}

### Step 3 — Optionally ensure you are the last holder

If necessary, create a fresh address and transfer tokens to it as the final transaction before the distributor’s `distributeYield` executes, making that fresh address the last entry in the holders set/list.
{% endstep %}

{% step %}

### Step 4 — Distributor executes

When `distributeYield` runs, the function calculates shares for all but the last holder using eligible balances and sends each computed share. The last holder receives `amount - distributedSum`, which includes any leftover or undistributed portions resulting from ineligible holders or rounding.
{% endstep %}
{% endstepper %}

## Notes

* The PoC demonstrates how typical mempool front-running or sandwiching can be used to exploit the remainder logic that favors the last holder.
* Links above are preserved as in the original report.
