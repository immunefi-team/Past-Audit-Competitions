# 58259 sc low broken operator logic inside alchemistcurator

**Submitted on Oct 31st 2025 at 19:33:02 UTC by @Cyborg for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58259
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Broken AlchemistCurator's operator Vault managing

## Description

## Brief/Intro

Missing logic inside AlchemistCurator leading to broken operator management of Vault's adapters.

## Vulnerability Details

AlchemistCurator's operator is supposed to manage the adapters *( strategies )* of the Morpho Vault - as we can see methods `setStrategy` and `removeStrategy` are protected with the modifier `onlyOperator` and these methods serve for adding or removing adapters from the Vault.

However the nature of Morpho's Vault require these actions to be processed as timelocked action, meaning the right approach is first to initiate a timelock action and after the waiting period has passed only then the changes can be applied to the vault wether it's adding or removing of an adapter *( source - <https://docs.morpho.org/curate/concepts/roles#capabilities-1> )*.

Inside AlchemistCurator *(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>)* we can see the operator has the ability to start the `IVaultV2.addAdapter` auction inside method `_submitSetStrategy`, but there isn't the same method for removing a strategy. The operator can finalize a removing adapter auction through the existing method `removeStrategy`, but this method is worthless as the operator cannot initiate the auction for removing an adapter as there is missing method that requests `vault.submit()` with parameter data `abi.encodeCall(IVaultV2.removeAdapter, adapter)`.

## Impact Details

AlchemistCurator's operator not being able to properly manage adapters for the Vault - can only set adapters, but cannot remove them.

## Recommendation

Consider introducing 2 new methods to AlchemistCurator.sol which are `submitRemoveStrategy` and `_submitRemoveStrategy`, just the same way as the already existing `submitSetStrategy` and `_submitSetStrategy`. This will allow the operator to be able to also to remove adapters from the Vault:

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

From here after the waiting period for auction has passed now the operator can successfully request the method `removeStrategy`.

## Proof of Concept

## Proof of Concept

Create test PoC file `src/test/AlchemistCurator.ImpossibleOperatorRemoveStrategy.t.sol` and run with command `forge test src/test/AlchemistCurator.ImpossibleOperatorRemoveStrategy.t.sol -vv`:

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {ErrorsLib} from "../../lib/vault-v2/src/libraries/ErrorsLib.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MockAlchemistCurator} from "./mocks/MockAlchemistCurator.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";

contract AlchemistCuratorTest is Test {
    using MYTTestHelper for *;

    MockAlchemistCurator public mytCuratorProxy;

    VaultV2 public vault;
    address public operator = address(0x2222222222222222222222222222222222222222); // default operator
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    MockMYTStrategy public mytStrategy;

    function setUp() public {
        vm.startPrank(admin);
        mytCuratorProxy = new MockAlchemistCurator(admin, operator);
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, address(mytCuratorProxy));
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        vm.stopPrank();
    }

    function test_impossible_operator_remove_strategy() public {
        vm.startPrank(operator);

        mytCuratorProxy.submitSetStrategy(address(mytStrategy), address(vault));
        _vaultFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));

        require(mytCuratorProxy.adapterToMYT(address(mytStrategy)) == address(0));

        mytCuratorProxy.setStrategy(address(mytStrategy), address(vault));

        /// validating that the strategy has been set as an adapter to the Morpho Vault successfull
        require(mytCuratorProxy.adapterToMYT(address(mytStrategy)) != address(0), "ERROR: setting adapter failed");
        require(vault.isAdapter(address(mytStrategy)) == true, "ERROR: setting adapter failed");

        /// The operator now has no option of starting the removing process of a strategy
        /// Removing an adapter from a Morpho Vault is timelocked action the same way as adding an adapter
        /// As shown above the operator has an option to initiate setting of an adapter through method submitSetStrategy and finalize the adapter setting through method setStrategy
        /// But this is not the same for removing - there is no way to initiate the procedure of removing an adapter and without it there is no way for the operator request method removeStrategy

        vm.expectRevert(ErrorsLib.DataNotTimelocked.selector); /// DataNotTimelocked() reverts, because the operator has no way to initiate vault.submit(abi.encodeCall(IVaultV2.removeAdapter, adapter))
        mytCuratorProxy.removeStrategy(address(mytStrategy), address(vault));

        /// the strategy adapter has not been successfully removed from the vault
        require(vault.isAdapter(address(mytStrategy)) == true);
    }

    function _vaultFastForward(bytes memory data) internal {
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + vault.timelock(selector));  /// each selector has it's own timelock
    }
}
```

This PoC shows that after successfully setting an adapter to the Vault, the AlchemistCurator's operator is stuck with removing it.
