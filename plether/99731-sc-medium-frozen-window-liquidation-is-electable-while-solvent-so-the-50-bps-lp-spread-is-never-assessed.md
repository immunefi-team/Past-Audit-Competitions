# \#99731 [SC-Medium] Frozen-window liquidation is electable while solvent, so the 50 bps LP spread is never assessed

**Submitted on Nov 15th 2026 at 01:44:02 UTC by @ChaseTheLight for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99731
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

While the oracle is frozen the engine offers two full-exit routes for the same position at the same pinned mark, and prices them about 46 basis points of notional apart. A voluntary close assesses the LP-owned frozen-close spread, 50 bps of reduced notional, plus the 4 bps execution fee: `libraries/CloseAccountingLib.sol:122-124` sets `frozenSpreadAssets` whenever `inputs.oracleFrozen`, and `libraries/CfdEnginePlanLib.sol:1041-1044` folds it and the fee into the close action charge. The liquidation plan computes neither. `LiquidationAccountingLib.buildLiquidationState` takes ten parameters and none of them is the frozen-close spread or the execution fee (`libraries/LiquidationAccountingLib.sol:47-58`), its whole action charge is unpaid carry plus VPI clawback (`libraries/CfdEnginePlanLib.sol:1562`), and the liquidation plan never enters `CloseAccountingLib` at all.

The cheaper route is electable by the position holder while the position is fully solvent. For the whole frozen span the liquidation-eligibility test runs against the elevated frozen-market margin: `libraries/CfdEnginePlanLib.sol:1450` substitutes `fadMarginBps` for `maintMarginBps` whenever `snap.isFadWindow`, and `oracleFrozen` implies `isFadWindow` on every branch of `MarketCalendarLib.marketStatus` (`libraries/MarketCalendarLib.sol:30-58`). At the release parameters that is a 3% bar against a 0.1% bar, while the bar a position had to clear at open is the 0.2% initial margin (`libraries/CfdEnginePlanLib.sol:431-437` with `libraries/OpenAccountingLib.sol:137`). So any position between 0.2% and 3% equity is liquidation-eligible while far from insolvent, for about 48 hours every week, on a calendar the contract computes itself.

Anyone may make that call. `OrderRouter.executeLiquidation` carries no role modifier (`OrderRouter.sol:493-496`), the credited keeper is `msg.sender` (`router/OrderLiquidationBatchLogic.sol:728`), and the only test that address faces where the bounty is credited is a zero-address check (`MarginClearinghouse.sol:1264-1269`). The holder opens the position themselves while the FX market is open, holds it into the freeze, calls the liquidation on themselves, and is credited half of the liquidation charge. The same call made by a stranger force-closes the solvent position against the holder's will and pays the stranger 5 bps of the holder's own liquidation reserve.

That liquidation does not assess the spread is declared design, stated in five places and tabulated in a sixth. The reason the document gives for it sits two lines below the exemption itself, at `SECURITY.md:845`: "This is a deliberate trade-off: preserve close and liquidation liveness during real closures without weakening live-market MEV protections." That rationale is general. It does not discuss a holder who calls the liquidation on their own solvent position, or the choice between two exits that are both available at the same pinned mark. The design also names what liquidation retains in place of the spread, and that substitute is worth 2.5 bps against the 50 bps it replaces. Per elected exit the pool receives about 42 bps less of LP-owned trading revenue than the protocol's own pricing assigns to that exit, and the treasury receives none of the 4 bps execution fee.

The proof of concept was run locally and offline, in process inside a network namespace whose only interface is loopback and which had no route off it, with nothing signed and nothing broadcast. The run's own self-check attempts one connection, to an unbound loopback port, and records that it failed; no chain RPC, no fork endpoint and no system of the project's is configured anywhere in it (Proof of Concept).

## Vulnerability Details

### Root cause

One position in one regime has two priced exits, and only one of them carries the charge the regime exists to collect.

| | Declared voluntary close | Elected liquidation |
|---|---|---|
| Plan entry | `libraries/CfdEnginePlanLib.sol:846-847`, `_buildCloseState` | `libraries/CfdEnginePlanLib.sol:1495-1502`, `_buildLiquidationState` |
| Frozen-close spread | `notional * frozenCloseSpreadBps / 10_000`, 50 bps (`libraries/CloseAccountingLib.sol:123`) | not a parameter of the call (`libraries/LiquidationAccountingLib.sol:47-58`) |
| Execution fee | 4 bps, folded with the spread (`libraries/CfdEnginePlanLib.sol:1043`) | absent from the action charge (`libraries/CfdEnginePlanLib.sol:1562`) |
| Execution price | the stored mark, returned unshifted (`PletherOracle.sol:937-939`) | the stored mark shifted 2.5 bps adversely (`PletherOracle.sol:742`, `:963`) |
| Charges assessed on the exit | 54 bps of notional | `max(notional * bountyBps / 10_000, minBountyAssets)`, capped at the account's liquidation reserve (`libraries/LiquidationAccountingLib.sol:65-72`) |
| Who is paid | LPs take the spread (`SECURITY.md:839`), the treasury takes the execution fee (`CfdEngineSettlementSidecar.sol:703-711`, recipient at `:709`) | `keeperShareBps` to the caller, `protocolShareBps` to the treasury, the remainder to LPs (`libraries/LiquidationAccountingLib.sol:74-78`) |
| Who may call it | the account owner | anyone (`OrderRouter.sol:493-496`) |

The spread is not omitted from liquidation by a branch a guard might cover. It is unreachable from the liquidation plan by construction. `frozenCloseSpreadBps` matches twenty-one lines in `packages/perps/src` and is consumed for computation at exactly one of them, `CloseAccountingLib.sol:123`. The other twenty are the public declaration (`CfdEngine.sol:150`), a constructor and an admin bound check with their assignments (`CfdEngine.sol:293`, `:328`, `:335`, `:681`; `CfdEngineAdmin.sol:116`), snapshot plumbing that copies the value toward the close plan (`CfdEngineLens.sol:751`, `CfdEngineSettlementSidecar.sol:224`, `CfdOrderPolicyEvaluatorBase.sol:255`, `libraries/CfdEnginePlanLib.sol:1343`), three interface getter declarations (`CfdEngineSettlementSidecar.sol:97`, `CfdOrderPolicyEvaluatorBase.sol:95`, `interfaces/ICfdEngineCore.sol:80`), struct fields and doc comments. The computational site is reachable only through `_buildCloseState`, which has one caller, `libraries/CfdEnginePlanLib.sol:847`, on the close plan. The liquidation plan reaches `_buildLiquidationState` and stops there.

The second half of the asymmetry is what makes the exempt route available to an account that does not need it. `libraries/CfdEnginePlanLib.sol:1450`, verbatim:

```solidity
        uint256 maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps;
```

That rate becomes the maintenance requirement at `libraries/PositionRiskAccountingLib.sol:143`, and eligibility is `state.equityAssets <= int256(state.maintenanceMarginAssets)` at `:144`. The flag reaches the liquidation snapshot from the same calendar the engine exposes publicly: `CfdEngine.isFadWindow()` (`CfdEngine.sol:882-884`) is read into `snap.isFadWindow` at `CfdEngineSettlementSidecar.sol:218`. `MarketCalendarLib.marketStatus` sets both `fadWindow` and `oracleFrozen` true for the whole of Saturday, sets `fadWindow` 30 minutes earlier than `oracleFrozen` on Friday and releases it 15 minutes later on Sunday (`FAD_LEAD` at `:15`, `FAD_LAG` at `:17`), and returns `(true, true)` for an override day. So the elevated bar is in force for every moment the oracle is frozen, and strictly longer.

The bar the position cleared at open is a different number. An open reverts `INSUFFICIENT_INITIAL_MARGIN` unless projected equity is strictly above the initial-margin requirement and the projected position is not liquidatable (`libraries/CfdEnginePlanLib.sol:431-437`), and that requirement is `newLots * price * initMarginBps / 10_000`, floored at `minBountyAssets` (`libraries/OpenAccountingLib.sol:137-139`). At the release parameters the open bar is 0.2% and the in-window eligibility bar is 3%, so a fifteenfold band of solvent positions is liquidation-eligible for the duration of the freeze. The revert that reads as a solvency gate does not close it: `CfdEngine.sol:1037-1038` reverts `CfdEngine__PositionIsSolvent()` only on `!delta.liquidatable`, and `delta.liquidatable` was computed against the elevated bar three frames earlier. The error name describes solvency; the predicate tests the window.

Nothing on the call path refuses the liquidation or withholds the bounty when the external caller is the account being liquidated. `OrderRouter.executeLiquidation` is `external payable nonReentrant` with no role modifier, and the project's own docstring at `OrderRouter.sol:486` reads "Permissionlessly liquidates an unsafe account using an account-adverse oracle price." `router/OrderLiquidationBatchLogic.sol:728` passes `msg.sender` as the keeper, and the project's own parameter doc at `router/OrderLiquidationHandler.sol:22` reads "Original external caller credited with any liquidation bounty." No guard in the handler tests the external caller's identity: `executeLiquidationBatchItem` checks internal dispatch at `:34` and a zero size at `:38`, and `msg.sender` is read nowhere else in the file. Payment is `settlementBalances[execution.keeper] += execution.keeperBountyAssets` at `MarginClearinghouse.sol:1268`, guarded by a zero-address test at `:1265` and nothing else, funded by debiting `liquidationReserveBalances[account]` at `:1236-1240`. A census over all of `packages/perps/src` for a keeper comparison returns three sites, and none of them conditions the bounty credit on the caller's identity: `CfdEngine.sol:1331` gates the keeper's terminal-curve mutation, `:1336` gates the keeper's bounty-recipient checkpoint, and `MarginClearinghouse.sol:1265` is the zero-address test that stands immediately before the credit. The two engine sites both run before `CfdEngine.sol:1339`, which hands the same `keeper` to the sidecar either way. They are also affirmative evidence that `keeper == account` is a state the authors knew was reachable: with the two addresses equal the same account would otherwise be opened for terminal-curve mutation twice, so the collision had to be handled for carry accounting. It was not handled for payment.

The last piece is the size of the mitigation the design names as liquidation's replacement for the spread. `SECURITY.md:843` states it: "liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread." While the oracle is frozen that pricing is synthetic. `PletherOracle.sol:742` sets the basket confidence to `basket.price * basketMaxConfidenceRatioBps / 10_000`, under the project's own comment at `:741` ("Liquidations receive a deterministic, conservative confidence shift while Pyth is closed"), and `PletherOracle.sol:963` takes `confidence * adverseConfidenceMultiplierBps / 10_000` as the shift. At the checked-in release values, `basketMaxConfidenceRatioBps = 10` and `adverseConfidenceMultiplierBps = 2_500` (`packages/perps/DEPLOYMENT.md:146-147`, corroborated by `script/VerifyPerpsArbitrumSepolia.s.sol:44` and its deployment assertion at `:424`), the confidence is 10 bps of the mark and the shift is 2.5 bps of it. The frozen voluntary close is returned completely unshifted (`PletherOracle.sol:937-939`), under a comment that states the relationship in the project's own words:

```solidity
        // The fixed frozen-close spread replaces the Pyth adverse-confidence shift for voluntary
        // close/reduce execution while the oracle is frozen. Basket confidence-width validation still
        // applies when the basket is built, and opens (if called directly) retain the adverse shift.
```

The shift and the spread are declared substitutes for the same risk, and one of them is twenty times the other.

### What the design declares, and what it does not

Two of the three mechanics this report composes are declared, and both are design rather than defect.

The spread exemption is declared at `ACCOUNTING_SPEC.md:156` ("Liquidations never assess this spread."), `:1188`, `:1250`, `SECURITY.md:843` and `WHITEPAPER.md:1304`, and tabulated beside the voluntary close at `WHITEPAPER.md:1107-1113`, whose "Oracle frozen, within limit" row reads "Stored Engine mark plus fixed spread" for a voluntary close against "Stored mark with conservative confidence shift" for a liquidation. The elevated frozen-market bar is declared at `WHITEPAPER.md:1110` ("Elevated FAD margin") and `:1119` ("The reference FAD maintenance margin is 3%"), and setting a parameter to a value the code allows is a trusted action under the programme's own terms.

What is not declared is the composition of the two, exercised by an account no rationale in the package's documentation describes. The stated reason for the liquidation exemption is `SECURITY.md:845`, the summary sentence of the bullet list the exemption closes: "This is a deliberate trade-off: preserve close and liquidation liveness during real closures without weakening live-market MEV protections." It names closure liveness in general terms, and it does not address an account that holds both exits and takes the cheaper one. The lines that turn on collectibility govern a different mechanic: `SECURITY.md:841`, `:990` and `:1308` each describe the waiver of an uncollectible spread portion on a terminal full close, and `:1335` is a bullet in the list whose subject at `:1333` is "oracle-frozen voluntary close/reduce execution", so it waives the spread on the voluntary route rather than explaining why the liquidation route never assesses it. Those lines do not describe the account exercised here: at about 1.8% equity on a 5,000 USDG position it holds about 91 USDG of equity against 25 USDG of spread, so the spread is collectible, the account is solvent, the voluntary route is open to it at the same mark, and it elects the exempt route anyway. The third mechanic, payment of the liquidation bounty to the position holder, appears nowhere in the package's documentation: a search across all sixteen Markdown files under `packages/perps` for self-liquidation, liquidating one's own position, or a keeper-equals-account condition returns nothing. `ACCOUNTING_SPEC.md:1146-1147` states the allocation rule, crediting the keeper and treasury shares and transferring the LP remainder, and says nothing about the keeper being the liquidated account.

### Trigger and preconditions

Three conditions, no privileged role, and the actor supplies all three themselves.

1. **An open position whose equity sits between the two bars.** Strictly above `notional * initMarginBps / 10_000` (0.2% at release) because that is what the open required, and at or below `notional * fadMarginBps / 10_000` (3% at release) because that is what the window tests. In ordinary terms, anything from roughly 33x leverage up to the 500x the 0.2% initial margin permits.
2. **The position must be opened while the FX market is open.** The frozen and FAD regimes are close-only for opens and increases. `libraries/OracleFreshnessPolicyLib.sol:59-62` sets `closeOnly = oracleFrozen || isFad` for open execution; a fresh open commit reverts `OrderRouter__CloseOnlyWindow()` on that policy (`router/OrderLiquidationBatchLogic.sol:332-334`), an open already committed is left pending with `PendingReason.CloseOnly` (`OrderRouterExecutionSidecar.sol:442-443`), and the project tabulates the same thing as a blocked risk increase for both regimes (`WHITEPAPER.md:1110-1111`). An in-window open is refused before the elevated bar at `libraries/CfdEnginePlanLib.sol:685` is reached. The position is opened on a weekday and held into the freeze. This is a step of the path, not an obstruction.
3. **The account's liquidation reserve must cover the liquidation charge.** It does by construction: the open plan sizes the reserve to `max(notional * bountyBps / 10_000, minBountyAssets)` and funds it out of the trader's own pledge (`libraries/CfdEnginePlanLib.sol:551-556`, applied by `CfdEngineSettlementSidecar.sol:480-483` through `MarginClearinghouse.reclassifyPnlPledgeToLiquidationReserve`).

Nothing else is needed. The window arrives on the clock. No Pyth payload and no ETH are required to call the liquidation inside it, because `PletherOracle.getUpdateFee` returns zero when the payload is empty or the oracle is frozen (`PletherOracle.sol:465-467`), and the stored mark stays valid for the whole span against `fadMaxStaleness = 3 days` (`CfdEngine.sol:140`, consumed at `PletherOracle.sol:695`).

The numbered path from funding to settled exit, with every hop's entry point, is in the Proof of Concept section.

### Recommended fix

Charge the frozen-close spread on an exit that is only eligible because the window raised the bar, and charge it through the liquidation **action charge** rather than through the bounty.

`libraries/CfdEnginePlanLib.sol:1450` already computes the active rate from both candidates, so the plan can cheaply learn which bar made the position eligible, and the spread then belongs in the liquidation action charge at `libraries/CfdEnginePlanLib.sol:1562` (the change that does it, and what it flips, are in the Proof of Concept). That is the right lever for three reasons. The collected and withheld action charge is recorded to the pool as LP revenue (`CfdEngineSettlementSidecar.sol:836-845`), which is where a voluntary close's spread goes, so the LP share then matches what the declared route would have paid LPs for the identical exit at the identical mark. The action-charge path already waives whatever it cannot collect (`libraries/CfdEnginePlanLib.sol:1609`) and never reverts, so a liquidation cannot be made to fail by adding a charge, which keeps the liveness the exemption was written for. And a genuinely unsafe account, liquidatable at the maintenance bar too, is left untouched by the condition.

The bounty is the wrong lever, and it is the lever a reader will reach for first. The charge computed in `LiquidationAccountingLib` is capped at `reachableCollateralAssets` (`libraries/LiquidationAccountingLib.sol:70-72`), and the value passed for it is `snap.liquidationReserveAssets` (`libraries/CfdEnginePlanLib.sol:1499`). That bucket is sized at open to `max(notional * bountyBps / 10_000, minBountyAssets)` (`libraries/OpenAccountingLib.sol:148-152`), which is 10 bps of notional at the release parameters. For a position already open against a reserve sized at the old rate, raising `bountyBps` or the floor is clamped straight back down to that reserve. For positions opened afterwards that same open-time sizing funds the larger reserve too, so the lever works prospectively only, and it raises the charge on every liquidation rather than on the exits this report is about.

The fix needs no new plumbing. The snapshot the liquidation plan receives already carries both inputs: `CfdEnginePlanTypes.RawSnapshot` declares `oracleFrozen` and `frozenCloseSpreadBps` (`CfdEnginePlanTypes.sol:144-145`), the engine's own liquidation path builds that snapshot through `CfdEngineSettlementSidecar.buildRawSnapshot` with the close-commit flag false (`CfdEnginePlanningReader.sol:61-63`, `CfdEngineSettlementSidecar.sol:154-161`), and both fields are therefore populated at `CfdEngineSettlementSidecar.sol:223-224`. Each is then read at exactly one place in the planner, `libraries/CfdEnginePlanLib.sol:1343-1344`, inside the close-state inputs. The liquidation plan holds the frozen flag and the spread rate and reads neither.

The structural fix is to make a frozen-window exit cost one thing regardless of which route carries it. Either assess the spread on every frozen-window exit and let the collectibility cap waive it where it genuinely cannot be paid, which is the rule the voluntary path already uses, or raise the frozen liquidation shift to match the spread, which the comment at `PletherOracle.sol:934-936` describes as its substitute on the voluntary route. Equalizing the two is a proposal here rather than something that comment requires. Today the two numbers differ by a factor of twenty and the account picks which one applies to it.

A third change closes the smallest limb on its own: pay no keeper share when `keeper == delta.account`. The engine already tests that identity twice, at `CfdEngine.sol:1331` and `:1336`, so the condition is available where the bounty is allocated. It moves the elected exit from about 7.5 bps to about 12.5 bps against a declared route at 54 bps, which recovers roughly 5 bps of a 46 bps gap and leaves the remaining 41 bps carried by the spread exemption itself. It is not a substitute for either change above.

## Impact Details

### The in-scope impact this maps to

The impact tier this report files on, condensed from the programme's Impacts in Scope list:

```text
Medium: theft of gas; SC unable to operate due to lack of token funds; block stuffing; griefing (no
profit motive but damage to users/protocol); unbounded gas consumption; temporary freezing of funds ≥1h.
```

The entry within it that governs is griefing, which the page states as "no profit motive but damage to users/protocol". This report establishes that damage on both counts, to users and to the protocol. What the holder gains on the route they elect is a charge that was never assessed rather than a transfer out of a balance at rest, and the figure is in the sections above. Where the damage falls on another account, the caller's own gain on the call is the keeper share the deployment sets for a liquidation it treats as eligible, 5 bps of notional as measured (`packages/perps/DEPLOYMENT.md:139-141`, and Proof of Concept test 5).

This report files at **Medium** on that entry, and both halves of the damage it names are present: about 42 bps of notional in LP-owned revenue and the treasury's 4 bps fee on each elected exit, and the forced closure of a solvent position when the caller is a stranger. The two limbs rest on different parts of that entry. A liquidation called by a stranger meets it as written: the caller's only gain is the protocol's own posted bounty, 5 bps of notional as measured, and the damage is the forced closure. A liquidation elected by the holder rests on the damage to users and the protocol, measured at 42.2 bps of LP-owned revenue and the treasury's 4 bps on each elected exit. Against the voluntary close at the same mark the elected route is about 46 bps of notional cheaper for the holder, a figure the sections above derive from the two routes' costs at the release parameters and that the Proof of Concept measures on the withdrawal and the conservation identity.

Damage to the protocol is the LP-owned charge. The frozen-close spread is LP-owned trading revenue by the protocol's own definitions: "the spread is LP-owned and never protocol-treasury revenue" (`SECURITY.md:839`), "assessed spread actually retained or collected as LP trading revenue" (`ACCOUNTING_SPEC.md:153`), reaching the pool through `recordClaimantInflow(..., Revenue, ...)` at `CfdEngineSettlementSidecar.sol:606-613` and the retained-revenue path at `:616`. On the elected route it is never assessed, so the pool is paid about 42 bps of notional less for the same exit, and the treasury is paid none of the 4 bps execution fee. No LP claim to that spread ever accrues on the elected route and none is taken: `libraries/CloseAccountingLib.sol:122-124` computes it only inside `if (inputs.oracleFrozen)` on the close path, and the liquidation plan never enters that library.

Damage to a user is the same call made by someone else. A stranger can force-close a fully solvent position for the whole of every frozen window and be credited 5 bps of the holder's own liquidation reserve for doing it. The holder loses the position at a mark pinned to the pre-freeze price, cannot reopen until the window ends because the frozen and FAD regimes are close-only for opens and increases (`libraries/OracleFreshnessPolicyLib.sol:59-62`, enforced at `router/OrderLiquidationBatchLogic.sol:332-334` and `OrderRouterExecutionSidecar.sol:442-443`), and has one prospective defence: `CfdEngine.addMargin` is gated only on `msg.sender == account`, a nonzero amount and an open position, with no window gate (`CfdEngine.sol:501-515`), so a holder who is watching can top up above the elevated bar. That bounds this limb; it does not remove it, because the holder has to know the bar moved.

The programme's own keeper rule puts the caller in scope either way:

```text
**KEEPERS ARE NOT TRUSTED: anyone can execute orders, liquidations, price updates and protections, so
harm caused by any caller acting as a keeper is IN SCOPE.** Keepers have no special privileges; all
actions remain subject to on-chain checks. Consider any contract-permitted behavior, including
deviations from the reference implementation.
```

### Loss mechanism and its bound

The two page-named charges fix the size. At the configuration the programme names as the one severity is assessed at, the declared route assesses 50 bps of frozen-close spread plus 4 bps of execution fee, 54 bps of notional, folded into one action charge at `libraries/CfdEnginePlanLib.sol:1043`. The elected route assesses none of it.

What the elected route charges instead follows from the checked-in liquidation defaults: `bountyBps = 10`, `keeperShareBps = 5_000`, `protocolShareBps = 0` (`packages/perps/DEPLOYMENT.md:139-141`). The charge is 10 bps of notional, half of it credited to the caller and the rest to LPs, so when the caller is the holder the net charge is 5 bps. Add the 2.5 bps adverse price shift the elected route retains and the holder pays about 7.5 bps where the declared route costs 54 bps. The gap is about 46 bps of notional per elected exit: about 42 bps of it LP-owned revenue, 4 bps of it the treasury's fee. As measured, the voluntary close also returns the 0.28 bps of VPI the position paid at open and the liquidation does not, so the gap is 46.2 bps, of which 42.2 is the pool's. The Proof of Concept section measures the atoms on both branches from one shared state and asserts that the holder's gain equals the pool's loss plus the treasury's loss exactly.

On both routes the pool's balance rises. The finding is the size of the rise, measured against the route the protocol's own pricing assigns to this exit. That is also what bounds the claim: this is revenue forgone on an exit that happened, not value removed from a balance at rest, so the per-exit figure applies to a frozen exit that would otherwise have gone through the declared route, not to every open position.

Per call the absolute size is the position's notional, and that is bounded by the engine's own open-side skew limit, a fraction of pool depth given by `maxSkewRatio` (`libraries/CfdEnginePlanLib.sol:473`, checked in at `0.4e18`). The rate is the stable number and it does not vary with size: at the checked-in minimum open notional of 1,000 USDG (`packages/perps/DEPLOYMENT.md:148`) the saving is about 4.6 USDG against cents of Arbitrum gas, so the election pays at every size the router accepts.

The frequency is fixed by the calendar rather than by an attacker. The frozen span runs from the 17:00 New York Friday boundary to the 17:00 Sunday boundary, with the elevated bar extending 30 minutes earlier and 15 minutes later, and the engine recomputes it from `block.timestamp` on every read (`libraries/MarketCalendarLib.sol:30-58`, docstring at `CfdEngine.sol:878-880`). Governance may add all-day holiday overrides. So the election is available roughly 48 hours in every week, to every account in the band, without coordination.

### Why not lower, and the out-of-scope entries to weigh

The effect is a recurring per-call loss of LP-owned revenue to the pool together with the forced closure of a solvent account's position, which is the damage the griefing entry above names. The comparison that decides it is against the best legitimate alternative rather than against nothing:

| The holder's alternatives inside the window | What it costs them | Does the fix change it? |
|---|---|---|
| Wait for the reopen, then close | the execution fee; no spread applies outside the window (`SECURITY.md:1333`) | No |
| Voluntary close inside the window | 54 bps at the page configuration | No |
| Elect the liquidation inside the window | about 7.5 bps | Yes, to about 57 bps, so the declared route becomes the cheaper one |

For a holder content to wait, the fix changes nothing and LPs were never owed a spread by them. For a holder who wants out during the freeze, the fix moves LP revenue on that exit from about 7.5 bps to about 50 bps if they then take the declared route, and to about 57 bps if they elect the liquidation anyway. That second population is the one the spread was built to price: a trader pays 54 bps to exit early only when exiting early is worth more than 54 bps to them, which is the adverse-selection exit the charge exists for. `WHITEPAPER.md:1422-1433` tabulates the historical Friday-to-Monday reference-rate moves, then states that the sample does "**not** validate a 3% FAD margin basis as sufficient", and then says that those thresholds are not liquidation buffers and that actual eligibility depends on starting equity measured against "the applicable maintenance basis". The applicable basis is the quantity this window changes.

**"Best-practice recommendations (except as Insights)"** does not apply. Value measurably moves between named parties on a permissionless call: the caller's settlement balance rises, the pool's total assets rise by less than the declared route would have given them, and the treasury's balance does not move at all. There is no recommendation here to decline.

**"Issues reachable only via inactive feature gates/flags/configs/networks"** does not apply. Permissionless keepers are named as active at launch on the programme page, and the page states that severity is assessed with the release parameters in `packages/perps/DEPLOYMENT.md` at this commit, listing the frozen-close spread and the execution fee among them. Every parameter the saving depends on is committed in that table at the value this report runs at, and the configuration parity table names each one alongside the two senior-cap and one minimum-notional harness deviations the finding does not depend on (Proof of Concept).

**"Setting a parameter to a value the code allows is a trusted action, not a bug"** does not reach it either. The report does not argue that `fadMarginBps = 300` is wrong. Even at a 0.5% frozen-market bar the same election is available to any position above 200x, because the defect is that the exempt route exists alongside the charged one for the same exit and the account chooses between them.

**The containment clause** does not reduce it. A guardian can pause new trading risk, LP deposits and LP epoch settlement, and the programme's own text states that "Closes, liquidations, and already-funded claims remain available" and that the existence of monitoring or a pause alone does not justify a downgrade. The exit routes this report compares are both on the list that stays available.

What this is not: no principal at rest is drained, nothing is frozen, no funds are unavailable for any period, and there is no insolvency. The position closes and the holder is paid out on both routes.

### Not a duplicate, and scope

Nothing in the programme's known-issue set carries this root cause. A term census over all five audit documents in the listed repository returns zero hits for `frozenCloseSpread`, zero for self-liquidation phrasing in any spelling, and zero for `keeperShare`; the single hit for "frozen spread" is about protection caps.

The nearest row by vocabulary is **LOW-51**, "Integrator can understate active maintenance during FAD windows", read in full. It is a view-only defect: `previewOpen()` copies a maintenance figure computed from `maintMarginBps` into a preview before checking validity, so an off-chain caller is shown the wrong number while the engine enforces the elevated one, at `CfdEngineLens.sol:258-268` and `libraries/OpenAccountingLib.sol:136-139`. Its remediation comment reads Addressed, with a fix commit, and the audit's summary table lists it as Acknowledged. Different root cause, different files, no charge asymmetry, and nothing in it touches the liquidation plan or the spread. This report is not on LOW-51's root cause, so the incomplete-fix rule does not apply to it. One datum from its own preconditions is worth keeping: its third precondition records that "Deployment uses fadMarginBps > maintMarginBps, which is allowed and used by the protocol".

**MED-10**, "Liquidator can liquidate short traders at prices below the configured cap-adjusted mark", is the nearest row by location: it lists `packages/perps/src/PletherOracle.sol:742-770` among its locations and this report cites `PletherOracle.sol:742` for the size of the retained shift. It is a different defect. MED-10 is about the order in which the confidence adjustment and the `CAP_PRICE` clamp are applied to a SHORT position above the cap, which produced a mispriced liquidation; its remediation comment reads Addressed, the audit's summary table lists it as Acknowledged, and that comment records that confidence adjustments remain part of liquidation pricing and that the fix changed their ordering rather than the policy. This report takes that policy as given and does not dispute the ordering: it is about a charge that one of two exit routes assesses and the other does not, on a LONG position far below the cap, at a mark the clamp never touches.

**HIGH-1**, the Arbitrum and Pyth clock alignment limitation, is a different mechanism and this report does not rely on it. That limitation is about commit timing against an already-published price. This report is a flat charge asymmetry at a mark that cannot move, with both branches priced off the same stored value, and it does not depend on the fifteen-second commit window or on seeing a price before committing.

The four `SECURITY.md` sections the programme declares known do not contain the root cause. `## Known Limitations` runs `SECURITY.md:1328-1457`, and the statement that liquidation does not assess the spread sits at `:843`, inside `### Oracle regimes` (`SECURITY.md:819`), under `## Oracle And Execution Security` (`:629`), which ends where `## Accounting And Solvency Security` opens at `:847`. The lines inside the declared range that come closest are about the spread not scaling with mark staleness (`:1334`) and about full-close liveness over an uncollectible spread (`:1335`); neither describes a solvent account electing the exempt route, and neither mentions the frozen-market margin interaction that makes the election available for the whole window. `ACCOUNTING_SPEC.md`, `WHITEPAPER.md` and `DEPLOYMENT.md` are not on the declared-known list at all.

The disclosure bar runs on the programme's listed repositories, and the page lists one. That repository carries zero public issues and zero pull requests of any state, and the scoped commit is its default branch's head, so nothing has been publicly disclosed there and there is no public fix between the scoped commit and now.

The project's own public repository, from which the listed one is forked, is not a listed repository under that bar. It is one commit ahead of the scoped commit, and that commit touches only `.gitignore`, a test map and five test-running scripts, so no file under `packages/perps/src` has changed and the defect stands at that repository's head as well. Its full public issue and pull-request set carries nothing on this root cause: no item describes the charge asymmetry between the two frozen-window exit routes, and none describes the caller being credited the bounty.

Every defect site and every hop on the path is inside `packages/perps/src` and none of them is among the six view-only contracts the programme excludes: `libraries/CloseAccountingLib.sol`, `libraries/LiquidationAccountingLib.sol`, `libraries/CfdEnginePlanLib.sol`, `libraries/PositionRiskAccountingLib.sol`, `libraries/MarketCalendarLib.sol`, `libraries/OpenAccountingLib.sol`, `CfdEngine.sol`, `CfdEngineSettlementSidecar.sol`, `OrderRouter.sol`, `router/OrderLiquidationBatchLogic.sol`, `router/OrderLiquidationHandler.sol`, `MarginClearinghouse.sol` and `PletherOracle.sol`. `CfdEngineOpenQuoter.sol`, which is excluded, is not on the path: it is instantiated by `CfdEngineLens.sol:53` and called only from `CfdEngineLens.sol:132`, a view surface. The only stand-ins the proof of concept uses are for Pyth price updates and the settlement token, the two components the programme permits mocking, and no in-scope contract is modified.

## References

Repository, pinned to the scoped commit: `https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Paths below are relative to `packages/perps/`.

1. `src/libraries/CloseAccountingLib.sol` - `#L122-L124` (the spread, minted only while the oracle is frozen, on the close path), `#L112-L115` (the project's own comment: frozen stale-price risk is priced separately through `frozenSpreadAssets`)
2. `src/libraries/CfdEnginePlanLib.sol` - `#L1041-L1044` (spread and fee folded into the close action charge), `#L1450` (the eligibility bar substitution, the defect), `#L1495-L1502` (the liquidation state build), `#L1499` (the liquidation reserve passed as the charge cap), `#L1562` (the whole liquidation action charge, and the fix site), `#L1609` (the action charge waives what it cannot collect), `#L846-L847` (the one caller of `_buildCloseState`), `#L431-L437` (the open-time bar), `#L685` (the margin rate the projected-open risk state is built at), `#L551-L556` (the liquidation reserve sized at open), `#L473` (the open-side skew limit), `#L882-L885` (the partial-close charge guard, `remainingSize > 0` gated and on the close plan only)
3. `src/libraries/LiquidationAccountingLib.sol` - `#L47-L58` (ten parameters, neither the frozen-close spread nor the execution fee), `#L65-L72` (the charge, its floor and its cap), `#L74-L78` (the keeper, treasury and LP split)
4. `src/libraries/PositionRiskAccountingLib.sol` - `#L143-L144` (the maintenance requirement and the eligibility predicate)
5. `src/libraries/MarketCalendarLib.sol` - `#L15`, `#L17` (`FAD_LEAD`, `FAD_LAG`), `#L30-L58` (`marketStatus`, all four branches and the override runway)
6. `src/libraries/OpenAccountingLib.sol` - `#L137-L139` (the initial-margin requirement), `#L148-L152` (the liquidation reserve target, sized to the bounty only)
7. `src/CfdEngine.sol` - `#L1037-L1038` (the revert that tests the window rather than solvency), `#L1331`, `#L1336`, `#L1339` (the two keeper identity tests, and the unconditional hand-off on the next line), `#L882-L884` (`isFadWindow`), `#L878-L880` (the calendar docstring), `#L140` (`fadMaxStaleness`), `#L501-L515` (`addMargin`, the holder's prospective defence)
8. `src/CfdEngineSettlementSidecar.sol` - `#L218` (the window read into the liquidation snapshot), `#L223-L224` (the frozen flag and the spread rate, both populated on that snapshot), `#L154-L161` (`buildRawSnapshot`, close-commit flag false), `#L480-L483` (the liquidation reserve applied at open), `#L606-L616` (LP revenue recorded from a close), `#L836-L845` (LP revenue recorded from a liquidation's action charge), `#L703-L711` (the close path's execution fee credited to the protocol treasury); `src/CfdEnginePlanTypes.sol` - `#L143-L145` (the snapshot's `isFadWindow`, `oracleFrozen` and `frozenCloseSpreadBps` fields); `src/CfdEnginePlanningReader.sol` - `#L61-L63` (the engine's own liquidation path, and its snapshot build); `src/libraries/CfdEnginePlanLib.sol` - `#L1343-L1344` (the only site that reads either field, inside the close-state inputs)
9. `src/OrderRouter.sol` - `#L486` (the project's own "Permissionlessly liquidates" docstring), `#L493-L496` (no role modifier), `#L85` (`commitOrder`), `#L240` (`executeOrder`)
10. `src/router/OrderLiquidationBatchLogic.sol` - `#L728` (`msg.sender` passed as the keeper), `#L321-L334` (`_validateOpenCommitAllowed`, the close-only revert on the open commit path); `src/router/OrderLiquidationHandler.sol` - `#L22` (the project's own parameter doc), `#L34`, `#L38` (the internal-dispatch and zero-size tests on `executeLiquidationBatchItem`); `src/OrderRouterExecutionSidecar.sol` - `#L442-L443` (an in-window open left pending as `CloseOnly`); `src/libraries/OracleFreshnessPolicyLib.sol` - `#L59-L62` (`closeOnly = oracleFrozen || isFad` for open execution), `#L37` (the project's own comment on it)
11. `src/MarginClearinghouse.sol` - `#L1234-L1235` (the charge composition), `#L1236-L1240` (debited from the account's liquidation reserve), `#L1264-L1269` (credited to the keeper, zero-address test only), `#L359` (`depositMargin`), `#L414-L418` (`withdrawMargin`)
12. `src/PletherOracle.sol` - `#L934-L939` (the frozen voluntary close returned unshifted, under the comment stating the spread replaces the shift), `#L741-L742` (the synthetic frozen confidence), `#L963` (the liquidation shift), `#L465-L467` (no update fee while frozen), `#L95` (the field initializer for the multiplier, which the release configuration overrides)
13. Declared design and its stated rationale: `ACCOUNTING_SPEC.md` - `#L153`, `#L156`, `#L1146-L1147` (the liquidation-charge allocation rule), `#L1188`, `#L1250`; `SECURITY.md` - `#L839`, `#L841`, `#L843`, `#L845`, `#L990`, `#L1308`, `#L1328-L1336`; `WHITEPAPER.md` - `#L1107-L1113`, `#L1110`, `#L1119`, `#L1304`, `#L1422-L1433`
14. Release parameters: `DEPLOYMENT.md` - `#L128-L156` (the table), `#L132` (`frozenCloseSpreadBps 50`), `#L134-L136` (`maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`), `#L139-L142` (`bountyBps 10`, `keeperShareBps 5_000`, `protocolShareBps 0`, `executionFeeBps 4`), `#L146-L147` (`basketMaxConfidenceRatioBps 10`, `adverseConfidenceMultiplierBps 2_500`)

File fingerprints at the scoped commit, for the principal files the claims above sit in:

| File | Bytes | Lines | sha256 |
| --- | --- | --- | --- |
| `src/libraries/CfdEnginePlanLib.sol` | 93,821 | 1,678 | `1962d7a3df8e23ca9b4059e45f2496b6bbe1a3e56f722b09e5b8541be9f7ae5f` |
| `src/libraries/CloseAccountingLib.sol` | 8,067 | 129 | `5d252ee0652cfc52259511daadaa54404e809c5486f434477a585720071796b9` |
| `src/libraries/LiquidationAccountingLib.sol` | 4,840 | 81 | `a71b98eea1133636c97a762249bfb9b551f8937a8550afc44bfe0dc9d4e8c1e8` |
| `src/libraries/PositionRiskAccountingLib.sol` | 7,841 | 147 | `aa3025cbb4de66c0c1191665a1b964f64e214a6ab3ef6e0bce9aab3120286183` |
| `src/libraries/MarketCalendarLib.sol` | 6,302 | 132 | `c3fdc6f9b81a256aa93ea0eea8d335826a0118d116ea4ecd710c7116bc0eb7df` |
| `src/CfdEngine.sol` | 70,136 | 1,484 | `98033dd004224072f6f7f3a6bfded8115ed36e3a10843300515b97e360aaee0c` |
| `src/OrderRouter.sol` | 29,515 | 557 | `8f8318db0889f5d098d2f5c8c6afc41c473d1ca516817fe2363ba3899bb48d7d` |
| `src/MarginClearinghouse.sol` | 86,100 | 1,833 | `20645164feef3d812ed20e277971046878650188de74ec490fbdcf3cf004ad09` |
| `src/PletherOracle.sol` | 51,762 | 1,070 | `a837f316cc8509c95f1303d5903a16f3c10c52c0e8ec86ae2aacae90dde25daa` |

## Link to Proof of Concept

https://gist.github.com/ChaseTheLight01/f399daf3f29a451105e9a546cf08d0ee

## Proof of Concept

## Proof of Concept

The test file, the runner, the suggested change, `RUN.md` and the capture are in this Gist: https://gist.github.com/ChaseTheLight01/f399daf3f29a451105e9a546cf08d0ee

One test file, five tests, extending the project's own Foundry test base at the scoped commit. The two exits under comparison run from a single shared state, so skew, lifetime VPI, carry, pool depth and the stored mark are identical on both of them. Both price off that one stored mark, and the test separates the two components of the difference between them: the 2.5 bps adverse shift the elected route retains, asserted as the whole price difference, and the charges each route assesses.

The control that makes the comparison mean anything is the second test, which runs the same position outside the window. Without it the comparison cannot show that the eligibility came from the window rather than from the position.

| Test | Enters at | Measures |
| --- | --- | --- |
| 1. the two frozen-window exits (`test_frozenWindowOffersTwoExitsChargedTensOfBasisPointsApart`) | `MarginClearinghouse.depositMargin`, then `OrderRouter.commitOrder` and `OrderRouter.executeOrder` for the open, then `OrderRouter.commitOrder` and `OrderRouter.executeOrder` again for a full close on one branch and `OrderRouter.executeLiquidation` on the other, both from one snapshot, and the same liquidation a third time from that snapshot with an unrelated address as the caller | the trader's, the pool's and the treasury's balance deltas on each branch, the spread each branch settles as the engine's own event reports it, the caller's keeper credit when the caller is the holder, isolated against the branch where the caller is someone else, the mark each branch used, the conservation identity between the branches, and the settlement token actually withdrawn afterwards on each branch through `MarginClearinghouse.withdrawMargin` |
| 2. solvency against the active bar (`test_solventPositionIsLiquidatableOnlyInsideTheFrozenWindow`) | the planner's liquidation read, and `OrderRouter.executeLiquidation` both outside and inside the frozen window | that at the moment of the elected exit the position's equity is strictly above the initial-margin requirement it was opened against and at or below the frozen-market requirement, that the planner reports the identical position not liquidatable outside the window and liquidatable inside it, and that the call reverts `CfdEngine__PositionIsSolvent()` outside the window and executes inside it |
| 3. the declared substitute, at the release value (`test_frozenLiquidationRetainsTwoAndAHalfBasisPointsOfShift`) | the price the elected exit executed at, against the engine's stored mark | the shift the oracle applies to a frozen liquidation price, at `adverseConfidenceMultiplierBps = 2_500`, expressed in basis points of that mark |
| 4. the calendar (`test_frozenWindowRecursWeeklyOnTheContractsOwnCalendar`) | `CfdEngine.isFadWindow()` and `CfdEngine.isOracleFrozen()` at six timestamps, and `MarketCalendarLib.marketStatus` directly at the same six | that the pair is (false, false) on a weekday, (false, false) ten minutes before the Friday shoulder opens, (true, false) inside that shoulder, (true, true) after the Friday boundary, (true, true) mid-weekend, and (true, true) again seven days later, with the boundary itself derived from the library rather than written into the test |
| 5. the same call by a stranger (`test_unrelatedCallerForcesTheSolventPositionOut`) | `OrderRouter.executeLiquidation` from an unrelated address | that a fully solvent position is force-closed and the caller is credited the share the planner computed; that the share is funded by debiting the holder's own liquidation reserve is source, at `MarginClearinghouse.sol:1236-1240` |

### Where the proof of concept runs, and what it never touches

The test was run locally and offline. The harness is the project's own Foundry suite, run in process with `forge test` against a local checkout of `packages/perps` at the scoped commit, with the build artifacts written to local disk. One test file is added to the project's own test directory and nothing else in the checkout is touched in the run that produces the first capture.

The runner enters a network namespace itself, with `unshare -rn`, and checks from inside it before any test runs: that the only interface is `lo`, that there is no usable IPv4 or IPv6 route, and that a loopback connect probe behaves as expected on an unbound port. If any check fails the runner stops without running a test. Those checks are the first lines of the capture the runner writes, which is the first of the two blocks at the end of this section.

No node is started and no chain client of any kind is run, no chain RPC is called, no fork endpoint is configured and no `--fork-url` is passed, no Plether API or other system of the project's is contacted, no key material is used, and nothing is signed or broadcast. Every transaction in the test executes in the Foundry EVM's own memory and exists only there.

Two in-process stand-ins, and they are the only two: the project's own `MockPyth` contract stands in for Pyth price updates, and its own `MockSettlementToken` stands in for USDG. Those are the two components the programme permits mocking, and no contract in `packages/perps/src` is replaced, patched or stubbed in the run that produces the first capture. The engine, the planner, the settlement sidecar, the clearinghouse, the pool, the tranche vaults, the order lifecycle book, the oracle and the lens are the ones at the scoped commit. One point of precision about the router: the test base the whole perps suite is built on deploys `LegacyOrderRouterHarness`, which inherits `OrderRouter` and adds a single helper for committing an order with scalar arguments. This test does not call that helper. `commitOrder`, `executeOrder`, `executeLiquidation` and `updateMarkPrice` are the inherited functions, unmodified.

Nothing in this report rests on a request to a Plether endpoint, a chain RPC, a block explorer or a node.

### The position that is attacked, and how the caller comes to hold it

The object is **one open position on the USD-basket perpetual market**, held by an account whose equity at the stored mark is strictly above the initial-margin requirement it was opened against and at or below the frozen-market requirement that applies for the whole frozen window.

**The caller is the position holder, and they create the position themselves.** They deposit their own margin through `MarginClearinghouse.depositMargin`, commit the open through `OrderRouter.commitOrder` and execute it through `OrderRouter.executeOrder`, on a weekday while the FX market is open, and hold it into the calendar freeze. The collateral at risk is their own and the position is their own, so there is no third-party object to find and no question about whether one exists: the test creates the position through the project's own order path rather than writing a position into storage, and asserts `isOracleFrozen()` and `isFadWindow()` both false at the moment of the open.

The fifth test is the same defect reached by a different caller. Its holder opens the position through the same order path as the first three tests (`MarginClearinghouse.depositMargin`, `OrderRouter.commitOrder`, `OrderRouter.executeOrder`, with `isOracleFrozen()` and `isFadWindow()` asserted false at the open), and an address that holds no role and no margin or position of its own then calls `OrderRouter.executeLiquidation` on it inside the frozen window.

### The test file, and the assertions that carry the finding

The file extends `BasePerpTest`, the project's own perps test base, and overrides its risk parameters with the release values so no value the finding depends on is a harness default where a release value exists; the three that differ are named in the parity table below and the finding depends on none of them.

Test 1 opens the position, advances the clock into the frozen weekend, takes a state snapshot, and runs both exits from it. On each branch it reads the trader's settlement balance, the pool's total assets and the treasury's settlement balance, and it asserts:

- the position is flat after each branch, so both are full exits of the same position;
- the stored mark the engine reports before the branch point equals the mark each branch priced against, with the elected branch's 2.5 bps adverse shift accounted for explicitly rather than absorbed into the difference;
- the declared branch reports a settled frozen-close spread equal to the figure the planner computed, and no entry anywhere in the elected branch's event log reports a settled frozen-close spread, which is asserted over the whole log array rather than at an expected position;
- the trader keeps strictly more on the elected branch, the pool receives strictly less by a figure asserted as a rate on notional, the declared branch moves exactly the execution fee to the treasury, and the elected branch does not move the treasury's balance at all;
- the credited caller's share, taken from the engine's own `PositionLiquidated` event rather than predicted, equals the share the planner computed, and the holder's balance delta on the branch they call themselves exceeds the delta on the branch an unrelated address calls by exactly that share, which isolates the credit instead of inferring it from the net;
- the trader withdraws the settlement balance through `MarginClearinghouse.withdrawMargin` on both branches, with the mock settlement token balance received asserted on each and strictly more of it taken out on the elected branch, so the saving is money that has left the protocol rather than a number in an accounting field;
- the trader's gain between the branches equals the pool's loss plus the treasury's loss, atom for atom, and equals the two charges less the VPI returned, the net liquidation charge and the retained shift;
- and the gain between the branches, expressed in basis points of notional, exceeds a floor well below the derived value and cannot exceed what the declared route assessed, so the test stays readable and still fails if the asymmetry is ever closed.

Test 2 runs the control first, while the market is open: the planner reports the position not liquidatable, and `OrderRouter.executeLiquidation` on it reverts `CfdEngine__PositionIsSolvent()`. It then advances into the window, asserts that the planner now reports the same position liquidatable, and asserts three figures at the moment of the elected exit, all computed from the risk parameters the engine itself returns rather than from literals: that equity is strictly above the initial-margin requirement the open was held to, strictly above the base maintenance requirement at that same moment, and at or below the frozen-market requirement. The position the open market refused to touch is then liquidated inside the window.

Test 3 takes the price the elected exit executed at, differences it against the engine's stored mark, and asserts the result in hundredths of a basis point of that mark, which is the precision 2.50 bps needs, with `adverseConfidenceMultiplierBps` set to the release `2_500` rather than the oracle's field initializer. It also asserts the ratio between the spread and the shift the design says replaces it. Test 4 drives the calendar alone, with no actor and no position. Test 5 is the third-party call, checked through the project's own liquidation parity helpers.

Config parity, every constant the finding depends on:

| Constant | Harness | Release | Does the finding depend on it? |
| --- | --- | --- | --- |
| `frozenCloseSpreadBps` | `50` | `50` | yes, it is the charge avoided |
| `maintMarginBps` | `10` | `10` | yes, it is the bar the window replaces |
| `initMarginBps` | `20` | `20` | yes, it is the bar the position cleared at open |
| `fadMarginBps` | `300` | `300` | yes, it is the bar the window imposes |
| `executionFeeBps` | `4` | `4` | yes, it is the second charge avoided |
| `bountyBps` | `10` | `10` | yes, it sizes what the elected route does charge |
| `keeperShareBps` | `5_000` | `5_000` | yes, it sizes what returns to the caller |
| `protocolShareBps` | `0` | `0` | yes, it is why the treasury receives nothing |
| `minBountyAssets` | `1e6` | `1e6` | no, the rate charge exceeds it at this position's notional and equals it at the release minimum open notional |
| `basketMaxConfidenceRatioBps` | `10` | `10` | yes, it is half of the substitute's size |
| `adverseConfidenceMultiplierBps` | `2_500` | `2_500` | yes, it is the other half |
| `maxSkewRatio` | `0.4e18` | `0.4e18` | no, it bounds per-call size and not the rate |
| `vpiFactor` | `0.01e18` | `0.01e18` | no. Lifetime VPI is identical at the branch point. The voluntary close also returns the 138,580 atoms (0.28 bps) paid at open, netted against its charge, and the liquidation does not return it, so the measured gap of 46.2 bps is slightly below the 46.5 bps of charges alone; this runs against the finding |
| `minOpenNotionalAssets` | `100e6` | `1_000e6` | no, the position is 5,000 USDG of notional and clears both |
| `maxSeniorExposureAssets` | `type(uint256).max - 1` | `40_000_000e6` | no. The project's own test base raises both senior caps so that funding a tranche is not itself a test of the caps; the position here is 5,000 USDG of notional against 700,000 USDG of junior depth and 200,000 USDG of senior depth |
| `maxSeniorShareBps` | `9999` | `8_000` | no, for the same reason |

### Attack steps by hand

Given the engine at the release parameters, and an account the attacker controls:

1. **On a weekday, deposit margin and open a position above roughly 33x leverage.** `MarginClearinghouse.depositMargin`, then `OrderRouter.commitOrder` and `OrderRouter.executeOrder`. The open must happen while the FX market is open, because the frozen and FAD regimes are close-only for opens (`libraries/OracleFreshnessPolicyLib.sol:59-62`, enforced at `router/OrderLiquidationBatchLogic.sol:332-334`). Nothing else about the open is unusual, and the open itself sizes and funds the liquidation reserve the next step spends.
2. **Wait for the Friday 17:00 New York boundary.** No transaction. From that moment `isOracleFrozen()` and `isFadWindow()` both read true, and the liquidation-eligibility bar is the 3% frozen-market margin for the next 48 hours.
3. **Decide whether to exit.** The stored mark is pinned, so both routes are priced off a number that cannot move, and the cost of each is known in advance.
4. **To exit, call `OrderRouter.executeLiquidation(ownAccount, "")` from the account itself.** No Pyth payload and no ETH are needed while the oracle is frozen. The position closes at the stored mark with a 2.5 bps adverse shift, the liquidation charge is 10 bps of notional, and half of it is credited straight back to the caller's settlement balance.
5. **Withdraw.** `MarginClearinghouse.withdrawMargin`. The frozen-close spread and the execution fee were never assessed, so about 46 bps of notional that the declared route would have moved to the pool and the treasury stays with the caller.
6. **Repeat next weekend.** The window is recomputed from `block.timestamp` on every read.

The same call at step 4, made by anyone else against a position in the band, force-closes it and credits that caller instead.

### The counterfactual: the fix patch

`frozen_window_liquidation_spread.patch`, against `src/libraries/CfdEnginePlanLib.sol`, is in the Gist, and the runner re-runs the five tests under it. It makes one change: inside `_planLiquidationActionSettlement`, where the liquidation action charge is assessed at `:1562`, it builds the position's risk state a second time at `snap.riskParams.maintMarginBps`, and where that base-bar state is not liquidatable while `snap.oracleFrozen` is true, it adds `notional * snap.frozenCloseSpreadBps / 10_000` to `delta.actionChargeAssessedAssets`. Both inputs are already on the snapshot the liquidation plan receives (Bug Description, the recommended fix); the private helper that assesses the charge takes the liquidation price from the frame above it, which is its only signature change, and no struct, no external interface and no storage layout moves.

With it applied, test 1 fails at the first assertion the change flips, the one on what the holder withdraws: the elected route then costs the holder 57.5 bps against the declared route's 53.7. The reconciliation of the gap, the pool comparison, the 40 bps floor under the pool's shortfall, the trader comparison and the 40 bps floor under the gap are flipped by the same change. Not flipped, and not meant to be: the treasury ones, because the added charge reaches the pool as LP revenue rather than the treasury; the spread-event ones, because the charge is added to the liquidation action charge and the close path's spread accounting is untouched; the conservation identity, because it is a property of the accounting rather than of the asymmetry; and the two upper bounds, on the pool's shortfall and on the gap, because a negative rate satisfies both. Tests 2, 3, 4 and 5 still pass, because none of them turns on the charge: the eligibility band, the oracle shift, the calendar and the third party's ability to force the close are all unchanged by the patch.

The patch applies with `patch -p1` and with `git apply`, and ends with a `# End of patch` line.

### What is measured, and what is argued

| Claim | How it is established | Result |
| --- | --- | --- |
| For the same full exit of the same position at the same stored mark, the elected route assesses no frozen-close spread and no execution fee while the declared route assesses 50 bps plus 4 bps | Measured, test 1, from one shared state: the declared branch's settled spread read out of the engine's own event and the whole of the elected branch's event log asserted free of one, the treasury's balance asserted to move by exactly the execution fee on the declared branch and not at all on the elected one, and the pool's shortfall asserted as a rate on notional | passed: 25,000,000 atoms of spread and 2,000,000 of fee on the declared branch, none on the elected one; pool shortfall 42 bps |
| A fully solvent position is liquidation-eligible inside the frozen window and is not eligible outside it | Measured, test 2, on the planner's own eligibility flag for the same position outside and inside the window, on the revert outside it and the executed liquidation inside it, and on the three equity-against-requirement figures at the moment of the exit. That the eligibility holds for the whole span rests on the `oracleFrozen` implies `isFadWindow` row below rather than on this measurement | passed: equity 91,609,001 atoms against 10,000,000 at open, 5,000,000 base and 150,000,000 frozen-market |
| The caller can be the position holder, and half of the liquidation charge is credited to the caller | Measured, test 1, three ways on the same shared state: the share the engine reports in its own event, the share the planner computed, and the difference between the holder's balance delta when the holder makes the call and when an unrelated address makes it | passed: 2,500,000 atoms credited |
| Both branches price off the same stored mark, and the only price difference between them is the 2.5 bps adverse shift the elected route retains | Measured, test 1, on the mark the engine reports before the branch point, on the price each branch executed at, and on the shift asserted as the whole difference between those two prices | passed: 100,000,000 declared, 100,025,000 elected |
| The adverse-confidence shift the design names as liquidation's substitute for the 50 bps spread is 2.5 bps of the mark | Measured, test 3, at the release `adverseConfidenceMultiplierBps = 2_500` | passed: 250 hundredths of a basis point |
| The trader's gain between the branches equals the pool's loss plus the treasury's loss, atom for atom | Measured, test 1, as a single equality on the difference | passed: 23,111,420 = 21,111,420 + 2,000,000, and = 27,000,000 - 138,580 - 2,500,000 - 1,250,000 |
| The caller's own withdrawable balance moves, and the pool's and the treasury's do too | Measured, test 1, by withdrawing through `MarginClearinghouse.withdrawMargin` on both branches, asserting the settlement token received on each and strictly more of it on the elected branch, and by reading the pool's total assets and the treasury's balance on both branches | passed: 170,997,581 atoms withdrawn declared, 194,109,001 elected |
| The window recurs weekly on a calendar the contract computes from `block.timestamp` | Measured, test 4, on six timestamps with no actor involved | passed: all six predicate pairs as predicted |
| The precondition needs no privileged action and no object somebody else had to create | Measured, test 1, which creates the position through `commitOrder` and `executeOrder` with both calendar predicates asserted false at the open | passed |
| An unrelated caller can force-close a solvent position and be credited the keeper share of the liquidation charge | Measured, test 5, for the force-close and the credit; that the share is funded by debiting the holder's own liquidation reserve is source (`MarginClearinghouse.sol:1236-1240`) | passed: 2,500,000 atoms, 5 bps of notional |
| `frozenCloseSpreadBps` is consumed for computation at one site in `packages/perps/src`, and the liquidation plan cannot reach it | Source: a use census over the package, twenty-one matching lines, one computational, plus a caller census on `_buildCloseState` | source |
| No guard conditions the bounty credit on the caller's identity | Source: a census over all of `packages/perps/src` for a keeper comparison returns three sites, each read, and none conditions the bounty credit on the caller's identity | source |
| `oracleFrozen` implies `isFadWindow` on every calendar branch | Source: all four branches of `marketStatus` plus the override runway, with both shoulder constants nonzero. Test 4 measures the predicate pairs that implication predicts | source, with the predicted pairs measured |
| The spread is LP-owned trading revenue rather than treasury revenue | Source: the project's own definitions, plus the pool-side credit path. Test 1 measures the pool's own total assets on both branches, which is the realization | source, with the realization measured |
| The per-call notional is bounded by the engine's open-side skew limit | Source: `libraries/CfdEnginePlanLib.sol:473` | source |
| The rationale the documentation gives for the liquidation exemption is general, and none of the sixteen Markdown documents under `packages/perps` discusses a solvent account electing the exempt route | Source: `SECURITY.md:845`, the summary line of the bullet list the exemption closes, with the collectibility lines read and shown to govern the voluntary route instead (Bug Description), plus a census over the sixteen Markdown files under `packages/perps` | source |

### What this proof of concept does not establish

1. **The absolute size of a single call in USDG.** The test measures the rate, on one position, and the rate does not vary with size. The largest position the engine will let one account hold directionally is a function of pool depth through the open-side skew limit, and pool depth is a deployment property rather than a code property.
2. **The value of the decision window itself.** A holder inside the frozen window is choosing between two exits at a mark that cannot move, which explains why they would exercise. It adds nothing to the figure: the declared route exits at the same stored mark (`PletherOracle.sol:937-939`), so the pool bears the identical gap whichever route is taken, and the whole realized effect of the fix is the charge difference.
3. **That a position in the eligibility band exists in the deployment today.** The first three tests do not need one, because the caller opens the position themselves, and the fourth reads the calendar with no position at all. The fifth test does, and the band is set by the parameters: inside the window a position is eligible once its equity is at or below 3% of notional (`fadMarginBps` 300 at `packages/perps/DEPLOYMENT.md:136`, selected at `libraries/CfdEnginePlanLib.sol:1450` and tested at `libraries/PositionRiskAccountingLib.sol:144`), which is any position held above roughly 33x, against a release initial margin of 20 bps (`DEPLOYMENT.md:135`).
4. **The deployed values of the parameters.** The test runs at the values checked in at the scoped commit, which is the configuration the programme states severity is assessed at.

### The captures

Both blocks below are contiguous runs of the one file the runner writes, `CAPTURE.log`, which is in the Gist beside the test file. The `spread waived atoms 138580` the declared route logs is 0.28 bps of the notional and is not a part of the spread that went uncollected: that branch's own figures close exactly, the holder's 26,863,839 atoms of debit against the pool's 24,863,839 and the treasury's 2,000,000. `CfdEngineSettlementSidecar.sol:724-747` credits the charge a close collects against the spread only after the execution fee and the carry, and counts the position's VPI among those prior charges only where its delta is positive (`:729-731`), so a VPI the close returns to the holder reappears in the residue the event reports as waived at `:657-663`. Test 1 asserts that the figure equals, to the atom, the lifetime VPI the position paid at open and the VPI the close returned. The first is the isolation self-check, the toolchain and target block, and the five tests. The second is the second half of the same file: the same five tests with the suggested change applied, and the hash of the target source after the reversal, which pairs with the hash in the first block.

```text
=== isolation self-check, from inside the namespace ===
started at 2026-11-08T23:13:04Z (UTC)
network interfaces: lo 
IPv4 route table entries: 0, of which off loopback: 0
IPv6 route table entries: 2, of which off loopback: 0
connect probe on an unbound loopback port: failed, as expected
isolation self-check passed: the only interface is loopback and there is no route off it

=== toolchain and target ===
forge Version: 1.6.0-v1.7.0
test file: packages/perps/test/perps/FrozenWindowExitCharge.t.sol
target source: packages/perps/src/libraries/CfdEnginePlanLib.sol
sha256 of the target source before the run: 1962d7a3df8e23ca9b4059e45f2496b6bbe1a3e56f722b09e5b8541be9f7ae5f
compiler and lint diagnostics are on standard output; CAPTURE.log is filtered through:
  grep -v -e '^Warning (' -e '^Note:' -e '^note:' -e '^[[:space:]]*|' -e '^[0-9][0-9]*[[:space:]]*|' -e '^[[:space:]]*-->' -e '^[[:space:]]*\^'

=== compiling, and listing the tests in the file ===
compilation succeeded; the list of tests in the file is on standard output

=== the five tests, on the repository as it stands ===
No files changed, compilation skipped

Ran 5 tests for test/perps/FrozenWindowExitCharge.t.sol:FrozenWindowExitChargeTest
[PASS] test_frozenLiquidationRetainsTwoAndAHalfBasisPointsOfShift() (gas: 3261060)
Logs:
  synthetic frozen confidence 100000
  retained shift, price units 25000
  retained shift, hundredths of a bp of the mark 250
  frozen-close spread, hundredths of a bp of notional 5000

[PASS] test_frozenWindowOffersTwoExitsChargedTensOfBasisPointsApart() (gas: 5527824)
Logs:
  notional atoms 5000000000
  stored mark 100000000
  declared route: frozen spread atoms 25000000
  declared route: execution fee atoms 2000000
  elected route: liquidation charge atoms 5000000
  elected route: caller's share atoms 2500000
  elected route: treasury share atoms 0
  position: lifetime VPI paid at open, atoms 138580
  declared route: VPI returned at close, atoms 138580
  declared voluntary close
    holder delta atoms -26863839
    holder delta bps of notional -53
    pool delta atoms 24863839
    pool delta bps of notional 49
    treasury delta atoms 2000000
  declared route: spread assessed atoms 25000000
  declared route: spread paid to LPs atoms 24861420
  declared route: spread waived atoms 138580
  settlement token withdrawn after the declared exit 170997581
  still locked after the declared exit 0
  elected liquidation, caller is the holder
    holder delta atoms -3752419
    holder delta bps of notional -7
    pool delta atoms 3752419
    pool delta bps of notional 7
    treasury delta atoms 0
  declared route execution price 100000000
  elected route execution price 100025000
  settlement token withdrawn after the elected exit 194109001
  still locked after the elected exit 0
  settlement token difference between the two routes 23111420
  credit to an unrelated caller atoms 2500000
  holder keeps more on the elected route, atoms 23111420
  holder keeps more on the elected route, bps of notional 46
  pool receives less on the elected route, atoms 21111420
  pool receives less on the elected route, bps of notional 42
  treasury receives less on the elected route, atoms 2000000

[PASS] test_frozenWindowRecursWeeklyOnTheContractsOwnCalendar() (gas: 136181)
Logs:
  New York boundary, second of day 75600
  a weekday 1710331200
    frozen-market window false
    oracle frozen false
  before the Friday shoulder 1710534000
    frozen-market window false
    oracle frozen false
  inside the Friday shoulder 1710535800
    frozen-market window true
    oracle frozen false
  after the Friday boundary 1710536700
    frozen-market window true
    oracle frozen true
  mid-weekend 1710590400
    frozen-market window true
    oracle frozen true
  the same hour seven days later 1711195200
    frozen-market window true
    oracle frozen true

[PASS] test_solventPositionIsLiquidatableOnlyInsideTheFrozenWindow() (gas: 3648745)
Logs:
  equity at the moment of the elected exit, atoms 91609001
  equity, bps of notional 183
  open-time requirement atoms 10000000
  base maintenance requirement atoms 5000000
  frozen-market requirement atoms 150000000

[PASS] test_unrelatedCallerForcesTheSolventPositionOut() (gas: 3553825)
Logs:
  credit to the unrelated caller, atoms 2500000
  credit to the unrelated caller, bps of notional 5

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 49.11ms (30.71ms CPU time)

Ran 1 test suite in 50.44ms (49.11ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)

exit status of the run above: 0
```

```text
=== the same five tests with frozen_window_liquidation_spread.patch applied ===
the change assesses the frozen-close spread on a liquidation that is eligible only because the
frozen-market window raised the liquidation bar, so the first test's comparison assertions no
longer hold and the other four are unaffected
the change applied
No files changed, compilation skipped

Ran 5 tests for test/perps/FrozenWindowExitCharge.t.sol:FrozenWindowExitChargeTest
[PASS] test_frozenLiquidationRetainsTwoAndAHalfBasisPointsOfShift() (gas: 3277964)
Logs:
  synthetic frozen confidence 100000
  retained shift, price units 25000
  retained shift, hundredths of a bp of the mark 250
  frozen-close spread, hundredths of a bp of notional 5000

[FAIL: the holder takes strictly more settlement token out through the elected route: 169102751 <= 170997581] test_frozenWindowOffersTwoExitsChargedTensOfBasisPointsApart() (gas: 6478316)
Logs:
  notional atoms 5000000000
  stored mark 100000000
  declared route: frozen spread atoms 25000000
  declared route: execution fee atoms 2000000
  elected route: liquidation charge atoms 5000000
  elected route: caller's share atoms 2500000
  elected route: treasury share atoms 0
  position: lifetime VPI paid at open, atoms 138580
  declared route: VPI returned at close, atoms 138580
  declared voluntary close
    holder delta atoms -26863839
    holder delta bps of notional -53
    pool delta atoms 24863839
    pool delta bps of notional 49
    treasury delta atoms 2000000
  declared route: spread assessed atoms 25000000
  declared route: spread paid to LPs atoms 24861420
  declared route: spread waived atoms 138580
  settlement token withdrawn after the declared exit 170997581
  still locked after the declared exit 0
  elected liquidation, caller is the holder
    holder delta atoms -28758669
    holder delta bps of notional -57
    pool delta atoms 28758669
    pool delta bps of notional 57
    treasury delta atoms 0
  declared route execution price 100000000
  elected route execution price 100025000
  settlement token withdrawn after the elected exit 169102751
  still locked after the elected exit 0

[PASS] test_frozenWindowRecursWeeklyOnTheContractsOwnCalendar() (gas: 136181)
Logs:
  New York boundary, second of day 75600
  a weekday 1710331200
    frozen-market window false
    oracle frozen false
  before the Friday shoulder 1710534000
    frozen-market window false
    oracle frozen false
  inside the Friday shoulder 1710535800
    frozen-market window true
    oracle frozen false
  after the Friday boundary 1710536700
    frozen-market window true
    oracle frozen true
  mid-weekend 1710590400
    frozen-market window true
    oracle frozen true
  the same hour seven days later 1711195200
    frozen-market window true
    oracle frozen true

[PASS] test_solventPositionIsLiquidatableOnlyInsideTheFrozenWindow() (gas: 3662815)
Logs:
  equity at the moment of the elected exit, atoms 91609001
  equity, bps of notional 183
  open-time requirement atoms 10000000
  base maintenance requirement atoms 5000000
  frozen-market requirement atoms 150000000

[PASS] test_unrelatedCallerForcesTheSolventPositionOut() (gas: 3567887)
Logs:
  credit to the unrelated caller, atoms 2500000
  credit to the unrelated caller, bps of notional 5

Suite result: FAILED. 4 passed; 1 failed; 0 skipped; finished in 61.68ms (46.11ms CPU time)

Ran 1 test suite in 63.08ms (61.68ms CPU time): 4 tests passed, 1 failed, 0 skipped (5 total tests)

Failing tests:
Encountered 1 failing test in test/perps/FrozenWindowExitCharge.t.sol:FrozenWindowExitChargeTest
[FAIL: the holder takes strictly more settlement token out through the elected route: 169102751 <= 170997581] test_frozenWindowOffersTwoExitsChargedTensOfBasisPointsApart() (gas: 6478316)

Encountered a total of 1 failing tests, 4 tests succeeded

Tip: Run `forge test --rerun` to retry only the 1 failed test

exit status of the run above: 1 (a failing first test is the expected result here)
the change was reversed
sha256 of the target source after the run: 1962d7a3df8e23ca9b4059e45f2496b6bbe1a3e56f722b09e5b8541be9f7ae5f

finished at 2026-11-08T23:15:44Z (UTC)
no endpoint outside the namespace was contacted; the only connection attempted was the self-check's probe of an unbound loopback port; nothing was signed or broadcast; every transaction above ran in the local test EVM
```

Reproduction, with the test file, the runner and the patch from the Gist and a checkout of the repository at the scoped commit:

```bash
cp FrozenWindowExitCharge.t.sol <repo>/packages/perps/test/perps/
./run_local_only.sh <repo>
```

The runner reports the toolchain version it used in the capture's own header. `RUN.md`, also in the Gist, carries the prerequisites, the configuration parity table, the harness inventory and the direct `forge test` invocation for anyone who would rather skip the runner.
