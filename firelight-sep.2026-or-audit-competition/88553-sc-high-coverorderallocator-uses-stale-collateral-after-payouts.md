# 88553 sc high coverorderallocator uses stale collateral after payouts

**Submitted on Aug 13th 2026 at 19:10:44 UTC by @Griffinf for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88553
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Target

Repository <https://github.com/immunefi-team/audit-comp-firelight>, branch v1\_audit\_ready, commit 42f9ea5e43d88b35197b901acc2a8c24b314fa62. Contracts: contracts/core/CoverOrderAllocator.sol (defect), contracts/core/FirelightVault.sol (payout checkpoint semantics). The vault is the V2 upgrade implementation for the live Flare proxy 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 (about 57.4 million FXRP, 4,651 holders); the proof of concept runs against that proxy's real storage on a mainnet fork.

## Severity

Claimed Critical, "Protocol insolvency". Fallback: the protocol demonstrably funds only 34,360,574.88855 FXRP of a valid 55,000,000 USD approved in-allocation loss, leaving 20,639,425.11145 FXRP unfunded while 100,000,000 USD of receipted cover remains outstanding against a vault with zero assets.

The invariant at stake: a payout for a previous-period incident that lands during the current period must reduce the collateral available to back newly committed current-period cover. It does not, and the proof of concept carries the consequence end to end on the fork: a routine paid claim, excess cover actually sold and receipted, and a valid covered loss the drained vault cannot honor. Every actor performs an ordinary authorized operation with correct inputs throughout, each incident assessed inside its settled allocation and each payout a single draw inside the vault's own bound. The vulnerability is that the allocator's implemented on-chain capacity constraint validates those inputs against collateral that has already been paid out, so the normal workflow exceeds the intended solvency constraint without any actor exceeding its role. That is the constraint-bypass case the program places in scope; a compromised allocator key timing its commit after a payout is only the adversarial variant of the same broken check.

## The bug

`_computeAvailableCapacity` values staked assets at the period start: `$.vault.totalAssetsAt($.vault.currentPeriodStart())` (CoverOrderAllocator.sol, line 711). The pin itself is documented and deliberate (line 690, matcher determinism), and this report does not dispute it. The defect is what it does across the payout window the vault design itself creates: an incident captured in period P stays payable through P+1, and `FirelightVault.payout` pushes the reduced total-assets checkpoint at the payout timestamp (FirelightVault.sol, line 952), which is strictly after P+1's period start whenever a P claim is paid during P+1. `totalAssetsAt` is a last-checkpoint-at-or-before lookup (lines 456 to 458), so every capacity computation for the whole of P+1 keeps counting the paid claim as collateral.

Two places in the code show the intent this breaks. The first-loss-buffer leg of the same collateral sum is read live at commit time (line 705): a buffer payout deflates capacity immediately, a vault payout never does, two legs of one solvency number with opposite freshness. And the vault blocks deposits while the current or previous period has an active incident, with the in-code rationale that fresh deposits would otherwise back unresolved exposure (FirelightVault.sol, lines 1112 to 1118): cross-period incident overlap is supposed to constrain current-period capital flows, yet nothing symmetric exists on the cover-sales side and nothing deflates capacity when the payout executes.

The stale value then propagates. `commitAllocation` validates the caller's `matchingCapacity` against it (lines 400 to 402) and stores it (line 352); the stored value is the settlement bound (lines 524 to 526) and the denominator of every per-protocol concentration cap (line 507); settlement never re-reads capacity.

## Impact

Measured by the proof of concept against the real proxy's storage. One honestly assessed 23,000,000 FXRP claim from the previous period is paid during the current one: live collateral falls to 34,360,574.88855 FXRP while the period-start snapshot still reports 57,360,574.88855. The allocator's live capacity ceiling remains 126,193,264 USD, proven by the MatchingCapacityOverflow revert arguments one unit above it, where the same configured formula on actual collateral permits at most 75,593,264 USD even with the divergence tolerance at the code maximum. The ordinary commit is accepted at 114,721,149 USD and stored as the settle bound and concentration-cap denominator, and settlement mints the cover NFT and pulls the premium for 100,000,000 USD of live cover. The overstatement scales directly with the amount paid out during the period.

The loss is realized rather than projected: one legitimate incident in the new period, assessed at 55,000,000 USD inside the settled allocation and inside the collateral the capacity check counted as available, is approved through the real IncidentManager. The vault pays its entire remaining 34,360,574.88855 FXRP and drains to exactly zero. The approved covered loss is underfunded by 20,639,425.11145 FXRP, a 37.5 percent shortfall, bounded by the 23,000,000 FXRP paid to the earlier claim while it still counted as collateral. The test ends with a valid receipted cover NFT, an approved in-allocation loss, and an empty vault.

## Prior art and distinctness

The Macro report touches this surface only in I-9, an informational item asking that the matching capacity the engine ran against be stored at commit time; the fix (commit 411e7ca) made it caller-supplied and validated by `_computeAvailableCapacity`. The stale vault leg predates that fix, was carried through it unchanged, and the report never discusses capacity across payouts, so this is neither a re-report of I-9 nor a bypass of its fix. Of the public known issues, the buffer-custody issue's stated boundary explicitly leaves capacity-snapshot-versus-spend-time divergence alive, and the aggregate-claim-cap issue is the opposite direction: claims exceeding cover under trusted roles, whereas here no role misbehaves and capacity exceeds collateral. A separately submitted report covers the vault's own payout cap failing to accumulate across multiple incidents captured in the same period; here the two incidents sit in different periods, each payout is a single draw inside the vault's own per-call bound, and the defect lives entirely in the allocator's capacity read for the following period's commitment. Different contract, different check, independent fixes.

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/7ed2ea37844ed630fc3377e2c140d2db>

## Proof of Concept

Attached: poc\_f1-1.js. Place it at test/poc\_f1-1.js in a checkout of the commit above, then run `npm install` and `npx hardhat test test/poc_f1-1.js`. It runs on a Flare mainnet fork pinned to block 67321080 (default public RPC; override with the FLARE\_RPC environment variable). Impersonation reproduces only the planned deployment: the ProxyAdmin owner performs the V2 upgrade and, as the vault's DEFAULT\_ADMIN, wires the repository's real CoverOrderAllocator, CoverNFT, and IncidentManager exactly as production would. A repository mock stablecoin and a price feed pinned at 1.00 USD are deployment configuration; the price scales the stale and the live capacity readings identically. Capacity configuration: minCAR 1.2, leverage 2.0 inside the documented band, concentration caps 60 and 40 percent.

The sequence: period-P cover sold and settled through the real commit and Merkle-proof flow; one incident assessed at exactly its settled allocation and paid through the real IncidentManager during P+1; the ordinary P+1 commit and settlement; then one valid incident against that settled cover, approved. Asserted with exact values: the P+1 snapshot still reports the pre-payout collateral; the on-chain capacity ceiling equals it exactly (the MatchingCapacityOverflow revert arguments one unit above prove it); the 114,721,149 USD commit is accepted and stored as the settle bound; settlement mints the cover NFT and pulls the premium for 100,000,000 USD of cover; and the final approval pays 34,360,574.88855 FXRP, drains the vault to exactly zero, and leaves 20,639,425.11145 FXRP of the approved loss unfunded, bounded by the collateral the stale read double-counted. The run ends:

```
  1 passing
```
