# 51197 sc high arc token owner can take upgrader role for themselves lockout the factory and upgrade the contract without the knowledge of the factory

**Submitted on Jul 31st 2025 at 21:43:30 UTC by @TeamJosh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51197
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Escalation of privilege

## Description

### Brief/Intro

Anyone can create an ArcToken from the factory; the creator of the ArcToken gets the default admin role and the factory itself gets the upgrader role and stores the implementation contract. This means that all upgrades to an ArcToken can only go through the ArcTokenFactory admin. However, the ArcToken owner can revoke the Upgrader Role from ArcTokenFactory and also give themselves this role. By doing this they can upgrade the token to whatever they want while the factory will still think it has a legitimate implementation contract.

### Vulnerability Details

The contract ArcToken uses `AccessControlUpgradeable`. This contract contains public functions `grantRoles` and `revokeRoles` that allow the default admin to grant and revoke any role. With this capability the default admin (who is the owner of the ArcToken) can revoke the upgraderRole access of the ArcTokenFactory, thereby blocking the ArcTokenFactory admin from performing upgrades.

The ArcToken admin can also upgrade the ArcToken themselves to a malicious contract. The factory will still store the original (or another) implementation contract address in its mapping, causing the factory to report an incorrect implementation for that token.

Code excerpt showing the factory upgrade and implementation mapping update:

```sol
    /**
     * @dev Upgrades a token to a new implementation
     * @param token Token address to upgrade
     * @param newImplementation Address of the new implementation
     */
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
@->        fs.tokenToImplementation[token] = newImplementation;

        emit TokenUpgraded(token, newImplementation);
    }
```

## Impact Details

{% stepper %}
{% step %}

### Inconsistent factory state

The ArcTokenFactory can end up reporting an implementation address that doesn't match the token's actual implementation after the token owner upgrades it without the factory's authorization.
{% endstep %}

{% step %}

### Access control bypass / privilege escalation

A token owner can revoke the factory's upgrader role and grant themselves that role, allowing them to perform upgrades normally reserved for the factory admin.
{% endstep %}

{% step %}

### Use of unwhitelisted implementation

The token owner can upgrade to an implementation that is not whitelisted by the factory, bypassing the factory's expected whitelisting mechanism.
{% endstep %}

{% step %}

### Ecosystem harm

Token owners can upgrade to unsecured or unaudited contracts, potentially enabling theft or malicious behavior that harms users and the ecosystem.
{% endstep %}
{% endstepper %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol#L185>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol#L279>

## Proof of Concept

{% stepper %}
{% step %}

### Step: Create a legitimate ArcToken

Create an ArcToken through the ArcTokenFactory (the factory will be assigned the upgrader role and store the token implementation in its mapping).
{% endstep %}

{% step %}

### Step: Revoke factory upgrader role

Call `revokeRoles` on the ArcToken contract to revoke the ArcTokenFactory Upgrader Role.
{% endstep %}

{% step %}

### Step: Grant yourself upgrader role

Call `grantRoles` on the ArcToken contract to grant yourself the Upgrader Role.
{% endstep %}

{% step %}

### Step: Upgrade to non-whitelisted implementation

Call `upgradeToAndCall` on the ArcToken to update the implementation to a non-whitelisted (malicious) implementation. The token now runs the malicious implementation while the factory's mapping may still point to a different (legitimate) implementation.
{% endstep %}
{% endstepper %}
