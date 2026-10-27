# 51929 sc low deactivating istransferallowed indirectly doses minting burning functionality

**Submitted on Aug 6th 2025 at 16:46:44 UTC by @Am3nh3l for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51929
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description / Brief

When `transfersAllowed` is disabled in `WhitelistRestrictions`, minting and burning ArcTokens become permanently impossible because `address(0)` cannot be whitelisted. This creates a permanent denial-of-service condition for core token functionality.

***

## Vulnerability Details

The `WhitelistRestrictions` contract controls transfer permissions via the `transfersAllowed` flag:

```solidity
function setTransfersAllowed(
    bool allowed
) external onlyRole(ADMIN_ROLE) {
    _getWhitelistStorage().transfersAllowed = allowed;
    emit TransfersRestrictionToggled(allowed);
}
```

When `transfersAllowed` is false, the `isTransferAllowed` function requires both `from` and `to` addresses to be whitelisted:

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();
    if (ws.transfersAllowed) {
        return true;
    }
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

However, `addToWhitelist` prevents whitelisting `address(0)`:

```solidity
function addToWhitelist(
    address account
) external onlyRole(MANAGER_ROLE) {
    if (account == address(0)) {
        revert InvalidAddress();
    }

    WhitelistStorage storage ws = _getWhitelistStorage();
    if (ws.isWhitelisted[account]) {
        revert AlreadyWhitelisted(account);
    }

    ws.isWhitelisted[account] = true;
    ws.whitelistedAddresses.add(account);
    emit WhitelistStatusChanged(account, true);
    emit AddedToWhitelist(account);
}
```

In `ArcToken`, the `_update` function, used for all token movements (including minting and burning), enforces transfer restrictions:

```solidity
function _update(address from, address to, uint256 amount) internal virtual override {
    //...
    address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
    if (specificTransferModule != address(0)) {
        transferAllowed =
@>1             transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
    }

    address globalTransferModule = IRestrictionsRouter(routerAddr).getGlobalModuleAddress(RestrictionTypes.GLOBAL_SANCTIONS_TYPE);
    if (globalTransferModule != address(0)) {
        try ITransferRestrictions(globalTransferModule).isTransferAllowed(from, to, amount) returns (
            bool globalAllowed
        ) {
@>2         transferAllowed = transferAllowed && globalAllowed;
        } catch {
            transferAllowed = false;
        }
    }

    if (!transferAllowed) {
@>3     revert TransferRestricted();
    }

    //...
}
```

Minting uses `from = address(0)` and burning uses `to = address(0)`. If `_getWhitelistStorage().transfersAllowed` is false, minting and burning fail because `address(0)` cannot be whitelisted, causing `isTransferAllowed` to return false and transactions to revert with `TransferRestricted`.

***

## Impact Details

Setting `transfersAllowed` to false indirectly prevents minting and burning of `ArcToken`, halting critical functionality like issuing new tokens or reducing supply. This DoS affects token management and disrupts operations. There is no recovery path without either opening transfers to all users (defeating the whitelist) or allowing `address(0)` to be whitelisted (currently prevented).

***

## Proof of Concept

{% stepper %}
{% step %}

### Steps to reproduce

1. Global/Specific admin calls `setTransfersAllowed(false)` in `WhitelistRestrictions`.
   {% endstep %}

{% step %}
2\. Token admin attempts to mint tokens to a user via `ArcToken.mint()`.
{% endstep %}

{% step %}
3\. Mint triggers a transfer check from `address(0)` to recipient; `isTransferAllowed(address(0), recipient, amount)` is called.
{% endstep %}

{% step %}
4\. `isTransferAllowed` requires both addresses to be whitelisted, but `address(0)` cannot be whitelisted due to `addToWhitelist` reverting on `address(0)`.
{% endstep %}

{% step %}
5\. The whitelist check fails and the mint transaction reverts with `TransferRestricted`.
{% endstep %}

{% step %}
6\. The same failure occurs for burn operations (which use `to = address(0)`).
{% endstep %}

{% step %}
7\. No recovery is possible since `address(0)` cannot be whitelisted under current contract logic.
{% endstep %}
{% endstepper %}

***

## References

(Add any relevant links to documentation or code)

* Target contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>

***
