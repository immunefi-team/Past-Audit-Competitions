# 53034 sc high arctokenfactory doesn t properly handle role management which allows users to arbitrary upgrade their arctoken s implementation

**Submitted on Aug 14th 2025 at 17:46:03 UTC by @valkvalue for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53034
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>

## Impacts

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Permanent freezing of funds
* Theft of gas

## Description

### Brief/Intro

`ArcTokenFactory` doesn't properly handle role management which allows users to arbitrarily upgrade their ArcToken's implementation. This poses great risks because a malicious implementation of the ArcToken could be deployed, enabling theft of user funds. This could also be used to deny upgrade operations for the admin of the `ArcTokenFactory`.

### Vulnerability Details

By design the `ArcTokenFactory` should only be able to upgrade implementations of existing ArcTokens to avoid malicious `ArcToken` implementations being deployed via the trusted factory.

The factory's `createToken()` grants roles to the deployer/owner and grants the `UPGRADER_ROLE` to the factory itself:

```solidity
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
```

The factory's `upgradeToken` enforces that the token was created by the factory and that the new implementation is whitelisted, then performs a UUPS upgrade:

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
}
```

However, the logic misses that `DEFAULT_ADMIN_ROLE` (granted to the ArcToken's creator) is the admin role for other roles by default in OpenZeppelin's AccessControl. That means the token creator (who holds `DEFAULT_ADMIN_ROLE`) can grant themselves the `UPGRADER_ROLE` (or otherwise manipulate role admins) and thus perform arbitrary upgrades of the token's implementation, circumventing the factory's intended upgrade control.

OpenZeppelin reference:

```
 * By default, the admin role for all roles is `DEFAULT_ADMIN_ROLE`, which means
 * that only accounts with this role will be able to grant or revoke other
 * roles. More complex role relationships can be created by using
 * {_setRoleAdmin}.
```

This allows a creator to change the whole implementation of that `ArcToken` while keeping it appearing as a legitimately factory-deployed token.

## Impact Details

Legit-deployed `ArcToken` can be modified in many ways to steal funds or disrupt protocol assumptions. Examples (non-exhaustive and illustrative):

* Remove global sanction checks or other safety logic from a new implementation.
* Create honeypot tokens that behave normally in some flows but steal funds in others.
* Integrate malicious behavior in other protocol flows (e.g., ArcTokenPurchase) so purchases don't transfer tokens while taking payment.
* Brick upgrades from the factory admin by revoking or changing the factory's upgrade permissions.
* Force users or admins to overspend gas via malicious logic.

## References

* ArcTokenFactory snippet: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcTokenFactory.sol#L192-L200>
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>

## Proof of Concept

{% stepper %}
{% step %}

### User deploys RWA token

A user (token creator) deploys an ArcToken via the factory.
{% endstep %}

{% step %}

### User grants themself the UPGRADE\_ROLE

Because the deployer was granted `DEFAULT_ADMIN_ROLE` during `createToken()`, they can call AccessControl's grant functions to grant themselves `UPGRADER_ROLE` (or change role admins).
{% endstep %}

{% step %}

### Malicious upgrade

The creator (now adversary) upgrades the token to a malicious implementation (while the token still appears as factory-deployed). The malicious implementation can:

* Remove or bypass global checks (sanctions, restrictions).
* Steal funds in arbitrary flows (honeypots).
* Revoke the factory's upgrade permissions to prevent remediation.
  {% endstep %}
  {% endstepper %}
