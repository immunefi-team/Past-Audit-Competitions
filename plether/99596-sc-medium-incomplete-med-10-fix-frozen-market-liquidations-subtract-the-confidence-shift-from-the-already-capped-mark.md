# \#99596 [SC-Medium] Incomplete MED-10 fix: frozen-market liquidations subtract the confidence shift from the already-capped mark

**Submitted on Nov 14th 2026 at 12:26:31 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99596
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Cecuro MED-10 found that when the neutral basket is above `CAP_PRICE`, SHORT accounts are liquidated below the cap-adjusted price, because the adverse confidence shift is subtracted from an already-capped number. The team marked it addressed:

> Single-account and batch liquidation pricing now apply the directional confidence adjustment to the raw, uncapped basket, then clamp the resulting liquidation price to CAP. [...] This removes the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark.

At a82d4fef the fix holds only while the oracle is live. During the scheduled oracle freeze, both liquidation entrypoints price from `engine.lastMarkPrice()`, which the Engine has already capped, and subtract the shift from that.

With MED-10's own inputs, frozen single and batch liquidation both return **199,950,000**: the pre-fix value MED-10 reported. For the same observation 20 seconds earlier, the live path returns the fixed **199,970,000**.

Any keeper can therefore liquidate a SHORT that is solvent at the correct price. At 1,000,000 USDG notional the trader loses **1,249.75 USDG**, of which **499.875 USDG** is credited to the keeper.

## Vulnerability Details

**Live path (fixed).** [`_updateLiquidationSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L531-L543) returns the uncapped `basket.price` as `rawPrice`. [`updateLiquidationPrice` and `updateLiquidationBatchPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L308-L343) shift it by side and clamp once. The helper's NatSpec reads: "Retains the pre-cap price for adverse liquidation shifts while keeping the neutral snapshot capped."

**Frozen path (not fixed).** When `policy.oracleFrozen` is set, [`_resolveCurrentBasket` returns `_frozenBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756) before any payload is parsed:

```solidity
function _frozenBasket(PriceMode mode, uint256 maxStaleness) internal view returns (BasketPrice memory basket) {
    basket.price = engine.lastMarkPrice();
    basket.publishTime = engine.lastMarkTime();
    ...
    // Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
    basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
}
```

The flow is:

1. [`CfdEngine.updateMarkPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1464-L1477) stores `min(price, CAP_PRICE)`.
2. For an above-cap basket, `_updateLiquidationSnapshot` therefore labels CAP as `rawPrice`.
3. The SHORT liquidation price becomes `CAP - shift`. That is the cap-then-shift arithmetic MED-10 describes.

The Oracle still holds the uncapped accepted basket ([`_acceptedBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L795-L799)), but the frozen branch does not read it.

The release settings are `CAP_PRICE` 200,000,000, `basketMaxConfidenceRatioBps` 10 and `adverseConfidenceMultiplierBps` 2,500. With MED-10's example basket (200,020,000, aggregate confidence 200,000):

| Same observation | SHORT liquidation price |
|---|---:|
| live, 21:59:50 UTC | min(200,020,000 − 50,000, CAP) = **199,970,000** |
| frozen, 22:00:10 UTC | 200,000,000 − 50,000 = **199,950,000** |

- For a basket of 200,100,000, the live SHORT price is 200,000,000 and the frozen price is again 199,950,000.
- Both permissionless entrypoints ([`OrderRouter`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L486-L519), [`OrderLiquidationBatchLogic`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L705-L790)) use the frozen price: single liquidation passes `snapshot.price`, and batch liquidation passes `snapshot.shortPrice` to every SHORT.
- A fresh Pyth payload changes nothing, because frozen resolution returns before parsing.

**How the frozen branch was missed.**

1. At the audited commit, frozen liquidation priced from the stored Pyth feeds.
2. The fix commit linked from MED-10 fuzzes `frozen = true` with raw baskets up to 300,000,000 and expects shift-before-cap ([`testFuzz_Liquidation_SingleAndBatchMatchRawBasketPolicy`](https://github.com/Plether-Fi/plether-core/blob/006c818d364282943e2063eab52ab0c89181304c/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L584-L623)).
3. Meanwhile, [PR #120](https://github.com/Plether-Fi/plether-core/commit/819192182833597641504b38cce7751bbcacecb8) switched frozen pricing to replaying the Engine mark.
4. In the merged fix ([#124](https://github.com/Plether-Fi/plether-core/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15)), the frozen branch of that test uses a stored mark of 100,000,000 ([test at a82d4fef](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L690-L737)), so the capped case is no longer exercised.

The [whitepaper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1069-L1076) says frozen liquidations reuse the Engine's last mark with a synthesized confidence shift. It does not set the order of that shift and the cap, and that order is exactly what the MED-10 fix changed. The trigger is MED-10's own precondition: the neutral basket is above `CAP_PRICE`.

## Impact Details

**Severity: Medium** (MED-10's severity). A permissionless keeper liquidates a solvent SHORT during the freeze, and the trader loses funds. The PoC uses release risk parameters (3% FAD margin, 10 bp liquidation charge, 50% keeper share) and a SHORT with 1,000,000 USDG notional at CAP, on both entrypoints.

| Same frozen state | Correct price (CAP) | Frozen price |
|---|---:|---:|
| Liquidation price | 200,000,000 | 199,950,000 |
| Equity | 30,100.00 | 29,850.00 |
| 3% FAD requirement | 30,000.00 | 29,992.50 |
| Result | solvent by 100.00 | liquidatable |

- Live liquidation of this account reverts with `CfdEngine__PositionIsSolvent`. After the freeze, single and batch liquidation both close it at 199,950,000 with empty update data.
- The trader loses 250.00 (price) + 999.75 (liquidation charge) = **1,249.75 USDG** against the correct state. Of this, **499.875 USDG** is credited to the keeper.
- The loss scales with notional. Every SHORT liquidated during the freeze also settles up to 2.5 bp of notional below the live price for the same observation.

**Recommendation:** in frozen mode, compute the liquidation shift from the uncapped basket behind the stored mark, then clamp once. For example, use `_acceptedBasket.price` when `_acceptedBasket.publishTime == engine.lastMarkTime()`, or store the uncapped price together with the Engine mark. Keep the neutral mark capped. Add frozen single and batch tests with the stored mark at CAP.

## References

- [Cecuro AI audit, MED-10 and team comment (pages 29-30)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [PletherOracle.sol L308-L343: live shift-then-cap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L308-L343)
- [PletherOracle.sol L531-L543: `rawPrice = basket.price` and NatSpec](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L531-L543)
- [PletherOracle.sol L729-L743: `_frozenBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743)
- [PletherOracle.sol L749-L756: frozen branch returns before parsing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756)
- [PletherOracle.sol L795-L799: `_acceptedBasket` retained](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L795-L799)
- [PletherOracle.sol L960-L972: shift helpers](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L960-L972)
- [CfdEngine.sol L1464-L1477: Engine mark capped before storage](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1464-L1477)
- [OrderRouter.sol L486-L519: public liquidation entrypoints](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L486-L519)
- [OrderLiquidationBatchLogic.sol L705-L790: snapshot consumption](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L705-L790)
- [PletherOracle.t.sol L690-L737: merged frozen test branch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L690-L737)
- [WHITEPAPER.md L1069-L1076: frozen regime](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1069-L1076)
- [Fix commit's frozen fuzz test (006c818d)](https://github.com/Plether-Fi/plether-core/blob/006c818d364282943e2063eab52ab0c89181304c/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L584-L623)
- [Frozen Engine-mark replay (#120)](https://github.com/Plether-Fi/plether-core/commit/819192182833597641504b38cce7751bbcacecb8)
- [Fix merged (#124)](https://github.com/Plether-Fi/plether-core/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15)

## Proof of Concept

# Proof of Concept

Place the test below at `packages/perps/test/poc/FrozenLiquidationCapOrdering.t.sol` (commit a82d4fefb02768e5b3bddcc97dee371eee24fea8) and run, from the repository root:

```sh
forge test --root packages/perps --match-path test/poc/FrozenLiquidationCapOrdering.t.sol -vvv
```

Setup:
- The test uses the repository's unmodified `BasePerpTest`.
- Pyth and USDG are the suite's standard test doubles.
- The release confidence settings are applied through the owner's 72-hour timelocked router configuration.
- Only public commit, mark-refresh and liquidation entrypoints are used.

## Test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Frozen SHORT liquidation subtracts the adverse confidence shift from the already-capped Engine mark.
/// Positions, the mark refresh and every liquidation go through public user or keeper entrypoints.
/// Pyth and USDG are the suite's standard test doubles; no protocol contract is impersonated.
contract FrozenLiquidationCapOrderingTest is BasePerpTest {

    address internal constant TRADER = address(0xA11CE);
    address internal constant LIVE_PROBE = address(0xB0B);
    address internal constant FROZEN_PROBE_SINGLE = address(0xC0C);
    address internal constant FROZEN_PROBE_BATCH = address(0xD0D);
    address internal constant KEEPER = address(0x4EE9E7);

    uint256 internal constant ENTRY_PRICE = 199_980_000;
    uint256 internal constant RAW_PRICE_ABOVE_CAP = 200_100_000;
    uint64 internal constant LIVE_CONFIDENCE = 200_000;
    uint256 internal constant FROZEN_SHORT_PRICE = 199_950_000;

    // The numeric example of Cecuro MED-10: raw basket 200,020,000, aggregate confidence 200,000, multiplier 2,500.
    uint256 internal constant MED10_RAW_PRICE = 200_020_000;
    uint64 internal constant MED10_CONFIDENCE = 200_000;
    uint256 internal constant MED10_FIXED_SHORT_PRICE = 199_970_000;
    uint256 internal constant MED10_PRE_FIX_SHORT_PRICE = 199_950_000;

    // 5,000 lots at the $2 CAP is exactly $1,000,000 notional.
    uint256 internal constant POSITION_SIZE = 500_000e18;
    uint256 internal constant CAP_NOTIONAL_ASSETS = 1_000_000e6;
    uint256 internal constant TRADER_FUNDING = 100_000e6;
    uint256 internal constant OPEN_MARGIN = 5_000e6;

    // Small public probes reveal the applied liquidation price without materially changing market skew.
    uint256 internal constant PROBE_SIZE = 10_000e18;
    uint256 internal constant PROBE_FUNDING = 2_000e6;
    uint256 internal constant PROBE_MARGIN = 100e6;

    // Friday 2024-12-06: FAD begins at 21:30 UTC and the oracle freezes at 22:00 UTC.
    uint256 internal constant OPEN_TIME = 1_733_433_600; // Thursday 2024-12-05 21:20:00 UTC
    uint256 internal constant LAST_LIVE_TIME = 1_733_522_390; // Friday 2024-12-06 21:59:50 UTC
    uint256 internal constant FROZEN_TIME = 1_733_522_410; // Friday 2024-12-06 22:00:10 UTC

    bytes32 internal constant POSITION_LIQUIDATED_TOPIC =
        keccak256("PositionLiquidated(address,uint8,uint256,uint256,uint256)");

    struct LiquidationObservation {
        bool found;
        CfdTypes.Side side;
        uint256 size;
        uint256 price;
        uint256 keeperBountyAssets;
    }

    struct PreparedShortState {
        uint256 liveLiquidationPrice;
        int256 correctCapEquityAssets;
        int256 frozenQuoteEquityAssets;
        uint256 correctCapRequirementAssets;
        uint256 frozenQuoteRequirementAssets;
        uint256 priceOrderingLossAssets;
        uint256 liquidationChargeAssets;
        uint256 keeperBountyAssets;
        uint256 avoidableTraderLossAssets;
    }

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
        return 10_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
    }

    /// @notice MED-10's own inputs. Live liquidation returns the fixed price; twenty seconds later, with no new
    ///         observation, the scheduled freeze prices the same observation at MED-10's pre-fix value.
    function test_Med10ExampleIsPricedAtPreFixValueAfterFreeze() public {
        _go(OPEN_TIME);
        _refreshLiveMark(ENTRY_PRICE, 0);
        _openPublicPosition(LIVE_PROBE, CfdTypes.Side.SHORT, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);
        _openPublicPosition(FROZEN_PROBE_SINGLE, CfdTypes.Side.SHORT, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);
        _openPublicPosition(FROZEN_PROBE_BATCH, CfdTypes.Side.SHORT, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);

        _go(LAST_LIVE_TIME);
        _refreshLiveMark(MED10_RAW_PRICE, MED10_CONFIDENCE);
        assertFalse(engine.isOracleFrozen(), "last live tick precedes freeze");
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "Engine stores the capped neutral mark");
        uint64 markTime = engine.lastMarkTime();

        LiquidationObservation memory live = _publicSingleLiquidation(LIVE_PROBE);
        assertTrue(live.found, "live SHORT liquidation event");
        assertEq(live.price, MED10_FIXED_SHORT_PRICE, "live path shifts the raw basket, then caps");

        _go(FROZEN_TIME);
        assertTrue(engine.isOracleFrozen(), "real Friday freeze reached");
        LiquidationObservation memory frozenSingle = _publicSingleLiquidation(FROZEN_PROBE_SINGLE);
        LiquidationObservation memory frozenBatch = _publicBatchLiquidation(FROZEN_PROBE_BATCH);
        assertTrue(frozenSingle.found && frozenBatch.found, "frozen SHORT liquidation events");
        assertEq(frozenSingle.price, MED10_PRE_FIX_SHORT_PRICE, "frozen single path: cap first, then shift");
        assertEq(frozenBatch.price, MED10_PRE_FIX_SHORT_PRICE, "frozen batch path: cap first, then shift");
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "no new mark was installed");
        assertEq(engine.lastMarkTime(), markTime, "same observation in both regimes");

        console2.log("MED-10 example (raw 200020000, confidence 200000, multiplier 2500)");
        console2.log("  Engine mark (capped)", engine.lastMarkPrice());
        console2.log("  live SHORT liquidation price", live.price);
        console2.log("  frozen SHORT liquidation price, single", frozenSingle.price);
        console2.log("  frozen SHORT liquidation price, batch", frozenBatch.price);
    }

    function test_PublicSingleLiquidationUsesCappedMarkBeforeShortConfidenceShift() public {
        PreparedShortState memory state = _prepareHealthyShortAtCap(false);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);

        LiquidationObservation memory actual = _publicSingleLiquidation(TRADER);

        _assertShortLiquidationAndLog("single", state, actual, keeperBefore);
    }

    function test_PublicBatchLiquidationUsesCappedMarkBeforeShortConfidenceShift() public {
        PreparedShortState memory state = _prepareHealthyShortAtCap(true);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);

        LiquidationObservation memory actual = _publicBatchLiquidation(TRADER);

        _assertShortLiquidationAndLog("batch", state, actual, keeperBefore);
    }

    function test_FrozenLongLiquidationRemainsCapped() public {
        _go(OPEN_TIME);
        _refreshLiveMark(ENTRY_PRICE, 0);
        _openPublicPosition(LIVE_PROBE, CfdTypes.Side.LONG, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);
        _openPublicPosition(TRADER, CfdTypes.Side.LONG, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);

        _go(LAST_LIVE_TIME);
        _refreshLiveMark(RAW_PRICE_ABOVE_CAP, LIVE_CONFIDENCE);
        LiquidationObservation memory live = _publicSingleLiquidation(LIVE_PROBE);
        assertTrue(live.found, "live LONG liquidation event");
        assertEq(uint256(live.side), uint256(CfdTypes.Side.LONG), "live LONG side");
        assertEq(live.price, CAP_PRICE, "live LONG raw-plus-shift is capped");

        _go(FROZEN_TIME);
        assertTrue(engine.isOracleFrozen(), "real Friday freeze reached");
        LiquidationObservation memory frozen = _publicBatchLiquidation(TRADER);
        assertTrue(frozen.found, "frozen LONG liquidation event");
        assertEq(uint256(frozen.side), uint256(CfdTypes.Side.LONG), "frozen LONG side");
        assertEq(frozen.price, CAP_PRICE, "frozen LONG capped-mark-plus-shift remains capped");

        console2.log("LONG control");
        console2.log("  live public LONG liquidation price", live.price);
        console2.log("  frozen public LONG liquidation price", frozen.price);
    }

    function _prepareHealthyShortAtCap(
        bool useBatchPath
    ) internal returns (PreparedShortState memory state) {
        _go(OPEN_TIME);
        _refreshLiveMark(ENTRY_PRICE, 0);
        _openPublicPosition(TRADER, CfdTypes.Side.SHORT, POSITION_SIZE, OPEN_MARGIN, TRADER_FUNDING);
        _openPublicPosition(LIVE_PROBE, CfdTypes.Side.SHORT, PROBE_SIZE, PROBE_MARGIN, PROBE_FUNDING);

        // An above-cap mock Pyth basket is installed through the public mark-refresh entrypoint.
        _go(LAST_LIVE_TIME);
        _refreshLiveMark(RAW_PRICE_ABOVE_CAP, LIVE_CONFIDENCE);
        assertTrue(engine.isFadWindow(), "last live tick is in FAD");
        assertFalse(engine.isOracleFrozen(), "last live tick precedes freeze");
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "Engine publicly stores the capped neutral mark");

        // Public live liquidation exposes the fixed raw-before-cap SHORT price in the Engine event.
        LiquidationObservation memory live =
            useBatchPath ? _publicBatchLiquidation(LIVE_PROBE) : _publicSingleLiquidation(LIVE_PROBE);
        assertTrue(live.found, "live SHORT liquidation event");
        assertEq(uint256(live.side), uint256(CfdTypes.Side.SHORT), "live SHORT side");
        assertEq(live.price, CAP_PRICE, "live SHORT shifts raw basket before final cap");
        state.liveLiquidationPrice = live.price;

        // Park the victim just above the 3% FAD boundary at the correct CAP price.
        uint256 requiredFadMarginAssets = CAP_NOTIONAL_ASSETS * 300 / 10_000;
        ICfdEngineTypes.LiquidationPreview memory capPreview = engineLens.previewLiquidation(TRADER, CAP_PRICE);
        int256 targetEquity = int256(requiredFadMarginAssets + 100e6);
        if (capPreview.equityAssets < targetEquity) {
            vm.prank(TRADER);
            engine.addMargin(TRADER, uint256(targetEquity - capPreview.equityAssets));
        }
        capPreview = engineLens.previewLiquidation(TRADER, CAP_PRICE);
        assertFalse(capPreview.liquidatable, "victim is healthy at the corrected live price");

        // The real public liquidation path also rejects this victim while the raw basket is still available.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, new bytes[](0));

        _go(FROZEN_TIME);
        assertTrue(engine.isOracleFrozen(), "real Friday freeze reached");

        // Compare both prices in the same frozen state so carry and the 3% margin regime are identical.
        ICfdEngineTypes.LiquidationPreview memory correct = engineLens.previewLiquidation(TRADER, CAP_PRICE);
        ICfdEngineTypes.LiquidationPreview memory faulty =
            engineLens.previewLiquidation(TRADER, FROZEN_SHORT_PRICE);
        assertFalse(correct.liquidatable, "same frozen account is healthy at shift-before-cap price");
        assertTrue(faulty.liquidatable, "capped-mark-before-shift quote flips eligibility");

        uint256 capNotionalAssets = (POSITION_SIZE / 1e20) * CAP_PRICE;
        assertEq(capNotionalAssets, CAP_NOTIONAL_ASSETS, "position is exactly $1M notional at CAP");

        state.correctCapEquityAssets = correct.equityAssets;
        state.frozenQuoteEquityAssets = faulty.equityAssets;
        state.correctCapRequirementAssets = CAP_NOTIONAL_ASSETS * 300 / 10_000;
        state.frozenQuoteRequirementAssets =
            ((POSITION_SIZE / 1e20) * FROZEN_SHORT_PRICE) * 300 / 10_000;
        state.priceOrderingLossAssets = uint256(correct.equityAssets - faulty.equityAssets);
        state.liquidationChargeAssets = faulty.liquidationChargeAssets;
        state.keeperBountyAssets = faulty.keeperBountyAssets;
        state.avoidableTraderLossAssets = state.priceOrderingLossAssets + state.liquidationChargeAssets;

        assertEq(state.priceOrderingLossAssets, 250e6, "$250 loss from the 50,000 price error");
        assertEq(state.liquidationChargeAssets, 999_750_000, "$999.75 avoidable liquidation charge");
        assertEq(state.keeperBountyAssets, 499_875_000, "$499.875 permissionless keeper gain");
        assertEq(state.avoidableTraderLossAssets, 1_249_750_000, "$1,249.75 direct avoidable trader loss");
    }

    function _assertShortLiquidationAndLog(
        string memory path,
        PreparedShortState memory state,
        LiquidationObservation memory actual,
        uint256 keeperBefore
    ) internal view {
        assertTrue(actual.found, "public Router must emit the liquidation price");
        assertEq(uint256(actual.side), uint256(CfdTypes.Side.SHORT), "liquidated side");
        assertEq(actual.size, POSITION_SIZE, "liquidated position has $1M notional at CAP");
        assertEq(actual.price, FROZEN_SHORT_PRICE, "frozen Router shifts the already-capped mark");
        assertEq(actual.keeperBountyAssets, state.keeperBountyAssets, "event keeper gain");
        assertEq(clearinghouse.balanceAssets(KEEPER) - keeperBefore, state.keeperBountyAssets, "keeper credited");

        (uint256 remainingSize,,,,,,) = engine.positions(TRADER);
        assertEq(remainingSize, 0, "false liquidation removes the healthy position");

        console2.log(path);
        console2.log("  position notional at CAP (USDG atoms)", CAP_NOTIONAL_ASSETS);
        console2.log("  live public SHORT liquidation price", state.liveLiquidationPrice);
        console2.log("  frozen public SHORT liquidation price", actual.price);
        console2.log("  equity at correct CAP price", state.correctCapEquityAssets);
        console2.log("  FAD requirement at correct CAP price", state.correctCapRequirementAssets);
        console2.log("  equity at frozen Router price", state.frozenQuoteEquityAssets);
        console2.log("  FAD requirement at frozen Router price", state.frozenQuoteRequirementAssets);
        console2.log("  loss caused by price ordering (USDG atoms)", state.priceOrderingLossAssets);
        console2.log("  avoidable liquidation charge (USDG atoms)", state.liquidationChargeAssets);
        console2.log("  permissionless keeper gain (USDG atoms)", state.keeperBountyAssets);
        console2.log("  direct trader loss versus correct healthy state", state.avoidableTraderLossAssets);
    }

    function _openPublicPosition(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 funding
    ) internal {
        _fundTrader(account, funding);
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, false);
        bytes[] memory openingTick = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(orderId, openingTick);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public position open");
    }

    function _publicSingleLiquidation(
        address account
    ) internal returns (LiquidationObservation memory observation) {
        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeLiquidation(account, new bytes[](0));
        observation = _liquidationEvent(vm.getRecordedLogs(), account);
    }

    function _publicBatchLiquidation(
        address account
    ) internal returns (LiquidationObservation memory observation) {
        address[] memory accounts = new address[](1);
        accounts[0] = account;
        vm.recordLogs();
        vm.prank(KEEPER);
        uint256 nextIndex = router.executeLiquidationBatch(accounts, new bytes[](0));
        assertEq(nextIndex, 1, "batch attempted the account");
        observation = _liquidationEvent(vm.getRecordedLogs(), account);
    }

    function _liquidationEvent(
        Vm.Log[] memory logs,
        address account
    ) internal view returns (LiquidationObservation memory observation) {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter != address(engine) || logs[i].topics.length < 2
                    || logs[i].topics[0] != POSITION_LIQUIDATED_TOPIC
                    || address(uint160(uint256(logs[i].topics[1]))) != account
            ) {
                continue;
            }
            observation.found = true;
            (observation.side, observation.size, observation.price, observation.keeperBountyAssets) =
                abi.decode(logs[i].data, (CfdTypes.Side, uint256, uint256, uint256));
            return observation;
        }
    }

    function _refreshLiveMark(
        uint256 price,
        uint64 confidence
    ) internal {
        uint64 publishTime = uint64(vm.getBlockTimestamp());
        bytes[] memory data = _tick(price, confidence, publishTime, publishTime - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _tick(
        uint256 price,
        uint64 confidence,
        uint64 publishTime,
        uint64 previousPublishTime
    ) internal returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        baseMockPyth.setAllPrices(ids, int64(uint64(price)), confidence, int32(-8), uint256(publishTime));
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(int64(uint64(price)), confidence, int32(-8), publishTime, previousPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _go(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

}
```

## Output

```text
Ran 4 tests for test/poc/FrozenLiquidationCapOrdering.t.sol:FrozenLiquidationCapOrderingTest
[PASS] test_FrozenLongLiquidationRemainsCapped() (gas: 5288736)
Logs:
  LONG control
    live public LONG liquidation price 200000000
    frozen public LONG liquidation price 200000000

[PASS] test_Med10ExampleIsPricedAtPreFixValueAfterFreeze() (gas: 7643124)
Logs:
  MED-10 example (raw 200020000, confidence 200000, multiplier 2500)
    Engine mark (capped) 200000000
    live SHORT liquidation price 199970000
    frozen SHORT liquidation price, single 199950000
    frozen SHORT liquidation price, batch 199950000

[PASS] test_PublicBatchLiquidationUsesCappedMarkBeforeShortConfidenceShift() (gas: 6218888)
Logs:
  batch
    position notional at CAP (USDG atoms) 1000000000000
    live public SHORT liquidation price 200000000
    frozen public SHORT liquidation price 199950000
    equity at correct CAP price 30100000000
    FAD requirement at correct CAP price 30000000000
    equity at frozen Router price 29850000000
    FAD requirement at frozen Router price 29992500000
    loss caused by price ordering (USDG atoms) 250000000
    avoidable liquidation charge (USDG atoms) 999750000
    permissionless keeper gain (USDG atoms) 499875000
    direct trader loss versus correct healthy state 1249750000

[PASS] test_PublicSingleLiquidationUsesCappedMarkBeforeShortConfidenceShift() (gas: 6210010)
Logs:
  single
    position notional at CAP (USDG atoms) 1000000000000
    live public SHORT liquidation price 200000000
    frozen public SHORT liquidation price 199950000
    equity at correct CAP price 30100000000
    FAD requirement at correct CAP price 30000000000
    equity at frozen Router price 29850000000
    FAD requirement at frozen Router price 29992500000
    loss caused by price ordering (USDG atoms) 250000000
    avoidable liquidation charge (USDG atoms) 999750000
    permissionless keeper gain (USDG atoms) 499875000
    direct trader loss versus correct healthy state 1249750000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 22.55ms (46.88ms CPU time)

Ran 1 test suite in 30.27ms (22.55ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```
