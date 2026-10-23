# 52843 sc low the zero address cannot be whitelisted which means during restrictions minting and burning cannot work

**Submitted on Aug 13th 2025 at 15:24:51 UTC by @heeze for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52843
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The ArcToken contract's transfer restriction logic prevents the zero address from being whitelisted. As a result, when transfer restrictions are active, minting (which sends tokens from the zero address) and burning (which sends tokens to the zero address) are blocked. If this occurs in production, minting and burning operations will revert, potentially freezing token supply management.

### Vulnerability Details

Within the `_update` internal function of the ArcToken contract, transfer restrictions are enforced by checking both specific and global restriction modules. The logic explicitly prevents the zero address from being whitelisted. This is problematic because ERC20 minting and burning operations inherently involve the zero address: minting transfers tokens from `address(0)` to a user, and burning transfers tokens from a user to `address(0)`. If transfer restrictions are enabled and the zero address cannot be whitelisted, any attempt to mint or burn tokens will revert due to `TransferRestricted()`.

Relevant excerpts:

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

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    // If transfers are unrestricted, allow all transfers
    if (ws.transfersAllowed) {
        return true;
    }

    // Otherwise, only allow if both the sender and receiver are whitelisted
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

```solidity
address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
if (specificTransferModule != address(0)) {
    transferAllowed =
        transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
    // The zero address cannot be whitelisted, which means during restrictions minting and burning cannot work
}
...
if (!transferAllowed) {
    revert TransferRestricted();
}
```

```solidity
function _mint(address account, uint256 value) internal {
    if (account == address(0)) {
        revert ERC20InvalidReceiver(address(0));
    }
    _update(address(0), account, value);
}
```

### Impact Details

Any attempt to mint new tokens or burn existing tokens by the permissioned roles will fail when transfer restrictions are active. This can lead to several consequences:

* Token supply cannot be increased or decreased, freezing supply management.
* Yield distribution mechanisms that rely on minting or burning may be disrupted.
* Emergency or administrative actions requiring minting/burning will be impossible.
* Projects relying on dynamic supply (e.g., rebasing, staking rewards) may be inoperable.

## References

* ArcToken.sol - \_update function: <https://github.com/plumenetwork/contracts/blob/main/arc/src/ArcToken.sol>
* ERC20 Mint/Burn mechanics: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol>
* RestrictionTypes: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/restrictions/WhitelistRestrictions.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Deploy ArcToken with a transfer restriction module enabled

* Deploy the ArcToken contract and configure it to use a transfer restriction module (e.g., a whitelist module).
* Ensure the restriction module is active and enforces whitelisting for all transfers.
  {% endstep %}

{% step %}

### Whitelist users to mint tokens to

* Use the restriction module’s admin functions to add several user addresses to the whitelist.
* Verify that these users are successfully whitelisted and eligible to receive tokens.
  {% endstep %}

{% step %}

### Attempt to mint tokens to the users

* Call the `mint` function on ArcToken, specifying one of the whitelisted users as the recipient.
* The ERC20 mint operation internally calls `_update(address(0), recipient, amount)`, representing a transfer from the zero address to the user.
  {% endstep %}

{% step %}

### Transaction reverts with TransferRestricted()

* The restriction module checks if both the sender (`address(0)`) and recipient are whitelisted.
* Since the zero address cannot be whitelisted, the check fails.
* The transaction reverts with the `TransferRestricted()` error, and no tokens are minted.
  {% endstep %}

{% step %}

### Attempt to burn tokens from any user

* Call the `burn` function on ArcToken, specifying a whitelisted user and an amount.
* The ERC20 burn operation internally calls `_update(user, address(0), amount)`, representing a transfer from the user to the zero address.
  {% endstep %}

{% step %}

### Transaction reverts with TransferRestricted() on burn

* The restriction module checks if both the sender (user) and recipient (`address(0)`) are whitelisted.
* Since the zero address cannot be whitelisted, the check fails.
* The transaction reverts with the `TransferRestricted()` error, and no tokens are burned.
  {% endstep %}

{% step %}

### Confirm inability to whitelist zero address

* Attempt to whitelist the zero address using the restriction module’s admin function.
* The module rejects the operation (e.g., reverts with `InvalidAddress()`), confirming that the zero address cannot be whitelisted.
* As long as restrictions are active, all mint and burn operations will fail due to the inability to whitelist the zero address.
  {% endstep %}
  {% endstepper %}
