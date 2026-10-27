# 50493 sc low immutable proxy implementation mapping in restrictionsfactory breaks upgrade logic

Submitted on Jul 25th 2025 at 11:36:24 UTC by @Paludo0x for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50493
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsFactory.sol>

Impacts:

* Temporary freezing of funds for at least 24 hours
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

`RestrictionsFactory` captures the initial proxy → implementation in `restrictionsToImplementation` mapping when deploying a `WhitelistRestrictions` module proxy but never updates that mapping on subsequent upgrades.

As a result, even benign upgrades leave the factory’s record stale, breaking any governance or tooling that relies on `getRestrictionsImplementation` to track module versions.

If an administrator upgrades a module to a new implementation (for bugfix or feature), the system loses the ability to recover or reference the correct logic, potentially freezing all dependent flows until manual intervention restores the old implementation.

### Vulnerability Details

The `RestrictionsFactory` exists to deploy generic, standalone whitelisting modules. Contracts that use these modules are expected to validate both:

1. the implementation address associated with a proxy via `factory.getRestrictionsImplementation(proxy);`
2. that this implementation is approved via `factory.isImplementationWhitelisted(impl);`

The bug: the `mapping(address => address) restrictionsToImplementation;` is written only once in `createWhitelistRestrictions()` and never updated thereafter, even when the proxy’s implementation is changed via `upgradeToAndCall(...)`.

This leads to two problematic scenarios:

* Benign upgrade → broken functionality

  A well-intentioned admin performs:

  * `proxy.upgradeTo(newImpl)`
  * removes the old implementation from allowed implementations via `removeWhitelistedImplementation()`
  * adds the new implementation to allowed implementations via `whitelistImplementation`

  Because `getRestrictionsImplementation(proxy)` still returns the old address, any tooling or contracts that rely on the factory to locate the module will fail to find the new implementation and halt interactions → temporary freezing of funds.
* Partial upgrade (old implementation left whitelisted) → whitelist inconsistency
  * The admin upgrades the proxy but neglects to remove the old implementation from `allowedImplementations`.
  * `getRestrictionsImplementation(proxy)` continues to return the old address, and `isImplementationWhitelisted(oldImpl)` remains true, so validation checks pass, but the proxy is executing unverified code that may not have been reviewed.

## Impact Details

* In the first case: High — Temporary freezing of funds for at least 24 hours. Recovery requires redeploying modules or manual state fixes via governance, potentially taking days and impacting users.
* In the second case: Critical — an important check/gate is bypassed, allowing a proxy to execute unverified code while factory records continue to point to the old implementation.

## Recommended fix

Introduce an atomic upgrade function in `RestrictionsFactory` that enforces the whitelist and keeps its mapping in sync:

```
function upgradeRestrictions(
    address proxy,
    address newImplementation
) external onlyRole(DEFAULT_ADMIN_ROLE) {
    FactoryStorage storage fs = _getFactoryStorage();
    bytes32 hash = _getCodeHash(newImplementation);
    require(fs.allowedImplementations[hash], "Impl not whitelisted");

    UUPSUpgradeable(proxy).upgradeTo(newImplementation);

    fs.restrictionsToImplementation[proxy] = newImplementation;
    emit RestrictionsUpgraded(proxy, newImplementation);
}
```

Additionally, remove `UPGRADER_ROLE` from individual module admins so all upgrades flow through this wrapper.

## Proof of Concept

To create a new whitelist module, the function `createWhitelistRestrictions()` is called:

```solidity
function createWhitelistRestrictions(
    address admin
) external returns (address) {
    // Deploy a fresh implementation
    WhitelistRestrictions implementation = new WhitelistRestrictions();

    // Add the implementation to the whitelist
    FactoryStorage storage fs = _getFactoryStorage();
    bytes32 codeHash = _getCodeHash(address(implementation));
    fs.allowedImplementations[codeHash] = true;

    // Deploy proxy with the fresh implementation
    bytes memory initData =
        abi.encodeWithSelector(WhitelistRestrictions.initialize.selector, admin != address(0) ? admin : msg.sender);

    address proxy = _deployProxy(address(implementation), initData);

    // Store the mapping
    fs.restrictionsToImplementation[proxy] = address(implementation);

    emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist");
    emit ImplementationWhitelisted(address(implementation));

    return proxy;
}
```

Use the stepper below to illustrate the key points in the PoC:

{% stepper %}
{% step %}

### Step: Whitelist and mapping set

* `fs.allowedImplementations[codeHash]` is set to `true` to whitelist the implementation deployment.
* `fs.restrictionsToImplementation[proxy] = address(implementation);` links the proxy with the implementation.
* The `initData` for initializing the proxy sets the admin (either `msg.sender` or the provided `admin`) who will be granted `UPGRADER_ROLE`.
  {% endstep %}

{% step %}

### Step: Proxy can be upgraded by UPGRADER\_ROLE

`WhitelistRestrictions` is an `UUPSUpgradeable` contract:

```solidity
contract WhitelistRestrictions is
    ITransferRestrictions,
    Initializable,
    UUPSUpgradeable,
    AccessControlEnumerableUpgradeable
```

The upgrade function is:

```solidity
function upgradeToAndCall(address newImplementation, bytes memory data) public payable virtual onlyProxy {
    _authorizeUpgrade(newImplementation);
    _upgradeToAndCallUUPS(newImplementation, data);
}
```

Authorization uses `UPGRADER_ROLE`:

```solidity
function _authorizeUpgrade(
    address newImplementation
) internal override onlyRole(UPGRADER_ROLE) { }
```

So any account with `UPGRADER_ROLE` can change the implementation directly on the proxy.
{% endstep %}

{% step %}

### Step: Factory lacks mapping update API

Factory has methods to manage whitelisted implementations, but no method updates `mapping(address => address) restrictionsToImplementation;`:

```solidity
function getRestrictionsImplementation(address restrictions) external view returns (address) {
    return _getFactoryStorage().restrictionsToImplementation[restrictions];
}

function whitelistImplementation(address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) { ... }

function removeWhitelistedImplementation(address implementation) external onlyRole(DEFAULT_ADMIN_ROLE) { ... }

function isImplementationWhitelisted(address implementation) external view returns (bool) { ... }
```

Because the factory mapping remains stale after a proxy-level upgrade, callers that rely on `getRestrictionsImplementation` will be misled.
{% endstep %}
{% endstepper %}

***

End of report.
