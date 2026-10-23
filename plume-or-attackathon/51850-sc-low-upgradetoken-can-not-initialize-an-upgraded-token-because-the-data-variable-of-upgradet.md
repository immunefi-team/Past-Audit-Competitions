# 51850 sc low upgradetoken can not initialize an upgraded token because the data variable of upgradetoandcall is hardcoded to empty string

Submitted on Aug 6th 2025 at 08:16:42 UTC by @kaysoft for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51850
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The second argument of `upgradeToAndCall()` called in the `upgradeToken(...)` is used to initialize variables during upgrade.

The `upgradeToken(...)` cannot initialize an upgraded token because the `data` variable of `upgradeToAndCall()` is hardcoded to an empty string.

```solidity
File: ArcTokenFactory.sol
 // Perform the upgrade (this assumes the token implements UUPSUpgradeable)
        UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");//FINDING: Empty string hardcoded blocks calling of init functions during upgrade.
```

### Vulnerability Details

The empty string (`""`) is used in the call to `upgradeToAndCall()` made in the `upgradeToken(...)` function in ArcTokenFactory.sol.

This prevents initializing state during an upgrade. For example, if a new upgrade adds `PausableUpgradeable`, its initialization function (which sets the pause-related state) cannot be invoked during the same transaction because no `data` payload is forwarded.

Code excerpt:

```solidity
function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) {
        FactoryStorage storage fs = _getFactoryStorage();

        // Ensure the token was created by this factory
        if (fs.tokenToImplementation[token] == address(0)) {
            revert TokenNotCreatedByFactory();
        }

        // Ensure the new implementation is whitelisted
        bytes32 codeHash = _getCodeHash(newImplementation);
        if (!fs.allowedImplementations[codeHash]) {
            revert ImplementationNotWhitelisted();
        }

        // Perform the upgrade (this assumes the token implements UUPSUpgradeable)
        UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");//@audit empty string limits opportunity to initialize variables when upgrading.

        // Update the implementation mapping
        fs.tokenToImplementation[token] = newImplementation;

        emit TokenUpgraded(token, newImplementation);
    }
```

## Impact Details

If a new implementation requires initialization (e.g., to set storage variables introduced in the upgrade), `upgradeToken(...)` cannot initialize those variables because the `data` argument passed to `upgradeToAndCall()` is hardcoded as an empty string. This prevents performing the initialization atomically with the upgrade.

## Recommendation

Allow passing initialization calldata to `upgradeToken(...)` since the function is access-controlled. Example suggested change:

```diff
--   function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) {
++   function upgradeToken(address token, address newImplementation, bytes initData) external onlyRole(DEFAULT_ADMIN_ROLE) {
        FactoryStorage storage fs = _getFactoryStorage();

        // Ensure the token was created by this factory
        if (fs.tokenToImplementation[token] == address(0)) {
            revert TokenNotCreatedByFactory();
        }

        // Ensure the new implementation is whitelisted
        bytes32 codeHash = _getCodeHash(newImplementation);
        if (!fs.allowedImplementations[codeHash]) {
            revert ImplementationNotWhitelisted();
        }

        // Perform the upgrade (this assumes the token implements UUPSUpgradeable)
--        UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");
++        UUPSUpgradeable(token).upgradeToAndCall(newImplementation, initData);

        // Update the implementation mapping
        fs.tokenToImplementation[token] = newImplementation;

        emit TokenUpgraded(token, newImplementation);
    }
```

This preserves access control while enabling atomic upgrade+initialize behavior.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Admin of ArcTokenFactory.sol attempts to upgrade an ArcToken to add Permit support using `ERC20PermitUpgradeable`.
{% endstep %}

{% step %}

### Step

The new implementation requires calling `__ERC20Permit_init("ArcToken")` (or similar) in its initializer to set permit-related state.
{% endstep %}

{% step %}

### Step

Because `upgradeToken(...)` calls `upgradeToAndCall(newImplementation, "")` with an empty data argument, the initialization calldata cannot be passed and the required initialization does not occur within the upgrade transaction. The init must be executed separately, meaning the token won't be properly initialized atomically during upgrade.
{% endstep %}
{% endstepper %}
