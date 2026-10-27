# #41766 \[SC-Insight] In \`Yeet.sol\`, storage slots only set in constructor should be declared \`immutable\`.

**Submitted on Mar 18th 2025 at 08:10:37 UTC by @Victor\_TheOracle for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41766
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**

## Description

## Brief/Intro

In `Yeet.sol`, several state variables that are only set in the constructor are not declared as `immutable`. This oversight leads to unnecessary storage usage and higher gas costs during contract interactions.

## Vulnerability Details

The Solidity compiler offers the `immutable` keyword for variables that are assigned only once during construction. By marking these variables as `immutable`, their values are embedded directly in the bytecode rather than stored in a storage slot, leading to significant gas optimizations. In `Yeet.sol`, the following variables are set exclusively in the constructor but are not marked as immutable:

```solidity
/// @notice yeetTokenAddress is the address of the YeetToken contract
//@audit (info) -----> Should be immutable
address public yeetTokenAddress;
/// @notice rewardsContract is the Reward contract
//@audit (info) -----> Should be immutable
Reward public rewardsContract;
/// @notice gameSettings is the YeetGameSettings contract
//@audit (info) -----> Should be immutable
YeetGameSettings public gameSettings;
/// @notice yeetback is the Yeetback contract
//@audit (info) -----> Should be immutable
Yeetback public yeetback;
```

## Impact Details

The primary impact of this issue is increased gas consumption during contract execution.

## References

Links to relevant lines:

1. <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L111>
2. <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L113>
3. <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L117>
4. <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L121>

## Proof of Concept

## Proof of Concept
