# 52796 sc low whitelist restriction in arctoken blocks all minting and burning

**Submitted on Aug 13th 2025 at 08:54:33 UTC by @ZeroExRes for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52796
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The WhitelistRestrictions module has a critical issue where enabling transfer restrictions (`transfersAllowed = false`) blocks all minting and burning operations, even for whitelisted addresses and role-protected administrative functions. This occurs because the module requires both sender AND recipient addresses to be whitelisted, but `address(0)` (used in mint/burn operations) cannot be whitelisted due to validation checks. This renders the whitelist feature completely unusable for transfer restrictions and can lead to inability to adjust tokenonomics.

## Vulnerability Details

The core issue lies in the interaction below:

1. WhitelistRestrictions.isTransferAllowed() Logic:

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();
    
    if (ws.transfersAllowed) {
        return true; // Unrestricted transfers
    }
    
    // BUG: Requires BOTH addresses to be whitelisted
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

2. Zero Address Cannot Be Whitelisted:

```solidity
function addToWhitelist(address account) external onlyRole(MANAGER_ROLE) {
    if (account == address(0)) {
        revert InvalidAddress(); // Cannot whitelist address(0)
    }
    // ...
}
```

3. ERC20 Mint/Burn Uses Zero Address: Openzeppelin implementation is

* Minting: `_update(address(0), recipient, amount)` - `from = address(0)`
* Burning: `_update(sender, address(0), amount)` - `to = address(0)`

4. ArcToken Enforces Restrictions on these Operations:

```solidity
function _update(address from, address to, uint256 amount) internal virtual override {
    // ...
    if (specificTransferModule != address(0)) {
        transferAllowed = transferAllowed && 
            ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
    }
    
    if (!transferAllowed) {
        revert TransferRestricted(); // Blocks mint/burn when whitelist active
    }
    // ...
}
```

{% stepper %}
{% step %}

### Execution Flow Demonstrating the Bug — step

1. Admin sets `transfersAllowed = false` to enable whitelist restrictions
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

2. Admin attempts to mint tokens to whitelisted user Alice
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

3. `ArcToken.mint()` calls `_update(address(0), alice, amount)`
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

4. `WhitelistRestrictions.isTransferAllowed(address(0), alice, amount)` is called
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

5. Returns `isWhitelisted[address(0)] && isWhitelisted[alice]` = `false && true` = `false`
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

6. Transaction reverts with `TransferRestricted()`
   {% endstep %}

{% step %}

### Execution Flow Demonstrating the Bug — step

7. Same issue occurs for burning operations
   {% endstep %}
   {% endstepper %}

## Impact Details

All minting and burning operations become permanently blocked when whitelist is enabled without disabling the whitelist entirely. In essence, this vulnerability makes the whitelist restriction feature completely unusable for its intended purpose, forcing admin to choose between transfer restrictions OR functional minting/burning, but never both. This may result in further issues, for example, transfer would need to allowed in whitelist restrictions, at least temporarily for mint/burn, risking non-compliant transfer by the existing holders.

## References

Mentioned above.

## Proof of Concept

Add to ArcToken.t.sol

```solidity
    function test_RevertWhen_MintingWithWhitelistEnabled() public {
    // 1. Verify alice is whitelisted (she was whitelisted in setUp)
    assertTrue(whitelistModule.isWhitelisted(alice));
    
    // 2. Try to whitelist address(0) - this should fail
    vm.expectRevert(WhitelistRestrictions.InvalidAddress.selector);
    whitelistModule.addToWhitelist(address(0));
    
    // 3. Confirm address(0) is NOT whitelisted
    assertFalse(whitelistModule.isWhitelisted(address(0)));
    
    // 4. Enable whitelist enforcement (disable open transfers)
    whitelistModule.setTransfersAllowed(false);
    assertFalse(whitelistModule.transfersAllowed());
    
    // 5. Try to mint tokens to alice (who IS whitelisted)
    // This fails because minting calls _update(address(0), alice, amount)
    // and address(0) cannot be whitelisted, so isTransferAllowed returns false
    vm.expectRevert(ArcToken.TransferRestricted.selector);
    token.mint(alice, 100e18);
}
```
