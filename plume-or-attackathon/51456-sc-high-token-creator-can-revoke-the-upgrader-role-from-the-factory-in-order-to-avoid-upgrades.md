# 51456 sc high token creator can revoke the upgrader role from the factory in order to avoid upgrades

* Report ID: #51456
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* Impacts: Contract fails to deliver promised returns, but doesn't lose value

## Brief / Intro

{% hint style="info" %}
Since the token creator is granted the DEFAULT\_ADMIN\_ROLE, the token creator can revoke or grant any role. This allows the token creator to remove the UPGRADER\_ROLE from the factory (or grant it to themselves), preventing the factory from performing intended upgrades or enabling arbitrary upgrades by the token owner.
{% endhint %}

## Vulnerability Details

When a token is created, the factory assigns multiple roles to the token creator and assigns the UPGRADER\_ROLE to the factory (address(this)):

```solidity
        token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
        token.grantRole(token.ADMIN_ROLE(), msg.sender);
        token.grantRole(token.MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
        token.grantRole(token.MINTER_ROLE(), msg.sender);
        token.grantRole(token.BURNER_ROLE(), msg.sender);
        token.grantRole(token.UPGRADER_ROLE(), address(this));
```

Because the token creator is granted DEFAULT\_ADMIN\_ROLE, they can revoke or grant any role, including revoking the UPGRADER\_ROLE from the factory. The token enforces upgrades via UUPS and requires UPGRADER\_ROLE:

```solidity
function _authorizeUpgrade(address newImplementation) internal override onlyRole(UPGRADER_ROLE) { }
```

The factory exposes an admin-only upgrade function that expects to be able to upgrade tokens it created:

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

If the token creator revokes UPGRADER\_ROLE from the factory, the factory's call to UUPSUpgradeable(token).upgradeToAndCall will revert because \_authorizeUpgrade will fail the onlyRole(UPGRADER\_ROLE) check. Conversely, the token owner (having DEFAULT\_ADMIN\_ROLE) can grant themself UPGRADER\_ROLE and upgrade the token implementation arbitrarily.

## Impact Details

* The ArcTokenFactory will be unable to upgrade tokens it created if the token creator revokes the UPGRADER\_ROLE from the factory.
* Token creators could upgrade their token to an arbitrary implementation by granting themselves UPGRADER\_ROLE, which may break intended guarantees or introduce malicious logic.
* Overall: intended upgradeability and central admin control can be lost or subverted by token creators who hold DEFAULT\_ADMIN\_ROLE.

## Proof of Concept

{% stepper %}
{% step %}
User creates a token by calling createToken on ArcTokenFactory.
{% endstep %}

{% step %}
The factory grants the user roles including DEFAULT\_ADMIN\_ROLE.
{% endstep %}

{% step %}
The factory also grants itself the UPGRADER\_ROLE to allow upgrades.
{% endstep %}

{% step %}
The token creator revokes every role that was granted to the ArcTokenFactory (including UPGRADER\_ROLE).
{% endstep %}

{% step %}
The factory (admin) can no longer upgrade the token using upgradeToken because \_authorizeUpgrade requires UPGRADER\_ROLE.
{% endstep %}

{% step %}
The token creator can grant themself UPGRADER\_ROLE and upgrade the token implementation to any address.
{% endstep %}
{% endstepper %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L144>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L261>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L648>
