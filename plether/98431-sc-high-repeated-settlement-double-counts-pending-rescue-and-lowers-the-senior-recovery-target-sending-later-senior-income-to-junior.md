# \#98431 [SC-High] Repeated settlement double-counts pending rescue and lowers the Senior recovery target, sending later Senior income to Junior

**Submitted on Nov 8th 2026 at 00:27:52 UTC by @Martelle88622 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98431
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

The pool holds cash from two liquidity groups. Senior is paid first. Junior is paid second.

Senior has a stored recovery target. It is called the high-water mark in code. After a loss, later income must refill Senior up to that target first. Only the rest goes to Junior. The spec states both rules this bug breaks. It says “Losses do not reduce the Senior high-water mark”. It also says pending inflows are held back in the split order “to prevent counting the same inflow twice” (`ACCOUNTING_SPEC.md:336-340`).

When the owner rescues a weak pool, the rescue is recorded as pending rescue money. It is not yet split into Senior or Junior ownership. A settlement step can credit only the part backed by real pool resources. The rest stays pending. It is held back from withdrawals. It should be applied later when backing arrives. The protocol’s own test states this directly. It says the leftover “must remain pending” and “should preserve senior’s recovery target” (`HousePoolWaterfall.t.sol:201-202`).

The problem: a second settlement with no new money does not treat the leftover as already counted. It subtracts the leftover from the same pool value a second time. This creates a fake loss that wipes the Senior balance the first step just created. It then rebuilds Senior from the leftover alone. It sets the recovery target to that leftover alone. The Senior target falls silently.

A concrete run from the proof test, in USDG (all values come from live trading, see Reachability):

1. Live trading leaves the pool with $142,028.00 cash and a $179,904.00 trader debt. The debt is created by the normal close path when a winning payout is larger than cash left. Split-ready value after rescue is the shortfall: $37,875.99.
2. The owner performs the documented rescue of $75,751.99 (exactly twice the shortfall). Pending rescue is $75,751.99. Split-ready value is $37,875.99.
3. First open settlement: $37,875.99 is credited to Senior. Senior balance is $37,875.99. The recovery target is the full $75,751.99 rescue. Pending rescue left is $37,875.99. Free cash is zero.
4. A second open settlement runs with the same cash, debt, and target. It holds back the remaining $37,875.99 from the same $37,875.99 of value. So the split order sees a total loss and wipes the Senior balance it just created. It then applies the $37,875.99 again. Senior balance is $37,875.99 again. But the recovery target is now $37,875.99. Pending rescue is gone. Free cash is $37,875.99.
5. A trader then loses $7,500.00 in a live trade. The loss is recorded as pool income:
   - After the second settlement (buggy): the $7,500.00 goes to Junior. Senior stays at $37,875.99. A Junior holder then withdraws real USDG against it in the test.
   - Without the second settlement (control): the same $7,500.00 restores Senior. Senior rises to $45,375.99 on the way to the $75,751.99 target. Junior gets nothing.

The second settlement is open to any account. Any Junior holder can create a matured zero-value withdrawal round. At that point Junior balance is zero, and a full-balance request is exempt from the minimum size. Then anyone can call the public `settleLpEpoch` step. A zero-value refund counts as real progress, so the second update is kept instead of reverting. No special role, false price data, or bad input is needed at any step.

## Reachability: only public steps, no special setup

Every state in the proof test is reached through normal entry points. The test has no storage writes (`stdstore` / `vm.store`). It has no fake calls as a protocol contract. It has no token burn. It has no direct call to a vault-only, engine-only, or router-only function. The full caller map:

| Step | Caller | Entry point | Special rights |
| --- | --- | --- | --- |
| LP deposits (Senior $200,000; Junior holders A and B $10 each) | the LPs | `TrancheVault.requestDeposit` then open `HousePool.settleLpEpoch(0, 0)` then `TrancheVault.claimDepositShares` | none |
| Share move giving holder C one share unit | Junior holder A | `TrancheVault.transfer` (plain token move) | none |
| Trader deposits | the traders | `MarginClearinghouse.deposit` | none |
| Winning SHORT opened at 0.50; winning LONG opened at 2.00 and closed at 0.50 and paid in cash; SHORT closed at 2.00 | traders commit, an unrelated keeper runs them | `OrderRouter.commitOrder`, `OrderRouter.executeOrder`, `OrderRouter.updateMarkPrice` with mock price updates | none (keeper is open to any account; prices stay inside the allowed `[0, CAP_PRICE]` range and use the protocol’s own test price points) |
| Trader debt of $179,904.00 created | normal close path | `CfdEngineSettlementSidecar` records the unpaid payout as a trader debt (spec: a price payout that cash cannot cover “becomes a same-account trader claim”, `ACCOUNTING_SPEC.md:1052-1053`) | none |
| Owner rescue of $75,751.99 and degraded-mode clear | the real engine owner (the test contract is the deployer) | `CfdEngine.recapitalizePool`, `CfdEngine.clearDegradedMode` | owner only, and this is the documented recovery step itself, not the exploit |
| Attack: full-balance withdrawal requests (A and B for the two passes, C’s one-unit head for the income settle, B again for cash-out) | Junior holders A, B, C | `TrancheVault.requestRedeem` | none |
| Attack: the two (later three) settlements | a fresh address with no role | `HousePool.settleLpEpoch(0, 0)`, the open path when no positions are open | none |
| Income: LONG opened at 0.50, closed at 2.00 at a $7,500.00 loss | trader commits, unrelated keeper runs it | `OrderRouter.commitOrder` / `executeOrder`; loss recorded as pool income by the settlement sidecar | none |
| Junior cash-out | Junior holder B | `TrancheVault.requestRedeem` then open settle then `TrancheVault.claimRedeem` | none |

The only owner calls are the rescue and the degraded-mode clear. They use the documented recovery entry points. That matches the “trusted role takes a normal action that sets off a bug for users” case. The rescue is the designed response to a trader-debt shortfall. The state itself is a designed state (“unpaid profit becomes a trader debt”). Everything that goes wrong after it is started by users with no rights. The bug is not that the owner rescued. The bug is that a second open settlement uses up the rescue leftover and rewrites the Senior target. The trader leaving his debt unpaid is not required: paying the debt lowers cash and debt by the same amount, so split-ready value does not change.

## Vulnerability Details

Some words first, so this reads on its own:

- Split-ready value (`D` in code as `distributable`): pool cash minus trader debts, plus or minus open position effects. In the proof test there are no open positions at settlement time, so it is cash minus debts.
- Pending rescue (`P` in code as `pendingRecapitalizationAssets`): rescue cash sent by the owner but not yet split into Senior or Junior ownership.
- Split order (called waterfall in code): first hold back pending rescue, then compare the rest with existing Senior plus Junior balances. A shortfall cuts Junior first, then Senior. Extra income first restores Senior up to its recovery target, then adds to Junior.
- Recovery target (called high-water mark in code): the Senior amount that later income must restore. Losses must not lower it.
- Update step (called reconcile in code): the regular update that re-splits value between Senior and Junior. Settlement is hourly and open to any account when no positions are open. `HousePool.settleLpEpoch` first runs this update, then funds or refunds the matured withdrawal head.

What is wrong, in the order it happens:

1. `HousePool._reconcile` builds a snapshot, caps pending buckets against `D`, plans the split with the capped amount, applies the split, then applies the capped buckets to live balances: `HousePool.sol:1555-1599`.
2. The cap uses the current total `D`, not the extra value above already credited balances: `HousePool.sol:1829-1834`.
3. `planReconcile` subtracts the capped bucket amount from `D` before it compares with existing balances: `HousePoolReconcilePlanLib.sol:64-105` (subtract at `:68-71`, loss cut at `:102-105`).
4. On an empty split order, the rescue apply step sets Senior balance to the applied amount. It sets the recovery target to `max(recapitalizationTargetAssets, amount)`: `HousePoolPendingPreviewLib.sol:111-135` (empty start at `:118-122`).
5. The target passed is the current unpaid pending bucket, not the original total rescue: `HousePool.sol:1759-1776` with `HousePoolPendingPreviewLib.sol:40-57`.
6. The pending bucket is then lowered by the full split-ready amount: `HousePool.sol:1787-1792`.
7. A zero-value withdrawal refund counts as progress: `HousePool.sol:944-987`. The no-progress check accepts it: `HousePool.sol:1179-1185`.
8. A full-balance withdrawal request is exempt from the minimum withdraw size, so a zero-value head can be created: `TrancheVault.sol:1611-1617`.
9. The settlement entry itself is open to any account when no positions are open: `HousePool.sol:797-821`.

Exact state trace (live proof values, USDG):

| Step | Cash | Debt | D | Pending P | Senior balance | Senior target | Free cash |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Rescue done | 217,780.00 | 179,904.00 | 37,875.99 | 75,751.99 | old pre-loss balance (wiped inside pass 1) | old target kept | 0 |
| First settle | 217,780.00 | 179,904.00 | 37,875.99 | 37,875.99 | 37,875.99 | 75,751.99 | 0 |
| Second settle | 217,780.00 | 179,904.00 | 37,875.99 | 0 | 37,875.99 | 37,875.99 | 37,875.99 |
| +$7,500 income, settle (buggy) | 225,280.00 | 179,904.00 | 45,375.99 | 0 | 37,875.99 | 37,875.99 | n/a |
| +$7,500 income, settle (control) | 225,280.00 | 179,904.00 | 45,375.99 | 0 | 45,375.99 | 75,751.99 | n/a |

Why the second pass does what it does:

1. After the first pass, Senior balance is `D` and the pending leftover is `P - D`.
2. The second pass caps the leftover at `D` again. It holds back `min(P - D, D)` from the same `D`, even though `D` already backs the Senior balance from the first pass.
3. If the leftover is at least the balance (`P - D >= D`, which is `P >= 2D`), the held-back amount leaves zero to compare. So the split order applies a loss equal to the Senior balance and wipes it.
4. The same capped amount is then applied to the empty split order. The target used is the leftover bucket. So the new recovery target is `max(P - D, D) = P - D`, not the original `P`. The target falls by `D`.
5. If `P < 2D`, the loss is only partial. The apply step fills the gap and the target holds. So the bug needs exactly `P >= 2D`. In pre-rescue terms, with shortfall `S = debt - cash` and rescue `R`, `D = R - S`. The range is `S < R <= 2S`. That is any rescue that more than covers the shortfall but not more than twice. The protocol’s own pending rescue test uses `P = 100,000` and `D = 49,960`. It meets this range. That test only checks one update.

The later income shift: with the original target `P` and Senior balance `D`, income must fill `P - D` before Junior gets anything. After the second pass, the target is `P - D` and the gap is `P - 2D` (zero at `P = 2D`). The Senior gap that disappears is `D`. For income at least that gap, Junior gets up to `D` more than in the control. The proof test sits at `P = 2D`. So the whole next income up to `D` is moved. The measured $7,500.00 is part of a possible $37,875.99.

Why the guards do not stop it:

- The zero-value refund is a real processed round, so `NoLpEpochProgress` does not fire and the update is not rolled back.
- The price freshness, reentry, and inflow checks do not require the update to give the same result twice.
- Preview and live run compute the same wrong result, so a match check cannot catch it.
- `reconcile()` itself is vault-only, but `settleLpEpoch` is open and runs the same update before it funds the head. The caller needs no vault rights.
- A further owner rescue does not undo it: the lowered target is the stored state. New rescue money would have the owner pay a second time to restore what the first rescue already paid for.

## Why this is not expected behavior

**Not expected behavior.** The design papers promise the opposite result. The split spec holds back pending inflows exactly “to prevent counting the same inflow twice”. It also states “Losses do not reduce the Senior high-water mark” (`ACCOUNTING_SPEC.md:336-340`). The protocol’s own split test covers the first settle. It says the leftover “must remain pending”. It says the recovery target is kept. It says the leftover settles later “once backing becomes available” (`HousePoolWaterfall.t.sol:177-220`). The second same-state settle breaks all three statements in one move. It counts the leftover against the same backing a second time. It creates a loss that lowers the target. And it uses up the leftover with no new backing. This is also not a timing shift between equal holders. Senior and Junior are different groups with a fixed priority. The target is a stored promise in the spec. And the proof test’s Junior holder leaves with cash that the control run pays to Senior.

## Impact Details

- The Senior recovery target drops by `D` ($37,875.99 in the proof test). Up to `D` of the next paid income goes to Junior instead of restoring Senior. The proof test uses the first $7,500.00 of that range through a live losing trade. The buggy run ends Senior $37,875.99 and Junior $7,500.00. The control run ends Senior $45,375.99 and Junior $0. A Junior holder then claims $73.79 of real USDG. That is his share from a $10 deposit of the moved $7,500.00. A larger Junior holder takes more, up to the full amount. The claimed harm is exactly this group-to-group move of recovery income. This is theft of unclaimed yield, which the competition rates High.
- The freed pending hold-back also raises free cash from 0 to `D`, so the protocol stops holding back the leftover rescue. This is a side effect. It is not claimed as a freeze. The claimed harm is the Senior recovery promise and the income shift.
- The caller needs only Junior shares and gas. The zero-value withdrawal heads are normal full-balance requests. The settlements are the open keeper path. There is no price fee beyond the public price update, no position, no bounty, and no special role.
- The state needed is the recovery state: an open trader debt (or equal debt) makes `D` smaller than the rescue `P`, with `P >= 2D`. That is the same as an owner rescue in `(S, 2S]` over the shortfall `S`. Both the proof test’s live trading state and the protocol’s own pending rescue test state meet this range. So it is not made up.
- The attack works once per pending rescue bucket. A new rescue or a new pending bucket is needed to repeat it. It is not a repeat drain. The amount is capped by `D`, at most half the rescue. This is why the report claims High, not Critical. No saved principal is taken and no lasting freeze happens. The loss is only future recovery income that moves to Junior. And the written rule “losses do not reduce the target” is broken. The target drop needs `P >= 2D`, which matches the protocol’s own test state (100,000 pending, 49,960 split-ready). The income move is proven in cash by the Junior cash-out in the test.

## References

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/HousePool.sol` (update, pending cap, live bucket apply, bucket lower, zero-value refund progress, no-progress check, open settlement)
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol` (hold-back before the split order)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol` (empty-order start and target choice)
- `packages/perps/src/TrancheVault.sol` (full-balance request exemption, refund path)
- `packages/perps/src/CfdEngine.sol:632-653` (`recapitalizePool`, the documented owner recovery), `packages/perps/src/CfdEngineSettlementSidecar.sol:600-623` (live income record; unpaid payout recorded as a trader debt)
- `packages/perps/ACCOUNTING_SPEC.md:336-340` (split rules: losses do not lower the Senior target; pending inflows are held back to stop double count), `:1052-1053` (unpaid price payout becomes a trader debt)
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol`, `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` (the first pass is checked; the leftover is checked to keep the Senior recovery target; no second same-state update is tested)
- `packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol`, `test_PostWipeoutRequiresExplicitRecapForHwmReset` (an explicit rescue resets the target; no repeat-update case)

## Link to Proof of Concept

https://gist.github.com/a-sklyarov2/af8b6271e4ab98d92890597b81f7373e

## Proof of Concept

## Proof of Concept

Runnable Foundry test file at pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It has no in-scope changes (zero diff under `packages/perps/src`). Mocks stand in only for the price updates and the USDG token, as the competition rules allow. Full file in the secret gist linked in the gist field below (filename `PocSeniorHwmReconcile.t.sol`). Both tests pass.

To run: in a checkout at that commit, place the gist file at `packages/perps/test/perps/PocSeniorHwmReconcile.t.sol`, then from the repo root run:

```bash
forge test --root packages/perps --match-path test/perps/PocSeniorHwmReconcile.t.sol -vv
# [PASS] test_SecondReconcileResetsHwmAndDivertsRevenueToJunior()
# [PASS] test_SingleReconcileControlKeepsHwmAndSeniorRestoration()
```

Privilege check of the proof test (also stated in the file header): every LP, trader, keeper, and settler call goes through open entry points from normal accounts. The settlement caller and the order keeper are fresh addresses with no role. Orders use the normal V3 `OrderRouter.commitOrder(OrderRequest)` / `executeOrder` path, not the old test helper, and LP settlement uses the open no-position `HousePool.settleLpEpoch(0, 0)` path. There is no `stdstore` / `vm.store`, no fake call as any protocol contract, no token burn, and no call to a vault-only, engine-only, or router-only function. The only owner calls are the documented recovery itself (`recapitalizePool`, `clearDegradedMode`), made directly by the test contract, which is the engine owner. That is the start state, not the exploit. Time moves forward only so hourly LP rounds can mature. The trader debt, the pool losses, the rescue, and the income all come from live order flow.

What the test does:

1. A Senior victim deposits $200,000. Two Junior holders deposit $10 each. All use the normal vault flow (request, open settle, claim). Holder A moves one share unit to holder C by a plain token move. A winning SHORT (120,000 at 0.50) is opened through the open router. A winning LONG (40,000 at 2.00, closed at 0.50) is paid in cash by the pool. The SHORT then closes at 2.00. Its payout is larger than the cash left. So it becomes a real trader debt of $179,904.00 against $142,028.00 of pool cash (shortfall $37,875.99). All orders are committed by the traders and run by an unrelated keeper against mock price updates.
2. The engine owner rescues with $75,751.99 (twice the shortfall) through the real `CfdEngine.recapitalizePool` and clears degraded mode. The test checks pending rescue $75,751.99, split-ready $37,875.99, `P = 2D`.
3. Holder A requests a full-balance Junior withdrawal. An unrelated account calls `pool.settleLpEpoch(0, 0)`. The update credits $37,875.99 to Senior. It sets the recovery target to the $75,751.99 rescue. It leaves $37,875.99 pending. It refunds A’s zero-value head. Checked: Senior $37,875.99, target $75,751.99, pending $37,875.99, free cash 0.
4. Holder B requests a full-balance withdrawal in the next round. The same unrelated account settles again. The second update holds back the remaining $37,875.99 from the same pool value. It wipes and rebuilds the Senior balance. It uses up the pending bucket. It lowers the target to $37,875.99. Checked: Senior $37,875.99, target $37,875.99, pending 0, free cash $37,875.99.
5. A third trader loses $7,500.00 in a live LONG trade (opened 0.50, closed 2.00), recorded as pool income by the settlement sidecar. Holder C’s one-unit head matures and a third open settlement splits the income. Checked: Senior stays $37,875.99 and Junior rises to $7,500.00. Holder B then claims back his refunded shares. He requests again. He is funded from Junior balance. He claims $73.79 of real USDG. That is his share from a $10 deposit of the moved $7,500.00. A larger Junior holder takes more, up to the full amount. The moved income is cash, not just a record.
6. The control test repeats the same live recovery state and the same income trade, but runs only the first settlement before the income. The leftover rescue settles and the income restores Senior. Checked: Senior $45,375.99, target $75,751.99, Junior $0. The only change between the runs is the attacker’s second open settlement.
Measured end states (USDG units, 6 decimals; the test logs them with `-vv`):

```text
recovery claim:        179904000000   pool cash before rescue: 142028001238
split-ready D:          37875998762   owner rescue P = 2D:      75751997524
paid income:              7500000000
buggy:   Senior 37875998762  Junior 7500000000   target 37875998762  (Junior holder B cash claimed: 73785472)
control: Senior 45375998762  Junior 0            target 75751997524
```
