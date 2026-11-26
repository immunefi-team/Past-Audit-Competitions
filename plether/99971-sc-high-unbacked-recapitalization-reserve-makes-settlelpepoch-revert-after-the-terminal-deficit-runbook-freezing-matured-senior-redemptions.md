# \#99971 [SC-High] Unbacked Recapitalization Reserve Makes `settleLpEpoch` Revert After the Terminal Deficit Runbook, Freezing Matured Senior Redemptions

**Submitted on Nov 15th 2026 at 23:27:00 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99971
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

After a terminal deficit is handled with the documented runbook (`recapitalizePool`, then `clearDegradedMode`), part of the recapitalization stays reserved in the pending claimant bucket although its cash has already been paid out to trader claims. LP settlement computes free cash net of that reservation, and a settle that makes no queue progress reverts and rolls back the only reconcile that could release it. Any trader can open a position that brings free cash to zero, because the open gate does not count the reservation; in larger pools the runbook reaches the same state with no position at all. From then on every `settleLpEpoch` reverts, matured Senior redemptions receive nothing, and no permissionless action can break the lock.

## Vulnerability Details

- Reservation without cash: `HousePool._settleLpEpoch` runs the reconcile (`HousePool.sol:835`) and then computes free cash net of `_pendingClaimantBucketAssets()` (`HousePool.sol:842-843`; same deduction in `_buildWithdrawalSnapshot`, `:1733`). After the runbook the residual in that bucket exceeds the pool's physical assets (asserted in the PoC): that part of the recapitalization already left through `settleTraderClaim`.
- Self-blocking release: with zero free cash the settle makes no progress and reverts with `HousePool__NoLpEpochProgress` (`HousePool.sol:1184`), discarding the reconcile that would release the residual. The project's own test (`HousePoolWaterfall.t.sol:195-213`) keeps the residual reserved only until it becomes backed and the next reconcile releases it; here it is never backed, and the only permissionless reconcile undoes itself.
- No third-party exit: a zero-value Junior head is never reached behind the Senior backlog (`HousePool.sol:878-879`), deposits are closed in both tranches (`maxRequestDeposit == 0`, `TrancheVault.sol:1594-1596`), and the Senior request cannot be cancelled. The other paths that commit a reconcile are owner-only (`finalizePoolConfig`, behind the 72-hour timelock; `assignUnassignedAssets`; seeding).
- Open gate: opens check `E >= L + B` (`SolvencyAccountingLib.sol:100-103`, via `CfdEnginePlanLib.sol:421-422, 770-772`) with `E = P − C`; the pending residual is counted only against fee top-ups (`CfdEnginePlanLib.sol:1031`). A position that passes the gate can still bring settle-time free cash to zero. A larger recapitalization does not prevent it: the attacker scales `L` with a skew-neutral LONG/SHORT pair (the PoC uses a surplus of ten times the deficit).
- Without an attacker: the smallest recapitalization that reopens trading is fixed by the skew cap and independent of pool size. In larger pools (the PoC uses ten times its base depth) it leaves a residual above the pool's physical assets with no position open, and the first Senior redemption after the runbook makes every settle revert.
- Runbook: `SECURITY.md:1505-1511` requires only `E >= L` before clearing degraded mode. It does not require a committed settlement before trading resumes, the only ordering that avoids the lock, and does not mention the residual. `SECURITY.md:492` treats insufficient free cash as a liveness state that resolves as conditions change; this one does not.

Not a known issue. Cecuro MED-12 accepts that partial-settlement residuals stay reserved, as in the test above where the residual is later backed and released; a residual that can never be backed and blocks its own release is not described anywhere. Nor is it the withdrawal firewall (`C + L + B`, `SECURITY.md:342`): in the PoC twins, the same physical assets and the same open position pay the Senior once the residual is released, so the frozen amount is exactly what the firewall would let the pool pay.

Feasibility. The starting state is a terminal deficit, which needs a large basket move with the two sides of the book settled one after the other. The loss the pool takes in that event is not what this report is about. The protocol models the state explicitly (`WHITEPAPER.md` §2.3, degraded mode, `recapitalizePool` and its runbook), and the bug is in the recovery: once the owner has recapitalized, the freeze depends on no further price movement, gap or timing, and it blocks cash already in the pool and owed to Senior LPs. Under the program's rules, impact is assessed before feasibility, and the owner's part is the documented runbook, a legitimate action that triggers a logic error.

## Impact Details

Temporary freezing of funds for at least 24 hours. Matured Senior redemptions stay unpaid for as long as an attacker holds a position, with no permissionless way out; the PoC asserts at least 24 hours. The cost is negligible at the smallest recapitalization, and if governance recapitalizes more generously the attacker keeps the lock by scaling the position, so the frozen amount grows with the recapitalization. In larger pools the documented recovery freezes the whole Senior tranche with no attacker. Junior settlement is blocked behind the Senior backlog for the same period. The owner can end the freeze only by recapitalizing again by roughly the attacker's `L` and settling in the same transaction, which pays the Senior from new owner capital, leaves the unbacked residual in place and can be met by a further increase of `L`; no response deadline is assumed (SCOPE, Operating Assumptions). Trader closes, liquidations and already-funded claims remain available.

## References

- `packages/perps/src/HousePool.sol`: `_settleLpEpoch` (:835, :842-843, :878-879, :1184), `_buildWithdrawalSnapshot` (:1733)
- `packages/perps/src/TrancheVault.sol:1594-1596`: `maxRequestDeposit`
- `packages/perps/src/libraries/SolvencyAccountingLib.sol:100-103`, `packages/perps/src/libraries/CfdEnginePlanLib.sol:421-422, 770-772, 1031`: open gate
- `packages/perps/SECURITY.md:342, :492, :1505-1511`; `packages/perps/WHITEPAPER.md` §2.3
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:195-213`
- PoC: `packages/perps/test/poc/RecapResidualSettlementFreeze.t.sol`, run from `packages/perps` with `forge test --match-path test/poc/RecapResidualSettlementFreeze.t.sol --match-contract RecapResidualSettlementFreezeTest -vv`. Production `OrderRouter`, launch parameters, mocks only for Pyth and USDG; the deficit is built by honest accounts trading through the router. Tests: the attacker freeze (at least 24 hours, residual above physical assets); control without the position; twin with the residual released and the same position (Senior paid); no third-party exit; generous recapitalization with a scaled pair; larger pool frozen with no attacker; its twin with the residual released (Senior paid); closing the position releases the queue.

## Proof of Concept

## Proof of Concept

The PoC is a single Foundry test file that runs on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It deploys the production `OrderRouter` with its sidecars and lifecycle book, installs the launch parameters from `packages/perps/DEPLOYMENT.md` through the 72-hour timelocks, and builds on the repository's own `BasePerpTest`. The only mocks are the out-of-scope components: Pyth (`MockPyth`, which relays signed prices) and USDG. No in-scope contract is modified.

1. Place the file at `packages/perps/test/poc/RecapResidualSettlementFreeze.t.sol`.
2. From `packages/perps`, run: `forge test --match-path test/poc/RecapResidualSettlementFreeze.t.sol --match-contract RecapResidualSettlementFreezeTest -vv`

If Foundry's auto-installer fails to resolve `solc 0.8.35`, set `FOUNDRY_SOLC` to the official 0.8.35 release binary.

Steps reproduced

1. Setup (LPs, legitimate): 160k USDG in the Junior tranche and 350k USDG in the Senior tranche, deposited and claimed through ordinary LP epochs.
2. Terminal deficit (honest traders): following `WHITEPAPER.md` §2.3, a LONG book opened at 4/3 is closed at 2/3 and a SHORT book opened at 2/3 is closed at 1.37, so the two sides settle one after the other. The test asserts that trader claims exceed pool assets and records the deficit `d`.
3. Runbook (owner, `SECURITY.md:1505-1511`): `recapitalizePool(d + 2,740 USDG)`, the smallest recapitalization that re-admits a minimum open, then `clearDegradedMode`, and the trader claim is serviced with `settleTraderClaim`. One ordinary keeper settlement of a Junior exit commits. Assertions: `pendingRecapitalizationAssets == d` and the residual exceeds `pool.totalAssets()`.
4. Attack (permissionless): two attacker accounts open a skew-neutral LONG/SHORT pair through `commitOrder` / `executeOrder`. The Senior LP requests redemption of its full balance.
5. Assertions: 25 hourly `settleLpEpoch` calls through the router all revert with `HousePool__NoLpEpochProgress` over at least 24 hours; the Senior's claimable assets stay 0; `getFreeAssets()` is 0; the residual still exceeds physical assets.
6. Controls and twins:
   * `test_control_withoutPosition`: the same state without the pair; the next settlement commits and the Senior is paid.
   * `test_twin_residualReleased_samePosition`: ordinary Junior-exit settlements release the residual first; with the same physical assets (±0.1 %), the same pair and the same request, the settlement commits and the Senior is paid.
   * `test_noThirdPartyExit`: while frozen, a zero-value Junior request does not make a settlement commit, `maxRequestDeposit` is 0 in both tranches, and the Senior cannot cancel its request.
   * `test_closingPositionReleasesQueue`: the attacker closes both legs and the next settlement pays the Senior.
   * `test_generousRecapitalization_scaledPosition`: the owner recapitalizes `d + 10d`; a pair scaled to the surplus keeps every settlement reverting for at least 24 hours.
   * `test_largerPool_runbookFreezesWithoutPosition`: at ten times the base pool depth, with no position open, the first Senior redemption after the runbook makes every settlement revert for at least 24 hours.
   * `test_largerPool_twin_residualReleased`: the same larger pool with the residual released by ordinary settlements first; the Senior is paid.

Expected output

```
[PASS] test_closingPositionReleasesQueue()
[PASS] test_control_withoutPosition()
[PASS] test_generousRecapitalization_scaledPosition()
[PASS] test_largerPool_runbookFreezesWithoutPosition()
[PASS] test_largerPool_twin_residualReleased()
[PASS] test_noThirdPartyExit()
[PASS] test_openPositionFreezesSeniorRedemption()
[PASS] test_twin_residualReleased_samePosition()
Suite result: ok. 8 passed; 0 failed; 0 skipped
```

Code:

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

/// @title An unbacked recapitalization residual blocks LP settlement after the terminal-deficit runbook
/// @notice After a terminal deficit, the owner recapitalizes `d + X` and clears degraded mode, and the trader claim is
///         serviced. The part `d` that paid the claim stays in `pendingRecapitalizationAssets` without cash behind it.
///         Settlement reserves it against free cash, and a settlement without progress reverts together with the
///         reconcile that would release it. A position admitted by the open gate (which ignores the reservation) then
///         keeps every settlement reverting; in larger pools the runbook alone does. Production OrderRouter, launch
///         parameters (DEPLOYMENT.md); only Pyth and USDG are mocked.
contract RecapResidualSettlementFreezeTest is BasePerpTest {

    // Launch parameters (DEPLOYMENT.md), installed through the 72 h timelocks; the frozen-close spread (50 bps)
    // is the base suite's FROZEN_CLOSE_SPREAD_BPS.
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint256 internal constant MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant MAINT_MARGIN_BPS = 10;
    uint256 internal constant INIT_MARGIN_BPS = 20;
    uint256 internal constant FAD_MARGIN_BPS = 300;
    uint256 internal constant BASE_CARRY_BPS = 500;
    uint256 internal constant EXECUTION_FEE_BPS = 4;
    uint256 internal constant SETTLEMENT_BUFFER_BPS = 25;
    uint256 internal constant MIN_OPEN_NOTIONAL = 1000e6;
    uint256 internal constant MAX_SENIOR_EXPOSURE = 40_000_000e6;
    uint256 internal constant MAX_SENIOR_SHARE_BPS = 8000;

    uint256 internal constant LOT = 1e20; // 100 units: 100 USDG notional at a 1.00 basket
    // Smallest two-sided deficit path the open gate admits (WHITEPAPER §2.3): LONG at 4/3, SHORT at 2/3,
    // LONG closed at 2/3, SHORT closed at 1.37.
    uint256 internal constant LONG_ENTRY = 133_333_333;
    uint256 internal constant SHORT_ENTRY = 66_666_667;
    uint256 internal constant EXIT_PRICE = 137_000_000;
    // Smallest recapitalization surplus above the deficit that re-admits a minimum 1,000 USDG open.
    uint256 internal constant SURPLUS = 2740e6;
    uint256 internal constant PAIR_LOTS = 16;
    uint256 internal constant HOURLY_SETTLES = 25;

    address internal juniorLp = makeAddr("juniorLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal thirdParty = makeAddr("thirdParty");
    address internal longTrader = makeAddr("longTrader");
    address internal shortTrader = makeAddr("shortTrader");
    address internal hedgeTrader = makeAddr("hedgeTrader");
    address internal attackerLong = makeAddr("attackerLong");
    address internal attackerShort = makeAddr("attackerShort");
    address internal keeper = makeAddr("keeper");

    OrderRouter internal orderRouter;
    uint256 internal px;
    uint256 internal nonce;
    uint256 internal exits;
    uint256 internal redeemId;

    // ------------------------------------------------------------------ deployment
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predicted = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook book = new OrderLifecycleBook(predicted, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar liqSidecar = new OrderRouterLiquidationBatchSidecar(predicted);
        OrderRouter production = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(liqSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(book)
        );
        assertEq(address(production), predicted, "router address");
        return LegacyOrderRouterHarness(payable(address(production)));
    }

    function setUp() public override {
        super.setUp();
        orderRouter = OrderRouter(payable(address(router)));
        px = 1e8;
        _enterLiveWindow();
        _fundLp(juniorVault, juniorLp, 160_000e6);
        _fundLp(seniorVault, seniorLp, 350_000e6);
        _applyLaunchConfig();
        _enterLiveWindow();
    }

    function _applyLaunchConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory risk;
        risk.riskParams = CfdTypes.RiskParams({
            vpiFactor: VPI_FACTOR,
            maxSkewRatio: MAX_SKEW_RATIO,
            maintMarginBps: MAINT_MARGIN_BPS,
            initMarginBps: INIT_MARGIN_BPS,
            fadMarginBps: FAD_MARGIN_BPS,
            baseCarryBps: BASE_CARRY_BPS,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
        risk.executionFeeBps = EXECUTION_FEE_BPS;
        risk.frozenCloseSpreadBps = FROZEN_CLOSE_SPREAD_BPS;
        risk.settlementBufferBps = SETTLEMENT_BUFFER_BPS;
        engineAdmin.proposeRiskConfig(risk);

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = MIN_OPEN_NOTIONAL;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        rc.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(rc);

        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = MAX_SENIOR_EXPOSURE;
        pc.maxSeniorShareBps = MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(pc);

        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
    }

    // ------------------------------------------------------------------ oracle, keeper and LP helpers
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

    function _enterLiveWindow() internal {
        uint256 day = vm.getBlockTimestamp() / 1 days;
        uint256 weekday = (day + 3) % 7;
        vm.warp((day + (weekday == 0 ? 7 : 7 - weekday)) * 1 days + 10 hours);
        vm.roll(vm.getBlockNumber() + 1);
        _mark();
    }

    /// @dev Moves the mark to `target` in steps of at most 10 %.
    function _goTo(
        uint256 target
    ) internal {
        while (px != target) {
            uint256 step = px / 10;
            px = target > px ? (px + step > target ? target : px + step) : (px - step < target ? target : px - step);
            _mark();
        }
    }

    /// @dev One keeper settlement through OrderRouter.settleLpEpoch at the start of the next live LP epoch.
    function _settleNextEpoch() internal returns (bool ok, bytes4 selector) {
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) + 60);
        vm.roll(vm.getBlockNumber() + 1);
        while (engine.isFadWindow() || engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            vm.roll(vm.getBlockNumber() + 1);
        }
        px = engine.lastMarkPrice();
        _tick();
        uint256 t = vm.getBlockTimestamp();
        bytes[] memory data = _payload(t, t - 1);
        vm.prank(keeper);
        bytes memory ret;
        (ok, ret) = address(orderRouter).call(abi.encodeCall(OrderRouter.settleLpEpoch, (data)));
        if (!ok && ret.length >= 4) {
            selector = bytes4(ret);
        }
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
        (bool ok,) = _settleNextEpoch();
        assertTrue(ok, "deposit settled");
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    /// @dev Adds `factor - 1` copies of the base LP book (160k Junior, 350k Senior).
    function _scalePool(
        uint256 factor
    ) internal {
        for (uint256 i = 1; i < factor; ++i) {
            _fundLp(juniorVault, makeAddr(string.concat("junior", vm.toString(i))), 160_000e6);
            _fundLp(seniorVault, makeAddr(string.concat("senior", vm.toString(i))), 350_000e6);
        }
    }

    /// @dev An ordinary Junior exit: a holder of one wiped Junior share requests its full balance.
    function _juniorExit() internal {
        address lp = address(uint160(uint256(keccak256(abi.encode("junior exit", ++exits)))));
        vm.prank(juniorLp);
        juniorVault.transfer(lp, 1);
        vm.prank(lp);
        juniorVault.requestRedeem(1, lp, lp);
    }

    function _seniorExit() internal {
        uint256 shares = seniorVault.balanceOf(seniorLp);
        vm.prank(seniorLp);
        redeemId = seniorVault.requestRedeem(shares, seniorLp, seniorLp);
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

    function _size(
        address t
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(t);
    }

    function _order(
        address t,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        _mark();
        uint256 window = orderRouter.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++nonce),
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
        vm.prank(t);
        orderRouter.commitOrder(req);

        uint64 head = orderRouter.nextExecuteId();
        (IOrderRouterAccounting.PendingOrderView memory pv,) = orderRouter.getPendingOrderView(head);
        _tick();
        uint256 pt = uint256(pv.commitTime) + 1;
        if (engine.lastMarkTime() > pt) {
            pt = engine.lastMarkTime();
        }
        if (pt > vm.getBlockTimestamp()) {
            pt = vm.getBlockTimestamp();
        }
        bytes[] memory data = _payload(pt, pv.commitTime);
        vm.prank(keeper);
        orderRouter.executeOrder(head, data);
    }

    function _open(
        address t,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin
    ) internal {
        uint256 s0 = _size(t);
        _order(t, side, lots * LOT, margin, false);
        assertEq(_size(t), s0 + lots * LOT, "open filled");
    }

    function _close(
        address t
    ) internal {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(t);
        _order(t, side, size, 0, true);
        assertEq(_size(t), 0, "close filled");
    }

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    /// @dev Skew-neutral pair: `lots` LONG and `lots` SHORT at 1 % margin, opened in chunks inside the skew cap.
    ///      The LONG side sets L (max profit = size x entry).
    function _openPair(
        uint256 lots
    ) internal {
        _depositMargin(attackerLong, (lots * px) / 20 + 1000e6);
        _depositMargin(attackerShort, (lots * px) / 20 + 1000e6);
        uint256 minLots = (MIN_OPEN_NOTIONAL + px - 1) / px;
        uint256 done;
        while (done < lots) {
            uint256 chunk = ((_depth() * 35) / 100) / px;
            if (chunk < minLots) {
                chunk = minLots;
            }
            uint256 c = lots - done < chunk ? lots - done : chunk;
            _open(attackerLong, CfdTypes.Side.LONG, c, (c * px) / 100 + 30e6);
            _open(attackerShort, CfdTypes.Side.SHORT, c, (c * px) / 100 + 30e6);
            done += c;
        }
    }

    // ------------------------------------------------------------------ deficit and runbook
    /// @dev Sequential two-sided settlement at the largest book the open gate admits. Returns the deficit d.
    function _terminalDeficit() internal returns (uint256 d) {
        _depositMargin(longTrader, 10_000_000e6);
        _depositMargin(shortTrader, 10_000_000e6);
        _depositMargin(hedgeTrader, 10_000_000e6);
        uint256 m = LONG_ENTRY > CAP_PRICE - SHORT_ENTRY ? LONG_ENTRY : CAP_PRICE - SHORT_ENTRY;
        uint256 lots = (_depth() * 9900) / (10_025 * m);

        _goTo(LONG_ENTRY);
        uint256 chunk = ((_depth() * 35) / 100) / LONG_ENTRY;
        for (uint256 done; done < lots;) {
            uint256 c = lots - done < chunk ? lots - done : chunk;
            _open(longTrader, CfdTypes.Side.LONG, c, (c * LONG_ENTRY) / 100);
            _open(hedgeTrader, CfdTypes.Side.SHORT, c, (c * LONG_ENTRY) / 100);
            done += c;
        }
        _close(hedgeTrader);
        _goTo(SHORT_ENTRY);
        chunk = ((_depth() * 35) / 100) / SHORT_ENTRY;
        for (uint256 done; done < lots;) {
            uint256 c = lots - done < chunk ? lots - done : chunk;
            _open(shortTrader, CfdTypes.Side.SHORT, c, (c * SHORT_ENTRY) / 100);
            done += c;
        }
        _close(longTrader);
        _goTo(EXIT_PRICE);
        _close(shortTrader);

        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        assertGt(claims, cash, "terminal deficit");
        d = claims - cash;
    }

    /// @dev SECURITY.md:1505-1511: recapitalize d + x, clear degraded mode once E >= L, service the claim.
    function _runbook(
        uint256 d,
        uint256 x
    ) internal {
        settlementToken.mint(address(this), d + x);
        settlementToken.approve(address(engine), d + x);
        engine.recapitalizePool(d + x);
        engine.clearDegradedMode();
        vm.prank(shortTrader);
        engine.settleTraderClaim(shortTrader);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claim serviced");
    }

    /// @dev Runbook followed by one ordinary keeper settlement of a Junior exit.
    function _recover(
        uint256 d
    ) internal {
        _runbook(d, SURPLUS);
        _juniorExit();
        (bool ok,) = _settleNextEpoch();
        assertTrue(ok, "ordinary settlement after the runbook");
        assertEq(pool.pendingRecapitalizationAssets(), d, "deficit part of the recapitalization still pending");
        assertGt(pool.pendingRecapitalizationAssets(), pool.totalAssets(), "pending residual exceeds physical assets");
    }

    /// @dev Ordinary Junior-exit settlements, with no Senior request queued, until the residual is released.
    function _releaseResidual() internal {
        while (pool.pendingRecapitalizationAssets() > 0) {
            _juniorExit();
            (bool ok,) = _settleNextEpoch();
            assertTrue(ok, "ordinary settlement");
        }
    }

    function _assertFrozen() internal {
        uint256 first;
        for (uint256 k; k < HOURLY_SETTLES; ++k) {
            (bool ok, bytes4 selector) = _settleNextEpoch();
            assertFalse(ok, "settlement must not commit");
            assertEq(selector, IHousePool.HousePool__NoLpEpochProgress.selector, "NoLpEpochProgress");
            if (k == 0) {
                first = vm.getBlockTimestamp();
            }
        }
        assertGe(vm.getBlockTimestamp() - first, 24 hours, "frozen for at least 24 hours");
        assertEq(seniorVault.claimableRedeemAssets(redeemId, seniorLp), 0, "Senior receives nothing");
        assertEq(pool.getFreeAssets(), 0, "no free cash");
    }

    function _assertSeniorPaid() internal {
        (bool ok,) = _settleNextEpoch();
        assertTrue(ok, "settlement commits");
        assertGt(seniorVault.claimableRedeemAssets(redeemId, seniorLp), 0, "Senior redemption funded");
    }

    // ------------------------------------------------------------------ tests
    /// @notice An open position admitted after the runbook freezes the matured Senior redemption.
    function test_openPositionFreezesSeniorRedemption() public {
        _recover(_terminalDeficit());
        _openPair(PAIR_LOTS);
        _seniorExit();
        _assertFrozen();
        assertGt(pool.pendingRecapitalizationAssets(), pool.totalAssets(), "residual still exceeds physical assets");
    }

    /// @notice Control: same state without the position; the Senior is paid.
    function test_control_withoutPosition() public {
        _recover(_terminalDeficit());
        _seniorExit();
        _assertSeniorPaid();
    }

    /// @notice Twin: ordinary settlements release the residual before the same position opens; same
    ///         physical assets, same position, same request; the Senior is paid up to the C + L + B firewall.
    function test_twin_residualReleased_samePosition() public {
        uint256 d = _terminalDeficit();
        uint256 snap = vm.snapshotState();
        _recover(d);
        _openPair(PAIR_LOTS);
        uint256 frozenAssets = pool.totalAssets();
        vm.revertToState(snap);

        _recover(d);
        _releaseResidual();
        _openPair(PAIR_LOTS);
        assertApproxEqRel(pool.totalAssets(), frozenAssets, 0.001e18, "same physical assets");
        _seniorExit();
        _assertSeniorPaid();
    }

    /// @notice No third party can make a settlement commit while frozen.
    function test_noThirdPartyExit() public {
        _recover(_terminalDeficit());
        _openPair(PAIR_LOTS);
        _seniorExit();
        (bool ok,) = _settleNextEpoch();
        assertFalse(ok, "frozen");

        // A zero-value Junior head is not reached behind the Senior backlog.
        _juniorExit();
        (ok,) = _settleNextEpoch();
        assertFalse(ok, "zero-value Junior head does not commit");
        // Deposits are closed in both tranches.
        assertEq(juniorVault.maxRequestDeposit(thirdParty), 0, "Junior deposits closed");
        assertEq(seniorVault.maxRequestDeposit(thirdParty), 0, "Senior deposits closed");
        // The matured Senior request cannot be cancelled.
        vm.prank(seniorLp);
        (ok,) = address(seniorVault)
            .call(abi.encodeWithSignature("cancelRedeemRequest(uint256,address)", redeemId, seniorLp));
        assertFalse(ok, "Senior request not cancellable");
    }

    /// @notice Closing the position releases the queue.
    function test_closingPositionReleasesQueue() public {
        _recover(_terminalDeficit());
        _openPair(PAIR_LOTS);
        _seniorExit();
        (bool ok,) = _settleNextEpoch();
        assertFalse(ok, "frozen");
        _close(attackerLong);
        _close(attackerShort);
        _assertSeniorPaid();
    }

    /// @notice A recapitalization of d + 10d: the position is scaled to L ~ surplus - d/2 with a skew-neutral pair,
    ///         opened before the first settlement after the runbook; the freeze holds.
    function test_generousRecapitalization_scaledPosition() public {
        uint256 d = _terminalDeficit();
        uint256 surplus = 10 * d;
        _runbook(d, surplus);
        _openPair((surplus - d / 2) / px);
        _seniorExit();
        _assertFrozen();
        assertGe(_maxLiability(), surplus - d / 2 - px, "L scaled with the surplus");
    }

    /// @notice At 10x the pool depth (~5.1M USDG) the runbook alone freezes the first Senior redemption: no position open.
    function test_largerPool_runbookFreezesWithoutPosition() public {
        _scalePool(10);
        _recover(_terminalDeficit());
        assertEq(_maxLiability(), 0, "no open position");
        _seniorExit();
        _assertFrozen();
    }

    /// @notice Twin at 10x: ordinary Junior-exit settlements release the residual first; the Senior is then paid.
    function test_largerPool_twin_residualReleased() public {
        _scalePool(10);
        _recover(_terminalDeficit());
        _releaseResidual();
        _seniorExit();
        _assertSeniorPaid();
    }

}
```
