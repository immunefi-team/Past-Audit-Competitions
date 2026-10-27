# 57245 sc medium needless iterations in for loops should be removed for better optimization and code maintenance

**Submitted on Oct 24th 2025 at 17:56:14 UTC by @Vanshika for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57245
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>

## Description

### Brief/Intro

Two internal functions in `Staking.sol` loop through a user’s array of stakes. Both functions loop through the entire array (which is unbounded), even when this is not needed.

### Vulnerability Details

`_consumeUnlockedSharesOrRevert()` and `_removeAnySharesFor()` in `Staking.sol` contain for loops that iterate over a user’s entire array of stakes. Each function tracks a `remaining` variable, which stores the number of shares still left to remove or consume. When `remaining` reaches 0 (i.e., the requested amount has been removed), the loop no longer needs to continue — but the current implementation still continues iterating through the (potentially large) remaining elements, performing unnecessary storage reads and math.

Relevant code snippets:

\_consumeUnlockedSharesOrRevert() (called by `withdraw()`):

```
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    Stake[] storage userStakes = stakes[staker];
    uint256 _min = minStakePeriod;
    uint256 nowTs = block.timestamp;
    uint256 remaining = need;

    for (uint256 i; i < userStakes.length && remaining > 0;) {
        Stake memory s = userStakes[i];
        if (nowTs >= s.timestamp + _min) {
            uint256 take = s.shares <= remaining ? s.shares : remaining;
            if (take == s.shares) {
                // full consume → swap and pop
                remaining -= take;
                userStakes[i] = userStakes[userStakes.length - 1];
                userStakes.pop();
                // don't ++i: a new element is now at index i
            } else {
                // partial consume
                userStakes[i].shares = s.shares - take;
                remaining = 0;
                unchecked {
                    ++i; // @audit loop should not continue if there are no shares remaining.
                }
            }
        } else {
            unchecked {
                ++i;
            }
        }
    }

    if (remaining != 0) revert MinStakePeriodNotMet();
}
```

\_removeAnySharesFor() (called by `emergencyWithdraw()`):

```
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
                ++i; // @audit loop should break here. 
            }
        }
    }
```

If a user has N items in the array and the required `remaining` amount is satisfied in an early element (or early few elements), the functions still continue and iterate up to the original loop bounds. Even though elements are popped and the array shrinks, the loop may perform additional unnecessary iterations (storage reads and writes) with `remaining == 0`.

## Impact Details

These loops do not make external calls and there is a practical limit to how many stakes a user will likely have, so this is not an Unbounded Gas Consumption vulnerability. Nonetheless, it is inefficient and wastes gas by performing avoidable storage access and computation. Fixing it improves gas usage and code clarity.

## Proof of Concept

Consider a user with 25 stakes. If the requested shares are fully satisfied by the first 2 stakes (which get removed), the array length becomes 23. However, the loop will continue iterating through the original bounds and perform additional reads/updates for the remaining indices even though `remaining` is already 0.

## Suggested Fix

When `remaining` becomes 0 (e.g., after a partial consume or reducing `remaining` to zero), break the loop instead of incrementing `i` and continuing. The following minimal change achieves that:

```diff
-                unchecked {
-                    ++i; 
-                }
+                break;
```

Applying this change (in both `_consumeUnlockedSharesOrRevert()` and `_removeAnySharesFor()`) prevents needless iterations after the requested shares have been satisfied.
