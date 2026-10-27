# 88963 sc insight capacityconfig field order wastes a full storage slot

**Submitted on Aug 16th 2026 at 01:14:54 UTC by @Web3Vinay for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88963
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/ICoverOrderAllocator.sol>
* **Impacts:**
  * Theft of gas
  * Unbounded gas consumption

## Description

## Brief/Intro

`CapacityConfig` declares `uint16 divergenceToleranceBps` as its last field, after three `uint256`s, so it starts a 6th storage slot of its own instead of packing into the 12 spare bytes already sitting next to `firstLossBufferToken` (20 bytes) in slot 1. Moving it next to that address drops the struct from 6 slots to 5 with no other change.

## Vulnerability Details

`contracts/core/interfaces/ICoverOrderAllocator.sol:176-191`:

```solidity
struct CapacityConfig {
    uint256 minCAR;                     // slot 0
    IERC20 firstLossBufferToken;        // slot 1  (20 bytes used, 12 free)
    address firstLossBuffer;            // slot 2
    uint256 effectiveLeverage;          // slot 3
    uint256 minOrderMarketCoverAmount;  // slot 4
@>  uint16 divergenceToleranceBps;      // slot 5  (2 bytes used, 30 free)
}
```

Written via `capacityConfigHistory.push(config)` (`CoverOrderAllocator.sol:673-674`, 6 cold SSTOREs per config) and read on the hot commit path (`_computeAvailableCapacity`, called from `commitAllocation`/`recommitAllocation`) and cross-contract on the payout path (`IncidentManager.sol:636-638` via `getCapacityConfigAt`).

Measured with two harnesses holding the identical fields in the two orderings (PoC below):

|                                                          | current (6 slots) | packed (5 slots) |         saving |
| -------------------------------------------------------- | ----------------: | ---------------: | -------------: |
| append one config                                        |           137,142 |          115,352 | **21,790 gas** |
| cold read of the fields `_computeAvailableCapacity` uses |            17,971 |           15,856 |  **2,115 gas** |

The reordering is safe for `CoverOrderAllocator`: `CapacityConfig` isn't pinned to a fixed proxy slot, it lives inside the ERC-7201 namespaced storage as `CapacityConfig[] capacityConfigHistory`, so changing the element layout only changes the dynamic array's contents. Still upgrade-breaking for an already-deployed allocator — belongs in a fresh deployment, not a live upgrade.

## Impact Details

No funds at risk, no functional defect — pure gas. `setCapacityConfig`/`initialize` pay 21,790 gas more than necessary per configuration; `commitAllocation`/`recommitAllocation` (at least once per period, and `recommitAllocation` is the documented emergency root-withdrawal path, where cheap-and-likely-to-land matters) and `IncidentManager.approveCurrentAssessment` each pay the extra 2,115-gas cold read.

## Mitigation

```diff
 struct CapacityConfig {
     uint256 minCAR;
     IERC20 firstLossBufferToken;
+    uint16 divergenceToleranceBps;
     address firstLossBuffer;
     uint256 effectiveLeverage;
     uint256 minOrderMarketCoverAmount;
-    uint16 divergenceToleranceBps;
 }
```

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/ICoverOrderAllocator.sol#L176-L191>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L673-L674>

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L702-L724>

## Proof of Concept

Two harnesses, identical fields, one reordering. Save as `test/CapacityConfigPacking.t.sol` and run `forge test --match-contract CapacityConfigPacking -vv`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// Field order verbatim from ICoverOrderAllocator.CapacityConfig -- 6 slots.
contract CurrentLayout {
    struct CapacityConfig {
        uint256 minCAR;
        IERC20 firstLossBufferToken;
        address firstLossBuffer;
        uint256 effectiveLeverage;
        uint256 minOrderMarketCoverAmount;
        uint16 divergenceToleranceBps;
    }

    CapacityConfig[] public history;

    function push(CapacityConfig calldata c) external {
        history.push(c);
    }

    function read(uint256 i) external view returns (uint256 acc) {
        CapacityConfig storage c = history[i];
        acc = uint256(uint160(address(c.firstLossBufferToken)));
        acc ^= uint256(uint160(c.firstLossBuffer));
        acc ^= c.effectiveLeverage;
        acc ^= c.minCAR;
        acc ^= uint256(c.divergenceToleranceBps);
    }
}

/// Same fields, divergenceToleranceBps moved next to firstLossBufferToken (20+2<=32) -- 5 slots.
contract PackedLayout {
    struct CapacityConfig {
        uint256 minCAR;
        IERC20 firstLossBufferToken;
        uint16 divergenceToleranceBps;
        address firstLossBuffer;
        uint256 effectiveLeverage;
        uint256 minOrderMarketCoverAmount;
    }

    CapacityConfig[] public history;

    function push(CapacityConfig calldata c) external {
        history.push(c);
    }

    function read(uint256 i) external view returns (uint256 acc) {
        CapacityConfig storage c = history[i];
        acc = uint256(uint160(address(c.firstLossBufferToken)));
        acc ^= uint256(uint160(c.firstLossBuffer));
        acc ^= c.effectiveLeverage;
        acc ^= c.minCAR;
        acc ^= uint256(c.divergenceToleranceBps);
    }
}

contract CapacityConfigPacking is Test {
    CurrentLayout cur;
    PackedLayout pkd;

    address constant TOKEN = address(0xBEEF);
    address constant WALLET = address(0xB0FFE7);

    function setUp() public {
        cur = new CurrentLayout();
        pkd = new PackedLayout();
        // Seed element 0 so the read benchmark below hits COLD slots, matching
        // production (_computeAvailableCapacity reads a config written earlier).
        cur.push(CurrentLayout.CapacityConfig(12_000, IERC20(TOKEN), WALLET, 12_000, 1, 7));
        pkd.push(PackedLayout.CapacityConfig(12_000, IERC20(TOKEN), 7, WALLET, 12_000, 1));
    }

    function test_gasDelta() public {
        uint256 g;

        g = gasleft();
        cur.read(0);
        uint256 readCur = g - gasleft();

        g = gasleft();
        pkd.read(0);
        uint256 readPkd = g - gasleft();

        g = gasleft();
        cur.push(CurrentLayout.CapacityConfig(12_000, IERC20(TOKEN), WALLET, 12_000, 1, 7));
        uint256 pushCur = g - gasleft();

        g = gasleft();
        pkd.push(PackedLayout.CapacityConfig(12_000, IERC20(TOKEN), 7, WALLET, 12_000, 1));
        uint256 pushPkd = g - gasleft();

        emit log_named_uint("push saving", pushCur - pushPkd);
        emit log_named_uint("read saving", readCur - readPkd);

        assertGt(pushCur, pushPkd);
        assertGt(readCur, readPkd);
    }
}
```

### Result

```
Ran 1 test for test/CapacityConfigPacking.t.sol:CapacityConfigPacking
[PASS] test_gasDelta() (gas: 290701)
Logs:
  push saving: 21790
  read saving: 2115

Suite result: ok. 1 passed; 0 failed; 0 skipped
```
