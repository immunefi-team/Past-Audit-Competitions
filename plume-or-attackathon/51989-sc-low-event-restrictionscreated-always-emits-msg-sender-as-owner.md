# 51989 sc low event restrictionscreated always emits msg sender as owner&#x20;

**Submitted on Aug 7th 2025 at 04:35:12 UTC by @Killua for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51989
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/RestrictionsFactory.sol>

## Description

The `createWhitelistRestrictions()` function in the `RestrictionsFactory` contract always emits `msg.sender` as the owner in the `RestrictionsCreated` event.

When an `admin` parameter is provided, that admin becomes the actual controller of the restrictions module, but the event incorrectly logs `msg.sender` as the owner. The actual owner should be: `admin != address(0) ? admin : msg.sender`

## Vulnerability Details

The event emission currently uses:

```
emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist");
```

This will always log `msg.sender` as the owner even when a different `admin` was supplied and initialized as the controller.

## Impact Details

* Event logs will not accurately reflect the actual control structure of deployed modules.
* Possible off-chain monitoring and tooling may misrepresent ownership/control of the deployed module.

## Proof of Concept

Relevant snippet showing initialization and event emit:

```solidity
function createWhitelistRestrictions(address admin) external returns (address) {
    // ... existing code ...
     
    bytes memory initData =
        abi.encodeWithSelector(WhitelistRestrictions.initialize.selector, admin != address(0) ? admin : msg.sender);
    
    address proxy = _deployProxy(address(implementation), initData);
    
    // @audit Emits msg.sender always. 
    emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist");

    //..//..
}
```

Suggested change (conceptual) — capture the actual admin and use it in the event emission:

```solidity
address actualAdmin = admin != address(0) ? admin : msg.sender;
emit RestrictionsCreated(proxy, actualAdmin, address(implementation), "Whitelist");
```

## References

Original event usage:

```
emit RestrictionsCreated(proxy, msg.sender, address(implementation), "Whitelist");
```
