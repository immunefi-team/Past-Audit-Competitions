# \#98248 [SC-Insight] Whitepaper promises pre-maturity cancellation of firm Senior deposits

**Submitted on Nov 6th 2026 at 09:50:35 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98248
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Brief/Intro
The pinned whitepaper tells LPs that a complete pending deposit can be cancelled before maturity. That promise also reads as applying to Senior deposits, but the Senior vault rejects cancellation from the moment a healthy request is accepted. A reader who deposits Senior USDG expecting a pre-maturity exit will discover that the commitment is firm. Correcting this paragraph would make the commitment clear before the user submits funds.

## Vulnerability Details
At [WHITEPAPER.md:695-703](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L695-L703), the sentence about cancelling a complete pending deposit has no Junior qualification. The next sentence describes later cancellation using an outdated “Senior-capacity escape” description. The code at [TrancheVault.sol:832-860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L832-L860) starts its normal cancellation block with `IS_SENIOR || currentLpEpoch() >= requestId`. Thus a healthy Senior request enters the block even before maturity and reverts `TrancheVault__SeniorDepositCommitted`; a comparable Junior request can cancel before maturity.

This is the intended Senior policy introduced by [PR #123](https://github.com/Plether-Fi/plether-core/pull/123) in response to MED-19. That change corrected comparable wording in [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L878-L883) and [ACCOUNTING_SPEC.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L291-L298), while the cited whitepaper sentence remained. [PR #132](https://github.com/Plether-Fi/plether-core/pull/132) also left that sentence unchanged.

## Impact Details
This is a Documentation Improvements Insight. The valuable condition is a user reading the whitepaper to decide whether to make a Senior deposit. The inaccurate cancellation promise can mislead that decision. No theft, new contract defect, or duration-based freeze is claimed.

## Recommended correction
Replace the generic cancellation sentence with: “A complete pending Junior deposit can be cancelled before its processing epoch. A Senior deposit is a firm commitment once accepted, including before its processing epoch. Senior requests can be refunded when the epoch is rejected, LP settlement is explicitly held, the projected terminal state wipes the tranche, or the Senior tranche is impaired after pending-deposit reconciliation.” Keep the following redemption sentence separate.

## References
Competition target: `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The linked source, whitepaper and corrected specification are all from that pin.

## Proof of Concept

Condition: a user reads the pinned whitepaper before submitting a Senior deposit during healthy operation, before the request’s processing epoch.

1. On the pinned Foundry fixture, at 10:00 UTC before the five-minute epoch cutoff, submit a 10,000 USDG Junior deposit request, then call `juniorVault.cancelPendingDeposit(juniorReqId)` from the controller. It returns 10,000 USDG.
2. From the same healthy fixture, submit a 10,000 USDG Senior deposit request. Assert `seniorVault.currentLpEpoch() < seniorReqId`: the request has not matured.
3. Call `seniorVault.cancelPendingDeposit(seniorReqId)` from its controller. It reverts `TrancheVault__SeniorDepositCommitted` despite the whitepaper’s unqualified pre-maturity promise.

This normal public-call control was run on the writing machine under explicit Forge 1.5.1 and Solc 0.8.35 at the competition pin: `test_Lead5_SeniorPendingDepositCannotBeCancelledBeforeProcessingBoundary`, 1 passed, 0 failed. The source is the test-only `packages/perps/test/perps/l16/L16DocsPromises.t.sol:562-584`; the scoped contracts were unchanged. A runnable test is supplementary evidence for this Insight; the contradiction is visible directly in the two pinned files above.
