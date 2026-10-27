# 50022 sc low missing admin pause unpause functions in tellerwithmultiassetsupportpredicateproxy contract

**Submitted on Jul 21st 2025 at 08:14:59 UTC by @honey0x0 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50022
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The `TellerWithMultiAssetSupportPredicateProxy` contract inherits from OpenZeppelin's `Pausable` contract and implements pause checks in its functions, but lacks public admin functions to actually pause or unpause the contract. This renders the pause functionality unusable while still having the infrastructure in place.

### Vulnerability Details

The contract inherits from `Pausable` and uses the `paused()` function to check if the contract is paused in its functions:

```solidity
contract TellerWithMultiAssetSupportPredicateProxy is Ownable, ReentrancyGuard, PredicateClient, Pausable {
```

```solidity
function deposit(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint,
    address recipient,
    CrossChainTellerBase teller,
    PredicateMessage calldata predicateMessage
)
    external
    nonReentrant
    returns (uint256 shares)
{
    if (paused()) {
        revert TellerWithMultiAssetSupportPredicateProxy__Paused();
    }
    ...
```

OpenZeppelin's `Pausable` exposes only internal functions `_pause()` and `_unpause()`:

```solidity
function _pause() internal virtual whenNotPaused {
    _paused = true;
    emit Paused(_msgSender());
}

function _unpause() internal virtual whenPaused {
    _paused = false;
    emit Unpaused(_msgSender());
}
```

The `TellerWithMultiAssetSupportPredicateProxy` contract does not expose any public functions that call these internal functions, making it impossible for the contract owner to pause or unpause the contract.

Typical public admin functions would be:

```solidity
function pause() external onlyOwner {
    _pause();
}

function unpause() external onlyOwner {
    _unpause();
}
```

### Impact Details

{% hint style="warning" %}
The contract has pause functionality built-in but no public way to activate it. In case of security incidents or emergencies, admins cannot pause the contract to prevent further deposits.
{% endhint %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L78-L79>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L133-L135>

## Proof of Concept

{% stepper %}
{% step %}

### Deployment

`TellerWithMultiAssetSupportPredicateProxy` is deployed with an `owner` address.
{% endstep %}

{% step %}

### Emergency

An emergency situation or a security incident arises which prompts the admin to pause the contract immediately.
{% endstep %}

{% step %}

### Attempt to Pause

Admin looks for public `pause()` / `unpause()` functions but none exist.
{% endstep %}

{% step %}

### Result

Contract can't be paused even though it has built-in pause functionality.
{% endstep %}
{% endstepper %}

## Recommended Fix

Add owner-restricted public functions that call the internal pause/unpause implementations from OpenZeppelin:

```solidity
function pause() external onlyOwner {
    _pause();
}

function unpause() external onlyOwner {
    _unpause();
}
```

This preserves the existing pause checks (e.g., `paused()`) while allowing the owner to operate the pause mechanism during emergencies.
