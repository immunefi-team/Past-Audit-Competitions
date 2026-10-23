# 88968 sc high cover capacity in coverorderallocator is sized from a period start snapshot that payouts cannot update so cover is sold against collateral that has already been paid out

**Submitted on Aug 16th 2026 at 02:39:46 UTC by @Hybirdss for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88968
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

Every capacity check in `CoverOrderAllocator` values the vault's collateral at `vault.totalAssetsAt(vault.currentPeriodStart())` — total assets at the instant the current period began (`CoverOrderAllocator.sol:704-724`). `FirelightVault.payout()` writes its accounting checkpoint at `block.timestamp`, which is always strictly after the period start, and the checkpoint lookup selects the last entry at or before the requested time (`FirelightVault.sol:456-458`, `:952`).

The period-start value is therefore immutable for the whole period: **no payout inside a period can ever reduce the collateral basis used to sell cover in that period.** Because the payout window spans two periods (`FirelightVault.sol:1130-1132`), one incident poisons the capacity basis of both.

No role misbehaves and no attacker is required: an honest, fully-approved claim in the ordinary two-period window leaves the allocator selling cover against collateral that has left the vault. Every permissioned decision in this path is *correct*: the consortium approves a genuine full loss exactly as assessed, and the allocator settles strictly within the contract's own capacity figure — the insolvency is produced by correct operation of the protocol, not by any trusted party making an incorrect business decision.

The capacity check is itself an intended protocol constraint, and the stale numerator defeats it without any role exceeding or misusing its authority. In the PoC, the contract's own full computed capacity — 20,000,000 USD — is sold for a period in which the vault holds 500,000 FXRP, a realized backing of 0.025: eight times below the floor the project's own code comment guarantees (`1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` = 0.20, `CoverOrderAllocator.sol:79-82`).

The protocol takes premium payment for cover it cannot honor — the in-scope Critical impact "Protocol insolvency".

### Vulnerability Details

The capacity formula mixes a live read with a frozen snapshot as if both were the same instant:

```solidity
// CoverOrderAllocator.sol:704-724 (_computeAvailableCapacity)
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer), ...);      // LIVE read
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()), ...);               // PERIOD-START snapshot
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;
uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

The vault leg cannot move during the period:

```solidity
// FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);  // last checkpoint key <= t
}
// FirelightVault.sol:952 (inside payout)
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);  // pushed at block.timestamp
```

A payout's timestamp is always strictly after the period start, so its checkpoint is never the one `upperLookupRecent(currentPeriodStart())` selects. The two-period payout window makes the dislocation the normal case, not an edge: an incident captured in period P is normally assessed and paid during P+1, draining the vault inside P+1 while the P+1 capacity basis still shows the pre-drain value.

Nothing downstream re-checks: `_validateMatchingCapacity` (`CoverOrderAllocator.sol:394-403`) runs only at commit/recommit; settlement reads the stored `commit.totalAvailableCapacity`; protocol concentration caps (`:507`) derive from the same inflated number.

#### A distinct root cause the prior audit never addressed

The Macro audit (29 July 2026) filed M-1 as "Unbounded `effectiveLeverage` lets real backing approach zero"; its fix — the `MAX_LEVERAGE_FACTOR` ratio bound — was applied and is fully in force throughout this PoC (`effectiveLeverage / minCAR` = 24,000/12,000 = 2.0 ≤ 5).

This report does not re-file M-1. The root cause here is the freshness of the numerator the ratio multiplies — a period-start snapshot a payout has already emptied. The Macro text contains zero occurrences of `totalAssetsAt` and `currentPeriodStart`; its only period-start discussions concern buyer funding pins (M-2) and concentration caps (L-2), never the collateral basis. The mechanism of this report is absent from it.

The remediation's code comment is cited for one purpose only — it states the backing floor the project believes it guarantees:

```solidity
// CoverOrderAllocator.sol:79-82
// Cap on `effectiveLeverage` relative to `minCAR`. Real backing behind sold cover is
// `minCAR / (effectiveLeverage x (1 + tolerance))`, so bounding the ratio floors it at
// `1 / (MAX_LEVERAGE_FACTOR x (1 + tolerance))` regardless of the configured `minCAR`.
```

With that bound in force, the realized backing is **0.025** — 8x below the comment's guaranteed floor of 1/(5 × (1 + 0)) = 0.20. Execution falsifies the project's stated floor, which is the point of the citation — not a claim about M-1's resolution status.

Calibration:

* The design backing for a config is `minCAR / effectiveLeverage` (0.50 here).
* The comment's floor is `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` (0.20).
* `minCAR` itself (1.2) is a CAR target, not the realized backing.

#### Not the published known issues, by channel

"Aggregate claims are not capped on-chain against total cover sold" concerns how much may be *paid out*; this concerns how much cover may be *sold* — a different quantity in a different function.

"If the underlying staked asset decreases in USD value, available capital and the Capital Adequacy Ratio decrease. Severe collateral drawdowns can reduce coverage capacity and may cause shortfalls or undercollateralization relative to outstanding cover obligations." describes *price* risk in the staked asset — that channel is priced into the same formula live (`assetPriceUSD` is a fresh oracle read at evaluation time, `:700`, consumed in the same formula at `:716`), and that known issue is the acknowledged market-risk residue of the CAR design.

In this report's PoC the asset price is constant at $1.00 throughout; the only quantity that moves is tokens that physically left the vault, and the only stale component of the formula is the vault snapshot. A disclosed price drawdown is a risk; a balance drain invisible to the enforcing check is a defect in the check.

#### The project's own integration documentation contradicts the stale snapshot

`ICoverOrderAllocator.sol:40-41` documents the capacity formula the off-chain matcher computes as `(config.firstLossBufferToken.balanceOf(config.firstLossBuffer) + vault.totalAssets() * assetPriceUSD / 10**priceFeedDecimals) * config.effectiveLeverage / config.minCAR` — a **live** `vault.totalAssets()` read.

The implementation enforces `vault.totalAssetsAt(currentPeriodStart())` (`:711`). One of the two is wrong by the project's own standard: after the PoC's payout, a matcher following the interface documentation sizes the period from the drained 500,000 FXRP balance, and the exhibited sale could not be committed; a matcher following the contract instead sells against collateral the documented formula would not allow.

The period-start snapshot is not a documented design choice — it contradicts the project's own specification of the same computation.

## Impact Details

**Selected impact (in scope, Critical): Protocol insolvency.**

* Cover liabilities are underwritten against capital that has already left the vault. The PoC sells the contract's full allowed capacity — 20,000,000 USD of cover (CoverNFT minted, premium collected) — against the 500,000 FXRP actually present: realized backing 0.025, i.e. 2.5 cents of collateral per dollar of sold cover.
* The overstatement equals the amount paid out in the period multiplied by the configured leverage factor (`effectiveLeverage / minCAR`, bounded by `MAX_LEVERAGE_FACTOR = 5`): a 9.5M payout re-opens 19M of phantom capacity.
* LP capital backs liabilities it never agreed to: the next incident's claims draw on vault assets that sold-cover premiums assumed were present.
* The remediation's written invariant ("floors it ... regardless of the configured minCAR") is falsified by execution at 0.025 vs 0.20 with the ratio check fully in force; the root cause (snapshot freshness) is distinct from what M-1 bounded (the ratio).
* **Funds at risk (guidelines: amount × token price at submission).** In the PoC: 19,500,000 USD of the 20,000,000 USD sold is unbacked (500,000 FXRP actually present). At scale: the deployed vault that the V2 upgrade carries forward holds 56,866,610.69 FXRP today (FXRP ≈ $1.00, ≈$57.15M; read-only RPC, Flare block 67436485, 2026-08-15; depositLimit 60,168,000 FXRP) on 1-day periods — the vulnerable capacity pipeline begins with the V2 deployment under audit, and from its first genuine paid claim the stale basis re-arms every period. With this config's `effectiveLeverage / minCAR` multiplier of 2.0, every 1 FXRP paid out re-opens 2 USD of capacity the check certifies: a single full-loss claim can re-open roughly twice its own amount in uncertified cover.

### Scale independence

The finding does not rest on the 20,000,000 figure. That amount is chosen to falsify the stated floor by the widest margin the contract permits; the defect does not depend on it.

Any cover sold in this window is backed by less collateral than the check certifies, and the contract emits no signal distinguishing the two — `_validateMatchingCapacity` (`:394-403`) returns the same approval for a solvent and an insolvent period.

Nor does the finding rest on an allocator judgment: the on-chain balance is visible to everyone, but the capacity check is the protocol's enforcement boundary, and settling inside the figure it certifies is not an incorrect business decision. A matcher following the project's documented live-`totalAssets()` formula would have sized the period at the drained balance; the enforcement contract approves the stale full figure instead.

## Risk Breakdown

* **Privileges required to create harm: none.** Every permissioned actor in the chain makes a *correct* decision: the curator books ordinary cover, the consortium approves a genuine full loss exactly as assessed, and the allocator settles strictly inside the figure the contract's own check certifies. Nothing an exclusion could attach to.
* **Complexity: minimal.** No price manipulation (oracle price constant at $1.00 throughout the PoC), no flash loan, no external protocol, no reentrancy, no timing race beyond the ordinary two-period payout window.
* **Preconditions:** exactly one genuine full-loss incident paid inside its normal window — the protocol's designed, expected case — plus LP capital in the vault. To be precise about timing: the vulnerable pipeline (V2 `FirelightVault` + `CoverOrderAllocator` + `IncidentManager`) begins at the upgrade under audit and does not exist on mainnet yet; what exists today is the capital base it will carry — the deployed legacy vault holds ≈$57.15M of staked LP capital (read-only RPC, 2026-08-15) that upgrades in place to the V2 implementation. So the capital precondition is live today, and the incident precondition is the protocol's designed, expected case from the first genuine claim after the upgrade.
* **Detectability: none.** The capacity check returns the same approval for a solvent and an insolvent period; no error, event, or revert distinguishes the two.
* **Parameter optimization:** none needed — the PoC reads the contract's own maximum via the `MatchingCapacityOverflow(matching, maxCapacity)` revert argument (`probeMaxCapacity`) and sells exactly that maximum. The worst case is not searched for; it is computed by the vulnerable contract itself. With this config, overstatement scales linearly at 2.0× every FXRP paid out, up to the 5× `MAX_LEVERAGE_FACTOR` ceiling under a more aggressive config.

## Recommendation

{% stepper %}
{% step %}

### Value collateral at the same instant capacity is certified

Replace `$.vault.totalAssetsAt($.vault.currentPeriodStart())` (`CoverOrderAllocator.sol:711`) with a live `$.vault.totalAssets()` read — which is also what the project's own interface documentation (`ICoverOrderAllocator.sol:40-41`) already specifies — and re-validate capacity at settlement, not only at commit/recommit (`:394-403`).
{% endstep %}

{% step %}

### Alternatively, make payouts update the snapshot basis

Keep the snapshot but make payouts update the basis: push a `_traceTotalAssets` checkpoint at `periodStart`-relevant granularity, or have `payout()` emit an event the allocator treats as a capacity invalidation for the live period.
{% endstep %}

{% step %}

### Align documentation and enforcement

Realign the interface documentation and the enforcement, whichever survives, and add an invariant test: after any payout of `x`, `availableCapacity()` must fall by at least `x × effectiveLeverage / minCAR` in the same period.
{% endstep %}
{% endstepper %}

## References

* Capacity formula (period-start snapshot): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol#L704-L724>
* The project's backing-floor comment: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol#L79-L82>
* `totalAssetsAt` (upperLookupRecent): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L456-L458>
* payout checkpoint pushed at block.timestamp: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L952>
* Two-period payout window: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol#L1130-L1132>
* Capacity validated only at commit/recommit: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol#L394-L403>
* Interface documenting the LIVE capacity formula (contradicting the snapshot): <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/ICoverOrderAllocator.sol#L40-L41>
* Live oracle price read in the same formula: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol#L716>
* Audited tree: <https://github.com/immunefi-team/audit-comp-firelight/tree/v1\\_audit\\_ready> (commit 42f9ea5)

## Link to Proof of Concept

<https://gist.github.com/Hybirdss/05d2acd993cb1a25adf66038c6e54740>

## Proof of Concept

**Attack summary** — Vulnerable: `CoverOrderAllocator._computeAvailableCapacity` (stale collateral basis) + `FirelightVault.payout` (writes the accounting checkpoint after the period start). Roles in chain: none malicious; every role acts correctly. Attacker: none required. Attacker capital/knowledge: none. Funds at risk: 19,500,000 USD of uncertified cover in the PoC; scales at 2.0× per FXRP paid out on live config. Environment: local Hardhat, in-scope contracts unmodified, no mainnet/testnet interaction.

Runs against the real `FirelightVault`, `CoverOrderAllocator`, `IncidentManager`, `CoverNFT` (only ERC-20s and the price feed are mocked). Repo at `v1_audit_ready` (42f9ea5); copy `PoC-1-stale-capacity-e2e.js` and `e2e-fixture.js` into `test/poc/` and run:

```bash
npm install
npx hardhat test test/poc/PoC-1-stale-capacity-e2e.js
```

The capacity shown is the contract's own computation, read from the `MatchingCapacityOverflow(matching, maxCapacity)` revert argument via `staticCall` (`e2e-fixture.js`, `probeMaxCapacity`) — no re-implementation of the formula.

### Step-by-step

{% stepper %}
{% step %}

### Period 0

An LP stakes 10,000,000 FXRP into the real `FirelightVault`.
{% endstep %}

{% step %}

### Period 1

The curator books 9,500,000 USD of cover for period 2 on the real `CoverOrderAllocator`.
{% endstep %}

{% step %}

### Period 2

The matching is committed and settled (CoverNFT #0 minted, premium collected). An incident is captured in period 2; a second order for period 3 is booked.
{% endstep %}

{% step %}

### Period 3: Read collateral and capacity

The collateral basis and the contract's allowed capacity are read (10,000,000 FXRP / 20,000,000 USD).
{% endstep %}

{% step %}

### Pay the approved claim

The period-2 incident is assessed and approved inside period 3 (the ordinary window {2,3}); the claim pays 9,500,000 FXRP to the payout receiver. The vault now really holds 500,000 FXRP.
{% endstep %}

{% step %}

### Observe the unchanged basis

**THE BUG:** the collateral basis for period 3 is re-read — it is **UNCHANGED** (10,000,000 FXRP), and the allowed capacity is **UNCHANGED** (20,000,000 USD).
{% endstep %}

{% step %}

### Sell cover against the stale basis

**THE SALE:** a 20,000,000 USD matching is committed and settled (CoverNFT #1 minted, premium transferred) while the vault holds 500,000 FXRP.

On roles: the allocator settles strictly inside the figure the contract's own check certifies — the balance is visible to everyone, but the capacity check is the enforcement boundary, and it returns the same approval for a solvent and an insolvent period.

A matcher following the project's documented formula (live `vault.totalAssets()`, `ICoverOrderAllocator.sol:40-41`) would have sized this period at the drained balance.
{% endstep %}

{% step %}

### Amount independence

The 20,000,000 amount is chosen only to falsify the stated floor by the widest margin; the defect is amount-independent — any cover sold in this window is under-backed relative to what the check certifies.
{% endstep %}
{% endstepper %}

### Environment & dependencies

Program PoC requirements / Web3 PoC guidelines: runnable Hardhat + Mocha test on the in-process `hardhat` EVM network — no mainnet or testnet interaction, no fork, no RPC endpoint, no `.env`, no keys.

Dependencies are exactly the repository's own `package.json` (`@nomicfoundation/hardhat-toolbox`, `@openzeppelin/merkle-tree`):

```bash
npm ci
npx hardhat compile
npx hardhat test test/poc/PoC-1-stale-capacity-e2e.js
```

Mainnet forking is not applicable to this finding — the V2 contracts in scope are not yet deployed (the live vault still runs the legacy predeposit implementation) — so the guidelines' sanctioned alternative is used: the project's own test suite with the audited source unmodified, every condition stated per step.

All files needed to run (test + fixture) are in the gist.

### Expected output

```
  [period 0] LP stakes                   10,000,000 FXRP
  [period 2] cover sold                  9,500,000 USD  (CoverNFT #0 minted)
  [period 3] collateral basis            10,000,000 FXRP
             capacity the contract allows 20,000,000 USD
             --- claim paid out ---       9,500,000 FXRP -> payout receiver
             vault really holds now       500,000 FXRP
             collateral basis (unchanged) 10,000,000 FXRP
             capacity still allowed       20,000,000 USD

  RESULT  cover in force for period 3    20,000,000 USD (the contract's own full capacity)
          real collateral behind it      500,000 FXRP  ($1.00/FXRP)
          realised backing ratio         0.025
          design backing (minCAR/effLev) 0.50    <- 20x above realised
          code-comment floor 1/(5*(1+0)) 0.20    <- 8x BELOW it:
          the code comment (COA:79-82) claims backing is floored at
          1/(MAX_LEVERAGE_FACTOR*(1+tol)) regardless of minCAR. Executed result: FALSE.
```

Assertions:

* The basis and capacity are unchanged after the payout.
* The settled cover equals the contract's own full capacity (`coverSold == maxCapacity`).
* CoverNFT #1 is minted and the premium transferred.
* The realized backing is below the floor the project's own comment guarantees (`collateral * 1e12 * 5 < cover`, i.e. backing < 0.20 — executed 0.025).

The repository's own suite passes unmodified in a clean tree (448 tests); adding the two PoC files above brings the same run to 449 passing.

The PoC uses the fixture's 7-day periods; the deployed vault runs 1-day periods (`periodConfigurations(0).duration = 86,400 s`, Flare mainnet, read via RPC) — a period length that carries across the upgrade — so in the V2 deployment the stale basis re-arms daily.
