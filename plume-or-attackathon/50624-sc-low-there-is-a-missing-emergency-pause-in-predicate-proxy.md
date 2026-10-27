# 50624 sc low there is a missing emergency pause in predicate proxy&#x20;

**Submitted on Jul 26th 2025 at 19:55:31 UTC by @XDZIBECX for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50624
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from OpenZeppelin's `Pausable` contract but is missing public `pause()` and `unpause()` functions. Because of this, administrators cannot pause the contract during emergency situations (e.g., security incidents or critical bugs). This can result in temporary freezing of user funds for at least 24 hours, as users could continue depositing through the predicate proxy even when the underlying system should be paused, leading to stuck transactions and inability to halt operations during emergencies.

## Vulnerability Details

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from `Pausable` but does not implement the necessary public functions to control the pause state:

```solidity
contract TellerWithMultiAssetSupportPredicateProxy is Ownable, ReentrancyGuard, PredicateClient, Pausable {
    // there is No public pause() function
    // there is No public unpause() function
```

OpenZeppelin's `Pausable` provides internal `_pause()` and `_unpause()` functions, but these are only accessible if the inheriting contract implements public wrapper functions. Without these functions, the contract is effectively unpausable because:

* The internal `_pause()` and `_unpause()` functions cannot be called externally
* The `paused()` state can never be changed from its initial `false` value
* Manual pause checks in functions will always pass since `paused()` returns `false`

The contract does include manual pause checks in its functions:

```solidity
if (paused()) {
    revert TellerWithMultiAssetSupportPredicateProxy__Paused();
}
```

These checks are ineffective because the pause state can never be set to `true` without the missing public functions. Suggested additions:

```solidity
function pause() external onlyOwner { _pause(); }
function unpause() external onlyOwner { _unpause(); }
```

Also consider using the `whenNotPaused` / `whenPaused` modifiers on user-entry functions instead of manual `if (paused())` checks.

## Impact Details

The bug can cause a temporary freezing of funds for at least 24 hours. Example scenario:

{% stepper %}
{% step %}

### During an emergency: administrators attempt to pause the system

The predicate proxy cannot be paused due to missing pause functions.
{% endstep %}

{% step %}

### Users continue depositing

The predicate proxy remains operational even when the underlying system should be paused.
{% endstep %}

{% step %}

### Funds become temporarily frozen

Deposits may fail or get stuck in the predicate proxy, but users can still initiate transactions that do not complete as expected.
{% endstep %}

{% step %}

### Recovery time exceeds 24 hours

The team may need to:

* Deploy a new predicate proxy with proper pause functionality
* Migrate users to the new contract
* Handle stuck transactions and refunds
* Update all dependent contracts (like `DexAggregatorWrapperWithPredicateProxy`)
  {% endstep %}
  {% endstepper %}

Financial impact:

* User deposits could be stuck in the predicate proxy during emergencies
* Cross-chain bridging operations may fail but still consume user funds
* Emergency response time is significantly delayed due to inability to pause operations
* Potential loss of user confidence and reputation damage

Duration: The funds would be frozen for at least 24 hours while the team attempts to resolve the emergency and deploy fixes.

## References

* [TellerWithMultiAssetSupportPredicateProxy.sol](broken://pages/90b118c59163ff4330c5bc4806c15a4230b9b32b)
* [OpenZeppelin Pausable Contract](broken://pages/b60467ccc27098dde41b8783e922f91e3bf18a40)
* [DexAggregatorWrapperWithPredicateProxy.sol](broken://pages/297bff1529eadcb7f89a040795b87f0a1979f86f) - Shows dependency on the predicate proxy

## Proof of Concept

```solidity
function testCannotPausePredicateProxy() public {
    // Deploy the contract
    TellerWithMultiAssetSupportPredicateProxy proxy = new TellerWithMultiAssetSupportPredicateProxy(
        address(this), address(0x1234), "policy"
    );

    // The contract is not paused by default
    assertEq(proxy.paused(), false);

    // Try to call pause() - this should fail to compile or revert at runtime
    (bool success, ) = address(proxy).call(abi.encodeWithSignature("pause()"));
    assertTrue(!success, "pause() should not exist");

    // Try to call _pause() - this should also fail
    (success, ) = address(proxy).call(abi.encodeWithSignature("_pause()"));
    assertTrue(!success, "_pause() should not exist");

    // The contract is still not paused
    assertEq(proxy.paused(), false);
}
```
