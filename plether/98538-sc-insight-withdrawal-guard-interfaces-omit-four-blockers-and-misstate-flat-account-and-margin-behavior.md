# \#98538 [SC-Insight] Withdrawal guard interfaces omit four blockers and misstate flat-account and margin behavior

**Submitted on Nov 8th 2026 at 14:40:34 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98538
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Description

### Summary

The two interfaces describing the same trader-withdrawal guard are incomplete in complementary ways. `IWithdrawGuard.checkWithdraw` says accounts without a position pass, while `ICfdEngineSettlementSidecar.validateWithdraw` reduces the guard to carry realization followed by fresh-mark, exact-lot, initial-margin health. The implementation can reject a flat account on an active order settlement floor; it rejects live positions while the engine is degraded or the mark is stale, rejects any carry that remains unsettled after realization, rejects an underfunded negative-VPI reserve, and evaluates health at the greater of initial margin and the currently active maintenance or FAD margin.

This is a Documentation Improvement, not a vulnerability-severity claim. The implementation is the source of truth and is not alleged to be incorrect.

### Exact discrepancy

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, the interface says:

- carry is realized;
- the mark must be fresh; and
- exact-lot initial-margin health is enforced.

The clearinghouse-facing `IWithdrawGuard` interface separately says that
accounts without a position pass. That is false when a flat account has an
active order settlement floor.

The implementation applies these branches; the stale-mark branch is the one already summarized by “fresh-mark,” while the others expose the omissions and threshold mismatch:

1. For a flat account, an active order settlement floor can reject the withdrawal (`CfdEngineSettlementSidecar.sol:261-271`).
2. For an open position, degraded mode rejects before mark evaluation (`:274-275`).
3. A stale mark rejects (`:278-280`).
4. Carry is realized, but any remainder still rejects (`:283-295`).
5. A negative lifetime-VPI reserve below its current target rejects (`:297-300`).
6. Position health is evaluated with `max(initMarginBps, fadMarginBps)` during FAD and `max(initMarginBps, maintMarginBps)` otherwise, not unconditionally at initial margin (`:307-331`).

### Why the improvement is valuable

An integrator or maintainer using either canonical interface as its function contract can build an incomplete withdrawal preflight or misclassify a valid revert. The distinction between a flat-account settlement floor, degraded/stale market state, unsettled carry, underfunded VPI backing and the active health threshold determines the corrective action.

The concrete `CfdEngine.checkWithdraw` NatSpec and the separate
`ICfdEngineAccountLens.getWithdrawableAssets` NatSpec already document most or
all of the correct behavior. Aligning both withdrawal-guard interfaces removes
conflicting contracts at the abstraction boundaries instead of requiring a
reader to locate and reconcile concrete implementation prose or a different
lens interface.

### Recommended correction

Replace the two interface descriptions with one consistent two-branch contract:

- Flat accounts remain subject to any active order settlement floor.
- Open-position withdrawals require a non-degraded engine, a fresh mark, zero carry remaining after realization, fully funded negative-VPI reserve, and equity strictly above the greater of initial and active maintenance/FAD margin.

Name the relevant typed failures or link to their interface declarations so clients can map each condition without reverse-engineering the implementation.

### Limitations

- No live exploit, loss, freezing impact or vulnerability severity is claimed.
- No runnable code is required under the programme's Insight rules.
- `CfdEngine.checkWithdraw` already states the complete rule,
  `ICfdEngineAccountLens.getWithdrawableAssets` describes much of the
  open-position behavior, and the top-level clearinghouse interface says
  generically that engine guards apply. Those are material counterexamples to
  any claim that the behavior is undocumented everywhere. The improvement is
  specifically to make both canonical interface contracts complete and
  consistent.
- `validateWithdraw` is callable only by the engine, so the direct audience is protocol maintainers, auditors and tooling around this module rather than arbitrary wallet callers.
- Private knownness is unknowable and filing-day public knownness still requires refresh.


## Proof of Concept

## Proof of Concept / Conditions under which the Insight is valuable

1. Check out commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Read `packages/perps/src/interfaces/IWithdrawGuard.sol:7-15` and
   `packages/perps/src/interfaces/ICfdEngineSettlementSidecar.sol:50-54`.
   Record the flat-account-pass statement and the one-sentence carry,
   fresh-mark, initial-margin summary.
3. Read `packages/perps/src/CfdEngineSettlementSidecar.sol:255-305`. Record the flat-account settlement-floor branch and the degraded, stale-mark, unsettled-carry and VPI-reserve failures for an open position.
4. Read `packages/perps/src/CfdEngineSettlementSidecar.sol:307-331`. The health basis is the greater of initial and the currently active maintenance/FAD margin.
5. Compare those implementation conditions with both interface descriptions
   and the already-correct concrete `CfdEngine.sol:722-728` control. Four
   independent rejection conditions are omitted, the flat-account statement is
   false under an active settlement floor, and the threshold is narrower in the
   sidecar interface than in code.

The Insight is valuable whenever an interface consumer preflights or explains a withdrawal for an account with an order settlement floor, degraded or stale market state, carry that cannot be fully collected, insufficient negative-VPI backing, or active FAD/maintenance margin above initial margin.

Expected result: the implementation correctly rejects under those states, while the public interface documentation does not tell the consumer that it will do so or states the wrong health threshold.
