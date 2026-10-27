# 58146 sc insight whitelist can be disabled repeatedly contradicting intended program behavior&#x20;

## #58146 \[SC-Insight] Whitelist can be disabled repeatedly, contradicting intended program behavior.

**Submitted on Oct 30th 2025 at 23:17:47 UTC by @chief\_hunter888 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58146
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/Whitelist.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

\# `Whitelist.disable()` in `Whitelist.sol` can be called repeatedly contradicting intended program outcomes

### Brief Summary

`IWhitelist.sol` NatSpec states that `disable()` `can only occur once'' and` cannot be reenabled.'' The implementation, however, does not enforce a one-time-only disable: calling `disable()` multiple times is possible, and each call emits another `WhitelistDisabled` event. This breaks the promised invariant and can confuse/harm off-chain consumers and waste gas.

Please refer to the POC test below.

Please refer to the POC attached below.

It follows these steps: Evidence / reproduction

1. Deploy Whitelist (implements IWhitelist).
2. Call disable() → WhitelistDisabled event is emitted and whitelist is disabled.
3. Call disable() again → call succeeds again and a second WhitelistDisabled event is emitted.

### Why this is a bug

The NatSpec contract-level promise (semantic invariant) claims that the disable function is a single irreversible function, but the code allows repeated disables. That mismatch is a correctness/contractual bug.

Off-chain systems and backends often rely on event semantics (e.g., first `WhitelistDisabled` events for a canonical transition). Multiple identical disable events can confuse indexing, lead to incorrect state assumptions, or create noisy logs. Repeated calls waste gas for callers and can hide the true moment of transition (first disable) in logs/analytics.

Even if repeated disables cause no on-chain safety hazard, the mismatch between documentation and behavior is a reliability/maintenance issue.

#### Bug Impact

Backend/indexers may record multiple disable events and either: (a) misinterpret which one is authoritative, or (b) need extra code to deduplicate — both add complexity and risk.

Admin is paying gas to call `disable()` again are wasting funds.

Degrades trust in the contract API and NatSpec guarantees.

#### Suggested remediation (short)

Enforce a one-time disable in code and revert on repeated calls, or update NatSpec to match current behavior.

Fix option A — enforce one-time disable (recommended)

Add a boolean guard and revert if already disabled:

bool public disabled;

event WhitelistDisabled(address indexed caller);

```
function disable() external {
    if (disabled) {
        revert WhitelistAlreadyDisabled();
    }
    disabled = true;
    emit WhitelistDisabled(msg.sender);
}
```

#### Fix option B — document current behavior (less desirable)

If repeated disables are intended, update NatSpec to remove the “only once” claim and state that calling disable() is idempotent and may emit multiple events. Prefer adding a guard instead of relying on idempotent events.

Ensure off-chain indexer picks up the first `WhitelistDisabled` event as the canonical transition (documented).

## References

The defining natspec in `IWhitelist.sol`

```
    /// @dev Disables the whitelist of the target whitelisted contract.
    ///
    /// This can only occur once. Once the whitelist is disabled, then it cannot be reenabled.
    function disable() external;
```

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/Whitelist.sol?utm\\_source=immunefi>

### Proof of Concept

### Proof of Concept

Please add this file as `Whitelist.t.sol` to your test/ suit and run the poc using:

```
forge test --match-test test_DisableCanBeCalledTwiceAndEmitsEventEachTime
[⠊] Compiling...
[⠃] Compiling 1 files with Solc 0.8.28
[⠊] Solc 0.8.28 finished in 863.61ms
Compiler run successful!

Ran 1 test for src/test/Whitelist.t.sol:WhitelistTest
[PASS] test_DisableCanBeCalledTwiceAndEmitsEventEachTime() (gas: 43941)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.69ms (1.17ms CPU time)
```

```
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.8.13;

import {Test} from "../../lib/forge-std/src/Test.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {IWhitelist} from "../interfaces/IWhitelist.sol";
import {Unauthorized, IllegalState} from "../base/Errors.sol";

contract WhitelistTest is Test {
    Whitelist public whitelist;
    
    address public owner;
    address public user1 = address(0x1);
    address public user2 = address(0x2);
    address public unauthorized = address(0x999);
    
    event AccountAdded(address account);
    event AccountRemoved(address account);
    event WhitelistDisabled();
    
    function setUp() external {
        owner = address(this);
        whitelist = new Whitelist();
    }
    
    /// @notice Test that demonstrates disable() can be called twice and emits event each time
    /// @dev This reveals a potential issue: disable() doesn't check if already disabled
    function test_DisableCanBeCalledTwiceAndEmitsEventEachTime() external {
        // First call to disable() should emit WhitelistDisabled event
        vm.expectEmit(true, true, true, true);
        emit WhitelistDisabled();
        whitelist.disable();
        
        // Verify whitelist is disabled
        assertTrue(whitelist.disabled(), "Whitelist should be disabled after first call");
        
        // Second call to disable() should ALSO emit WhitelistDisabled event
        // This is the issue: no check to prevent redundant calls
        vm.expectEmit(true, true, true, true);
        emit WhitelistDisabled();
        whitelist.disable();
        
        // Whitelist is still disabled (no change in state)
        assertTrue(whitelist.disabled(), "Whitelist should still be disabled after second call");
        
        // FINDING: disable() can be called multiple times, emitting events each time
        // This wastes gas and pollutes event logs with duplicate events
        // RECOMMENDATION: Add a check like:
        //   if (disabled) revert IllegalState();
        // OR at minimum don't emit the event if already disabled
    }
}
```
