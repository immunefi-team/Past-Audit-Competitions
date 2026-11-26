# \#99454 [SC-Insight] Repeated borrow-base sync adds gas to valid close commitments

**Submitted on Nov 13th 2026 at 18:16:31 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99454
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro
A valid close commitment repeats borrow-base synchronization after its settlement sidecar has already collected carry. If carry consumes position margin, the carry path has updated the borrow base; otherwise the borrow-base inputs are unchanged. The final sync therefore adds trader-paid gas to an ordinary risk-reducing action without changing accounting. This is a code-optimization Insight; no fund loss is claimed.

## Vulnerability Details
At the pinned revision, CfdEngine.reserveCloseOrderExecutionBounty delegates close admission to CfdEngineSettlementSidecar and then unconditionally calls _syncPositionBorrowBase(account, _positions[account]) at CfdEngine.sol:623. The sidecar plans the close, calls settlementRealizeCarry(account), and reads post-carry account buckets (CfdEngineSettlementSidecar.sol:325-352). It does not otherwise mutate position margin.

When accrued carry consumes margin, _realizeCarryFromSettlement already calls _syncPositionBorrowBaseToMargin(pos, marginBefore - marginConsumedAssets) at CfdEngine.sol:1428. When no margin is consumed, position margin and the stored borrow base stay unchanged. The outer call rereads the clearinghouse margin bucket, recomputes the same value, and updates side accounting and storage with no change.

The terminal-curve finalizer after this call remains necessary. The suggested optimization removes only the repeated borrow-base sync while preserving admission, carry collection, and _endTerminalCurveMutation.

## Impact Details
Any trader with a live position pays this extra work when committing a valid close through the Router. In an independent pinned-revision replay, the existing public close test used 5,086,746 gas before the focused change and 5,083,669 gas after it: a 3,077-gas reduction. The test passed in both builds, and a separate zero-carry close control passed with the change. These are local test gas figures, not a deployed-transaction measurement. There is no demonstrated theft, freeze, or incorrect accounting; the selected impact is Code Optimizations and Enhancements Insight.

## Mitigation
For the pinned settlement sidecar, remove the outer _syncPositionBorrowBase call after confirming that any margin-consuming carry branch synchronizes the borrow base before returning. Keep the terminal-curve mutation finalizer and all close checks. A different sidecar implementation should explicitly preserve that invariant.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L598-L624
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L325-L352
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449

## Proof of Concept

## Proof of Concept / condition of value

1. At pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, fund a trader, open a live LONG, allow carry to accrue, and submit a valid close with OrderRouter.commitOrder. The repository's test_CloseCommit_FreshCarryCheckpointUsesPositionMargin in test/perps/spec/trader/OrderRouterCommitment.t.sol exercises this public path and asserts that carry reduces margin.
2. Run: forge test --root packages/perps --match-path test/perps/spec/trader/OrderRouterCommitment.t.sol --match-test test_CloseCommit_FreshCarryCheckpointUsesPositionMargin --gas-report -vv
3. In a disposable copy of the same revision, remove only this line after the settlement-sidecar call in CfdEngine.reserveCloseOrderExecutionBounty:

_syncPositionBorrowBase(account, _positions[account]);

4. Re-run the same test. It should still pass its carry and margin assertions with lower gas. The unchanged _endTerminalCurveMutation call must remain. The zero-carry control test_DustFullResidualCloseCommit_Allowed also passed on the patched build. Its margin is unchanged, so this outer sync makes no accounting difference.

The improvement is valuable whenever a user commits a close for a live position; it does not depend on a malicious actor or unusual oracle data.
