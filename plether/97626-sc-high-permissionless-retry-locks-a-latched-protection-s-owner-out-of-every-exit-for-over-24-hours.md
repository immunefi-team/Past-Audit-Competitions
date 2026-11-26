# \#97626 [SC-High] Permissionless retry locks a latched protection's owner out of every exit for over 24 hours

**Submitted on Nov 2nd 2026 at 14:53:13 UTC by @albemiglio for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97626
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Permanent freezing of funds

## Description

## Brief/Intro
Written with AI assistance under my direction. I reran the PoC and checked each claim against the code myself, and I take full responsibility for this report.

Since the MED-9 fix (commit eadc5c58), a protection close carries the trader's own financial ceilings. If a price gap breaches a ceiling, the close fails with `ConstraintViolation` and the protection returns to `Latched`. From that state the owner has one way out: `amendLatchedPositionProtectionCloseLimits`. It requires the status to be `Latched` and the account to have no pending orders. But `retryPositionProtectionClose` is permissionless, so any address can re-queue the same failing attempt the moment the protection relatches. That puts the protection back in `Triggered` and the account back in "pending". A griefer that settles the doomed attempt and retries in the same transaction never leaves the owner a block in which amend, cancel or a manual close can succeed. The owner's margin stays locked in a position they cannot exit: for a whole frozen weekend (30 h shown), or until they are liquidated.

## Vulnerability Details
All references are at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, in `packages/perps/src/`.

- `PositionProtectionBook.sol:315-327`, `amendLatchedPositionProtectionCloseLimits`: owner only, reverts unless the status is `Latched` and `pendingOrderCounts(owner) == 0`.
- `PositionProtectionBook.sol:382-405`, `retryPositionProtectionClose`: callable by anyone. It needs `Latched`, the same position and an empty queue, then commits a new attempt with the stored (failing) ceilings, which sets the status to `Triggered`.
- `PositionProtectionBook.sol:408-418` (`_cancel`) and `:341-347` (`cancelPositionProtectionAndCommitClose`): both revert unless the status is `Armed` or `PendingOpen`.
- `router/OrderLiquidationBatchLogic.sol:314`: a manual close reverts with `ProtectionActive`.

The griefer is about 10 lines of code (`RetryGriefer` / `ExpiryRetryGriefer` in the PoC). In one transaction it calls `executeOrder(attempt)`, which fails on the ceiling (or expires the attempt once its deadline has passed, with no price data), then calls `retryPositionProtectionClose(id)`. After every griefer transaction the protection is `Triggered`, so all four exits of the owner revert.

Failed protection attempts pay no keeper, so no honest keeper has a reason to race the griefer to execute the attempt first. The cost is one Arbitrum transaction per execution window: 60 per hour, 2,066 for 30 hours in the PoC.

## Impact Details
**Temporary freezing of funds for at least 24 hours (High).** In the weekend PoC the stop-loss latches on Friday before the oracle freeze. The frozen mark keeps every attempt failing the same ceiling, and the griefer keeps the owner out for 30 hours inside one frozen window (`test_poc_weekendGriefKeepsOwnerOutForOver24h`). Without the griefer the owner amends during the freeze and exits at the frozen price (`test_control_weekendOwnerAmendsAndExitsWhileFrozen`).

**Damage to the user.** Stop-loss example from the PoC:
- Position: LONG 10,000 units with 2,000 USDG margin.
- Protection: stop at 1.10, gross-debit ceiling of 1,200 USDG.
- The market gaps to 1.15 and the stop-loss close fails on the ceiling.
- Control: the owner amends and exits with **18,491.2** USDG of settlement.
- Attack: the owner is locked in while the price keeps moving and is liquidated, ending with **17,999.8** USDG. That is **491.4** USDG more lost, about a quarter of the margin.

Facts:
- The trigger needs no role. Any address can call it, and only the victim's protection id is needed.
- The precondition is a protection whose ceilings a price gap breaches. The MED-9 fix added these ceilings precisely to protect against gaps.
- It is not covered by the known issues:
  - Cecuro MED-9 (Addressed) says tight limits "may prevent execution until conditions improve or the owner amends them"; this report shows a third party can remove the owner's half of that remedy.
  - The scan Low on unpaid trigger bounties and the SECURITY.md "Router and keeper limitations" describe liveness when nobody retries, not a third party retrying to lock the owner out.
  - DEPLOYMENT.md asks keepers not to loop a failing attempt. That is operator guidance, not a control against an attacker.

**Recommendation.** Any of these works:
- let the owner amend while the protection is `Triggered`, cancelling the live attempt;
- make `retryPositionProtectionClose` owner-only after a `ConstraintViolation`;
- extend the atomic cancel-and-close path to `Latched` protections.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L418
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L314
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md (Router and keeper limitations)
- Cecuro AI audit report 2026-10-26, MED-9 (`audits/plether-ai-audit-report-2026-10-26.pdf`)

## Link to Proof of Concept

https://gist.github.com/albemiglio/5278f8b8d41b63826c995cf35c9a374f

## Proof of Concept

- forge 1.8.5 with solc 0.8.35 and via-IR, using the repo's own `PositionProtectionTest` harness with mock Pyth. The four PoC files go in `packages/perps/test/poc_fixes/`.
- **15 of 15 tests pass**, controls included. The key log lines:
  - `griefer cycles: 25` with the victim's position still open;
  - the control settlement of `18491200000` against `17999800000` under attack;
  - `griefer transactions: 2066` and `hours blocked: 30`.
- In every control, the same state without the griefer lets the owner amend and close.

Prerequisites: a checkout of `immunefi-team/audit-comp-plether` at a82d4fef, and Foundry 1.8.5. Copy the four files from the secret Gist https://gist.github.com/albemiglio/5278f8b8d41b63826c995cf35c9a374f (`LatchedAmendGrief.t.sol`, `LatchedAmendGriefExpiry.t.sol`, `LatchedStopLossGrief.t.sol`, `LatchedWeekendGrief.t.sol`) into `packages/perps/test/poc_fixes/`. Then run from `packages/perps`:

```
FOUNDRY_GAS_LIMIT=9223372036854775807 FOUNDRY_FFI=false FOUNDRY_TEST=test/poc_fixes \
forge test --match-path 'test/poc_fixes/*' --match-test 'test_poc_|test_control_' -vv
```

The raised gas limit is needed only for the 30-hour loop, which runs in a single test.

Expected output: `15 tests passed, 0 failed`, with the log lines above.
