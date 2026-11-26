# \#98532 [SC-Insight] Clarify that uncovered carry blocks partial close commitment instead of remaining unsettled

**Submitted on Nov 8th 2026 at 13:29:04 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98532
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Description

### Summary

The NatSpec for `CfdEngine.reserveCloseOrderExecutionBounty` says that carry is collected from position margin and
then free settlement, and that any uncovered amount remains unsettled. That sentence is only true for the full-close
commitment branch. After projecting the carry collection, `CloseCommitmentLib.project` reverts
`CfdEngine__PartialCloseCarryUnfunded(unpaidCarryAssets)` whenever the requested close is partial and any carry remains
uncovered.

This is a documentation discrepancy. The planner behavior is treated as correct, and no vulnerability impact is
claimed.

### Exact discrepancy

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

- `CfdEngine.sol:592-624` contains the unqualified uncovered-carry NatSpec and the concrete wrapper call into the
  settlement sidecar.
- `ICfdEngine.sol:69-77` repeats the same unqualified behavior contract.
- `OrderRouterBase.sol:99-108` shows that the router's zero-bounty close-admission hook calls that concrete engine
  entrypoint.
- `CfdEngineSettlementSidecar.sol:335-352` builds the account snapshot and delegates the commitment plan to
  `ICfdEnginePlanner.planCloseCommit` before realizing carry.
- `CfdEnginePlanner.sol:19-23` delegates that plan to `CloseCommitmentLib.project`.
- `CfdEnginePlanLib.sol:597-605,614-637` mutates the planning snapshot after margin-first/free-settlement collection and
  leaves any remainder in `snap.unsettledCarryAssets`.
- `CloseCommitmentLib.sol:33-39` records that remainder in `effects.carryOutstandingAssets`, but then reverts if
  `size != snap.position.size` and the remainder is nonzero.

The full-close branch is the decisive control. When `size == snap.position.size`, the partial-only block is skipped,
so the commitment may return the outstanding carry for later terminal recovery or waiver. A partial commitment never
returns that state; it reverts and the attempted state transition rolls back.

### Why the improvement is valuable

A maintainer or integrator who reads the function-level NatSpec or generated contract documentation can infer that a
partial reduction may be committed while uncovered carry simply stays on the account. The actual admission contract
requires the partial close to fund all carry. Correct wording lets such a consumer distinguish an admissible partial
reduction from a full-exit flow and handle the typed rejection deliberately.

The improvement is narrow: qualify the existing sentence rather than changing the planner. Suggested wording:

> Carry uses position margin first, then free settlement. A full close may leave an uncovered amount to the terminal
> settlement rules; a partial close reverts if any carry remains uncovered.

### Material limitations

- `packages/perps/CLOSE_PREVIEW.md:72-76,88-102` already documents the correct partial/full rule and typed error.
- `packages/perps-aa-client/src/orderV3.ts:36-58` already classifies the error and advises reviewing a full exit.
- `ICfdEngine.sol` is labelled a legacy aggregate interface; the proposed target is the concrete in-scope
  `CfdEngine.sol`, whose own NatSpec repeats the mismatch.
- The separate `PartialCloseUnhealthy` condition is not part of this report.
- No claim is made that all Plether consumers are currently misled, that any transaction succeeded incorrectly, or
  that funds, gas, or bytecode are affected.
- Filing-day knownness, authenticated form target, rendered Preview, fee, and operator SEND remain open.


## Proof of Concept

## Proof of Concept / Conditions under which this Insight is valuable

Runnable code is not required for this Insight. The deterministic source comparison is:

1. Check out commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Read `packages/perps/src/CfdEngine.sol:592-624` and
   `packages/perps/src/interfaces/ICfdEngine.sol:69-77`. Record the unqualified statement that uncovered carry remains
   unsettled and the concrete wrapper's sidecar delegation.
3. Follow the router hook at `OrderRouterBase.sol:99-108` into the concrete engine entrypoint, then follow the call
   through `CfdEngineSettlementSidecar.sol:335-352` and `CfdEnginePlanner.sol:19-23` to
   `CloseCommitmentLib.project`.
4. Read `CfdEnginePlanLib.sol:597-605,614-637`. Confirm that carry collection mutates the snapshot and leaves the
   uncollected remainder in `snap.unsettledCarryAssets`.
5. Read `CloseCommitmentLib.sol:33-39`. Confirm that a nonzero remainder causes
   `CfdEngine__PartialCloseCarryUnfunded` exactly when `size != snap.position.size`.
6. Use `size == snap.position.size` as the control. The partial-only branch is skipped, so the commitment can return
   `carryOutstandingAssets`; this does not assert that every later full-close check succeeds.
7. Compare `packages/perps/CLOSE_PREVIEW.md:76,90,102`, which already states the narrower rule, with the contradictory
   function-level NatSpec.

The Insight is valuable when a maintainer, ABI-documentation reader, or independent integrator relies on the concrete
contract's function documentation rather than the separate close-preview guide to decide whether a carry-indebted
partial close can be committed.

Expected result: only the full-close commitment branch can return uncovered carry as outstanding. A partial-close
commitment with any uncovered carry reverts instead of preserving that remainder in a successful commitment.
