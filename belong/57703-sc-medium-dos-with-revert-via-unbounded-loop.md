# 57703 sc medium dos with revert via unbounded loop

**Submitted on Oct 28th 2025 at 10:07:09 UTC by @lllll for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57703
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Permanent freezing of funds

{% hint style="danger" %}
Vulnerability: Potential Denial of Service (DoS) — unbounded loop in \_removeAnySharesFor may cause transactions to run out of gas and revert, preventing legitimate withdrawals or other operations that rely on this function.
{% endhint %}

## Description

### Brief / Intro

The function \_removeAnySharesFor contains an unbounded loop that iterates over the stakes\[staker] array and removes shares until the requested shares amount is satisfied. If a user accumulates a very large number of stake entries (i.e., stakes\[staker].length becomes very large), calling \_removeAnySharesFor can consume an arbitrarily large amount of gas and eventually run out of gas and revert.

### Vulnerability Details

The implementation iterates over userStakes and removes elements via swap-and-pop or decrements an element. The loop condition is unbounded:

```js
for (uint256 i; i < userStakes.length && remaining > 0;) {
    ...
}
```

Each iteration can modify the array length, and in the worst case (many small stakes), the number of iterations grows with the number of entries. A malicious or inadvertent accumulation of many small stakes can therefore make the function revert due to gas exhaustion.

### Impact Details

If this function is used during withdraw/exit flows, affected users may be unable to withdraw staked assets, potentially resulting in permanent freezing of funds for those users.

## References

Vulnerable snippet:

```js
function _removeAnySharesFor(address staker, uint256 shares) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 remaining = shares;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        uint256 stakeShares = userStakes[i].shares;
        if (stakeShares <= remaining) {
            remaining -= stakeShares;
            userStakes[i] = userStakes[userStakes.length - 1];
            userStakes.pop();
            // don't ++i: a new element is now at index i
        } else {
            userStakes[i].shares = stakeShares - remaining;
            remaining = 0;
            unchecked {
                ++i;
            }
        }
    }
}
```

## Proof of Concept

```js
function _addStake(address staker, uint256 shares) external {
    stakes[staker].push(Stake({shares: shares, timestamp: block.timestamp}));
}

function _removeAnySharesFortest(address staker, uint256 shares) external {
    _removeAnySharesFor(staker, shares);
}

function testDoS_removeAnySharesFor() public {
    address staker = address(0x123);

    for(uint160 i = 0; i < 2000; i++) {
        staking._addStake(staker, 1e18);
    }
    
    uint256 gasBefore = gasleft();
    staking._removeAnySharesFortest(staker, 2000 * 1e18);
    uint256 gasUsed = gasBefore - gasleft();

    console.log("gas used:", gasUsed);
    assertLt(gasUsed, 1_000_000, "Gas too high, DoS potential!");
}
```
