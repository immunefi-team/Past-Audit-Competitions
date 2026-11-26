# \#98795 [SC-Medium] an open or increase never lowers the liquidation reserve to its new target, so collateral the spec defines as PnL pledge is held as reserve, positions are liquidated early and the excess is charged to the trader or taken from LPs

**Submitted on Nov 10th 2026 at 08:09:06 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98795
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

every open or same-side increase recomputes the liquidation reserve target from the resulting notional, and the spec says collateral beyond that target is PnL pledge. the planner instead keeps `max(previous reserve, new target)`. after an increase at a price that lowers the target, the excess stays locked in the reserve.

health checks count only pledge and claim, so the trapped excess is invisible to them. the position becomes liquidatable while its spec equity is still above maintenance. at liquidation the excess raises the charge for a LONG, and for a SHORT liquidated with a loss above the pledge it goes back to the trader instead of to LPs. in the PoC a 1_000_000 USDG SHORT that adds 12 lots after a 10% move keeps 98.92 USDG of excess, is liquidatable at an equity of 883.76 against a spec equity of 982.68, and leaves LPs 98.92 USDG short.

## Vulnerability Details

the spec defines the target after an open or increase and classifies everything above it ([ACCOUNTING_SPEC.md#L806-L813](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L806-L813)):

> After an open/increase, the target liquidation reserve is: `max(minBountyAssets, floor(resultingNotionalAssets * bountyBps / 10_000))`
>
> Approved position collateral beyond that target is PnL pledge. A reduction recomputes the target and releases excess reserve; price movement alone never permits a liquidation charge to consume PnL pledge.

the open planner raises the new target back to the old reserve whenever the target is lower ([CfdEnginePlanLib.sol#L551-L556](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L551-L556)):

```solidity
        delta.liquidationReserveBeforeAssets = snap.liquidationReserveAssets;
        delta.liquidationReserveAfterAssets = delta.openState.liquidationReserveTargetAssets;
@>      if (delta.liquidationReserveAfterAssets < delta.liquidationReserveBeforeAssets) {
@>          delta.liquidationReserveAfterAssets = delta.liquidationReserveBeforeAssets;
        }
        uint256 reserveIncreaseAssets = delta.liquidationReserveAfterAssets - delta.liquidationReserveBeforeAssets;
```

and the settlement sidecar only ever moves collateral into the reserve on an open ([CfdEngineSettlementSidecar.sol#L480-L484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L480-L484)):

```solidity
@>      if (delta.liquidationReserveAfterAssets > delta.liquidationReserveBeforeAssets) {
            clearinghouse.reclassifyPnlPledgeToLiquidationReserve(
                delta.account, delta.liquidationReserveAfterAssets - delta.liquidationReserveBeforeAssets
            );
        }
```

the same open path follows the target in both directions for the other reserve it manages. when the VPI rebate reserve target falls in an open, the excess is released in that fill ([CfdEnginePlanLib.sol#L348-L355](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L348-L355), [CfdEngineSettlementSidecar.sol#L436-L439](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L436-L439)). the liquidation reserve is the only one that ratchets.

the stored reserve therefore depends on the path, not on the position. a SHORT of 10_000 lots at 1.00 sets a 1_000 USDG reserve. after +12 lots at 0.90 the target is `0.1% * 10_012 * 100 * 0.90 = 901.08` USDG, but the reserve stays 1_000. an account that opens the same 10_012 lots directly at 0.90 holds exactly 901.08. the excess is `bountyBps / 1e4 * (N * P1 - (N + K) * P2)`: 98.92 USDG per 1_000_000 notional after a 10% move, and it grows with notional, the size of the move and every further increase.

the excess then distorts three things.

1. health ignores it. price health is computed on `pos.margin + traderClaimBalance` ([CfdEnginePlanLib.sol#L1461](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1461)), so collateral that the spec classifies as pledge is missing from equity. in the PoC the SHORT is first liquidatable at an equity of 883.762256 against a maintenance of 883.765066. counting the excess as pledge, its equity is 982.682256 and it is not liquidatable. the LONG case is the same: 1_004.270282 against 1_004.274424, and 1_103.190282 with the excess.
2. the charge consumes it. the liquidation charge is capped at the stored reserve ([CfdEnginePlanLib.sol#L1495-L1503](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1495-L1503)). a LONG liquidated at its first liquidatable price pays 1_000 USDG where its target allows 901.08, so the trader loses 98.92 USDG of what the spec calls pledge to the keeper and LPs. that is exactly what "price movement alone never permits a liquidation charge to consume PnL pledge" rules out.
3. bad debt returns it to the trader. price seizure is capped at the pledge, and whatever is left in the reserve after the charge is zeroed with no settlement debit ([MarginClearinghouse.sol#L1279-L1285](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1279-L1285)), so it stays in the trader's free settlement:

```solidity
        uint256 releasedLiquidationReserveAssets = liquidationReserveBalances[account];
        if (releasedLiquidationReserveAssets > 0) {
@>          liquidationReserveBalances[account] = 0;
            emit MarginUnlocked(
                account, IMarginClearinghouse.MarginBucket.LiquidationReserve, releasedLiquidationReserveAssets
            );
        }
```

when the loss exceeds the pledge, the excess that should have been seized as pledge is handed back to the trader and the write-off to LPs is larger by the same amount. in the PoC two SHORT accounts with the same size and the same pledge + reserve (119_198.70 USDG) are liquidated at the same price and pay the same 770.924 USDG charge. the ratchet account keeps 98.92 USDG more and the pool receives 98.92 USDG less (118_584.16 against 118_683.08). a SHORT liquidated after a 0.111% gap below its first liquidatable price already produces a 99.93 USDG write-off, with the full 98.92 excess returned to the trader.

## Impact Details

- positions are liquidated while their equity, as the spec classifies it, is above maintenance. in the PoC the band is 98.92 USDG of equity per 1_000_000 notional after a 10% move, and the trader loses the position and pays the full liquidation charge.
- a LONG liquidated with the excess in its reserve pays 98.92 USDG more charge than its target allows, split between the keeper and LPs.
- a SHORT liquidated with a loss above its pledge keeps 98.92 USDG that the spec assigns to pledge, and LPs absorb the same amount as an extra write-off.
- every account that increases after a favourable move carries the excess, it compounds over repeated increases, and it persists until a reduction recomputes the target.

severity: Medium, Griefing (no profit motive for an attacker, but damage to the users or the protocol). no attacker is needed: ordinary increases put every such account in this state, the users lose positions and liquidation charge they should keep, and LPs absorb larger write-offs.

## Recommendation

set the reserve to the recomputed target on every open, and release any excess to PnL pledge in the same fill, the same way the VPI rebate reserve is handled:

```diff
         delta.liquidationReserveBeforeAssets = snap.liquidationReserveAssets;
         delta.liquidationReserveAfterAssets = delta.openState.liquidationReserveTargetAssets;
-        if (delta.liquidationReserveAfterAssets < delta.liquidationReserveBeforeAssets) {
-            delta.liquidationReserveAfterAssets = delta.liquidationReserveBeforeAssets;
-        }
-        uint256 reserveIncreaseAssets = delta.liquidationReserveAfterAssets - delta.liquidationReserveBeforeAssets;
+        uint256 reserveIncreaseAssets = delta.liquidationReserveAfterAssets > delta.liquidationReserveBeforeAssets
+            ? delta.liquidationReserveAfterAssets - delta.liquidationReserveBeforeAssets
+            : 0;
```

and in `CfdEngineSettlementSidecar` move `liquidationReserveBeforeAssets - liquidationReserveAfterAssets` back to PnL pledge when the target is lower.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L546-L580

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L480-L484

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L348-L355

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L433-L440

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1455-L1463

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1495-L1503

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1279-L1285

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L141-L152

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L806-L813

## Proof of Concept

## Proof of Concept

the test deploys the in-scope contracts through the existing `BasePerpTest` harness at release risk parameters, with the release Router minimum open notional of 1_000 USDG installed through the production timelock. only Pyth (`MockPyth`) and USDG (mock token) are mocked. every trade is committed through the production Router and executed by a keeper, marks use `OrderRouter.updateMarkPrice`, and liquidations use `OrderRouter.executeLiquidation`.

1. create `packages/perps/test/perps/poc/LiquidationReserveRatchet.t.sol` with the content below (no harness changes are needed)
2. from `packages/perps`, run:

```
forge test --match-path test/perps/poc/LiquidationReserveRatchet.t.sol -vv
```

tests:

- `test_increaseKeepsReserveAboveTarget`: the reserve stays 1_000 against a 901.08 target, and a control opening the same final position gets the target
- `test_negativeControl_increaseAtHigherPrice_reserveEqualsTarget`: an increase that raises the target sets the reserve exactly to it
- `test_shortBadDebtLiquidation_excessReturnedToTrader`: same size, same pledge + reserve, same charge; the trader keeps 98.92 more and LPs receive 98.92 less
- `test_shortRealisticGap_lpShortfallEqualsExcess`: early liquidation band and LP shortfall after a 0.111% gap
- `test_longLiquidation_extraCharge`: the LONG pays 98.92 more charge, with the same early liquidation band
- `test_reductionReleasesExcess`: a reduction recomputes the target, and the same open fill releases the VPI reserve while keeping the liquidation reserve excess

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @title PoC: an open/increase keeps the liquidation reserve at max(previous reserve, new target).
/// @notice CfdEnginePlanLib._planOpenReserveFunding (CfdEnginePlanLib.sol:551-556) sets the post-open reserve to the
///         recomputed target and then raises it back to the previous reserve whenever the target is lower. After a
///         same-side increase at a lower price the stored reserve therefore stays above the target that
///         ACCOUNTING_SPEC.md:806-811 defines, and the excess is never credited to PnL pledge.
/// @dev Local deploy of the repo contracts at a82d4fef (BasePerpTest) with release risk parameters
///      (DEPLOYMENT.md:126-165) and the release Router values minOpenNotionalAssets = 1,000 USDG and
///      adverseConfidenceMultiplierBps = 2,500 (DEPLOYMENT.md:147-148), installed through the production 72 h
///      OrderRouterAdmin timelock. Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. Every trade goes
///      through the production Router FIFO (LegacyOrderRouterHarness.commitOrder -> production V3 commitOrder, then a
///      keeper calls OrderRouter.executeOrder). Marks use the permissionless OrderRouter.updateMarkPrice, liquidations
///      the permissionless OrderRouter.executeLiquidation, and the control's top-up the trader-only CfdEngine.addMargin.
contract LiquidationReserveRatchetTest is BasePerpTest {

    address internal constant RATCHET = address(0xA11CE);
    address internal constant CONTROL = address(0xC0C0);
    address internal constant OTHER = address(0xB0B);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant P1 = 1e8; // 1.00
    uint256 internal constant P2 = 0.9e8; // 0.90, a 10% fall
    uint256 internal constant P_UP = 1.1e8; // negative control: increase at a higher price
    uint256 internal constant DEEP_PRICE = 0.77e8; // common price that puts BOTH A/B accounts past their pledge

    uint256 internal constant N_LOTS = 10_000; // 1,000,000 USDG notional at 1.00 (1 lot = 1e20 size = 100 tokens)
    uint256 internal constant K_LOTS = 12; // 1,080 USDG at 0.90 >= release minOpenNotionalAssets (1,000 USDG)

    uint256 internal constant SHORT_DEPOSIT = 150_000e6;
    uint256 internal constant SHORT_MARGIN = 120_000e6; // survives the 10% adverse move before the increase
    uint256 internal constant LONG_DEPOSIT = 20_000e6;
    uint256 internal constant LONG_MARGIN = 6_000e6;
    uint256 internal constant INCREASE_MARGIN = 100e6;
    uint256 internal constant CONTROL_MARGIN = 5_000e6; // control opens small, then tops up to the same collateral

    /// @dev Mirrors ICfdEngineSettlementSidecar.PriceLossWrittenOff (ICfdEngineSettlementSidecar.sol:25) for expectEmit.
    event PriceLossWrittenOff(address indexed account, uint256 amountAssets);

    struct LiqOutcome {
        uint256 price;
        uint256 reserveBefore;
        uint256 pledgeBefore;
        uint256 charge;
        uint256 keeperBounty;
        uint256 lpFee;
        int256 equity;
        uint256 maintenance;
        uint256 traderFreeBefore;
        uint256 traderFreeAfter;
        uint256 traderSettlementBefore;
        uint256 traderSettlementAfter;
        uint256 poolCashIn;
        uint256 keeperCredit;
        uint256 writeOff;
        uint256 reserveAfter;
    }

    // ---------------------------------------------------------------------
    // Release parameters (DEPLOYMENT.md:126-165)
    // ---------------------------------------------------------------------

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

    /// @dev 10M USDG Junior depth so a 1M position fits maxSkewRatio 0.4 and the settlement buffer.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");

        // Release Router values (DEPLOYMENT.md:147-148) through the production 72 h timelock.
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1_000e6;
        config.adverseConfidenceMultiplierBps = 2_500;
        _setRouterConfig(config);
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minOpenNotionalAssets");

        _mark(P1);
    }

    // ---------------------------------------------------------------------
    // 1. The increase keeps the reserve above the recomputed target
    // ---------------------------------------------------------------------

    /// @notice SHORT N lots at 1.00, then +K lots at 0.90. The stored reserve stays 1,000 USDG while the recomputed
    ///         target is 901.08 USDG. A control that opens the same final size at 0.90 gets exactly the target.
    function test_increaseKeepsReserveAboveTarget() public {
        _fundTrader(RATCHET, SHORT_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.SHORT, N_LOTS, SHORT_MARGIN, false, P1);
        uint256 r1 = clearinghouse.liquidationReserveAssets(RATCHET);
        assertEq(r1, _target(N_LOTS, P1), "open at 1.00 sets reserve = target");
        assertEq(r1, 1_000e6, "R1 = 1,000 USDG");

        _mark(P2);
        uint256 pledgeBefore = clearinghouse.pnlPledgeAssets(RATCHET);
        _fill(RATCHET, CfdTypes.Side.SHORT, K_LOTS, INCREASE_MARGIN, false, P2);
        uint256 pledgeAfter = clearinghouse.pnlPledgeAssets(RATCHET);
        uint256 reserveAfter = clearinghouse.liquidationReserveAssets(RATCHET);
        uint256 target = _target(N_LOTS + K_LOTS, P2);

        assertEq(target, 901_080_000, "spec target after increase = 901.08 USDG");
        assertEq(reserveAfter, r1, "increase leaves the reserve at the previous value");
        assertGt(reserveAfter, target, "stored reserve > recomputed target");
        uint256 excess = reserveAfter - target;
        // Bound: (bountyBps / 1e4) * (N*P1 - (N+K)*P2) in USDG atoms.
        assertEq(excess, ((N_LOTS * P1 - (N_LOTS + K_LOTS) * P2) * 10) / 10_000, "excess = 0.1% of notional drop");
        assertEq(excess, 98_920_000, "excess = 98.92 USDG");
        // The excess is not credited to PnL pledge: pledge grows by at most the new margin (less trade cost).
        assertLe(pledgeAfter, pledgeBefore + INCREASE_MARGIN, "pledge not credited with the excess");

        // Control: the same final size opened directly at 0.90.
        _fundTrader(CONTROL, SHORT_DEPOSIT);
        _fill(CONTROL, CfdTypes.Side.SHORT, N_LOTS + K_LOTS, CONTROL_MARGIN, false, P2);
        uint256 controlReserve = clearinghouse.liquidationReserveAssets(CONTROL);
        (uint256 sizeR,,,,,,) = engine.positions(RATCHET);
        (uint256 sizeC,,,,,,) = engine.positions(CONTROL);
        assertEq(sizeR, sizeC, "same final size");
        assertEq(controlReserve, target, "control reserve == target");

        console2.log("===== test_increaseKeepsReserveAboveTarget (SHORT, 1M notional, 10% fall) =====");
        console2.log("reserve after open of 10,000 lots @1.00 (R1)  ", r1);
        console2.log("recomputed target, 10,012 lots @0.90           ", target);
        console2.log("ratchet reserve after +12 lots @0.90           ", reserveAfter);
        console2.log("control reserve, 10,012 lots opened @0.90      ", controlReserve);
        console2.log("excess held as reserve instead of PnL pledge   ", excess);
        console2.log("ratchet pledge before/after increase           ", pledgeBefore, pledgeAfter);
    }

    /// @notice Negative control from the candidate: an increase at a HIGHER price makes the target exceed the old
    ///         reserve, so max(before, target) == target and no excess exists.
    function test_negativeControl_increaseAtHigherPrice_reserveEqualsTarget() public {
        _fundTrader(RATCHET, SHORT_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.SHORT, N_LOTS, SHORT_MARGIN, false, P1);
        _mark(P_UP);
        // The reserve now has to GROW by 101.32 USDG, and the planner funds a reserve increase only from the
        // order's newly supplied pledge (CfdEnginePlanLib.sol:557-576; else LIQUIDATION_RESERVE_UNFUNDED = code 9).
        // So this increase supplies 500 USDG of margin instead of INCREASE_MARGIN.
        _fill(RATCHET, CfdTypes.Side.SHORT, K_LOTS, 500e6, false, P_UP);
        uint256 reserve = clearinghouse.liquidationReserveAssets(RATCHET);
        assertEq(reserve, _target(N_LOTS + K_LOTS, P_UP), "reserve == target when the target rises");
        assertEq(reserve, 1_101_320_000, "target at 1.10 = 1,101.32 USDG");
        console2.log("negative control: reserve after +12 lots @1.10 == target", reserve);
    }

    // ---------------------------------------------------------------------
    // 2. SHORT bad-debt liquidation: the excess goes back to the trader, not to LPs
    // ---------------------------------------------------------------------

    /// @notice A/B at a common liquidation price past both accounts' pledge. Both have the same size and the same
    ///         total position collateral (pledge + reserve); only the reserve/pledge split differs. The ratchet account
    ///         gets the excess back as free settlement and the pool receives exactly the excess less.
    function test_shortBadDebtLiquidation_excessReturnedToTrader() public {
        uint256 snap = vm.snapshotState();

        // A: ratchet path.
        _fundTrader(RATCHET, SHORT_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.SHORT, N_LOTS, SHORT_MARGIN, false, P1);
        _mark(P2);
        _fill(RATCHET, CfdTypes.Side.SHORT, K_LOTS, INCREASE_MARGIN, false, P2);
        uint256 totalCollateral =
            clearinghouse.pnlPledgeAssets(RATCHET) + clearinghouse.liquidationReserveAssets(RATCHET);
        LiqOutcome memory a = _liquidate(RATCHET, DEEP_PRICE);

        vm.revertToState(snap);

        // B: control, same final size opened at 0.90, topped up to the same pledge + reserve.
        _mark(P2);
        _fundTrader(CONTROL, SHORT_DEPOSIT);
        _fill(CONTROL, CfdTypes.Side.SHORT, N_LOTS + K_LOTS, CONTROL_MARGIN, false, P2);
        _equalizeCollateral(CONTROL, totalCollateral);
        LiqOutcome memory b = _liquidate(CONTROL, DEEP_PRICE);

        uint256 target = _target(N_LOTS + K_LOTS, P2);
        uint256 excess = a.reserveBefore - target;

        _logOutcome("A: RATCHET (10,000 @1.00 then +12 @0.90)", a);
        _logOutcome("B: CONTROL (10,012 @0.90, same pledge + reserve)", b);

        assertEq(a.reserveBefore, 1_000e6, "A reserve = R1");
        assertEq(b.reserveBefore, target, "B reserve = target");
        assertEq(a.pledgeBefore + a.reserveBefore, b.pledgeBefore + b.reserveBefore, "same total collateral");
        assertEq(b.pledgeBefore - a.pledgeBefore, excess, "B pledge = A pledge + excess");
        assertEq(a.charge, b.charge, "same charge (same size, same price)");
        assertEq(a.charge, ((N_LOTS + K_LOTS) * DEEP_PRICE * 10) / 10_000, "charge = notional(P_liq) * 10 bps");
        assertLt(a.charge, target, "SHORT: charge at a lower price is below the target");
        assertGt(a.writeOff, excess, "A: loss above pledge is written off (> excess)");
        assertGt(b.writeOff, 0, "B: loss above pledge is written off");
        assertEq(a.reserveAfter, 0, "reserve bucket zeroed");

        // Trader side: the leftover reserve (incl. the excess) stays in the trader's free settlement.
        uint256 aFreeGain = a.traderFreeAfter - a.traderFreeBefore;
        uint256 bFreeGain = b.traderFreeAfter - b.traderFreeBefore;
        assertApproxEqAbs(aFreeGain, a.reserveBefore - a.charge, 0.1e6, "A: free settlement gains R1 - charge");
        assertApproxEqAbs(bFreeGain, b.reserveBefore - b.charge, 0.1e6, "B: free settlement gains target - charge");
        assertApproxEqAbs(aFreeGain - bFreeGain, excess, 0.1e6, "trader keeps the excess vs control");

        // LP side: the pool receives exactly the excess less.
        assertApproxEqAbs(b.poolCashIn - a.poolCashIn, excess, 0.1e6, "pool receives excess less vs control");

        console2.log("excess (A reserve - target)                    ", excess);
        console2.log("trader free-settlement gain A - B              ", aFreeGain - bFreeGain);
        console2.log("pool cash received B - A (LP shortfall)        ", b.poolCashIn - a.poolCashIn);
    }

    /// @notice Ratchet account alone, liquidated after a realistic gap below its first liquidatable price (about
    ///         0.11%). The loss exceeds pledge + excess, so under the spec classification the excess would have been
    ///         seized as pledge; here it is unlocked to the trader. Also shows the premature-liquidation band: at the
    ///         first liquidatable price, equity + excess is still above maintenance.
    function test_shortRealisticGap_lpShortfallEqualsExcess() public {
        _fundTrader(RATCHET, SHORT_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.SHORT, N_LOTS, SHORT_MARGIN, false, P1);
        _mark(P2);
        _fill(RATCHET, CfdTypes.Side.SHORT, K_LOTS, INCREASE_MARGIN, false, P2);

        uint256 excess = clearinghouse.liquidationReserveAssets(RATCHET) - _target(N_LOTS + K_LOTS, P2);

        uint256 pFirst = _firstLiquidatablePriceShort(RATCHET);
        _assertPrematureBand(RATCHET, pFirst, excess);

        // Price at which loss = pledge + excess + 1 USDG (SHORT loss = entry cost - lots * price).
        uint256 pGap = (engine.positionEntryCostAssetAtoms(RATCHET) - clearinghouse.pnlPledgeAssets(RATCHET) - excess
                - 1e6) / (N_LOTS + K_LOTS);

        LiqOutcome memory o = _liquidate(RATCHET, pGap);
        _logOutcome("RATCHET liquidated after a small gap", o);

        assertGt(o.writeOff, excess, "write-off exceeds the excess");
        uint256 released = o.reserveBefore - o.charge;
        assertGt(released, excess, "released reserve includes the full excess");
        assertApproxEqAbs(o.traderFreeAfter - o.traderFreeBefore, released, 0.1e6, "released reserve -> trader");
        assertApproxEqAbs(o.poolCashIn, o.pledgeBefore + o.lpFee, 0.1e6, "pool cash = whole pledge + LP fee (+carry)");
        // Spec counterfactual: pledge would be pledge + excess and the loss covers it, so LPs would seize
        // min(loss, pledge + excess) = pledge + excess. Shortfall = min(excess, writeOff) = excess.
        assertEq(o.writeOff < excess ? o.writeOff : excess, excess, "LP shortfall vs spec = excess");

        console2.log("first liquidatable price (1e8)                 ", pFirst);
        console2.log("liquidation price after gap (1e8)              ", pGap);
        console2.log("gap below first liquidatable price (0.01 bps)  ", ((pFirst - pGap) * 1_000_000) / pFirst);
        console2.log("excess                                         ", excess);
        console2.log("released reserve to trader (R1 - charge)       ", released);
        console2.log("of which spec-pledge excess (LP shortfall)     ", excess);
    }

    // ---------------------------------------------------------------------
    // 3. LONG liquidation: the excess is consumed as extra liquidation charge
    // ---------------------------------------------------------------------

    /// @notice LONG N at 1.00 (profits as the price falls), +K at 0.90, then the price rises back to the first
    ///         liquidatable price. The uncapped charge exceeds R1, so the charge consumes the whole stored reserve,
    ///         98.92 USDG more than a reserve equal to the target allows. A/B at the same price and same collateral.
    function test_longLiquidation_extraCharge() public {
        uint256 snap = vm.snapshotState();

        // A: ratchet LONG.
        _fundTrader(RATCHET, LONG_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.LONG, N_LOTS, LONG_MARGIN, false, P1);
        _mark(P2);
        _fill(RATCHET, CfdTypes.Side.LONG, K_LOTS, INCREASE_MARGIN, false, P2);
        uint256 target = _target(N_LOTS + K_LOTS, P2);
        uint256 r1 = clearinghouse.liquidationReserveAssets(RATCHET);
        uint256 excess = r1 - target;
        uint256 totalCollateral = clearinghouse.pnlPledgeAssets(RATCHET) + r1;

        uint256 pFirst = _firstLiquidatablePriceLong(RATCHET);
        // maintMarginBps == bountyBps == 10 at release, so maintenance == uncapped charge at this price.
        uint256 uncappedCharge = _assertPrematureBand(RATCHET, pFirst, excess);
        assertGt(uncappedCharge, r1, "uncapped charge > stored reserve");

        LiqOutcome memory a = _liquidate(RATCHET, pFirst);

        vm.revertToState(snap);

        // B: control LONG, same size opened at 0.90 and topped up to the same pledge + reserve.
        _mark(P2);
        _fundTrader(CONTROL, LONG_DEPOSIT);
        _fill(CONTROL, CfdTypes.Side.LONG, N_LOTS + K_LOTS, CONTROL_MARGIN, false, P2);
        _equalizeCollateral(CONTROL, totalCollateral);
        LiqOutcome memory b = _liquidate(CONTROL, pFirst);

        _logOutcome("A: RATCHET LONG (10,000 @1.00 then +12 @0.90)", a);
        _logOutcome("B: CONTROL LONG (10,012 @0.90, same pledge + reserve)", b);

        assertEq(a.writeOff, 0, "A: ordinary liquidation, no write-off");
        assertEq(a.charge, r1, "A: charge consumes the whole stored reserve");
        assertEq(b.reserveBefore, target, "B reserve = target");
        assertEq(b.charge, target, "B: charge capped at the target reserve");
        assertEq(a.charge - b.charge, excess, "extra charge = excess");
        assertApproxEqAbs(a.keeperBounty - b.keeperBounty, excess / 2, 1, "keeper gets half the excess");
        assertApproxEqAbs(a.lpFee - b.lpFee, excess / 2, 1, "LPs get half the excess");
        assertEq(a.keeperCredit, a.keeperBounty, "keeper credit matches preview");

        console2.log("first liquidatable price (1e8)                 ", pFirst);
        console2.log("uncapped charge at that price                  ", uncappedCharge);
        console2.log("excess                                         ", excess);
        console2.log("extra charge paid by ratchet LONG (A - B)      ", a.charge - b.charge);
    }

    // ---------------------------------------------------------------------
    // 4. Counter-evidence / consistency
    // ---------------------------------------------------------------------

    /// @notice (a) A partial reduction recomputes the target on the remaining size and releases the excess
    ///         (CfdEnginePlanLib.sol:866-875, CfdEngineSettlementSidecar.sol:652-654), so the spec's reduction rule works
    ///         and a trader can recover the excess with a minimum-size partial close. (b) In the same kind of increase,
    ///         the open path releases the VPI rebate reserve when its target falls (CfdEnginePlanLib.sol:348-355) but
    ///         keeps the liquidation reserve above its target.
    function test_reductionReleasesExcess() public {
        uint256 snap = vm.snapshotState();

        // (a) Ratchet LONG, then a partial close of K lots at 0.90.
        _fundTrader(RATCHET, LONG_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.LONG, N_LOTS, LONG_MARGIN, false, P1);
        _mark(P2);
        _fill(RATCHET, CfdTypes.Side.LONG, K_LOTS, INCREASE_MARGIN, false, P2);
        uint256 reserveBeforeClose = clearinghouse.liquidationReserveAssets(RATCHET);
        assertGt(reserveBeforeClose, _target(N_LOTS + K_LOTS, P2), "ratchet excess present before the reduction");
        uint256 pledgeBeforeClose = clearinghouse.pnlPledgeAssets(RATCHET);

        _mark(P2);
        _fill(RATCHET, CfdTypes.Side.LONG, K_LOTS, 0, true, P2);
        uint256 reserveAfterClose = clearinghouse.liquidationReserveAssets(RATCHET);
        assertEq(reserveAfterClose, _target(N_LOTS, P2), "reduction recomputes the target and releases the excess");
        console2.log("===== (a) reduction path =====");
        console2.log("reserve before partial close (R1)              ", reserveBeforeClose);
        console2.log("reserve after closing 12 lots @0.90 (target)   ", reserveAfterClose);
        console2.log("released by the reduction                      ", reserveBeforeClose - reserveAfterClose);
        console2.log("pledge before / after partial close            ", pledgeBeforeClose, clearinghouse.pnlPledgeAssets(RATCHET));

        vm.revertToState(snap);
        _vpiReserveReleasedOnOpen();
    }

    /// @dev Part (b) of test_reductionReleasesExcess.
    function _vpiReserveReleasedOnOpen() internal {
        // (b) VPI reserve released on open while the liquidation reserve is ratcheted in the same fill.
        // OTHER opens SHORT, RATCHET opens LONG against it (skew-reducing -> negative VPI -> VPI rebate reserve).
        _fundTrader(OTHER, SHORT_DEPOSIT);
        _fill(OTHER, CfdTypes.Side.SHORT, N_LOTS, SHORT_MARGIN, false, P1);
        _fundTrader(RATCHET, LONG_DEPOSIT);
        _fill(RATCHET, CfdTypes.Side.LONG, N_LOTS, LONG_MARGIN, false, P1);
        uint256 vpiReserve0 = clearinghouse.vpiRebateReserveAssets(RATCHET);
        assertGt(vpiReserve0, 0, "VPI rebate reserve funded by the skew-reducing open");
        // OTHER closes, so RATCHET's LONG becomes the heavy side; its next increase pays positive VPI.
        _fill(OTHER, CfdTypes.Side.SHORT, N_LOTS, 0, true, P1);
        _mark(P2);
        uint256 increaseLots = 100;
        uint256 liqReserveBefore = clearinghouse.liquidationReserveAssets(RATCHET);
        _fill(RATCHET, CfdTypes.Side.LONG, increaseLots, INCREASE_MARGIN, false, P2);
        uint256 vpiReserve1 = clearinghouse.vpiRebateReserveAssets(RATCHET);
        uint256 liqReserveAfter = clearinghouse.liquidationReserveAssets(RATCHET);
        uint256 liqTarget = _target(N_LOTS + increaseLots, P2);
        (,,,,,, int256 vpiAccrued) = engine.positions(RATCHET);

        assertLt(vpiReserve1, vpiReserve0, "open path releases the VPI reserve when its target falls");
        assertEq(vpiReserve1, vpiAccrued < 0 ? uint256(-vpiAccrued) : 0, "VPI reserve == its recomputed target");
        assertEq(liqReserveAfter, liqReserveBefore, "same fill: liquidation reserve not released");
        assertGt(liqReserveAfter, liqTarget, "liquidation reserve above its target");

        console2.log("===== (b) open path: VPI reserve vs liquidation reserve (same fill) =====");
        console2.log("VPI reserve before / after +100 lots @0.90     ", vpiReserve0, vpiReserve1);
        console2.log("VPI reserve released on open                   ", vpiReserve0 - vpiReserve1);
        console2.log("liquidation reserve before / after             ", liqReserveBefore, liqReserveAfter);
        console2.log("liquidation reserve target after               ", liqTarget);
        console2.log("liquidation reserve excess kept                ", liqReserveAfter - liqTarget);
    }

    // ---------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------

    /// @dev Spec target (ACCOUNTING_SPEC.md:806-809; OpenAccountingLib.sol:141-152): max(minBounty, lots*price*bps/1e4).
    function _target(
        uint256 lots,
        uint256 price
    ) internal pure returns (uint256 t) {
        t = (lots * price * 10) / 10_000;
        if (t < 1e6) {
            t = 1e6;
        }
    }

    /// @dev Production Router FIFO: commit (test adapter -> production V3 commitOrder), then keeper executeOrder.
    function _fill(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        _tick();
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, lots * CfdTypes.SIZE_QUANTUM, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev Permissionless mark refresh through the Router.
    function _mark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Trader-only CfdEngine.addMargin tops the control up to the same pledge + reserve as the ratchet account.
    function _equalizeCollateral(
        address account,
        uint256 totalCollateral
    ) internal {
        uint256 current = clearinghouse.pnlPledgeAssets(account) + clearinghouse.liquidationReserveAssets(account);
        assertGt(totalCollateral, current, "control needs a top-up");
        vm.prank(account);
        engine.addMargin(account, totalCollateral - current);
        assertEq(
            clearinghouse.pnlPledgeAssets(account) + clearinghouse.liquidationReserveAssets(account),
            totalCollateral,
            "control equalized"
        );
    }

    /// @dev Production OrderRouter.executeLiquidation by a permissionless keeper.
    function _liquidate(
        address account,
        uint256 price
    ) internal returns (LiqOutcome memory o) {
        bytes[] memory data = _mockPythUpdateData(price);
        o.price = price;
        _recordPreview(o, account, price);
        _recordBefore(o, account);
        uint256 poolCashBefore = settlementToken.balanceOf(address(pool));
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);

        _executeLiquidation(account, data, o.writeOff > 0);

        (uint256 sizeAfter,,,,,,) = engine.positions(account);
        assertEq(sizeAfter, 0, "position liquidated");
        o.reserveAfter = clearinghouse.liquidationReserveAssets(account);
        o.traderFreeAfter = _freeSettlementAssets(account);
        o.traderSettlementAfter = clearinghouse.balanceAssets(account);
        o.poolCashIn = settlementToken.balanceOf(address(pool)) - poolCashBefore;
        o.keeperCredit = clearinghouse.balanceAssets(KEEPER) - keeperBefore;
    }

    function _recordPreview(
        LiqOutcome memory o,
        address account,
        uint256 price
    ) internal view {
        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(account, price);
        assertTrue(preview.liquidatable, "liquidatable");
        o.charge = preview.liquidationChargeAssets;
        o.keeperBounty = preview.keeperBountyAssets;
        o.lpFee = preview.lpLiquidationFeeAssets;
        o.equity = preview.equityAssets;
        // Write-off = price loss - claim consumed (0 here) - pledge consumed, where pledge consumed = seized - LP fee
        // (CfdEnginePlanLib.sol:1512-1521, :1526). Matches the PriceLossWrittenOff amount the sidecar emits.
        if (preview.pnlAssets < 0) {
            uint256 loss = uint256(-preview.pnlAssets);
            uint256 pledgeConsumed = preview.seizedCollateralAssets - preview.lpLiquidationFeeAssets;
            o.writeOff = loss > pledgeConsumed ? loss - pledgeConsumed : 0;
        }
        (uint256 size,,,,,,) = engine.positions(account);
        o.maintenance = ((size / CfdTypes.SIZE_QUANTUM) * price * 10) / 10_000;
    }

    function _recordBefore(
        LiqOutcome memory o,
        address account
    ) internal view {
        o.reserveBefore = clearinghouse.liquidationReserveAssets(account);
        o.pledgeBefore = clearinghouse.pnlPledgeAssets(account);
        o.traderFreeBefore = _freeSettlementAssets(account);
        o.traderSettlementBefore = clearinghouse.balanceAssets(account);
    }

    /// @dev Keeper calls the production OrderRouter.executeLiquidation. When the plan writes off price loss, the call
    ///      must emit PriceLossWrittenOff(account, ...) (CfdEngineSettlementSidecar.sol:858); checked with expectEmit
    ///      on the indexed account topic (recorded-log decoding is avoided: it does not fit the via-IR stack).
    function _executeLiquidation(
        address account,
        bytes[] memory data,
        bool expectWriteOff
    ) internal {
        if (expectWriteOff) {
            vm.expectEmit(true, false, false, false);
            emit PriceLossWrittenOff(account, 0);
        }
        vm.prank(KEEPER);
        router.executeLiquidation(account, data);
    }

    /// @dev At `price` the account is liquidatable (equity <= maintenance), but equity + excess, i.e. equity with the
    ///      excess classified as PnL pledge as ACCOUNTING_SPEC.md:810 requires, is still above maintenance.
    function _assertPrematureBand(
        address account,
        uint256 price,
        uint256 excess
    ) internal view returns (uint256 maintenance) {
        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(account, price);
        maintenance = ((N_LOTS + K_LOTS) * price * 10) / 10_000;
        assertTrue(preview.liquidatable, "liquidatable at first price");
        assertLe(preview.equityAssets, int256(maintenance), "equity <= maintenance");
        assertGt(preview.equityAssets + int256(excess), int256(maintenance), "spec equity (pledge + excess) > maintenance");
        console2.log("premature band: equity (pledge + PnL) at first liquidatable price");
        console2.logInt(preview.equityAssets);
        console2.log("  maintenance margin                           ", maintenance);
        console2.log("  equity + excess (spec classification)        ", uint256(preview.equityAssets + int256(excess)));
    }

    /// @dev Highest price at which a SHORT is liquidatable (SHORT loses as the price falls).
    function _firstLiquidatablePriceShort(
        address account
    ) internal view returns (uint256) {
        uint256 lo = DEEP_PRICE;
        uint256 hi = P2;
        require(engineLens.previewLiquidation(account, lo).liquidatable, "lo must be liquidatable");
        require(!engineLens.previewLiquidation(account, hi).liquidatable, "hi must be safe");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.previewLiquidation(account, mid).liquidatable) {
                lo = mid;
            } else {
                hi = mid;
            }
        }
        return lo;
    }

    /// @dev Lowest price at which a LONG is liquidatable (LONG loses as the price rises).
    function _firstLiquidatablePriceLong(
        address account
    ) internal view returns (uint256) {
        uint256 lo = P2;
        uint256 hi = 1.2e8;
        require(!engineLens.previewLiquidation(account, lo).liquidatable, "lo must be safe");
        require(engineLens.previewLiquidation(account, hi).liquidatable, "hi must be liquidatable");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.previewLiquidation(account, mid).liquidatable) {
                hi = mid;
            } else {
                lo = mid;
            }
        }
        return hi;
    }

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _logOutcome(
        string memory label,
        LiqOutcome memory o
    ) internal pure {
        console2.log("=====", label, "=====");
        console2.log("  liquidation price (1e8)                ", o.price);
        console2.log("  liquidation reserve before             ", o.reserveBefore);
        console2.log("  PnL pledge before                      ", o.pledgeBefore);
        console2.log("  pledge + reserve                       ", o.pledgeBefore + o.reserveBefore);
        console2.log("  equity (pledge + claim + PnL)          ", o.equity);
        console2.log("  maintenance margin                     ", o.maintenance);
        console2.log("  liquidation charge                     ", o.charge);
        console2.log("  keeper bounty / LP fee                 ", o.keeperBounty, o.lpFee);
        console2.log("  price loss written off                 ", o.writeOff);
        console2.log("  pool cash received                     ", o.poolCashIn);
        console2.log("  trader free settlement before / after  ", o.traderFreeBefore, o.traderFreeAfter);
        console2.log("  trader settlement before / after       ", o.traderSettlementBefore, o.traderSettlementAfter);
        console2.log("  reserve after                          ", o.reserveAfter);
    }

}
```

output:

```
Ran 6 tests for test/perps/poc/LiquidationReserveRatchet.t.sol:LiquidationReserveRatchetTest
[PASS] test_increaseKeepsReserveAboveTarget() (gas: 6626332)
Logs:
  ===== test_increaseKeepsReserveAboveTarget (SHORT, 1M notional, 10% fall) =====
  reserve after open of 10,000 lots @1.00 (R1)   1000000000
  recomputed target, 10,012 lots @0.90            901080000
  ratchet reserve after +12 lots @0.90            1000000000
  control reserve, 10,012 lots opened @0.90       901080000
  excess held as reserve instead of PnL pledge    98920000
  ratchet pledge before/after increase            118100099981 118198695272

[PASS] test_longLiquidation_extraCharge() (gas: 11295157)
Logs:
  premature band: equity (pledge + PnL) at first liquidatable price
  1004270282
    maintenance margin                            1004274424
    equity + excess (spec classification)         1103190282
  ===== A: RATCHET LONG (10,000 @1.00 then +12 @0.90) =====
    liquidation price (1e8)                 100307074
    liquidation reserve before              1000000000
    PnL pledge before                       4198695170
    pledge + reserve                        5198695170
    equity (pledge + claim + PnL)           1004270125
    maintenance margin                      1004274424
    liquidation charge                      1000000000
    keeper bounty / LP fee                  500000000 500000000
    price loss written off                  0
    pool cash received                      3694425045
    trader free settlement before / after   13899692000 14903962125
    trader settlement before / after        19098387170 14903962125
    reserve after                           0
  ===== B: CONTROL LONG (10,012 @0.90, same pledge + reserve) =====
    liquidation price (1e8)                 100307074
    liquidation reserve before              901080000
    PnL pledge before                       4297615170
    pledge + reserve                        5198695170
    equity (pledge + claim + PnL)           -98896809845
    maintenance margin                      1004274424
    liquidation charge                      901080000
    keeper bounty / LP fee                  450540000 450540000
    price loss written off                  98896809845
    pool cash received                      4748155170
    trader free settlement before / after   14034781426 14034781426
    trader settlement before / after        19233476596 14034781426
    reserve after                           0
  first liquidatable price (1e8)                  100307074
  uncapped charge at that price                   1004274424
  excess                                          98920000
  extra charge paid by ratchet LONG (A - B)       98920000

[PASS] test_negativeControl_increaseAtHigherPrice_reserveEqualsTarget() (gas: 4609086)
Logs:
  negative control: reserve after +12 lots @1.10 == target 1101320000

[PASS] test_reductionReleasesExcess() (gas: 14566470)
Logs:
  ===== (a) reduction path =====
  reserve before partial close (R1)               1000000000
  reserve after closing 12 lots @0.90 (target)    900000000
  released by the reduction                       100000000
  pledge before / after partial close             4198695170 4313518477
  ===== (b) open path: VPI reserve vs liquidation reserve (same fill) =====
  VPI reserve before / after +100 lots @0.90      499875036 491736571
  VPI reserve released on open                    8138465
  liquidation reserve before / after              1000000000 1000000000
  liquidation reserve target after                909000000
  liquidation reserve excess kept                 91000000

[PASS] test_shortBadDebtLiquidation_excessReturnedToTrader() (gas: 8490886)
Logs:
  ===== A: RATCHET (10,000 @1.00 then +12 @0.90) =====
    liquidation price (1e8)                 77000000
    liquidation reserve before              1000000000
    PnL pledge before                       118198695272
    pledge + reserve                        119198695272
    equity (pledge + claim + PnL)           -111957304851
    maintenance margin                      770924000
    liquidation charge                      770924000
    keeper bounty / LP fee                  385462000 385462000
    price loss written off                  111957304851
    pool cash received                      118584157272
    trader free settlement before / after   29899692000 30128768000
    trader settlement before / after        149098387272 30128768000
    reserve after                           0
  ===== B: CONTROL (10,012 @0.90, same pledge + reserve) =====
    liquidation price (1e8)                 77000000
    liquidation reserve before              901080000
    PnL pledge before                       118297615272
    pledge + reserve                        119198695272
    equity (pledge + claim + PnL)           -11858384881
    maintenance margin                      770924000
    liquidation charge                      770924000
    keeper bounty / LP fee                  385462000 385462000
    price loss written off                  11858384881
    pool cash received                      118683077272
    trader free settlement before / after   30034781324 30164937324
    trader settlement before / after        149233476596 30164937324
    reserve after                           0
  excess (A reserve - target)                     98920000
  trader free-settlement gain A - B               98920000
  pool cash received B - A (LP shortfall)         98920000

[PASS] test_shortRealisticGap_lpShortfallEqualsExcess() (gas: 7867786)
Logs:
  premature band: equity (pledge + PnL) at first liquidatable price
  883762256
    maintenance margin                            883765066
    equity + excess (spec classification)         982682256
  ===== RATCHET liquidated after a small gap =====
    liquidation price (1e8)                 88172331
    liquidation reserve before              1000000000
    PnL pledge before                       118198695272
    pledge + reserve                        119198695272
    equity (pledge + claim + PnL)           -99926879
    maintenance margin                      882781377
    liquidation charge                      882781377
    keeper bounty / LP fee                  441390688 441390689
    price loss written off                  99926879
    pool cash received                      118640085956
    trader free settlement before / after   29899692000 30016910633
    trader settlement before / after        149098387272 30016910633
    reserve after                           0
  first liquidatable price (1e8)                  88270582
  liquidation price after gap (1e8)               88172331
  gap below first liquidatable price (0.01 bps)   1113
  excess                                          98920000
  released reserve to trader (R1 - charge)        117218623
  of which spec-pledge excess (LP shortfall)      98920000

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 30.96ms (94.33ms CPU time)
```
