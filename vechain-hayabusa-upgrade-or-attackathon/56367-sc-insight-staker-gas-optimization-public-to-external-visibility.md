# 56367 sc insight staker gas optimization public to external visibility

**Submitted on Oct 15th 2025 at 06:25:52 UTC by @caslo for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56367
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/blob/release/hayabusa/builtin/gen/staker.sol>
* **Impacts:** Gas inefficiency (low severity)

{% hint style="info" %}
Classification: Gas Optimization / Low Severity - Informational
{% endhint %}

## Brief / Intro

The `totalStake()` and `queuedStake()` view functions in the Staker contract are declared with `public` visibility but are never called internally within the contract. Using `public` instead of `external` for these functions results in unnecessary gas costs for users querying stake information. These functions could be declared as `external` to reduce gas consumption by 41–70 gas per call.

## Vulnerability Details

In Solidity, `public` functions can be called both internally and externally, while `external` functions can only be called externally. Marking a function `public` causes the compiler to generate extra code to handle internal calls and copy parameters to memory, which can increase gas usage when only external calls are needed.

Current implementation in `staker.sol`:

```solidity
// Line 33-35
function totalStake() public view returns (uint256 totalVET, uint256 totalWeight) {
    return StakerNative(address(this)).native_totalStake();
}

// Line 40-42
function queuedStake() public view returns (uint256 queuedVET) {
    return StakerNative(address(this)).native_queuedStake();
}
```

Analysis of the entire contract (396 lines) confirms:

* Neither `totalStake()` nor `queuedStake()` are called internally within the contract
* Both functions only delegate to native calls and serve as external interfaces
* No other functions in the contract reference these functions

Gas impact verified through testing (Foundry):

```
totalStake():
  PUBLIC:   5913 gas
  EXTERNAL: 1524 gas
  Savings:  4389 gas (74% reduction)

queuedStake():
  PUBLIC:   1414 gas
  EXTERNAL: 1373 gas
  Savings:  41 gas (2.9% reduction)
```

## Impact Details

* Type: Gas Inefficiency (Low Severity)
* User Impact:
  * Every external call to `totalStake()` wastes \~70 gas unnecessarily
  * Every external call to `queuedStake()` wastes \~41 gas unnecessarily
  * These are query functions likely called frequently by validators, delegators, and monitoring systems
* Financial Impact:
  * No funds are at risk
  * No security breach is possible
  * Impact is limited to slightly higher transaction costs for users
* Cumulative Cost:
  * If called thousands of times per day, cumulative wasted gas could be meaningful over time, but not critical

## Recommended Fix

Change visibility from `public` to `external`:

```diff
- function totalStake() public view returns (uint256 totalVET, uint256 totalWeight) {
+ function totalStake() external view returns (uint256 totalVET, uint256 totalWeight) {
     return StakerNative(address(this)).native_totalStake();
 }

- function queuedStake() public view returns (uint256 queuedVET) {
+ function queuedStake() external view returns (uint256 queuedVET) {
     return StakerNative(address(this)).native_queuedStake();
 }
```

Benefits:

* Reduced gas costs for all callers
* No breaking changes (ABI remains compatible)
* Follows Solidity best practices
* Signals clear intent that functions are external-only

## Test Verification

All tests can be run with:

```bash
forge test --match-path test/StakerViewVisibility.t.sol -vv
```

<details>

<summary>Test Output</summary>

```
========================================
Gas Comparison: PUBLIC vs EXTERNAL
========================================

totalStake():
  PUBLIC:   5913 gas
  EXTERNAL: 1524 gas
  Savings:  4389 gas (74%)

queuedStake():
  PUBLIC:   1414 gas
  EXTERNAL: 1373 gas
  Savings:  41 gas (2%)

========================================

Suite result: ok. 7 passed; 0 failed; 0 skipped
```

</details>

## References

* Test Implementation: `test/StakerViewVisibility.t.sol`
* Comparison Contract: `src/StakerVisibilityCompare.sol`
* Detailed Report: `ViewVisibilityOptimizationReport.md`
* Quick Guide: `QUICK_START_GUIDE.md`
* Solidity Documentation on Function Visibility: <https://docs.soliditylang.org/en/v0.8.20/contracts.html#function-visibility>
* Original Contract: `staker.sol` (lines 33 and 40)

## Proof of Concept

Link to PoC: <https://gist.github.com/caslodev/da69ef9b273b3846460741f5f08121d2>

***

Note: Most bug bounty programs categorize this as "Gas Optimization" rather than a security vulnerability. Check the program's scope to see if gas optimizations are eligible for rewards.
