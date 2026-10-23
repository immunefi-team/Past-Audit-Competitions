# 28792 - \[SC - Low] Return value of low level isnt checked executio...

Submitted on Feb 27th 2024 at 14:20:17 UTC by @Kenzo for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28792

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Return value of low level isn't checked, execution will resume even if the called contract throws an exception.

## Vulnerability Details

The function `executeTransaction` in the `TimeLock` contract is used to Executes a transaction after the delay period for Operations Multisig and Community multisig which can execute transactions without any delay. The problem occuring in the low level call made to `target` in `_executeTransaction` function. With a low level call we must verify the return value whether is false or true. Otherwise a false entry of transaction may register.

```js
function executeTransaction(address target, bytes calldata callData, uint256 operationId)
        external
        returns (bool success, bytes memory returnData)
    {
        // Community Multisig can do things without any delay
        if (msg.sender == COMMUNITY_MULTISIG) {
@>            return _executeTransaction(target, callData);
        }

        // Operations multisig needs to queue it and then execute after a delay
        if (msg.sender != OPERATIONS_MULTISIG) {
            revert Unauthorized();
        }

        bytes32 txHash = keccak256(abi.encode(target, callData, operationId));
        uint256 lockedUntil = queue[txHash];

        // slither-disable-next-line incorrect-equality
        if (lockedUntil == 0) {
            revert InvalidTransaction(txHash);
        }

        if (block.timestamp < lockedUntil) {
            revert Locked(txHash, lockedUntil);
        }

        queue[txHash] = 0;
@>  (success, returnData) = _executeTransaction(target, callData);

        emit TransactionExecuted(txHash, target, callData, operationId);

        return (success, returnData);
    }

```

```js
    function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
@>        return target.call(callData);
    }
```

## Impact Details

Severity: Low

Similar Incident: <https://www.kingoftheether.com/postmortem.html?source=post\\_page-----fe794a7cdb6f-------------------------------->

## Recommendation:

Check the bool returned by the `.call` function call and revert if it is false.

## Proof of Concept

Execution will resume even if the called contract throws an exception. If the call fails accidentally or an attacker forces the call to fail, this may cause unexpected behavior in the subsequent program logic.

Affected Code:

```js
    function _executeTransaction(address target, bytes calldata callData) internal returns (bool, bytes memory) {
        // slither-disable-next-line arbitrary-send-eth
@>        return target.call(callData);
    }
```
