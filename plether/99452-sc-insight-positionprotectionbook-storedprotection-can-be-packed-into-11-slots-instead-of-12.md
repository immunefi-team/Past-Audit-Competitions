# \#99452 [SC-Insight] PositionProtectionBook.StoredProtection can be packed into 11 slots instead of 12

**Submitted on Nov 13th 2026 at 17:52:08 UTC by @Lion44261 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99452
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Description

`PositionProtectionBook.StoredProtection` declares its sub-32-byte fields in four separate groups, so the compiler places them in four slots (0, 1, 9 and 11). Together they take 79 bytes, which fits in three slots. Moving `triggerPublishTime`, `triggeredLeg` and `status` next to the other small fields drops slot 11. The record shrinks from 384 to 352 bytes (12 to 11 slots), and the create path writes one fewer fresh slot.

Two other layouts were checked in the same pass. No change is proposed for either:

- `CfdEngine.lastMarkTime` (`uint64`, slot 21) sits between `uint256 lastMarkPrice` and `uint256[2] sideBorrowBaseAssets`. Neither neighbour can absorb those 8 bytes. `sideCarryTimestamp` (slot 26, 16 bytes used) and `degradedMode` (slot 28, 1 byte used) do have room, but only if `lastMarkTime` is moved across unrelated state. That move was not measured and is not proposed.
- `TrancheVault.maintenanceFeeAprBps`, `maintenanceFeeRecipient` and `maintenanceFeeCheckpointBoundary` occupy 3 slots in any order. Reordering them does not remove a slot.

## Brief/Intro

This is a gas-only Insight. With the packed layout, each protection created through the `_create` path costs 21,890 gas less in a storage-only harness that mirrors the contract's writes (270,025 vs 248,135 gas). The trigger path gets no saving: it measured 48 gas more in the packed layout (57,898 vs 57,946). No funds are affected.

## Vulnerability Details

### 1. `PositionProtectionBook.StoredProtection` (one slot can be saved)

```solidity
// packages/perps/src/PositionProtectionBook.sol:137-154
    struct StoredProtection {
        uint64 protectionId;
        uint64 parentOrderId;
        uint64 linkedOrderId;
        address account;
        CfdTypes.Side side;
        uint256 size;
        uint256 takeProfitTriggerPrice;
        uint256 stopLossTriggerPrice;
        PositionProtectionTypes.CloseFinancialLimits closeLimits;
        uint64 armedAt;
        uint64 armedBlock;
        uint64 armedPublishTime;
        uint256 triggerMarkPrice;
        uint64 triggerPublishTime;
        PositionProtectionTypes.PositionProtectionTriggerLeg triggeredLeg;
        PositionProtectionTypes.PositionProtectionStatus status;
    }
```

The compiler layout (from `forge inspect PositionProtectionBook storageLayout`, full output in the PoC) is 384 bytes, i.e. 12 slots:

| Slot | Fields | Bytes used |
|---|---|---|
| 0 | `protectionId`, `parentOrderId`, `linkedOrderId` | 24 |
| 1 | `account`, `side` | 21 |
| 2–4 | `size`, `takeProfitTriggerPrice`, `stopLossTriggerPrice` | 32 each |
| 5–8 | `closeLimits` (4 × `uint256`) | 32 each |
| 9 | `armedAt`, `armedBlock`, `armedPublishTime` | 24 |
| 10 | `triggerMarkPrice` | 32 |
| 11 | `triggerPublishTime`, `triggeredLeg`, `status` | 10 |

The small fields are seven `uint64` (56 bytes), one `address` (20 bytes) and three one-byte enums (3 bytes), 79 bytes in total, spread over four slots. Moving `triggerPublishTime` into slot 0 (24 → 32 bytes) and `triggeredLeg` and `status` into slot 1 (21 → 23 bytes) leaves slot 11 empty. Compiled, the packed struct is 352 bytes, i.e. 11 slots (PoC). Every other field keeps its slot.

Why this changes gas on creation: `_create` followed by `_arm` writes `status` together with fields in slots 0, 1 and 9.

```solidity
// packages/perps/src/PositionProtectionBook.sol:233-246
        protectionId = nextPositionProtectionId++;
        StoredProtection storage protection = _protections[protectionId];
        protection.protectionId = protectionId;
        protection.account = account;
        protection.side = position.side;
        protection.size = position.size;
        protection.takeProfitTriggerPrice = params.takeProfitTriggerPrice;
        protection.stopLossTriggerPrice = params.stopLossTriggerPrice;
        _setCloseLimits(protection, params.closeLimits);
        _clearinghouse()
            .recordBountyReservation(
                account, IMarginClearinghouse.BountyKind.ProtectionTrigger, protectionId, triggerBountyAssets
            );
        _arm(protection, currentPublishTime);
```

```solidity
// packages/perps/src/PositionProtectionBook.sol:1059-1067
    function _arm(
        StoredProtection storage protection,
        uint64 publishTime
    ) private {
        protection.armedAt = uint64(block.timestamp);
        protection.armedBlock = ExecutionBlockNumberLib.current();
        protection.armedPublishTime = publishTime;
        protection.status = PositionProtectionTypes.PositionProtectionStatus.Armed;
    }
```

In the current layout, `status` is the only field written to slot 11 on creation, so creation pays for a fresh zero-to-nonzero slot just for one byte. In the packed layout, `status` shares slot 1 with `account`, which the same call already wrote.

The trigger path (`_recordActivation` then `_recordCloseAttempt`) writes `triggerMarkPrice`, `triggerPublishTime`, `triggeredLeg`, `linkedOrderId` and `status`:

```solidity
// packages/perps/src/PositionProtectionBook.sol:557-559
        protection.triggerMarkPrice = markPrice;
        protection.triggerPublishTime = publishTime;
        protection.triggeredLeg = leg;
```

```solidity
// packages/perps/src/PositionProtectionBook.sol:571-573
        uint64 previousLinkedOrderId = protection.linkedOrderId;
        protection.linkedOrderId = linkedOrderId;
        protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
```

Today that touches slots 10, 11 and 0. In the packed layout it touches slots 10, 0 and 1, the same count, so the trigger path gains nothing. The harness measured it 48 gas more expensive.

Nothing reads the struct by raw slot offset (the only `assembly` block in the file bubbles revert data), there is no `StoredProtection(...)` positional constructor, and the contract is not behind a proxy. Reordering the fields is a pure layout change.

### 2. `CfdEngine.lastMarkTime` (adjacent fields cannot absorb it; no move is proposed)

```solidity
// packages/perps/src/CfdEngine.sol:106-112
    SideState[2] public sides;
    /// @notice Most recently accepted cached mark price, with 8 decimals and bounded by `CAP_PRICE` on router paths.
    uint256 public lastMarkPrice;
    /// @notice Oracle publish timestamp associated with `lastMarkPrice`, in Unix seconds.
    uint64 public lastMarkTime;
    /// @notice Aggregate carry borrow base by side index (`0` LONG; `1` SHORT), in 6-decimal settlement token units.
    uint256[2] public sideBorrowBaseAssets;
```

Compiler layout: `lastMarkPrice` occupies all of slot 20, `lastMarkTime` sits alone in slot 21 (8 of 32 bytes), and `sideBorrowBaseAssets` starts at slot 22. A static array always starts a new slot and fills whole slots, so neither neighbour can take the 8 bytes. `admin` (slot 9) has 12 free bytes, but `terminalNavBook`, `_terminalCurveMutationDepth`, `sides` and `lastMarkPrice` lie between them, so packing with it would mean moving the declaration across that state. Narrowing `lastMarkPrice` is not proposed.

Other slots have spare room: the address slots 6–10 and 43, `sideCarryTimestamp` in slot 26 (16 of 32 bytes used) and `degradedMode` alone in slot 28. Any of those can hold a `uint64` only if `lastMarkTime` is moved across `sideBorrowBaseAssets` and `sideCarryIndex`. No such move was measured. This report does not propose one and claims no saving for `CfdEngine`.

### 3. `TrancheVault` maintenance-fee fields (checked, cannot drop a slot)

```solidity
// packages/perps/src/TrancheVault.sol:138-145
    address public seedReceiver;
    uint256 public seedShareFloor;

    uint256 public maintenanceFeeAprBps;
    address public maintenanceFeeRecipient;
    uint256 public maintenanceFeeCheckpointBoundary;
    MaintenanceFeeConfig public pendingMaintenanceFeeConfig;
    uint256 public maintenanceFeeConfigActivationTime;
```

Compiler layout: `maintenanceFeeAprBps` is slot 29, `maintenanceFeeRecipient` slot 30 (20 bytes) and `maintenanceFeeCheckpointBoundary` slot 31. Each `uint256` fills a slot, and a 20-byte address cannot share a slot with a `uint256`, so every ordering of these three uses 3 slots. The neighbours do not help:

- `seedShareFloor` (slot 28) is a `uint256`.
- `pendingMaintenanceFeeConfig` (slots 32–33) is a struct, and a struct always starts a new slot.
- `seedReceiver` (slot 27) has only 12 free bytes, which is too small for another address.

Checked: reordering cannot drop a slot, and this report does not claim a saved SLOAD on the fee path.

## Impact Details

Gas only. No funds are lost, frozen or redirected.

- `StoredProtection`: in a storage-only harness that copies the field order and the write sequence of `_create` + `_arm`, creating one record costs 270,025 gas in the current layout and 248,135 gas in the packed layout, a difference of 21,890 gas per record. The trigger write sequence costs 57,898 vs 57,946 gas, 48 gas more in the packed layout. These are harness numbers. The real contract was not recompiled with the reordered struct, because the in-scope sources were left unmodified.
- The extra slot is a fixed per-record cost. It does not grow with the number of other protection records. In the existing test `test_F3_protectionTriggerCostIndependentOfOtherRecords`, triggering the same record with 0 and with 20 other accounts' records present costs 1,105,400 and 1,105,446 gas, with the same 32 SLOAD / 6 SSTORE accesses on the book. That 46-gas difference is unrelated to packing and is not a packing saving.
- `CfdEngine.lastMarkTime`: the two neighbouring slots cannot absorb it. A non-adjacent move into `sideCarryTimestamp` or `degradedMode` was not measured and is not claimed. The `TrancheVault` fee fields stay 3 slots in every order, so no SLOAD saving is claimed there.

## Mitigation

Only for `PositionProtectionBook.StoredProtection`. Move the three small trailing fields next to the existing small fields:

```diff
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -137,18 +137,18 @@
     struct StoredProtection {
         uint64 protectionId;
         uint64 parentOrderId;
         uint64 linkedOrderId;
+        uint64 triggerPublishTime;
         address account;
         CfdTypes.Side side;
+        PositionProtectionTypes.PositionProtectionTriggerLeg triggeredLeg;
+        PositionProtectionTypes.PositionProtectionStatus status;
         uint256 size;
         uint256 takeProfitTriggerPrice;
         uint256 stopLossTriggerPrice;
         PositionProtectionTypes.CloseFinancialLimits closeLimits;
         uint64 armedAt;
         uint64 armedBlock;
         uint64 armedPublishTime;
         uint256 triggerMarkPrice;
-        uint64 triggerPublishTime;
-        PositionProtectionTypes.PositionProtectionTriggerLeg triggeredLeg;
-        PositionProtectionTypes.PositionProtectionStatus status;
     }
```

No change is proposed for `CfdEngine` or `TrancheVault`.

## References

- `StoredProtection` definition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L137-L154
- `_create` record writes: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L233-L246
- `_arm`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L1059-L1067
- `_recordActivation` and `_recordCloseAttempt`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L547-L578
- `CfdEngine` mark and carry state: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L106-L121
- `TrancheVault` seed and maintenance-fee state: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L138-L145


## Proof of Concept

## Proof of Concept

Environment: forge 1.9.0, solc 0.8.35, `via_ir = true`, optimizer 200 runs. The harness imports only `forge-std`, not any protocol contract. Under `via_ir`, time is read with `vm.getBlockTimestamp()`.

### Step 1: compiler layout of the real contracts

Commands (run in the Foundry project that compiles the in-scope sources):

```bash
forge inspect --out out-submit063 --cache-path cache-submit063 PositionProtectionBook storageLayout --json \
  | jq -c '.types | to_entries[] | select(.key | startswith("t_struct(StoredProtection")) | .value | {numberOfBytes}, (.members[] | {label, slot, offset})'
```

```
{"numberOfBytes":"384"}
{"label":"protectionId","slot":"0","offset":0}
{"label":"parentOrderId","slot":"0","offset":8}
{"label":"linkedOrderId","slot":"0","offset":16}
{"label":"account","slot":"1","offset":0}
{"label":"side","slot":"1","offset":20}
{"label":"size","slot":"2","offset":0}
{"label":"takeProfitTriggerPrice","slot":"3","offset":0}
{"label":"stopLossTriggerPrice","slot":"4","offset":0}
{"label":"closeLimits","slot":"5","offset":0}
{"label":"armedAt","slot":"9","offset":0}
{"label":"armedBlock","slot":"9","offset":8}
{"label":"armedPublishTime","slot":"9","offset":16}
{"label":"triggerMarkPrice","slot":"10","offset":0}
{"label":"triggerPublishTime","slot":"11","offset":0}
{"label":"triggeredLeg","slot":"11","offset":8}
{"label":"status","slot":"11","offset":9}
```

```bash
forge inspect --out out-submit063 --cache-path cache-submit063 CfdEngine storageLayout --json \
  | jq -c '. as $l | .storage[] | select(.slot | tonumber | . >= 6 and . <= 29) | {label, slot, offset, bytes: $l.types[.type].numberOfBytes}'
```

```
{"label":"pool","slot":"6","offset":0,"bytes":"20"}
{"label":"planner","slot":"7","offset":0,"bytes":"20"}
{"label":"settlementSidecar","slot":"8","offset":0,"bytes":"20"}
{"label":"admin","slot":"9","offset":0,"bytes":"20"}
{"label":"terminalNavBook","slot":"10","offset":0,"bytes":"20"}
{"label":"_terminalCurveMutationDepth","slot":"11","offset":0,"bytes":"32"}
{"label":"sides","slot":"12","offset":0,"bytes":"256"}
{"label":"lastMarkPrice","slot":"20","offset":0,"bytes":"32"}
{"label":"lastMarkTime","slot":"21","offset":0,"bytes":"8"}
{"label":"sideBorrowBaseAssets","slot":"22","offset":0,"bytes":"64"}
{"label":"sideCarryIndex","slot":"24","offset":0,"bytes":"64"}
{"label":"sideCarryTimestamp","slot":"26","offset":0,"bytes":"32"}
{"label":"unsettledCarryAssets","slot":"27","offset":0,"bytes":"32"}
{"label":"degradedMode","slot":"28","offset":0,"bytes":"1"}
{"label":"_activeRiskParams","slot":"29","offset":0,"bytes":"320"}
```

```bash
forge inspect --out out-submit063 --cache-path cache-submit063 TrancheVault storageLayout --json \
  | jq -c '. as $l | .storage[] | select(.slot | tonumber | . >= 26 and . <= 34) | {label, slot, offset, bytes: $l.types[.type].numberOfBytes}'
```

```
{"label":"withdrawalEscrowAssets","slot":"26","offset":0,"bytes":"32"}
{"label":"seedReceiver","slot":"27","offset":0,"bytes":"20"}
{"label":"seedShareFloor","slot":"28","offset":0,"bytes":"32"}
{"label":"maintenanceFeeAprBps","slot":"29","offset":0,"bytes":"32"}
{"label":"maintenanceFeeRecipient","slot":"30","offset":0,"bytes":"20"}
{"label":"maintenanceFeeCheckpointBoundary","slot":"31","offset":0,"bytes":"32"}
{"label":"pendingMaintenanceFeeConfig","slot":"32","offset":0,"bytes":"64"}
{"label":"maintenanceFeeConfigActivationTime","slot":"34","offset":0,"bytes":"32"}
```

### Step 2: harness with the current and the packed field order

`test/InsightStoragePacking.t.sol`. `CurrentProtectionLayout` copies the field order of `StoredProtection` at the frozen commit; `PackedProtectionLayout` applies the Mitigation diff. Both `create` functions perform the field writes of `_create` + `_arm` in the same order; both `trigger` functions perform the field writes of `_recordActivation` + `_recordCloseAttempt`. The enums copy the member lists of `CfdTypes.Side`, `PositionProtectionTriggerLeg` and `PositionProtectionStatus`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

enum Side {
    LONG,
    SHORT
}

enum TriggerLeg {
    None,
    TakeProfit,
    StopLoss
}

enum Status {
    None,
    PendingOpen,
    Armed,
    Triggered,
    Executed,
    Failed,
    Cancelled,
    Liquidated,
    Latched
}

struct CloseFinancialLimits {
    uint256 maxExecutionNotionalAssets;
    uint256 maxGrossAccountDebitAssets;
    uint256 maxActionChargeAssets;
    uint256 maxExplicitFeesAssets;
}

/// Same field order as PositionProtectionBook.StoredProtection at a82d4fe.
contract CurrentProtectionLayout {

    struct StoredProtection {
        uint64 protectionId;
        uint64 parentOrderId;
        uint64 linkedOrderId;
        address account;
        Side side;
        uint256 size;
        uint256 takeProfitTriggerPrice;
        uint256 stopLossTriggerPrice;
        CloseFinancialLimits closeLimits;
        uint64 armedAt;
        uint64 armedBlock;
        uint64 armedPublishTime;
        uint256 triggerMarkPrice;
        uint64 triggerPublishTime;
        TriggerLeg triggeredLeg;
        Status status;
    }

    mapping(uint64 => StoredProtection) internal _protections;

    /// Field writes of `_create` followed by `_arm`.
    function create(
        uint64 id,
        address account,
        uint256 size,
        uint256 tp,
        uint256 sl,
        CloseFinancialLimits calldata limits,
        uint64 publishTime
    ) external {
        StoredProtection storage p = _protections[id];
        p.protectionId = id;
        p.account = account;
        p.side = Side.SHORT;
        p.size = size;
        p.takeProfitTriggerPrice = tp;
        p.stopLossTriggerPrice = sl;
        p.closeLimits = limits;
        p.armedAt = uint64(block.timestamp);
        p.armedBlock = uint64(block.number);
        p.armedPublishTime = publishTime;
        p.status = Status.Armed;
    }

    /// Field writes of `_recordActivation` followed by `_recordCloseAttempt`.
    function trigger(
        uint64 id,
        uint256 markPrice,
        uint64 publishTime,
        uint64 linkedOrderId
    ) external {
        StoredProtection storage p = _protections[id];
        p.triggerMarkPrice = markPrice;
        p.triggerPublishTime = publishTime;
        p.triggeredLeg = TriggerLeg.TakeProfit;
        p.linkedOrderId = linkedOrderId;
        p.status = Status.Triggered;
    }

}

/// Identical fields; `triggerPublishTime`, `triggeredLeg` and `status` moved next to the other small fields.
contract PackedProtectionLayout {

    struct StoredProtection {
        uint64 protectionId;
        uint64 parentOrderId;
        uint64 linkedOrderId;
        uint64 triggerPublishTime;
        address account;
        Side side;
        TriggerLeg triggeredLeg;
        Status status;
        uint256 size;
        uint256 takeProfitTriggerPrice;
        uint256 stopLossTriggerPrice;
        CloseFinancialLimits closeLimits;
        uint64 armedAt;
        uint64 armedBlock;
        uint64 armedPublishTime;
        uint256 triggerMarkPrice;
    }

    mapping(uint64 => StoredProtection) internal _protections;

    function create(
        uint64 id,
        address account,
        uint256 size,
        uint256 tp,
        uint256 sl,
        CloseFinancialLimits calldata limits,
        uint64 publishTime
    ) external {
        StoredProtection storage p = _protections[id];
        p.protectionId = id;
        p.account = account;
        p.side = Side.SHORT;
        p.size = size;
        p.takeProfitTriggerPrice = tp;
        p.stopLossTriggerPrice = sl;
        p.closeLimits = limits;
        p.armedAt = uint64(block.timestamp);
        p.armedBlock = uint64(block.number);
        p.armedPublishTime = publishTime;
        p.status = Status.Armed;
    }

    function trigger(
        uint64 id,
        uint256 markPrice,
        uint64 publishTime,
        uint64 linkedOrderId
    ) external {
        StoredProtection storage p = _protections[id];
        p.triggerMarkPrice = markPrice;
        p.triggerPublishTime = publishTime;
        p.triggeredLeg = TriggerLeg.TakeProfit;
        p.linkedOrderId = linkedOrderId;
        p.status = Status.Triggered;
    }

}

interface IProtectionLayout {

    function create(
        uint64 id,
        address account,
        uint256 size,
        uint256 tp,
        uint256 sl,
        CloseFinancialLimits calldata limits,
        uint64 publishTime
    ) external;

    function trigger(
        uint64 id,
        uint256 markPrice,
        uint64 publishTime,
        uint64 linkedOrderId
    ) external;

}

contract InsightStoragePackingTest is Test {

    uint64 internal constant ID = 7;

    /// `_protections` is the only state variable in both layouts, so it sits at slot 0.
    function _recordBase() internal pure returns (uint256) {
        return uint256(keccak256(abi.encode(uint256(ID), uint256(0))));
    }

    function _run(
        address target
    ) internal returns (uint256 createGas, uint256 triggerGas, uint256 usedSlots) {
        vm.warp(1_800_000_000);
        vm.roll(100);
        CloseFinancialLimits memory limits = CloseFinancialLimits(1e30, 2e30, 3e30, 4e30);
        uint64 publishTime = uint64(vm.getBlockTimestamp()) - 1;

        vm.cool(target);
        uint256 g = gasleft();
        IProtectionLayout(target).create(ID, address(0xCA01), 10_000e18, 0.9e8, 1.1e8, limits, publishTime);
        createGas = g - gasleft();

        vm.warp(vm.getBlockTimestamp() + 12);
        vm.roll(101);
        vm.cool(target);
        g = gasleft();
        IProtectionLayout(target).trigger(ID, 0.9e8, uint64(vm.getBlockTimestamp()), 42);
        triggerGas = g - gasleft();

        uint256 base = _recordBase();
        for (uint256 i; i < 12; ++i) {
            if (vm.load(target, bytes32(base + i)) != bytes32(0)) {
                usedSlots = i + 1;
            }
        }
    }

    function test_storedProtectionPackingSavesOneSlot() public {
        address current = address(new CurrentProtectionLayout());
        address packed = address(new PackedProtectionLayout());

        (uint256 createCurrent, uint256 triggerCurrent, uint256 slotsCurrent) = _run(current);
        (uint256 createPacked, uint256 triggerPacked, uint256 slotsPacked) = _run(packed);

        console2.log("create write gas current / packed", createCurrent, createPacked);
        console2.log("trigger write gas current / packed", triggerCurrent, triggerPacked);
        console2.log("highest non-zero record slot + 1, current / packed", slotsCurrent, slotsPacked);

        assertEq(slotsCurrent, 12, "current layout spans 12 slots");
        assertEq(slotsPacked, 11, "packed layout spans 11 slots");
        assertLt(createPacked, createCurrent, "create writes one fewer fresh slot");
    }

}
```

### Step 3: harness struct layouts

```bash
forge inspect --out out-submit063 --cache-path cache-submit063 CurrentProtectionLayout storageLayout --json \
  | jq -c '.types | to_entries[] | select(.key | startswith("t_struct(StoredProtection")) | .value | {numberOfBytes}, (.members[] | {label, slot, offset})'
```

```
{"numberOfBytes":"384"}
{"label":"protectionId","slot":"0","offset":0}
{"label":"parentOrderId","slot":"0","offset":8}
{"label":"linkedOrderId","slot":"0","offset":16}
{"label":"account","slot":"1","offset":0}
{"label":"side","slot":"1","offset":20}
{"label":"size","slot":"2","offset":0}
{"label":"takeProfitTriggerPrice","slot":"3","offset":0}
{"label":"stopLossTriggerPrice","slot":"4","offset":0}
{"label":"closeLimits","slot":"5","offset":0}
{"label":"armedAt","slot":"9","offset":0}
{"label":"armedBlock","slot":"9","offset":8}
{"label":"armedPublishTime","slot":"9","offset":16}
{"label":"triggerMarkPrice","slot":"10","offset":0}
{"label":"triggerPublishTime","slot":"11","offset":0}
{"label":"triggeredLeg","slot":"11","offset":8}
{"label":"status","slot":"11","offset":9}
```

This is identical, slot for slot and offset for offset, to the real `PositionProtectionBook.StoredProtection` in Step 1.

```bash
forge inspect --out out-submit063 --cache-path cache-submit063 PackedProtectionLayout storageLayout --json \
  | jq -c '.types | to_entries[] | select(.key | startswith("t_struct(StoredProtection")) | .value | {numberOfBytes}, (.members[] | {label, slot, offset})'
```

```
{"numberOfBytes":"352"}
{"label":"protectionId","slot":"0","offset":0}
{"label":"parentOrderId","slot":"0","offset":8}
{"label":"linkedOrderId","slot":"0","offset":16}
{"label":"triggerPublishTime","slot":"0","offset":24}
{"label":"account","slot":"1","offset":0}
{"label":"side","slot":"1","offset":20}
{"label":"triggeredLeg","slot":"1","offset":21}
{"label":"status","slot":"1","offset":22}
{"label":"size","slot":"2","offset":0}
{"label":"takeProfitTriggerPrice","slot":"3","offset":0}
{"label":"stopLossTriggerPrice","slot":"4","offset":0}
{"label":"closeLimits","slot":"5","offset":0}
{"label":"armedAt","slot":"9","offset":0}
{"label":"armedBlock","slot":"9","offset":8}
{"label":"armedPublishTime","slot":"9","offset":16}
{"label":"triggerMarkPrice","slot":"10","offset":0}
```

### Step 4: run the harness

```bash
forge test --out out-submit063 --cache-path cache-submit063 --match-path test/InsightStoragePacking.t.sol -vv
```

Output (forge-std compiler warnings omitted):

```
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 1.35s
Compiler run successful with warnings:

Ran 1 test for test/InsightStoragePacking.t.sol:InsightStoragePackingTest
[PASS] test_storedProtectionPackingSavesOneSlot() (gas: 1116415)
Logs:
  create write gas current / packed 270025 248135
  trigger write gas current / packed 57898 57946
  highest non-zero record slot + 1, current / packed 12 11

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 673.04µs (504.16µs CPU time)

Ran 1 test suite in 11.25ms (673.04µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Reading the output:

- Slot count, from `forge inspect`: 12 slots (384 bytes) current, 11 slots (352 bytes) packed. The `vm.load` scan agrees: after create and trigger, the highest non-zero slot of the record is slot 11 in the current layout and slot 10 in the packed one.
- Create write (`_create` + `_arm` field sequence): 270,025 vs 248,135 gas, so packing saves 21,890 gas per record.
- Trigger write (`_recordActivation` + `_recordCloseAttempt` field sequence): 57,898 vs 57,946 gas, so packing saves nothing on this path (+48 gas).

