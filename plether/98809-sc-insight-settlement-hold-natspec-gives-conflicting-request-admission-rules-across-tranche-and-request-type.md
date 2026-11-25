# \#98809 [SC-Insight] Settlement-hold NatSpec gives conflicting request-admission rules across tranche and request type

**Submitted on Nov 10th 2026 at 10:28:31 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98809
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Summary

Six in-scope source/API comments give conflicting descriptions of the dedicated LP epoch
settlement hold. Four say LP request admission remains open without identifying an exception;
two say Senior requests stop without limiting that statement to deposits. The implementation is
a request-type-by-tranche matrix: the hold blocks only **Senior deposit-request admission**.
It does not, by itself, block Junior deposit requests or either tranche's redemption requests.

The correction is a Documentation Improvement, not a vulnerability claim. The category covers
substantial documentation changes that clarify complex logic and security-relevant behavior and
help prevent incorrect implementations. Here the comments sit on emergency-admin, pool, and
integration-facing surfaces and give incompatible guidance about which ordinary request paths are
admitted during an incident control.

### Exact behavior matrix

Under the isolated settlement-hold flag, with ordinary lifecycle gates otherwise satisfied:

| Operation | Senior | Junior |
| --- | --- | --- |
| Deposit-request admission | Blocked by the hold | Not blocked merely by the hold |
| Redemption-request admission | Not blocked merely by the hold | Not blocked merely by the hold |
| Deposit activation | Deferred by the hold | Deferred by the hold |
| New redemption funding | Deferred by the hold | Deferred by the hold |

Already-funded claims are independent of this request-admission statement. Full emergency
containment separately invokes the ordinary HousePool pause and therefore has additional effects;
this report concerns the dedicated settlement hold in isolation.

### Six affected comments and one correction

The following four comments omit the blocked Senior-deposit cell:

1. `interfaces/IHousePoolEmergencyAdmin.sol:9-13` says the hold operates “without pausing LP
   request admission.”
2. `interfaces/IPerpsLPViews.sol:31-39` says the hold does not imply request admission is
   disabled while describing both tranches and both request directions.
3. `EmergencyPauseCoordinator.sol:142-149` says the hold acts without pausing LP request
   admission.
4. `interfaces/PerpsViewTypes.sol:122-138` says the flag defers work without disabling request
   admission.

The following two comments overstate the Senior exception and are inaccurate for Senior
redemptions:

5. `HousePool.sol:398-405` says the hold stops “Senior request admission.”
6. `interfaces/IHousePool.sol:286-293` says “Senior requests stop.”

All six should state the same matrix: Senior **deposit-request** admission stops; Junior deposit
requests and both Senior and Junior redemption requests remain available subject to their normal
gates; deposit activation and new redemption funding are separately deferred. This is one
documentation root, not six submissions.

### Why ordinary Senior deposits stop

`TrancheVault._requestDeposit()` runs `_requireRequestDepositPreflight()` before the
Senior-specific reservation call. During the hold, `maxRequestDeposit()` obtains zero capacity
because `HousePool.canAcceptTrancheDeposits(true)` is false, so the ordinary call reverts at
preflight. `HousePool.reserveSeniorDeposit()` has a separate hold guard; it is defense in depth,
not a later branch normally reached after that preflight failure.

`TrancheVault.requestRedeem()` and `maxRequestRedeem()` contain no settlement-hold gate, which is
why both tranches' redemption-request admission remains available subject to ordinary checks.
Settlement later defers new redemption funding while the hold remains active.

### Why the correction is valuable

The correction is valuable whenever a maintainer, integrator, generated-documentation consumer,
test author, or incident responder uses these source/API comments to understand, implement, test,
or operate the dedicated settlement hold across the four tranche-and-request combinations. The
hold need not already be active and no consumer failure must already have occurred: documentation
is used to build integrations and incident procedures before the control is needed.

Under that condition, the current comments do not provide one implementable model. Four omit the
Senior-deposit exception, while two extend it incorrectly to Senior redemptions. Replacing them
with the exact matrix above clarifies complex incident-control logic and helps prevent
misunderstandings and incorrect implementations. No current consumer failure is claimed.

Correct public operational documentation elsewhere narrows the marginal benefit, but it does not
remove the contradictory source/API contracts at the pinned competition commit.

### Public overlap and incomplete-fix boundary

Cecuro `MED-19` publicly reported refundable Senior-capacity reservation cycling and states that
the repair uses an explicit settlement hold that closes new Senior admission. This report does
not re-report reservation cycling or claim the held Senior-deposit behavior is novel. It reports
the residual documentation matrix: commit
`153c0f195a51f9ef4f57ecb978f59ff604460298` introduced the hold and four admission-neutral
comments; commit `d757d0b175b5c2154babee946362ff90bd585f22` added the Senior deposit guards
and broad “Senior requests” wording, leaving the four neutral comments in place. The pinned
result is an incomplete documentation fix: four comments omit the deposit exception and two
extend it incorrectly to Senior redemptions.

Cecuro `LOW-23` is adjacent Documentation Improvements prior art about a withdrawal-capacity
view omitting the hold as a zero-return cause. It does not concern request admission or these six
comments. No exact retained report was found with this same six-comment target, four-cell matrix,
and correction. That is a bounded offline comparison, not proof about private or concurrent
reports; filing-day refresh remains necessary.

### Scope, recommendation, and limitations

The primary selectable Portal asset is the pinned `packages/perps/src` tree, which covers the
five affected row-4 source files. `HousePool.sol` is separately listed as a row-2 asset and is the
sixth same-root affected source. Filing should proceed only if the authenticated form permits the
cross-row description under one report.

Replace the six comments with the exact matrix above. Keep the similarly worded
`PerpsPublicLens.sol` comment contextual only because that file is expressly excluded. Do not add
mature-Junior cancellation policy, hold expiry/governance design, settlement-liveness changes,
or vulnerability-severity claims; those are different roots.

No current frontend failure, production incident, asset loss, or vulnerability severity is
claimed. The audit-linked object `86e3c54fe57e65c7f1afd9c3c0e9c09cc9bc1a88` was unavailable in
the local clone and was not retrieved; the cited local `153c0f...` and `d757d0...` history is the
lineage relied upon.

### One-finding and leakage boundary

This report contains only the settlement hold's four-cell request-admission documentation
invariant and its six in-scope comments. Prior items are included only to distinguish knownness.
It does not disclose or combine separate oracle, pricing, accounting, cancellation, liveness,
governance, or optimization findings.


## Proof of Concept

### Static source demonstration

Use frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. Read the four admission-neutral comments at
   `IHousePoolEmergencyAdmin.sol:9-13`, `IPerpsLPViews.sol:31-39`,
   `EmergencyPauseCoordinator.sol:142-149`, and `PerpsViewTypes.sol:122-138`.
2. Read the two broad Senior comments at `HousePool.sol:398-405` and
   `IHousePool.sol:286-293`.
3. For a Senior deposit request, follow `TrancheVault.sol:577-585` into the preflight at
   `1574-1600`, then `maxRequestDeposit()` at `1313-1325`. The hold makes
   `HousePool.canAcceptTrancheDeposits(true)` false, so the maximum is zero and preflight reverts.
   `HousePool.sol:768-776` independently guards `reserveSeniorDeposit()` as defense in depth.
4. For a Junior deposit request, the settlement-hold condition does not make
   `canAcceptTrancheDeposits(false)` false; ordinary capacity and lifecycle gates still apply.
5. For redemptions, inspect `TrancheVault.requestRedeem()` at `606-624` and
   `maxRequestRedeem()` at `1328-1334`. Neither has a settlement-hold admission gate, for either
   tranche.
6. Separately inspect the settlement paths: deposit activation is deferred by the hold, and
   `HousePool.sol:797-803,1348-1356,1684-1689` defers new redemption funding. Those later-phase
   effects do not change request admission.

The resulting matrix is:

```text
                         Senior                    Junior
deposit request          blocked by hold           not blocked merely by hold
redemption request       not blocked merely by hold not blocked merely by hold
deposit activation       deferred                  deferred
new redemption funding   deferred                  deferred
```

### Why the correction is valuable

The correction is valuable whenever a maintainer, integrator, generated-documentation consumer,
test author, or incident responder uses these source/API comments to understand, implement, test,
or operate the dedicated settlement hold across the four tranche-and-request combinations. The
hold need not already be active and no consumer failure must already have occurred: documentation
is used to build integrations and incident procedures before the control is needed.

Under that condition, the current comments do not provide one implementable model. Four omit the
Senior-deposit exception, while two extend it incorrectly to Senior redemptions. Replacing them
with the exact matrix above clarifies complex incident-control logic and helps prevent
misunderstandings and incorrect implementations. No current consumer failure is claimed.

Correct operational documentation elsewhere limits the marginal benefit, but it does not remove
the contradictory source/API contracts at the pinned competition commit.

The exact correction is documentation-only: qualify the four admission-neutral comments with the
Senior deposit exception, and narrow the two broad Senior comments from “Senior requests” to
“Senior deposit requests.” No transaction, network action, executable vulnerability PoC, or
attachment is required for this source-level demonstration.

