# 52041 sc low in arctoken attacker can reposition to last holder and capture entire yield remainder

**Submitted on Aug 7th 2025 at 14:00:24 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52041
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Vulnerability Details

In `ArcToken::distributeYield()`, any rounding remainder is awarded to the last address in the on-chain holders set.

A malicious holder can deliberately transfer their entire balance to an unused wallet causing their old address to be removed and the new one appended last. By front-running a yield distribution, the attacker would sweep the full dust amount.

### Impact Details

Although the amount that an attacker would gain could be low (it depends on yield token real value and decimals), the attack can be done easily by anyone who can transfer ArcTokens without restriction, and can be repeated.

Therefore the reporter defined the vulnerability as high.

## Recommended Mitigation

{% stepper %}
{% step %}

### Redirect remainder to neutral address

Redirect the rounding remainder to a neutral address (e.g., treasury or burn pool) instead of giving it to the last holder.
{% endstep %}

{% step %}

### Use higher-precision arithmetic

Use higher-precision arithmetic to minimize dust and eliminate the need for remainder handling.
{% endstep %}
{% endstepper %}

## Proof of Concept

This is the relevant snippet of `distributeYield()`, where distribution to last holder is performed:

```solidity
function distributeYield(
    uint256 amount
) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
    ...
    uint256 lastProcessedIndex = holderCount > 0 ? holderCount - 1 : 0;
    for (uint256 i = 0; i < lastProcessedIndex; i++) {
     ...
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

This is the overridden implementation in `ArcToken` of the internal function `_update()` which is called during common ERC20 transfers. It's clear that if ArcTokens funds are moved to a new address, this one is inserted in the `holders` array as the last element:

```solidity
// Override _update to track holders and enforce transfer restrictions via router/modules
function _update(address from, address to, uint256 amount) internal virtual override {
    ArcTokenStorage storage $ = _getArcTokenStorage();

    ...
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
            catch { /* Ignore if hook not implemented or fails? */ }
    }
}
```

This is the implementation of `EnumerableSet::add()`:

```solidity
/**
 * @dev Add a value to a set. O(1).
 *
 * Returns true if the value was added to the set, that is if it was not
 * already present.
 */
function _add(Set storage set, bytes32 value) private returns (bool) {
    if (!_contains(set, value)) {
        set._values.push(value);
        // The value is stored at length-1, but we add 1 to all indexes
        // and use 0 as a sentinel value
        set._positions[value] = set._values.length;
        return true;
    } else {
        return false;
    }
}
```
