# 31497 - \[SC - Low] executeBatch lacks payable so ethers can not be...

Submitted on May 20th 2024 at 16:11:09 UTC by @OxRizwan for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31497

Report type: Smart Contract

Report severity: Low

Target: <https://immunefi.com>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Logic errors

## Description

## Brief/Intro

`executeBatch()` lacks `payable` so ethers can not be a part of bath execution

## Vulnerability Details

In `TimelockExecutor.sol` contract, `executeBatch()` is used to execute the ready operations containing the batch transactions.

```solidity
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,     @audit // ether value sent along with call
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 descriptionHash,
        uint256 chainId
    ) public virtual onlyRole(EXECUTOR_ROLE) {
        require(targets.length == values.length, "TimelockExecutor: length mismatch");
        require(targets.length == payloads.length, "TimelockExecutor: length mismatch");

        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, descriptionHash, chainId);

        _beforeCall(id, predecessor);
        for (uint256 i = 0; i < targets.length; ++i) {
            _execute(id, i, targets[i], values[i], payloads[i]);
        }
        _afterCall(id);
    }
```

This function takes `values` as a param and part of batch execution. The values are the ethers which are sent along with call. `executeBatch()` calls internal function `execute()` for the transactions execution which is implemented as below:

```solidity
    function _execute(bytes32 id, uint256 index, address target, uint256 value, bytes calldata data) private {
        string memory errorMessage = "Governor: call reverted without message";
@>      (bool success, bytes memory returndata) = target.call{ value: value }(data);
        Address.verifyCallResult(success, returndata, errorMessage);
        require(success, "TimelockExecutor: underlying transaction reverted");

        emit CallExecuted(id, index, target, value, data);
    }
```

It can be seen at (@), the ether value is indeed a part of `executeBatch()` function as the value is not hardcoded to 0.

Now, the issue is that, `executeBatch()` will revert when msg.value > 0. The current implementation of the `executeBatch()` function within the smart contract lacks the payable keyword. This omission leads to a critical issue where any transaction that attempts to send ether (ETH) to this function will fail. Since the function is designed to allow the executor to execute arbitrary calls and potentially send ETH, the inability to accept ETH due to the missing payable specifier means that:

1. The contract does not behave as intended when interacting with functions or operations requiring ETH transfers.
2. Any attempt to send ETH to this function will revert and result in a failure of the intended operation.
3. ETH sent to this non-payable function will be stuck and effectively lost, leading to financial losses for the users.

## Impact Details

`executeBatch()` will fail when value is greater than 0 so `executeBatch()` can not be succesfully executed due to this issue

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/TimelockExecutor.sol#L275-L293>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/TimelockExecutor.sol#L316>

## Recommendation to fix

Add `payable` to `executeBatch()` function.

Consider below changes:

```diff
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 descriptionHash,
        uint256 chainId
-    ) public virtual onlyRole(EXECUTOR_ROLE) {
+    ) public payable virtual onlyRole(EXECUTOR_ROLE) {
        require(targets.length == values.length, "TimelockExecutor: length mismatch");
        require(targets.length == payloads.length, "TimelockExecutor: length mismatch");

        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, descriptionHash, chainId);

        _beforeCall(id, predecessor);
        for (uint256 i = 0; i < targets.length; ++i) {
            _execute(id, i, targets[i], values[i], payloads[i]);
        }
        _afterCall(id);
    }
```

## Proof of Concept

The issue can be easily understood with above description and recommendation to fix.

Additionally, the affected contract i.e `TimelockExecutor.sol` is actually referred from openzeppelin's `TimelockController.sol` where this issue is not present. If you see the `executeBatch()` of openzeppelin's `TimelockController.sol` then this function has `payable` keyword so ethers can be part of batch execution. This can be checked as below:

```solidity
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
@>  ) public payable virtual onlyRoleOrOpenRole(EXECUTOR_ROLE) {
        if (targets.length != values.length || targets.length != payloads.length) {
            revert TimelockInvalidOperationLength(targets.length, payloads.length, values.length);
        }

        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);

        _beforeCall(id, predecessor);
        for (uint256 i = 0; i < targets.length; ++i) {
            address target = targets[i];
            uint256 value = values[i];
            bytes calldata payload = payloads[i];
            _execute(target, value, payload);
            emit CallExecuted(id, i, target, value, payload);
        }
        _afterCall(id);
    }
```

Reference link: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d947fb056d6a7eb099013076ac5ea5a69e9fec06/contracts/governance/TimelockController.sol#L391>
