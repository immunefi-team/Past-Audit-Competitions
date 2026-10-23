# 57251 sc low curator cannot remove adapter due to timelock requirement

**Submitted on Oct 24th 2025 at 18:17:29 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57251
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Curator contract interacts with a Morpho Vault, this VaultV2 uses a timelock mechanism for sensitive functions such as adding and removing adapters. While adding an adapter works as expected because it creates a timelock for this request beforehand ( submit `IVaultV2.addAdapter`), removing an adapter fails because setting a timelock (submit `IVaultV2.removeAdapter`) beforehand is missing.

When a curator adds a new strategy, the process involves two steps:

1. Submitting the new adapter: ( this will start the timelock for this process )

```
bytes memory data = abi.encodeCall(IVaultV2.addAdapter, adapter);
vault.submit(data); 
```

2. Executing the addAdapter ( this will execute if the previous timelock is expired)

After the timelock expires, `setStrategy` calls `vault.addAdapter(adapter)` to finalize adding the adapter.

## Vulnerability Details

However, when removing a strategy, the function `removeStrategy` calls:

```
vault.removeAdapter(adapter);
```

without first submitting the removal action for timelock approval:

```
bytes memory data = abi.encodeCall(IVaultV2.removeAdapter, adapter); // this is not in alchemix
vault.submit(data);
```

As a result, the following check in the vault fails:

```
require(executableAt[msg.data] != 0, ErrorsLib.DataNotTimelocked());
```

Because no pending timelock entry exists for that `removeAdapter` call.

The relevant code snippet:

```
    function submit(bytes calldata data) external {
        require(msg.sender == curator, ErrorsLib.Unauthorized());
        require(executableAt[data] == 0, ErrorsLib.DataAlreadyPending());

        bytes4 selector = bytes4(data);
        uint256 _timelock =
            selector == IVaultV2.decreaseTimelock.selector ? timelock[bytes4(data[4:8])] : timelock[selector];
        executableAt[data] = block.timestamp + _timelock;
        emit EventsLib.Submit(selector, data, executableAt[data]);
    }

    function removeAdapter(address account) external {
        timelocked();
        if (isAdapter[account]) {
            for (uint256 i = 0; i < adapters.length; i++) {
                if (adapters[i] == account) {
                    adapters[i] = adapters[adapters.length - 1];
                    adapters.pop();
                    break;
                }
            }
            isAdapter[account] = false;
        }
        emit EventsLib.RemoveAdapter(account);
    }
```

Since `removeAdapter` is timelocked, it must first be submitted before execution.

## Impact Details

The curator is unable to remove adapters because the required timelock submission step is missing. As a result, `removeStrategy` reverts on the vault’s timelock check, now the adapter cannot be removed.

## Remediaton

Add a function to be able to queue adapter removal via the vault’s timelock using: (`IVaultV2.removeAdapter`)

```
vault.submit(abi.encodeCall(IVaultV2.removeAdapter, adapter)).
```

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

1. Curator calls `submitSetStrategy` to add a adapter
2. Curator calls `setStrategy` after the previous step's timelock is expired and it will add the adapter
3. Curator calls `removeStrategy` to remove the adapter, however it fails because submitting the (`IVaultV2.removeAdapter`) data is not set for this data

Please add the following test to the existing `AlchemistCuratorTest` suite.

```
    function testSetStrategyCannotRemove() public {
        vm.startPrank(operator);
        mytCuratorProxy.submitSetStrategy(address(mytStrategy), address(vault));
        _vaultFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        mytCuratorProxy.setStrategy(address(mytStrategy), address(vault));

        vm.expectRevert(abi.encode("DataNotTimelocked()"));
        mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault));
        vm.stopPrank();
    }
```
