# 29318 - \[SC - Insight] Timelock contract should use canExecuteTransact...

Submitted on Mar 13th 2024 at 22:35:50 UTC by @shanb1605 for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29318

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/Timelock.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Timelock contract's `executeTransaction()` has multiple checks before executing a transaction:

```solidity
        require(txData.state == TxState.Queued, "Timelock: transaction is not queued");
        require(
            txData.queueTimestamp + txData.cooldown <= block.timestamp,
            "Timelock: transaction is not yet executable"
        );
        require(
            txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,
            "Timelock: transaction is expired"
        );
        
         require(!vaultFreezer.isFrozen(vault), "Timelock: vault is frozen");
```

Instead of these multiple checks immunefi should consider using `canExecuteTransaction()`

## Vulnerability Details

Using multiple `require` statements may cost too much gas to execute transaction.

## Impact Details

Immunefi promises to offer a smooth arbitration between the project and Whitehat, using multiple `require` statements will cost too much gas to execute.

## Recommendation

Immunefi can add this snippet if they wish to handle this as a single require statement:

```solidity
require(canExecuteTransaction(txHash) == true);
```

## Proof of Concept

I have commented out unnecessary require statements on `executeTransaction()` and added a single line of code as per recommendation.

Also, I have changed the function visibility of `canExecuteTransaction()`from `external` to `public`

```solidity
    function executeTransaction(bytes32 txHash) external {
        TxStorageData memory txData = txHashData[txHash];
        //Fix
        require(canExecuteTransaction(txHash) == true, "!F");

        /*
        require(txData.state == TxState.Queued, "Timelock: transaction is not queued");
        require(
            txData.queueTimestamp + txData.cooldown <= block.timestamp,
            "Timelock: transaction is not yet executable"
        );
        require(
            txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp,
            "Timelock: transaction is expired"
        );*/

        (address to, uint256 value, bytes memory data, Enum.Operation operation, address vault) = abi.decode(
            txData.execData,
            (address, uint256, bytes, Enum.Operation, address)
        );

        require(msg.sender == vault, "Timelock: only vault can execute transaction");
        //require(!vaultFreezer.isFrozen(vault), "Timelock: vault is frozen");

        txHashData[txHash].state = TxState.Executed;

        emit TransactionExecuted(txHash, to, vault, value, data, operation);
        immunefiModule.execute(vault, to, value, data, operation);
    }

    function canExecuteTransaction(bytes32 txHash) public view returns (bool) {
        TxStorageData memory txData = txHashData[txHash];
        (, , , , address vault) = abi.decode(txData.execData, (address, uint256, bytes, Enum.Operation, address));
        return
            !vaultFreezer.isFrozen(vault) &&
            txData.state == TxState.Queued &&
            txData.queueTimestamp + txData.cooldown <= block.timestamp &&
            (txData.expiration == 0 || txData.queueTimestamp + txData.cooldown + txData.expiration > block.timestamp);
    }
```

***Run forge test --mp test/foundry/Timelock.t.sol and everything works fine :)***
