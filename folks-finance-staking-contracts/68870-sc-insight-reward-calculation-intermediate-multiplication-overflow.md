# 68870 sc insight reward calculation intermediate multiplication overflow

**Submitted on Mar 11th 2026 at 18:15:40 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68870
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

### Brief/Intro

In `Staking._stake`, the reward is computed by multiplying three terms — `amount`, `aprBps`, and `stakingDurationSeconds` — before dividing by the denominator. In Solidity 0.8.x, this multiplication reverts on overflow. For sufficiently large inputs the intermediate product exceeds `type(uint256).max`, causing every `stake()` call for that input combination to revert unexpectedly and permanently denying the staking service to affected users. `Math.mulDiv` (which uses 512-bit intermediate arithmetic and never overflows) is already imported via OpenZeppelin (`import {Math} from "@openzeppelin/contracts/utils/math/Math.sol"`), so this is a one-line change with no new dependencies.

### Vulnerability Details

The affected computation is at `Staking.sol#L267–L268`:

```solidity
// src/Staking.sol#L267-L268
uint256 rewardBpsDenominator = 1e4 * 365 days;
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;
```

The expression evaluates left-to-right:

1. `amount * stakingPeriod.aprBps` — first intermediate product
2. `(result) * stakingPeriod.stakingDurationSeconds` — second intermediate product
3. `/ rewardBpsDenominator` — division applied only after both multiplications

In Solidity 0.8.x, step 2 reverts with a panic if the product exceeds `type(uint256).max`. The division that would reduce the value never executes.

As APR or duration increase, the threshold decreases and becomes reachable at lower amounts. The fix — `Math.mulDiv` — is already imported in the contract and performs the equivalent `(a * b) / c` calculation without intermediate overflow.

### Impact Details

**Impact category:** Security Best Practices

### References

* Affected lines: [`Staking.sol#L267–L268`](broken://pages/4f2a1f7450470c0da2fa6b97178e68c7e6cd96bf)
* OpenZeppelin `Math.mulDiv`: already imported at `Staking.sol#L13`

## Recommendation

Replace the chained multiplication with `Math.mulDiv`, which computes `floor(a × b / denominator)` using 512-bit intermediate arithmetic and never overflows:

```solidity
uint256 rewardBpsDenominator = 1e4 * 365 days;
uint256 reward = Math.mulDiv(amount, stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds, rewardBpsDenominator);
```

`Math.mulDiv` is already imported via OpenZeppelin (`import {Math} from "@openzeppelin/contracts/utils/math/Math.sol"`), so this is a one-line change with no new dependencies.

## Proof of Concept

**File:** `report/poc/test/RewardOverflow.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

contract RewardOverflowTest is Test {
    uint256 constant REWARD_BPS_DENOMINATOR = 1e4 * 365 days; // 315_360_000_000

    uint32  constant APR_BPS  = 10_000;   // 100% APR
    uint64  constant DURATION = 365 days; // 31_536_000 s

    // One unit above the safe threshold — overflows.
    uint256 constant OVERFLOW_AMOUNT = type(uint256).max / (uint256(APR_BPS) * DURATION) + 1;

    // External wrapper so vm.expectRevert can catch the arithmetic panic.
    function rewardCurrent(uint256 amount) external pure returns (uint256) {
        // mirrors Staking.sol#L267-L268 exactly
        return (amount * APR_BPS * DURATION) / REWARD_BPS_DENOMINATOR;
    }

    function test_Comparison() public {
        console.log("amount              :", OVERFLOW_AMOUNT);

        // --- current: panics ---
        vm.expectRevert(); // arithmetic overflow (panic 0x11)
        this.rewardCurrent(OVERFLOW_AMOUNT);
        console.log("current code        : REVERT (arithmetic overflow)");

        // --- OZ Math.mulDiv: succeeds ---
        uint256 reward = Math.mulDiv(OVERFLOW_AMOUNT, uint256(APR_BPS) * DURATION, REWARD_BPS_DENOMINATOR);
        console.log("Math.mulDiv result  :", reward);
    }
}
```

**Command:**

```bash
forge test --match-path "test/RewardOverflow.t.sol" -vv
```

**Output:**

```
Ran 1 test for test/RewardOverflow.t.sol:RewardOverflowTest
[PASS] test_Comparison() (gas: 13579)
Logs:
  amount              : 367174306308080274681541682549111833629090514540970839800410908194
  current code        : REVERT (arithmetic overflow)
  Math.mulDiv result  : 367174306308080274681541682549111833629090514540970839800410908194

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.62ms
```
