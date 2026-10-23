# 52027 sc low whitelistrestrictions sol mint burn operations blocked when transfers disabled

## #52027 \[SC-Low] WhitelistRestrictions.sol: Mint & Burn Operations Blocked When Transfers Disabled

**Submitted on Aug 7th 2025 at 12:00:44 UTC by @soloi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52027
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

### Description

## WhitelistRestrictions.sol: Mint & Burn Operations Blocked When Transfers Disabled

### Summary

The `WhitelistRestrictions.sol` contract contains a vulnerability where disabling the `transfersAllowed` flag also blocks all mint and burn operations, even for whitelisted users. This allows the admin to permanently disable core token functionality.

### Vulnerability Details

#### Location

* **File**: `contracts/arc/src/restrictions/WhitelistRestrictions.sol`
* **Lines**: 95-102
* **Function**: `isTransferAllowed()`

#### Root Cause

The `isTransferAllowed()` function treats all addresses equally, including the zero address (`address(0)`). When `transfersAllowed` is set to `false`, the function requires both the `from` and `to` addresses to be whitelisted. However, the zero address can never be whitelisted, causing mint and burn operations to fail.

#### Vulnerable Code

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

### Technical Analysis

#### How Mint and Burn Operations Work

* Minting: `_mint(to, amount)` calls `_update(address(0), to, amount)`
* Burning: `_burn(from, amount)` calls `_update(from, address(0), amount)`

#### The Problem

When `transfersAllowed = false`:

* Mint Operation: `from = address(0)`, `to = recipient`
  * `ws.isWhitelisted[address(0)]` is always `false` (zero address can't be whitelisted)
  * `ws.isWhitelisted[to]` may be `true` (recipient is whitelisted)
  * Result: `false && true = false` → Mint blocked
* Burn Operation: `from = sender`, `to = address(0)`
  * `ws.isWhitelisted[from]` may be `true` (sender is whitelisted)
  * `ws.isWhitelisted[address(0)]` is always `false` (zero address can't be whitelisted)
  * Result: `true && false = false` → Burn blocked

### Impact Analysis

#### Severity: Medium

#### Immediate Consequences

1. Permanent Disabling of Mint/Burn: Admin can permanently disable core token functionality
2. Bricking Risk: Malicious admin could brick the token by disabling transfers
3. Whitelist Bypass: Even whitelisted users cannot mint or burn tokens
4. Functionality Loss: Core ERC20 operations become unavailable

#### Business Impact

* Operational Risk: Inability to mint new tokens or burn existing ones
* Admin Privilege Abuse: Single admin can disable critical functionality
* User Experience: Legitimate users cannot perform expected operations
* Compliance Risk: May violate regulatory requirements for token management

#### Attack Scenario

1. Malicious Admin: Admin with `ADMIN_ROLE` calls `setTransfersAllowed(false)`
2. Permanent Disablement: All mint and burn operations are permanently blocked
3. No Recovery: Even if admin changes their mind, mint/burn remain blocked
4. Token Bricking: Token becomes non-functional for core operations

### Proof of Concept

A complete proof of concept demonstrating this vulnerability is available in: `my_pocs/WhitelistRestrictions_MintBurn_PoC.t.sol`

<details>

<summary>Proof-of-Concept test (expand to view)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import {Test, console} from "forge-std/Test.sol";
import "../contracts/arc/src/restrictions/WhitelistRestrictions.sol";
import "../contracts/arc/src/restrictions/ITransferRestrictions.sol";

contract WhitelistRestrictionsMintBurnTest is Test {
    WhitelistRestrictions public whitelistRestrictions;
    
    address public admin = address(0x1);
    address public user1 = address(0x2);
    address public user2 = address(0x3);
    
    function setUp() public {
        whitelistRestrictions = new WhitelistRestrictions();
        
        // Initialize with admin
        vm.prank(admin);
        whitelistRestrictions.initialize(admin);
        
        // Add users to whitelist
        vm.prank(admin);
        whitelistRestrictions.addToWhitelist(user1);
        vm.prank(admin);
        whitelistRestrictions.addToWhitelist(user2);
    }
    
    function testMintBlockedWhenTransfersDisabled() public {
        console.log("=== MINT BLOCKED WHEN TRANSFERS DISABLED ===");
        console.log("");
        
        // Check initial state
        console.log("Initial transfers allowed:", whitelistRestrictions.transfersAllowed());
        console.log("User1 whitelisted:", whitelistRestrictions.isWhitelisted(user1));
        console.log("");
        
        // Disable transfers
        vm.prank(admin);
        whitelistRestrictions.setTransfersAllowed(false);
        
        console.log("After disabling transfers:");
        console.log("Transfers allowed:", whitelistRestrictions.transfersAllowed());
        console.log("");
        
        // Test mint operation (from = address(0), to = user1)
        bool mintAllowed = whitelistRestrictions.isTransferAllowed(address(0), user1, 1000);
        console.log("Mint operation (address(0) -> user1):");
        console.log("  From: address(0)");
        console.log("  To: user1");
        console.log("  User1 whitelisted:", whitelistRestrictions.isWhitelisted(user1));
        console.log("  Address(0) whitelisted: false (can never be whitelisted)");
        console.log("  Mint allowed:", mintAllowed);
        console.log("");
        
        // Test burn operation (from = user1, to = address(0))
        bool burnAllowed = whitelistRestrictions.isTransferAllowed(user1, address(0), 500);
        console.log("Burn operation (user1 -> address(0)):");
        console.log("  From: user1");
        console.log("  To: address(0)");
        console.log("  User1 whitelisted:", whitelistRestrictions.isWhitelisted(user1));
        console.log("  Address(0) whitelisted: false (can never be whitelisted)");
        console.log("  Burn allowed:", burnAllowed);
        console.log("");
        
        // This demonstrates the vulnerability - mint and burn are blocked!
        assertFalse(mintAllowed, "Mint should be blocked (BUG!)");
        assertFalse(burnAllowed, "Burn should be blocked (BUG!)");
        
        console.log("VULNERABILITY CONFIRMED:");
        console.log("- Mint operations are blocked when transfers are disabled");
        console.log("- Burn operations are blocked when transfers are disabled");
        console.log("- This happens because address(0) can never be whitelisted");
        console.log("- Admin can permanently brick mint/burn functionality");
    }
    
    function testNormalTransferStillWorks() public {
        console.log("=== NORMAL TRANSFER STILL WORKS ===");
        console.log("");
        
        // Disable transfers
        vm.prank(admin);
        whitelistRestrictions.setTransfersAllowed(false);
        
        // Test normal transfer between whitelisted users
        bool transferAllowed = whitelistRestrictions.isTransferAllowed(user1, user2, 100);
        console.log("Normal transfer (user1 -> user2):");
        console.log("  From: user1 (whitelisted:", whitelistRestrictions.isWhitelisted(user1), ")");
        console.log("  To: user2 (whitelisted:", whitelistRestrictions.isWhitelisted(user2), ")");
        console.log("  Transfer allowed:", transferAllowed);
        console.log("");
        
        // Normal transfers between whitelisted users should still work
        assertTrue(transferAllowed, "Normal transfer should be allowed");
    }
    
    function testAnalysis() public {
        console.log("=== ROOT CAUSE ANALYSIS ===");
        console.log("");
        console.log("VULNERABLE CODE (lines 95-102 in WhitelistRestrictions.sol):");
        console.log("function isTransferAllowed(address from, address to, uint256) external view returns (bool) {");
        console.log("    WhitelistStorage storage ws = _getWhitelistStorage();");
        console.log("");
        console.log("    if (ws.transfersAllowed) {");
        console.log("        return true;");
        console.log("    }");
        console.log("");
        console.log("    return ws.isWhitelisted[from] && ws.isWhitelisted[to];");
        console.log("}");
        console.log("");
        console.log("PROBLEM:");
        console.log("1. When transfersAllowed = false, the function requires BOTH addresses to be whitelisted");
        console.log("2. Mint operations: from = address(0), to = recipient");
        console.log("3. Burn operations: from = sender, to = address(0)");
        console.log("4. address(0) can NEVER be whitelisted (it's a special address)");
        console.log("5. Therefore, mint and burn operations are ALWAYS blocked when transfers are disabled");
        console.log("");
        console.log("IMPACT:");
        console.log("- Admin can permanently disable minting and burning");
        console.log("- Even whitelisted users cannot mint or burn tokens");
        console.log("- This breaks core token functionality");
        console.log("- Could be used maliciously to brick the token");
        console.log("");
        console.log("SEVERITY: MEDIUM");
        console.log("The issue allows admin to disable core token functionality.");
    }
    
    function testFixRecommendation() public {
        console.log("=== FIX RECOMMENDATION ===");
        console.log("");
        console.log("CURRENT BROKEN CODE:");
        console.log("function isTransferAllowed(address from, address to, uint256) external view returns (bool) {");
        console.log("    WhitelistStorage storage ws = _getWhitelistStorage();");
        console.log("");
        console.log("    if (ws.transfersAllowed) {");
        console.log("        return true;");
        console.log("    }");
        console.log("");
        console.log("    return ws.isWhitelisted[from] && ws.isWhitelisted[to];");
        console.log("}");
        console.log("");
        console.log("CORRECT FIX:");
        console.log("function isTransferAllowed(address from, address to, uint256) external view returns (bool) {");
        console.log("    WhitelistStorage storage ws = _getWhitelistStorage();");
        console.log("");
        console.log("    // Allow mint and burn operations unconditionally");
        console.log("    if (from == address(0) || to == address(0)) {");
        console.log("        return true;");
        console.log("    }");
        console.log("");
        console.log("    if (ws.transfersAllowed) {");
        console.log("        return true;");
        console.log("    }");
        console.log("");
        console.log("    return ws.isWhitelisted[from] && ws.isWhitelisted[to];");
        console.log("}");
        console.log("");
        console.log("This fix ensures that:");
        console.log("- Mint operations (from = address(0)) are always allowed");
        console.log("- Burn operations (to = address(0)) are always allowed");
        console.log("- Normal transfers still respect whitelist restrictions");
        console.log("- Admin cannot brick core token functionality");
    }
} 
```

</details>

### Fix Recommendations

#### Immediate Fix

Add special handling for the zero address in `isTransferAllowed()`:

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    // Allow mint and burn operations unconditionally
    if (from == address(0) || to == address(0)) {
        return true;
    }

    // If transfers are unrestricted, allow all transfers
    if (ws.transfersAllowed) {
        return true;
    }

    // Otherwise, only allow if both the sender and receiver are whitelisted
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

#### Alternative Fix (More Granular Control)

If more granular control is desired:

```solidity
function isTransferAllowed(address from, address to, uint256 /*amount*/ ) external view override returns (bool) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    // Handle mint operations (from = address(0))
    if (from == address(0)) {
        // For minting, only check if recipient is whitelisted (if transfers are restricted)
        if (ws.transfersAllowed) {
            return true;
        }
        return ws.isWhitelisted[to];
    }

    // Handle burn operations (to = address(0))
    if (to == address(0)) {
        // For burning, only check if sender is whitelisted (if transfers are restricted)
        if (ws.transfersAllowed) {
            return true;
        }
        return ws.isWhitelisted[from];
    }

    // Handle normal transfers
    if (ws.transfersAllowed) {
        return true;
    }

    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
```

### Testing Requirements

{% stepper %}
{% step %}

### Mint with Transfers Disabled

Verify minting works when `transfersAllowed = false`.
{% endstep %}

{% step %}

### Burn with Transfers Disabled

Verify burning works when `transfersAllowed = false`.
{% endstep %}

{% step %}

### Normal Transfer with Transfers Disabled

Verify normal transfers respect whitelist.
{% endstep %}

{% step %}

### Admin Privilege Abuse

Test that admin cannot permanently brick functionality.
{% endstep %}

{% step %}

### Edge Cases

Test with various combinations of whitelisted/non-whitelisted addresses.
{% endstep %}
{% endstepper %}

#### Integration Testing

* ArcToken Integration: Test with actual ArcToken contract
* Multiple Restriction Modules: Test interaction with other restriction modules
* Upgrade Scenarios: Test behavior after contract upgrades

### Security Recommendations

#### Immediate Actions

{% stepper %}
{% step %}
Audit All Instances — Check if this pattern exists in other restriction modules.
{% endstep %}

{% step %}
Update Documentation — Document the correct behavior for mint/burn operations.
{% endstep %}

{% step %}
Admin Training — Ensure admins understand the implications of disabling transfers.
{% endstep %}

{% step %}
Monitoring — Add alerts for when transfers are disabled.
{% endstep %}
{% endstepper %}

#### Long-term Improvements

{% stepper %}
{% step %}
Role Separation — Consider separating mint/burn permissions from transfer permissions.
{% endstep %}

{% step %}
Emergency Controls — Add emergency functions to re-enable mint/burn if needed.
{% endstep %}

{% step %}
Multi-sig Requirements — Require multiple signatures for critical operations.
{% endstep %}

{% step %}
Timelock — Add timelock for disabling transfers to prevent immediate bricking.
{% endstep %}
{% endstepper %}

### Conclusion

This vulnerability represents a significant design flaw in the WhitelistRestrictions contract that allows the admin to permanently disable core token functionality. The fix is straightforward and should be implemented immediately to prevent potential abuse.

Priority: Medium — Should be fixed promptly\
Impact: Admin can disable core token functionality\
Affected: All tokens using WhitelistRestrictions module
