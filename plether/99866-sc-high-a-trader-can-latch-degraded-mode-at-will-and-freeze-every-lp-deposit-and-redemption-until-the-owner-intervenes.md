# \#99866 [SC-High] A trader can latch degraded mode at will and freeze every LP deposit and redemption until the owner intervenes

**Submitted on Nov 15th 2026 at 15:07:58 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99866
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

Any trader can push the engine into degraded mode without causing a loss to anyone. Degraded mode blocks every LP epoch settlement. While it is latched:

- no LP redemption is funded;
- no LP deposit is activated;
- a deposit whose epoch has started cannot be cancelled, so the depositor's USDG sits in vault escrow.

The owner cannot clear the latch while the trader keeps one position open. The latch also stays set after the trader has left and the pool holds millions in free cash, until the owner calls `clearDegradedMode()`.

The trigger needs no privileged role, no keeper failure and no price gap: only the basket about 0.5% away from the trader's entry price on a Friday, in either direction.

## Root cause

1. **The solvency latch can be tripped by one ordinary payout.** After every close and liquidation `CfdEngine._enterDegradedModeIfInsolvent` (CfdEngine.sol:1351, called at :1320 and :1344) latches when pool cash net of claims `E` is below the larger side's maximum profit `L`. Opens are admitted only while `E >= L + 0.25% L`. A profit paid to the *smaller* side removes cash while `L` is unchanged, so one payout above that 0.25% cushion latches the mode.
2. **The Friday margin switch pays such a profit in cash.** At the Friday window the maintenance rate goes from 0.1% to 3% (CfdEnginePlanLib.sol:1450). A position that is in profit but margined under 3% becomes liquidatable. Liquidation pays its whole profit from the pool (CfdEnginePlanLib.sol:1529, CfdEngineSettlementSidecar.sol:848) with no price-impact charge.
3. **A pending deposit has no exit while degraded.** `HousePool._settleLpEpoch` reverts in degraded mode (HousePool.sol:832, :1864). `TrancheVault.cancelPendingDeposit` (TrancheVault.sol:832-875) lets a controller cancel a started epoch only if the epoch was rejected, the tranche is wiped, or Senior is impaired. Degraded mode is not among those cases. The deposit can move neither forward nor back.
4. **Nothing clears the latch when solvency returns.** `clearDegradedMode` (CfdEngine.sol:658) is owner-only and requires `E >= L`.

## Attack

1. From two accounts open LONG N and SHORT N, with N up to the pool's capacity and about 1.5% margin on each leg. The pair is hedged: while both legs are open the attacker has no price exposure and pays only carry.
2. Wait for a Friday on which the basket is roughly 0.5% away from the entry price, up or down. Just before 20:30 UTC, add margin to the losing leg so it is above 3%.
3. At 20:30 UTC the winning leg is liquidatable under the 3% rule. Any keeper liquidates it; the attacker does not need to send that transaction. The pool pays the leg's profit, `E` drops below `L`, and degraded mode latches.
4. Keep the other leg open for as long as the freeze should last. (From here the attacker carries that leg's price exposure, which can be hedged on any FX venue.)

LPs lose nothing at the trigger: the profit paid to one leg is matched by the other leg's fully collateralised loss, which LP value already counts. The protocol is in containment although nothing went wrong.

## Impact

Measured in the PoC on a 5.0M USDG pool with a 0.50% move:

- After the keeper's Friday liquidation `E` = 4,985,654 and `L` = 4,988,657. Senior plus Junior value rose from 5,002,000 to 5,010,826 at the same moment.
- At the Sunday reopen, 24 hours later and 48 hours later: `settleLpEpoch` reverts `HousePool__DegradedMode`; the depositor's `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive`; 100,000 USDG is in vault escrow, none in the depositor's wallet, none claimable; an existing LP's matured redemption is unfunded.
- The owner's `clearDegradedMode()` reverts `CfdEngine__StillInsolvent` while the attacker holds the other leg. The owner must first inject capital (3,004 USDG in this run).
- After the attacker closes everything the pool holds 5,014,904 USDG of free cash and `L` is zero. LP settlement still reverts until the owner clears the latch. Only then does the deposit activate and the redemption fund.

So every pending LP deposit and the whole redemption queue are frozen from Friday evening for as long as the attacker chooses, and afterwards until the owner acts.

The attacker's outlay in the PoC was 20,000 to 21,400 USDG (fees, carry, the liquidation charge) plus about 8% of the pool as temporary margin. An idle pool is the expensive case; when other traders already use most of the capacity a much smaller pair is enough. I do not claim a profit for the attacker.

## How often the trigger is available

The PoC's threshold probe: a 0.40% move does not latch, 0.45% latches when the basket falls, 0.50% latches in both directions.

EUR/USD is 57.6% of the basket. Over 125 weeks of 2023-25 tick data its Monday-to-Friday change was at least 0.50% in 66 weeks (53%). The attacker can hold the pair over several weeks, paying carry, until such a Friday comes.

## Why this is not accepted design

- **"Degraded mode is documented containment."** ACCOUNTING_SPEC.md "Degraded Mode" says closes and liquidations latch it when `E < L` and lists LP settlement as blocked, as containment of raw insolvency. Here nothing is insolvent and LP value rises at the trigger; a trader chooses to put the protocol into containment and keep it there. The section's remedy, "owner action to clear the mode after solvency is genuinely restored", is unavailable while the attacker keeps a position. The spec never says a pending deposit has no exit; it only covers funded claims ("Already-funded LP claims remain claimable").
- **"Lack of liquidity."** The frozen deposit is the depositor's own USDG in escrow and needs no pool liquidity. After the attacker exits the pool has 5.0M of free cash and settlement still reverts. This differs from report #98270, where LP cash was reserved by open positions; here the block is a latch plus a missing cancel path.
- **"The owner can resolve it."** The owner takes no part in the trigger and cannot prevent it, cannot clear while the attacker holds a leg without injecting capital, and faces a repeat the next Friday. Funds not moving until an owner transaction arrives is the impact, not a mitigation.
- **"A bet on a price move."** No gap and no direction are needed. The pair is hedged, either direction works, and a 0.5% weekly move occurred in about half of all weeks.
- **"Needs keepers to fail."** The opposite: an honest keeper performs the liquidation that latches the mode. In the PoC an unrelated keeper executes every order, the liquidation and every settlement.
- **"Liquidation does not compute a fresh price-impact charge" (listed limitation).** That line describes a pricing simplification, not that it lets a trader take cash out past the solvency boundary and trap LP funds.

## Recommended fix

- Let a controller cancel an unfinalised deposit, and an unfunded redemption request, while the engine is degraded (add it to the exceptions at TrancheVault.sol:851-860).
- Clear the latch automatically, or let LP settlement proceed, once `E >= L` again.
- When a smaller-side profit would take `E` below `L`, record the excess as a trader claim instead of paying it in cash; the engine already has that mechanism.

## Link to Proof of Concept

https://gist.github.com/chj2934/295793af12f3cfd61133008198f6be35

## Proof of Concept

## Proof of concept

Secret gist: https://gist.github.com/chj2934/295793af12f3cfd61133008198f6be35

Copy its four `.sol` files into `packages/perps/test/fresh/degradedlatch/` and run from `packages/perps`:

```
forge test --match-path test/fresh/degradedlatch/DegradedLatchProd.t.sol -vv --gas-limit 100000000000
```

`DegradedLatchProd.t.sol` is the scenario. `Fail27Base.sol`, `Prod22Base.sol` and `LiveParams.sol` are harness only and build on the repository test base `test/perps/BasePerpTest.sol`.

Production path throughout: the release six-feed oracle with real confidence, V3 order commits, an unrelated keeper executing each order at the first tick after its commit, the keeper performing the Friday liquidation and every LP settlement. Cheatcodes are used only for time and oracle publishing. Result: 4 passed.

- `test_Attack_BasketUp50bps`, `test_Attack_BasketDown50bps`: the sequence above, with every statement in "Impact" asserted.
- `test_Control_SamePairSameMove_NoLatch_DepositActivates`: same pair, same move, both legs topped up above 3%. Nothing is liquidated, no latch, the deposit activates at the Sunday reopen.
- `test_Probe_MoveThreshold`: prints which move sizes latch.

