# 51941 sc high token creator can revoke factory s upgrade capability permanently blocking upgrades

**Submitted on Aug 6th 2025 at 19:03:53 UTC by @Am3nh3l for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51941
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

{% hint style="warning" %}
Token creators can permanently disable the factory's ability to upgrade token implementations by revoking the factory's UPGRADER\_ROLE, creating a permanent denial-of-service for upgrades (blocking security fixes and improvements).
{% endhint %}

## Brief / Intro

Token creators can revoke the factory's `UPGRADER_ROLE`, permanently preventing protocol admins from upgrading token implementations. This creates a permanent denial-of-service vulnerability for critical security updates. Other factory-related roles granted to the token creator can also be revoked.

## Vulnerability Details

During token creation the factory grants itself `UPGRADER_ROLE`:

```solidity
token.grantRole(token.UPGRADER_ROLE(), address(this));
```

The factory exposes an upgrade function that relies on UUPS upgradeability:

```solidity
function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) {
    ...
    UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");
    ...
}
```

However, the token creator is granted `DEFAULT_ADMIN_ROLE` and `ADMIN_ROLE` during creation:

```solidity
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
token.grantRole(token.ADMIN_ROLE(), msg.sender);
```

ArcToken's `_authorizeUpgrade` enforces that the caller has `UPGRADER_ROLE`:

```solidity
/**
 * @dev Authorization for upgrades
 */
function _authorizeUpgrade(
    address newImplementation
) internal override onlyRole(UPGRADER_ROLE) { }
```

Because `DEFAULT_ADMIN_ROLE` / `ADMIN_ROLE` holders can call `revokeRole` (from `AccessControlUpgradeable`), a malicious token creator can revoke the factory's `UPGRADER_ROLE`. Once revoked, `upgradeToAndCall` will fail the `_authorizeUpgrade` check, preventing any future upgrades initiated by the factory.

## Impact Details

The token creator can block the factory admin from upgrading the `ArcToken` to a new implementation, potentially leaving the token stuck on a vulnerable or outdated version. This undermines the intended upgradeability of the UUPS proxy system and prevents applying fixes or improvements.

## Proof of Concept

{% stepper %}
{% step %}
User creates a token via the factory.
{% endstep %}

{% step %}
The factory grants itself `UPGRADER_ROLE` during token creation.
{% endstep %}

{% step %}
Malicious token creator calls `revokeRole` to revoke the factory's `UPGRADER_ROLE`.
{% endstep %}

{% step %}
Factory admin attempts to upgrade the token via `upgradeToAndCall`.
{% endstep %}

{% step %}
The token's `_authorizeUpgrade` check (requires `UPGRADER_ROLE`) fails for the factory, so the upgrade is blocked.
{% endstep %}
{% endstepper %}

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
