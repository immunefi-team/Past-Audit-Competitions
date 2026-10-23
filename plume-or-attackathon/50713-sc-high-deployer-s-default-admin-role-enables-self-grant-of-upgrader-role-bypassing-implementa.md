# 50713 sc high deployer s default admin role enables self grant of upgrader role bypassing implementation whitelist

**Submitted on Jul 27th 2025 at 18:52:08 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50713
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

`ArcTokenFactory::createToken()` gives the external caller `msg.sender` the DEFAULT\_ADMIN\_ROLE on every new ArcToken.

Because DEFAULT\_ADMIN\_ROLE is the admin role of all other roles, the deployer can immediately grant themselves UPGRADER\_ROLE and execute `upgradeTo()` on the token proxy installing any implementation, reviewed or not.

This bypasses the factory’s `allowedImplementations` whitelist and defeats the intended upgrade safety mechanism — any byte-code can be injected as the new ArcToken implementation.

## Impact Details

Category is High because the intended safety flow to allow only trusted upgrades is completely bypassed.

With a new implementation the malicious upgrader could do anything, for instance:

* mint arbitrary number of tokens
* rug-pull by changing yieldToken to an attacker-owned ERC20 and “distributing” worthless tokens

## Recommended Fix

{% hint style="warning" %}
Change the admin of UPGRADER\_ROLE to itself or to any other SUPER\_ADMIN controlled by the factory's DEFAULT\_ADMIN\_ROLE address so DEFAULT\_ADMIN\_ROLE cannot directly self-assign UPGRADER\_ROLE.

Example:

```
_setRoleAdmin(UPGRADER_ROLE, UPGRADER_ROLE);  // self-admin
```

This ensures DEFAULT\_ADMIN\_ROLE cannot grant UPGRADER\_ROLE without prior governance.
{% endhint %}

## Proof of Concept

<details>

<summary>Granting DEFAULT_ADMIN_ROLE to deployer while UPGRADER_ROLE is assigned to factory</summary>

In `ArcTokenFactory::createToken()` the DEFAULT\_ADMIN\_ROLE is granted to `msg.sender` while the UPGRADER\_ROLE is granted to the same contract `address(this)`:

```solidity
function createToken( ... ) external returns (address) {
    ....
    // Grant all necessary roles to the owner
    // Grant the DEFAULT_ADMIN_ROLE to the deployer
    token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
    token.grantRole(token.ADMIN_ROLE(), msg.sender);
    token.grantRole(token.MANAGER_ROLE(), msg.sender);
    token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
    token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
    token.grantRole(token.MINTER_ROLE(), msg.sender);
    token.grantRole(token.BURNER_ROLE(), msg.sender);
    token.grantRole(token.UPGRADER_ROLE(), address(this));
    ...
}
```

In `AccessControlUpgradeable` the `DEFAULT_ADMIN_ROLE` corresponds to `bytes32 0x00`, therefore if `ADMIN_ROLE` has not been changed the `DEFAULT_ADMIN_ROLE` can grant anyone the role of `UPGRADER_ROLE`:

```solidity
bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
...
function getRoleAdmin(bytes32 role) public view virtual returns (bytes32) {
    AccessControlStorage storage $ = _getAccessControlStorage();
    return $._roles[role].adminRole;
}
...

function grantRole(bytes32 role, address account) public virtual onlyRole(getRoleAdmin(role)) {
    _grantRole(role, account);
}
```

In fact in `ArcToken` only the address corresponding to `UPGRADER_ROLE` can authorize upgrade of implementation:

```solidity
function _authorizeUpgrade(
    address newImplementation
) internal override onlyRole(UPGRADER_ROLE) { }
```

The intended address allowed to upgrade the token implementation is the `DEFAULT_ADMIN_ROLE` of the `ArcTokenFactory`. This is enforced in `ArcTokenFactory::upgradeToken()`:

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
    UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");

    // Update the implementation mapping
    fs.tokenToImplementation[token] = newImplementation;

    emit TokenUpgraded(token, newImplementation);
}
```

Because the deployer gets `DEFAULT_ADMIN_ROLE` on the token, they can call `grantRole(UPGRADER_ROLE, attacker)` (or to themselves) and then call `UUPSUpgradeable.upgradeTo()` to set any implementation — bypassing the `allowedImplementations` whitelist check performed in the factory-level `upgradeToken()`.

</details>
