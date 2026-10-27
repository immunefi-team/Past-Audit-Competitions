# 88534 sc high checkpoint accounting desync in the payout function leads to protocol insolvency a withdrawal at the exact period start collapses the exposure cap

**Submitted on Aug 13th 2026 at 16:44:17 UTC by @Cyberjaytech for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88534
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`FirelightVault.payout()` sizes every cover claim from two views of the *same* capital that contradict each other: `withdrawAssets[capturePeriod + 1]` counts a pending withdrawal as **still exposed** to slashing, while the exposure cap `totalAssetsAt(capturePeriodStart)` has already **removed** it. A withdrawal mined in the block whose timestamp is exactly the period start lands on both sides of that contradiction, and the cap wins — so a fully collateralised claim is paid a fraction of what the vault holds and has itself classified as slashable. In production this means a cover buyer receives 10% of a valid claim while the vault visibly holds 100% of the collateral; two periods later the un-slashed capital leaves the vault and the shortfall becomes unrecoverable. No privileged role misbehaves at any point, and the bug also fires by accident whenever an ordinary user's withdrawal lands in a boundary block.

## Vulnerability Details

**Asset:** `contracts/core/FirelightVault.sol` **Commit:** `v1_audit_ready` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

### The contradiction

`payout()` (lines 892-966) computes what it will pay from two figures that are meant to describe the same capital:

```solidity
// line 911 — the exposure CAP
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);

// line 916 — withdrawals the function itself treats as STILL EXPOSED
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

// lines 926-927
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The contract's own comment at lines 914-915 states the intent without ambiguity:

> *"Withdrawals requested during the capture period are assigned to capturePeriod + 1 and remain payable while payout is allowed."*

So `withdrawAssets[capturePeriod + 1]` is, by design, capital that has **not** escaped its slashing obligation. Yet the cap on line 911 is derived from a checkpoint trace in which that same capital has already been deducted. `Math.min` then selects the smaller of the two — the cap — and the capital the function just declared exposed is never used to pay the claim it was still backing.

### Why a boundary block triggers it

Two writes collide on the timestamp `S` (the period start).

**1 — The withdrawal is booked one period forward.** `_requestWithdraw()` computes the target bucket from the *current* period:

```solidity
// line 1008
uint256 period = currentPeriod() + 1;
...
// line 1015
withdrawAssets[period] += assets;
```

For a block whose timestamp equals `S`, `currentPeriod()` has **already** rolled over to the new period `P`. The assets therefore land in bucket `P + 1` — precisely the bucket `payout()` treats as exposed on line 916.

**2 — The same call writes a checkpoint keyed at `S`.** `withdraw()` ends with:

```solidity
// line 775 (redeem() does the same at line 741)
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

and `_logTrace` pushes with the block timestamp as the key, storing the **post-withdrawal** total:

```solidity
// lines 1051-1055
uint48 ts = Time.timestamp();
...
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

**3 — The lookup is inclusive.** `totalAssetsAt()` is a plain `upperLookupRecent`:

```solidity
// lines 456-457
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

An incident captured at `S` resolves through `_periodAndStartAt()` (line 1123) to `capturePeriodStart == S`. Because the lookup includes the key `S`, it finds the checkpoint the withdrawal just wrote and returns the **reduced** figure.

Net result: the withdrawal is added to the payable set and subtracted from the cap, in the same function call.

### The one-second negative control

A withdrawal at `S + 1` writes its checkpoint at key `S + 1`, which `upperLookupRecent(S)` never sees. The cap keeps the full pre-withdrawal figure and the claim is paid in full. Identical vault contents, identical payable set, one second apart, 10x difference in what the vault pays:

| whale withdraws at | `withdrawAssets[P+1]` | `totalAssets()` | payable set   | cap `totalAssetsAt(S)` | vault ERC20 balance | **paid**          |
| ------------------ | --------------------- | --------------- | ------------- | ---------------------- | ------------------- | ----------------- |
| `S + 0`            | 900,000,000           | 100,000,000     | 1,000,000,000 | **100,000,000**        | 1,000,000,000       | **100,000,000**   |
| `S + 1`            | 900,000,000           | 100,000,000     | 1,000,000,000 | 1,000,000,000          | 1,000,000,000       | **1,000,000,000** |

This isolates the withdrawal's block timestamp as the single causal input. Both rows are reproduced in the PoC.

### Recommended fix

The invariant to restore: capital that `payout()` counts as exposed must also be counted inside the exposure cap. The cleanest correction is to read the cap from strictly *before* the period start, so nothing that happens inside the boundary block can move it:

```solidity
// FirelightVault.sol:911
uint256 assetsAtCapturePeriod = capturePeriodStart == 0
    ? totalAssetsAt(0)
    : totalAssetsAt(capturePeriodStart - 1);
```

Do **not** fix this in `_requestWithdraw` by re-bucketing the boundary withdrawal into `P`: that would let capital escape slashing by withdrawing exactly at the boundary, turning an under-payment bug into an over-escape bug. The bucketing on line 1008 is correct; the cap is what is wrong.

### Not a declared known issue

The program declares *"Aggregate claims are uncapped on-chain; MVP depends on trusted roles and approval processes."* That note concerns claims being **over**-paid in aggregate, and a dependency on trusted roles. This report is the opposite failure: a single claim is **under**-paid, below collateral the vault holds and has itself classified as slashable — and **no role misbehaves**. The curator, assessor and payout roles all act correctly, and the vault still refuses to pay.

The other two declared known issues concern the first-loss buffer (unescrowed, and lacking a price oracle). The buffer plays no part in this report; the defect is entirely inside the vault's own accounting.

## Impact Details

### Funds at risk

Every cover claim whose capture period begins in the same block as any withdrawal request. The shortfall equals the size of that boundary withdrawal, bounded only by the vault's TVL — there is no ceiling in the code that limits how much of a claim can be refused this way.

In the PoC, a 1,000-unit claim against a vault holding exactly 1,000 units is paid **100**: a **900-unit shortfall, 90% of the claim**, on a claim that was fully collateralised. Scale is linear in the size of the boundary withdrawal, so a whale-sized withdrawal can suppress essentially the entire claim.

Two parties lose at once:

* **The cover buyer** receives a fraction of a claim the protocol was funded to cover in full. This is the exact obligation the product exists to honour.
* **The remaining LPs** carry a vault that cannot pay its cover, while the withdrawing LP escapes the slashing it was booked for and collects the underwriting yield anyway.

### Deliberate path — no privileged role required

Preconditions: none beyond being an LP. Period boundaries are public and deterministic — `currentPeriodEnd()` is a view function and the boundary is `epoch + n * duration`.

1. A large LP reads `currentPeriodEnd()` to obtain the next period start `S`.
2. The LP submits `withdraw()` targeted at the block whose timestamp is `S`. On a \~12s chain with a 7-day period this is a single-block target, reachable by ordinary transaction timing or by paying for inclusion.
3. Their assets are booked into `withdrawAssets[P + 1]` — the bucket the protocol considers still slashable. On paper the LP remains exposed.
4. Any incident captured at `S` during period `P` is now capped at `totalAssetsAt(S)`, which excludes the LP's assets.
5. `payout()` transfers only the reduced amount. The LP's capital is never touched.
6. Two periods later the LP calls `claimWithdraw()` and walks away with the collateral that should have absorbed the claim — **810,000,000 of 900,000,000** in the PoC, the small remainder being their pro-rata share of the reduced payout.

The LP has taken the yield of underwriting cover without ever bearing the risk.

### Accidental path — no attacker at all

The bug fires on its own whenever any user's withdrawal happens to be mined in a boundary block and an incident is captured at that period start. Every period boundary is a fresh opportunity, and the busier the vault, the more likely a boundary block carries a withdrawal. Frequency scales with adoption.

### Why the loss is permanent

* `payout()` may only run during the capture period or the one after it (`_isPeriodInPayoutWindow`, line 1130). Once that window closes, the claim can never be topped up.
* The un-slashed collateral becomes claimable two periods later and leaves the vault. The PoC's second test confirms the whale receives all 810,000,000.

By the time the shortfall is noticed, the capital that should have covered it is gone. There is no recovery path and no role that can undo it.

### Mapping to the selected in-scope impact

**Protocol insolvency.** At the end state, the cover buyer remains a creditor for 900 units while the vault retains 90, with the payout window closed and the collateral withdrawn. The protocol's liability for a claim it was fully collateralised to cover exceeds the assets remaining to meet it, with no mechanism to recover. That is insolvency in effect — deferred by two periods, but unconditional once the boundary block occurs.

## References

Audited source, `v1_audit_ready` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`:

* [`payout()` — the two contradicting reads](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L966)
* [Line 911 — exposure cap `totalAssetsAt(capturePeriodStart)`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911)
* [Line 916 — withdrawals treated as still exposed](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L916)
* [Lines 926-927 — `Math.min` applies the cap](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927)
* [`_requestWithdraw()` — bucketing at `currentPeriod() + 1`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028)
* [Line 775 — `withdraw()` pushes the post-withdrawal checkpoint](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L775)
* [Line 741 — `redeem()` reaches the same push](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L741)
* [`_logTrace()` — timestamp-keyed push](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1044-L1056)
* [`totalAssetsAt()` — inclusive `upperLookupRecent`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L456-L457)
* [`_periodAndStartAt()` — resolves `capturePeriodStart`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1123-L1128)
* [`_isPeriodInPayoutWindow()` — the window that closes](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1130-L1132)

OpenZeppelin `Checkpoints.upperLookupRecent` — inclusive upper-bound semantics (`value at the highest key <= timestamp`), the property this report turns on: <https://docs.openzeppelin.com/contracts/5.x/api/utils#Checkpoints>

## Link to Proof of Concept

<https://gist.github.com/Spunkli/1a37be3f059f649436833504d3fc5d2e>

## Proof of Concept

## Proof of Concept

## How to run

The PoC is a Hardhat test against the audited repository at `v1_audit_ready` (`42f9ea5e43d88b35197b901acc2a8c24b314fa62`). It runs entirely on the local Hardhat network — no forked state, no live transaction, no external RPC.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight
cd audit-comp-firelight
git checkout v1_audit_ready
npm install
# save the file below as test/zz_report_f4.js
npx hardhat test test/zz_report_f4.js
```

Result: **2 passing**.

## What the test proves

Two runs of an identical scenario. Two LPs deposit 900 and 100 units (1,000 total collateral). A 1,000-unit cover claim is captured at the period start `S`. The **only** difference between the runs is the block timestamp of the whale's withdrawal:

| whale withdraws at | `withdrawAssets[P+1]` | `totalAssets()` | payable set   | cap `totalAssetsAt(S)` | vault ERC20 balance | **paid**          |
| ------------------ | --------------------- | --------------- | ------------- | ---------------------- | ------------------- | ----------------- |
| `S + 0`            | 900,000,000           | 100,000,000     | 1,000,000,000 | **100,000,000**        | 1,000,000,000       | **100,000,000**   |
| `S + 1`            | 900,000,000           | 100,000,000     | 1,000,000,000 | 1,000,000,000          | 1,000,000,000       | **1,000,000,000** |

The `S + 1` row is the negative control: it isolates the block timestamp as the single causal input. Vault contents are byte-identical in both rows (1,000,000,000 ERC20 held), and `payout()` itself computes the same payable set (1,000,000,000) in both. Only the cap diverges — and the claim is paid 10x less.

Test 2 then proves the loss is permanent: after the capped payout, 810,000,000 remains in the withdrawal bucket, and two periods later the whale calls `claimWithdraw()` and receives all 810,000,000. The collateral that should have absorbed the claim has left the vault, and the payout window has closed.

## Test output

```
  Firelight F4 — period-start withdrawal collapses the payout exposure cap
   [EXACT S] withdraw mined at periodStart +0  <-- EXACT PERIOD START
        withdrawAssets[2]           : 900000000   (payout() treats this as exposed)
        totalAssets()               : 100000000
        activePayableAmount         : 1000000000
        totalAssetsAt(periodStart)  : 100000000   <-- EXPOSURE CAP
        vault ERC20 balance         : 1000000000
        payout(1000e6) actually PAID: 100000000

   [S + 1s ] withdraw mined at periodStart +1
        withdrawAssets[2]           : 900000000   (payout() treats this as exposed)
        totalAssets()               : 100000000
        activePayableAmount         : 1000000000
        totalAssetsAt(periodStart)  : 1000000000   <-- EXPOSURE CAP
        vault ERC20 balance         : 1000000000
        payout(1000e6) actually PAID: 1000000000

   ==> identical state, identical payable set (1000000000),
       claim paid 100000000 instead of 1000000000.
       900000000 of a valid cover claim is refused
       while the vault demonstrably holds the collateral.
    ✔ same vault contents, same payable set: 100e6 paid instead of 1000e6

   after the capped payout: withdrawAssets[2] = 810000000
   whale claimed          : 810000000 / 900000000 originally deposited

   ==> the shortfall is permanent: the collateral that should have
       absorbed the claim has left the vault.
    ✔ the un-slashed collateral leaves the vault intact two periods later

  2 passing
```

## Test source — `test/zz_report_f4.js`
