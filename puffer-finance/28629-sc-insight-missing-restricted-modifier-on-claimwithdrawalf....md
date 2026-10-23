# 28629 - \[SC - Insight] Missing restricted modifier on claimWithdrawalF...

Submitted on Feb 22nd 2024 at 18:38:33 UTC by @imaybeghost for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28629

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Throughout the PufferVault.sol wherever there is a function with natspec specifying Restricted access, is implemented with restricted modifier. Unfortunately, `claimWithdrawalFromEigenLayer(IEigenLayer.QueuedWithdrawal,IERC20[], uint256 )` is supposed to be `Restricted access` but no restricted modifier is specified

## Vulnerability Details

```
    /**
     * @notice Claims stETH withdrawals from EigenLayer
     * Restricted access
     * @param queuedWithdrawal The queued withdrawal details
     * @param tokens The tokens to be withdrawn
     * @param middlewareTimesIndex The index of middleware times
     */
    function claimWithdrawalFromEigenLayer(
        IEigenLayer.QueuedWithdrawal calldata queuedWithdrawal,
        IERC20[] calldata tokens,
        uint256 middlewareTimesIndex
    ) external virtual {...}

```

As You can check in natspec, the function is supposed to be restricted but no restricted modifier is implemented to prevent unexpected execution.

Conversely, throughout the contract all the functions whose natspec specifies the function as `Restricted access` is implemented with `restricted` modifier E.g:

```
   /**
     * @notice Deposits stETH into `stETH EigenLayer strategy`
     * Restricted access
     * @param amount the amount of stETH to deposit
     */
    function depositToEigenLayer(uint256 amount) external virtual restricted {..}
```

```
 /**
     * @notice Initiates stETH withdrawals from EigenLayer
     * Restricted access
     * @param sharesToWithdraw An amount of EigenLayer shares that we want to queue
     */
    function initiateStETHWithdrawalFromEigenLayer(uint256 sharesToWithdraw) external virtual restricted {...}
```

similarly for, `initiateETHWithdrawalsFromLido(uint256[] calldata amounts)`

## Impact Details

This leads to the unauthorized access to `claimWithdrawalFromEigenLayer()` when in fact the protocol assumes it to be implemented with access controls in place

## References

<https://github.com/PufferFinance/pufETH/blob/main/src/PufferVault.sol#L215-L226>

## Proof of Concept

Since the straightforwardness of the issue, i dont think POC is neccessary, I think adding POC for this issue would be Vacous and an opportunity cost for both of us
