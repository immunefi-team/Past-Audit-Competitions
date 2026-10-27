# 88638 sc high computeavailablecapacity values the first loss buffer live and the vault at the period start so the vault half of a payout stays in the capital adequacy calculation for the rest&#x20;

**Submitted on Aug 14th 2026 at 09:56:37 UTC by @ret2basic for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88638
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator._computeAvailableCapacity` adds two collateral sources and reads them on **two different clocks**:

```solidity
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer),   // live
    ...
);
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),             // period start
    ...
);
```

A claim payout draws from **both**: the waterfall in `IncidentManager._executePayout` takes the buffer first and the vault second. `FirelightVault.payout` records its new asset total with `_traceTotalAssets.push(Time.timestamp(), ...)` — a key strictly *inside* the period — and `totalAssetsAt` resolves `<= key`, so the period-start reading is untouched by it.

The result is that one payout moves half of the capacity calculation and not the other half. Measured on the runnable PoC, with a single incident paid in the period following its capture:

| Collateral source             | Before payout | After payout | Reflected in capacity         |
| ----------------------------- | ------------- | ------------ | ----------------------------- |
| First-loss buffer (live read) | 500,000       | **0**        | **yes**, in full              |
| Vault (period-start read)     | 1,000,000     | 450,000 held | **no**, still reads 1,000,000 |

**1,100,000 USD of collateral has left the contract and is still counted.** At the fixture's `effectiveLeverage / minCAR` of 24,000 / 12,000 that is **2,200,000 USD of cover capacity backed by nothing**, and it stays that way until the next period begins.

## Vulnerability Details

### The gate that is supposed to bound this is the one that is wrong

`commitAllocation` is the protocol's on-chain solvency check. It refuses a matcher's claimed capacity that exceeds what the collateral supports:

```solidity
function _validateMatchingCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period,
    uint256 matchingCapacity,
    uint256 totalAllocated
) private view {
    uint256 maxCapacity = _computeAvailableCapacity($, period);
    if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
    if (totalAllocated > matchingCapacity) revert TotalAllocationOverflow(totalAllocated, matchingCapacity);
}
```

Settlement does not re-check. `_settleCoverOrder` reads the **stored** `commit.totalAvailableCapacity` for its per-protocol concentration caps, so once a period is committed the number is fixed for every order settled against it. `commitAllocation` also allows exactly one commit per period (`PeriodAlreadyCommitted`), so a single stale evaluation sets the whole period's book.

### Every other on-chain risk limit is a fraction of the number that is wrong

The committed figure is not just a ceiling on the period's total. `_settleCoverOrder` derives each per-protocol diversification limit from it:

```solidity
uint256 protocolConcentrationCap =
    (_getEffectiveProtocolConcentration(concHash, order.period) * commit.totalAvailableCapacity) /
        BPS_DENOMINATOR;
if (newProtocolConcentrationCover > protocolConcentrationCap)
    revert ProtocolConcentrationOverflow(newProtocolConcentrationCover, protocolConcentrationCap);
```

So a capacity inflated by a factor of *k* inflates **every concentration ceiling by the same factor of&#x20;*****k***, in absolute terms. A protocol the risk policy limits to 20% of capacity can then absorb 20% of an overstated number — three times the cover the policy intended, on the measurement below — and the check reports success. The defect does not relax one bound. It relaxes the entire on-chain risk-limit structure at once, because every limit in it is expressed as a fraction of the one number that is stale.

This also answers the reading that `_validateMatchingCapacity` is a loose sanity bound with the real control elsewhere. It is not a bound the protocol can afford to be approximate about: it is the denominator of every other bound.

### The sequence, with nothing out of the ordinary in it

{% stepper %}
{% step %}

## Period `P` begins

The vault holds `A`; `totalAssetsAt(periodStart(P)) == A`.
{% endstep %}

{% step %}

## An incident is approved during `P`

An incident captured in `P-1` is approved during `P` — permitted, and intended: `_isPeriodInPayoutWindow` accepts `currentPeriod == capturePeriod + 1`.

The waterfall empties the buffer, then draws `L` from the vault. The vault now holds `A - L`.
{% endstep %}

{% step %}

## The period-start reading stays stale

`totalAssetsAt(periodStart(P))` **still returns `A`**. The payout's checkpoint was written at its own timestamp, which is greater than `periodStart(P)`.
{% endstep %}

{% step %}

## The period allocation is committed

That period's `commitAllocation` runs and is bounded by `(0 + A · price) · leverage / minCAR`, not by `(0 + (A-L) · price) · …`.
{% endstep %}
{% endstepper %}

Nobody has to misbehave for this to occur. Step 2 is the consortium honouring a claim, step 4 is the allocator committing the day's matching. They are separate parties on separate schedules with nothing sequencing them.

### The same money backs the claim that consumed it and the cover sold after it

The collateral removed in step 2 was underwriting period `P-1` and has been paid out to honour `P-1`'s claim. The cover committed in step 4 is sold for period `P`. Both are sized on the same fXRP. The protocol's capital adequacy ratio — `minCAR`, the one solvency constraint the contracts enforce at all — is satisfied by the gate while being violated in fact.

### The contract states its own solvency floor, and this breaks it

Two constants in the same file carry the protocol's solvency policy in their comments, and both are statements about **real** backing:

```solidity
// Hard ceiling for `CapacityConfig.divergenceToleranceBps`. Kept small: the tolerance
// is accepted under-collateralization vs the CAR target, only meant to absorb the
// price/FLB drift between the off-chain matcher snapshot and commit inclusion.
uint16 private constant MAX_DIVERGENCE_TOLERANCE_BPS = 1_000; // 10%

// Cap on `effectiveLeverage` relative to `minCAR`. Real backing behind sold cover is
// `minCAR / (effectiveLeverage × (1 + tolerance))`, so bounding the ratio floors it at
// `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` regardless of the configured `minCAR`.
uint256 private constant MAX_LEVERAGE_FACTOR = 5;
```

The second is an invariant with a proof sketch attached: *whatever* `minCAR` is configured to, real backing cannot fall below `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))`. The first bounds the deliberate shortfall at **10%**, and names the exact scenario it exists for — drift between the matcher's snapshot and the commit.

Both statements depend on the collateral term in `_computeAvailableCapacity` being real. It is not. The PoC measures the commitment the contract accepted against everything the protocol actually holds:

|                                                                         |                        |
| ----------------------------------------------------------------------- | ---------------------- |
| Cover committed and accepted                                            | 2,400,000 USD          |
| Collateral actually held (vault + buffer, valued as capacity values it) | **400,000 USD**        |
| Floor the contract computes for this configuration                      | **5,000 bps (50%)**    |
| Realised backing                                                        | **1,666 bps (16.66%)** |

That is **three times below the floor the file writes down for itself**, and the shortfall exceeds `MAX_DIVERGENCE_TOLERANCE_BPS` — the only under-collateralization the design accepts — by an order of magnitude. The tolerance cannot absorb it, because the tolerance measures the matcher's drift and this shortfall is on the chain's side of the comparison.

### The control isolates the stale read and nothing else

The PoC commits a capacity of 2,400,000 USD in the payout period. It is accepted. It then advances one period, changing **nothing else** — the vault holds exactly the same 200,000 fXRP — and submits the identical commitment. It reverts:

```
MatchingCapacityOverflow(2400000e18, 800000e18)
```

The vault balance is the same in both attempts. The only difference is whether `currentPeriodStart()` has moved past the payout's checkpoint.

### A third test pins the mechanism to the checkpoint key

A payout executed in the **boundary second itself** writes at key `periodStart(P)` and therefore *does* show up: `totalAssetsAt(periodStart(P))` equals the post-payout balance, and capacity falls immediately. One second later it does not. The defect is the relationship between the checkpoint key and the period start, not the payout.

## Impact Details

**Official impact.** *"Protocol insolvency"* — Critical. Cover is committed and settled against collateral the contract has already transferred out. The overstatement equals the vault's share of the payout and is multiplied by `effectiveLeverage / minCAR`; on the fixture's configuration that is 2× the missing collateral, and `MAX_LEVERAGE_FACTOR` permits more.

**Duration.** From the payout until the end of that period. On the deployed schedule `periodConfigurations[0].duration` is **86,400 seconds**, so up to one day of the protocol's cover book can be written against absent collateral, and the payout window spans two such periods.

**Who is left short.** The buyers of the cover sold in step 4. When an incident in period `P` is assessed, `payout` pays `min(amount, min(totalAssetsAt(periodStart(P)), activePayableAmount))`, and `activePayableAmount` is what the vault actually holds. The contract cannot conjure the missing assets, so the claim is paid down to the real balance and the buyer receives less than the cover they were sold and charged a premium for.

### The snapshot read is deliberate, and the reasoning behind it is one-directional

The project tested this read and wrote its rationale into a test name:

```js
it('uses totalAssetsAt(currentPeriodStart) — mid-period deposits do NOT inflate capacity', ...)
  await vault.setTotalAssetsAtSnapshot(ethers.parseUnits('2000', 6))
  await vault.setTotalAssets(ethers.parseUnits('5000', 6))
```

*(`test/coverOrderAllocator/commitAllocation.js`)*

So the choice is considered, not accidental — and the argument for it is **conservatism**: capital that arrives mid-period must not be counted until the period it opens. That argument is sound in the direction it is made.

The mechanism is symmetric and the argument is not. Both tests that exercise it set the snapshot **below** the live value — a deposit. **No test sets the snapshot above the live value**, which is the only thing a payout can produce, and in that direction the identical line of code stops being conservative: capital that *left* mid-period keeps being counted until the period it opens.

### The desync itself is already documented — for the other direction

`test/FirelightVault/checkpoint.js` contains a test whose name is exactly what this report is about:

```js
it('documents the desync: a donation without checkpoint is missed by the next period-start snapshot', async () => {
  // Forward assets directly (donation-style, like the pre-fix distributor did)
  await ctx.token_contract.mintTo(ctx.firelight_vault.target, REWARDS)
  expect(await vault.totalAssets()).to.equal(DEPOSIT + REWARDS)      // live view sees it
  await advanceOnePeriod(vault)
  expect(await vault.totalAssetsAt(periodStart)).to.equal(DEPOSIT)   // history does not
})
```

The divergence between the live view and the period-start snapshot is therefore **known, named, and regression-tested**. The comment "like the pre-fix distributor did" records that it was found and fixed on the path where assets arrive — `VaultRewardDistributor` now calls `checkpointTotalAssets()` in both of its distribution functions.

The path where assets *leave* has the same divergence. `payout` does push a checkpoint, but at `Time.timestamp()` — inside the period — so the period the payout happens in still reads its opening figure, and that figure is what capacity consumes.

### The one test of falling collateral reduces the half that works

`test/coverOrderAllocator/commitAllocation.js` has a test named *"recommit recomputes capacity and rejects amounts the reduced collateral no longer supports"*. It opens with:

```js
await vault.setTotalAssets(0)
...
// Buffer holder drains 500 USDC → balance 500 → fresh capacity = 1000.
await usdc.connect(firstLossBufferWallet).transfer(buyer1.address, parseUnits('500', 6))
```

The vault's contribution is set to **zero**, and the collateral that falls is the **buffer** — the term read live. So the suite's coverage of `_computeAvailableCapacity` is:

| Direction            | Which term           | Tested                                                                           |
| -------------------- | -------------------- | -------------------------------------------------------------------------------- |
| Collateral rises     | vault (snapshot)     | **yes** — "mid-period deposits do NOT inflate capacity"                          |
| Collateral falls     | buffer (live)        | **yes** — "recommit … rejects amounts the reduced collateral no longer supports" |
| **Collateral falls** | **vault (snapshot)** | **no**                                                                           |

The missing cell is the only one a claim payout produces, and it is the one where the snapshot read stops being conservative.

### The strongest objection, stated rather than left for a judge to find

**The harm requires the allocator to actually commit the inflated figure.** An off-chain matcher that independently read live vault assets and committed conservatively would never let the stale gate bind. This is a defect in the protocol's solvency bound rather than a self-executing exploit, and a judge who weighs it as such is reading it correctly.

Three things argue it is nonetheless the constraint that binds — and the second is the repository's own answer to the objection:

* **`divergenceToleranceBps` exists precisely because the matcher's view is expected to lag the chain.** The design's stated answer to a matcher working from a snapshot is the on-chain recomputation that bounds it. Here the on-chain recomputation is stale *in the same direction as the matcher would be*, so it cannot catch the case it was written for.
* **The matcher is specified as a mirror of this function, in the project's own repository.** `test/coverOrderAllocator/testVectors.js` derives the value it passes as `matchingCapacity` like this:

  ```js
  // Live capacity mirroring _computeAvailableCapacity (tolerance 0): the value the
  // off-chain matcher would have run against, passed as matchingCapacity.
  const matchingCapacity =
    ((flb18 + (assets18 * assetPriceUSD) / 10n ** 18n) * effectiveLeverage) / minCAR
  ```

  A matcher built to that specification does not read live vault assets independently — it reproduces the on-chain formula from the on-chain inputs. So the honest matcher and the on-chain bound consume the *same* stale number, and the commitment in this report is what an honest matcher produces, not a deviation from one.
* **The contract already accepts that a payout must lower capacity.** It does exactly that for the buffer, in the same expression, on the same payout. The vault half is not a deliberate policy choice made differently — it is the same policy, missed.

One factual note on the manual alternative, offered without leaning on it. A commitment can be withdrawn by `CONFIG_ADMIN_ROLE` through `cancelCommitAllocation` or `recommitAllocation`, but both refuse once any order has settled (`SettlementsAlreadyStarted`), so the interval in which a human could catch this is exactly `settlementGracePeriod`. That value has one assignment in the contract, behind `CONFIG_ADMIN_ROLE`, and `initialize` does not set it — it is the only configuration value in `CoverOrderAllocator` the deployment path leaves at its default. At the default of zero, `commitAllocation` and `batchSettleCoverOrder` (both `ALLOCATOR_ROLE`) can run in one transaction. A deployment that sets the parameter has the manual window; one that does not, does not. Either way the on-chain bound is the automatic check, and it is the one this report is about.

### Why the program's exclusions do not cover this

The rule, in full:

> *"Reports requiring these trusted parties to act maliciously **or incorrectly** within their documented privileges are not eligible **unless they demonstrate that the role can exceed its intended authority or bypass an intended protocol constraint**."*

**This report does not require any trusted party to act maliciously or incorrectly, so the rule's precondition is not met.** The consortium approves an assessed claim — that is the action it exists to take, and the claim is genuine. The allocator commits the day's matching against a capacity figure **the contract itself computed and accepted**. Neither has acted wrongly, and neither could have detected the shortfall from on-chain data: the capacity function returns the inflated number to everyone who asks it, including the off-chain matcher the project's own test vectors define as *"mirroring `_computeAvailableCapacity`"*.

**And the exception clause is satisfied independently.** The intended protocol constraint is the one the file writes down for itself — *"real backing behind sold cover is `minCAR / (effectiveLeverage × (1 + tolerance))`"* — enforced on-chain by `_validateMatchingCapacity`. The measured result of a commitment that gate **accepted** is 16.66% against a stated floor of 50%. The constraint is not merely unmet; the mechanism built to enforce it reports success while it is violated by a factor of three, and by an order of magnitude more than the 10% the design accepts as deliberate under-collateralization.

The two disclosed known issues:

* *"Aggregate claims are not capped on-chain against total sold cover."* That is the **claims** side of the balance sheet — total claims versus total cover. This is the **collateral** side — total cover versus assets actually held. Neither implies the other, and fixing the known issue would not change any number in this report.
* *The first-loss buffer is an external, unescrowed wallet, and the protocol relies on it remaining funded and approved.* **This report does not rely on the buffer in either direction.** The first PoC runs with an empty buffer so the whole payout falls on the vault; the fourth runs with the buffer **funded and paying its full share**, and the shortfall persists — 37.5% real backing against the same 50% floor. Whatever the buffer does, the vault's half of a payout stays inside the capacity calculation for the rest of the period, and that is the entire finding.

### The finding does not need an empty buffer, and the fourth test proves it

An empty first-loss buffer is not what the protocol assumes, so a finding that required one would be a weaker finding. `test_theDefectSurvivesAFundedBuffer` funds the buffer with enough to carry half the loss and repeats the measurement:

|                             | Empty buffer  | **Funded buffer**                                   |
| --------------------------- | ------------- | --------------------------------------------------- |
| Buffer paid                 | 0             | **800,000** (drained; capacity registers all of it) |
| Vault paid                  | 800,000       | **400,000**                                         |
| Period-start snapshot after | **unchanged** | **unchanged**                                       |
| Floor the contract states   | 5,000 bps     | 5,000 bps                                           |
| **Realised backing**        | **1,666 bps** | **3,750 bps**                                       |

Funding the buffer shrinks the vault's share of the payout, and the vault's share is precisely the part capacity cannot see — so the shortfall shrinks with it. It does not close: real backing is **37.5%** against a stated floor of **50%**, still past the 10% the design accepts. The defect scales with how much of a claim the vault carries, and the vault carries everything the buffer cannot.

## Recommended fix

Read both collateral sources on the same clock. Either clock is defensible; mixing them is not.

**The recommendation is the third option below.** The first two are listed because they are the obvious ones, and each has a cost worth seeing before it is chosen.

* **Subtract what the period has already paid.** *(recommended)* Accumulate the vault's share of each payout per period — `paidFromActive` is already computed inside `payout` — and deduct that total from the period-start figure in `_computeAvailableCapacity`. This is the only option that closes the gap **without changing any behaviour the project has deliberately chosen**: cover is still sized on the collateral that opened the period, mid-period deposits still do not inflate it, and collateral that has left the contract stops being counted. It is additive — one mapping and one subtraction.
* **Live, for both.** Replace `totalAssetsAt(currentPeriodStart())` with `totalAssets()`. It is the smallest diff and matches the buffer's treatment in the same expression, and it removes the boundary-second asymmetry the third test records. **But it reverses a tested, deliberate decision**: `test/coverOrderAllocator/commitAllocation.js` asserts *"mid-period deposits do NOT inflate capacity"*, and this change makes them inflate it. It should not be taken without deciding that the conservatism that test protects is no longer wanted.
* **Period start, for both.** Keep the vault snapshot and snapshot the buffer to match. Symmetric, and it preserves the stated intent — but it makes the *buffer* side worse rather than the vault side better: a buffer drained mid-period would keep counting for the rest of the period, which is the exact defect in this report pointed at the other reserve.

## References

* `contracts/core/CoverOrderAllocator.sol` — `_computeAvailableCapacity`, `_validateMatchingCapacity`, `commitAllocation`, `recommitAllocation`, `_settleCoverOrder`
* `contracts/core/FirelightVault.sol` — `payout`, `totalAssetsAt`, `currentPeriodStart`, `_traceTotalAssets`
* `contracts/core/IncidentManager.sol` — `_executePayout` waterfall (buffer first, vault second)
* Deployed vault `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` at block 67,324,835 — `periodConfigurations[0].duration == 86400`

## Proof of Concept

### Prerequisites

* **Toolchain:** `forge 1.7.1` (Foundry), `solc 0.8.28` (the version the project pins for `contracts/core/*`), Node.js with `npm` for the dependency tree.
* **No RPC required.** The PoC runs entirely on a local EVM. No environment variables, credentials, or forked state are used.
* **Target:** `https://github.com/immunefi-team/audit-comp-firelight`, branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.
* **Contracts under test:** the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT` and `IncidentManager` from `contracts/core/`, each behind a real `ERC1967Proxy`, initialized through their production `initialize` functions.

### Attacker capabilities

**There is no attacker.** Every actor in this PoC exercises a documented power for its documented purpose:

* The **curator** creates a cover order, commits the day's matching, and reports an incident with a capture timestamp.
* The **consortium** approves the assessed claim. That approval is what executes the payout.
* The **allocator** commits the next matching run.

No role does anything it is not meant to do, no order is misreported, no loss is fabricated, and nothing is done twice that is meant to be done once. The only thing that varies between the target case and the control is **which period the commitment lands in** — that is, whether `currentPeriodStart()` has moved past the payout's checkpoint.

No `vm.store`, no `vm.etch`, no role granted to defeat a check, and no capital that did not enter through `deposit`.

### Environmental doubles, disclosed

* `MockERC20` as the vault asset (6 decimals, matching the live FXRP) and as the premium / first-loss-buffer token (6 decimals).
* `MockAggregatorV3` as the price feed, **held at a constant $2.00 throughout**, refreshed after each `vm.warp` only so it stays inside `maxPriceAge`. Its value never changes, so it cannot account for any measured difference.
* The capacity configuration is the fixture's: `minCAR = 12_000`, `effectiveLeverage = 24_000`, `divergenceToleranceBps = 0`. That makes capacity exactly `2 × collateral`, and at a $2.00 price exactly `4 × the vault's fXRP` — which is why every figure below is a round number and can be checked by hand.
* **The fixture's period is 7 days; the deployed period is 1 day.** The shorter live period makes the blind spot arrive 365 times a year instead of 52. It does not change the mechanism, which depends only on the ordering of a payout and a commitment inside one period.

### Reproduction steps

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
# add fuzz/FullChainFixture.sol and fuzz/CapacityBlindToSamePeriodPayout.t.sol
forge test --match-path 'fuzz/CapacityBlindToSamePeriodPayout.t.sol' -vv
```

### Result

```
Ran 5 tests for fuzz/CapacityBlindToSamePeriodPayout.t.sol:CapacityBlindToSamePeriodPayoutTest
[PASS] test_aPayoutInTheBoundarySecondIsVisibleToCapacity() (gas: 1651390)
Logs:
  boundary-second payout, snapshot / live: 200000000000 200000000000

[PASS] test_coverIsCommittedAgainstAssetsAPayoutAlreadyRemoved() (gas: 1834457)
Logs:
  vault assets, live   before/after payout: 1000000000000 200000000000
  period-start snapshot before/after payout: 1000000000000 1000000000000
  capacity from the snapshot: 4000000000000000000000000
  capacity from live assets : 800000000000000000000000

[PASS] test_onePayoutMovesTheBufferHalfOfCapacityAndNotTheVaultHalf() (gas: 1690836)
Logs:
  buffer   before/after: 500000000000 0
  vault    before/after: 1000000000000 450000000000
  snapshot before/after: 1000000000000 1000000000000
  collateral gone but still counted (USD 18dp): 1100000000000000000000000
  and levered into capacity (USD 18dp)        : 2200000000000000000000000

[PASS] test_realBackingFallsBelowTheFloorTheContractStates() (gas: 1772238)
Logs:
  committed cover (USD 18dp): 2400000000000000000000000
  collateral held (USD 18dp): 400000000000000000000000
  floor the contract states (bps): 5000
  realised backing          (bps): 1666

[PASS] test_theDefectSurvivesAFundedBuffer() (gas: 1809728)
Logs:
  buffer paid   : 800000000000
  vault  paid   : 400000000000
  snapshot after: 1000000000000
  capacity stale / live: 4000000000000000000000000 2400000000000000000000000
  floor / realised (bps): 5000 3750

Suite result: ok. 5 passed; 0 failed; 0 skipped
```

### What each test establishes

#### `test_coverIsCommittedAgainstAssetsAPayoutAlreadyRemoved` — the finding, with its control

The vault falls from 1,000,000 fXRP to 200,000 while `totalAssetsAt(currentPeriodStart())` stays at 1,000,000. Capacity computed from the snapshot is 4,000,000 USD; computed from what the vault actually holds it is 800,000 USD.

A commitment of **2,400,000 USD** — three times the live collateral supports — is accepted.

The control then advances one period and changes nothing else. The test asserts this explicitly:

```solidity
assertEq(vault.totalAssets(), liveAfter, "the vault holds exactly what it held above");
assertEq(vault.totalAssetsAt(vault.currentPeriodStart()), liveAfter, "but now the snapshot agrees with it");
```

and the identical commitment now reverts:

```
MatchingCapacityOverflow(2400000e18, 800000e18)
```

Same vault, same balance, same call, opposite outcome. The only variable is whether the period start has moved past the payout's checkpoint.

#### `test_onePayoutMovesTheBufferHalfOfCapacityAndNotTheVaultHalf` — why this is a defect rather than a design choice

One payout, drawn from both collateral sources by the same waterfall. The buffer read — `firstLossBufferToken.balanceOf(firstLossBuffer)` — falls 500,000 → 0 and capacity registers all of it. The vault read — `totalAssetsAt(currentPeriodStart())` — does not move by a single unit, though the vault paid 550,000.

The protocol therefore already holds that a payout must reduce capacity. It applies that rule to one of its two collateral sources.

#### `test_realBackingFallsBelowTheFloorTheContractStates` — the finding measured against the protocol's own stated invariant

`CoverOrderAllocator` writes its solvency policy into two constants' comments: the divergence tolerance is "accepted under-collateralization vs the CAR target" capped at 10%, and "real backing behind sold cover is `minCAR / (effectiveLeverage x (1 + tolerance))`". For the capacity configuration under test that floor is **50%**. The commitment the contract accepted in the first test has **16.66%** behind it -- three times below its own floor, and past the 10% tolerance by an order of magnitude. The tolerance cannot cover it: the tolerance measures the *matcher's* drift, and this shortfall is on the chain's side of that comparison.

#### `test_aPayoutInTheBoundarySecondIsVisibleToCapacity` — the mechanism, isolated

A payout executed in the boundary second writes its checkpoint at key `currentPeriodStart()` itself. `Checkpoints.Trace256` resolves `<= key`, so that write *is* what the period-start lookup returns, and capacity sees the loss immediately. One second later it does not. This rules out any explanation other than the checkpoint key's position relative to the period start.

### Reading the numbers

Capacity is `(buffer_usd + vault_fxrp × price) × effectiveLeverage / minCAR`, widened by `divergenceToleranceBps` (zero here). With price $2.00 and `24_000 / 12_000 = 2`:

|                                             | fXRP (6dp) | USD (18dp) | Capacity (18dp) |
| ------------------------------------------- | ---------- | ---------- | --------------- |
| Vault at period start (what capacity reads) | 1,000,000  | 2,000,000  | **4,000,000**   |
| Vault actually held after payout            | 200,000    | 400,000    | **800,000**     |
| Difference                                  | 800,000    | 1,600,000  | **3,200,000**   |

Every figure is reproducible by hand from the logs above.

### Complete runnable PoC files

#### `poc-inputs/foundry.toml`

```toml
[profile.default]
src = "contracts"
test = "fuzz"
out = "forge-out"
cache_path = "forge-cache"
libs = ["node_modules"]
# Vendored Flare FAsset sources pin =0.8.23 and are out of the audit scope.
skip = ["contracts/fasset/**"]
solc = "0.8.28"
evm_version = "london"
via_ir = true
optimizer = true
optimizer_runs = 200
remappings = [
  "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/",
  "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/",
  "flare-smart-contracts/=node_modules/flare-smart-contracts/",
  "flare-smart-contracts-v2/=node_modules/flare-smart-contracts-v2/",
  "forge-std/=node_modules/forge-std/src/",
]

[invariant]
runs = 400
depth = 200
fail_on_revert = false
```

#### `poc-inputs/fuzz/FullChainFixture.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {FirelightVault} from "../contracts/core/FirelightVault.sol";
import {CoverOrderAllocator} from "../contracts/core/CoverOrderAllocator.sol";
import {CoverNFT} from "../contracts/core/CoverNFT.sol";
import {IncidentManager} from "../contracts/core/IncidentManager.sol";
import {ICoverOrderAllocator} from "../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {IIncidentManager} from "../contracts/core/interfaces/IIncidentManager.sol";
import {IFirelightVault} from "../contracts/core/interfaces/IFirelightVault.sol";
import {IAggregatorV3} from "../contracts/core/interfaces/IAggregatorV3.sol";
import {MockERC20} from "../contracts/test/MockERC20.sol";
import {MockAggregatorV3} from "../contracts/test/MockAggregatorV3.sol";

/// @dev The real FirelightVault / CoverOrderAllocator / CoverNFT / IncidentManager
///      wired together the way production does. The project's own suites substitute
///      a mock vault into both the allocator and the incident tests, so this is the
///      only place the four meet.
abstract contract FullChainFixture is Test {
    uint48 internal constant PERIOD = 7 days;

    MockERC20 internal fxrp; // vault asset, 6 dp
    MockERC20 internal usdc; // premium + first-loss-buffer token, 6 dp
    MockAggregatorV3 internal feed; // asset/USD, 18 dp

    FirelightVault internal vault;
    CoverOrderAllocator internal allocator;
    CoverNFT internal coverNFT;
    IncidentManager internal incidents;

    address internal admin = address(0xA0);
    address internal curator = address(0xC0);
    address internal allocatorRole = address(0xA1);
    address internal flbWallet = address(0xFB);
    address internal buyer = address(0xB1);
    address internal payoutRecipient = address(0xB2);
    address internal payoutReceiver = address(0xB3);
    address internal lp = address(0x11);

    bytes32 internal marketId;
    uint256 internal refNonce;

    function _deployFullChain() internal {
        vm.warp(1_800_000_000);
        fxrp = new MockERC20("fXRP", "fXRP", 6);
        usdc = new MockERC20("USDC", "USDC", 6);
        feed = new MockAggregatorV3(18, 2e18);

        FirelightVault vImpl = new FirelightVault();
        FirelightVault.InitParams memory p = FirelightVault.InitParams({
            defaultAdmin: admin,
            limitUpdater: admin,
            blocklister: admin,
            pauser: admin,
            periodConfigurationUpdater: admin,
            rescuer: admin,
            depositLimit: 100_000_000e6,
            periodConfigurationDuration: PERIOD
        });
        vault = FirelightVault(
            address(
                new ERC1967Proxy(
                    address(vImpl),
                    abi.encodeCall(
                        FirelightVault.initialize,
                        (IERC20(address(fxrp)), "stfXRP", "stfXRP", abi.encode(p))
                    )
                )
            )
        );

        CoverNFT nImpl = new CoverNFT();
        coverNFT = CoverNFT(
            address(
                new ERC1967Proxy(
                    address(nImpl),
                    abi.encodeCall(
                        CoverNFT.initialize,
                        ("Firelight Cover", "FLC", "", admin, address(0), address(0), address(0))
                    )
                )
            )
        );

        address[] memory premiumTokens = new address[](1);
        premiumTokens[0] = address(usdc);
        ICoverOrderAllocator.ProtocolConcentrationInput[] memory conc =
            new ICoverOrderAllocator.ProtocolConcentrationInput[](1);
        conc[0] = ICoverOrderAllocator.ProtocolConcentrationInput({
            chainId: 1, protocol: "Morpho", maxProtocolConcentrationBps: 10_000
        });
        ICoverOrderAllocator.Market[] memory markets = new ICoverOrderAllocator.Market[](1);
        markets[0] = ICoverOrderAllocator.Market({chainId: 1, protocol: "Morpho", market: bytes32("market-A")});
        marketId = keccak256(abi.encode(uint64(1), string("Morpho"), bytes32("market-A")));

        CoverOrderAllocator aImpl = new CoverOrderAllocator();
        allocator = CoverOrderAllocator(
            address(
                new ERC1967Proxy(
                    address(aImpl),
                    abi.encodeCall(
                        CoverOrderAllocator.initialize,
                        (
                            ICoverOrderAllocator.InitParams({
                                vault: IFirelightVault(address(vault)),
                                premiumCollector: address(0xC01),
                                coverNFT: coverNFT,
                                priceFeedAdapter: IAggregatorV3(address(feed)),
                                maxPriceAge: 7 days,
                                premiumTokens: premiumTokens,
                                admin: admin,
                                adminRole: admin,
                                curatorRole: curator,
                                allocatorRole: allocatorRole,
                                configAdminRole: admin,
                                initialProtocolConcentrations: conc,
                                newMarkets: markets,
                                capacityConfig: ICoverOrderAllocator.CapacityConfig({
                                    minCAR: 12_000,
                                    firstLossBufferToken: IERC20(address(usdc)),
                                    firstLossBuffer: flbWallet,
                                    effectiveLeverage: 24_000,
                                    minOrderMarketCoverAmount: 1,
                                    divergenceToleranceBps: 0
                                })
                            })
                        )
                    )
                )
            )
        );

        IncidentManager iImpl = new IncidentManager();
        incidents = IncidentManager(
            address(
                new ERC1967Proxy(
                    address(iImpl),
                    abi.encodeCall(
                        IncidentManager.initialize,
                        (
                            admin, curator, admin, admin, admin, admin, admin, admin,
                            payoutReceiver,
                            ICoverOrderAllocator(address(allocator)),
                            IAggregatorV3(address(feed)),
                            uint48(7 days)
                        )
                    )
                )
            )
        );

        vm.startPrank(admin);
        coverNFT.grantRole(coverNFT.MINTER_ROLE(), address(allocator));
        vault.grantRole(vault.PAYOUT_ROLE(), address(incidents));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidents));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), admin);
        vault.addToPayoutAllowlist(payoutReceiver);
        vm.stopPrank();
    }

    function _lpDeposit(uint256 amount6) internal {
        fxrp.mint(lp, amount6);
        vm.startPrank(lp);
        fxrp.approve(address(vault), type(uint256).max);
        vault.deposit(amount6, lp);
        vm.stopPrank();
    }

    /// @dev Creates, commits and settles one cover order, returning it and the
    ///      period it covers. The order is raised in the period before the covered
    ///      one, exactly as production does.
    function _sellCover(uint256 coverUsd) internal returns (uint256 orderId, uint256 coveredPeriod) {
        ICoverOrderAllocator.MarketAllocationInput[] memory mas =
            new ICoverOrderAllocator.MarketAllocationInput[](1);
        mas[0] = ICoverOrderAllocator.MarketAllocationInput({
            marketId: marketId, coverRateAnnual: 500, coverAmount: coverUsd
        });
        vm.prank(curator);
        orderId = allocator.createCoverOrder(
            buyer, payoutRecipient, "rAddr", address(usdc), mas, ICoverOrderAllocator.CoverOrderType.NEW
        );

        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        coveredPeriod = vault.currentPeriod();

        usdc.mint(buyer, 1_000_000e6);
        vm.prank(buyer);
        usdc.approve(address(allocator), type(uint256).max);
        vm.prank(flbWallet);
        usdc.approve(address(incidents), type(uint256).max);

        ICoverOrderAllocator.MarketCoverAllocation[] memory mcas =
            new ICoverOrderAllocator.MarketCoverAllocation[](1);
        mcas[0] = ICoverOrderAllocator.MarketCoverAllocation({marketId: marketId, allocatedCover: coverUsd});
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(orderId, mcas))));

        vm.prank(allocatorRole);
        allocator.commitAllocation(coveredPeriod, leaf, coverUsd, coverUsd);
        vm.prank(allocatorRole);
        allocator.settleCoverOrder(orderId, mcas, new bytes32[](0));
    }

    /// @dev Raises an incident on the settled order and drives it to the point
    ///      where only approval (and therefore payout) remains.
    function _openAndSubmitAssessment(uint256 orderId, uint256 lossUsd)
        internal
        returns (uint256 incidentId)
    {
        refNonce++;
        vm.prank(curator);
        incidentId =
            incidents.createIncident(uint48(block.timestamp), "hack", keccak256(abi.encode("ref", refNonce)));
        vm.prank(curator);
        incidents.confirmIncident(incidentId, "ipfs://report");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] =
            IIncidentManager.AssessmentLoss({coverTokenId: orderId, marketId: marketId, amount: lossUsd});
        vm.prank(curator);
        incidents.addAssessmentLosses(incidentId, losses);
        vm.prank(curator);
        incidents.submitCurrentAssessment(incidentId);
    }
}
```

#### `poc-inputs/fuzz/CapacityBlindToSamePeriodPayout.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ICoverOrderAllocator} from "../contracts/core/interfaces/ICoverOrderAllocator.sol";
import {FullChainFixture} from "./FullChainFixture.sol";

/// @dev `_computeAvailableCapacity` reads its two collateral sources with two
///      different time semantics:
///
///        firstLossBufferToken.balanceOf(firstLossBuffer)   <- live
///        vault.totalAssetsAt(vault.currentPeriodStart())   <- period-start snapshot
///
///      A payout draws the buffer first and the vault second, and pushes its
///      vault checkpoint at `Time.timestamp()` — inside the period, therefore
///      strictly after `currentPeriodStart()`. The `<= key` lookup at the period
///      start is unchanged by it.
///
///      So the buffer half of a payout lowers capacity immediately and the vault
///      half stays invisible until the next period begins. Cover committed in
///      the remainder of that period is sized on collateral that has already
///      left the contract, and the overstatement is multiplied by
///      `effectiveLeverage / minCAR`.
///
///      Fixture configuration: minCAR 12_000, effectiveLeverage 24_000,
///      divergenceTolerance 0, price 2 USD => capacity == 4x the vault's fXRP.
contract CapacityBlindToSamePeriodPayoutTest is FullChainFixture {
    uint256 internal constant LP_ASSETS = 1_000_000e6; // fXRP, 6dp
    uint256 internal constant COVER_USD = 2_000_000e18; // canonical 18dp
    uint256 internal constant LOSS_USD = 1_600_000e18; // canonical 18dp

    function setUp() public {
        _deployFullChain();
        _lpDeposit(LP_ASSETS);
        // The buffer is left empty on purpose: the whole loss then falls on the
        // vault, which is the half of the waterfall that capacity cannot see.
        // With a funded buffer the effect is smaller but not different in kind.
    }

    /// The capacity gate accepts, after a payout, a commitment it would refuse
    /// one period later against the very same vault balance.
    function test_coverIsCommittedAgainstAssetsAPayoutAlreadyRemoved() public {
        (uint256 orderId,) = _sellCover(COVER_USD);
        uint256 incidentId = _openAndSubmitAssessment(orderId, LOSS_USD);

        // Into the next period — the second half of the payout window — and a
        // full hour past the boundary second, so the payout's own checkpoint
        // cannot land on `currentPeriodStart()`.
        vm.warp(vault.currentPeriodEnd());
        vm.warp(block.timestamp + 1 hours);
        feed.setAnswer(2e18);

        uint256 payoutPeriod = vault.currentPeriod();
        uint256 snapshotBefore = vault.totalAssetsAt(vault.currentPeriodStart());
        uint256 liveBefore = vault.totalAssets();

        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        uint256 snapshotAfter = vault.totalAssetsAt(vault.currentPeriodStart());
        uint256 liveAfter = vault.totalAssets();

        console.log("vault assets, live   before/after payout:", liveBefore, liveAfter);
        console.log("period-start snapshot before/after payout:", snapshotBefore, snapshotAfter);

        assertLt(liveAfter, liveBefore, "the payout removed assets from the vault");
        assertEq(snapshotAfter, snapshotBefore, "the period-start snapshot did not move");
        assertEq(snapshotAfter, LP_ASSETS, "and still reads the pre-payout balance");

        // capacity == assets * price(2) * leverage(24000)/minCAR(12000), scaled 6dp -> 18dp
        uint256 staleCapacity = snapshotAfter * 1e12 * 4;
        uint256 liveCapacity = liveAfter * 1e12 * 4;
        console.log("capacity from the snapshot:", staleCapacity);
        console.log("capacity from live assets :", liveCapacity);

        // A commitment between the two: unbacked, but the gate does not know it.
        uint256 unbacked = liveCapacity + (staleCapacity - liveCapacity) / 2;
        assertGt(unbacked, liveCapacity, "the commitment exceeds live collateral");

        vm.prank(allocatorRole);
        allocator.commitAllocation(payoutPeriod, keccak256("root"), unbacked, unbacked);

        assertEq(
            allocator.getAllocationCommitment(payoutPeriod).totalAvailableCapacity,
            unbacked,
            "committed against collateral that is no longer in the vault"
        );

        // ---- Control: the identical commitment, the identical vault balance,
        // one period later, when the snapshot has finally caught up.
        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        uint256 nextPeriod = vault.currentPeriod();

        assertEq(vault.totalAssets(), liveAfter, "the vault holds exactly what it held above");
        assertEq(
            vault.totalAssetsAt(vault.currentPeriodStart()),
            liveAfter,
            "but now the snapshot agrees with it"
        );

        vm.expectRevert(
            abi.encodeWithSelector(
                ICoverOrderAllocator.MatchingCapacityOverflow.selector, unbacked, liveCapacity
            )
        );
        vm.prank(allocatorRole);
        allocator.commitAllocation(nextPeriod, keccak256("root"), unbacked, unbacked);
    }

    /// The allocator states its own solvency floor in a constant's comment:
    ///
    ///     // Real backing behind sold cover is
    ///     // `minCAR / (effectiveLeverage × (1 + tolerance))`, so bounding the
    ///     // ratio floors it at `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))`
    ///     // regardless of the configured `minCAR`.
    ///     uint256 private constant MAX_LEVERAGE_FACTOR = 5;
    ///
    /// and calls the tolerance "accepted under-collateralization vs the CAR
    /// target", capped at 10%. Both statements are about *real* backing. Neither
    /// survives a stale collateral term: this test computes the floor from the
    /// capacity configuration in force and measures what the accepted commitment
    /// actually has behind it.
    function test_realBackingFallsBelowTheFloorTheContractStates() public {
        (uint256 orderId,) = _sellCover(COVER_USD);
        uint256 incidentId = _openAndSubmitAssessment(orderId, LOSS_USD);

        vm.warp(vault.currentPeriodEnd());
        vm.warp(block.timestamp + 1 hours);
        feed.setAnswer(2e18);

        uint256 period = vault.currentPeriod();
        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        uint256 staleCapacity = vault.totalAssetsAt(vault.currentPeriodStart()) * 1e12 * 4;
        uint256 liveCapacity = vault.totalAssets() * 1e12 * 4;
        uint256 committed = liveCapacity + (staleCapacity - liveCapacity) / 2;

        vm.prank(allocatorRole);
        allocator.commitAllocation(period, keccak256("root"), committed, committed);

        // minCAR / (effectiveLeverage * (1 + tolerance)), in basis points, from
        // the fixture's capacity configuration: 12_000 / (24_000 * 1.0) = 50%.
        uint256 floorBps = (12_000 * 10_000 * 10_000) / (24_000 * (10_000 + 0));

        // What the commitment actually has behind it: every asset the protocol
        // holds, valued the way capacity values it.
        uint256 collateralUsd = vault.totalAssets() * 1e12 * 2 + usdc.balanceOf(flbWallet) * 1e12;
        uint256 realisedBps = (collateralUsd * 10_000) / committed;

        console.log("committed cover (USD 18dp):", committed);
        console.log("collateral held (USD 18dp):", collateralUsd);
        console.log("floor the contract states (bps):", floorBps);
        console.log("realised backing          (bps):", realisedBps);

        assertLt(
            realisedBps,
            floorBps,
            "real backing is below the floor the contract computes for itself"
        );
        // And by more than the 10% under-collateralization the protocol accepts.
        assertLt(
            realisedBps * (10_000 + 1_000) / 10_000,
            floorBps,
            "the gap exceeds MAX_DIVERGENCE_TOLERANCE_BPS, the only slack the design allows"
        );
    }

    /// The blind spot is the checkpoint key, not the payout: a payout landing in
    /// the boundary second writes *at* `currentPeriodStart()` and is therefore
    /// visible to capacity immediately. One second later it is not.
    function test_aPayoutInTheBoundarySecondIsVisibleToCapacity() public {
        (uint256 orderId,) = _sellCover(COVER_USD);
        uint256 incidentId = _openAndSubmitAssessment(orderId, LOSS_USD);

        // Land exactly on the boundary second and pay out there.
        vm.warp(vault.currentPeriodEnd());
        feed.setAnswer(2e18);
        assertEq(block.timestamp, vault.currentPeriodStart(), "we are on the boundary second");

        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        uint256 snapshot = vault.totalAssetsAt(vault.currentPeriodStart());
        console.log("boundary-second payout, snapshot / live:", snapshot, vault.totalAssets());
        assertEq(snapshot, vault.totalAssets(), "the snapshot recorded the payout");
        assertLt(snapshot, LP_ASSETS, "capacity sees the loss in the same period");
    }

    /// One payout, two collateral sources, two answers. The waterfall drains the
    /// buffer first and the vault second; capacity registers the buffer half at
    /// once and the vault half not at all. Whichever time base is intended, the
    /// two cannot both be right — and the live buffer read shows the design does
    /// expect a payout to lower capacity.
    function test_onePayoutMovesTheBufferHalfOfCapacityAndNotTheVaultHalf() public {
        uint256 bufferFunding = 500_000e6; // USDC, 6dp
        usdc.mint(flbWallet, bufferFunding);

        (uint256 orderId,) = _sellCover(COVER_USD);
        uint256 incidentId = _openAndSubmitAssessment(orderId, LOSS_USD);

        vm.warp(vault.currentPeriodEnd());
        vm.warp(block.timestamp + 1 hours);
        feed.setAnswer(2e18);

        uint256 bufferBefore = usdc.balanceOf(flbWallet);
        uint256 snapshotBefore = vault.totalAssetsAt(vault.currentPeriodStart());

        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        uint256 bufferAfter = usdc.balanceOf(flbWallet);
        uint256 snapshotAfter = vault.totalAssetsAt(vault.currentPeriodStart());
        uint256 vaultLive = vault.totalAssets();

        console.log("buffer   before/after:", bufferBefore, bufferAfter);
        console.log("vault    before/after:", snapshotBefore, vaultLive);
        console.log("snapshot before/after:", snapshotBefore, snapshotAfter);

        assertLt(bufferAfter, bufferBefore, "the buffer paid its share");
        assertLt(vaultLive, snapshotBefore, "and so did the vault");

        // The capacity inputs, read exactly as `_computeAvailableCapacity` reads them.
        assertEq(bufferAfter, 0, "capacity's buffer input fell by the whole buffer payment");
        assertEq(
            snapshotAfter,
            snapshotBefore,
            "capacity's vault input did not fall by one unit of the vault payment"
        );

        uint256 blindSpotUsd = (snapshotBefore - vaultLive) * 1e12 * 2; // fXRP -> USD at 2
        console.log("collateral gone but still counted (USD 18dp):", blindSpotUsd);
        console.log("and levered into capacity (USD 18dp)        :", blindSpotUsd * 2);
        assertGt(blindSpotUsd, 0, "the vault half of the payout is invisible to capacity");
    }

    /// The same measurement with the first-loss buffer **funded**, because an
    /// empty buffer is not what the protocol assumes and a finding that needs one
    /// is a weaker finding.
    ///
    /// The buffer pays first, so funding it shrinks the vault's share of the
    /// payout -- and the vault's share is the only part capacity cannot see. The
    /// defect therefore survives at a smaller size, and this pins that size.
    function test_theDefectSurvivesAFundedBuffer() public {
        uint256 funded = 800_000e6; // covers half the loss
        usdc.mint(flbWallet, funded);

        (uint256 orderId,) = _sellCover(COVER_USD);
        uint256 incidentId = _openAndSubmitAssessment(orderId, LOSS_USD);

        vm.warp(vault.currentPeriodEnd());
        vm.warp(block.timestamp + 1 hours);
        feed.setAnswer(2e18);

        uint256 period = vault.currentPeriod();
        uint256 snapshotBefore = vault.totalAssetsAt(vault.currentPeriodStart());

        vm.prank(admin);
        incidents.approveCurrentAssessment(incidentId);

        uint256 bufferAfter = usdc.balanceOf(flbWallet);
        uint256 liveAfter = vault.totalAssets();
        uint256 snapshotAfter = vault.totalAssetsAt(vault.currentPeriodStart());

        console.log("buffer paid   :", funded - bufferAfter);
        console.log("vault  paid   :", snapshotBefore - liveAfter);
        console.log("snapshot after:", snapshotAfter);

        assertGt(funded - bufferAfter, 0, "the buffer carried its share");
        assertGt(snapshotBefore - liveAfter, 0, "and the vault carried the rest");
        assertEq(snapshotAfter, snapshotBefore, "capacity still cannot see the vault's share");

        // Capacity reads the funded buffer live *and* the stale vault figure.
        uint256 staleCapacity = (bufferAfter * 1e12 + snapshotAfter * 1e12 * 2) * 2;
        uint256 liveCapacity = (bufferAfter * 1e12 + liveAfter * 1e12 * 2) * 2;
        uint256 committed = liveCapacity + (staleCapacity - liveCapacity) / 2;
        console.log("capacity stale / live:", staleCapacity, liveCapacity);

        vm.prank(allocatorRole);
        allocator.commitAllocation(period, keccak256("root"), committed, committed);

        uint256 collateralUsd = liveAfter * 1e12 * 2 + bufferAfter * 1e12;
        uint256 realisedBps = (collateralUsd * 10_000) / committed;
        uint256 floorBps = (12_000 * 10_000 * 10_000) / (24_000 * 10_000);
        console.log("floor / realised (bps):", floorBps, realisedBps);
        assertLt(realisedBps, floorBps, "real backing is still below the contract's own floor");
    }
}
```
