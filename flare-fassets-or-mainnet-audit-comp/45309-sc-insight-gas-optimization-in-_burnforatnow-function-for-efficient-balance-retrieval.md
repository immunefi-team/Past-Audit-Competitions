# #45309 \[SC-Insight] Gas Optimization in \`\_burnForAtNow\` Function for efficient balance retrieval

**Submitted on May 12th 2025 at 13:18:46 UTC by @Bluedragon for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45309
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-labs-ltd/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**

## Description

## Summary:

The `_burnForAtNow` function in the `CheckPointable` contract uses the `balanceOfAt()` function to retrieve the balance of an address at the current block. This approach is inefficient as `balanceOfAt()` internally calls `valueAt()`, which performs a binary search, leading to higher gas consumption. A more efficient alternative is to use a new function `balanceOfNow()` that leverages `valueAtNow()` to directly retrieve the latest checkpoint value.

## Vulnerability Details:

* **Root Cause:** The use of `balanceOfAt()` for retrieving the current balance introduces unnecessary computational complexity due to the binary search performed by `valueAt()`.
* **Affected Function:** `_burnForAtNow`
* **File:** `contracts/fassetToken/implementation/CheckPointable.sol`
* **Technical Details:** The `valueAt()` function performs a binary search to find the value at a specific block, which is unnecessary when retrieving the balance for the current block. The `valueAtNow()` function, which directly retrieves the last checkpoint value, can be used instead.

## Impact:

* Increased gas costs for the `_burnForAtNow` function.
* Reduced efficiency of the contract, especially in scenarios with frequent token burns.

## Recommended Mitigation:

Implement a new function `balanceOfNow()` in the `CheckPointable` contract:

```solidity
function balanceOfNow(address _owner) public view returns (uint256 _balance) {
    return balanceHistory.valueAtNow(_owner);
}
```

## Proof of Concept

## Proof of Concept:

1. Call the `_burnForAtNow` function with a valid `_owner` and `_amount`.
2. Observe the gas usage due to the invocation of `balanceOfAt()` and its reliance on `valueAt()`.
