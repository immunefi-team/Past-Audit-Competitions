# 90401 sc high exact boundary withdrawal overwrites opening exposure and underpays valid claims by 50 at launch leverage

**Submitted on Aug 24th 2026 at 20:42:52 UTC by @CertiK for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90401
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief/Intro

An unprivileged shareholder can request a withdrawal in a block timestamped exactly at a Firelight coverage-period boundary. The withdrawal is assigned to the new period's delayed bucket and remains payable for that period, but it also replaces the boundary checkpoint used as the incident payout cap.

With the repository's ordinary `24,000 / 12,000 = 2x` capacity setting, a 50 FXRP withdrawal from a 100 FXRP vault leaves enough active capital for the normal allocator to commit 100 USD of cover 18 seconds later. A valid 100 USD incident captured after settlement then receives only 50 FXRP. `IncidentManager` closes the incident, and proportional payout accounting leaves the other 50 FXRP of value with the shareholder.

### Vulnerability Details

`FirelightVault.withdraw()` requests delayed withdrawal and logs the reduced active-asset total at the current timestamp (`FirelightVault.sol:755-775`, `1044-1055`). `_requestWithdraw()` assigns the request to `currentPeriod() + 1` (`FirelightVault.sol:1008-1019`). At exact start `T` of period N, `currentPeriod()` is already N, so the request goes into bucket N+1. `payout()` expressly includes that bucket as payable for a period-N incident (`FirelightVault.sol:914-926`).

The checkpoint library replaces the latest value when a write has the same timestamp key (`Checkpoints.sol:203-211`). `totalAssetsAt()` uses an inclusive lookup that returns the last checkpoint with key less than or equal to the query (`FirelightVault.sol:456-458`, `Checkpoints.sol:217-223`). The withdrawal at `T` therefore changes `totalAssetsAt(T)` to the post-withdrawal active balance, even though the withdrawal belongs to period N and remains slashable.

The payout code then uses that inclusive value as its opening-exposure cap (`FirelightVault.sol:900-927`):

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

For the proven case:

```
totalAssetsAt(T - 1)       = 100 FXRP
totalAssetsAt(T)           =  50 FXRP
currentActiveAssets        =  50 FXRP
capturePeriodWithdrawals   =  50 FXRP
activePayableAmount        = 100 FXRP
payout request             = 100 FXRP
paidAmount                 =  50 FXRP
```

The later allocation is valid. `CoverOrderAllocator` reads the same 50-unit boundary checkpoint and applies `effectiveLeverage / minCAR` (`CoverOrderAllocator.sol:710-724`). At 2x, 50 active FXRP authorizes 100 USD of matching capacity. The proof keeps the first-loss-buffer balance at zero.

The order settles, and a legitimate incident is captured at `T+30`, after settlement. The normal assessment path requests 100 FXRP. `IncidentManager.approveCurrentAssessment()` records the actual payment and changes the incident to `CLOSED` after its single payout call (`IncidentManager.sol:423-438`). It has no residual payout transition.

The correct opening exposure is the last checkpoint strictly before `T`. Because Firelight's checkpoints use whole-second keys, the correct query is `totalAssetsAt(T - 1)`.

### Attack scenario

1. A 100 USD cover order is pending for period N. The vault has 100 FXRP active at `T-1`.
2. An unprivileged shareholder sends a conditional 50 FXRP withdrawal for exact period start `T`.
3. At `T`, the request enters bucket N+1, the period-N-payable bucket, and overwrites the inclusive checkpoint to 50 FXRP.
4. At `T+18`, the ordinary allocator commits 100 USD. The remaining 50 FXRP at 2x capacity authorizes the commitment.
5. The order settles normally.
6. At `T+30`, a legitimate 100 USD incident occurs and is processed through the normal curator and assessment workflow.
7. The vault has 100 FXRP payable, split between 50 active and 50 delayed, but transfers only 50 FXRP.
8. The incident closes. The shareholder retains 25 FXRP in the reduced withdrawal claim and 25 FXRP of remaining-share value.

### Impact Details

The victim is the valid covered-loss beneficiary represented by Firelight's configured payout receiver. The asset is 6-decimal FXRP at `0xAd552A648C74D49E10027AB8a618A3ad4901c5bE` on Flare.

The PoC proves a 50 FXRP shortfall on a 100 FXRP-equivalent loss. At capacity multiplier L, as much as `1 - 1/L` of attacker-controlled opening capital can escape the valid payout while full-opening-capital cover remains settleable. That is 50% at 2x, 60% at 2.5x, and 80% at the 5x maximum.

At Flare block `68,172,280`, the existing vault proxy reported `55,990,018.761132 FXRP` in `totalAssets()`. At an XRP reference price of $1.48 on 2026-08-24, this was about $82.9 million. The 50% launch-leverage scale ceiling is about 28.0 million FXRP or $41.4 million. This is a forward scale reference, not a claim that one shareholder owns half the vault or that Phase 2 is currently deployed. Actual loss is bounded by attacker-controlled stake, settled cover, and the approved loss.

| Actor                             | Correct result | Vulnerable result |    Delta |
| --------------------------------- | -------------: | ----------------: | -------: |
| Covered-loss beneficiary          |       100 FXRP |           50 FXRP | -50 FXRP |
| Shareholder delayed claim         |         0 FXRP |           25 FXRP | +25 FXRP |
| Shareholder remaining-share value |         0 FXRP |           25 FXRP | +25 FXRP |

This maps to the competition's Critical `Protocol insolvency` impact. The conservative fallback is High because exact-boundary inclusion and shareholder capital are preconditions. Generic CVSS v3.1 is `AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:H/A:H` = 7.4 High.

### Preemptive rebuttals

* Privilege: the shareholder is unprivileged. The allocator, curator, and assessment approver make valid calls through their ordinary lifecycle. No malicious privileged decision is needed.
* Leverage: 2x leverage authorizes the order but does not justify excluding target-period delayed assets. The `T+1` control uses the same leverage and inventory and pays 100 FXRP.
* Partial payout: all 100 FXRP is present in the active and delayed buckets that `payout()` counts. The shortfall comes only from the exact-boundary historical cap. The incident then closes.
* Timing: period starts are known whole-second timestamps. A Coston2 scan found an exact-timestamp block at 344 of 704 sampled Firelight deployment boundaries. A helper can revert unless `block.timestamp == T`, so a skipped timestamp does not change vault state.
* Runner ordering: the proof withdraws first. The allocator commits 18 seconds later, matching the fastest observed frozen-ABI Coston2 delay.
* Known issues: the proof uses zero FLB, one cover, one incident, and one loss. It does not use the unescrowed-FLB or aggregate-claims disclosures.
* Source route: the report targets the exact frozen competition commit. The competition states that the listed Phase 2 contracts are intended for deployment or upgrade and requires a runnable local PoC.

### Recommended fix

Use a strict pre-boundary snapshot in both payout and allocation capacity:

```diff
- uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
+ uint48 snapshot = capturePeriodStart == 0 ? 0 : capturePeriodStart - 1;
+ uint256 assetsAtCapturePeriod = totalAssetsAt(snapshot);

- $.vault.totalAssetsAt($.vault.currentPeriodStart()),
+ uint48 periodStart = $.vault.currentPeriodStart();
+ $.vault.totalAssetsAt(periodStart == 0 ? 0 : periodStart - 1),
```

Add `T-1`, `T`, and `T+1` regression tests that cover withdrawal, allocation, settlement, and one approved incident.

### References

* Frozen commit: <https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62>
* `FirelightVault.sol`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* `CoverOrderAllocator.sol`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* `IncidentManager.sol`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/IncidentManager.sol>
* `Checkpoints.sol`: <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* Firelight scope: <https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/>

## Link to Proof of Concept

<https://gist.github.com/DicksonWu654/8da9a36b81e3ed2906fd72b28be1c851>

## Proof of Concept

## Proof of Concept

Private gist: <https://gist.github.com/DicksonWu654/8da9a36b81e3ed2906fd72b28be1c851>

The gist contains:

* `SUBMISSION.md`, the self-contained report
* `boundary-withdrawal-poc.js`, the Hardhat PoC
* `hardhat-run.log`, a fresh complete run

### Run instructions

```bash
git clone --branch v1_audit_ready --single-branch \
  https://github.com/immunefi-team/audit-comp-firelight.git firelight-boundary-poc
cd firelight-boundary-poc
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci

gh gist clone https://gist.github.com/DicksonWu654/8da9a36b81e3ed2906fd72b28be1c851 poc-bundle
cp poc-bundle/boundary-withdrawal-poc.js test/boundary-withdrawal-poc.js

npx hardhat test test/boundary-withdrawal-poc.js
```

No RPC URL, private key, or environment variable is required.

### What the runner does

1. Deploys the frozen real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager` implementations with a 6-decimal vault asset and one-day periods.
2. Deposits 100 FXRP for the shareholder and creates one 100 USD cover order for the next period.
3. Attack branch: withdraws 50 FXRP at exact start `T`; asserts 100 FXRP at `T-1`, 50 FXRP at `T`, and 50 FXRP in the target-period delayed bucket.
4. Commits 100 USD at `T+18` under 2x capacity with zero first-loss-buffer funds, then settles the cover.
5. Captures a 100 USD incident at `T+30`, after settlement, and approves the assessment through `IncidentManager`.
6. Asserts that 100 FXRP was requested, 50 FXRP was paid, the incident is `CLOSED`, and the shareholder retains 50 FXRP.
7. Control 1 withdraws at `T-1`. Its delayed assets belong to the prior period, so a 50 FXRP payment is correct.
8. Control 2 withdraws at `T+1`. Its delayed assets are target-period payable but cannot overwrite the checkpoint at `T`, so the full 100 FXRP is paid.

### Decisive output

```
  PoC: exact-boundary withdrawal reduces a valid incident payout
boundary withdrawal mined first; allocator committed 18 seconds later
commitment and settled cover: 100 USD from normal 2x capacity
payable vault assets: 50 active + 50 delayed = 100 FXRP
requested target payout: 100 FXRP; paid: 50 FXRP
closed incident leaves shareholder with 50 FXRP of retained value
    ✔ withdraws at T, commits 18 seconds later, and pays only 50 of a valid 100 loss
    ✔ control: a T-1 withdrawal makes only 50 target-period assets payable, so paying 50 is correct
    ✔ control: the same withdrawal at T+1 remains fully slashable and pays 100

  3 passing (5s)
```

### Why the logs prove the finding

| Assertion                                     | Conclusion                                                               |
| --------------------------------------------- | ------------------------------------------------------------------------ |
| `totalAssetsAt(T - 1) == 100 FXRP`            | All 100 units were exposed immediately before period N.                  |
| `totalAssetsAt(T) == 50 FXRP`                 | The exact-boundary withdrawal replaced the inclusive opening checkpoint. |
| Active 50 plus delayed 50 equals 100 FXRP     | The vault still classifies all 100 units as target-period payable.       |
| Capacity is 100 USD and FLB is zero           | The normal 2x vault-capacity route authorizes the later commitment.      |
| `PayoutExecuted` requests 100 and pays 50     | The overwritten historical cap causes the shortfall.                     |
| Incident is `CLOSED`                          | The partial payout does not leave a retryable residual incident.         |
| Shareholder retains 25 delayed plus 25 active | The 50 FXRP shortfall remains with the withdrawing shareholder.          |
| `T-1` pays 50 and `T+1` pays 100              | Exact-boundary replacement is the harmful delta.                         |
