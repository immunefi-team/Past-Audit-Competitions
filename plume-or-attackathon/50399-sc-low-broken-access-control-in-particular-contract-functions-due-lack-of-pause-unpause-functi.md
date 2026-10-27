# 50399 sc low broken access control in particular contract functions due lack of pause unpause functionality

Submitted on Jul 24th 2025 at 08:43:02 UTC by @KKam86 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50399
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Contract `TellerWithMultiAssetSupportPredicateProxy` incorrectly integrates OpenZeppelin `Pausable`. As a result the contract does not expose pause/unpause functionality that is responsible for stopping particular contract functions in emergency states. This can lead to unwanted situations like risky user deposits during an emergency.

### Vulnerability Details

`TellerWithMultiAssetSupportPredicateProxy` imports and inherits OpenZeppelin `Pausable`:

```solidity
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

contract TellerWithMultiAssetSupportPredicateProxy is
    Ownable,
    ReentrancyGuard,
    PredicateClient,
    Pausable
```

However, `Pausable` only provides internal functions `_pause()` and `_unpause()`:

```solidity
function _pause() internal virtual whenNotPaused {
    _paused = true;
    emit Paused(_msgSender());
}

function _unpause() internal virtual whenPaused {
    _paused = false;
    emit Unpaused(_msgSender());
}
```

No public/external functions are exposed in the parent or this contract to call `_pause()` / `_unpause()`. The contract contains checks like:

```solidity
if (paused()) {
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}
```

but with no way to change `_paused` from its initial `false` value, these checks will never trigger. `Pausable` initializes `_paused` to `false`:

```solidity
constructor() {
    _paused = false;
}

function paused() public view virtual returns (bool) {
    return _paused;
}
```

Thus owner cannot activate the pause state and cannot stop functions that are intended to be blocked during emergencies.

## Impact Details

{% stepper %}
{% step %}

### Deposits during emergencies

Depositing funds during emergency situations can be risky, especially when the bug is discovered and triggered by bad actors. Users can potentially lose funds. Deposit functions should clearly be not accessible during contract pause:

```solidity
* @notice Allows users to deposit into the BoringVault, if the teller contract is not paused
```

{% endstep %}

{% step %}

### Admins can't restrict access during fixes

Fixing security flaws without preventing user calls to particular contract functions is dangerous. Admins do not have the pause/unpause controls required to restrict access to these functions while remediations are applied.
{% endstep %}
{% endstepper %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L20>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L78-L80>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L133-L135>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol?utm\\_source=immunefi#L57>
* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/acd4ff74de833399287ed6b31b4debf6b2b35527/contracts/utils/Pausable.sol#L43-L45>
* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/acd4ff74de833399287ed6b31b4debf6b2b35527/contracts/utils/Pausable.sol#L74-L76>
* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/acd4ff74de833399287ed6b31b4debf6b2b35527/contracts/utils/Pausable.sol#L103-L106>
* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/acd4ff74de833399287ed6b31b4debf6b2b35527/contracts/utils/Pausable.sol#L115-L118>

## Proof of Concept

<details>

<summary>Forge inspect methods output (shows no pause/unpause functions exposed)</summary>

Use the following command in the project root (`attackathon-plume-network-nucleus-boring-vault`):

forge inspect src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol methods --json

Output:

```
{
  "deposit(address,uint256,uint256,address,address,(string,uint256,address[],bytes[]))": "0edb4e20",
  "depositAndBridge(address,uint256,uint256,(uint32,address,address,uint64,bytes),address,(string,uint256,address[],bytes[]))": "787d5152",
  "genericUserCheckPredicate(address,(string,uint256,address[],bytes[]))": "88b472ce",
  "getPolicy()": "ce1e4626",
  "getPredicateManager()": "a4b3bc01",
  "owner()": "8da5cb5b",
  "paused()": "5c975abb",
  "renounceOwnership()": "715018a6",
  "setPolicy(string)": "6b4c991b",
  "setPredicateManager(address)": "e0a7704a",
  "transferOwnership(address)": "f2fde38b"
}
```

This output lists external/public functions of `TellerWithMultiAssetSupportPredicateProxy`. No pause()/unpause() or similar admin functions are present, so the owner cannot pause the contract.

</details>
