# #45731 \[SC-Insight] Off-by-One Logic in Escrow End Timestamp Calculation May Cause Unintended Escrow Delay

**Submitted on May 19th 2025 at 19:53:04 UTC by @MRXSNOWDEN for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45731
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>
* **Impacts:**

## Description

## Brief/Intro

The \_getNextEscrowEndTimestamp function in CoreVaultManager uses a <= comparison to determine if the next escrow end timestamp should be moved to the following day, while the comment specifies "less than 12 hours." This off-by-one logic discrepancy could result in escrows being delayed by an extra day when the difference is exactly 12 hours, potentially causing operational inefficiencies or user confusion.

## Vulnerability Details

The relevant code in CoreVaultManager.sol is :

```solidity
if (escrowEndTimestamp <= block.timestamp + 12 hours) { // less than 12 hours from now, move to the next day
    escrowEndTimestamp += 1 days;
}
```

The comment states "less than 12 hours," but the code uses <=, which means that if the escrow end timestamp is exactly 12 hours from the current block timestamp, it will still be pushed to the next day. The correct logic to match the comment should use < instead of <=. This subtle difference can cause escrows to be delayed by a full day in edge cases, which may not be the intended behavior

## Impact Details

While this issue does not directly result in loss or theft of funds, it can cause escrows to be unnecessarily delayed by up to 24 hours in certain edge cases. This could lead to operational inefficiencies, user frustration, or missed deadlines in time-sensitive applications. The impact is best categorized as a Code Optimizations and Enhancements Insight, as it affects the precision and predictability of the contract's time-based logic

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CoreVaultManager.sol#L902-L903>

## Proof of Concept

## Proof of Concept

1- Suppose the current block.timestamp is T

2- The calculated escrowEndTimestamp is exactly T + 12 hours

3 - Since escrowEndTimestamp == block.timestamp + 12 hours, the condition is true, and the timestamp is incremented by 1 day

4 - As a result, the escrow end timestamp is now T + 1 day + 12 hours, which is a full day later than what the comment and likely the intended logic suggest

Change the condition to :

```solidity
if (escrowEndTimestamp < block.timestamp + 12 hours) { ... }
```
