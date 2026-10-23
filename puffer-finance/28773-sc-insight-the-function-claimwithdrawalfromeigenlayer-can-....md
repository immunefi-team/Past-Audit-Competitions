# 28773 - \[SC - Insight] The function claimWithdrawalFromEigenLayer can ...

Submitted on Feb 26th 2024 at 18:37:32 UTC by @crazy\_squirrel for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28773

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Theft of unclaimed yield
* Protocol insolvency

## Description

## Brief/Intro

The `claimWithdrawalFromEigenLayer` function in the `PufferVault` is marked as ***restricted*** in the NatSpec comment.

However, it doesn't have the appropriate `restricted` modifier, and can be called by anyone instead.

## Vulnerability Details

#### `claimWithdrawalFromEigenLayer`

```solidity
function claimWithdrawalFromEigenLayer(
    IEigenLayer.QueuedWithdrawal calldata queuedWithdrawal,
    IERC20[] calldata tokens,
    uint256 middlewareTimesIndex
) external virtual;
```

Completes the process of withdrawing stETH from EigenLayer's stETH strategy contract

*Effects*

* Claims the previously queued withdrawal from EigenLayer's stETH strategy contract
* Transfers stETH from EigenLayer's stETH strategy contract to this vault contract

*Requirements*

* There must be a corresponding queued withdrawal created previously via function `initiateStETHWithdrawalFromEigenLayer`
* Enough time must have elapsed since creation of the queued withdrawal such that it is claimable at the time of this function call

## Impact Details

Provide a detailed breakdown of possible losses from an exploit, especially if there are funds at risk. This illustrates the severity of the vulnerability, but it also provides the best possible case for you to be paid the correct amount. Make sure the selected impact is within the program’s list of in-scope impacts and matches the impact you selected.

## References

<https://github.com/PufferFinance/pufETH/blob/d340d40a2ebb72993cd7dd6049a78a01bcef32ae/src/PufferVault.sol#L217>

## Proof of Concept
