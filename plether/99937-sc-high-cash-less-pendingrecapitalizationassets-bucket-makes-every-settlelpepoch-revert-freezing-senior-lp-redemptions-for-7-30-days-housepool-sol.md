# \#99937 [SC-High] Cash-less `pendingRecapitalizationAssets` bucket makes every `settleLpEpoch` revert, freezing Senior LP redemptions for 7–30+ days (`HousePool.sol`)

**Submitted on Nov 15th 2026 at 19:37:30 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99937
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`CfdEngine.recapitalizePool` is the owner's documented exit from degraded mode: it moves USDG into `HousePool` and records it in `pendingRecapitalizationAssets`, a bucket a later reconcile is meant to allocate. That cash counts as pool backing immediately, so traders can be paid from it, but `HousePool.payOut` never decrements the bucket. LP settlement keeps reserving the full bucket after its cash has already left. Free assets fall to 0, no redemption can be funded, and `settleLpEpoch` reverts `HousePool__NoLpEpochProgress` — a revert that also rolls back the reconcile that would have shrunk the bucket, so every subsequent permissionless settlement fails identically. In the PoC, 26,970.30 USDG of Senior LP cash stays frozen across 28 consecutive hourly LP epochs and is still frozen after 30 days, on a flat book with no other claimant.

## Vulnerability Details

The pending-recapitalization bucket is credited when the owner recapitalizes but is never debited when that same cash is paid out to a trader, so the reservation outlives its backing and deadlocks LP settlement.

```solidity
// packages/perps/src/HousePool.sol#L1787-L1803
function _decreasePendingClaimantBuckets(...) internal {
    pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;   // only what a reconcile settles
    pendingTradingRevenueAssets  -= claimantBuckets.revenueAssets;
}
function _recordPendingClaimantInflow(IHousePool.ClaimantInflowKind kind, uint256 amount) internal {
    if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
        pendingRecapitalizationAssets += amount;
    } ...
}
```

```solidity
// packages/perps/src/HousePool.sol#L604-L613  (payOut: pays trader from recap cash, bucket untouched)
function payOut(address recipient, uint256 amount) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) revert HousePool__Unauthorized();
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

LP settlement reserves the **full nominal** bucket, and the no-progress revert discards the reconcile that would have consumed it:
- `HousePool.sol:835` — `_settleLpEpoch` runs `_reconcile`, which settles only `min(bucket, distributable)`; the rest stays pending even with no cash behind it.
- `HousePool.sol:842-843` — `freeAssets = _buildWithdrawalSnapshot(..., _pendingClaimantBucketAssets()).freeAssets`; `_buildWithdrawalSnapshot` (`:1727-1735`) reserves `reservedPendingClaimantAssets` in full. `getSettlementFreeAssets` (`:1242-1255`) does the same.
- `HousePool.sol:878-879` — a Senior backlog skips the Junior phase.
- `HousePool.sol:1183-1185` — `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` when no epoch is processed, rolling back the whole transaction (including the line-835 reconcile), so the next attempt starts from the same state.

Compare `HousePoolReconcilePlanLib.sol:73-74`, which clips `unassignedAssets` to `distributable`; the pending claimant buckets get no such clipping.

No unprivileged actor can clear it: the other `_reconcile` callers are owner-only (`finalizePoolConfig` `:341`, only on a `seniorRateBps` change `:336`; `assignUnassignedAssets` `:671`; `initializeSeedPosition` `:736`); `reconcile()` (`:1471`) is `onlyVault` with no vault call site in `src/`; and a matured redeem cannot be cancelled (`TrancheVault.sol:894-896`).

Introduced by the fix for **Cecuro LOW-22** ("Governance cannot use the documented non-dilutive recapitalization path"): that fix added `recapitalizePool` with deferred allocation through reconciliation (NatSpec at `CfdEngine.sol:629`), and the deferred allocation is exactly what deadlocks.

### Affected locations
- `HousePool.sol` bucket credit/debit — `packages/perps/src/HousePool.sol#L1787-L1803`
- `HousePool.payOut` (no bucket decrement) — `packages/perps/src/HousePool.sol#L604-L613`
- `HousePool._settleLpEpoch` full-bucket reservation + no-progress rollback — `packages/perps/src/HousePool.sol#L835`, `#L842-L843`, `#L1183-L1185`
- `CfdEngine.recapitalizePool` — `packages/perps/src/CfdEngine.sol#L632-L653`

### Preconditions
- The pool enters degraded mode, the owner recapitalizes the exact documented shortfall `C + L − P` (SECURITY.md:155-162, :1499-1510), then calls `clearDegradedMode` — all within the documented runbook, not a malicious action. A documented trusted action that triggers a bug is in scope per the program rules.
- The recapitalized side then realizes a large win, so the paid-out cash drives the bucket to at least twice the remaining pool cash (`bucket >= 2P`). This needs large cumulative basket moves (see Feasibility). The book ends flat: no trader claim, no liability, nothing unassigned.
- Traders, the LP, and the keeper are ordinary EOAs; no attacker profit is required. No privileged role acts maliciously. No in-scope contract is modified.

### Attack path
1. Pool holds 60,000 USDG Junior + seeds; a Senior LP deposits 40,000 USDG via `requestDeposit`, settled in an epoch.
2. Two traders open 900 LONG / 900 SHORT at 1.00 (the LONG over-margined so it is never liquidatable; keeper inaction plays no part).
3. Basket → 1.90; the SHORT closes in profit; the payout pushes `E < L`; degraded mode latches.
4. Owner recapitalizes exactly `C + L − P = 67,498.16` USDG, then `clearDegradedMode`.
5. Basket → 0.30; the LONG closes in profit; the recap cash is paid to the trader. Book is now flat: pool cash 26,970.30 USDG, `pendingRecapitalizationAssets` still 67,498.16 USDG, `getFreeAssets() == 0`.
6. The Senior LP requests full redemption. A keeper calls `router.settleLpEpoch` at each of 28 consecutive hourly epochs. All 28 revert `HousePool__NoLpEpochProgress`; Senior claimable stays 0; `cancelRedeemRequest` reverts.

## Impact Details

This is **Temporary freezing of funds for at least 24 hours** because 26,970.30 USDG of Senior LP cash (and Junior redemptions queued behind it) cannot be settled for 28 consecutive hourly epochs and remains frozen after 30 days, with no permissionless exit.

**Funds at risk:** 26,970.30 USDG of Senior LP cash in the PoC — on a flat book with no claims, no liability and nothing unassigned, so all of this cash belongs to the LPs. Every Senior redeemer is affected; Junior redemptions queue behind the Senior backlog; new deposit requests also revert while the state persists. The figure is the pool cash with no other claimant; it scales with real pool size. USDG is the 6dp settlement token (par $1). Competition reward is pool-distributed (Primary $56,000 / All-Stars $16,000 / Podium $8,000, USDC).

**Not Critical:** nothing is stolen, and the freeze is not permanent. Recovery paths exist (all shown in the PoC): (1) owner adds extra capital beyond the shortfall (boundary `bucket − 2P` = 13,557.56 USDG, becomes a Senior windfall, unrecoverable); (2) a 72h-timelocked `finalizePoolConfig` that moves `seniorRateBps` by 1 bp persists a reconcile and unblocks with **no new capital** (`test_CRB1_repeatedRateChanges`: bucket 67,498.16 → 0, Senior paid 26,311.50 USDG, escrow 0); (3) enough new trader losses realized into the pool. The Senior is paid in full once unblocked, and committed deposits stay cancellable, so "permanent freezing" and "insolvency" are not met.

## Recommendation

Do not reserve pending claimant value that has no cash behind it. After settling `min(bucket, distributable)`, zero the residual (its cash has already left the pool), mirroring how `HousePoolReconcilePlanLib.planReconcile` clips `unassignedAssets`:

```diff
 function _applyPendingClaimantBucketsLive(...) internal {
-    if (_pendingClaimantBucketAssets(settleableClaimantBuckets) == 0) {
-        return;
-    }
+    if (_pendingClaimantBucketAssets(settleableClaimantBuckets) != 0) {
         ... existing settlement ...
         _decreasePendingClaimantBuckets(settleableClaimantBuckets);
+    }
+    // Residual exceeds distributable equity: its cash already left the pool; stop reserving it.
+    pendingRecapitalizationAssets = 0;
+    pendingTradingRevenueAssets  = 0;
 }
```

Alternatively, cap the `reservedPendingClaimantAssets` passed to `_buildWithdrawalSnapshot` (`:842-843`, `:1254`) at the cash-backed part of the bucket, and treat a reconcile-only settlement as progress so `_emitLpEpochSettled` (`:1183-1185`) does not discard it. (Sketches; not run against the full suite.)

## References
- Source (in scope): `packages/perps/src/HousePool.sol`, `packages/perps/src/CfdEngine.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Runbook: SECURITY.md:155-162, :1499-1510
- Prior item this fix introduced the bug from: Cecuro LOW-22 (eligible under the program's "fix introduces a new bug" rule)

### Dedupe
- **Cecuro LOW-22** (addressed by adding `recapitalizePool`): this is a **new bug introduced by that fix** — valid under the program rule "the fix at a82d4fef … introduces a new bug".
- Pre-audit SC-L1 ("stays pending, stays reserved") and Cecuro MED-12 ("partial-settlement residuals stay reserved") describe the reservation as intended; neither covers the reservation outliving its cash or the no-progress revert discarding the consuming reconcile (the deadlock). Not MED-24/MED-29 (refund epochs, rounding).

## Link to Proof of Concept

https://gist.github.com/Akashneelesh/af6bc377406b77affa104e9061228545

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit. Files inherit `PocBase` / `LpEpochPocBase` (DEPLOYMENT.md release parameters; `minOpenNotionalAssets` set to the release 1000 USDG through the router's 72h timelock). Only Pyth (`MockPyth`) and USDG mocked; no in-scope contract modified. Keeper pranks marked `// SETUP:` stand in for an untrusted EOA.

**Setup** (from the repo root, `packages/perps`):
```bash
# Main freeze (28/28 settlement reverts) + control:
forge test --root packages/perps --match-path "test/perps/audit/H_Repro1.t.sol" -vv

# Feasibility grid + organic recoverability (grid test needs the raised gas cap):
FOUNDRY_GAS_LIMIT=9000000000000000000 forge test --root packages/perps --match-path "test/perps/audit/H_Repro1b.t.sol" -vv

# Capital-free unblock via one +1bp rate change (fully drains the bucket):
forge test --root packages/perps --match-path "test/perps/audit/CRB1_H1RepeatedRateUnblock.t.sol" -vv
```

**Output** (real runs):
```
# H_Repro1.t.sol
[PASS] test_H1_control_seniorFundedWhenRecapCashNotConsumed() (gas: 15710627)
  control: senior LP received 40000730592
[PASS] test_H1_seniorRedemptionFrozenByPhantomRecapBucket() (gas: 19761498)
  owner recapitalization (exact C + L - P) 67498160919
  after all positions closed
    pool cash P          26970300000
    trader claims C      0
    pendingRecap bucket  67498160919
    getFreeAssets        0
    unassigned           0
  settleLpEpoch attempts over consecutive hourly epochs 28 NoLpEpochProgress reverts 28
  senior LP USDG frozen (pool cash with no other claimant) 26970300000
[PASS] test_H1_unblockNeedsExtraOwnerCapital() (gas: 16222400)
  extra owner capital needed beyond the shortfall 14557560919
Suite result: ok. 3 passed; 0 failed; 0 skipped

# H_Repro1b.t.sol (9 passed) — key lines:
[PASS] test_H1b_extraCapitalBoundaryAndRecoverability()  boundary x* = bucket - 2P 13557560919 ; x = x*: settled false
[PASS] test_H1b_feasibility_singleEpisodeGrid()          smallest cumulative move that freezes (u+d, %) 150
[PASS] test_H1b_feasibility_multiCycle2pct()             2% ping-pong: first frozen at cycle 0 (never)
[PASS] test_H1b_organic_nonCapitalActions()              (b) after 30 days: settled false ; (f) timelocked rate change -> settled true
[PASS] test_H1b_organic_traderLossRevenue()              round 3 cumulative revenue 4541960911 -> settled true
Suite result: ok. 9 passed; 0 failed; 0 skipped

# CRB1_H1RepeatedRateUnblock.t.sol
[PASS] test_CRB1_repeatedRateChanges() (gas: 16398546)
  owner recapitalization (exact C + L - P) 67498160919
    settle ok/fail 2 2 ; bucket 0 ; senior LP received 26311497340 ; senior remaining escrow shares 0
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

PoC files: `packages/perps/test/perps/audit/H_Repro1.t.sol` (main), `H_Repro1b.t.sol` (feasibility/recoverability), `CLH2_EasyPathPhantomRecapFreeze.t.sol` (permanence), `CRB1_H1RepeatedRateUnblock.t.sol` (capital-free unblock); dig copy under `dig/HIGH-phantom-recap-bucket/v2/`.

### Feasibility note (stated for the triager)
Reaching `bucket >= 2P` requires large cumulative basket moves — the smallest freezing path on the single-episode grid is a 150% cumulative move (+90%, then down to 0.60); a 2% ping-pong for 140 cycles never froze. For a six-currency FX basket a ~150% move is an extreme tail. Per the program's "impact first, feasibility only scales payout" rule, this bounds likelihood/payout, not the impact class (a measured 7-day and 30-day LP freeze with no permissionless exit).
