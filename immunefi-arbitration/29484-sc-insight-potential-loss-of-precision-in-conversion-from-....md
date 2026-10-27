# 29484 - \[SC - Insight] Potential Loss of Precision in Conversion from ...

Submitted on Mar 20th 2024 at 23:27:54 UTC by @Flare for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29484

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/WithdrawalSystem.sol>

Impacts:

* Precision

## Description

## Brief/Intro

The `setUp` function in the `WithdrawalSystem` contract involves converting a `uint256` value to a `uint32` for setting the cooldown period for withdrawals. This conversion may lead to a loss of precision and unexpected behavior if the original value exceeds the range representable by a `uint32`

## Vulnerability Details

<https://github.com/immunefi-team/vaults/blob/main/src/WithdrawalSystem.sol?utm\\_source=immunefi#L28C5-L44C6>

The `setUp` function in the `withdrawlSystem` contract accepts parameters, including `_txCooldown`, which represents the cooldown period for withdrawals. Within the function, the `_txCooldown` value is cast from uint256 to uint32 before being passed to the `_setTxCooldown` function. This type conversion can potentially lead to issues due to loss of precision if the original value is larger than what can be represented by a uint32

## Impact Details

the impact of the potential loss of precision in the conversion from `uint256` to `uint32`

## References

change input var \_txCooldown as uint32 in `setUp` function

## Proof of Concept
