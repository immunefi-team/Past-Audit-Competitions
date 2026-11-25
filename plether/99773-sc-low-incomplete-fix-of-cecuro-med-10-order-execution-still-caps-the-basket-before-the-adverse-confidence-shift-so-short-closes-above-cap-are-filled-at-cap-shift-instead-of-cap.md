# \#99773 [SC-Low] Incomplete fix of Cecuro MED-10: order execution still caps the basket before the adverse confidence shift, so SHORT closes above CAP are filled at `CAP - shift` instead of `CAP`

**Submitted on Nov 15th 2026 at 07:48:39 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99773
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Cecuro MED-10 showed that liquidation pricing capped the basket at `CAP_PRICE` first and applied the adverse confidence shift afterwards. A SHORT was therefore priced below CAP even when the shifted raw price was still above CAP. Commit `eadc5c58` (PR #124) fixed this, but only in the two liquidation functions. The order-execution path in the same contract (single and batch) still uses the old order.

So when the raw basket is above CAP, every SHORT close is filled at `CAP - shift` instead of `min(raw - shift, CAP) = CAP`. This covers manual closes and protection-triggered TP/SL, in both single and batch execution. In the PoC, a 300,000-unit SHORT close loses **164.93 USDG** of realized settlement balance, and the pool keeps exactly that amount. No attacker is needed: every affected close moves trader funds to LPs automatically.

### Plain Summary

- The SHORT side profits as the basket rises, and its payout is capped at `CAP = 2.0`.
- The basket is at 2.2. Pyth reports a small confidence interval, so the oracle moves the price against the trader by `shift = 0.00055`.
- Correct order (the MED-10 rule): `2.2 - 0.00055 = 2.19945`. This is still above CAP, so the fill is **2.0**. The shift has no effect.
- Actual order: cap first to 2.0, then subtract 0.00055. The fill is **1.99945**. The trader loses `0.00055` per unit, which is 165 USDG on 300k units.

### Vulnerability Details

`_snapshotFromBasket` caps first:

```solidity
// PletherOracle.sol:887
snapshot.price = _clampToCap(basket.price);
```

Both order-execution paths then pass the **capped** price into the shift:

```solidity
// PletherOracle.sol:557-558 (single)  and  :283-284 (batch), identical
snapshot = _snapshotFromBasket(basket, policy, false);
@> snapshot.price = _orderExecutionPrice(request, snapshot.price, basket.confidence, policy.oracleFrozen);
```

```solidity
// PletherOracle.sol:915-922, :940
bool adverseUp = request.side == CfdTypes.Side.SHORT ? !request.isClose : request.isClose;
...
@> return price - shift;                                    // SHORT close / LONG open -> CAP - shift
...
return _clampToCap(_adverseOrderPrice(request, price, confidence));
```

The fixed liquidation path in the same contract shifts the raw price and caps once:

```solidity
// PletherOracle.sol:342
snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));
// PletherOracle.sol:316-317 (batch) - same pattern with rawPrice
```

The repo's own MED-10 regression test (`packages/perps/test/perps/spec/oracle/PletherOracle.t.sol:675`, `test_Liquidation_ShortStaysCappedWhenShiftLeavesRawPriceAboveCap`) asserts that the SHORT stays at CAP when `raw - shift >= CAP`. Order execution has no such test, and it breaks this rule.

The same code also prices a LONG open above CAP at `CAP - shift`. I did not test that path separately.

### Impact Details

| PoC (release params, actually run) | Value |
|---|---|
| Raw basket / CAP | 2.2 / 2.0 |
| Confidence (max accepted at 0.1% ratio) / shift (×0.25) | 0.0022 / 0.00055 |
| `raw - shift` | 2.19945 (≥ CAP, so the correct fill is CAP) |
| SHORT size closed | 300,000 units (600k USDG notional at CAP) |
| Trader balance, conf 0 → conf max | 499,639.80 → 499,474.87 USDG |
| **Trader loss** | **164.934 USDG** (`size × shift` = 165, minus a smaller fee on the lower notional) |
| Pool kept | 164.934 USDG (exactly the trader's loss) |

- About **2.75 bp of capped notional per close** at release params. The loss scales linearly with size, and it grows if governance raises the multiplier or ratio. The code allows a product up to `< 1e8`, so the shift can reach about 10% of price.
- **Who is hit:** every SHORT that closes while the basket is above CAP. No attacker is needed; this is a systematic mis-fill. SHORT take-profit orders are the most exposed, because they trigger exactly when the price is high.
- **Who benefits:** LPs.
- **Broken invariant:** "adverse confidence is applied to the pre-cap basket and the result is capped once". This is the MED-10 remediation, and the fix applies it only to liquidations.
- The fuzz test (raw in [2.01, 4.0], any accepted confidence, 10k–300k units, full or partial close, 256 runs) shows a loss of about `closeSize × shift` on every run.

- **Why "Direct theft of user funds":** this is not a lower return. The trader's settlement balance is reduced. The pool's payout is reduced by exactly the same amount (trader −164.934 USDG, pool +164.934 USDG), so the funds change owner. The loss is realized at execution and cannot be recovered. It happens on every SHORT close above CAP without any action by the victim beyond closing, and protection-triggered closes do it without the user acting at all.

Suggested impact: Direct theft of user funds. The transfer is systematic (no attacker) and the per-close amount is bounded by `size × shift`. Severity is the judge's call.

### Why this is not a known issue

- **Cecuro MED-10** (`audits/plether-ai-audit-report-2026-10-26.pdf`, p. 28-30) has the same root cause and is marked "Addressed". The fix (`eadc5c58`) changed only `updateLiquidationPrice` / `updateLiquidationBatchPrice`. In the order-execution functions it only added a zero-price revert. This report shows that the fix is incomplete.
- **`packages/perps/SECURITY.md:652-653`** mentions "a downward adjustment that exhausts the capped execution base reverts". That is a revert rule for zero prices. It does not say that a SHORT close above CAP should be filled below CAP, and that outcome contradicts the ordering the team adopted for MED-10.

### Mitigation

Make the same change as the MED-10 fix. Pass the pre-cap `basket.price` into `_orderExecutionPrice` at `:284` and `:558`, and cap once at the end: `_clampToCap(_adverseOrderPrice(request, basket.price, confidence))`. Keep `snapshot.markPrice` neutral and capped. In the frozen-close branch, return the capped neutral price.

### References

- `packages/perps/src/PletherOracle.sol:887`: cap applied in `_snapshotFromBasket`
- `packages/perps/src/PletherOracle.sol:557-558`: single execution shifts the capped price
- `packages/perps/src/PletherOracle.sol:283-284`: batch execution shifts the capped price
- `packages/perps/src/PletherOracle.sol:905-941`: `_adverseOrderPrice` / `_orderExecutionPrice`
- `packages/perps/src/PletherOracle.sol:316-317, :342`: fixed liquidation path (raw → shift → cap)
- `packages/perps/test/perps/spec/oracle/PletherOracle.t.sol:675`: MED-10 regression test (liquidation only)
- Commit `eadc5c58` (PR #124): MED-10 fix

## Proof of Concept

## Proof of Concept


Save as `packages/perps/test/fixreview-a-order-cap-ordering.t.sol` and run:

```
forge test --root packages/perps --match-path test/fixreview-a-order-cap-ordering.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

// Standalone PoC. Save as packages/perps/test/fixreview-a-order-cap-ordering.t.sol and run:
//   forge test --root packages/perps --match-path test/fixreview-a-order-cap-ordering.t.sol -vv
/// @notice Fix review of Cecuro MED-10 (fixed in eadc5c58 / PR #124): the liquidation paths now shift the RAW basket
///         by adverse confidence and clamp to CAP afterwards. The ordinary order-execution path
///         (PletherOracle._orderExecutionPrice) still shifts the ALREADY-CAPPED price. When the raw basket is above CAP
///         and raw - shift >= CAP, a SHORT close (and a LONG open) is filled at CAP - shift instead of CAP.
///         Release params: MM 0.1%, IM 0.2%, vpi 0.01, maxSkew 0.4, adverseConfidenceMultiplierBps 2500,
///         basketMaxConfidenceRatioBps 10. Only Pyth and USDG are mocked.
contract FixreviewACapOrderingTest is BasePerpTest {

    address internal constant S = address(0x5404);
    address internal constant L = address(0x1046);
    uint256 internal constant ADVERSE_MULT = 2500;
    uint256 internal constant CONF_RATIO = 10;

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
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), ADVERSE_MULT);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), CONF_RATIO);
        _fundTrader(S, 200_000e6);
        _fundTrader(L, 200_000e6);
        _refresh(1e8, 0);
    }

    // ------------------------------------------------------------------ helpers
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

    /// @dev Commits then executes one order at a later historical tick carrying `conf` basket confidence.
    function _commitExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        uint256 conf
    ) internal returns (uint64 id) {
        vm.prank(account);
        id = router.commitOrder(side, size, margin, 0, isClose);
        uint256 commitTime = vm.getBlockTimestamp();
        vm.warp(commitTime + 1);
        vm.roll(vm.getBlockNumber() + 1);
        router.executeOrder(id, _payload(price, conf, commitTime + 1, commitTime));
    }

    function _size(
        address a
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _maxConf(
        uint256 raw
    ) internal pure returns (uint256) {
        // Accepted iff conf * 10_000 <= price * ratio (equal weights, equal feeds -> basket conf == feed conf).
        return raw * CONF_RATIO / 10_000;
    }

    // ------------------------------------------------------------------ deterministic PoC
    /// @dev Same pre-state, same raw basket 2.2 (above CAP 2.0); only basket confidence differs.
    ///      Correct policy (MED-10: shift raw, then cap): fill = min(2.2 - shift, 2.0) = 2.0 in both runs.
    ///      Actual: with confidence the SHORT close fills at CAP - shift, so the trader loses ~size * shift to the pool.
    function test_ShortCloseAboveCap_FilledBelowCap_TraderLosesSizeTimesShift() public {
        uint256 size = 300_000e18;
        _commitExecute(S, CfdTypes.Side.SHORT, size, 20_000e6, false, 1e8, 0);
        assertEq(_size(S), size, "short open");

        uint256 raw = 2.2e8;
        uint256 conf = _maxConf(raw); // 220_000 = 0.1% of raw, the max the release ratio accepts
        uint256 shift = conf * ADVERSE_MULT / 10_000; // 55_000 = 0.00055 per unit
        assertGe(raw - shift, engine.CAP_PRICE(), "even the adverse price is above CAP");

        uint256 snap = vm.snapshotState();

        // Run A: zero confidence -> fill at CAP
        uint256 poolBefore = pool.totalAssets();
        _commitExecute(S, CfdTypes.Side.SHORT, size, 0, true, raw, 0);
        assertEq(_size(S), 0, "closed (conf 0)");
        uint256 balNoConf = _settlementBalance(S);
        uint256 poolPaidNoConf = poolBefore - pool.totalAssets();
        vm.revertToState(snap);

        // Run B: same raw price, max accepted confidence -> fill at CAP - shift
        poolBefore = pool.totalAssets();
        _commitExecute(S, CfdTypes.Side.SHORT, size, 0, true, raw, conf);
        assertEq(_size(S), 0, "closed (conf max)");
        uint256 balConf = _settlementBalance(S);
        uint256 poolPaidConf = poolBefore - pool.totalAssets();

        uint256 traderLoss = balNoConf - balConf;
        uint256 expectedLoss = size * shift / 1e8 / 1e12; // size(1e18) * shift(1e8) -> USDG(1e6) = 165e6

        console2.log("raw basket                  ", raw);
        console2.log("CAP                         ", engine.CAP_PRICE());
        console2.log("confidence / shift          ", conf, shift);
        console2.log("raw - shift (>= CAP)        ", raw - shift);
        console2.log("trader balance, conf 0      ", balNoConf);
        console2.log("trader balance, conf max    ", balConf);
        console2.log("trader loss (USDG 6dp)      ", traderLoss);
        console2.log("size * shift (USDG 6dp)     ", expectedLoss);
        console2.log("pool paid out, conf 0       ", poolPaidNoConf);
        console2.log("pool paid out, conf max     ", poolPaidConf);
        console2.log("pool keeps (LP gain)        ", poolPaidNoConf - poolPaidConf);

        // Bug proof: the fill depends on confidence even though raw - shift >= CAP.
        assertLt(balConf, balNoConf, "SHORT close above CAP is filled below CAP");
        // Loss == size * shift (small diff = execution fee charged on a lower notional).
        assertApproxEqRel(traderLoss, expectedLoss, 0.001e18, "loss ~= size * shift");
        // The value goes to the pool / LPs.
        assertLt(poolPaidConf, poolPaidNoConf, "pool keeps the difference");
    }

    // ------------------------------------------------------------------ fuzz
    /// @dev Random raw basket above CAP + shift, random confidence within the release ratio, random size,
    ///      full or partial close. Every run: the SHORT is paid less with confidence than without,
    ///      and the loss is ~closeSize * shift.
    function testFuzz_ShortCloseAboveCap_AlwaysLosesSizeTimesShift(
        uint256 rawSeed,
        uint256 confSeed,
        uint256 sizeSeed,
        uint256 fracSeed
    ) public {
        uint256 raw = bound(rawSeed, 2.01e8, 4e8);
        uint256 conf = bound(confSeed, 1, _maxConf(raw));
        uint256 shift = conf * ADVERSE_MULT / 10_000;
        vm.assume(shift >= 100 && raw - shift >= engine.CAP_PRICE());
        uint256 size = bound(sizeSeed, 10_000e18, 300_000e18) / 1e20 * 1e20;
        uint256 closeSize = fracSeed % 2 == 0 ? size : (size * bound(fracSeed, 50, 99) / 100) / 1e20 * 1e20;

        _commitExecute(S, CfdTypes.Side.SHORT, size, 20_000e6, false, 1e8, 0);
        uint256 snap = vm.snapshotState();
        _commitExecute(S, CfdTypes.Side.SHORT, closeSize, 0, true, raw, 0);
        assertEq(_size(S), size - closeSize, "close executed (conf 0)");
        uint256 a = _settlementBalance(S);
        vm.revertToState(snap);
        _commitExecute(S, CfdTypes.Side.SHORT, closeSize, 0, true, raw, conf);
        assertEq(_size(S), size - closeSize, "close executed (conf>0)");
        uint256 b = _settlementBalance(S);

        assertLt(b, a, "SHORT close above CAP filled below CAP");
        assertApproxEqRel(a - b, closeSize * shift / 1e8 / 1e12, 0.01e18, "loss ~= closeSize * shift");
    }

}
```

## Result:

Actually run: `forge test --root packages/perps --match-path test/fixreview-a-order-cap-ordering.t.sol -vv`

```
Ran 2 tests for test/fixreview-a-order-cap-ordering.t.sol:FixreviewACapOrderingTest
[PASS] testFuzz_ShortCloseAboveCap_AlwaysLosesSizeTimesShift(uint256,uint256,uint256,uint256) (runs: 256, μ: 5303122, ~: 5289628)
[PASS] test_ShortCloseAboveCap_FilledBelowCap_TraderLosesSizeTimesShift() (gas: 5309092)
Logs:
  raw basket                   220000000
  CAP                          200000000
  confidence / shift           220000 55000
  raw - shift (>= CAP)         219945000
  trader balance, conf 0       499639799876
  trader balance, conf max     499474865876
  trader loss (USDG 6dp)       164934000
  size * shift (USDG 6dp)      165000000
  pool paid out, conf 0        300209101672
  pool paid out, conf max      300044167672
  pool keeps (LP gain)         164934000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 867.92ms (872.41ms CPU time)
```

