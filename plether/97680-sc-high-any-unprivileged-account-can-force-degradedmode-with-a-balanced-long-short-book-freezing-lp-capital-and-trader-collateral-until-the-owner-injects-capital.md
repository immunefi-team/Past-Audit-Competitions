# \#97680 [SC-High] Any unprivileged account can force degradedMode with a balanced long/short book, freezing LP capital and trader collateral until the owner injects capital

**Submitted on Nov 2nd 2026 at 16:33:21 UTC by @Kio77583 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97680
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`CfdEngine` admits new positions on two independent checks: a skew cap that bounds `|longLots - shortLots|`, and a solvency check that requires effective pool assets `E` to cover maximum liability `L` plus a 25 bps settlement buffer `B`. Neither check bounds the quantity that actually drives `L`.

`_maxLiability()` returns `max(longMaxProfit, shortMaxProfit)`, so a book that is perfectly flat in lots still carries liability proportional to total notional. `L` grows with the book while `E` only grows with margin and fees, so the two converge. A single unprivileged account can open a matched long/short pair sized so that the pair passes admission while leaving `E - L` smaller than one ordinary daily move of the FX basket. It then closes the leg that profits. `E` falls below `L`, the `degradedMode` latch sets, and LP withdrawals, epoch settlement and new orders stop across the protocol.

The latch is one-way. It clears only through `clearDegradedMode()`, which is `onlyOwner` and reverts `CfdEngine__StillInsolvent()` while `E < L`. There is no permissionless recovery, no timeout and no self-healing path — the latch survives seven days and survives the attacker fully unwinding the position that caused it.

On the Arbitrum Sepolia v2.0.0 deployment the shortfall at the latch is 469,484.035482 USDG against a 102.1M pool. The attacker's net cost to produce it is 535,699.503878 USDG, 52 bps of the pool.

## Vulnerability Details

Admission runs through `CfdEnginePlanLib`. The skew gate is at `CfdEnginePlanLib.sol:473-480`:

```solidity
if (
    plan.projectedSkewNotional > int256(uint256(cfg.maxSkewNotionalAssets)) ||
    plan.projectedSkewNotional < -int256(uint256(cfg.maxSkewNotionalAssets))
) {
    revert ICfdEngineTypes.CfdEngine__SkewCapExceeded();
}
```

`projectedSkewNotional` is signed. A matched long/short pair contributes zero to it. Nothing caps `longNotional + shortNotional`.

The solvency gate is `_applyOpenSettlementBufferAfterPlan` (`CfdEnginePlanLib.sol:698-779`), which requires `E_after >= L_after + B_after`, with `B = 25 bps` of `L` (`settlementBufferBps = 25`, `CfdEngine.sol:153`). This is a level check: it asks whether the pool covers the book right now, never how little headroom is left relative to a plausible price move.

`L` comes from `_maxLiability()` (`CfdEngine.sol:1044-1050`):

```solidity
function _maxLiability() internal view returns (uint256) {
    uint256 longMaxProfit = _sides[uint8(Side.LONG)].maxProfitAssets;
    uint256 shortMaxProfit = _sides[uint8(Side.SHORT)].maxProfitAssets;
    return longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
}
```

Per-lot max profit is asymmetric (`CfdMath.sol:76-89`): LONG pays `lots * entryPrice`, SHORT pays `lots * CAP - entryCost`. A flat pair therefore raises both side aggregates, raises `L` by roughly the larger of the two, and contributes nothing to the gate meant to bound risk.

The latch is `_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1351-1369`):

```solidity
function _enterDegradedModeIfInsolvent(
    address account,
    uint256 pendingPoolPayoutAssets
) internal {
    if (degradedMode) { return; }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
    if (effectiveAssetsAfter > pendingPoolPayoutAssets) {
        effectiveAssetsAfter -= pendingPoolPayoutAssets;
    } else {
        effectiveAssetsAfter = 0;
    }
    uint256 maxLiabilityAssets = _maxLiability();
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

Called from `_applyClose` (`:1320`) and `_applyLiquidation` (`:1344`), both with `pendingPoolPayoutAssets = 0`. The comparison is bare `E < L` — no buffer, no tolerance, no hysteresis. Admission demands `E >= L + B`; the latch fires at `E < L`. The gap between those two thresholds is the whole attack surface, and `B` is only 25 bps.

Once set, `CfdEngine`, `OrderRouter` and `HousePool` all gate on it: `CfdEngine__DegradedMode()` (`0x7e9b3b04`), `OrderRouter__DegradedMode()` (`0x22a6276e`, `IOrderRouterErrors.sol:122`), `HousePool__DegradedMode()` (`0x9f7adff3`). Third parties are turned away at `commitOrder`, before execution is reached.

Clearing is `onlyOwner` (`CfdEngine.sol:658-667`):

```solidity
function clearDegradedMode() external onlyOwner {
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared(msg.sender);
}
```

So the owner has to restore solvency before the latch can be lifted at all. `recapitalizePool` (`:632-653`) is the intended route and takes external USDG; it explicitly does not clear the latch by itself.

### Sequence

Against the live v2.0.0 book the pool held `totalAssets` 102,101,925.750632 with `L` 1,121,001.055324 and `B` 2,802.502639 — roughly 91x headroom.

A matched pair was grown to 1,038,532 long lots against 985,544 short lots at mark 96.784883. The fully loaded book carried `L` 101,828,834.230916 against `E` 102,103,847.378791. At that point `E - L` was 275,013.147875 while `B` was 254,572.085577, so admission still passed with 20,441.062298 to spare. The fill stopped because the **settlement buffer** ceiling was reached — `E / (1 + settlementBufferBps)` — not because of the skew cap, which is never the binding constraint for a matched pair.

One basket leg then moved -1.20% and the attacker closed the leg that profited. `E` fell to 101,359,350.195434 while `L` stayed at 101,828,834.230916. The latch set with a shortfall of 469,484.035482.

A 5 bps grid sweep puts the threshold between 70 and 75 bps: at 70 bps the close leaves `E - L` at +38,053.125140 and the latch does not fire; at 75 bps it fires with a shortfall of 12,751.338298. **75 bps is the smallest latching move measured.** For EUR/JPY/GBP/CAD/SEK/CHF against USD that is an ordinary day, not a tail event.

### Why the obvious defences do not fire

- **Skew cap** — matched pairs are skew-neutral by construction, so the signed cap never binds. The fill terminated on the settlement buffer instead.
- **The buffer itself** — 25 bps of `L`. The move needed to cross from `E >= L + B` to `E < L` is three times that, and still an unremarkable FX day.
- **Position and order limits** — `minOpenNotionalAssets` (1,000.000000), `maxPendingOrders` (5), the execution config hash and the pending/terminal status machinery all stay live and were probed as negative controls. They constrain how fast the book is built, not whether it can be built. 81 of 81 router orders accepted; zero terminal failures.
- **Pool size** — the attack scales with the pool, because `L` is driven by notional and notional is bounded only by the attacker's margin, which comes back on close.

### Scope of the claim

The latch does not turn other traders' open profits into unpayable claims. Checked directly on v2.0.0: an unrelated profitable trader was paid in cash — custody balance 99,901.298326 to 100,390.392410, `traderClaimBalanceAssets` 0 — and a flat account withdrew 100,390.392410 USDG while `degradedMode` was true. The freeze bites on LP settlement, new deposits, new orders and withdrawal liveness, which is exactly what the in-scope High describes.

## Impact Details

The in-scope impact is **temporary freezing of funds for at least 24 hours**. The duration is bounded only by how long the owner takes to notice and to source external USDG. Nothing in the contract shortens it, and two separate tests confirm it does not expire: the latch is still set after seven days, and still set after the attacker fully unwinds.

What stops while the latch is set:

| Surface | State |
|---|---|
| `pool.isWithdrawalLive()` | false |
| LP epoch settlement (`settleLpEpoch`) | reverts `HousePool__DegradedMode()` (`0x9f7adff3`) |
| New deposits / redemption claims | queued, unsettleable |
| New orders — commit stage | reverts `OrderRouter__DegradedMode()` (`0x22a6276e`) |
| Engine withdraw | reverts `CfdEngine__DegradedMode()` (`0x7e9b3b04`) |
| Blocked open, terminal code | `2` (DEGRADED_MODE) |
| Clearing | `clearDegradedMode()`, `onlyOwner`, reverts `CfdEngine__StillInsolvent()` while `E < L` |

Measured on the Arbitrum Sepolia v2.0.0 deployment at pinned block 325332107:

| Quantity | Value (USDG, 6 dp) |
|---|---|
| Pool `totalAssets` | 102,101,925.750632 |
| Senior / junior principal | 1,101,051.250778 / 101,002,963.167560 |
| `L` / `B` at baseline | 1,121,001.055324 / 2,802.502639 |
| `L` with the book loaded | 101,828,834.230916 |
| `E` with the book loaded, before the close | 102,103,847.378791 |
| Admission headroom at that point (`E - L - B`) | 20,441.062298 |
| **Shortfall at the latch (engine path)** | **469,484.035482** |
| Shortfall at the latch (router path) | 465,283.236004 |
| Shortfall after 7 days | 469,973.129566 |
| Smallest latching move measured | 75 bps |
| Owner USDG required to clear | 469,985.942576 |
| Attacker net cost | 535,699.503878 (52 bps of pool) |

The attacker's outlay does not vanish. 419,565.639614 lands in the pool and 116,133.864264 in the protocol treasury — 535,699.503878 exactly, the ledger closes with nothing unaccounted. About 22% of what the attacker spends ends up as protocol revenue.

Senior tranche LPs are the exposed class. Senior holds 1,101,051.250778 of the 102.1M, and the senior high-water-mark ratchet depends on periodic epoch settlement that the latch blocks.

### Severity

**Not Critical.** `recapitalizePool` plus `clearDegradedMode` is a working recovery path and the PoC exercises it end to end: the owner funds the pool from its own wallet with 469,985.942576 USDG, calls `clearDegradedMode()`, and `degradedMode` goes false. A separate test confirms `pool.isWithdrawalLive()` returns true once the latch is lifted. The freeze is long and unbounded, not permanent.

**Not griefing.** The program draws the line at $1 of attacker cost per $100 of damage. The attacker spends 535,699.503878 to freeze a 102,101,925.750632 pool — roughly 1:191, on the not-griefing side of the program's own threshold. The damage figure is the frozen pool, because LP capital and trader collateral are the funds whose access is denied; the owner's repair bill is a separate cost on top.

**Not "unable to operate due to lack of token funds."** That Medium category covers a pool that happens to be underfunded for its obligations. Here the pool was funded at 91x headroom and an unprivileged account walked it to insolvency through the sanctioned admission path. The missing bound is the bug.

### Relationship to known issues and existing tests

The Known Issues list, Cecuro's report, the SC Audit Studio pre-audit and the AI scan all cover degraded mode, but every one of them addresses what happens *after* the latch is set — stop-losses cannot be armed, integrators misread the state, lens classification is off. None asks who can set it. The closest adjacent item is Cecuro's note on stale planner margin aggregates, and Cecuro's own text states that it does not let a trader pass open-time solvency checks.

The repository has no test that attempts to force the latch from an unprivileged account. Existing degraded-mode tests set `degradedMode` through owner or harness paths and then assert downstream behaviour.

`CloseAccountingLib.sol:110-116` shows the design is alert to related pressure — its lifetime VPI clamp names the LP-sandwich attack in its NatSpec. The gap here is on the other axis: gross notional, not skew.

### Fix direction

Bound gross exposure, not just net. Any of these closes it:

1. Cap `longNotional + shortNotional`, or `L` itself, as a fraction of `E`, checked in `_applyOpenSettlementBufferAfterPlan` alongside the existing level check.
2. Size the settlement buffer off realized volatility rather than a fixed 25 bps, so admission leaves headroom proportional to a plausible move.
3. Make the latch re-evaluate: a permissionless clear once `E >= L + B` holds again. This cuts the duration — and alone would have ended the freeze the moment the attacker unwound — but does not stop the trigger.

## References

- `packages/perps/src/CfdEngine.sol` — `_maxLiability` 1044-1050, `_enterDegradedModeIfInsolvent` 1351-1369, `clearDegradedMode` 658-667, `recapitalizePool` 632-653, `_availableCashForFreshPoolPayouts` 865-871, `settlementBufferBps` 153, latch call sites 1320 and 1344
- `packages/perps/src/libraries/CfdEnginePlanLib.sol` — skew gate 473-480, settlement-buffer admission 698-779
- `packages/perps/src/HousePool.sol` — `settleLpEpoch` gate 797-820, `totalAssets` 426-429
- `packages/perps/src/CfdMath.sol` — `calculateExactMaxProfit` 76-89
- `packages/perps/src/interfaces/IOrderRouterErrors.sol:122` — `OrderRouter__DegradedMode`
- Deployment v2.0.0, Arbitrum Sepolia (421614), `sourceCommit 1209ca8a`: engine `0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA`, pool `0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e`, router `0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444`, clearinghouse `0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A`


## Link to Proof of Concept

https://gist.github.com/abdugani3261-ltm/ba1c1074612cd83e58e162745dd29f89

## Proof of Concept


## Proof of Concept

One Foundry run: 8 suites, **13 tests, 13 passed, 0 failed**. Everything forks Arbitrum Sepolia (chain 421614) at pinned block **325332107** and runs against the real v2.0.0 deployment — no mock engine, pool, router or clearinghouse. Only the Pyth endpoint and the ArbSys precompile are substituted, plus USDG, which the program permits for out-of-scope components.

The tests split into two groups that cover different halves on purpose:

| Group | What it drives | What it establishes |
|---|---|---|
| `DegradedLatchV2Test` — 6 tests (A-F) | Engine entrypoints with the router pranked | Admission, solvency, the latch and the recovery path in the **deployed bytecode** of `0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA` |
| `RouterPath*` — 7 tests | `OrderRouter.commitOrder` → `executeOrderBatch`, real keeper path | The same sequence is reachable by an ordinary external account through the public router, with no privileged role |

One shows the arithmetic holds in bytecode that is live right now; the other shows the public path to it is open. They are linked because the deployed bytecode and the audited commit are the same source:

```
$ git diff --stat 1209ca8a a82d4fef -- packages/perps/src
(no output)
```

`1209ca8a` is the `sourceCommit` recorded in the v2.0.0 deployment manifest; `a82d4fef` is the competition scope commit. `packages/perps/src` is byte-identical between them, so the commit-local router suite executes the same source that the fork suite executes as bytecode.

### Provenance

The repository ships two deployment manifests. The engine at `0x3dc9C0A1f9C745A4B08BD5C2E6c7aE613561c20D` is the superseded 2026-08-26 release (`sourceCommit 69fa3e2b`, runtime hash `0xf61a42cb75e6b83ccbbb1c7046f41d75c9d793de4105a6fe259eb8174c9b8b42`, codesize 24439) and does not match the audited commit. The matching deployment is v2.0.0, 2026-10-28:

```
engine         0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA
pool           0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e
clearinghouse  0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A
navbook        0x61C7f7Cf91c295a86250E1e05a91901e8E1e51A3
router         0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444
oracle         0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2
USDG (mock)    0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D
Pyth           0x0B73614636C855Bf23F342F307FB981A3e47f42B
```

Every address and every runtime codehash below is read from the fork inside the tests, not taken from the manifest. Engine runtime hash at the pinned block: `0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b`, codesize 23137 — exact manifest match. Pool `0x5e684cc8…`, router `0x6c65de67…`, execution sidecar `0xfc72fcbe…`, clearinghouse `0xc8e49658…`, oracle `0x1ecb1ca7…`, lifecycle book `0xc3f34d5c…`.

Both substituted contracts have their real codehash printed before the etch, so the swap is visible rather than implied: Pyth `0x7e693eea…`, and the ArbSys precompile at `0x…0064`, which is node-implemented (`0xfe`) and has to be etched for `ExecutionBlockNumberLib` to work under a fork. The real `singleUpdateFeeInWei()` is read from the live Pyth contract **before** the etch, and it is 0 on this deployment — so the fee the stub charges is the fee the real endpoint charges.

### Running it

```bash
export ARB_SEPOLIA_RPC="<archive RPC endpoint>"

forge test -vvv --fork-url "$ARB_SEPOLIA_RPC" --fork-block-number 325332107
```

That single command reproduces `FORGE_RUN.log` verbatim: 8 suites, 13 tests, all passing. To run one group on its own, add `--match-path 'test/DegradedLatchV2Audited.t.sol'` or `--match-path 'test/RouterPath*'`.

**An archive endpoint is required.** The public `https://sepolia-rollup.arbitrum.io/rpc` no longer serves state at block 325332107 and returns `historical state is not available`. Any archive provider for Arbitrum Sepolia works.

Toolchain: Foundry 1.5.1, solc 0.8.35.

Every figure below was produced under **both** solc pipelines and is bit-identical, so none of it is an artifact of the compilation mode: `FORGE_RUN.log` is `via_ir = false` (13/13) and `CROSSCHECK_via_ir_true.run.log` is `via_ir = true` (13/13). The package ships `via_ir = false` because under `via_ir = true` solc 0.8.35 common-subexpression-eliminates the `TIMESTAMP` opcode within a function, so a test contract's own `block.timestamp` read can be hoisted across a `vm.warp` — measured directly, a pre-warp read returns the post-warp value while an external call frame correctly observes the full 604800 s. Every timestamp this PoC relies on is read inside the engine, pool, router or clearinghouse, which are external frames; the cross-check log is what establishes that rather than leaving it to argument.

### Baseline read from the fork

```
chainid                   421614
pinned L2 fork block      325332107
block.number (L1 view)    12048497
pool totalAssets          102,101,925.750632
seniorPrincipal             1,101,051.250778
juniorPrincipal           101,002,963.167560
E before                  102,101,925.750632
L before                    1,121,001.055324
longMaxProfit               1,121,001.055324
shortMaxProfit                105,794.962268
settlementBufferBps / executionFeeBps     25 / 4
totalTraderClaimBalanceAssets                  0
```

### `DegradedLatchV2Test` — 6 tests, deployed bytecode

**test_A — the latch.** Entry mark 96.784883, attacker takes 1,038,532 long lots against 985,544 short lots. After a -1.20% move and closing the profitable leg:

```
E after close   101,359,350.195434
L after close   101,828,834.230916
shortfall L-E       469,484.035482
```

`settleLpEpoch` reverts `0x9f7adff3`, engine `withdraw` reverts `0x7e9b3b04`, a blocked open comes back with terminal code `2` (DEGRADED_MODE). `totalTraderClaimBalanceAssets` stays 0. A bystander with an open profit closes successfully and is paid **in cash** — custody 99,901.298326 to 100,390.392410 — with no trader claim recorded. A flat account withdraws 100,390.392410 USDG while `degradedMode` is true. Seven days on, `E / L` is 101,358,861.101350 / 101,828,834.230916 and the latch is still set.

**test_B — control, no close.** The same fully loaded book with no close: `E / L` = 102,103,847.378791 / 101,828,834.230916, not degraded. This is the admission headroom — `E - L` 275,013.147875 against `B` 254,572.085577, so 20,441.062298 of room over `L + B`. The fill loop stops here because the settlement-buffer ceiling `E / (1 + settlementBufferBps)` is reached, which is what makes it the binding gate rather than the skew cap.

**test_C — control, close at no profit.** `E / L` = 102,537,895.823670 / 101,828,834.230916, not degraded. The close alone does not latch; realized profit on the closing leg is what does.

**test_D — threshold sweep**, fresh attack at each step on a 5 bps grid:

```
65 bps   E 101,917,693.086374   degraded false
70 bps   E 101,866,887.356056   degraded false
75 bps   E 101,816,082.892618   degraded TRUE
80 bps   E 101,765,277.596354   degraded TRUE
85 bps   E 101,714,473.566960   degraded TRUE
90 bps   E 101,663,669.754586   degraded TRUE
```

`L` is 101,828,834.230916 throughout. **Smallest latching move: 75 bps.**

**test_E — full cycle costs.** The owner rescue is the real two-call path: `recapitalizePool` funds the pool from the owner's own wallet (it explicitly does not clear the latch), then `clearDegradedMode` lifts it. Both `onlyOwner`.

```
owner forced recapitalization       469,985.942576
attacker value before / after    40,000,000.000000 / 39,464,300.496122
attacker NET SPEND                  535,699.503878   (52 bps of pool)
pool totalAssets before / after 102,101,925.750632 / 102,991,477.332822
pool net gain                       889,551.582190
  of which owner recapitalization   469,985.942576
protocolTreasury net gain           116,133.864264
router/keeper net gain                           0
attacker residual pnl pledge              0 long, 0 short
```

419,565.639614 to the pool plus 116,133.864264 to the treasury is 535,699.503878 — the attacker's net spend, to the atom. Nothing is unaccounted and the attacker extracts nothing.

**test_F — no permissionless recovery.** After the latch, the attacker closes the entire losing leg, which collapses `L`:

```
latched              E 101,358,848.288341  L 101,828,834.230916
after full unwind    E 102,521,491.390246  L   1,121,001.055324
```

`E` now exceeds `L` by more than 101M. `degradedMode` is still `true` and `pool.isWithdrawalLive()` is still `false`. A call from `0xBADBAD` to `clearDegradedMode()` reverts. The owner then clears it having spent **0 USDG**, and `isWithdrawalLive()` returns true.

This is the part that turns a momentary crossing into a sustained freeze: the condition that set the latch is long gone and the protocol stays down until the owner intervenes.

### `RouterPath*` — 7 tests, public router path

**test_A — latch through the real router.** Config read from the fork: `maxPendingOrders` 5, `minOpenNotionalAssets` 1,000.000000, `openOrderExecutionBountyBps` 1, `maxOpenOrderExecutionBountyAssets` 0.200000. Entry mark 96.698850, mark after the move 95.542375.

```
open orders routed / terminal-failed        81 / 0
router nextCommitId after                        100
E after close / L after close    101,363,172.412589 / 101,828,455.648593
shortfall L-E                           465,283.236004
settleLpEpoch revert                          0x9f7adff3
commit revert (OrderRouter__DegradedMode)     0x22a6276e
E / L one week later             101,363,172.412589 / 101,828,455.648593
owner forced recapitalization           465,283.236005
real Pyth singleUpdateFeeInWei                   0 wei
pyth parse / update calls                     43 / 43
```

Eighty-one opens committed and executed, none rejected, none terminal-failed. A third party is turned away at `commitOrder`, before execution is ever reached. The owner's `clearDegradedMode()` is asserted to revert `CfdEngine__StillInsolvent` while `E < L`. A week later the shortfall has not moved.

**test_E — costs through the router.**

```
forced recapitalization             465,004.654686
owner USDG left after paying                      0
attacker NET SPEND                  536,608.954182   (52 bps of pool)
pool totalAssets before / after 102,101,925.750632 / 102,987,586.542675
protocolTreasury net gain           115,936.616825
keeper (executor) bounty income              16.200000
```

**test_G — admission guards are live.** Five negative controls confirming nothing was disabled to make the attack work:

```
G1  OrderRouter__CommitValidation          0xfc99bcef, code 11 (min notional)
G2  OrderRouter__ExecutionConfigMismatch   0x715f764c
G3  OrderRouter__TooManyPendingOrders      0xd997657d
G4  pending status / reason                1 / 4
G5  terminal status / reason                2 / 1
```

**test_F — no permissionless recovery, router side.** The router-path mirror of the engine-level test_F: the latch survives the attacker's full unwind and only the owner can lift it.

**test_B / test_C — controls through the router.** Same fill with no close, and with a close at no profit. Neither latches.

**test_00 — router smoke test.** A single order committed and executed end to end, establishing the router harness itself is sound before any of the attack tests run.

### Files

```
poc-fork/
  REPRODUCE.md                         one-command reproduction + the via_ir note
  foundry.toml                         solc 0.8.35, via_ir = false
  src/CfdTypes.sol
  src/RouterPathStubs.sol
  test/DegradedLatchV2Audited.t.sol    DegradedLatchV2Test                    (A-F, 6 tests)
  test/RouterPathBase.sol              shared router harness
  test/RouterPathLatch.t.sol           RouterPathLatchTest, RouterPathCostsTest,
                                       RouterPathControl1Test, RouterPathControl2Test,
                                       RouterPathRecoveryTest                 (5 tests)
  test/RouterPathAdmission.t.sol       RouterPathAdmissionTest                (G1-G5)
  test/RouterPathSmoke.t.sol           RouterPathSmoke
  FORGE_RUN.log                        shipped config, 8 suites / 13 tests / 13 passed
  CROSSCHECK_via_ir_true.run.log       same 13 tests under via_ir = true, identical figures
  DegradedLatchV2Audited.run.log       engine-level group on its own, 6 passed
```

All three logs are raw `forge test` output with gas, traces and emitted events intact.
