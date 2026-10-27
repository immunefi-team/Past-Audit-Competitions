# 88753 sc high boundary snapshot lets withdrawals escape claims

**Submitted on Aug 14th 2026 at 21:37:18 UTC by @Griffinf for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88753
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Target

* Repository: <https://github.com/immunefi-team/audit-comp-firelight>
* Branch: v1\_audit\_ready, commit 42f9ea5e43d88b35197b901acc2a8c24b314fa62
* Contracts: contracts/core/FirelightVault.sol and contracts/core/lib/Checkpoints.sol

## Severity

Claimed Critical, "Protocol insolvency". A valid 57,360,574.888550 FXRP covered loss, assessed strictly inside settled cover allocations and approved by ordinary honest roles, is paid only 48,371,573.738982: a permanent 8,989,001.149568 FXRP shortfall. The payout window is capture period plus one, there is no retry, the code processes incidents "with the available amount, even if paidAmount is zero" (line 929), and the cover receipt NFT remains outstanding against an obligation the protocol has provably failed to meet.

Secondary applicable Critical impact, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". The proof of concept ends with a machine-verified ERC20 transfer of exactly 1,408,670.359806 FXRP into the attacker's wallet through the public claimWithdraw path; the control run, identical in every byte except that the withdrawal executes one second later, delivers those same assets to the receiver of the approved claim and leaves the attacker exactly zero. That differential shows the defect is not merely accounting or griefing. Every attacker-controlled step is roleless; the intervening cover and incident actors perform ordinary authorized operations on correct inputs.

We do not claim High: no current High category describes either demonstrated outcome.

## The bug

Three individually correct mechanisms compose when a user flow executes in a block timestamped exactly at a period start S:

1. Every deposit and withdrawal pushes a total-assets checkpoint keyed at the block timestamp, and same-key pushes overwrite (FirelightVault.sol lines 1044-1056; Checkpoints.sol lines 207-208).
2. A block at exactly the boundary second belongs to the new period, whose start is S (lines 1123-1128).
3. The snapshot read is a less-than-or-equal lookup (lines 456-458), so a checkpoint keyed at S is returned as the period-start value.

A withdrawal executed at S therefore writes the reduced total as period P's snapshot, while the withdrawn assets are booked into withdrawAssets\[P+1] (line 1015) and still counted payable (lines 916, 926). Every claim captured in P is clamped by the escaped amount (lines 911, 927), the pro rata draw under-consumes the attacker's withdrawal bucket, and one period later claimWithdraw (lines 788-811) hands the unconsumed remainder to the attacker. The code states its anti-escape intent (lines 918-920): withdrawals "should also remain payable; otherwise exposed assets could escape by withdrawing first". At any timestamp after S that holds; at exactly S the escape happens one layer down, in the checkpoint write, and nothing can repair the snapshot once the boundary second has passed.

Timing is measured, not inferred: the deployed vault runs one-day periods, and of its last 90 boundaries, 75 (83.3 percent) had a block timestamped exactly at S; the 15 misses landed 1 to 7 seconds late (per-boundary data attached). Withdrawals are not incident-gated. For a staker already intending to exit, a missed boundary attempt degrades to the ordinary withdrawal path without an additional protocol penalty.

## Impact

Measured on a Flare mainnet fork of the deployed vault (57,360,574.888550 FXRP across 4,651 holders) with the production claim machinery end to end: 58,000,000 USD of cover sold and settled through the real CoverOrderAllocator, one incident assessed strictly inside its settled allocations, approval paid through the real IncidentManager.

The largest staker withdraws their full 8,989,001.149568 FXRP at exactly S. The approved claim requests the full payable 57,360,574.888550 and the vault pays only 48,371,573.738982. In period P+2 the attacker claims 1,408,670.359806 FXRP as a real transfer; the remaining 7,580,330.789762 of the escaped stake stays backing the other stakers. The control run pays the claim in full and the attacker's claim reverts with NoWithdrawalAmount. One second of timing moves 8,989,001.149568 FXRP, 15.7 percent of the vault, out of an approved claim, permanently.

## Distinctness from known issues and prior public reports

The published aggregate-cap known issue concerns trusted incident and assessment roles staying within permissions; here the exploit steps (withdraw, claimWithdraw) are public, every privileged actor acts honestly on correct inputs, and what breaks is the per-call exposure bound the vault does implement: an unprivileged bypass of an implemented constraint. It is not the prior competition's period-derivation cluster (input-handling defects, since fixed; here the period math is correct and the defect is which side of the boundary a same-second checkpoint write lands on) and not the audit's payout refactor (which changed only how the same values are resolved). Two separately submitted reports consume this snapshot but operate on a correctly recorded one and remain exploitable after this write-side defect is fixed; freezing the snapshot at rollover repairs this report alone.

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/d48500b18bce0032554063fa0bbfeea6>

## Proof of Concept

Attached: poc\_f4-2.js (runnable proof) and census-flare-boundaries.txt (read-only boundary census). Place the test at `test/poc_f4-2.js` in a checkout of commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, then:

```bash
npm install
npx hardhat test test/poc_f4-2.js
```

Flare mainnet fork pinned to block 67321080 (override RPC with the `FLARE_RPC` environment variable). Two runs from identical fork state: each upgrades the real deployed proxy at `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` to the V2 implementation through the protocol's own authorized actors (all real storage preserved), wires the repository's real CoverOrderAllocator, CoverNFT, and IncidentManager, sells and settles the cover through the real Merkle flow, and reads S from the vault's own currentPeriodEnd. The attack run pins the withdrawal block to exactly S, the control run to S+1; nothing else differs. Asserted to the base unit: the inconsistent snapshot/bucket pair at S, the real IncidentManager claim paying exactly the corrupted cap, the attacker's claimWithdraw transferring exactly 1,408,670.359806 FXRP in the attack run and reverting with exactly zero in the control. Impersonation (ProxyAdmin, the staker) is fork setup only; the exploit steps are public paths.
