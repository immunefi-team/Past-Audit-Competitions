# 28625 - \[SC - Insight] Gas griefing is possible on external call

Submitted on Feb 22nd 2024 at 17:24:08 UTC by @djxploit for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28625

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

If an external calls returns large data, then storing it becomes costly. This will result in paying huge amount of gas for copying the returned data to memory, by the MULTISIGs

## Vulnerability Details

In the function `executeTransaction` of `Timelock.sol` contract, when a transaction is executed through `_executeTransaction` function, it returns the status of the low-level call and any returned data.

```
(success, returnData) = _executeTransaction(target, callData);
```

The data returned by the `target` address, i.e `returnData` is copied to memory. Now if the size of the `returnData` is very large, then the `msg.sender` in our case the MULTISIGs, will have to pay a huge amount amount of gas for copying the returned data to memory

## Impact Details

`target` addresses (malicious actors) can launch a gas griefing attack on MULTISIGs, by returning a huge size of data.

It is recommended to implement the low level call using assembly.

## References

<https://solodit.xyz/issues/m-03-gas-griefingtheft-is-possible-on-unsafe-external-call-pashov-none-zerem-markdown>

## Proof of Concept

In `executeTransaction` function of `Timelock.sol` contract

```
(success, returnData) = _executeTransaction(target, callData);
```

```
    function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
        return target.call(callData);
    }

```
