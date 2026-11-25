# \#99141 [SC-Medium] Skew-reducing close nets the VPI rebate against the execution fee, so the treasury funds the rebate instead of pool assets

**Submitted on Nov 12th 2026 at 03:39:20 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99141
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

On a skew-reducing close, the protocol execution fee is capped at the action charge the close actually collects, and the VPI rebate has already reduced that charge. The treasury is therefore credited `fee − rebate`, and nothing tops it up. In effect the treasury funds the trader's rebate. An open with the same skew-reducing economics credits the full fee and pays the rebate from pool cash.

The documented split is the opposite. WHITEPAPER.md:411-413: "a VPI rebate reduces [pool assets]; the protocol execution fee belongs to treasury rather than LP assets."

Measured at release parameters on Forge 1.5.1, same snapshot, same skew removed by a close versus an equal opposite open:
- Max position at the 1M-depth skew cap (400,800 USDG notional): the close credits the treasury **0** of its 160.32 USDG fee. The open credits the full 160.32.
- 6,000 USDG close: the treasury is short 0.18 USDG.

The trader nets the same on both paths. The difference moves from the treasury to the LPs.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/libraries/CfdEnginePlanLib.sol` (`_planCloseActionSettlement`)

## Vulnerability Details

The close action settlement computes the fee base from the action charge that was withheld or collected (`CfdEnginePlanLib.sol:1066-1075`):

```solidity
uint256 nonClawbackWithheldAssets = delta.actionChargeWithheldAssets - vpiClawbackWithheldAssets;
uint256 nonVpiReserveCollectedAssets = delta.actionChargeCollectedAssets - delta.vpiRebateReserveConsumedAssets;
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

The action charge on a close is the execution fee plus the VPI. A skew-reducing close has negative VPI, so the action charge is `fee − rebate`, and the eligible base shrinks by the rebate. `delta.executionFeeAssets` is reduced to that base before the top-up is computed (`:1055-1058`). The top-up then covers only `executionFeeAssets − credited`, so the part of the fee consumed by the rebate is never credited to the treasury. Once the rebate is at least the fee, the treasury gets nothing.

The open path treats the same economics differently (`CfdEnginePlanLib.sol:718-748`). It keeps the full `executionFeeAssets` and tops up any shortfall from free pool cash, so the pool, not the treasury, funds the rebate.

The two paths can remove the same skew with identical trader economics, yet one credits the treasury the full fee and the other credits `fee − rebate`.

## Impact Details

The protocol treasury does not receive the execution fee the release configuration charges (4 bps of close notional) on any skew-reducing close with a VPI rebate. The LPs keep it instead.

| Release parameters, 1M depth | CLOSE: trader / treasury / pool | OPEN: trader / treasury / pool | Treasury short |
|---|---|---|---|
| 5,000 tokens (6,000 USDG notional) | −2.220362 / +2.220360 / +0.000002 | −2.220360 / +2.400000 / −0.179640 | 0.179640 |
| 334,000 tokens (400,800 USDG notional, skew cap) | +640.628556 / 0 / −640.628556 | +640.639232 / +160.320000 / −800.959232 | 160.320000 |

The trader's result matches to within 0.011 USDG of VPI rounding. The pool keeps exactly what the treasury loses.

At the release Senior cap (40M Junior plus Senior depth, positions up to 16M notional) the same rule removes up to 6,400 USDG of fee per close. Every skew-reducing close pays, so on rebalancing flow the treasury loses up to the whole execution fee.

Severity: Low, "Contract fails to deliver promised returns". The treasury does not receive the configured execution fee. No user principal is lost.

## Recommendation

Assess the close execution fee before netting the VPI rebate, as the open path does. Credit `min(executionFee, collected + withheld + freePoolCash)` to the treasury, and pay the rebate from pool assets as WHITEPAPER.md:411-413 describes. Or apply the open path's `directlyCredited + topUp` split to closes.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1066-1075`: close fee capped by the rebate-reduced eligible base
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1055-1058`: top-up computed from the already-reduced fee
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:718-748`: open path, full fee plus top-up from free pool cash
- `packages/perps/WHITEPAPER.md:411-413`: the rebate reduces pool assets; the execution fee belongs to the treasury
- `packages/perps/DEPLOYMENT.md`: release execution fee 4 bps, vpiFactor 0.01e18, maxSkewRatio 0.4e18

## Proof of Concept

## Steps

1. Clone `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Save the file below as `packages/perps/test/perps/CloseRebateFeeAttribution.t.sol`. It extends the repository's own `BasePerpTest` and sets the release RiskParams from `packages/perps/DEPLOYMENT.md`.
3. Run with Forge 1.5.1 from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/CloseRebateFeeAttribution.t.sol -vv
```

Each test opens a LONG that creates skew, takes a snapshot, then removes exactly that skew two ways: the same account closes its LONG (close path), or another account opens an equal SHORT (open path). It prints the acting account's, the treasury's and the pool's cash deltas on both paths and asserts that the actor nets the same while the treasury is credited less on the close and the pool keeps the difference.

## Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: on a CLOSE with negative VPI (a skew-reducing rebate), CfdEnginePlanLib._planCloseActionSettlement caps the
// protocol execution fee at the action charge actually collected, which the rebate has already reduced, and the
// protocol-fee top-up then covers only that reduced fee. The treasury therefore funds the trader's rebate and the
// pool keeps it. On an OPEN the same skew-reducing economics credit the full fee to the treasury and the pool funds
// the rebate (top-up from free pool cash). WHITEPAPER.md:411-413: "a VPI rebate reduces [physical pool assets];
// the protocol execution fee belongs to treasury rather than LP assets."
//
// Place at packages/perps/test/perps/CloseRebateFeeAttribution.t.sol and run (Forge 1.5.1, pin a82d4fef):
//   forge test --root packages/perps --match-path test/perps/CloseRebateFeeAttribution.t.sol -vv
//
// Release parameters (DEPLOYMENT.md / v2.0.0 manifest): vpiFactor 0.01e18, maxSkewRatio 0.4e18, execution fee 4 bps.

import {console} from "forge-std/console.sol";
import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

contract CloseRebateFeeAttribution is BasePerpTest {
    address trader = address(0xA11CE);
    address other = address(0xB0B2);

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

    function _next() internal {
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 60);
    }

    function _wealth(address who) internal view returns (int256) {
        return int256(settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who));
    }

    function _treasury() internal view returns (int256) {
        return int256(clearinghouse.balanceAssets(engine.protocolTreasury()));
    }

    function _poolCash() internal view returns (int256) {
        return int256(settlementToken.balanceOf(address(pool)));
    }

    struct Out {
        int256 actor;
        int256 treasury;
        int256 pool;
    }

    // A 6,000-USDG LONG creates skew. The second event removes exactly that skew:
    //   close == true : the same account closes its LONG (close path)
    //   close == false: another account opens an equal SHORT (open path)
    function _second(uint256 size, bool close) internal returns (Out memory o) {
        _fundTrader(trader, 400_000e6);
        _fundTrader(other, 400_000e6);
        _next();
        _open(trader, CfdTypes.Side.LONG, size, size / 1e18 * 1e6 / 5, 1.2e8);
        _next();
        address actor = close ? trader : other;
        int256 a0 = _wealth(actor);
        int256 r0 = _treasury();
        int256 p0 = _poolCash();
        if (close) {
            _close(trader, CfdTypes.Side.LONG, size, 1.2e8);
        } else {
            _open(other, CfdTypes.Side.SHORT, size, size / 1e18 * 1e6 / 5, 1.2e8);
        }
        o.actor = _wealth(actor) - a0;
        o.treasury = _treasury() - r0;
        o.pool = _poolCash() - p0;
    }

    function _compare(uint256 size) internal {
        uint256 snap = vm.snapshotState();
        Out memory c = _second(size, true);
        vm.revertToState(snap);
        Out memory o = _second(size, false);
        vm.revertToState(snap);

        uint256 fee = (size * 1.2e8 / 1e20) * engine.executionFeeBps() / 10_000;
        console.log("size (tokens) / quoted execution fee (atoms)", size / 1e18);
        console.log(fee);
        console.log("CLOSE path: actor delta / treasury delta / pool cash delta");
        console.logInt(c.actor);
        console.logInt(c.treasury);
        console.logInt(c.pool);
        console.log("OPEN path:  actor delta / treasury delta / pool cash delta");
        console.logInt(o.actor);
        console.logInt(o.treasury);
        console.logInt(o.pool);
        console.log("treasury short on the close path by");
        console.logInt(o.treasury - c.treasury);

        // Same skew-reducing economics: the acting account's net outcome is the same on both paths ...
        assertApproxEqAbs(c.actor, o.actor, 20_000, "actor nets the same on both paths (within 0.02 USDG of VPI rounding)");
        // ... but the open path credits the treasury more and the pool funds the difference.
        assertGt(o.treasury, c.treasury, "open credits the treasury more than the close");
        assertLt(o.pool, c.pool, "open funds the rebate from pool cash; the close does not");
        assertApproxEqAbs(o.treasury - c.treasury, c.pool - o.pool, 20_000, "the difference moves treasury -> pool");
    }

    function test_SmallClose_5k() public {
        _compare(5_000e18);
    }

    // Max position at the 1M-pool skew cap (334,000 tokens x 1.20 = 400,800 USDG): the close rebate exceeds the
    // 4 bps fee, so the close path credits the treasury nothing for this close.
    function test_MaxClose_400k() public {
        _compare(334_000e18);
    }
}
```

## Output (Forge 1.5.1, run at the pin)

```text
Ran 2 tests for test/perps/CloseRebateFeeAttribution.t.sol:CloseRebateFeeAttribution
[PASS] test_MaxClose_400k() (gas: 3043708)
Logs:
  size (tokens) / quoted execution fee (atoms) 334000
  160320000
  CLOSE path: actor delta / treasury delta / pool cash delta
  640628556
  0
  -640628556
  OPEN path:  actor delta / treasury delta / pool cash delta
  640639232
  160320000
  -800959232
  treasury short on the close path by
  160320000
[PASS] test_SmallClose_5k() (gas: 3043550)
Logs:
  size (tokens) / quoted execution fee (atoms) 5000
  2400000
  CLOSE path: actor delta / treasury delta / pool cash delta
  -2220362
  2220360
  2
  OPEN path:  actor delta / treasury delta / pool cash delta
  -2220360
  2400000
  -179640
  treasury short on the close path by
  179640
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 14.68ms (9.32ms CPU time)
Ran 1 test suite in 22.75ms (14.68ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```
