# \#99871 [SC-Low] Any caller can put the Engine mark ahead of the oracle's accepted price every second with self-executed failing orders; creating or replacing a position protection then reverts for every account until a live refresh at or after that mark is accepted

**Submitted on Nov 15th 2026 at 15:50:43 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99871
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Executing an order at its historical Pyth tick writes that tick's time into the Engine mark but not into the oracle's accepted basket. While the Engine mark is the later of the two, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert `PletherOracle__PriceOutOfOrder` for every account, because the freshness check they share reads one market-wide observation. The refusal came with the fix for the listed audit finding MED-28: with the Engine mark ahead, create and replace succeed at the parent commit (`d757d0b1`) and revert at the commit that completed the fix (`eadc5c58`). The mark is written before the order is run, so an order that fails at execution is enough, and a caller that executes its own failing orders receives its own execution bounty back. In the PoC one such order per second puts the Engine mark ahead again in 3,594 of 3,600 seconds at a net cost of 0.00 USDC; a third party's live refresh clears it each time. Where nobody else refreshes, one loop refuses all N accounts at once (N = 100 owners, each one second behind: 4,300 refused attempts and 0 amends landed, against 100 of 100 in the control). The limits: nothing is frozen. Liquidation, LP settlement and protection triggers are not blocked for a keeper that sends a current price. With the mark ahead an owner can cancel a protection, cancel-and-close, add margin and withdraw. An owner whose refresh and protection call carry the same second's price is not refused on a uniform price tape. The caller gains nothing from the refusals. What is refused, while the loop runs, is setting or moving a stop-loss or take-profit for any owner who is a second or more behind.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

Site 1, the mark write:

```solidity
// OrderRouterExecutionSidecar.sol:1053-1061
function _updateEngineMark(
    IOrderRouterExecutionHost host,
    OracleResult memory result
) private {
    ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
    if (result.publishTime > engine_.lastMarkTime()) {
        engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
    }
}
```

It is called as soon as the oracle resolves an order's price (`:979` single, `:1036` batch). In a live market that price is the first Pyth tick after the order's commit time. Both order-price paths build the snapshot with the ordering check off (`PletherOracle.sol:283`, `:557`) and do not write the oracle's accepted basket; only the live path does (`PletherOracle.sol:798`). So after such an execution the Engine mark time is later than the accepted basket's.

Site 2, the check shared by the three entrypoints (`createPositionProtection` at `PositionProtectionBook.sol:229`, `replacePositionProtection` at `:295`, `commitOpenOrderWithProtection` at `:463`):

```solidity
// PositionProtectionBook.sol:862-886, _freshProtectionObservation
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // :880
// :894, _acceptedMarkInstalled, the guard called next (:883)
return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

`getLatestPrice` reads the accepted basket with the ordering check on (`PletherOracle.sol:571-580`) and reverts when `snapshot.publishTime < markTime` (`:892-896`).

Regression of a listed fix. The oracle read at `PositionProtectionBook.sol:880` is the validation described in the response to a fixed audit finding (Cecuro MED-28), which says the guard rejects arming when "the accepted oracle mark is ahead of, or inconsistent with, the Engine mark" and that "A qualifying mark must be installed before arming can proceed". Here the Engine mark is ahead: that guard accepts it (`:894`) and the read before it reverts. MED-28 reported the other direction, arming against a cached Engine mark while the oracle already held a newer price, and asked for a read of the oracle's current price. `eadc5c58` (PR #124, 2026-10-23) added it (`:880-882`). The tie to MED-28 is by content: the commit message names only MED-8 and MED-9, and the response's "Protection validation" link is to upstream commit `8f6cfd3d`, absent from this repository's history. At the parent, `d757d0b1`, the function read only the Engine mark and ended in the same guard (from `81919218`, the response's other link): with the mark ahead, create and replace succeed there and revert at `eadc5c58` (same test at both commits, control passing at both; see Proof of Concept). The ordering requirement is that fix's purpose and we do not dispute it. The new bug is that any caller can put every account outside it again every second at no cost. The programme's condition for a report on a fixed finding is that "the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug". `EqualTimeMarkMismatch` is the equal-time guard from fixed finding MED-18. Neither response states that any caller can undo the qualifying mark every second at no cost.

Why a failing order is enough. `executeOrder` resolves the price and writes the mark first (`OrderRouterExecutionSidecar.sol:212`), and only then runs the order inside a `try` (`:223-229`). A typed failure is settled as `Failed` without reverting (`_settleTypedFailure`, `:630-655`), so the mark write stays. The PoC's failing order is an open whose `minPostPositionEquityAssets` bound cannot be met.

Why it costs nothing. The order's execution bounty is paid to the executor. A caller that executes its own orders pays the bounty from one of its addresses to another.

What clears the state: a live `updateMarkPrice` whose basket is at or after the Engine mark's time. A refresh with a payload one second older reverts with the same error (asserted). Where feeds publish at different cadences, a same-second refresh can instead revert `PletherOracle__EqualTimeMarkMismatch` (`PletherOracle.sol:897-900`): same time as the Engine mark, different price.

## Impact Details
The state, market-wide and harm cases run a control and a loop branch from one snapshot, with the same prices and times; the harm control owner also uses a same-second price. The other cases have no control branch. Amounts are USDC.

State. Three unrelated accounts, and one order of the filler executed at its historical tick (an 11-lot fill, or an order that ends `Failed`; both tested). In the control all three calls succeed. After the order `replacePositionProtection`, `createPositionProtection` and `commitOpenOrderWithProtection` each revert `PriceOutOfOrder(1709798420,1709798421)`, `0x8e3e110d`.

Loop. Failing orders, one per second, from 14 filler accounts in rotation (5 pending orders allowed per account), executed by the filler; an honest keeper refreshes live every second. Over 3,600 s: `Failed` receipts 3,600; seconds with the Engine mark ahead after the filler's step 3,594 (the first 6 s are warm-up); bounty leaving the filler's accounts and reaching its executor 396.00; filler net 0.00; Pyth fees in ETH: executor 0.0000432, keeper 0.0000216. Fees, VPI, carry, forfeits, treasury and pool deltas are all 0. If a third party executes the orders instead, the filler's net over 120 s is −13.20 (the bounty). L2 gas is not modelled.

Who is refused. Refused attempts out of 3,600 (one refresh-then-replace attempt per second for an hour); filler of real 100-lot fills, placed first in each second, its commits timed to the tape:

| Owner, 1 fill/s | Uniform tape | Mixed-cadence tape |
|---|---|---|
| Same-second price, replace in the next block | 0 | 2,100 |
| Price one second old | 3,600 | 3,300 |
| Same-second price, refresh and replace back to back in one block | 0 | 2,100 |

On the mixed-cadence tape (feeds every 1/2/1/3/4/3 s) an owner holding the current second's price is refused in 2,100 of 3,600 seconds, in one block or two: `EqualTimeMarkMismatch`, the refresh colliding with the mark the historical fill wrote. In the market-wide, owner and Friday runs every refusal is at the owner's refresh, so the replace is not reached; the state test shows the three calls themselves reverting.

Market-wide. 100 accounts, each a long of 10,000 notional with a stop 3.0 % away, each trying every second to move it to 2.5 % with a price one second old, while the index moves toward the stop (scripted: 2.0 % at T+20, 2.8 % at T+40, 5.0 % at T+70). Failing-order loop, filler net 0.00. Control: 100 amends landed, 0 refused. Loop: 0 landed, 4,300 refused. Sum of the owners' results: −27,660.33 against −50,325.34.

Friday close-only window (10 minutes run). The filler sends failing closes of positions it opened earlier, 11 lots at a time: 600 `Failed`, the mark ahead in 600 s, the owner (one attempt per 10 s, price one second old) refused 60 of 60; filler delta −0.43.

Harm, and what it is conditional on. One owner of 1,000,000 notional, one second behind, that keeps retrying while the index moves toward its old stop; uniform tape, same path; the filler here makes real 100-lot short fills. Control (amend lands at the first attempt, T+28; new stop fires at T+44): −26,921.24. Under the loop: the retrier (43 refusals, old stop fires at T+70) −49,766.06; a same-second owner with one atomic call −27,233.23; an owner that cancels-and-closes at T+28 −18,565.58. On a reversal the refusal helps the owner, +11,347.13 against −26,921.24, because the old stop never fires; on a drift that crosses the new stop but not the old one and stays there, there is no difference (−24,817.31 against −24,846.74).

Limits of the result:

- No funds are frozen. With the loop running at 11,200 lots, `withdrawMargin` of all free margin succeeds, and cancel-and-close then withdraw-all leaves the owner flat 7 s after the commit with 1,096,759.57 received, the ledger balance. A plain close with a protection armed reverts (`0x3a0535a3`); the owner cancels first.
- Liquidation, LP settlement and protection triggers are not blocked for a keeper that sends a current price: each refreshes the oracle with the keeper's own payload in the same transaction (`router/OrderLiquidationBatchLogic.sol:718-719`, `:755-756`, `:561-562`, `:607-608`). A keeper whose payload is a second behind is refused there, as it is after any fill at the newest tick, loop or no loop; on mixed cadences a same-second payload can collide there too (`PletherOracle.sol:897-900`). We claim nothing for those paths; they are not in the shipped tests.
- Not refused on the uniform tape: the same-second owner and the atomic refresh-and-amend (0 of 3,600). The atomic call is made by a smart account; EIP-7702 was not run.
- The loop row counts seconds in which the mark was ahead right after the filler's step, not how long it stays ahead: the keeper's live refresh follows in the same block, was accepted in all 3,600 seconds and clears the state. Refused calls are counted in the market-wide, owner and Friday rows; the market-wide, owner and harm runs have no third-party refresher.
- The harm figures mix three things: the refusal, the owner's one-second lag, and the filler's own price impact. The same-second figure isolates the last: 311.98 worse than control. The harm filler is the real-fill one; its own accounts end the main path at +5,262.56 and +5,252.84 (`atkDelta`), the result of its short fills on the scripted rise. The failing-order filler (net 0.00) was run for the state, loop, market-wide and Friday rows, and its harm variant is in the package but not in the run list.
- The harm is a uniform-tape result, conditional on the owner's behaviour. On the mixed-cadence tape the same retrier lands after 6 refusals and ends at −27,316.99 against −26,929.16, the same as the same-second owner there.
- Pyth is a test double; every price path and both tapes are scripted and no likelihood is claimed. The filler acts first in every block. Real Pyth cadence and latency, sequencer ordering and a deeper pool were not run.
- Not release values: pool deposits 800,000 Junior and 2,000,000 Senior, tape cadence, four 0.25 s blocks per second, staleness limits, execution windows, bounties.
- Printed, not asserted: executor ETH over 3,600 s; `withdrawMargin` and the refusal selectors in the not-frozen test; the filler's balance over the Friday window. The filler's peak capital was not measured.

Closest exclusions.

- Known issues. `SECURITY.md:719`, `:740-741` ("mark-price ordering remain nonterminal") concern a queued order's execution; `:545` concerns triggering; `:687-694` and the programme's open finding "a caller can still pick an older valid signed price per feed" concede a choice among live prices, and here the tick is the one the execution rule prescribes; `:1343-1375` says queues do not "prevent repeated fresh invalid submissions". None names protection creation. Fixed findings MED-28 and MED-18: see "Regression of a listed fix".
- By design. `SECURITY.md:563-564` says that for creation "freshness, position/queue checks, and post-lock collateral admission still apply", and `ORACLE_SYNCHRONIZATION.md:150-151` that "historical execution may advance the Engine without advancing the accepted live basket". Neither says that this refuses protection creation, or that any caller can cause it every second at no cost.
- Trust. No privileged role acts: "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections".
- Griefing. The measured spend is 0.00 USDC plus Pyth fees and gas. The programme's ratio baseline "Applies to High or Critical findings where the attacker gains no profit"; we claim Medium.
- Our own report 99080 concerns retries of a Latched protection's close: a different function, state and fix.
- Severity. For Medium: the refusal came with a fix, in a state the guard beside it accepts (`:894`); it is market-wide and free; on mixed cadences a caller holding the current second's price is refused too. Against it: no funds at risk, owners can close, keepers with a current price are not blocked, the same-second pattern works on a uniform tape.

## Recommendation
Either of:

- In `_freshProtectionObservation`, accept the Engine's stored mark when it is at or ahead of the accepted basket and within the freshness bound, or refresh inside the call.
- Do not advance the Engine mark for an order that ends `Failed`. This addresses the failing-order variant only; a filled order sets the same state.

We have not built or tested a patch.

## References
`OrderRouterExecutionSidecar.sol:212`, `:223-229`, `:630-655`, `:979`, `:1036`, `:1053-1061`; `PositionProtectionBook.sol:229`, `:295`, `:463`, `:862-886`, `:888-895`; `PletherOracle.sol:283`, `:557`, `:571-580`, `:798`, `:892-900`; `router/OrderLiquidationBatchLogic.sol:561-562`, `:607-608`, `:718-719`, `:755-756`; `SECURITY.md:545`, `:563-564`, `:687-694`, `:719`, `:740-741`, `:1343-1375`; commits `d757d0b175b5c2154babee946362ff90bd585f22`, `eadc5c587e2f2f1ec26c51132472ab876d6edc15`.

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/690ec6e76365781b0f6e876ba62b91d7

## Proof of Concept

## Proof of Concept
Test files for the repository's Foundry suite at the in-scope commit; nothing in `src/` is modified. Files: `M1MarkAhead.t.sol`, `MxBase.sol`, `MxSim.t.sol`, `MyBase.sol`, `MyCost.t.sol`, `MyMarket.t.sol`, `MyHarm.t.sol`, `MyOwner.t.sol`, `MyFad.t.sol`, `MyFrozen.t.sol`, `MultiTimePyth.sol`, `PythMultiTimeBase.sol`. Expected: `34 tests passed, 0 failed` in 12 suites.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/poc/m1/pythmt
cp M1MarkAhead.t.sol Mx*.sol My*.sol packages/perps/test/poc/m1/
cp MultiTimePyth.sol PythMultiTimeBase.sol packages/perps/test/poc/m1/pythmt/
forge test --root packages/perps --match-path 'test/poc/m1/*.sol' \
  --match-test 'test_State|test_Cost_k3_120|test_Cost_k3_thirdParty_120|test_Hour_k3|test_Market_N10|test_Scale_1000|test_Scale_10000|test_Paths_10000|test_Owner_|test_FadClose_|test_Frozen_Max' \
  --no-match-contract MyHarmFreeFillerTest --gas-limit 9000000000000000000 --memory-limit 8589934592 -vv
```

Tested with forge 1.7.1.

What the tests use:

- Release configuration, asserted from the deployed contracts in `M1MarkAhead.t.sol` `setUp`: Senior limits, minimum open notional, confidence multiplier and ratio, Senior rate, pending-order cap, trigger bounty, execution fee, settlement buffer, frozen close spread, FAD (Friday close-only) runway, the six feed ids and inversions. The other files' base installs and asserts only the minimum open notional, the two Senior limits and the multiplier; the rest are the base fixture's defaults.
- Fixtures. Pyth is a test double with a six-feed basket and a scripted tape (each feed every 6 s, staggered 1 s; or cadences 1/2/1/3/4/3 s). One test-only hook, `PythMultiTimeBase._stepAdj`, applies the scripted index steps. The token is a mock.
- Pranks. Targets are externally owned addresses sending their own calls, with three exceptions seen in traces: the repository's base fixture pranks the router address and the test contract during setup; the smart account used for the atomic strategy is funded by a prank of its own address; the test contract is the owner for the timelocked proposals in `setUp`. The honest keeper's refreshes and triggers are sent by the keeper address, and the sender is asserted: the keeper's ETH falls by exactly the Pyth fee of each call it lands.
- Cheatcodes: `warp`, `roll`, `prank`, `startPrank`, `stopPrank`, `deal` (ETH only), `snapshotState`, `revertToState`, `pauseGasMetering`, `recordLogs`, `getRecordedLogs`, `getBlockNumber`, `getBlockTimestamp`, `toString`, and `getNonce`, `computeCreateAddress` in the base fixture. No `store`, `etch`, `mockCall` or token `deal`.

Output, trimmed (raw amounts in 1e6 units; "..." marks cuts, including line breaks; the note in brackets is ours):

```
[STATE] ATTACK (one order that FAILS at execution, status Failed): replace / create / commitOpenOrderWithProtection revert PriceOutOfOrder(1709798420,1709798421) 0x8e3e110d...
== COST failing10-1h kind=3 secs=3600 ... receipts: Failed: 3600 ... seconds with engine mark ahead of accepted basket after the fill: 3594 ... filler net outflow USDC(1e6): 0
LOOP-lag1 lots=10000 path=0 mode=1 prot=1 loop=1 | victimDelta=-49766061258 atkDelta=5252842080 attempts=43
OWN-C lag0 same-block ... | attempts=3600 ok=1500 refreshFail=2100(POO 0,Eq 2100) ...   [mixed-cadence suite]
=== FROZEN-CHECK lots=11200 ... br4 cancelPositionProtectionAndCommitClose: OK ... br4 seconds until victim flat: 7 ... br4 wallet received: 1096759573079
```

Before and after the fix: `PrefixState.at-d757d0b1.t.sol` and `PrefixState.at-eadc5c58.t.sol`, the state test reduced to replace and create, in one version per commit. Adaptations, tests only: struct fields named `*Usdc` at those commits, `closeLimits` only at the fix commit, `virtual` added to one fixture hook (`BasePerpTest-virtual.diff`).

```
git checkout -f d757d0b175b5c2154babee946362ff90bd585f22   # then eadc5c587e2f2f1ec26c51132472ab876d6edc15
git apply BasePerpTest-virtual.diff   # then the copy commands of the gist README (that commit's PrefixState file and the two fixture files)
FOUNDRY_TEST=test/poc/prefix FOUNDRY_SCRIPT=test/poc/prefix forge test --root packages/perps --match-path 'test/poc/prefix/*.sol' -vv

[d757d0b1] [PRE-FIX STATE ATTACK stranger's failed order executed] engine mark time=1709539221 accepted basket time=1709539214 | unrelated replacePositionProtection -> OK | unrelated createPositionProtection -> OK
[eadc5c58] [PRE-FIX STATE ATTACK ...] ... | unrelated replacePositionProtection -> PriceOutOfOrder | unrelated createPositionProtection -> PriceOutOfOrder
```

Asserted at both commits: the mark is ahead, the control succeeds, and the attack outcome shown (at `eadc5c58` on the full revert bytes). Commit labels are ours.
