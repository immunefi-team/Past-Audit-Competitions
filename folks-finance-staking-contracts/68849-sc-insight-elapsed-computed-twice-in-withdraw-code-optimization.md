# 68849 sc insight elapsed computed twice in withdraw code optimization&#x20;

**Submitted on Mar 11th 2026 at 16:09:21 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68849
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

### Brief/Intro

In `Staking._withdraw`, the expression `block.timestamp - userStake.unlockTime` is evaluated twice on consecutive lines when computing `accruedAmount` and `accruedReward`. This is a redundant computation: the same arithmetic result is produced both times from the same immutable inputs within a single function call. While this is not a security vulnerability, it is an avoidable gas inefficiency and a minor readability issue that can be eliminated with a one-line local variable.

### Vulnerability Details

The `_withdraw` internal function computes the elapsed time since unlock on two back-to-back lines:

```solidity
// src/Staking.sol#L317-L320
uint256 accruedAmount =
    _getAccrued(userStake.amount, userStake.unlockDuration, block.timestamp - userStake.unlockTime);
uint256 accruedReward =
    _getAccrued(userStake.reward, userStake.unlockDuration, block.timestamp - userStake.unlockTime);
```

Both calls pass the identical sub-expression `block.timestamp - userStake.unlockTime` as the third argument. Within the EVM, each evaluation of this expression re-reads `userStake.unlockTime` and performs the subtraction.

### Impact Details

This finding carries no direct security impact. No funds are at risk, and no invariant is violated. The consequence in the worst case (no compiler optimisation) is a marginal gas overhead on every `withdraw` call — one redundant subtraction instruction and one redundant read of `userStake.unlockTime`. In practice the overhead is negligible, but the fix also improves code clarity for future maintainers and auditors.

**Impact category:** Code Optimizations and Enhancements

### References

* Affected lines: [`https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L317–L320`](broken://pages/4f2a1f7450470c0da2fa6b97178e68c7e6cd96bf)

## Recommendation

Cache the sub-expression in a named local variable before the two `_getAccrued` calls:

```solidity
uint256 elapsed = block.timestamp - userStake.unlockTime;
uint256 accruedAmount = _getAccrued(userStake.amount, userStake.unlockDuration, elapsed);
uint256 accruedReward = _getAccrued(userStake.reward, userStake.unlockDuration, elapsed);
```

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";

contract ElapsedComputedTwiceTest is Test {
    uint256 constant AMOUNT          = 1_000e18;
    uint256 constant REWARD          = 100e18;
    uint256 constant UNLOCK_DURATION = 30 days;

    function setUp() public {
        vm.warp(10 * 365 days);
    }

    function _getAccrued(uint256 amount, uint256 duration, uint256 elapsed)
        internal pure returns (uint256)
    {
        uint256 e = elapsed < duration ? elapsed : duration;
        return (amount * e) / duration;
    }

    function test_GasComparison() public {
        uint256 unlockTime = block.timestamp - 15 days;

        // --- current: elapsed computed twice ---
        uint256 g0 = gasleft();
        uint256 a1 = _getAccrued(AMOUNT, UNLOCK_DURATION, block.timestamp - unlockTime);
        uint256 r1 = _getAccrued(REWARD, UNLOCK_DURATION, block.timestamp - unlockTime);
        uint256 gasCurrent = g0 - gasleft();

        // --- optimized: elapsed cached once ---
        uint256 g1 = gasleft();
        uint256 elapsed = block.timestamp - unlockTime;
        uint256 a2 = _getAccrued(AMOUNT, UNLOCK_DURATION, elapsed);
        uint256 r2 = _getAccrued(REWARD, UNLOCK_DURATION, elapsed);
        uint256 gasOptimized = g1 - gasleft();

        console.log("Gas (current   - elapsed computed twice):", gasCurrent);
        console.log("Gas (optimized - elapsed cached once)   :", gasOptimized);
        console.log("Gas saved per withdraw call             :", gasCurrent - gasOptimized);

        assertEq(a1, a2);
        assertEq(r1, r2);
    }
}
```

**Command:**

```bash
forge test --match-path "test/ElapsedComputedTwice.t.sol" -vv
```

**Output:**

```
Ran 1 test for test/ElapsedComputedTwice.t.sol:ElapsedComputedTwiceTest
[PASS] test_GasComparison() (gas: 9122)
Logs:
  Gas (current   - elapsed computed twice): 1425
  Gas (optimized - elapsed cached once)   : 1251
  Gas saved per withdraw call             : 174
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.32ms
```

Both variants produce identical results (`assertEq` passes), and the optimized path saves **174 gas** per `withdraw` call.
