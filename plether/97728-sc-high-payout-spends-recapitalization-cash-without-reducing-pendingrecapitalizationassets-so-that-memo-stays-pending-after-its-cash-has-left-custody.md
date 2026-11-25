# \#97728 [SC-High] `payOut` spends recapitalization cash without reducing `pendingRecapitalizationAssets`, so that memo stays pending after its cash has left custody

**Submitted on Nov 2nd 2026 at 21:34:17 UTC by @OxSCSamurai for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97728
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

`payOut` spends recapitalization cash without reducing `pendingRecapitalizationAssets`, so that memo stays pending after its cash has left custody

## Severity

Insight

## Severity justification

Category claimed: Insight, “Security best practices.”

Since this mechanism sits close to several impact rows, I would rather draw the lines myself than leave them to inference.

The Low row (“Contract fails to deliver promised returns, but doesn't lose value”) is not claimed. The entitled claim gets paid in full, and the memo never lands on junior principal as an extra charge. `SECURITY.md:161-162` documents that `recapitalizePool` queues claimant allocation, and on this trace that queue entry is created exactly as documented. `SECURITY.md:345` documents that a trader claim stays senior to pool liquidity until it is paid, and this trace pays it. `capClaimantBuckets` is the pre-audit SC-L1 fix, and it behaves as written here: a bucket that current distributable cash cannot cover settles not at all.

Nor am I claiming the freeze rows, either the Medium one (at least 1 hour) or the High one (at least 24 hours). Nothing the protocol already holds for an LP or a trader is shown locked. As for `canAcceptTrancheDeposits`: it does return false on this trace, but it would return false anyway, because the zero-cash loss waterfall wipes projected principal while share supply remains, and that alone closes those views with no memo involved. So the memo is not offered as a freeze, and I do not claim it is the only reason those views are false.

What the PoC actually shows is a stored-state defect. `payOut` transfers the recap cash and leaves `pendingRecapitalizationAssets` equal to that receipt. A committed reconcile then leaves raw custody at 0, both principals at 0, and the pool book equal to the still-pending memo.

# Vulnerability details

## Summary

I found this while tracing a trader claim through `settleTraderClaim` and checking what the pool's book looked like afterward.

`CfdEngine.recapitalizePool` pulls USDG into `HousePool` and queues the same amount in `pendingRecapitalizationAssets`. A later `settleTraderClaim` pays that cash out through `HousePool.payOut`. `payOut` decreases `accountedAssets` and transfers the token. It never reads or writes `pendingRecapitalizationAssets` or `pendingTradingRevenueAssets`.

Reconcile absorbs only the slice of those buckets that current distributable cash can settle. After the payout, that slice is zero for the cash the memo was counting, so the memo stays pending. The pool book used below is `seniorPrincipal + juniorPrincipal + unassignedAssets + pendingRecapitalizationAssets + pendingTradingRevenueAssets`. Immediately after `payOut`, and before reconcile, that book exceeds raw custody by the logged pre-reconcile gap. The committed reconcile then writes both principals to zero through the ordinary loss waterfall, because distributable cash is zero. It does not reduce the memo. Raw custody stays 0, so the book after that reconcile is the memo itself. A second reconcile seven days later still leaves the memo unchanged.

The trader who is paid is the entitled claimant: the call is `settleTraderClaim` with `msg.sender == account`, and the token lands in the clearinghouse. The finding is that unbacked memo, nothing more. It is not a freeze finding.

## Root cause

Two writers share the pool's raw balance. Only one of them maintains the pending-bucket memo.

1. `recapitalizePool` (`CfdEngine.sol:632-652`) transfers USDG into the pool, checks the balance increase equals the argument, then `recordClaimantInflow(..., Recapitalization, CashArrived)`. That path does `accountedAssets += amount` (`HousePool.sol:640-641`) and `pendingRecapitalizationAssets += amount` (`HousePool.sol:1798-1799`). At that instant the new cash and the new memo match.

2. `settleTraderClaim` (`CfdEngine.sol:559-590`) is the claimant (`msg.sender == account` at `CfdEngine.sol:562-564`). It reverts when `pool.totalAssets() < totalTraderClaimBalanceAssets` (`CfdEngine.sol:224-226`) and otherwise calls `pool.payOut(address(clearinghouse), amount)` (`CfdEngine.sol:238`).

```604:612:packages/perps/src/HousePool.sol
function payOut(address recipient, uint256 amount) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

3. The only decrement of the pending buckets is `_applyPendingClaimantBucketsLive` (`HousePool.sol:1759-1776`), and it decrements the settleable slice only. That slice is `capClaimantBuckets(pending, distributable)` (`HousePool.sol:1829-1833`, `HousePoolPendingPreviewLib.sol:72-82`). Distributable cash does not include tokens `payOut` has already transferred, so an unbacked memo is left pending on purpose by this cap.

`pendingTradingRevenueAssets` is the same shape, and `payOut` ignores it too. The measured case below is the recapitalization bucket.

`SECURITY.md` records that trader claims are senior ("Trader claim balance remains a senior claim on pool liquidity until serviced") and that owner recapitalization "queues claimant allocation for normal reconciliation" (`SECURITY.md`, Owner-funded recapitalization). It does not say the queue remains after that cash has been transferred to a claim. The invariant tables have a row for claim seniority and a row for terminal deficit. They have no row that pending claimant buckets are backed by unspent cash.

## Scope

- Code in scope is commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`packages/perps/src` is identical at that commit). Assets: `HousePool.sol`, `CfdEngine.sol`, and the HousePool libraries those functions call. `git rev-parse HEAD` in the in-scope tree is that commit. The PoC adds tests only. It does not edit those contracts.
- The program page lists recapitalize-the-pool as an immediate owner action, and it states: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." `recapitalizePool` is that documented action (`SECURITY.md`: immediate, owner-only, exact receipt, queues `Recapitalization` with `CashArrived`). The claimant settlement that spends the cash is permissionless for the entitled account.
- Known issues. The published open Cecuro rows are MED-7, MED-13, MED-16, LOW-57, MED-22, MED-24, MED-29, and MED-30. None of them is claimant-bucket backing or `payOut`. The same-root-cause rule ("a report on the same root cause is valid only if it shows that the fix at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` is incomplete or introduces a new bug") does not capture this report: the root cause is a different writer, and the earlier fixes are present and behaving as written. See the next section.

## Not the same issue as Cecuro MED-1 or pre-audit SC-L1

**Cecuro MED-1** (Acknowledged; response says addressed) is a stale-mark release of reserved buckets into principal:

> "A senior-rate finalization can consume pending claimant buckets even when the cached oracle mark is already stale under the protocol's real PoolReconcile policy. ... If the 30-second-old mark still reports distributable at least 100, finalizePoolConfig() consumes the bucket, moves it into tranche principals, and clears the reservation."

The response says senior-rate finalization now checks the pool freshness limit and the PoolReconcile policy before consuming pending claimant buckets. The trigger is `finalizePoolConfig` plus a mark that passes the pool staleness limit and fails the tighter engine limit. This report never calls `finalizePoolConfig`, never uses a stale mark, and never moves the bucket into principal. `payOut` leaves the bucket in place after the cash is gone. That is the opposite direction from MED-1.

**Pre-audit SC-L1** (fixed in `plether-core#29`) is uncapped credit into principal:

> "The principal is overstated when `_reconcile()` applies pending claimant buckets without capping the credit against the liability-adjusted distributable amount. ... a PoC showed the raw principal at 100,000 USDC, while the correct value was 49,960 USDC."

The response says: "Pending recap/revenue is now capped to what's actually settleable after MTM/deferred liabilities. Anything that can't be settled yet stays pending, stays reserved everywhere, and gets applied later once it's backed."

That cap is in the code this report executes (`capClaimantBuckets`). On the end-to-end path it is why the memo stays pending: distributable cash is already gone, so the cap settles nothing. SC-L1 was principal credited above NAV; this is cash leaving through `payOut` while the memo stays behind. The SC-L1 fix is not incomplete. It simply does not run on `payOut`, so it never sees this spend.

The 15 October 2026 AI scan does not mention `pendingRecapitalization`. `SECURITY.md` line 1032 ("no residual pending claimant assets") is the initial trading-activation check. The live deposit view is separate: `_entriesMaySettle` / `_projectedEntriesMaySettle` fold the pending buckets into `projectedUnassignedAssets` (`HousePool.sol:1107-1108` and `:1126-1127`), and `trancheDepositsAllowed` returns false when that value is nonzero (`HousePoolTrancheGateLib.sol:32-34`). That predicate is intentional. On this trace it is not the only closed view: `canAcceptTrancheDeposits` also returns false when projected principal is zero and share supply remains (`HousePool.sol:494-498`), which is the state after the zero-cash loss waterfall.

## Link to Proof of Concept

https://gist.github.com/dappconsulting/144b0f75ad9cb6096f9921f8f9b2b810

## Proof of Concept

# Proof of Concept

Secret Gist:
> https://gist.github.com/dappconsulting/144b0f75ad9cb6096f9921f8f9b2b810

Attach these three files under `packages/perps/test/perps/audit/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. They are new tests. They do not modify `packages/perps/src`.

The impact test is `GhostMemoPocTest.test_poc_recapMemoGhost`. The trader claim is created with `router.commitOrder` and `router.executeOrder`. The test does not write engine storage. The test pranks `juniorVault` and calls `pool.reconcile()` (`onlyVault`, `HousePool.sol:1471`) so `_reconcile` commits. That external entry is not production: `TrancheVault` never calls `reconcile()`, and nothing in `packages/perps/src` does. `settleLpEpoch` calls the same `_reconcile` (`HousePool.sol:835`). With no matured epoch it then reverts `HousePool__NoLpEpochProgress` (`HousePool.sol:1183-1185`) and rolls that body back. The prank is how the test keeps the post-reconcile storage.

Run from `packages/perps` on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No `packages/perps/src` edits. Default Foundry profile. Plain `forge test`:

```
forge test --match-contract 'GhostMemoPocTest|GhostUpgradeTest|JudgeGhostUpgradeTest' -vv
```

Expected: 3 suites, 10 tests passed, 0 failed, 0 skipped. `test_poc_recapMemoGhost` passes its assertions. It does not revert.

A verification run of that command on 2026-11-02, from `packages/perps`, produced: 10 passed, 0 failed, 0 skipped. The impact test's logs matched the block below, and its post-reconcile assertions passed (raw 0, both principals 0, book equal to the memo).

| Suite | Result |
|---|---|
| `GhostMemoPocTest` | 2 passed, 0 failed |
| `GhostUpgradeTest` | 4 passed, 0 failed |
| `JudgeGhostUpgradeTest` | 4 passed, 0 failed |

`GhostUpgradeTest` and `JudgeGhostUpgradeTest` plant `traderClaimBalanceAssets` with `stdstore`. They are controls with the cash still present, not the reachability proof; that is `test_poc_recapMemoGhost` alone.

## Steps in `test_poc_recapMemoGhost`

1. `_fund` Alice and Carol. `_landLiveMonday` moves to a live Monday outside the oracle-frozen window and the FAD window. `_commitStrict` / `_executeStrict` open Carol's short at 0.50 and Alice's long at 1.50, each sized to about 60% of pool cash. The test asserts both positions are open and there is no recap memo.

2. The same helpers close the long at the floor and the short at the cap, short closed second. Its payout is larger than the cash that close left, so the engine records Carol's trader claim instead of paying it. The test checks `totalTraderClaimBalanceAssets > pool.totalAssets()`, that the claim balance sits on Carol, and that `pendingRecapitalizationAssets == 0`.

3. The test contract, which is the engine owner, approves and calls `engine.recapitalizePool(claims - cash)`. Assertions: the memo equals that gap, raw cash rises by the gap, the book rises by the same gap, `totalAssets()` covers the claim, and book-minus-raw does not widen on the recap itself.

4. `vm.prank(CAROL)` then `engine.settleTraderClaim(CAROL)`. The assertions here: claim paid in full, memo unchanged, book unchanged, raw down by the full claim, and book-minus-raw widened by that payout to at least the memo. `clearDegradedMode` succeeds. `canAcceptTrancheDeposits(true)` and `canAcceptTrancheDeposits(false)` both come back false. I record those views for completeness; they are not the claimed impact.

5. `vm.prank(juniorVault)` then `pool.reconcile()`, the shortcut described above. This time the assertions are: memo unchanged, raw custody 0, junior principal 0, senior principal 0, `unassignedAssets` 0, and the book equal to the memo. Both deposit views stay false.

6. Warp seven days, then `juniorVault` calls `pool.reconcile()` again. Assertions: the memo still holds the same value, both principals stay 0, raw is unchanged, the book is still the memo, and the junior deposit view stays false.

Logged by that test (6-decimal USDG atoms):

```
claims paid from recap cash (atoms): 600889380000
unbacked recap memo (atoms): 200079347021
book-raw while memo unbacked (atoms): 1202079347021
memo still pending after reconcile (atoms): 200079347021
```

In USDG (divide by `1e6`): the entitled claim paid is 600,889.380000. The memo that remains pending is 200,079.347021. The logged book-minus-raw of 1,202,079.347021 is the pre-reconcile gap, immediately after `payOut`. It exceeds the memo by exactly 1,002,000.000000, which is principal that reconcile has not written down yet. After the committed reconcile the test asserts raw is 0, both principals are 0, and the book equals the memo, 200,079.347021. The memo is 200,079.347021 before and after both reconciles.

The second test in the same contract, `test_poc_fullClaimsRecap_seniorReceivesOwnerSurplus`, is a negative control. It uses the same router orders, then `recapitalizePool(claims)` rather than `claims - cash`, then `settleLpEpoch`. It is not the impact of this report. Its logs are quoted under Impact so a full-suite run is accounted for.

## What the bounding suites measure

`JudgeGhostUpgradeTest` (4/4), claim seeded in storage:

- `test_chargeIsRecapFree_unbackedClaimAloneWritesJuniorDown`: with no recap, reconcile reduces junior principal by 50,000.000000 USDG (`charge (no recap) 50000000000`). Book equals cash afterward.
- `test_partialClaim_chargeEqualsClaimNotMemo`: recap 50,000 USDG, claim paid 20,000 USDG. Junior is reduced by 20,000.000000 USDG (`charge (20k claim vs 50k memo) 20000000000`). `unassignedAssets` becomes 50,000.000000 USDG. Book equals cash afterward.
- `test_backedClaimEquilibrium_recapCausesNoJuniorCharge`: junior principal stays 1,000,999.990868 USDG (`junior0` and `juniorAfter` both `1000999990868`).
- `test_recapWithUnsettledClaim_chargeIdenticalToRecapFreeCase`: the junior reduction matches the no-recap arm. The claim is still outstanding and the cash is still in the pool.

`GhostUpgradeTest` (4/4) checks that a stranger cannot queue a recap, that a backed recap does not by itself reduce junior principal, and that on the seeded healthy-pool ordering (cash still present) reconcile routes the memo into `unassignedAssets` and book equals raw (`bookAfter` and `rawAfter` both `1002000000000`). That ordering is not `test_poc_recapMemoGhost`. On the end-to-end path the memo stays in `pendingRecapitalizationAssets` and is not moved into `unassignedAssets`. Junior principal on that path is written to zero by the loss waterfall, not by the memo.

# Impact

Numbers below are the `test_poc_recapMemoGhost` logs unless a bounding test is named.

- Custody identity. Immediately after the claimant is paid, and before reconcile, the pool book exceeds raw custody by 1,202,079.347021 USDG. Of that gap, 200,079.347021 USDG is `pendingRecapitalizationAssets`. The other 1,002,000.000000 USDG is principal reconcile has not written down yet. The committed reconcile writes both principals to 0, leaves raw at 0, leaves `unassignedAssets` at 0, and leaves the memo at 200,079.347021 USDG. The book is then that memo. A second reconcile seven days later still reports the same memo, the same raw balance of 0, and the same book.
- Deposit views. Immediately after the payout, and again after both reconciles, `canAcceptTrancheDeposits` is false for senior and for junior. Recorded so the full picture is visible; not claimed as a freeze. On this trace the zero-cash waterfall also sets projected principal to 0 while share supply remains (`HousePool.sol:494-498`), which closes the same views without the memo. The nonzero-pending predicate (`HousePoolTrancheGateLib.sol:32-34`) is an additional stored input, not the impact row. `assignUnassignedAssets` (`HousePool.sol:660-676`) assigns `unassignedAssets` only, after a reconcile. This ordering leaves the memo pending, so that owner call has no unassigned balance to clear.
- Who is paid. 600,889.380000 USDG is paid to Carol, the short whose close created the claim, via the clearinghouse. That payment is the senior claim. It is not an unprivileged withdrawal of LP principal.
- Junior principal. Where cash is still present, the junior reduction equals the claim, not the memo: 20,000 USDG on a 50,000 USDG memo, and 50,000 USDG with no recap at all (`JudgeGhostUpgradeTest`). On the end-to-end path the committed reconcile sets junior principal and senior principal to 0 because distributable cash is 0. That wipe is the ordinary loss waterfall after the claim took custody. The memo is not the amount of the wipe, and the memo is not absorbed. The test asserts both principals are 0 and the memo is unchanged.
- Free-cash view. `getFreeAssets` reserves pending claimant inflows (`HousePool.sol:1230-1236`). After the payout it subtracts a memo whose tokens have already left. That is the same unbacked memo, visible in a second getter. It is not a separate bug.

## Out of this report

- No claim that balances already in the tranches are frozen for 1 hour or for 24 hours. `canAcceptTrancheDeposits` is false on this trace. New deposits are not funds the protocol already holds, and the zero-cash principal wipe closes those views as well.
- No claim that a senior LP is paid junior principal. `test_poc_fullClaimsRecap_seniorReceivesOwnerSurplus` logs claims `606886280000`, cash before the recap `404810133109`, shortfall `202076146891`, `seniorAfter 404810133109` (equal to that pre-recap cash), `juniorAfter 0`, and a 10,000 USDG senior deposit withdrawing `368008906370`. That withdrawal is that LP's share of the cash the payout left behind. That cash is the owner's surplus over the 202,076.146891 USDG shortfall that already existed because claims exceeded cash. Junior principal was already behind that shortfall before the owner sent funds.
- No claim that the owner must send a second recap equal to the first, or that junior is charged the full memo.

## Measured state versus the impact rows

The cash-still-present controls show the junior reduction, when cash remains, equals the claim and also occurs with no recap. The end-to-end path writes both principals to zero because the claim took all custody. It does not write junior down by the memo, and it does not absorb the memo. Existing funds are not shown locked for the Medium row “Temporary freezing of funds for at least 1 hour,” or for the High row “Temporary freezing of funds for at least 24 hours.” The tokens that move are the entitled claim, paid to the clearinghouse. The companion test's senior exit is the owner's surplus over a shortfall that predated the recap.

The Low row (“Contract fails to deliver promised returns, but doesn't lose value”) is not claimed: the entitled claim is paid, and the principal wipe is the ordinary zero-cash loss waterfall rather than a missed return equal to the memo. The category used here is the Insight row “Security best practices.” After a documented owner recap and a documented claimant settlement, the pool stores a 200,079.347021 USDG memo against raw custody of 0. Before reconcile the book exceeds raw by 1,202,079.347021 USDG. After the committed reconcile the book equals the memo. That stored memo is reachable with router orders at the in-scope commit. It is not MED-1 or SC-L1.

# Mitigation

Decrement the pending buckets inside `payOut` by the cash that call actually transfers, using the same order `capClaimantBuckets` already uses (recapitalization, then trading revenue). The claimant is still paid in full. Only the memo shrinks, and only by the amount of that memo's cash which left.

```solidity
function payOut(address recipient, uint256 amount) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    uint256 fromRecap = amount < pendingRecapitalizationAssets ? amount : pendingRecapitalizationAssets;
    uint256 rest = amount - fromRecap;
    uint256 fromRevenue = rest < pendingTradingRevenueAssets ? rest : pendingTradingRevenueAssets;
    pendingRecapitalizationAssets -= fromRecap;
    pendingTradingRevenueAssets -= fromRevenue;
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

Epoch settlement already calls `_reconcile` before it decreases `accountedAssets` (`HousePool.sol:835`, `:861`, `:887`). The missing update is `payOut`.

If a later cash exit is added, keep one helper that both `payOut` and that exit call: reduce the pending buckets, then require `pendingRecapitalizationAssets + pendingTradingRevenueAssets` is still covered by the raw balance those buckets are supposed to represent. A view-only cap would hide the memo in getters and would leave the stored bucket wrong. The stored bucket is what the deposit gate ORs into `projectedUnassignedAssets`.
