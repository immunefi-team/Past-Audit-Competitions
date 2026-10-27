# 51286 sc low event restrictionscreated uses wrong owner

Submitted on Aug 1st 2025 at 12:25:00 UTC by @p1ranh4 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network)

* Report ID: #51286
* Report Type: Smart Contract
* Severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsFactory.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The event `RestrictionsCreated` emitted when creating a new Whitelist restriction uses the wrong owner address.

### Vulnerability Details

This issue will mess up with external indexing services or systems that rely on correct data.

{% code title="RestrictionsFactory.createWhitelistRestrictions (excerpt)" %}

```
```

{% endcode %}

```sol
function createWhitelistRestrictions(address admin) external returns (address) {
    // Deploy a fresh implementation
    WhitelistRestrictions implementation = new WhitelistRestrictions();

    // Add the implementation to the whitelist
    FactoryStorage storage fs = _getFactoryStorage();
    bytes32 codeHash = _getCodeHash(address(implementation));
    fs.allowedImplementations[codeHash] = true;

    // Deploy proxy with the fresh implementation
    bytes memory initData =
        abi.encodeWithSelector(WhitelistRestrictions.initialize.selector, admin != address(0) ? admin : msg.sender); // @audit

    address proxy = _deployProxy(address(implementation), initData);

    // Store the mapping
    fs.restrictionsToImplementation[proxy] = address(implementation);

    emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist"); // @audit
    emit ImplementationWhitelisted(address(implementation));

    return proxy;
}
```

The contract is created with the admin being either: `admin` or `msg.sender` if `admin == address(0)`, however, the event will always emit with `msg.sender` as the owner.

In the current context, `msg.sender` as owner means nothing as this information is not stored anywhere. When an admin is set, the `msg.sender` has no special "access" or role. It is not relevant to emit the event in that way.

Maybe renaming the `owner` event parameter to `admin` would also make more sense.

## Impact Details

{% hint style="warning" %}
Breaks external systems and emits wrong information. This affects the contract's external interface and data integrity. Many systems in the DeFi ecosystem rely heavily on events for indexing and monitoring.
{% endhint %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsFactory.sol#L87>

## Proof of Concept

No PoC provided.
