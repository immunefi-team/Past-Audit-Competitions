# \#97893 [SC-Low] Order execution applies the adverse confidence shift to the already-CAP-clamped price (execution/liquidation parity break)

**Submitted on Nov 3rd 2026 at 19:14:14 UTC by @jacku705 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97893
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Program: Plether — Immunefi Audit Competition ($80,000 USDC pool)
Severity claimed: Low
Scope: packages/perps/src/ — PletherOracle.sol, frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 (immunefi-team/audit-comp-plether)
PoC: packages/perps/test/competition/Lane1.t.sol, tests test_C1_* (full source in the Proof of Concept)

## Summary

The two pricing paths that apply the protocol's adverse confidence shift disagree about the order of operations whenever the raw basket price exceeds CAP_PRICE ($2.00). The liquidation path shifts the raw basket price and then caps the result — the ordering Cecuro MED-10's fix established. The order-execution path caps the basket price first and then applies the adverse shift to the capped price (only upward results are re-clamped). Downward-shift request types (SHORT closes, LONG opens) therefore fill below the policy price whenever the raw basket trades above the cap, while the identical economic exit via liquidation prices at the policy price. Under a governance-valid extreme confidence policy the same ordering makes downward order executions revert outright (PletherOracle__ZeroExecutionPrice) while liquidation of the same basket still executes — a liveness asymmetry between the two paths.

## Root cause

- Execution path: PletherOracle._updateOrderExecutionPrice (src/PletherOracle.sol:545) builds its snapshot via _snapshotFromBasket (src/PletherOracle.sol:878), which sets snapshot.price = _clampToCap(basket.price) (src/PletherOracle.sol:887) — the raw basket is clamped to CAP_PRICE before any shift. _orderExecutionPrice (src/PletherOracle.sol:925) then applies _adverseOrderPrice (src/PletherOracle.sol:905) to that already-capped price; the final _clampToCap on the result (src/PletherOracle.sol:940) only re-bounds upward shifts, so a downward shift leaks below the policy price by exactly the amount the raw basket exceeded the cap (bounded by the shift size).
- Liquidation path (the fixed sibling): updateLiquidationPrice computes snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence)) (src/PletherOracle.sol:342) — shift the raw price first, cap second. The batch liquidation path does the same (src/PletherOracle.sol:316–317).
- The batch order path (updateBatchOrderExecutionPrice, src/PletherOracle.sol:245) shares the execution path's capped-base shift, including through its cache-reuse branch (proven in the PoC).

## Impact (proven by PoC)

At the Cecuro-era policy numbers (aggregate confidence 200,000; adverse multiplier 2,500 bps → shift 50,000; raw basket 200,020,000 against CAP_PRICE 200,000,000):

- SHORT close via order execution fills at 199,950,000.
- The identical exit via liquidation prices at 199,970,000 (min(200,020,000 − 50,000, 200,000,000)).
- Proven gap: 20,000 price units on the same basket, same configuration — the execution path double-counts the above-cap excess against the trader. The gap is the raw basket's excess over the cap (capped at the shift size), so it scales with how far the basket trades above $2.00 and with the governance-set confidence policy.
- Upward shifts are unaffected: a LONG close at the same basket lands exactly on the cap (200,000,000) in both paths — the leak is one-sided, against downward-shift request types.
- Extreme-policy leg: under a governance-valid policy at the confidence gate's edge (ratio 3,333 bps × multiplier 30,000 bps; product 99,990,000 < 100,000,000; raw basket 300,000,000; shift 299,970,000), the downward order execution reverts PletherOracle__ZeroExecutionPrice because the shift exceeds the capped price, while liquidation of the same basket executes at 30,000. Order execution is bricked in a regime where liquidation still works.

## Severity justification (judge-narrowed)

Claimed Low, honestly bounded:

- The parity break arms only in a stress regime: the raw basket must trade above the $2.00 cap (≈2× the ~0.96 level the protocol's fixtures use). At the Cecuro-era policy numbers the per-trade leakage is 20,000 units on a ~200,000,000 price (≈1 bp of price); it grows only if governance raises the confidence policy or the basket trades further above the cap.
- No funds are stolen and no insolvency results; the harm is a systematic, one-sided execution-price leak against downward-shift traders plus the extreme-policy liveness asymmetry.
- A design-intent dispute is foreseeable and is disclosed rather than omitted: the execution path's documentation describes "capped and then shifted" behaviour. The submission's claim is narrower than "the execution path is wrong in isolation": the two documented policies for the same adverse-shift concept produce different prices for the same exit, and the liquidation side's ordering is the one the project itself adopted when fixing MED-10. Graded Medium only if governance has actually raised the confidence policy in production; on the evidence at this commit, Low is the defensible grade.

## Disclosure regarding the duplicate baseline

Cecuro MED-10 ("Liquidator can liquidate shorts below cap-adjusted mark") is a published baseline finding — about the liquidation path, and its fix (shift-then-cap) is present and PoC-verified at this commit. This report does not re-claim MED-10. The claim is the residual of the same root cause in the sibling path the fix did not touch: the order-execution path (single and batch, including cache reuse) still applies cap-then-shift. The PoC's parity test prices the identical basket through both paths at this commit and measures the divergence directly. No other baseline item (SC Audit Studio report, 2026-10-15 automated scan, collaborative audit, Cantina CLI review — see the competition workspace baseline) covers the execution path's shift ordering.

## Reproduction

Run from a clean working directory:

forge test --root packages/perps --match-path 'test/competition/Lane1.t.sol' --threads 1 -vv

The six test_C1_* tests are this report's PoC (four on the full-stack BasePerpTest harness, two standalone against the oracle with the spec mocks). The file's remaining tests are unrelated kill/regression controls from the same lane (production-basket precision, inverted-confidence gate, cache-forgery consumer check, market-calendar boundaries) and are not part of this report.

## Proof of concept — verified results

Re-run by the drafting coordinator on 4 Nov 2026 (forge 1.8.6, repo profile solc 0.8.35 / via_ir / optimizer 200, --threads 1), reproducing the 2 Nov 2026 parent verification (10/10 PASS) exactly:

[PASS] test_C1_BatchAndCacheReuse_AboveCap_SameCappedShift() (gas: 669288)
[PASS] test_C1_LongClose_AboveCap_UpwardShiftReclamped() (gas: 548086)
[PASS] test_C1_LongOpen_AboveCap_ExecutionShiftsCappedPrice() (gas: 548240)
[PASS] test_C1_ShortClose_AboveCap_ExecutionShiftsCappedPrice() (gas: 548533)
[PASS] test_C1_ExtremeValidPolicy_ExecutionReverts_LiquidationExecutes() (gas: 1203500)
[PASS] test_C1_LiquidationVsExecution_AboveCap_ParityBreak() (gas: 1199545)
Ran 2 test suites: 10 tests passed, 0 failed, 0 skipped (10 total tests)

Key assertions: SHORT close fills at 199,950,000 while liquidation of the identical basket prices at 199,970,000 (liq.price - exec.price == 20,000); the batch path and its cache reuse repeat 199,950,000; under the extreme valid policy the execution reverts PletherOracle__ZeroExecutionPrice while liquidation executes at 30,000.

## Suggested remediation

Apply the shift to the raw basket price in the execution path and clamp afterwards — i.e. price order executions as _clampToCap(_adverseOrderPrice(request, basket.price, confidence)) on the unclamped basket — matching the liquidation path's established ordering. The snapshot's neutral markPrice should remain capped, as it is today.

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {
    PletherOracleEngineMock,
    PletherOracleVaultMock
} from "../perps/spec/oracle/PletherOracleMocks.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

/// @dev Exposes the internal calendar library for boundary verification.
contract Lane1CalendarHarness {
    function status(uint256 ts) external pure returns (bool fad, bool frozen) {
        return MarketCalendarLib.marketStatus(ts, false, false, 0);
    }
}

/// @title Lane L1 — index basket pricing (PletherOracle)
/// @notice Candidate C1 is exercised on the real BasePerpTest stack (production-shaped
///         CfdEngine + LegacyOrderRouterHarness + PletherOracle, CAP_PRICE = 2e8).
contract Lane1HarnessTest is BasePerpTest {

    function _applyOraclePolicy(uint256 multiplier, uint256 ratioBps) internal {
        vm.prank(address(router));
        pletherOracle.applyConfig(
            IPletherOracle.OracleConfig({
                orderExecutionStalenessLimit: 60,
                liquidationStalenessLimit: 15,
                basketMaxConfidenceRatioBps: ratioBps,
                orderSettlementWindow: 15,
                maxComponentPublishTimeDivergence: 5,
                adverseConfidenceMultiplierBps: multiplier
            })
        );
    }

    function _historicalBasket(uint256 price, uint64 conf, uint64 commitTime)
        internal
        returns (bytes[] memory data)
    {
        uint256 publish = uint256(commitTime) + 5;
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), conf, int32(-8), publish, commitTime);
        data = new bytes[](1);
        data[0] = hex"01";
    }

    function _request(uint64 commitTime, CfdTypes.Side side, bool isClose)
        internal
        pure
        returns (IPletherOracle.OrderExecutionRequest memory)
    {
        return IPletherOracle.OrderExecutionRequest({
            commitTime: commitTime, targetPrice: 0, side: side, isClose: isClose, revertOnHistoricalUnavailable: true
        });
    }

    /// C1: raw basket 200_020_000 (> CAP 200_000_000), aggregate confidence 200_000,
    /// multiplier 2500 bps -> adverse shift 50_000 (the Cecuro MED-10 numbers).
    /// Shift-then-cap policy price for a downward shift: min(200_020_000 - 50_000, CAP) = 199_970_000.
    /// The order-execution path instead shifts the already-capped price: 199_950_000.
    function test_C1_ShortClose_AboveCap_ExecutionShiftsCappedPrice() public {
        vm.warp(SETUP_TIMESTAMP + 1000);
        _applyOraclePolicy(2500, 10);
        uint64 commit = uint64(block.timestamp) - 10;
        bytes[] memory data = _historicalBasket(200_020_000, 200_000, commit);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            pletherOracle.updateOrderExecutionPrice(address(this), data, _request(commit, CfdTypes.Side.SHORT, true));

        assertTrue(ok, "execution resolves");
        assertEq(snap.markPrice, 200_000_000, "neutral mark is capped");
        assertEq(snap.price, 199_950_000, "BUG: capped price shifted down a second time");
        assertLt(snap.price, 199_970_000, "below the shift-then-cap policy price by 20_000");
    }

    function test_C1_LongOpen_AboveCap_ExecutionShiftsCappedPrice() public {
        vm.warp(SETUP_TIMESTAMP + 1000);
        _applyOraclePolicy(2500, 10);
        uint64 commit = uint64(block.timestamp) - 10;
        bytes[] memory data = _historicalBasket(200_020_000, 200_000, commit);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            pletherOracle.updateOrderExecutionPrice(address(this), data, _request(commit, CfdTypes.Side.LONG, false));

        assertTrue(ok, "execution resolves");
        assertEq(snap.price, 199_950_000, "BUG: capped price shifted down a second time");
    }

    /// Upward shifts are re-clamped, so the leak is one-sided (downward shifts only).
    function test_C1_LongClose_AboveCap_UpwardShiftReclamped() public {
        vm.warp(SETUP_TIMESTAMP + 1000);
        _applyOraclePolicy(2500, 10);
        uint64 commit = uint64(block.timestamp) - 10;
        bytes[] memory data = _historicalBasket(200_020_000, 200_000, commit);

        (bool ok, IPletherOracle.PriceSnapshot memory snap) =
            pletherOracle.updateOrderExecutionPrice(address(this), data, _request(commit, CfdTypes.Side.LONG, true));

        assertTrue(ok, "execution resolves");
        assertEq(snap.price, 200_000_000, "upward shift lands on the cap, matching policy");
    }

    /// The batch path (and its cache) shares the same capped-base shift.
    function test_C1_BatchAndCacheReuse_AboveCap_SameCappedShift() public {
        vm.warp(SETUP_TIMESTAMP + 1000);
        _applyOraclePolicy(2500, 10);
        uint64 commit = uint64(block.timestamp) - 10;
        bytes[] memory data = _historicalBasket(200_020_000, 200_000, commit);
        IPletherOracle.BatchOrderPriceCache memory empty;

        (bool ok, IPletherOracle.PriceSnapshot memory snap, IPletherOracle.BatchOrderPriceCache memory cache) =
            pletherOracle.updateBatchOrderExecutionPrice(
                address(this), data, _request(commit, CfdTypes.Side.SHORT, true), empty
            );
        assertTrue(ok, "batch execution resolves");
        assertEq(snap.price, 199_950_000, "BUG: batch path shifts the capped price");
        assertTrue(cache.hasHistoricalBasket, "cache produced");

        uint256 parses = baseMockPyth.parseUniqueCallCount();
        (bool ok2, IPletherOracle.PriceSnapshot memory reused,) = pletherOracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _request(commit + 1, CfdTypes.Side.SHORT, true), cache
        );
        assertTrue(ok2, "cache reuse resolves");
        assertEq(reused.price, 199_950_000, "BUG: reused cache repeats the capped shift");
        assertEq(baseMockPyth.parseUniqueCallCount(), parses, "reuse performs no new parse");
    }
}

/// @notice Standalone oracle tests with the lightweight engine/vault mocks (same pattern as
///         test/perps/spec/oracle/PletherOracle.t.sol), plus calendar and production-basket checks.
contract Lane1StandaloneTest is Test {

    bytes32 internal constant FEED_A = bytes32(uint256(1));
    bytes32 internal constant FEED_B = bytes32(uint256(2));

    MockPyth internal pyth;
    PletherOracleEngineMock internal engine;
    PletherOracleVaultMock internal vault;
    PletherOracle internal oracle;

    bytes32[] internal feedIds;
    uint256[] internal weights;
    uint256[] internal basePrices;
    bool[] internal inversions;

    receive() external payable {}

    function setUp() public {
        pyth = new MockPyth();
        pyth.setSynchronizeLegacyUniquePrices(true);
        engine = new PletherOracleEngineMock();
        vault = new PletherOracleVaultMock();
        engine.setOrderRouter(address(this));

        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        basePrices.push(1e8);
        basePrices.push(1e8);
        inversions.push(false);
        inversions.push(false);

        oracle = new PletherOracle(address(engine), address(vault), address(pyth), feedIds, weights, basePrices, inversions);
    }

    function _applyPolicy(uint256 multiplier, uint256 ratioBps) internal {
        oracle.applyConfig(
            IPletherOracle.OracleConfig({
                orderExecutionStalenessLimit: 60,
                liquidationStalenessLimit: 15,
                basketMaxConfidenceRatioBps: ratioBps,
                orderSettlementWindow: 15,
                maxComponentPublishTimeDivergence: 5,
                adverseConfidenceMultiplierBps: multiplier
            })
        );
    }

    function _data() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = hex"01";
    }

    function _request(CfdTypes.Side side, bool isClose)
        internal
        pure
        returns (IPletherOracle.OrderExecutionRequest memory)
    {
        return IPletherOracle.OrderExecutionRequest({
            commitTime: 990, targetPrice: 0, side: side, isClose: isClose, revertOnHistoricalUnavailable: true
        });
    }

    /// C1 parity: identical basket + config priced through liquidation (fixed, MED-10)
    /// and through delayed order execution (residual capped-base shift).
    function test_C1_LiquidationVsExecution_AboveCap_ParityBreak() public {
        vm.warp(1000);
        _applyPolicy(2500, 10);
        engine.setPosition(1e18, CfdTypes.Side.SHORT);
        pyth.setPrice(FEED_A, int64(200_020_000), uint64(200_000), int32(-8), 990);
        pyth.setPrice(FEED_B, int64(200_020_000), uint64(200_000), int32(-8), 990);

        IPletherOracle.PriceSnapshot memory liq =
            oracle.updateLiquidationPrice(address(this), _data(), address(0xCAFE));
        assertEq(liq.price, 199_970_000, "liquidation: shift raw basket, then cap (MED-10 fix)");

        pyth.setAllUniquePrices(feedIds, int64(200_020_000), uint64(200_000), int32(-8), 995, 990);
        (bool ok, IPletherOracle.PriceSnapshot memory exec) =
            oracle.updateOrderExecutionPrice(address(this), _data(), _request(CfdTypes.Side.SHORT, true));
        assertTrue(ok, "execution resolves");
        assertEq(exec.price, 199_950_000, "execution: cap first, then shift (residual)");
        assertEq(liq.price - exec.price, 20_000, "same economic exit priced 20_000 apart");
    }

    /// C1 aggravation: a governance-valid extreme confidence policy (ratio 3333 bps x
    /// multiplier 30_000 bps, product 99_990_000 < 100_000_000) makes downward order
    /// executions REVERT above the cap while liquidation still executes.
    function test_C1_ExtremeValidPolicy_ExecutionReverts_LiquidationExecutes() public {
        vm.warp(1000);
        _applyPolicy(30_000, 3333);
        // Raw basket 300_000_000, aggregate confidence 99_990_000 (exactly at the 3333 bps gate).
        // Shift = 99_990_000 * 30_000 / 10_000 = 299_970_000.
        engine.setPosition(1e18, CfdTypes.Side.SHORT);
        pyth.setPrice(FEED_A, int64(300_000_000), uint64(99_990_000), int32(-8), 990);
        pyth.setPrice(FEED_B, int64(300_000_000), uint64(99_990_000), int32(-8), 990);

        IPletherOracle.PriceSnapshot memory liq =
            oracle.updateLiquidationPrice(address(this), _data(), address(0xCAFE));
        assertEq(liq.price, 30_000, "liquidation executes at raw-minus-shift");
        assertEq(liq.markPrice, 200_000_000, "neutral mark capped");

        pyth.setAllUniquePrices(feedIds, int64(300_000_000), uint64(99_990_000), int32(-8), 995, 990);
        vm.expectRevert(IPletherOracle.PletherOracle__ZeroExecutionPrice.selector);
        oracle.updateOrderExecutionPrice(address(this), _data(), _request(CfdTypes.Side.SHORT, true));
    }

    /// KILL K-precision: production 6-leg basket (v1.3.0 fork config + fixture spot/conf/expo)
    /// reproduces exact rational basket math to within 20 units (~0.002 bps). Rounding in
    /// _invertPythPrice / per-leg flooring is immaterial.
    function test_K_ProductionBasket_MatchesExactMath() public {
        MockPyth pyth6 = new MockPyth();
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory w = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inv = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = bytes32(uint256(11 + i));
        }
        w[0] = 576000000000000000; w[1] = 136000000000000000; w[2] = 119000000000000000;
        w[3] = 91000000000000000; w[4] = 42000000000000000; w[5] = 36000000000000000;
        bases[0] = 117500000; bases[1] = 638000; bases[2] = 134480000;
        bases[3] = 72880000; bases[4] = 10860000; bases[5] = 126100000;
        inv[0] = false; inv[1] = true; inv[2] = false; inv[3] = true; inv[4] = true; inv[5] = true;

        uint256 publish = 1_709_532_005; // Mon 2024-03-04 10:00:05 UTC, inside its hour epoch
        pyth6.setPrice(ids[0], int64(111896), uint64(2), int32(-5), publish);
        pyth6.setPrice(ids[1], int64(158482), uint64(5), int32(-3), publish);
        pyth6.setPrice(ids[2], int64(132350), uint64(4), int32(-5), publish);
        pyth6.setPrice(ids[3], int64(142640), uint64(4), int32(-5), publish);
        pyth6.setPrice(ids[4], int64(1001278), uint64(111), int32(-5), publish);
        pyth6.setPrice(ids[5], int64(83268), uint64(2), int32(-5), publish);

        PletherOracle prodOracle =
            new PletherOracle(address(engine), address(vault), address(pyth6), ids, w, bases, inv);
        vm.warp(1_709_532_010);
        IPletherOracle.PriceSnapshot memory snap =
            prodOracle.updatePrice(address(this), _data(), IPletherOracle.PriceMode.MarkRefresh);

        // Exact rational basket = 96_059_570.04 ; on-chain integer math yields 96_059_556.
        assertEq(snap.price, 96_059_556, "on-chain production basket value");
        assertApproxEqAbs(snap.price, 96_059_570, 20, "within 20 units of exact rational value");
        (, uint256 confidence) = prodOracle.getLatestPoolReconcilePrice();
        assertEq(confidence, 2516, "aggregate confidence (exact 2514.82, +1.2 rounding)");
    }

    /// KILL K-LOW36-regression: the LOW-36 example shape (old linear formula lands exactly on
    /// the 10 bps cap and would pass) is now rejected using the transformed upper endpoint.
    function test_K_InvertedConfidence_UsesUpperEndpoint_RejectsBorderline() public {
        vm.warp(1000);
        inversions[1] = true;
        basePrices[1] = 50_000_000;
        PletherOracle invOracle =
            new PletherOracle(address(engine), address(vault), address(pyth), feedIds, weights, basePrices, inversions);
        pyth.setPrice(FEED_A, int64(100_000_000), uint64(0), int32(-8), 990);
        pyth.setPrice(FEED_B, int64(200_000_000), uint64(400_000), int32(-8), 990);
        // Old (pre-fix) contribution: 50_000_000 * 400_000 / 200_000_000 = 100_000 == cap -> passed.
        // Correct upper-endpoint contribution: 100_200 > cap -> must revert.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__BasketConfidenceTooWide.selector,
                IPletherOracle.PriceMode.MarkRefresh,
                uint256(100_200),
                uint256(100_000_000),
                uint256(10)
            )
        );
        invOracle.updatePrice(address(this), _data(), IPletherOracle.PriceMode.MarkRefresh);
    }

    /// KILL K-cache-forgery (as an exploitable issue): the permissionless batch entrypoint
    /// will price off a fully fabricated caller-supplied cache with no Pyth parse at all.
    /// Documented here because no on-chain consumer accepts an external cache: the Router's
    /// BatchExecutionState starts empty each batch transaction and only chains oracle-returned
    /// caches (OrderRouterExecutionSidecar._prepareBatchOracle). A direct caller only fools
    /// itself and any ETH it sends on the reuse path stays in the oracle.
    function test_K_ForgedBatchCache_ReturnsUnauthenticatedPrice_ButNoConsumer() public {
        vm.warp(1000);
        pyth.setPrice(FEED_A, int64(100_000_000), int32(-8), 995);
        pyth.setPrice(FEED_B, int64(100_000_000), int32(-8), 995);

        IPletherOracle.BatchOrderPriceCache memory forged = IPletherOracle.BatchOrderPriceCache({
            hasHistoricalBasket: true,
            minReusableCommitTime: 990,
            price: 150_000_000,
            confidence: 0,
            publishTime: 995,
            oldestPublishTime: 994
        });
        (bool ok, IPletherOracle.PriceSnapshot memory snap,) =
            oracle.updateBatchOrderExecutionPrice(address(this), new bytes[](0), _request(CfdTypes.Side.LONG, false), forged);

        assertTrue(ok, "forged cache is accepted by the oracle entrypoint");
        assertEq(snap.price, 150_000_000, "attacker-chosen price returned with zero confidence shift");
        assertEq(pyth.parseUniqueCallCount(), 0, "no Pyth parse authenticates the cache");
    }

    /// KILL K-calendar: MarketCalendarLib frozen boundaries across both 2026 DST transitions.
    function test_K_MarketCalendar_DstBoundaries() public {
        Lane1CalendarHarness cal = new Lane1CalendarHarness();
        bool fad;
        bool frozen;
        // Fri 2026-03-06, 17:00 EST = 22:00 UTC (standard time, pre spring-forward)
        (fad, frozen) = cal.status(1772834399);
        assertFalse(frozen, "Fri 21:59:59 UTC not yet frozen");
        (fad, frozen) = cal.status(1772834400);
        assertTrue(frozen, "Fri 22:00:00 UTC frozen");
        (fad, frozen) = cal.status(1772834400 - 1800);
        assertTrue(fad, "FAD leads the Friday close by 30 min");
        assertFalse(frozen, "FAD lead is not yet frozen");
        // Sun 2026-03-08, 17:00 EDT = 21:00 UTC (spring-forward day)
        (fad, frozen) = cal.status(1773003599);
        assertTrue(frozen, "Sun 20:59:59 UTC still frozen");
        (fad, frozen) = cal.status(1773003600);
        assertFalse(frozen, "Sun 21:00:00 UTC reopened");
        assertTrue(fad, "FAD lags the Sunday open by 15 min");
        // Fri 2026-10-30, 17:00 EDT = 21:00 UTC (pre fall-back)
        (fad, frozen) = cal.status(1793393999);
        assertFalse(frozen, "Fri 20:59:59 UTC not yet frozen");
        (fad, frozen) = cal.status(1793394000);
        assertTrue(frozen, "Fri 21:00:00 UTC frozen");
        // Sun 2026-11-01, 17:00 EST = 22:00 UTC (fall-back day)
        (fad, frozen) = cal.status(1793570399);
        assertTrue(frozen, "Sun 21:59:59 UTC still frozen");
        (fad, frozen) = cal.status(1793570400);
        assertFalse(frozen, "Sun 22:00:00 UTC reopened");
    }
}
