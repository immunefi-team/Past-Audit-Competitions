# 56832 sc low alchemistcurator contract doesn t allow to remove strategies from the myt morpho v2 vault&#x20;

**Submitted on Oct 21st 2025 at 04:25:42 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56832
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

AlchemistCurator contract is designed to allow for updating MYT vault caps and add/remove strategies to the MYT vault.

Adding strategies to the MYT vault is straightforward:

* an operator calls `submitSetStrategy`, specifying the adapter address and the vault address, which allows to submit data containing the call request to the vault and trigger the timelock period
* Operator waits for the timelock period of the vault to pass before calling `setStrategy` which effectively adds a new strategy to the vault

The problem arises because the AlchemistCurator contract doesn't allow the operator to submit data to the vault for strategy removal. This is problematic because `removeAdapter` uses a timelock just like `addAdapter`. Therefore, there is no way to remove a strategy form the MYT vault with the current implementation.

## Vulnerability Details

AlchemistCurator contract implements the following `removeStrategy` function:

```
    function removeStrategy(address adapter, address myt) external onlyOperator {
        require(adapter != address(0), "INVALID_ADDRESS");
        require(myt != address(0), "INVALID_ADDRESS");
        _setStrategy(adapter, myt, true); // remove
    }
```

This function calls the internal `_setStrategy` function:

```
    function _setStrategy(address adapter, address myt, bool remove) internal {
        // @audit why not only in setStrategy?
        adapterToMYT[adapter] = myt;

        IVaultV2 vault = _vault(adapter);
        if (remove) {
            vault.removeAdapter(adapter);
        } else {
            vault.addAdapter(adapter);
        }
        emit StrategySet(adapter, myt);
    }
```

`_setStrategy` will call `vault.removeAdapter(adapter)`, but this call will revert given that no data has previously been submitted in order to trigger the timelock.

## Impact Details

The impact of this issue can be considered as low as it doesn't affect user funds and only affects the protocol functionalities.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in AlchemistCurator.t.sol file:

```
    function testRemoveStrategy() public {
        vm.startPrank(operator);

        // submit the new strategy adapter to add to the vault
        mytCuratorProxy.submitSetStrategy(address(mytStrategy), address(vault));
        _vaultFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        // effectively add a new adapter to the vault after timelock period
        mytCuratorProxy.setStrategy(address(mytStrategy), address(vault));

        // failure on attempt to remove the previously added strategy because of timelock
        // AlchemistCurator contract should implement a `submitRemoveStrategy` function
        mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault));
        vm.stopPrank();
    }
```

This test highlights the missing logic in AlchemistCurator contract, which is not able to submit data in order to call the `removeAdapter` function of the MYT vault.

A fix for this bug would be to add a `submitRemoveStrategy` function:

```
    function submitRemoveStrategy(address adapter, address myt) external onlyOperator {
        require(adapter != address(0), "INVALID_ADDRESS");
        require(myt != address(0), "INVALID_ADDRESS");
        _submitRemoveStrategy(adapter, myt);
    }

    function _submitRemoveStrategy(address adapter, address myt) internal {
        IVaultV2 vault = IVaultV2(myt);
        bytes memory data = abi.encodeCall(IVaultV2.removeAdapter, adapter);
        vault.submit(data);
        emit SubmitRemoveStrategy(adapter, myt);
    }
```
