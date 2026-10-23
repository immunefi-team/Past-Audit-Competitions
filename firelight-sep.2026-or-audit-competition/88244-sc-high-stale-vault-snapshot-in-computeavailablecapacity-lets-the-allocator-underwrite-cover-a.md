# 88244 sc high stale vault snapshot in computeavailablecapacity lets the allocator underwrite cover against collateral an incident payout already spent breaching the enforced solvency floor pro&#x20;

**Submitted on Aug 12th 2026 at 11:12:53 UTC by @Gyuu for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88244
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator._computeAvailableCapacity` (`:696-725`) computes the period's underwriting ceiling by adding two collateral legs that have **different temporal semantics**: the first-loss buffer is read as a **live balance** (`:705`), while the vault leg is read as a **period-start snapshot** (`:711`).

`FirelightVault.payout` transfers assets out of the vault (`:962`) and records the reduction by pushing a checkpoint at `Time.timestamp()` (`:952`) - a key strictly **greater** than `currentPeriodStart()` (`:342-344`). Because `totalAssetsAt` resolves to `_traceTotalAssets.upperLookupRecent(ts)` (`:456-457`), which returns the last checkpoint **at or before** the requested key, the payout is invisible at `:711` for the remainder of the period.

For that entire window the allocator commits and settles cover against vault assets that have already left the contract. In the PoC, a payout removing 90% of the vault leaves the underwriting ceiling completely unchanged, and $20,000 of cover is then sold and settled against $1,000 of remaining collateral - a 20x over-subscription, driving real backing to 5% against a floor of 18.18% that the contract itself enforces and documents as unbreachable.

No attacker is required. Every actor stays inside its documented permissions.

**Assets:** `contracts/core/CoverOrderAllocator.sol` (primary), `contracts/core/FirelightVault.sol`, `contracts/core/IncidentManager.sol`. **Commit:** `immunefi-team/audit-comp-firelight`, branch `v1_audit_ready`, head `42f9ea5e43d88b35197b901acc2a8c24b314fa62`. **Program priority area:** "Capacity calculation and solvency constraints".

## Vulnerability Details

### The two collateral legs

```solidity
// contracts/core/CoverOrderAllocator.sol:704-717
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer),  // :705  LIVE balance
    ...
);
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),            // :711  PERIOD-START SNAPSHOT
    ...
);
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical; // :717
```

`IncidentManager._executePayout` drains the first-loss buffer **first** (`IncidentManager.sol:645-658`) and reaches the vault only for the remainder (`:684`). A payout therefore reduces capacity by exactly its first-loss portion and by **zero** of its vault portion - and the vault portion carries the large residual loss by construction of the waterfall.

The error is widened by checkpoint sparsity. `_traceTotalAssets` is written only on deposit/withdraw flows (`_logTrace:1055`), on `payout` (`:952`), and via `checkpointTotalAssets` (`:523`). Nothing guarantees a checkpoint exists at a period boundary; the team documents this gap itself at `FirelightVault.sol:517-521` ("historical `totalAssetsAt` lookups would miss them until the next flow"). The value read at `:711` can therefore be materially older than `currentPeriodStart()`.

### A period-frozen ceiling is not the documented design

The contract states its intent three times, and each statement is incompatible with a frozen vault term.

Header NatSpec (`CoverOrderAllocator.sol:32-35`):

> "Capacity is **recomputed** on-chain at commit/recommit from the first-loss buffer balance plus oracle-priced staked assets ... widened by the per-period `divergenceToleranceBps` to absorb **price/buffer drift** between the off-chain snapshot and commit inclusion."

The stated intent is recomputation from current inputs, not a frozen period-start ceiling. And the drift sources the team enumerated are **price and buffer** - vault-asset drift is absent from the list, because the staked-assets leg was believed to be current.

Immediately above the recommit validation (`:375-377`):

> "Validate against **fresh capacity** (symmetric with `commitAllocation`) so recommit can capture recovered **price/FLB** and **stays bound to the period's real collateral** within tolerance."

"Stays bound to the period's real collateral" is precisely what `:711` does not achieve. `_replaceableCommitment` repeats the assumption a third time: "live capacity and period-start reads only correspond to it" (`:422`).

Control 3 of the PoC is the empirical form of the same point: the same $5,000 removed from the buffer leg lowers capacity immediately, and removed from the vault leg lowers it not at all.

### The protocol records the impairment; the allocator is never told

When an incident becomes active, `IncidentManager._increaseIncident` calls `vault.setActiveIncident(period, true)` (`:813`, also `:829`) and the vault immediately stops accepting capital: `maxDeposit` returns 0 (`FirelightVault.sol:373-374`), `maxMint` returns 0 (`:388-389`), and `deposit`/`mint` revert `CurrentPeriodHasActiveIncident` (`:660`, `:696`).

`CoverOrderAllocator.sol` contains **zero** references to `hasActiveIncident`. `commitAllocation` (`:332`), `settleCoverOrder` (`:446`) and `batchSettleCoverOrder` (`:451`) are incident-unaware, and `IncidentManager` never writes to the allocator - it only reads views from it (`:348-350`, `:636`).

One gate closes on the inflow of collateral while the other continues selling claims against that same collateral, in the same block and the same protocol state.

### No path exists to correct the ceiling mid-period

| Candidate correction                         | Why it fails                                                                                                                                       | Reference      |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `recommitAllocation` re-validates capacity   | routes to `_validateMatchingCapacity:400` to the same stale `:711`; refreshes only price (`:700`) and first-loss balance (`:705`)                  | `:378`, `:400` |
| ...and is unavailable once settlement starts | `_replaceableCommitment` reverts `SettlementsAlreadyStarted` when `commit.totalSettledCover > 0`                                                   | `:434`         |
| `cancelCommitAllocation` withdraws the root  | same `totalSettledCover == 0` precondition, via the same helper                                                                                    | `:411`, `:434` |
| Re-commit at a corrected capacity            | `commitAllocation` reverts `PeriodAlreadyCommitted` once a root exists                                                                             | `:344`         |
| Incident flag halts underwriting             | zero references to `hasActiveIncident` in the allocator                                                                                            | -              |
| `IncidentManager` notifies the allocator     | no such write exists; it only reads views (`:348-350`, `:636`)                                                                                     | -              |
| Settlement re-checks capacity                | it does not: concentration caps are computed against the frozen `commit.totalAvailableCapacity` written at commit time (`:352`) and read at `:507` | `:352`, `:507` |

No role - however privileged, diligent or well-informed - has a function to call that lowers a period's capacity after a payout. Only the next period boundary clears it.

### Role topology

`PAYOUT_ROLE` and the allocator's commit/settle role are distinct roles on distinct contracts, granted at distinct times:

|                 | `PAYOUT_ROLE`                                                                                                                                                 | `ALLOCATOR_ROLE`                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Declared        | `FirelightVaultStorage.sol:31`                                                                                                                                | `CoverOrderAllocator.sol:63`                                                |
| Enforced        | `FirelightVault.payout:896`                                                                                                                                   | `commitAllocation:337`, `settleCoverOrder:446`, `batchSettleCoverOrder:451` |
| Granted         | Not in `initialize`; the vault's `InitParams` (`FirelightVault.sol:55-64`) has no payout field. Granted post-deployment by `DEFAULT_ADMIN_ROLE`               | `initialize` grants `ALLOCATOR_ROLE` to `params.allocatorRole` (`:182`)     |
| Intended holder | the `IncidentManager` contract - `payout` is reachable in production only via `_executePayout:684`, itself reachable only from `approveCurrentAssessment:403` | the off-chain matching engine (`:46`)                                       |

Nothing forces one address to hold both, and the documented responsibilities (`IncidentManager.sol:39-47`, `CoverOrderAllocator.sol:42-48`) assign them to different functions. Per the table above this does not matter: coordination between the two roles cannot avoid the defect.

### Scope: this is a bypass of an enforced protocol constraint

The program excludes reports that require "a privileged role holder to act within the permissions explicitly granted to that role ... unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint."

This report makes **no allegation about any role's conduct**. Every actor in the PoC stays inside its documented permissions. What is demonstrated is the second limb: an intended protocol constraint, computed and enforced on-chain, is bypassed.

The allocator caps period underwriting at `_computeAvailableCapacity` and reverts `MatchingCapacityOverflow` above it (`:401`). The ceiling is `collateral x effectiveLeverage / minCAR`, widened by `divergenceToleranceBps` (`:719-724`). Governance cannot set the inputs freely: `_setCapacityConfig` pins `minCAR` to `[12000, 50000]` (`:662`, `:84-85`), caps tolerance at 10% (`:668`, `:78`), and bounds leverage relative to `minCAR` - `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR`, with `MAX_LEVERAGE_FACTOR = 5` (`:665-666`, `:82`).

The contract states in its own comment what that bound guarantees (`:79-81`):

> "Real backing behind sold cover is `minCAR / (effectiveLeverage * (1 + tolerance))`, so bounding the ratio floors it at `1 / (MAX_LEVERAGE_FACTOR * (1 + tolerance))` **regardless of the configured `minCAR`**."

That floor is `1 / (5 * 1.10)` = **18.18% real backing**, and the code asserts no configuration can go below it. The PoC reaches **5%** - 3.6x below the floor - using a capacity configuration that `_setCapacityConfig` accepts unmodified, with no parameter touched and no role exceeding its permissions. The ceiling is not raised; the *collateral input* to the formula is stale, so the constraint is evaluated against assets the protocol no longer holds and is satisfied by a value that is not true.

The bound at `:665-666` is Firelight's remediation of Macro **M-1** (see References): the fix is live and untouched, but it is bypassable from the other side of the formula. And because the ceiling is enforced in code rather than delegated to operator judgement, this is not one of the permissioned-MVP behaviours the program declares out of scope.

## Impact Details

The damage has two components: one realised the moment the defect fires, one an obligation the protocol takes on and cannot cover.

### (a) Realised loss - premiums charged for cover that is already unbacked

Settlement takes the buyer's money in the same transaction that issues the cover. `_settleCoverOrder` executes `IERC20(order.premiumToken).safeTransferFrom(order.buyer, $.premiumCollector, finalPremiumNative)` (`:535`) immediately before minting the cover NFT (`:537`). The buyer pays at the instant the cover is issued, and at that instant the collateral backing it is already gone.

The payment is final:

* The premium goes to `$.premiumCollector` (`:535`), an external address set by `ADMIN_ROLE` (`setPremiumCollector:580`). It is not vault collateral and does not back the cover it paid for.
* There is no refund path in scope. Both cancellation entrypoints require `status == PENDING` (`cancelCoverOrder:551`, `cancelExpiredOrders:566`) and settlement has already moved the order to `MATCHED`/`PARTIAL` (`:519`). The string `refund` does not appear in any file under `contracts/core/`.

In the PoC, **19.178083 USDC** of premium is charged for $20,000 of cover backed by $1,000 of collateral. The loss to cover buyers is immediate, certain and non-recoverable.

### (b) Enforceable liability - the maximum owed rises 1:1, with no collateral gate

The over-sold allocation is an obligation, not a bookkeeping artefact. `IncidentManager.addAssessmentLosses` (`:291`) validates each assessed loss **only** against the allocated cover amount - `if (amount > allocatedCoverAmount) revert AssessmentLossAmountTooBig` (`:352-353`), reading the allocation from the allocator at `:348-350`. There is no check against collateral anywhere in that path. Every dollar of over-subscription raises the maximum assessable loss by one dollar.

On the paying side, `FirelightVault.payout` caps at `Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount))` (`:927`). The shortfall does not revert or queue - it lands on the insured as a silently under-paid claim.

**Parties harmed.** Cover buyers lose the premium immediately (a) and receive partial settlement on any subsequent claim (b). Stakers are levered past the solvency floor `minCAR` exists to enforce (`:662`).

### Quantification

Parameters are identical to the team's own fixture (`test/setup/fixtures.js:164-171`), declared in the PoC at `test/poc/hx4_capacity_after_payout.js:81-88`.

| Measure                                             | Value                                                                                                 |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Governance-permitted cover : collateral             | 2.0x (`effectiveLeverage / minCAR` = 24000/12000)                                                     |
| Realised cover : collateral after payout            | **20.0x** ($20,000 sold vs. $1,000 held)                                                              |
| Effective collateralisation                         | **5%**, against the 50% this configuration permits                                                    |
| **Hard floor enforced for any legal configuration** | **18.18%** = `1/(MAX_LEVERAGE_FACTOR * (1 + max tolerance))` (`:79-82`, `:78`) - **breached by 3.6x** |
| Premium charged for the unbacked cover              | 19.178083 USDC, non-refundable, paid to a non-vault collector                                         |

The final row determines scope. The 50% figure reflects a configuration choice. The 18.18% figure is not a choice - it is the floor `_setCapacityConfig` imposes on every configuration it accepts (`:665-666`) and the invariant the code documents (`:79-81`). No setting of `minCAR`, `effectiveLeverage` or `divergenceToleranceBps` produces 5% real backing through the intended path.

**General magnitude.** If a payout removes fraction `f` of period-start vault assets and the first-loss buffer is exhausted, realised leverage becomes `L / (1 - f)`, where `L` is the configured multiplier: `f = 0.5` gives 2x overshoot, `f = 0.9` gives 10x overshoot (the PoC). The window runs from the payout to the end of the period, up to one full `duration` (7 days in the fixture, `fixtures.js:105`). It does not compound across periods - the next period re-snapshots correctly (Control 2) - so the defect is bounded per period and repeatable across periods.

### Preconditions

1. An incident is confirmed and its payout draws on the vault, i.e. exceeds the first-loss buffer (`IncidentManager._executePayout:645-684`). Incidents are exogenous events.
2. Underwriting continues in the same period after the payout. This holds under either ordinary ordering: the period's `commitAllocation` lands after the payout, or it landed before and settlement simply continues - `settleCoverOrder` (`:446`) has no incident guard and validates against the frozen `commit.totalAvailableCapacity` (`:352` to `:507`).
3. *(Consequential damage only.)* A further claim against that period's cover exceeds the vault's remaining assets.

Conditions 1 and 2 are sufficient for the realised loss in (a). Condition 3 governs only the magnitude of (b).

### Severity

The program lists **"Protocol insolvency"** as a Critical Smart Contract impact. That is the matching impact: settled, enforceable cover obligations (b) exceed the assets backing them (vault plus buffer) by 20x in the demonstrated case, and buyer funds are transferred for that cover (a).

Stated plainly: there is no attacker and no direct theft. The protocol harms its own users by selling a product it cannot back, and the trigger is an exogenous incident rather than an attacker-chosen moment. If triage weighs that precondition heavily, **High** is the conservative landing. Two discounts do not apply on the evidence:

* *Contingent loss.* (a) is a realised, non-refundable transfer of buyer funds (`:535`) in the same transaction that mints cover already unbacked, with no refund mechanism in scope.
* *Trusted role.* Every actor remains inside documented permissions and reads the only capacity figure the protocol publishes. `_computeAvailableCapacity` is referenced exactly twice in the codebase - the call at `:400` and a comment at `:407` - and has no view export: the exported views are `getEffectiveCapacityConfig` (`:940`) and `getCapacityConfigAt` (`:944`), i.e. the *parameters*, not the resulting ceiling. The sole externally observable capacity figure is the one embedded in the `MatchingCapacityOverflow(requested, maxCapacity)` revert at `:401`, which returns the stale value. An off-chain matcher querying the protocol for its ceiling receives that value, commits within it, and the contract accepts. No role has a function available to correct it.

## Why the existing test suite does not observe this

The allocator suite and the vault suite are disjoint, and the defect lives on the boundary between them.

* All eight allocator suites in `test/coverOrderAllocator/` run against a **mock** vault: seven via `deployCoverOrderAllocator` (`test/setup/fixtures.js:106`), and `testVectors.js` by deploying `MockCoverOrderAllocatorVault` directly (`testVectors.js:111-112`). The fixture states the reason: "Uses a mock vault (instead of the full FirelightVault) so tests are self-contained and can directly set currentPeriod / totalAssets / period duration" (`fixtures.js:102-103`).
* `contracts/test/MockCoverOrderAllocatorVault.sol` implements `totalAssetsAt(uint48)` so that it **ignores its timestamp argument entirely** (`:198-199`): `return _hasSnapshot ? _totalAssetsAtSnapshot : _totalAssets;`. The time-sensitivity of `upperLookupRecent` - the whole mechanism of this bug - is structurally unobservable there. The same mock hard-codes `hasActiveIncident` to `false` (`:221`).
* Nine vault suites use `deployVault`; none constructs an allocator.
* The `IncidentManager` suite mocks both counterparties at once (`MockIncidentManagerVault`, `MockIncidentManagerCoverOrderAllocator` - `test/incidentManager/fixtures.js:72-73`, `:84-85`), so the payout/capacity interaction has no home there either.

No file in the repository instantiates the real `FirelightVault` and the real `CoverOrderAllocator` together.

## Recommended fix

Make both collateral legs agree in time. In decreasing order of preference:

1. **Read the vault leg defensively at `:711`:** `min($.vault.totalAssetsAt($.vault.currentPeriodStart()), $.vault.totalAssets())`. A payout contracts capacity immediately, while the period-start value still caps opportunistic mid-period inflows. Smallest diff, monotone-safe, no new storage, no new role. **This alone removes the demonstrated impact.**
2. **Have `FirelightVault.payout` record the reduction against the period-start key**, so the period's own snapshot reflects assets spent against it. `Checkpoints.Trace` keys must be non-decreasing, so this requires a per-period "assets consumed" accumulator subtracted at read time rather than a back-dated push.
3. **Gate `commitAllocation` / `_settleCoverOrder` on `hasActiveIncident`**, requiring a `recommitAllocation` at refreshed capacity before settlement resumes. Largest behavioural change, and it halts legitimate settlement during incidents, but it closes the notification gap directly.

## References - adjacent exclusions, and why none reaches this finding

**1. Collateral drawdown (out of scope).** The program excludes: "If the underlying staked asset decreases in USD value, available capital and the Capital Adequacy Ratio decrease. Severe collateral drawdowns can reduce coverage capacity and may cause shortfalls or undercollateralization relative to outstanding cover obligations."

*Mechanism.* This item concerns the staked asset **decreasing in USD value** - a price move. The price leg is read live at `:700` (`getPrice`) and behaves correctly; this report raises no complaint about it. In the PoC the oracle is pinned at $1.00/fXRP for the entire run and never moves: it is deployed at `ONE_E18` (`hx4_capacity_after_payout.js:72`) and every subsequent `setAnswer` re-stamps the same value (`:184`, `:261`, `:281`), solely to refresh feed freshness after the test advances past `maxPriceAge`. No price movement participates in the result. The collateral does not depreciate - it leaves the contract, transferred out by `FirelightVault.payout` (`:962`).

*Direction.* The item states that a drawdown **decreases** available capital and therefore **reduces** coverage capacity. This report demonstrates the opposite behaviour: after a 90% drawdown, coverage capacity does not move at all (asserted for equality). The exclusion describes capacity correctly shrinking; the finding is that it fails to shrink.

**2. Aggregate claims not capped (known issue).** "Aggregate claims are not capped on-chain against total cover sold. The MVP relies on trusted incident/assessment roles and the approval process to prevent duplicate, excessive, or otherwise invalid claims."

This governs **claims vs. cover sold** - the payout side exceeding what was underwritten. This report concerns **cover sold vs. collateral held** - the underwriting side exceeding what backs it. Different quantities, different contract, different code path: the known issue is `IncidentManager` lacking a per-coverage `remainingCoverage` decrement across incidents; this finding is `CoverOrderAllocator` reading a stale collateral term at `:711`. Fixing either leaves the other untouched. The stated mitigation also does not transfer: it relies on trusted roles and an approval process at claim time, and no role has any function that lowers a period's capacity after a payout. The same distinction answers the `payout` NatSpec at `FirelightVault.sol:885-886`, which allocates responsibility in the same direction.

**3. First-loss buffer items (out of scope / known issues).** The program declares the buffer to be an external, unescrowed wallet whose funding the protocol relies on, and its token to be treated as USD-pegged.

That is the **first-loss leg**, and this report does not rely on it. The buffer leg is read live (`:705`); the main test funds it with **zero** (`hx4_capacity_after_payout.js:84`) specifically to remove it from the experiment and isolate the vault term. The only test that funds it is Control 3, whose purpose is to contrast the two legs. The impaired quantity here is the **vault** leg: assets escrowed inside the protocol, moved only by a protocol-internal `payout` path, and read as a period-start snapshot at `:711`. The accepted-risk set marks the boundary precisely: the program accepts that an unescrowed external wallet may be drained without capacity reacting. It does not state that escrowed vault collateral may be spent without capacity reacting.

## References - prior-audit check: Macro *Firelight A-1* (completed 29 July 2026)

The program rules that "any unpatched or unresolved vulnerabilities disclosed in these reports are not eligible for rewards." The listed report - 0xMacro, *Firelight A-1*, v1.0.0, audit performed 26 June - 13 July 2026, commit `d0d3733` - was read in full, with specific attention to capacity computation, period snapshots, `totalAssetsAt`, the payout/capacity interaction, over-subscription and solvency.

**This finding does not appear in it.** Macro reported 18 issues (2 Medium, 4 Low, 1 Gas, 11 Informational); none concerns the temporal semantics of the collateral inputs to `_computeAvailableCapacity`. Macro's only unresolved items are **L-2** (Acknowledged - lowering a concentration cap mid-period can strand orders) and **I-1** (Acknowledged - cover NFTs are transferable but carry no on-chain rights). Neither is this bug.

**M-1** - "Unbounded `effectiveLeverage` lets real backing approach zero and nullifies the `minCAR` floor" (Medium, Fixed) - is the nearest neighbour. M-1 concerns the **multiplier** (`effectiveLeverage` had no upper bound, fixed in `_setCapacityConfig` validation, requiring `CONFIG_ADMIN_ROLE` to set an extreme parameter). This report concerns the **collateral** (the vault leg read stale at `_computeAvailableCapacity:711`, requiring nobody to act beyond an ordinary payout, and present at the in-scope commit).

Firelight fixed M-1 in the stronger relative form Macro recommended: `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR` with `MAX_LEVERAGE_FACTOR = 5` (`:665-666`, `:82`), plus a `minCAR` upper bound from I-7 (`:662`, `:84-85`). That fix is live at the in-scope commit and this PoC does not disturb it - the configuration used sits well inside every bound and `_setCapacityConfig` accepts it unmodified. The relationship is complementary rather than duplicative: Macro's remediation floors real backing at 18.18% *on the assumption that the collateral term is true*. This report shows the collateral term is not true after a payout, so the guarantee the fix was written to provide does not hold in that window. M-1 bounded the multiplier; the freshness of the quantity it multiplies remains unbounded.

**L-4** ("Code enforces a CAR floor of 1.0 while the specification requires 1.2", Low, Fixed) is likewise a configuration-side bound (`MIN_CAR_BPS = 12000`, `:84`, `:662`), correct and uncontested here.

**Buffer-drawdown trust assumption.** Macro's trust model states of the first-loss buffer wallet: "If it is funded at commitment and then drawn down, sold cover becomes underbacked." This is the closest the audit comes to the impact described here. It concerns an accepted trust assumption about an **external, unescrowed wallet** the protocol does not control, disclosed as such and mirrored in the program's known-issue list. This report concerns **escrowed vault collateral** moved by a protocol-internal payout path, where the impairment is invisible to the capacity formula for structural reasons (`upperLookupRecent` against a period-start key). Control 3 demonstrates the distinction empirically.

**Public disclosure.** Nothing in the Macro report, the program's known-issue list, the previous competition's published findings, or the OpenZeppelin review describes this defect.

## Link to Proof of Concept

<https://gist.github.com/EnriqueMarques/d7286ccddb27fe3c193054d2aa001a3d>

## Proof of Concept

## Proof of Concept

A runnable Hardhat test is provided: `test/poc/hx4_capacity_after_payout.js` (2 tests, both passing). It is the **first file in the repository to instantiate the real `FirelightVault` and the real `CoverOrderAllocator` together** - every existing allocator suite uses a mock vault whose `totalAssetsAt` ignores its timestamp argument.

### Environment

|              |                                                                                                 |
| ------------ | ----------------------------------------------------------------------------------------------- |
| Framework    | Hardhat 2.22.19, local network, no fork required                                                |
| Node         | v26.0.0                                                                                         |
| solc         | 0.8.23 / 0.8.28, optimizer on, `runs: 10000`, `evmVersion: london` (`hardhat.config.js:94-110`) |
| Dependencies | `@openzeppelin/contracts 5.2.0`, `contracts-upgradeable ~5.4.0`, `merkle-tree ^1.0.8`           |

```
npm install                                    # runs `npx patch-package` postinstall
npx hardhat test test/poc/hx4_capacity_after_payout.js
```

Verified reproducible from a cold build (`rm -rf artifacts cache`, then the command above): 133 contracts recompiled, both tests pass.

### Setup

Deploy the **real** `FirelightVault` (via the repo's own `deployVault` fixture in `test/setup/fixtures.js`) and the **real** `CoverOrderAllocator`, wired to each other. Only the ERC20 token and the price feed are test doubles (`MockERC20`, `MockAggregatorV3`).

Capacity configuration is identical to the team's own fixture (`test/setup/fixtures.js:164-171`):

* `minCAR = 12000`, `effectiveLeverage = 24000` (i.e. 2.0x, well inside `MAX_LEVERAGE_FACTOR = 5`)
* `divergenceToleranceBps = 0`
* oracle pinned at $1.00/fXRP for the whole run (never moved; later `setAnswer` calls re-stamp the same value only to refresh feed freshness after time travel)
* **first-loss buffer funded with 0**, so the vault leg is isolated and the buffer plays no part

Capacity is measured **exactly**, not estimated: the contract exposes no capacity view, so the test probes the `MatchingCapacityOverflow(requested, maxCapacity)` revert at `CoverOrderAllocator.sol:401` via `commitAllocation.staticCall` and reads the ceiling from the revert data.

### Steps

1. **Period 0** - a depositor stakes **10,000 fXRP** into the vault (`vault.deposit`). Confirm `currentPeriod() == 0`.
2. **Advance to period 1.** The curator books two cover orders targeting the next period: **$12,000 Morpho** and **$8,000 Aave** - exactly the 6000/4000 bps per-protocol concentration caps.
3. **Advance to period 2.** Measure the ceiling: `capacityBefore = 20000.0` (10,000 fXRP x $1.00 x 24000/12000).
4. **Flag the incident** captured in period 1 as active in periods 1 and 2 (`vault.setActiveIncident`).
5. **Execute the payout.** `PAYOUT_ROLE` calls `FirelightVault.payout(receiver, 9000 fXRP, periodStart(1) + 100)`. Assert the vault now holds exactly **1,000 fXRP** (`totalAssets() == DEPOSIT - PAYOUT`).
6. **Measure the ceiling again.** `capacityAfter == capacityBefore` - **asserted for equality**, not merely observed. It is still 20000.0 after 90% of the collateral has left the contract. The honest ceiling for 1,000 fXRP would be 2000.0, and the test asserts `capacityAfter > honestCapacity`.
7. **Sell the cover.** The allocator commits and settles the full **$20,000** against that stale ceiling: `commitAllocation(2, root, commitCapacity, commitCapacity)` followed by two `settleCoverOrder` calls. Cover NFTs are minted and the buyers' premiums are transferred out.
8. **Assert insolvency.** `soldCoverUSD > realCollateralUSD` - read from the contract's own record `getAllocationCommitment(2).totalSettledCover` (`:952`, written at `:524`), not from test arithmetic. Result: **$20,000 of cover against $1,000 of collateral = 20x**.
9. **Assert the realised loss.** The premium actually charged is measured as the **delta in the premium collector's token balance** across the two settlements, and cross-checked against the contract's own `commit.totalSettledPremium` (`:527`). Result: **19.178083 USDC**, transferred to an address that is not the vault.

### Output

All printed values are deterministic; only the mocha timings vary between runs.

```
  H-X4 - capacity underwritten against already-paid-out collateral
  capacity before payout : 20000.0
  capacity after  payout : 20000.0
  honest capacity        : 2000.0
  cover SOLD             : 20000.0
  vault collateral left  : 1000.0
  over-subscription      : 20 x
  premium CHARGED        : 19.178083 USDC (non-refundable)
  premium collector      : NOT the vault
    OK  allocator sells cover against vault assets a payout already spent (443ms)
    OK  reads the first-loss buffer live but the vault snapshot stale

  2 passing (458ms)
```

### Controls

Each control is an assertion in the same file, included to show the harness is measuring correctly and the defect is isolated.

1. **Opposing gates.** With the incident active, `maxDeposit == 0` and `deposit` reverts `CurrentPeriodHasActiveIncident`, while $20,000 of cover has just been sold against that same vault. The protocol refuses recapitalisation and permits underwriting in the same state.
2. **Self-heals next period.** At period 3 the ceiling does drop to $2,000, matching the collateral that actually exists. This proves the measurement is correct and isolates the defect to the intra-period window rather than to a permanently frozen trace.
3. **Leg asymmetry (test 2).** Removing $5,000 from the *first-loss* leg moves the ceiling $30,000 to $20,000 immediately. Removing the same $5,000 from the *vault* leg via `payout` leaves it at $20,000. Same function, same amount, opposite responses - this is the empirical core of the finding.

### Notes for the reviewer

* The premium figure 19.178083 USDC is deterministic: $20,000 x 5% annual x 604800/31536000, with `Ceil` rounding.
* No mainnet or testnet interaction of any kind was performed. All testing is local.
* The full test source is available on request and can be supplied as a secret Gist if preferred.
