# 89428 sc high inclusive period start snapshot in firelightvault lets any liquidity provider cap incident payouts to near zero and destroy 80 of a period s underwriting capacity

**Submitted on Aug 19th 2026 at 09:21:41 UTC by @emilh for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89428
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`FirelightVault.totalAssetsAt(periodStart)` resolves through `Checkpoints.upperLookupRecent`, which returns the last checkpoint whose key is **lower or equal** to the search key. A withdrawal mined in the boundary block therefore writes its own checkpoint at exactly `periodStart` and is included in that period's own snapshot, even though the same withdrawal is bucketed as belonging to the period. Two independent consumers read that value: `FirelightVault.payout` uses it as the ceiling on incident payouts, and `CoverOrderAllocator._computeAvailableCapacity` uses it as the ceiling on cover sales. A liquidity provider with no privileged role can therefore either collapse the payout ceiling, leaving an approved cover claim permanently unpayable while its own capital stays intact, or collapse the underwriting ceiling, preventing the protocol from selling any cover for a full period while the collateral backing it sits untouched in the vault.

## Vulnerability Details

**Root cause: the snapshot lookup includes the boundary block.**

```solidity
// contracts/core/FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

```solidity
// contracts/core/lib/Checkpoints.sol:221-224
function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
    uint208 idx = self._trace.upperLookupRecent(key);
    return idx > 0 ? self._values[idx] : 0;
}
```

A checkpoint written at `key == periodStart` is returned by `totalAssetsAt(periodStart)`.

**Every withdrawal writes such a checkpoint.** `withdraw` and `redeem` both push the post-withdrawal figure at `block.timestamp`:

```solidity
// contracts/core/FirelightVault.sol:741 (redeem) and :775 (withdraw)
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

`totalAssets()` is `super.totalAssets() - pendingWithdrawAssets` (`:427-429`), and `_requestWithdraw` adds the requested amount to `pendingWithdrawAssets`, so the pushed value is already reduced by the withdrawal.

**But the bucketing places the same withdrawal inside the period.** `_requestWithdraw` assigns it to `currentPeriod() + 1`:

```solidity
// contracts/core/FirelightVault.sol:1008
uint256 period = currentPeriod() + 1;
```

and `periodAtTimestamp` uses floor division, so at `block.timestamp == periodStart` of period C the vault already reports `currentPeriod() == C`:

```solidity
// contracts/core/FirelightVault.sol:316-320
return periodConfiguration.startingPeriod
     + _timestampSinceEpoch(timestamp, periodConfiguration.epoch) / periodConfiguration.duration;
```

The withdrawal therefore lands in `withdrawAssets[C + 1]`, which `payout` explicitly treats as still exposed.

**Consumer 1: the incident payout ceiling.**

```solidity
// contracts/core/FirelightVault.sol:911
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
// contracts/core/FirelightVault.sol:916
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
// contracts/core/FirelightVault.sol:927
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The assets remain formally inside the payable set, while the ceiling that governs how much of that set can be drawn is computed as if they had already left.

**This is not the intended trade-off.** The contract's own comment states the opposite intent:

```solidity
// contracts/core/FirelightVault.sol:914-916
// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.
```

and at `:918-920`, that withdrawals during the following period stay payable "otherwise exposed assets could escape by withdrawing first". The boundary block defeats exactly that protection.

**Consumer 2: the period's underwriting ceiling.** The same snapshot gates cover sales:

```solidity
// contracts/core/CoverOrderAllocator.sol:711
$.vault.totalAssetsAt($.vault.currentPeriodStart())
// contracts/core/CoverOrderAllocator.sol:400-401
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
```

Because the lookup is inclusive, the value governing an entire period is simply whatever the last vault operation at or before the boundary left behind, and it is immutable for the rest of the period: any later deposit writes a checkpoint with a strictly greater key and cannot raise it. A provider whose `redeem` is the last vault operation before a boundary fixes that period's capacity at the post-redeem figure, and re-depositing in the next block restores the live collateral without restoring the snapshot.

Note that this second path does not require any attacker intent. Any sufficiently large redeem that happens to be the last vault operation before a period boundary silently destroys that period's capacity, so the defect also degrades the protocol under ordinary use.

## Impact Details

**Path 1: an approved cover claim becomes permanently unpayable while the exposed capital escapes.**

Measured on the repository's own fixtures, with a vault of 1000 tokens (whale 900, small LP 100) and an approved incident loss of 500:

|                                  | Honest exit (boundary + 1 second) | Boundary block |
| -------------------------------- | --------------------------------- | -------------- |
| Paid to the cover buyer          | 500                               | 100            |
| Whale's pending bucket remaining | 450                               | 810            |

The buyer is short 400 of the 500 it was owed, and the whale avoids 360 of loss it should have absorbed. The only difference between the two runs is a one second shift in when the withdrawal is mined.

The shortfall is unrecoverable. `IncidentManager.approveCurrentAssessment` closes the incident with whatever the vault returns and does not retry:

```solidity
// contracts/core/IncidentManager.sol:432-434
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
```

`_validateIncidentPeriod` then blocks any new incident for that period once the payout window closes, so the difference cannot be claimed later.

**Path 2: the cover market cannot open for an entire period.**

Measured: live `totalAssets` 100,000 fXRP before the boundary, snapshot pinned at 20,000 after the boundary redeem, live value restored to 100,000 when the provider re-deposits, snapshot still 20,000. On-chain maximum capacity is 40,000e18 canonical against a live capacity of 200,000e18, so **80% of the period's underwriting capacity is destroyed while the collateral is fully present**, and `commitAllocation` sized against the real collateral reverts `MatchingCapacityOverflow` for the whole period (one week at the configured duration). Matched orders for that period expire `PENDING`.

**Preconditions.** The attacker holds a material share of the vault; the reduction in the ceiling equals the amount withdrawn. Period boundaries are public and deterministic (`periodConfigurations` is a public array, `currentPeriodEnd()` is a public view), and no incident needs to exist yet, so the position can be taken pre-emptively at any boundary and retried at the next one. Path 2 does not even require landing in one specific block: being the last vault operation before the boundary is sufficient. No privileged role is involved in either path; the trigger is the permissionless `withdraw` / `redeem`.

**Recommended fix.** Read the snapshot as of the last instant before the period opens, at both call sites, so that no operation belonging to period C can influence period C's own snapshot:

```solidity
// contracts/core/FirelightVault.sol:911
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1);
// contracts/core/CoverOrderAllocator.sol:711
$.vault.totalAssetsAt($.vault.currentPeriodStart() - 1)
```

A dedicated test (`C01_fix_check.js`, included in the PoC) confirms this closes the hole without regressing honest pre-period exits: a withdrawal requested at `capturePeriodStart - 1` is bucketed into `withdrawAssets[C]`, which `payout` never draws from, and is correspondingly absent from the `capturePeriodStart - 1` snapshot, so inclusion in the payable set and inclusion in the exposure ceiling coincide for every timestamp.

## References

* `contracts/core/FirelightVault.sol:456-458` (`totalAssetsAt`)
* `contracts/core/lib/Checkpoints.sol:221-224` (inclusive `upperLookupRecent`)
* `contracts/core/FirelightVault.sol:741`, `:775`, `:1044-1057` (checkpoint written on every withdrawal)
* `contracts/core/FirelightVault.sol:1008` (bucketing to `currentPeriod() + 1`)
* `contracts/core/FirelightVault.sol:316-320` (floor division places the boundary inside the period)
* `contracts/core/FirelightVault.sol:911`, `:916`, `:927`, `:914-920` (payout ceiling and the stated intent)
* `contracts/core/CoverOrderAllocator.sol:711`, `:400-401` (underwriting ceiling)
* `contracts/core/IncidentManager.sol:432-434` (incident closes without retry)
* CWE-193: Off-by-one Error

## Link to Proof of Concept

<https://gist.github.com/emilhuseyn/a7bf97c27b86dfdd3cdc2205bb6ad7cb>

## Proof of Concept

## Proof of Concept

All tests run against the competition branch with no modification to any contract, using the repository's own fixtures.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready
npm install
# place the four files from the Gist into test/poc/
npx hardhat test test/poc/C01_boundary_block_snapshot.js   # 3 passing
npx hardhat test test/poc/C01_fix_check.js                 # 3 passing
npx hardhat test test/poc/S5_capacity_boundary_dos.js      # 2 passing
```

### Path 1: collapsing the incident payout ceiling

Step by step:

1. A whale (900 of the vault's 1000 tokens) and a small LP (100) are deposited. No incident exists yet.
2. The whale reads the next period boundary from the public `currentPeriodEnd()` and submits `withdraw()` so that it is mined in the block whose `block.timestamp` equals that boundary, which is the start of period C.
3. `_requestWithdraw` assigns the request to `withdrawAssets[C + 1]`, because `currentPeriod()` already reports C at that timestamp. `payout` treats that bucket as still exposed.
4. The same transaction pushes a `_traceTotalAssets` checkpoint with key exactly `capturePeriodStart` and the reduced value, because `totalAssets()` is net of `pendingWithdrawAssets`.
5. An incident is later captured in period C and approved for a loss of 500.
6. `payout` computes `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)`, which now returns the post-withdrawal figure, and caps the payment at it.
7. The cover buyer is paid 100 instead of 500. `IncidentManager` closes the incident and the remaining 400 can never be claimed. The whale's pending bucket keeps 810 instead of 450.

Test output:

```
  C-01 payout cap bypass via period-boundary block
    BASELINE: withdrawing one second after the boundary leaves the full exposure payable
    EXPLOIT: withdrawing inside the boundary block collapses the payout cap
      buyer paid   honest=500000000  attack=100000000
      whale bucket honest=450000000  attack=810000000
      buyer shortfall=400000000  whale loss avoided=360000000
    QUANTIFIED: the whale avoids 360 tokens of loss and the buyer loses 400

  3 passing
```

The baseline case performs the identical withdrawal one second after the boundary and the buyer is paid in full, isolating the boundary block as the sole cause.

### Path 2: destroying the period's underwriting capacity

Step by step:

1. The vault holds 100,000 fXRP of live collateral.
2. A large provider calls `redeem()` as the last vault operation before a period boundary. Its checkpoint becomes the one returned by `totalAssetsAt(currentPeriodStart())` for the whole of the new period.
3. In the next block, already inside the new period, the provider deposits the capital back. Live `totalAssets()` returns to 100,000, but the snapshot cannot be raised, because the new checkpoint has a strictly greater key.
4. The off-chain matcher sizes an allocation commitment against the real, live collateral and calls `commitAllocation`.
5. `_validateMatchingCapacity` compares it against `_computeAvailableCapacity`, which is still derived from the pinned snapshot, and reverts `MatchingCapacityOverflow`.
6. No cover order can be committed or settled for that period. Matched orders expire `PENDING`.

Test output:

```
  S5 / cover-market DoS via the period-start snapshot
    live totalAssets before boundary: 100000000000
    totalAssetsAt(periodStart) after the boundary redeem: 20000000000
    live totalAssets after the whale re-deposits : 100000000000
    but totalAssetsAt(periodStart) is unchanged  : 20000000000
    matcher capacity (live)     : 200000000000000000000000
    on-chain max capacity       : 40000000000000000000000
    capacity destroyed for the whole period: 80%
    commitAllocation with the real, live-collateral capacity REVERTS

  2 passing
```

### Fix validation

`C01_fix_check.js` applies the proposed `capturePeriodStart - 1` read and asserts three properties: the boundary block withdrawal no longer moves the ceiling, an honest exit made before the period opens is still excluded from it, and that same exit is bucketed outside the payable set, so the two mechanisms agree at every timestamp.

```
  C-01 remediation check
    boundary-1 snapshot is immune to the boundary-block withdrawal
    boundary-1 snapshot still excludes an honest exit made before the period opens
    that pre-period exit is bucketed OUTSIDE the payable set, confirming the two are consistent

  3 passing
```

The full test sources are in the linked secret Gist: `C01_boundary_block_snapshot.js`, `C01_fix_check.js`, `S5_capacity_boundary_dos.js`, and the harness `S3_seam_harness.js` that the third test imports.
