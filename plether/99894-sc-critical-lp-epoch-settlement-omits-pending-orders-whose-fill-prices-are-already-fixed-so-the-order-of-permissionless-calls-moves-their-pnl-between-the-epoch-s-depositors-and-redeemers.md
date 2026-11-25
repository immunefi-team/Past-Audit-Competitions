# \#99894 [SC-Critical] LP epoch settlement omits pending orders whose fill prices are already fixed, so the order of permissionless calls moves their PnL between the epoch's depositors and redeemers

**Submitted on Nov 15th 2026 at 17:09:05 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99894
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value". If the judge reads it as informational, it is the Insight.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`: the Router's LP settlement path `OrderLiquidationBatchLogic.settleLpEpoch` (`router/OrderLiquidationBatchLogic.sol:549-581`) and `HousePool.settleLpEpoch` (`HousePool.sol:797-821`), together with the delayed-order mark update `OrderRouterExecutionSidecar._updateEngineMark` (`OrderRouterExecutionSidecar.sol:1053-1060`).

**Relation to prior findings:** the closest known issue is the AI scan's "Order execution accepts oracle updates older than the engine's cached mark". It describes the behaviour this report builds on, but a different harm (a trader choosing a favourable older price against the pool) with a different fix location. This report's root is in LP settlement. See "Relation to prior findings".

## Summary

LP deposits and redemptions settle once per hourly epoch, through the permissionless `OrderRouter.settleLpEpoch`, at a freshly pushed mark. `SECURITY.md:849-860` ("Exact symmetric LP NAV") promises that LP entry and exit price off the same terminal snapshot, and that "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV".

A delayed order's fill price is fixed as soon as its unique historical tick, the first tick after `commitTime` (within 15 s, `PletherOracle.sol:87`), is published. The order can still wait to execute until its deadline (60 s by default, `router/OrderExecutionOrchestrator.sol:11`). Settlement NAV is the pool's physical assets, less trader claims, plus or minus the terminal price delta of the Engine's **open** positions (`libraries/HousePoolAccountingLib.sol:60-77`, `TerminalNavBookV2.sol:208-237`). A pending order's PnL from its fixed fill to the settlement mark is not part of it, and the settlement path does not read the order queue (`router/OrderLiquidationBatchLogic.sol:549-581`).

So when an epoch boundary passes while such an order is pending, two permissionless calls race:

- **execute → settle:** the position exists at settlement, and its marked PnL is in the price at which the epoch's redeemers exit and its depositors enter.
- **settle → execute:** settlement prices both directions without it. The order then opens at its old fill. The Engine keeps the newer settlement mark (`OrderRouterExecutionSidecar.sol:1053-1060`), so the position's marked PnL against that mark is in place immediately. It falls on whoever holds shares after settlement, including the depositor who has just entered at a NAV that did not carry it. The redeemer has already left at a NAV that did not carry it.

Both orderings end with the same position (size, margin, entry) and the same Engine mark. Only the split between the epoch's entering and exiting LPs differs.

**At the release parameters**, the ordering effect reproduces. The run uses the pinned release script's own deployment (`script/DeployPerpsArbitrumSepolia.s.sol`) and the live v1.3.1 bootstrap: VPI factor 0.01, skew cap 40%, a 500M junior seed and a 10M senior seed. A 20M LONG is pending at the boundary. About 50M of junior is redeemed and 50M deposited, each about 7% of ~700M junior. With **no price move at all**, settle → execute and execute → settle differ by **185.29 USDC** between the redeemer and the depositor. The pending open's execution charge (its VPI, 2,594.08 USDC into junior NAV) lands on whichever cohort the call order picks. A price move adds a second part, about 136 USDC per basis point in this configuration. In the measured direction it offsets the VPI part: 50.79 USDC at 1 bp and 497.68 at 5 bp. The account that orders the calls receives nothing, and an LP who holds through is unchanged. At the fixture sizes used below, the 500M release seed dilutes the effect to 0.048 USDC at a 1% move.

**In the fixture configuration** of the PoC below (VPI 0, ~201,000 junior USDC; see "Fixture notes"), a 50,000-token LONG fills at 1.00 at `b − 9` (`b` is the epoch boundary) and settlement runs at 0.99 at `b + 1`. 50,000 of about 201,000 junior USDC is redeemed and 50,000 deposited. Under settle → execute, the redeemer receives **124.378110 USDC more** and the new depositor's shares are worth **124.378126 USDC less** than under execute → settle. An LP who holds through the epoch is unchanged in both orders. The mirror case (the market moves against the trader) moves the same amount the other way. The transfer is linear in the move: 1.24 USDC at 1 bp.

The same root covers more than one fresh open. The PoC shows two further cases (fixture configuration):
- **Pending closes.** A trader whose position is open at settlement commits a full close at `b − 10`, with its fill fixed at `b − 9`. Settlement marks the still-open position at the new price, and the close then realizes at the old fill. The redeemer's payout differs by **±124.38 USDC** between the two orderings at a 1% move (124.378153 and 124.378066), about the same as the open case. The depositor's change is not the exact negative of the redeemer's here (see the PoC section). So the precondition can be an existing trader's exit in the seconds before a boundary, if that close is otherwise executable, not only a fresh open. The PoC shows one fully funded full close; partial closes and other configurations are not tested.
- **More than two outcomes: a prefix of the queue.** Three unrelated traders' opens are pending at the boundary (LONG 50,000, SHORT 80,000 and LONG 40,000, all filled at 1.00; settlement at 0.99). Executions are FIFO and permissionless. So a settler can execute the first `k`, settle, and leave the rest for afterwards, if its calls land in that order before another permissionless keeper executes the head or settles. This is a feasible scheduling choice, not exclusive control, and it is not a free permutation of the queue: only FIFO prefixes are available, and expiry, slippage, collateral or solvency checks can rule some out. The test has no competing keepers; it shows that all four prefixes are feasible for this queue. The redeemer's payout for `k` = 0, 1, 2, 3 is 49,999.99, 49,875.61, 50,074.62 and 49,975.12 USDC. **That is a 199.00 USDC range, against 24.88 USDC between "all after" and "all before" (8×).** All four final states hold the same three position sizes. The most a settler can move is the spread between the highest and lowest redeemer payout over the feasible prefixes of the pending fixed fills at the boundary, which include orders of accounts it does not control.

The amount is small. At release liquidity, a material amount needs a large pending order plus entering and exiting cohorts that are each a large share of junior. Even then it is about 0.1-1.0 bp of the losing cohort's position in the measured runs (50.79-497.68 USDC on a 50M cohort; 185.29 USDC, about 0.4 bp, with no move). The other cohort gains the same amount. When the move runs the other way, the price part adds to the VPI part instead of offsetting it. At 5 bp that is roughly 870 USDC (about 1.7 bp), derived from the measured parts, not run. A competing keeper that executes at the fill tick removes the price part for orders whose fill precedes the boundary, but not the VPI part. The protocol in aggregate loses nothing. The Impact section states the limits, and the first objection a reviewer will raise is answered under "The first objection".

## Relation to prior findings

**AI scan, "Order execution accepts oracle updates older than the engine's cached mark"** (info; a known issue under the program's rules). It describes the enabling behaviour: an order can execute at a historical price older than the Engine's cached mark, and the mark is not rewound. It frames the harm as a self-executing trader who keeps a favourable older update, waits for a newer adverse mark and opens at the old price against the pool. At the pinned commit the order's execution tick is the unique first tick after commit (`PletherOracle.sol:596-640`), so the trader no longer chooses the price. A late execution at the historical fill is the design, and this report does not dispute it. This report's root is that **LP epoch settlement does not account for that already-fixed fill**. Its harm runs between LPs, the epoch's depositors against its redeemers, not between trader and pool. The fix belongs in settlement, not in order execution.

**Cecuro MED-18** (fixed): its fix note says an older historical fill cannot rewind a newer mark. That is the behaviour this report relies on, not its root.

**Cecuro MED-7** (acknowledged): a keeper's choice of basket component. Its comment says a sole keeper already has execution-timing discretion. That concerns which tick an order executes at, not which LP cohort bears an order whose price is already fixed. **Cecuro HIGH-1** (acknowledged): a trader commits after seeing a tick. This report does not need the trader to know anything at commit.

**AI scan, permissionless settlement timing in the frozen window:** the same class (who times a permissionless settlement), but a different root (the settlement fee).

**`SECURITY.md:1044-1052`** (LP request cutoff) says the cutoff "is not an immutable settlement snapshot" and that "trading, closes, liquidations, claims, carry, and pool accounting can change NAV" before settlement. That covers state that changes before settlement and that settlement then prices. Here the order's price is already fixed before settlement, and settlement does not price it at all. Whether it lands in the entering or the exiting cohort is decided after the boundary, by which permissionless call lands first. Nothing in the Known Limitations says settlement excludes pending orders.

The SC Audit Studio pre-audit, the collaborative audit and the Cantina report have no epoch-settlement ordering item.

## Vulnerability details

The Router's settlement path pushes the current tick, installs it as the Engine mark and settles the pool against it. It never looks at the order queue (`router/OrderLiquidationBatchLogic.sol:561-563`):

```solidity
        IPletherOracle.PriceSnapshot memory snapshot =
            oracle.updatePrice{value: pythFee}(msg.sender, pythUpdateData, IPletherOracle.PriceMode.PoolReconcile);
        host.engine().updateMarkPrice(snapshot.markPrice, snapshot.publishTime);
```

`HousePool.settleLpEpoch` (`HousePool.sol:797-821`) then takes one accounting snapshot. Its price exposure comes from the terminal NAV book, which values the Engine's open-interest curves at the Engine mark (`TerminalNavBookV2.sol:208-237`). Terminal equity is physical assets, less trader claims, plus or minus that delta (`libraries/HousePoolAccountingLib.sol:60-77`). Redemptions and deposits of the matured epoch are priced from it (`HousePool.sol:835-938`). A pending open has no position yet, so its PnL from its fixed fill to this mark is not in the snapshot. A pending close does have an installed position, and the snapshot values it at this mark; what the snapshot omits is the pending close's adjustment to that position (below).

The pending order still executes afterwards at its fixed historical fill. The Engine mark is updated only if the execution tick is newer (`OrderRouterExecutionSidecar.sol:1053-1060`):

```solidity
        if (result.publishTime > engine_.lastMarkTime()) {
            engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
        }
```

The fill tick is older than the settlement mark, so the newer settlement mark stays. The new position's PnL against that mark, (mark − fill) × size in the LPs' direction, enters NAV right after the epoch's LPs have been priced without it.

Both entry points are permissionless: `OrderRouter.executeOrder` (`OrderRouter.sol:240`) and `OrderRouter.settleLpEpoch` (`OrderRouter.sol:479`, "Permissionless and available while the router admin is paused"; `SECURITY.md:506-513`: "Keepers are permissionless executors").

For a balanced epoch away from caps, the amount moved is about `(redeemed share of the tranche's equity) × |mark − fill| × size`. In the PoC that is `(50,000 / 201,000) × 500 ≈ 124.38` USDC. With unequal deposits and redemptions, tranche waterfalls, caps or partial funding the split is different, and the PoC does not cover those cases.

A close is the same omission from the other side. The position is open at settlement, so it is valued at the settlement mark `M`. But its close fill `P_f` is already fixed, and executing the close afterwards realizes `(P_f − M) × size` against post-settlement holders. With several pending orders `i = 1..N` (FIFO by commit time; `executeOrder` is permissionless on the head), a settler can execute a prefix `k` before settling, for any `k` that the FIFO, expiry and solvency checks leave executable. The redeeming cohort's payout then moves by about `(redeemed share) × Σ_{i≤k} s_i (M − P_i)`. The settler chooses `k` only if its calls land in that order before another keeper executes the head or settles. Mixed directions make an interior prefix more extreme than either end, as in the PoC.

## Impact

**Who acts, and who pays.** Either call can be submitted by any account. Whoever lands first chooses the ordering, but does not bear the result. The loss lands on a third party: the depositor (when the market moved in the trader's favour) or the redeemer (when it moved against the trader). That party took no action after making its LP request before the cutoff. An LP already in the settling epoch can pick the ordering that favours it. The PoC shows the redeemer calling `settleLpEpoch` herself before the order keeper and receiving the 124.378110 USDC. No adversary is needed at all: the result also follows whenever the settlement keeper simply runs before the order keeper. At the release parameters, a settler with no shares and no position receives no settlement token and ends with the same ETH in both orders. `OrderLiquidationBatchLogic.sol:556-579` forwards only the quoted Pyth fee and refunds the rest. So the account that orders the calls gains nothing, and a third party bears the result.

**What the contract fails to deliver.** A depositor is promised entry at a NAV that already carries the discount for old trader obligations (`SECURITY.md:854-855`). In the settle-first order the depositor enters without that discount and then bears part of an obligation whose price was fixed before the boundary. A redeemer is likewise promised exit at the same snapshot, and in the mirror case leaves without a gain that was already fixed. The value moves between LPs of one tranche. In the single-open case an LP who holds through is unchanged, and the protocol loses nothing in aggregate. This matches the program's Low impact "Contract fails to deliver promised returns, but doesn't lose value".

**Limits, stated plainly:**
- The order must be pending when the epoch settles, with its fill tick already published. In practice it was committed within its execution window before the boundary (60 s by default, `router/OrderExecutionOrchestrator.sol:11`; the governance limit is 1 h, `OrderRouterAdmin.sol:31`) and has not yet been executed. Ordinary keepers execute within seconds of the fill tick, so this is mostly orders committed in the last seconds before a boundary, or orders behind a held FIFO head. It can happen at most once per epoch (hourly). The order can be an open or a close, so an existing trader's ordinary exit qualifies. All such orders at one boundary count together, and a settler whose calls land first can choose how many of them execute first (the FIFO-prefix test); another permissionless keeper can act before it.
- A competing keeper narrows the window, measured at the release parameters:
  - A keeper that executes at the fill tick's own second (`b − 9`) leaves no order pending at settlement.
  - What survives is an order committed in the last second before `b`. Its fill tick is `b` itself, so the settler and the keeper race inside that second. The VPI part still moves 185.29 USDC there with no price move.
  - One second of keeper latency at a 1 bp move leaves 50.79 USDC.
  - An unprivileged contract can settle and execute in one transaction. A competing keeper's later execution of the same order then reverts, so the settler needs to land first only once.
- At the release parameters the amount has two parts:
  - The **VPI part** needs no price move: 185.29 USDC with a 20M pending open and ~7% cohorts.
  - The **price part** is linear in the move between the fill tick and the settlement mark: about 136 USDC per bp in the same configuration.
  - Measured: 185.29 USDC flat, 50.79 at 1 bp and 497.68 at 5 bp. In this direction the two parts offset; in the other they add.
  - At the PoC's fixture sizes, the 500M seed dilutes the effect to 0.047944 USDC at 1%.
- Measured in the fixture configuration (VPI 0, ~201,000 junior; see "Fixture notes"): 1.243781 USDC at 1 bp, 6.218906 at 5 bp, 24.875625 at 20 bp, 124.378126 at 100 bp. The PoC's 1% is a stress illustration.
- The beneficiary must already be in the settling epoch. The five-minute request cutoff (`TrancheVault.sol:39`) prevents seeing a pending order and then joining, and a redeemer needs cooldown-aged shares. The actor chooses the ordering, not the cohort.
- A fixed fill is not an unconditional obligation. FIFO eligibility, expiry, slippage, collateral and solvency checks still decide whether the order executes. The PoC shows a case where it executes in both orderings.
- The depositor's loss is marked share value, not realized cash. The redeemer's gain is cash.
- Making the precondition yourself (committing an order seconds before a boundary) costs fees that exceed a basis-point-sized transfer. This is not presented as a profitable strategy.

### The first objection

"This is async noise. The window is under a minute, the amounts are a few basis points, and a pending order is not a position. In each ordering the epoch is still priced off one authenticated snapshot, `SECURITY.md:1044-1052` says the cutoff is not an immutable snapshot, and keeper timing discretion is accepted (Cecuro MED-7). Insight at most."

Our answer, point by point:
- **Narrow window.** It is narrow, but it recurs at every hourly boundary that has a pending order. Its edges are protocol parameters (the 60 s execution window, the 15 s fill search), not market conditions.
- **Small amounts.** Agreed, and the severity claimed is the lowest paid tier for that reason. At release liquidity it is about 0.1-1.0 bp of the losing cohort's position in the measured runs, with an equal gain to the other cohort. It is not zero, and part of it (the VPI part, 185.29 USDC on 50M cohorts) needs no price move at all. The price part scales linearly with pending notional and the move. It is also more than a coin flip between two outcomes: a settler whose calls land first can choose among the feasible FIFO prefixes of the pending queue (199.00 USDC range against 24.88 USDC for the two ends in the PoC).
- **Whose loss.** It is not the actor's own loss. The account that orders the calls bears nothing; the cost lands on a depositor or redeemer who did nothing after requesting.
- **"A live keeper executes first."** It shrinks the window but does not close it (release parameters). Orders whose fill precedes the boundary are executed before settlement and leave nothing to reorder. An order committed in the last second before `b` is still raced inside one second. There the VPI part moves 185.29 USDC with no price move, and one second of latency at 1 bp leaves 50.79 USDC.
- **"A pending order is not a position."** Its price is fixed before the boundary. Once it executes, its PnL against the settlement mark is an old trader obligation in the exact sense of `SECURITY.md:854-855`. Which cohort bears it is decided by a call race after the boundary, not by the market. The cutoff text covers changes that settlement prices; here settlement skips one it cannot see.

We accept that the docs do not promise call-order independence for unexecuted orders in so many words. If the judge holds that pending orders are outside the "Exact symmetric LP NAV" promise, this report is the Insight: LP settlement can be ordered ahead of orders whose fills are already fixed, which moves their PnL between entering and exiting LPs.


## Recommendation

Make LP epoch settlement wait for orders whose fill is already fixed, so that every order whose price is known at the settlement mark is in the settlement NAV. In the Router's `settleLpEpoch` path (`router/OrderLiquidationBatchLogic.sol:549-581`), before calling `HousePool.settleLpEpoch`, one option is to revert while the global FIFO head (`nextExecuteId`) is a pending order whose `commitTime` is earlier than the settlement snapshot's `publishTime`. The orders are FIFO by commit time, so checking the head is enough. Its fill tick then lies at or before the settlement mark. This covers closes as well as opens, and every pending order, not just one: settlement can then only run once no order with a fixed pre-settlement fill is left, so the settler has no prefix to choose. A second option is to execute, or expire, such orders first in the same call. Each order expires within its execution window (60 s by default), so either option delays settlement by at most one window, and the existing expiry and terminal rules prevent a held head from blocking settlement for longer. A regression test like `test_Property_DepositorDoesNotInheritFixedFillObligation` above covers it. This is a sketch; it has not been implemented or run against the maintainers' suite.


## Link to Proof of Concept

https://gist.github.com/Richter2552/02c222de349d148bb3a02fe83011aec8

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `AtomicLpEpochSettlementTestFixture` (`packages/perps/test/perps/shared/AtomicLpEpochSettlementFixture.sol`). After the fixture's `setUp` it uses public entry points only, from accounts with no role:

- `TrancheVault` request/claim.
- `HousePool.settleLpEpoch` while no position is open (permissionless then, `SECURITY.md:319`).
- The production V3 `OrderRouter.commitOrder(OrderRequest)`, from the trader's own account.
- `router.updateMarkPrice`, `router.settleLpEpoch` and `router.executeOrder`.

Pyth is the maintainers' `MockPyth`, with explicit per-feed payloads that satisfy the oracle's unique-historical-tick check. USDC is the fixture's mock token. Both are outside the scope. No protocol contract is pranked and no storage is written.

Fixture notes, disclosed:
- The maintainers' `BasePerpTest` deploys the Router as `LegacyOrderRouterHarness` (`test/utils/LegacyOrderRouterHarness.sol`). It is a subclass of `OrderRouter` that adds only a scalar `commitOrder` adapter (it builds a V3 request and delegatecalls the production commit) and an `IS_TEST` constant. It overrides no settlement or execution function, and this test does not use the adapter: it calls the V3 commit directly. So the code that settles and executes is the production Router's.
- `BasePerpTest` also seeds 1,000 USDC junior and 1,000 USDC senior through `initializeSeedPosition` (held by the test contract), so junior equity is about 201,000 USDC.
- The ticks carry zero confidence, with all basket feeds moving together. This is a controlled fixture choice, not a production-parameter calibration.
- The fixture's risk and pool parameters are valid contract configuration (the Engine constructor accepts them, `CfdEngine.sol:304-327`), but they are not the repository's initial release defaults:

| Parameter | Fixture (this PoC) | Initial release defaults |
|---|---|---|
| VPI factor | 0 | `0.01e18` |
| Maximum skew ratio | `1e18` | `0.4e18` |
| Maintenance / initial margin (bps) | 100 / 150 | 10 / 20 |
| Junior / senior seed | 1,000 USDC each | 1 settlement-token unit each (`1e6`) |
| Senior exposure limit / senior share limit | `type(uint256).max - 1` (effectively unbounded) / 9,999 bps | 40,000,000 units (`40_000_000e6`) / 8,000 bps |

Sources. Fixture: `packages/perps/test/perps/shared/AtomicLpEpochSettlementFixture.sol:124-145` and `packages/perps/test/perps/BasePerpTest.sol:115-116,168-208,255-260`. Release: `packages/perps/DEPLOYMENT.md:123-156` (checked-in script and constructor defaults, which that document says are not a claim about any live deployment) and `script/DeployPerpsArbitrumSepolia.s.sol:434-446`.

**The amounts this PoC prints are fixture-configuration measurements.** The same root was re-run at the release parameters: the release script's own deployment, VPI 0.01, skew cap 40% and a 500M junior seed (see "Release-parameter run" below). The ordering effect reproduces there. At this PoC's sizes the 500M seed dilutes it to 0.048 USDC. With a 20M pending LONG and 50M entering and exiting cohorts, it moves:
- 185.29 USDC with no price move, because the pending order's VPI (2,594.08 USDC) also lands on whichever cohort the call order picks;
- 50.79 USDC at 1 bp;
- 497.68 USDC at 5 bp.

The account that orders the calls receives nothing. A competing keeper that executes at the fill tick removes the price part for orders whose fill precedes the boundary. It does not remove the VPI part: a same-second race still moves 185.29 USDC.

Each test of the single-open case runs the same state twice from one snapshot: **A** = `executeOrder` then `settleLpEpoch`, **B** = `settleLpEpoch` then `executeOrder`, with the same ticks. Each of those runs asserts that B settles with the order pending and A with none, that the order executes in both, and that both end with the same position size, margin, entry (the fill, 1.00) and Engine mark and mark time. It also asserts the redeemer is fully funded. The two wider cases below make fewer assertions; each bullet lists exactly what its runs assert. The **property** tests state the behaviour the code should have and **fail** on the pinned code. The **control** and **characterize** tests pass.

The last section of the file covers the two wider cases, with the same LP setup:
- **Pending close.** TRADER opens a 50,000 LONG at 1.00 before the epoch, and it executes. TRADER then commits a full close at `b − 10`, with the fill at 1.00 at `b − 9`. Arm A executes the close and then settles; arm B settles and then executes the close. Both arms use the production V3 commit with `isClose = true`. Each arm asserts that the close executes, that TRADER is flat afterwards, and that the redeemer is fully funded; it also asserts the pending-close count at settlement (0 in A, 1 in B). The arms do not compare trader cash or claims, carry indexes or the final mark. The depositor's change is not the exact negative of the redeemer's in this case. At −1%, ALICE (redeemer) is paid 124.378153 USDC less in B, and CAROL's (depositor's) final value is 124.611013 higher. At +1%, ALICE is paid 124.378066 more, and CAROL is 124.146056 lower. The test does not account for the other balances (BOB, ALICE's retained shares, the seed holders, reserves, carry), so it shows the redeemer's payout depending on call order, not an exact depositor-to-redeemer transfer.
- **FIFO prefix.** T1, T2 and T3 commit opens at `b − 13`, `b − 12` and `b − 11` (LONG 50,000, SHORT 80,000, LONG 40,000; fills at 1.00). For each `k` from 0 to 3, the first `k` execute, then the epoch settles, then the rest execute. Each run asserts that every execution returns Executed, that the final sizes are 50,000, 80,000 and 40,000, and that the redeemer is fully funded. Entry, margin, the final mark and time, the pending counts at settlement, trader balances and claims are not compared across prefixes.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card8240S1SettleBeforePendingFill.t.sol packages/perps/test/perps/spec/lp/
cd packages/perps

forge test --match-contract '^Card8240S1SettleBeforePendingFillTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card8240S1SettleBeforePendingFillTest$' --match-test '^test_(Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35, via-IR). Result: the six control and characterize tests pass, and the five property tests fail. Excerpt of `forge test -vv` for the five tests of the single-open case (USDC amounts in 6-decimal units; prices in 8 decimals; order status 2 = Executed):

```console
[PASS] test_Characterize_RedeemerSettlesFirstHerself() (gas: 18324756)
Logs:
  settler = ALICE, -1%: ALICE gain / CAROL loss (USDC 6dp) 124378110 124378126
  BOB value A / B 99751234670 99751234681

[PASS] test_Characterize_Scaling() (gas: 73038662)
Logs:
  move (bps) / CAROL loss (USDC 6dp) 1 1243781
  move (bps) / CAROL loss (USDC 6dp) 5 6218906
  move (bps) / CAROL loss (USDC 6dp) 20 24875625
  move (bps) / CAROL loss (USDC 6dp) 100 124378126

[PASS] test_Control_FlatMark_OrderIrrelevant() (gas: 18077881)
Logs:
  flat, A execute->settle
    pending at settle / order status 0 2
    engine mark / time     100000000 1709546401
    ALICE redeem assets    49999990935
    CAROL value (final)    49999999999
    BOB value (final)      99999990912
  flat, B settle->execute
    pending at settle / order status 1 2
    engine mark / time     100000000 1709546401
    ALICE redeem assets    49999990935
    CAROL value (final)    49999999999
    BOB value (final)      99999990912

[FAIL: depositor inherits the fixed-fill obligation: 49875621873 !~= 49999999999 (max delta: 2000000, real delta: 124378126)] test_Property_DepositorDoesNotInheritFixedFillObligation() (gas: 19151042)
Logs:
  -1%, A execute->settle
    pending at settle / order status 0 2
    engine mark / time     99000000 1709546401
    ALICE redeem assets    49875612825
    CAROL value (final)    49999999999
    BOB value (final)      99751234670
  -1%, B settle->execute
    pending at settle / order status 1 2
    engine mark / time     99000000 1709546401
    ALICE redeem assets    49999990935
    CAROL value (final)    49875621873
    BOB value (final)      99751234681
  CAROL loss from call order (USDC 6dp) 124378126
  ALICE gain from call order (USDC 6dp) 124378110

[FAIL: redeemer payout depends on call order: 49999990935 !~= 50124369044 (max delta: 2000000, real delta: 124378109)] test_Property_RedeemerPayoutIndependentOfCallOrder() (gas: 18641950)
Logs:
  +1%, A execute->settle
    pending at settle / order status 0 2
    engine mark / time     101000000 1709546401
    ALICE redeem assets    50124369044
    CAROL value (final)    49999999999
    BOB value (final)      100248747153
  +1%, B settle->execute
    pending at settle / order status 1 2
    engine mark / time     101000000 1709546401
    ALICE redeem assets    49999990935
    CAROL value (final)    50124378126
    BOB value (final)      100248747142
  ALICE loss from call order (USDC 6dp) 124378109
  CAROL gain from call order (USDC 6dp) 124378127

```

The six tests for the pending close and the FIFO prefix (same run):

```console
[PASS] test_Characterize_FifoPrefixRangeExceedsBinaryArms() (gas: 59430376)
Logs:
  FIFO prefix, -1%
    k executed before settlement / ALICE payout / CAROL value 0 49999990935 49975124374
    k executed before settlement / ALICE payout / CAROL value 1 49875612825 50099688796
    k executed before settlement / ALICE payout / CAROL value 2 50074617800 49900608942
    k executed before settlement / ALICE payout / CAROL value 3 49975115313 49999999999
  range over all prefixes / over the binary arms (USDC 6dp) 199004975 24875622

[PASS] test_Control_FifoPrefixFlatMark_PrefixIrrelevant() (gas: 59026631)
Logs:
  FIFO prefix, flat
    k executed before settlement / ALICE payout / CAROL value 0 49999990935 49999999999
    k executed before settlement / ALICE payout / CAROL value 1 49999990935 49999999999
    k executed before settlement / ALICE payout / CAROL value 2 49999990935 49999999999
    k executed before settlement / ALICE payout / CAROL value 3 49999990935 49999999999

[PASS] test_Control_PendingCloseFlatMark_OrderIrrelevant() (gas: 22393047)
Logs:
  pending close, flat: ALICE A / B 50000005203 50000005160
  pending close, flat: CAROL A / B 49999999999 50000000043

[FAIL: redeemer payout depends on the settler's FIFO prefix: 50074617800 !~= 49875612825 (max delta: 2000000, real delta: 199004975)] test_Property_FifoPrefix_RedeemerPayoutIndependentOfPrefix() (gas: 62356646)

[FAIL: pending close: redeemer payout depends on call order: 49875627050 !~= 50000005203 (max delta: 2000000, real delta: 124378153)] test_Property_PendingCloseDown1pct_RedeemerPayoutIndependentOfCallOrder() (gas: 23923323)
Logs:
  pending close, -1%: ALICE A(execute->settle) / B(settle->execute) 50000005203 49875627050
  pending close, -1%: CAROL A / B 49999999999 50124611012

[FAIL: pending close: redeemer payout depends on call order: 50124383269 !~= 50000005203 (max delta: 2000000, real delta: 124378066)] test_Property_PendingCloseUp1pct_RedeemerPayoutIndependentOfCallOrder() (gas: 23821245)
Logs:
  pending close, +1%: ALICE A(execute->settle) / B(settle->execute) 50000005203 50124383269
  pending close, +1%: CAROL A / B 49999999999 49875853943

Suite result: FAILED. 6 passed; 5 failed; 0 skipped
```

Expected output on a fixed contract: all five property tests pass. The characterize tests then fail, because they pin the transfer. The three controls still pass.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| control: the settlement tick equals the fill price (1.00); the order of `executeOrder` and `settleLpEpoch` makes no difference | `test_Control_FlatMark_OrderIrrelevant` | passes |
| the market moves 1% in the trader's favour after the fill (0.99 at `b + 1`); the depositor's final value should not depend on the call order | `test_Property_DepositorDoesNotInheritFixedFillObligation` | **FAILS**: CAROL 124.378126 USDC lower, ALICE paid 124.378110 more |
| the market moves 1% against the trader (1.01); the redeemer's payout should not depend on the call order | `test_Property_RedeemerPayoutIndependentOfCallOrder` | **FAILS**: ALICE paid 124.378109 less, CAROL 124.378127 higher |
| the redeemer herself calls `settleLpEpoch` first: her gain is the depositor's loss (within 1,000 atoms), above 100 USDC; BOB, who holds through, is unchanged | `test_Characterize_RedeemerSettlesFirstHerself` | passes |
| the transfer at 1, 5, 20 and 100 bp is positive and linear; gain equals loss and BOB is unchanged at each | `test_Characterize_Scaling` | passes |
| control: a pending full close at a flat market; the order of the close and the settlement makes no difference | `test_Control_PendingCloseFlatMark_OrderIrrelevant` | passes |
| a pending full close (fill fixed at `b − 9`), settlement 1% lower; the redeemer's payout should not depend on the call order | `test_Property_PendingCloseDown1pct_RedeemerPayoutIndependentOfCallOrder` | **FAILS**: ALICE paid 124.378153 less (CAROL 124.611013 higher; not the exact negative) |
| the same close, settlement 1% higher | `test_Property_PendingCloseUp1pct_RedeemerPayoutIndependentOfCallOrder` | **FAILS**: ALICE paid 124.378066 more (CAROL 124.146056 lower; not the exact negative) |
| control: three traders' pending opens at a flat market; how many execute before settlement makes no difference | `test_Control_FifoPrefixFlatMark_PrefixIrrelevant` | passes |
| the same three orders, settlement 1% lower; the redeemer's payout should not depend on how many execute first | `test_Property_FifoPrefix_RedeemerPayoutIndependentOfPrefix` | **FAILS**: 199.004975 USDC range over `k` = 0..3 |
| the range over all prefixes is above 100 USDC and above 4× the range between the two ends (`k` = 0 and `k` = 3) | `test_Characterize_FifoPrefixRangeExceedsBinaryArms` | passes (199.004975 against 24.875622) |

### Release-parameter run (numbers in the Summary and Impact)

This second test checks that the root survives the shipped parameters and a competing keeper. It is not the inline PoC above. It runs against a state built by running the pinned repository's own release script, `DeployPerpsArbitrumSepolia.run()` (`script/DeployPerpsArbitrumSepolia.s.sol`), inside a Foundry test. That run keeps the script's order, parameters, release subclasses, wiring and `require` checks. The build then replays the live v1.3.1 bootstrap from the repository's release evidence (`deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.1/`): it mints and seeds 500M junior and 10M senior, refreshes the six-feed mark and activates trading.

Read back on chain in that state:
- vpiFactor `0.01e18`, maxSkewRatio `0.4e18`, maintenance and initial margin 10 and 20 bps;
- the Router's `minOpenNotionalAssets` = `1000e6`;
- senior caps of 40M and 8,000 bps;
- the live activation mark 0.95963705.

The test is about 400 lines. Its deployment step and the generated typed interfaces are not reproduced here, and we can provide them on request.

Where this state differs from the live Arbitrum Sepolia release (disclosed, none load-bearing for the result):
- **Pyth.** It is the maintainers' `MockPyth` at the release Pyth address, with the live feeds, exponents and activation prices. It verifies no signatures.
- **Deployer key.** It is a named key, so every contract address differs from the live one.
- **Chain.** The chain id and block are Arbitrum Sepolia's, and an `ArbSys` stand-in returns `block.number`.
- **Code.** The code is the pinned commit, not the 24-commit-older live build. The junior maintenance fee is 0 at the pin, against 100 bps live.

Every step after the build is a public call by an account with no role. No in-scope contract is pranked, stored or etched. Test sizes:
- **Large cohort.** ALICE and BOB hold 100M junior each next to the 500M seed. ALICE redeems about 50M and CAROL deposits 50M, each about 7% of ~700M junior.
- **Pending order.** TRADER's 20M LONG is committed at `b − 10`, with its fill at the live mark at `b − 9`. The settlement tick is at `b + 1`.
- **Settler.** SETTLER holds no shares and no position.

Result on the pinned code: 7 PASS, 5 FAIL by design.

| test | what it shows | result |
|---|---|---|
| `test_R1_S1_Control_Flat` | this report's fixture sizes (50k LONG; ALICE/BOB 100k; CAROL 50k), no move | passes (2 atoms) |
| `test_R1_S1_Characterize_Down1pct_DilutedToCents` | the same sizes, −1% | passes: ALICE moves 0.047944 USDC; the 500M seed dilutes the fixture's 124.38 |
| `test_R2_Property_Large_Flat_ExecutionChargesOrderingNeutral` | large cohort, **no price move** | **FAILS**: 185.291498 USDC moved (CAROL 185.291564 the other way; BOB within 21 atoms) |
| `test_R2_Characterize_ExecutionChargesRaiseJuniorNav` | attribution | passes: executing the open alone raises junior `totalAssets` by 2,594.080844; × ALICE's 7.14% ≈ 185.29 |
| `test_R2_Property_Large_OrderingNeutral_1bp` | large cohort, −1 bp | **FAILS**: 50.791492 (the price part offsets the VPI part) |
| `test_R2_Property_Large_OrderingNeutral_5bp` | large cohort, −5 bp | **FAILS**: 497.679962 (CAROL 497.680159) |
| `test_R2_Characterize_SettlerIndifferent_HolderUnchanged` | who gains and who loses | passes: SETTLER gets 0 settlement token and identical ETH in both orders; BOB unchanged; ALICE and CAROL move in opposite directions |
| `test_R3_Characterize_LiveKeeperBeforeBoundary_NoPendingOrder` | a competing keeper executes at the fill tick (`b − 9`) | passes: no order is pending at settlement |
| `test_R3_Property_SameSecondZeroMove_ChargesStillMove` | commit at `b − 1`, fill tick `b`, settlement tick `b` at the same price: the settler races the keeper inside one second | **FAILS**: 185.291498 |
| `test_R3_Property_OneSecondKeeperLatency_1bp` | settlement 1 s after the fill tick, −1 bp | **FAILS**: 50.791492 |
| `test_R3_Characterize_AtomicBundle_CompetingKeeperTooLate` | an unprivileged contract settles and executes in one transaction | passes: the order executes; a competing keeper's later `executeOrder` on it reverts |
| `test_R3_Characterize_SameSecondDifferentPrice` | synthetic: two prices in one publish second | passes (characterization only; real Pyth publishes one price per feed per second, so no claim uses it) |

Excerpt of `forge test -vv` for the release-parameter run (USDC in 6-decimal units; A = execute → settle, B = settle → execute):

```console
[FAIL: flat: ALICE depends on the order through execution charges: 50000164632802 !~= 49999979341304 (max delta: 2000000, real delta: 185291498)] test_R2_Property_Large_Flat_ExecutionChargesOrderingNeutral() (gas: 20076517)
Logs:
  L flat ALICE (redeemer) A / B 50000164632802 49999979341304
  L flat CAROL (depositor) A / B 49999999999999 50000185291563
  L flat BOB (holds) A / B 100000344486430 100000344486409
  L flat |dALICE| / |dCAROL| (1e-6 USDC) 185291498 185291564
  L flat SETTLER token A / B 0 0
  L flat SETTLER eth A / B 10000000000000000000 10000000000000000000

[PASS] test_R2_Characterize_ExecutionChargesRaiseJuniorNav() (gas: 8284578)
Logs:
  junior totalAssets before / after execute 699999678434372 700002272515216
  pool totalAssets before / after execute 710000000000000 710002594080844
  ALICE redeem shares / junior supply 50000004756469152 700000034246654080

[FAIL: same second, same price: ALICE still depends on the order: 50000164634614 !~= 49999979343116 (max delta: 2000000, real delta: 185291498)] test_R3_Property_SameSecondZeroMove_ChargesStillMove() (gas: 20074192)

[FAIL: ALICE's redemption depends on the settle/execute order: 50000030132796 !~= 49999979341304 (max delta: 2000000, real delta: 50791492)] test_R2_Property_Large_OrderingNeutral_1bp() (gas: 19994612)

[FAIL: ALICE's redemption depends on the settle/execute order: 49999481661342 !~= 49999979341304 (max delta: 2000000, real delta: 497679962)] test_R2_Property_Large_OrderingNeutral_5bp() (gas: 20274197)

[PASS] test_R1_S1_Characterize_Down1pct_DilutedToCents() (gas: 19244494)
Logs:
  S1 -1% |dALICE| / |dCAROL| (1e-6 USDC) 47944 47943

[PASS] test_R3_Characterize_AtomicBundle_CompetingKeeperTooLate() (gas: 8773523)
Logs:
  competing keeper after bundle: reverted (order no longer pending)

Suite result: FAILED. 7 passed; 5 failed; 0 skipped
```

### Test file: `packages/perps/test/perps/spec/lp/Card8240S1SettleBeforePendingFill.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {AtomicLpEpochSettlementTestFixture} from "../../shared/AtomicLpEpochSettlementFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title LP epoch settlement ignores a pending order whose fill price is already fixed
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to AtomicLpEpochSettlement.t.sol in
///         packages/perps/test/perps/spec/lp/ and run with the repository's own foundry.toml.
/// @dev SECURITY.md:849-860 ("Exact symmetric LP NAV"): LP entry and exit pricing consume the same snapshot, and new
///      LPs "do not inherit old trader obligations at a higher deposit NAV". A delayed order's fill is fixed at its
///      unique historical tick right after commit, but the order can wait up to its execution deadline (60 s by
///      default). `OrderRouter.settleLpEpoch` refreshes the mark and settles the matured epoch
///      (router/OrderLiquidationBatchLogic.sol:549-581). Settlement NAV takes its price exposure from the Engine's
///      open-position book only (TerminalNavBookV2); pool cash, trader claims and caps also enter NAV, but a pending
///      order's PnL from its fixed fill to the settlement mark does not, and settlement does not read the order queue.
///      When the order executes afterwards, the Engine keeps the newer settlement mark
///      (OrderRouterExecutionSidecar.sol:1053-1060), so the position opens at the old fill with its marked PnL against
///      that mark already in place, and that PnL falls on whoever holds shares after settlement. Both calls are
///      permissionless, so whoever lands first decides whether the epoch's redeemers or its depositors bear it. Both
///      orderings end with the same position (size, margin, entry) and the same Engine mark.
///      Public entry points only after the fixture's setUp: `TrancheVault` request/claim, `HousePool.settleLpEpoch`
///      while no position is open (permissionless then, SECURITY.md:319), the V3 `OrderRouter.commitOrder(OrderRequest)`
///      from the trader's own account, `router.updateMarkPrice`, `router.settleLpEpoch` and `router.executeOrder` from
///      accounts with no role. Pyth is the maintainers' MockPyth and USDC is the fixture's mock token (both outside
///      the scope). No protocol contract is pranked and no storage is written.
///      Fixture notes: the maintainers' BasePerpTest deploys the Router as `LegacyOrderRouterHarness`
///      (test/utils/LegacyOrderRouterHarness.sol), a subclass of `OrderRouter` that only adds a scalar `commitOrder`
///      adapter and an `IS_TEST` constant; it overrides no settlement or execution path, and this test does not use the
///      adapter (it calls the production V3 commit). BasePerpTest also seeds 1,000 USDC junior and 1,000 USDC senior
///      through `initializeSeedPosition` (held by the test contract), so junior equity is about 201,000 USDC.
///      The LP-epoch fixture sets VPI to zero.
///      The root reaches further than one open: the last section shows a pending CLOSE (an existing trader's exit in
///      the seconds before the boundary) and a settler that picks how many of several traders' pending orders execute
///      before it settles (a FIFO prefix), which moves more than either all-before or all-after.
contract Card8240S1SettleBeforePendingFillTest is AtomicLpEpochSettlementTestFixture {
    address internal constant KEEPER = address(0x6EE9E2); // any EOA, no role
    uint256 internal constant P_FILL = 100_000_000; // 1.00, the order's fill tick
    uint256 internal constant SIZE = 50_000e18;
    uint256 internal constant MARGIN = 5_000e6;
    uint256 internal constant TOL = 2e6; // 2 USDC of rounding headroom

    uint256 private _clientNonce;

    struct World {
        uint256 reqId;
        uint256 boundary;
        uint64 orderId;
        uint64 commitTime;
        uint256 aliceRedeemShares;
        uint256 carolDepositAssets;
    }

    struct Out {
        uint256 pendingAtSettle; // TRADER's pending orders when settleLpEpoch runs
        uint8 orderStatus; // lifecycle status returned by executeOrder
        uint256 aliceAssets; // USDC paid to the redeemer
        uint256 carolShares; // shares minted to the depositor
        uint256 carolValue; // value of those shares in the final state
        uint256 bobValue; // value of BOB's untouched shares in the final state
        uint256 traderSize;
        uint256 traderMargin;
        uint256 traderEntry;
        uint256 markPrice;
        uint64 markTime;
    }

    // -------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // -------------------------------------------------------------------------------------------------------------

    /// One basket tick at `price`, published at `publishTime`, with `previous` as the previous publish time.
    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previous
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(u);
    }

    /// A production V3 open request (slippage bound never binding).
    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private returns (uint64 orderId) {
        orderId = _commitV3Order(trader, side, size, margin, false);
    }

    /// A production V3 open or close request (slippage bound never binding: 1 for a LONG open or SHORT close,
    /// type(uint256).max for a SHORT open or LONG close).
    function _commitV3Order(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card-8240-s1", trader, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    /// ALICE and BOB hold 100k junior each; in epoch e ALICE requests to redeem half and CAROL requests a 50k deposit.
    /// TRADER commits a 50k LONG at b - 10 (b = epoch e's start); its fill tick is b - 9. Returns at b + 1.
    function _world() internal returns (World memory w) {
        vm.deal(KEEPER, 100 ether);
        vm.deal(ALICE, 100 ether);
        uint256 aliceShares = _seedJuniorLp(ALICE, 100_000e6);
        _seedJuniorLp(BOB, 100_000e6);
        uint256 cooled = juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 cooledA = juniorVault.lastDepositTime(ALICE) + juniorVault.DEPOSIT_COOLDOWN();
        vm.warp((cooled > cooledA ? cooled : cooledA) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasket(P_FILL, 0, block.timestamp);
        router.updateMarkPrice(_emptyUpdateData());

        _fundTrader(TRADER, 20_000e6);

        w.aliceRedeemShares = aliceShares / 2;
        w.reqId = _requestJuniorRedeem(ALICE, w.aliceRedeemShares);
        w.carolDepositAssets = 50_000e6;
        uint256 carolReq = _requestJuniorDeposit(CAROL, w.carolDepositAssets);
        assertEq(carolReq, w.reqId, "ghost: deposit and redemption share epoch e");
        w.boundary = pool.lpEpochStart(w.reqId);
        assertLt(block.timestamp + juniorVault.LP_REQUEST_CUTOFF_DURATION(), w.boundary, "ghost: requests before cutoff");

        // TRADER commits at b - 10 (trader orders have no LP cutoff).
        vm.warp(w.boundary - 10);
        vm.roll(vm.getBlockNumber() + 1);
        w.orderId = _commitV3(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(w.orderId);
        w.commitTime = pending.commitTime;
        assertEq(uint256(w.commitTime), w.boundary - 10, "ghost: committed at b - 10");

        // b + 1: the fill tick (commitTime + 1 = b - 9) is history and the order is still inside its execution window.
        vm.warp(w.boundary + 1);
        vm.roll(vm.getBlockNumber() + 1);
        assertLt(block.timestamp, uint256(w.commitTime) + router.maxExecutionWindowSeconds(), "ghost: order executable");
    }

    /// KEEPER (no role) executes the order with its unique historical fill tick at P_FILL.
    function _execute(
        World memory w
    ) internal returns (uint8 status) {
        bytes[] memory hist = _ticks(P_FILL, w.commitTime + 1, w.commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1 ether}(w.orderId, hist);
        status = uint8(res.status);
    }

    /// `settler` (no role) settles the matured epoch through the Router at the current tick `m`.
    function _settle(
        address settler,
        uint256 m
    ) internal {
        vm.prank(settler);
        router.settleLpEpoch{value: 1 ether}(_ticks(m, block.timestamp, 0));
    }

    function _readOut(
        World memory w,
        uint8 status,
        uint256 pendingAtSettle
    ) internal returns (Out memory o) {
        o.pendingAtSettle = pendingAtSettle;
        o.orderStatus = status;
        uint256 claimShares = juniorVault.claimableRedeemRequest(w.reqId, ALICE);
        assertEq(claimShares, w.aliceRedeemShares, "ghost: ALICE fully funded");
        vm.prank(ALICE);
        o.aliceAssets = juniorVault.claimRedeem(w.reqId, claimShares, ALICE, ALICE);
        o.carolShares = _claimJuniorDeposit(w.reqId, CAROL);
        o.carolValue = juniorVault.convertToAssets(o.carolShares);
        o.bobValue = juniorVault.convertToAssets(juniorVault.balanceOf(BOB));
        (o.traderSize, o.traderMargin, o.traderEntry,,,,) = engine.positions(TRADER);
        o.markPrice = engine.lastMarkPrice();
        o.markTime = engine.lastMarkTime();
    }

    /// Arm A: execute -> settle. Arm B: settle -> execute. Same state, same ticks, same callers otherwise.
    function _arms(
        uint256 m,
        address settler
    ) internal returns (Out memory a, Out memory b) {
        uint256 s = vm.snapshotState();
        World memory w = _world();
        uint8 st = _execute(w);
        uint256 pend = router.pendingOrderCounts(TRADER);
        _settle(settler, m);
        a = _readOut(w, st, pend);
        vm.revertToState(s);

        w = _world();
        pend = router.pendingOrderCounts(TRADER);
        _settle(settler, m);
        assertEq(engine.lastMarkTime(), w.boundary + 1, "ghost: settlement mark is newer than the fill tick");
        st = _execute(w);
        b = _readOut(w, st, pend);
        vm.revertToState(s);

        assertEq(a.pendingAtSettle, 0, "ghost: A settles with no pending order");
        assertEq(b.pendingAtSettle, 1, "ghost: B settles with the order pending");
        assertEq(a.orderStatus, uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: A executed");
        assertEq(b.orderStatus, uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: B executed");
        assertEq(a.traderSize, SIZE, "ghost: A position open");
        assertEq(a.traderSize, b.traderSize, "ghost: same final position");
        assertEq(a.traderEntry, P_FILL, "ghost: A opened at the fixed fill");
        assertEq(a.traderEntry, b.traderEntry, "ghost: same entry");
        assertEq(a.traderMargin, b.traderMargin, "ghost: same margin");
        assertEq(a.markPrice, b.markPrice, "ghost: same final mark");
        assertEq(a.markTime, b.markTime, "ghost: same final mark time");
    }

    function _gap(
        uint256 x,
        uint256 y
    ) internal pure returns (uint256) {
        return x > y ? x - y : 0;
    }

    function _log(
        string memory tag,
        Out memory o
    ) internal pure {
        console2.log(tag);
        console2.log("  pending at settle / order status", o.pendingAtSettle, o.orderStatus);
        console2.log("  engine mark / time    ", o.markPrice, o.markTime);
        console2.log("  ALICE redeem assets   ", o.aliceAssets);
        console2.log("  CAROL value (final)   ", o.carolValue);
        console2.log("  BOB value (final)     ", o.bobValue);
    }

    // -------------------------------------------------------------------------------------------------------------
    // Tests
    // -------------------------------------------------------------------------------------------------------------

    /// Control: the market has not moved since the fill (settlement tick == fill price). The order of the two
    /// permissionless calls does not matter.
    function test_Control_FlatMark_OrderIrrelevant() public {
        (Out memory a, Out memory b) = _arms(P_FILL, KEEPER);
        _log("flat, A execute->settle", a);
        _log("flat, B settle->execute", b);
        assertApproxEqAbs(a.carolValue, b.carolValue, TOL, "flat: CAROL value");
        assertApproxEqAbs(a.aliceAssets, b.aliceAssets, TOL, "flat: ALICE payout");
    }

    /// PROPERTY (SECURITY.md:854-855, the doc's own direction): the market moves 1% in the trader's favour after the
    /// fill, so LPs owe the pending order's PnL. The new depositor's final value must not depend on whether a third
    /// party settles the epoch before or after executing the order. FAILS on the pinned code.
    function test_Property_DepositorDoesNotInheritFixedFillObligation() public {
        (Out memory a, Out memory b) = _arms(P_FILL * 99 / 100, KEEPER);
        _log("-1%, A execute->settle", a);
        _log("-1%, B settle->execute", b);
        console2.log("CAROL loss from call order (USDC 6dp)", _gap(a.carolValue, b.carolValue));
        console2.log("ALICE gain from call order (USDC 6dp)", _gap(b.aliceAssets, a.aliceAssets));
        assertApproxEqAbs(b.carolValue, a.carolValue, TOL, "depositor inherits the fixed-fill obligation");
    }

    /// PROPERTY (the mirror image): the market moves 1% against the trader after the fill, so the pending order owes
    /// LPs. The redeemer's payout must not depend on the order of the two calls. FAILS on the pinned code.
    function test_Property_RedeemerPayoutIndependentOfCallOrder() public {
        (Out memory a, Out memory b) = _arms(P_FILL * 101 / 100, KEEPER);
        _log("+1%, A execute->settle", a);
        _log("+1%, B settle->execute", b);
        console2.log("ALICE loss from call order (USDC 6dp)", _gap(a.aliceAssets, b.aliceAssets));
        console2.log("CAROL gain from call order (USDC 6dp)", _gap(b.carolValue, a.carolValue));
        assertApproxEqAbs(b.aliceAssets, a.aliceAssets, TOL, "redeemer payout depends on call order");
    }

    /// The redeemer herself can take the step: ALICE calls settleLpEpoch first and receives the same transfer.
    /// What she gains is what CAROL loses; BOB, who holds through the epoch, is unaffected.
    function test_Characterize_RedeemerSettlesFirstHerself() public {
        (Out memory a, Out memory b) = _arms(P_FILL * 99 / 100, ALICE);
        uint256 gain = b.aliceAssets - a.aliceAssets;
        uint256 loss = a.carolValue - b.carolValue;
        console2.log("settler = ALICE, -1%: ALICE gain / CAROL loss (USDC 6dp)", gain, loss);
        console2.log("BOB value A / B", a.bobValue, b.bobValue);
        assertGt(gain, 100e6, "ALICE gains over 100 USDC by settling first");
        assertApproxEqAbs(gain, loss, 1e3, "ALICE's gain is CAROL's loss");
        assertApproxEqAbs(a.bobValue, b.bobValue, 1e3, "the holder-through is unaffected");
    }

    /// Scaling: the transfer is linear in the move between the fill and the settlement tick (trader-wins direction).
    function test_Characterize_Scaling() public {
        uint256[4] memory bps = [uint256(1), 5, 20, 100];
        for (uint256 i; i < bps.length; ++i) {
            (Out memory a, Out memory b) = _arms(P_FILL * (10_000 - bps[i]) / 10_000, KEEPER);
            uint256 loss = a.carolValue - b.carolValue;
            console2.log("move (bps) / CAROL loss (USDC 6dp)", bps[i], loss);
            assertGt(loss, 0, "CAROL loses at every move");
            assertApproxEqAbs(loss, b.aliceAssets - a.aliceAssets, 1e3, "ALICE's gain is CAROL's loss");
            assertApproxEqAbs(a.bobValue, b.bobValue, 1e3, "the holder-through is unaffected");
        }
    }

    // -------------------------------------------------------------------------------------------------------------
    // The same root, two wider cases: (1) a pending CLOSE of an open position, (2) a settler that picks how many of
    // several traders' pending orders execute before settlement (a FIFO prefix), not just "all before" or "all after"
    // -------------------------------------------------------------------------------------------------------------

    address internal constant T1 = address(0x7101);
    address internal constant T2 = address(0x7102);
    address internal constant T3 = address(0x7103);

    struct Pending {
        uint64 orderId;
        uint64 commitTime;
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// The LP side of `_world`, part 1: ALICE and BOB hold 100k junior each, cooled down, mark at P_FILL.
    function _lpSeed() internal {
        vm.deal(KEEPER, 100 ether);
        _seedJuniorLp(ALICE, 100_000e6);
        _seedJuniorLp(BOB, 100_000e6);
        uint256 cooled = juniorVault.lastDepositTime(BOB) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 cooledA = juniorVault.lastDepositTime(ALICE) + juniorVault.DEPOSIT_COOLDOWN();
        _warpTo((cooled > cooledA ? cooled : cooledA) + 1);
        _setBasket(P_FILL, 0, block.timestamp);
        router.updateMarkPrice(_emptyUpdateData());
    }

    /// The LP side of `_world`, part 2: in epoch e ALICE requests to redeem half and CAROL requests a 50k deposit.
    function _lpRequests() internal returns (World memory w) {
        w.aliceRedeemShares = juniorVault.balanceOf(ALICE) / 2;
        w.reqId = _requestJuniorRedeem(ALICE, w.aliceRedeemShares);
        w.carolDepositAssets = 50_000e6;
        uint256 carolReq = _requestJuniorDeposit(CAROL, w.carolDepositAssets);
        assertEq(carolReq, w.reqId, "ghost: deposit and redemption share epoch e");
        w.boundary = pool.lpEpochStart(w.reqId);
        assertLt(block.timestamp + juniorVault.LP_REQUEST_CUTOFF_DURATION(), w.boundary, "ghost: requests before cutoff");
    }

    function _commitNow(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (Pending memory p) {
        p.orderId = _commitV3Order(trader, side, size, margin, isClose);
        (IOrderRouterAccounting.PendingOrderView memory v,) = router.getPendingOrderView(p.orderId);
        p.commitTime = v.commitTime;
        assertEq(uint256(p.commitTime), block.timestamp, "ghost: committed now");
    }

    /// KEEPER (no role) executes `p` with its unique historical fill tick (commitTime + 1) at P_FILL.
    function _executePending(
        Pending memory p
    ) internal returns (uint8 status) {
        bytes[] memory hist = _ticks(P_FILL, p.commitTime + 1, p.commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1 ether}(p.orderId, hist);
        status = uint8(res.status);
    }

    function _claimBoth(
        World memory w
    ) internal returns (uint256 alice, uint256 carol) {
        uint256 claimShares = juniorVault.claimableRedeemRequest(w.reqId, ALICE);
        assertEq(claimShares, w.aliceRedeemShares, "ghost: ALICE fully funded");
        vm.prank(ALICE);
        alice = juniorVault.claimRedeem(w.reqId, claimShares, ALICE, ALICE);
        carol = juniorVault.convertToAssets(_claimJuniorDeposit(w.reqId, CAROL));
    }

    function _sizeOf(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ---- (1) pending CLOSE ----

    /// TRADER opens a 50k LONG at P_FILL well before epoch e and it executes. The LP requests land. At b - 10 TRADER
    /// commits a full close; its fill tick is b - 9 at P_FILL. Returns at b + 1, with the close still executable.
    function _closeWorld() internal returns (World memory w, Pending memory close) {
        _lpSeed();
        _fundTrader(TRADER, 20_000e6);
        Pending memory open_ = _commitNow(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, false);
        _warpTo(block.timestamp + 2);
        assertEq(_executePending(open_), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: setup open executed");
        assertEq(_sizeOf(TRADER), SIZE, "ghost: position open before the epoch");
        w = _lpRequests();
        _warpTo(w.boundary - 10);
        close = _commitNow(TRADER, CfdTypes.Side.LONG, SIZE, 0, true);
        _warpTo(w.boundary + 1);
        assertLt(block.timestamp, uint256(close.commitTime) + router.maxExecutionWindowSeconds(), "ghost: close executable");
    }

    /// Arm A: execute the close -> settle. Arm B: settle -> execute the close. Both end flat.
    function _closeArms(
        uint256 m
    ) internal returns (uint256 aliceA, uint256 carolA, uint256 aliceB, uint256 carolB) {
        uint256 s = vm.snapshotState();
        (World memory w, Pending memory c) = _closeWorld();
        assertEq(_executePending(c), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: A close executed");
        assertEq(router.pendingOrderCounts(TRADER), 0, "ghost: A settles with no pending close");
        _settle(KEEPER, m);
        assertEq(_sizeOf(TRADER), 0, "ghost: A flat");
        (aliceA, carolA) = _claimBoth(w);
        vm.revertToState(s);

        (w, c) = _closeWorld();
        assertEq(router.pendingOrderCounts(TRADER), 1, "ghost: B settles with the close pending");
        _settle(KEEPER, m);
        assertEq(_executePending(c), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: B close executed");
        assertEq(_sizeOf(TRADER), 0, "ghost: B flat");
        (aliceB, carolB) = _claimBoth(w);
        vm.revertToState(s);
    }

    /// Control: flat market. The order of the close and the settlement does not matter.
    function test_Control_PendingCloseFlatMark_OrderIrrelevant() public {
        (uint256 aA, uint256 cA, uint256 aB, uint256 cB) = _closeArms(P_FILL);
        console2.log("pending close, flat: ALICE A / B", aA, aB);
        console2.log("pending close, flat: CAROL A / B", cA, cB);
        assertApproxEqAbs(aA, aB, TOL, "flat close: ALICE payout");
        assertApproxEqAbs(cA, cB, TOL, "flat close: CAROL value");
    }

    /// PROPERTY: a close whose fill was fixed before the boundary, market 1% lower at settlement. The redeemer's
    /// payout must not depend on the order of the close and the settlement. FAILS on the pinned code.
    function test_Property_PendingCloseDown1pct_RedeemerPayoutIndependentOfCallOrder() public {
        (uint256 aA, uint256 cA, uint256 aB, uint256 cB) = _closeArms(P_FILL * 99 / 100);
        console2.log("pending close, -1%: ALICE A(execute->settle) / B(settle->execute)", aA, aB);
        console2.log("pending close, -1%: CAROL A / B", cA, cB);
        assertApproxEqAbs(aB, aA, TOL, "pending close: redeemer payout depends on call order");
    }

    /// PROPERTY (the mirror image): market 1% higher at settlement. FAILS on the pinned code.
    function test_Property_PendingCloseUp1pct_RedeemerPayoutIndependentOfCallOrder() public {
        (uint256 aA, uint256 cA, uint256 aB, uint256 cB) = _closeArms(P_FILL * 101 / 100);
        console2.log("pending close, +1%: ALICE A(execute->settle) / B(settle->execute)", aA, aB);
        console2.log("pending close, +1%: CAROL A / B", cA, cB);
        assertApproxEqAbs(aB, aA, TOL, "pending close: redeemer payout depends on call order");
    }

    // ---- (2) FIFO prefix over three unrelated traders' pending orders ----

    /// Three unrelated traders commit opens at b - 13, b - 12 and b - 11 (fills fixed one second later, all at
    /// P_FILL): T1 LONG 50k, T2 SHORT 80k, T3 LONG 40k. The FIFO executes them in that order. Returns at b + 1.
    function _fifoWorld() internal returns (World memory w, Pending[3] memory p) {
        _lpSeed();
        _fundTrader(T1, 20_000e6);
        _fundTrader(T2, 20_000e6);
        _fundTrader(T3, 20_000e6);
        w = _lpRequests();
        _warpTo(w.boundary - 13);
        p[0] = _commitNow(T1, CfdTypes.Side.LONG, 50_000e18, 5_000e6, false);
        _warpTo(w.boundary - 12);
        p[1] = _commitNow(T2, CfdTypes.Side.SHORT, 80_000e18, 8_000e6, false);
        _warpTo(w.boundary - 11);
        p[2] = _commitNow(T3, CfdTypes.Side.LONG, 40_000e18, 4_000e6, false);
        _warpTo(w.boundary + 1);
    }

    /// The settler lets the first k pending orders execute, settles the epoch, then the rest execute.
    /// Every final world holds the same three positions.
    function _prefix(
        uint256 m,
        uint256 k
    ) internal returns (uint256 alice, uint256 carol) {
        uint256 s = vm.snapshotState();
        (World memory w, Pending[3] memory p) = _fifoWorld();
        for (uint256 i; i < k; ++i) {
            assertEq(_executePending(p[i]), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: executed before");
        }
        _settle(KEEPER, m);
        for (uint256 i = k; i < 3; ++i) {
            assertEq(_executePending(p[i]), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: executed after");
        }
        assertEq(_sizeOf(T1), 50_000e18, "ghost: T1 open");
        assertEq(_sizeOf(T2), 80_000e18, "ghost: T2 open");
        assertEq(_sizeOf(T3), 40_000e18, "ghost: T3 open");
        (alice, carol) = _claimBoth(w);
        vm.revertToState(s);
    }

    function _prefixSweep(
        uint256 m,
        string memory tag
    ) internal returns (uint256[4] memory alice, uint256 minA, uint256 maxA) {
        minA = type(uint256).max;
        console2.log(tag);
        for (uint256 k; k <= 3; ++k) {
            uint256 carol;
            (alice[k], carol) = _prefix(m, k);
            console2.log("  k executed before settlement / ALICE payout / CAROL value", k, alice[k], carol);
            if (alice[k] < minA) minA = alice[k];
            if (alice[k] > maxA) maxA = alice[k];
        }
    }

    /// Control: flat market. How many orders execute before settlement does not matter.
    function test_Control_FifoPrefixFlatMark_PrefixIrrelevant() public {
        (, uint256 minA, uint256 maxA) = _prefixSweep(P_FILL, "FIFO prefix, flat");
        assertApproxEqAbs(minA, maxA, TOL, "flat: prefix choice irrelevant");
    }

    /// The settler's choice is a selection over every prefix, not a coin flip: the range of ALICE's payout over all
    /// prefixes exceeds the range over the two binary arms (k = 0, k = 3).
    function test_Characterize_FifoPrefixRangeExceedsBinaryArms() public {
        (uint256[4] memory a, uint256 minA, uint256 maxA) = _prefixSweep(P_FILL * 99 / 100, "FIFO prefix, -1%");
        uint256 binary = a[0] > a[3] ? a[0] - a[3] : a[3] - a[0];
        console2.log("range over all prefixes / over the binary arms (USDC 6dp)", maxA - minA, binary);
        assertGt(maxA - minA, 100e6, "prefix range above 100 USDC");
        assertGt(maxA - minA, 4 * binary, "prefix range exceeds 4x the binary arms");
    }

    /// PROPERTY: the redeemer's payout must not depend on how many already-fixed pending orders a third party lets
    /// execute before it settles. FAILS on the pinned code.
    function test_Property_FifoPrefix_RedeemerPayoutIndependentOfPrefix() public {
        (, uint256 minA, uint256 maxA) = _prefixSweep(P_FILL * 99 / 100, "FIFO prefix, -1%");
        assertApproxEqAbs(maxA, minA, TOL, "redeemer payout depends on the settler's FIFO prefix");
    }
}
```

