# 56427 sc insight src utils permissionedproxy sol setpermissionedcall incomplete event emission because it doesnt include value argument for signature

**Submitted on Oct 15th 2025 at 22:07:09 UTC by @hunter0xweb3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56427
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/PermissionedProxy.sol>
* **Impacts:**

## Description

## Brief/Intro

src/utils/PermissionedProxy.sol::setPermissionedCall emits an event when a method signature is enabled (function called with `value` argument = true) or disabled (function called with `value` argument = false) in protocol via setPermissionedCall method. However the emitted event doesnt include `value` argument value leading to incomplete emitted information about permissioned call changes for a signature AlchemistAllocator.sol contract inherits from PermissionedProxy.sol

## Vulnerability Details

The issue occurs because src/utils/PermissionedProxy.sol::setPermissionedCall doesnt emmit the `value` variable value for the function call. This leads to incomplete emitted information about if the method signature is now allowed or not:

```solidity
    function setPermissionedCall(bytes4 sig, bool value) external onlyAdmin {
        permissionedCalls[sig] = value;
@>        emit AddedPermissionedCall(sig);
    }
```

* AlchemistAllocator.sol contract inherits from PermissionedProxy.sol so AlchemistAllocator emits incomplete information too:

```solidity
import {PermissionedProxy} from "./utils/PermissionedProxy.sol";
//...
contract AlchemistAllocator is PermissionedProxy,
```

## Impact Details

* The information emitted in the setPermissionedCall's event is incomplete because doesnt include the status of the method signature, ie doesnt show if the method signature was enabled or disabled in the protocol.
* AlchemistAllocator.sol contract inherits from PermissionedProxy.sol\
  so AlchemistAllocator emits incomplete information

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/PermissionedProxy.sol#L43-L46\\>
<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L15>

## Proof of Concept

## Proof of Concept

1. Call with a privileged address (admin or an approved operator) AlchemistAllocator::setPermissionedCall with some signature and value = true, to enable signature in the protocol
2. Call again with same account AlchemistAllocator::setPermissionedCall but now with value = false, to disable method signature in the protocol
3. Observe both emitted events for AlchemistAllocator::setPermissionedCall are identical, because src/utils/PermissionedProxy.sol::setPermissionedCall doesnt contain information about value argument

**Recommendation**

Include `value` variable value in emitted event:

```
    function setPermissionedCall(bytes4 sig, bool value) external onlyAdmin {
        permissionedCalls[sig] = value;
@>      emit AddedPermissionedCall(sig, value);
    }
```
