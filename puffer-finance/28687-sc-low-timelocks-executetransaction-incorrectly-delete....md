# 28687 - \[SC - Low] Timelocks executeTransaction incorrectly delete...

Submitted on Feb 23rd 2024 at 23:58:18 UTC by @honeymewn for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28687

Report type: Smart Contract

Report severity: Low

Target: <https://etherscan.io/address/0x3C28B7c7Ba1A1f55c9Ce66b263B33B204f2126eA#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Timelock's executeTransaction deletes transaction after executing it. However, if it fails it should keep it otherwise operations multisig will have to wait for a week to reexecute it.

## Vulnerability Details

executeTransaction in Timelock.sol executes a transaction from the queue. However there's no revert on failure. Therefore if proposed transaction fails due to insufficient gas or PufferVault temporarily not having enough funds for a deposit community will have to wait for another delay\_period to reexecute it.

## Impact Details

Operations will have to wait for a week or ask community to execute a transaction if it's urgent.

## References

<https://github.com/PufferFinance/pufETH/blob/main/src/Timelock.sol#L217>

## Recommendation

I suggested slightly modifying the function to keep the txn on failure. One can always call `cancelTransaction` if needed.

```solidity
(success, returnData) = _executeTransaction(target, callData);
if (success) {
    queue[txHash] = 0;
}
```

Optionally add a reentrancy guard but I don't really think this is needed here.

## Proof of Concept

As I said earlier a transaction should be kept when it fails. Otherwise operations will need to wait for another week to reexecute. PoC demonstrates a case when operations multisig runs out of gas when pausing targets (there could be other cases of when a transaction fails temporarily). Run with:

```sh
forge test --match-path ./test/unit/Timelock.t.sol --match-test "test_checkExecuteTransaction.*"
```

```solidity
// Timelock.t.sol

contract CustomDeployer is DeployPuffETH {
    // we need to extract a private key to populate accessManager
    function getDeployerKey() public view returns(uint256) {
        return _deployerPrivateKey;
    }
}

contract TimelockTest is Test {
// ...

    CustomDeployer public deployPuff;
    function setUp() public {
        deployPuff = new CustomDeployer();
        // ...
    }

    function setPausers(uint gasLimit) public returns(bytes32 txHash, address[] memory targets) {
        vm.startBroadcast(deployPuff.getDeployerKey());

        bytes4[] memory selectors = new bytes4[](1);
        selectors[0] = UUPSUpgradeable.upgradeToAndCall.selector;

        targets = new address[](20);
        for (uint i = 0; i < targets.length; i++) {
            targets[i] = address(uint160(i + 1));
            accessManager.setTargetFunctionRole(address(targets[i]), selectors, 1);
            assertTrue(!accessManager.isTargetClosed(targets[i]), "target should be open");
        }
        vm.stopBroadcast();

        bytes memory callData = abi.encodeCall(Timelock.pause, (targets));

        vm.startPrank(timelock.OPERATIONS_MULTISIG());
        uint256 operationId = 1234;

        txHash = timelock.queueTransaction(address(timelock), callData, operationId);

        uint256 lockedUntil = block.timestamp + timelock.delay();

        assertTrue(timelock.queue(txHash) != 0, "should be queued");

        vm.warp(lockedUntil + 1);
        timelock.executeTransaction{gas: gasLimit}(address(timelock), callData, operationId);
    }

    function test_checkExecuteTransactionFail() public {
        // not enough gas for a multicall
        (bytes32 txHash, address[] memory targets) = setPausers(500_000);
        // not enough gas for transaction to go through therefore multicall fails
        for (uint i = 0; i < targets.length; i++) {
            assertTrue(!accessManager.isTargetClosed(targets[i]), "target should be open");
        }
        // we shouldn't actually remove it
        assertTrue(timelock.queue(txHash) != 0, "should be queued"); // <---- fails
    }

    function test_checkExecuteTransactionOk() public {
        (bytes32 txHash, address[] memory targets) = setPausers(1_000_000);
        // transaction successfully went through and closed targets
        for (uint i = 0; i < targets.length; i++) {
            assertTrue(accessManager.isTargetClosed(targets[i]), "target should be open");
        }
        assertTrue(timelock.queue(txHash) == 0, "should be removed from queue");
    }
}

```
