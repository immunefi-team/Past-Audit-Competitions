# \#98898 [SC-Insight] Terminal receipts stamp the L1 ancestor block on Arbitrum, diverging from the L2 commit block

**Submitted on Nov 10th 2026 at 19:50:44 UTC by @Bozwkd for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98898
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Summary

The protocol deliberately abstracts block numbers through `ExecutionBlockNumberLib.current()` because, on Arbitrum, Solidity's `block.number` returns the **L1 ancestor** block (shared by many L2 blocks) rather than the L2 block. Every MEV-sensitive lifecycle stamp — the order `commitBlock`, the protection `armedBlock`, and the same-block execution shield — uses this library and therefore the **L2** block number (`ArbSys.arbBlockNumber()`).

`OrderLifecycleBook`, however, stamps each terminal order receipt's `terminalBlock` with **raw `block.number`**. On Arbitrum One this is the L1 ancestor block. As a result the two block fields that describe a single order's lifetime — `commitBlock` and `terminalBlock` — are recorded on **two different chains' clocks**. Any consumer of the signed V5 receipt that reasons about block-based timing (`terminalBlock − commitBlock`, finality windows, dispute logic) receives nonsensical values, typically an underflow because the L1 block number is far smaller than the L2 block number.

No protocol logic compares the two fields, and the receipt hash is internally consistent, so there is no fund-loss or replay vector. The impact is confined to correctness of consumer-facing receipt evidence, which is why this is reported as Low.

# Root Cause

The team created `ExecutionBlockNumberLib` precisely to avoid the Arbitrum `block.number` footgun, and documents the intent (`ExecutionBlockNumberLib.sol:10-13`):

```solidity
/// @notice Returns the block number of the chain executing the transaction.
/// @dev On Arbitrum, Solidity's block.number is an ancestor L1 block number and can be shared by many L2 blocks.
///      The ArbSys precompile returns the L2 block number. An unavailable precompile on a listed Arbitrum chain
///      reverts instead of silently weakening same-block protections.
library ExecutionBlockNumberLib {
    address private constant ARB_SYS = address(100);
    function current() internal view returns (uint64) {
        if (block.chainid == 42_161 || block.chainid == 42_170 || block.chainid == 421_614) {
            return uint64(IArbSysBlockNumber(ARB_SYS).arbBlockNumber());
        }
        return uint64(block.number);
    }
}
```

Order commit and protection arming use this L2-aware source:

- `OrderRouter.sol:171` — `commitBlock: ExecutionBlockNumberLib.current()`
- `OrderRouter.sol:466` — protection-trigger close `commitBlock: ExecutionBlockNumberLib.current()`
- `PositionProtectionBook.sol:1064` — `protection.armedBlock = ExecutionBlockNumberLib.current()`
- `OrderRouterExecutionSidecar.sol:447` — same-block shield compares `ExecutionBlockNumberLib.current() == order.commitBlock`

But the terminal receipt bypasses the library and reads raw `block.number` (`OrderLifecycleBook.sol:434-441`):

```solidity
if (block.number > type(uint64).max || block.timestamp > type(uint64).max) {
    revert OrderLifecycleBook__TerminalClockOverflow();
}

uint64 terminalBlock = uint64(block.number);          // ← L1 ancestor block on Arbitrum
uint64 terminalTime = uint64(block.timestamp);
receiptHash = keccak256(
    abi.encode(RECEIPT_TYPEHASH, block.chainid, address(this), ROUTER, terminalBlock, terminalTime, receipt)
);
```

`terminalBlock` is then persisted into the order's `TerminalOutcome` and surfaced to receipt consumers (`OrderLifecycleBook.sol:446,456`; receipt type `PletherOrderReceiptV5`). On Arbitrum One, `commitBlock` is on the L2 clock while `terminalBlock` is on the L1 clock — a direct contradiction of the library's stated purpose of not "silently weakening" block-based accounting.

# Attack Path

This is a deterministic correctness defect rather than an attacker-driven exploit; the "path" is the normal order lifecycle on Arbitrum One:

1. A trader commits an order. The router records `commitBlock = ExecutionBlockNumberLib.current()` → the **L2** block number (e.g. `150_000_000`).
2. A keeper executes or the order otherwise terminalizes. `OrderLifecycleBook` records `terminalBlock = uint64(block.number)` → the **L1 ancestor** block number (e.g. `21_000_000`).
3. The finalized `PletherOrderReceiptV5` now contains `commitBlock = 150_000_000` and `terminalBlock = 21_000_000` for the same order.
4. A receipt consumer (indexer, monitoring system, dispute/finality tool, or integrator ABI documented in the release) computes `terminalBlock − commitBlock` to derive elapsed execution blocks and receives an underflow / garbage value, or concludes the order "terminalized before it was committed."

# Impact

- **On-chain evidence is internally inconsistent on the target chain.** The two block fields of a single receipt use different clocks, so any block-delta or ordering inference over a receipt is wrong on Arbitrum One.
- **Silent, not fail-safe.** The library was explicitly designed to revert rather than weaken block semantics; this path silently uses the weak source, defeating that guarantee for the receipt.

 # Mitigation

Stamp `terminalBlock` through the same abstraction every other lifecycle block field uses, so the whole receipt is on one clock:

```solidity
// OrderLifecycleBook.sol
uint64 terminalBlock = ExecutionBlockNumberLib.current();
```

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {ExecutionBlockNumberLib} from "@plether/perps/libraries/ExecutionBlockNumberLib.sol";

/// @dev Minimal ArbSys precompile stub: returns a constant L2 block number.
contract ArbSysMock {
    function arbBlockNumber() external pure returns (uint256) {
        return 150_000_000;
    }
}

/// @dev Exposes the two block-stamp sources used by the protocol.
contract BlockClockHarness {
    /// Mirrors OrderRouter.commitBlock / PositionProtectionBook.armedBlock (L2-aware).
    function commitStamp() external view returns (uint64) {
        return ExecutionBlockNumberLib.current();
    }

    /// Mirrors OrderLifecycleBook terminalBlock (raw block.number).
    function terminalStamp() external view returns (uint64) {
        return uint64(block.number);
    }
}

contract TerminalBlockClockMismatchTest is Test {
    uint256 private constant ARBITRUM_ONE = 42_161;
    address private constant ARB_SYS = address(100);

    function test_commitAndTerminalBlocksUseDifferentClocksOnArbitrum() public {
        // Arbitrum One: raw block.number is the slow L1 ancestor; ArbSys returns the fast L2 block.
        vm.chainId(ARBITRUM_ONE);
        vm.roll(21_000_000); // L1 ancestor block.number
        vm.etch(ARB_SYS, address(new ArbSysMock()).code); // L2 arbBlockNumber() == 150_000_000

        BlockClockHarness h = new BlockClockHarness();

        uint64 commitBlock = h.commitStamp(); // L2 clock
        uint64 terminalBlock = h.terminalStamp(); // L1 clock

        assertEq(commitBlock, uint64(150_000_000), "commit stamp uses L2 arbBlockNumber");
        assertEq(terminalBlock, uint64(21_000_000), "terminal stamp uses L1 block.number");
        assertTrue(commitBlock != terminalBlock, "same-order block fields are on different clocks");

        // A receipt consumer computing elapsed blocks sees terminal BEFORE commit -> underflow/garbage.
        assertLt(terminalBlock, commitBlock, "terminalBlock appears earlier than commitBlock on Arbitrum");
    }
}
```
