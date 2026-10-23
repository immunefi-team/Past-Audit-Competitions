# 52499 sc high arctoken factory s admin cannot upgrade an arctoken

* Submitted on Aug 11th 2025 at 09:21:24 UTC by @IronsideSec for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #52499
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

ArcToken governance is inconsistent with the intended “factory-controlled upgrades.” The token deployer (token `DEFAULT_ADMIN_ROLE`) can revoke the factory’s `UPGRADER_ROLE` on a token, causing the factory’s `upgradeToken` to revert. Additionally, the token deployer can grant themselves `UPGRADER_ROLE` and upgrade the token to any implementation, bypassing the factory’s whitelist. This can lead to governance DoS on upgrades or arbitrary/malicious upgrades.

Note: see `Recommendations` section, if you intend this is an intended design.

### Vulnerability Details

* Because the token deployer has `DEFAULT_ADMIN_ROLE` on the token, they can:
  * Revoke the factory’s `UPGRADER_ROLE` on that token, making factory upgrades revert.
  * Grant themselves `UPGRADER_ROLE` and then call `upgradeTo/upgradeToAndCall` to a non-whitelisted implementation, bypassing factory controls.

High-level steps (replicated as a POC test) are shown below.

{% stepper %}
{% step %}

### Step: Create a token via factory

User creates a token via factory; user is token `DEFAULT_ADMIN_ROLE`.
{% endstep %}

{% step %}

### Step: Revoke factory upgrader role

User revokes factory’s `UPGRADER_ROLE` on that token:

```solidity
ArcToken(token).revokeRole(ArcToken(token).UPGRADER_ROLE(), address(factory))
```

{% endstep %}

{% step %}

### Step: Factory admin whitelists implementation and attempts upgrade

Factory admin whitelists a new implementation in the factory and calls:

```solidity
factory.upgradeToken(token, newImpl)
```

{% endstep %}

{% step %}

### Step: Call reverts

Call reverts with `AccessControlUnauthorizedAccount(factory, UPGRADER_ROLE)` from the token’s `_authorizeUpgrade`.
{% endstep %}
{% endstepper %}

Additional technical observations:

* Factory grants itself `UPGRADER_ROLE` at token creation:

Source: arc/src/ArcTokenFactory.sol

```solidity
// ... in createToken()
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
token.grantRole(token.ADMIN_ROLE(), msg.sender);
token.grantRole(token.MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
token.grantRole(token.MINTER_ROLE(), msg.sender);
token.grantRole(token.BURNER_ROLE(), msg.sender);
token.grantRole(token.UPGRADER_ROLE(), address(this));
```

* Token upgrade authorization checks only for `UPGRADER_ROLE`; it does not verify the factory whitelist:

Source: arc/src/ArcToken.sol

```solidity
function initialize(
    // ...
) public initializer {
    require(routerAddress_ != address(0), "Router address cannot be zero");
    // ...
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(ADMIN_ROLE, msg.sender);
    _grantRole(MANAGER_ROLE, msg.sender);
    _grantRole(YIELD_MANAGER_ROLE, msg.sender);
    _grantRole(YIELD_DISTRIBUTOR_ROLE, msg.sender);
    // ...
}

function _authorizeUpgrade(
    address newImplementation
) internal override onlyRole(UPGRADER_ROLE) { }
```

* Factory’s upgrade entrypoint enforces whitelist but relies on being able to call the token’s UUPS upgrade, which requires the token-level `UPGRADER_ROLE`:

Source: arc/src/ArcTokenFactory.sol

```solidity
function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) {
    // ...
    bytes32 codeHash = _getCodeHash(newImplementation);
    if (!fs.allowedImplementations[codeHash]) {
        revert ImplementationNotWhitelisted();
    }

    UUPSUpgradeable(token).upgradeToAndCall(newImplementation, "");

    fs.tokenToImplementation[token] = newImplementation;
}
```

### Impact Details

* Factory cannot upgrade tokens it created (and is expected to govern) if the token deployer revokes `UPGRADER_ROLE` → governance DoS on applying critical fixes or parameter changes.
* Token deployer can self-assign `UPGRADER_ROLE` and upgrade to arbitrary, non-whitelisted implementations → potential malicious logic deployment and asset loss.
* Severity: High for governance integrity; potentially Critical if the token holds or manages funds post-upgrade.

### Recommendations

* If the factory is intended to be the sole upgrade authority:
  * Restrict token upgrades to the factory: in `ArcToken._authorizeUpgrade`, require `msg.sender == factory` OR a factory-owned role that is not administrable by the token deployer (e.g., set the role admin to a factory-controlled admin role, not the token `DEFAULT_ADMIN_ROLE`).
  * Alternatively, make the token’s `UPGRADER_ROLE` admin a role held by the factory, preventing the token deployer from revoking or reassigning it.
* If the design intentionally allows token-level sovereignty:
  * Enforce factory whitelist at the token level: in `ArcToken._authorizeUpgrade`, also validate that the `newImplementation` is whitelisted by the factory (e.g., call a factory getter like `isImplementationWhitelisted(newImplementation)`), so even sovereign upgrades cannot bypass the whitelist.
  * Consider requiring that any upgrade be initiated via the factory (e.g., token delegates upgrade authority to factory), even if deployer retains other management roles.

### References

See code snippets with GitHub links in the Vulnerability Details section above:

* Factory createToken: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L200>
* ArcToken initialize & \_authorizeUpgrade: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L113-L117> and <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L770-L772>
* Factory upgradeToken: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L261-L276>

## Proof of Concept

Link to PoC: <https://gist.github.com/IronsideSec/3181bbdc8a917b6207580c69a96610ef>

Follow the steps in the gist above to reproduce.
