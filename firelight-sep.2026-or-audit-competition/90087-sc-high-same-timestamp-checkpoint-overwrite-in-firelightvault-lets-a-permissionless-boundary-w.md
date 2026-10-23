# 90087 sc high same timestamp checkpoint overwrite in firelightvault lets a permissionless boundary withdrawal underpay a valid same period payout

**Submitted on Aug 22nd 2026 at 22:11:06 UTC by @mohamadreza for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90087
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`FirelightVault.payout()` caps every incident payout by the vault's total assets at the start of the covered period, read from a timestamp-keyed checkpoint intended to represent an immutable exposure cap. Because the checkpoint library overwrites the previous value when a new checkpoint uses the same timestamp, a permissionless `withdraw()` included in the period-boundary block can replace the true period-start snapshot with the post-withdraw balance. In production, this allows an ordinary staker to reduce the exposure cap to zero or to a lower value, causing an approved in-window payout for that period to be under-delivered while the withdrawn assets remain fully claimable by the staker.

## Vulnerability Details

`payout()` sizes the payout using the assets committed at the start of the covered period:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
```

The checkpoint trace is written on deposits, withdrawals, and redemptions through `_logTrace()`, using the current block timestamp as the key:

```solidity
uint48 ts = Time.timestamp();
if (isLogAssets) {
    _traceTotalAssets.push(ts, _totalAssets);
}
```

The underlying `Checkpoints.Trace256.push()` implementation overwrites the stored value when a new checkpoint uses the same timestamp key:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
}
```

As a result, `_traceTotalAssets` is used for two incompatible purposes:

1. a mutable per-transaction history of current total assets; and
2. the historical period-start exposure snapshot trusted by `payout()`.

Same-key replacement is valid for the first purpose but corrupts the second. A deposit or withdrawal included in a block whose timestamp equals the period boundary can silently rewrite the value later used as the payout exposure cap.

### Full-escape attack path

1. A staker deposits `600,000` units during period 0. The legitimate period-1 start exposure is therefore `600,000`.
2. At exactly `periodOneStart`, the staker calls `withdraw(600,000)`.
3. `_requestWithdraw()` assigns the assets to `withdrawAssets[currentPeriod() + 1]`, which is `withdrawAssets[2]`.
4. `_logTrace()` writes a post-withdraw active total of `0` using the same timestamp key, overwriting the original `600,000` period-start snapshot.
5. `totalAssetsAt(periodOneStart)` now returns `0`, while `withdrawAssets[2]` still contains `600,000`.
6. A valid, approved, in-window payout for a `600,000` loss captured in period 1 is executed:

```
assetsAtCapturePeriod = 0
activePayableAmount   = 600,000
paidAmount            = min(600,000, 0, 600,000) = 0
```

7. The configured payout recipient receives nothing, while the staker later claims the entire `600,000` withdrawal bucket.

The withdrawal is a normal permissionless vault operation. No privileged role, oracle manipulation, invalid incident, or invalid payout amount is required. The execution condition is that the transaction is included in a block whose timestamp equals the predictable period boundary. The attacker cannot force block inclusion, but can target the known boundary block.

### Partial and deposit-side variants

Withdrawing only part of the position reduces the period-start cap by the withdrawn amount and causes the valid payout to be under-delivered proportionally.

The inverse direction is also possible: a deposit included at the exact period boundary can overwrite the snapshot with a larger value, exposing fresh capital to a loss that predates that deposit.

These are consequences of the same root cause and share the same remediation; they are not separate findings.

## Impact Details

Selected impact: `Contract fails to deliver promised returns, but doesn't lose value`.

A valid, approved, in-window incident payout for the covered period can be silently under-delivered. In the full-escape case, an approved `600,000` payout is reduced to `0`. In the partial case, an approved `600,000` payout is reduced to `300,000`.

The value is not stolen or burned: the withdrawn assets remain in the vault's delayed withdrawal bucket and are later claimed by the same staker. However, the contract fails to deliver the approved payout to the configured payout recipient, and the effective coverage for that period is reduced or eliminated.

Demonstrated loss breakdown using a six-decimal vault asset:

* Full escape: approved payout of `600,000`; actual payout is `0`.
* Partial escape: approved payout of `600,000`; actual payout is `300,000`.

The attacker only needs to be an ordinary staker. No privileged role, allowlist entry, oracle manipulation, or invalid incident is required. `withdraw()` and `redeem()` are not blocked by the active-incident deposit restriction. The primary condition is a timing/inclusion condition: the withdrawal must be included in a block whose timestamp equals the deterministic period-start second. The live configuration uses one-day periods (confirmed on-chain: `currentPeriodConfiguration()` returns duration = 86400 on the deployed proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`), so the boundary is publicly predictable.

## References

* [FirelightVault payout logic (exposure cap)](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L939-L958)
* [FirelightVault withdrawal and redemption logic](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L757-L817)
* [FirelightVault checkpoint logging](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1074-L1086)
* [Checkpoints.Trace256.push() same-key overwrite](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L215)
* [Firelight audit competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)

## Link to Proof of Concept

<https://gist.github.com/rezar13r/38ce86af560becfa36a59e8ccf0f03a2>

## Proof of Concept

## Proof of Concept

A complete Foundry PoC is available in the following Secret Gist:

<https://gist.github.com/rezar13r/38ce86af560becfa36a59e8ccf0f03a2>

The PoC file is:

```
M1_PeriodStartWithdrawalEscape.t.sol
```

It tests the real `FirelightVault` implementation behind an `ERC1967Proxy` in a local EVM environment. No transaction is sent to Firelight mainnet.

### Full-escape scenario

1. Deploy the vault with a one-day period duration and configure a normal payout role and payout recipient.
2. An ordinary staker deposits `600,000` asset units during period 0.
3. Before period 1 starts, the correct historical exposure is:

   ```
   totalAssetsAt(periodOneStart) = 600,000
   ```
4. Move the block timestamp to exactly `periodOneStart`.
5. The ordinary staker calls the normal permissionless withdrawal function:

   ```solidity
   vault.withdraw(600_000, staker, staker);
   ```
6. The withdrawal is assigned to the delayed withdrawal bucket:

   ```
   withdrawAssets[2] = 600,000
   ```
7. The withdrawal also calls `_logTrace()`, which writes the post-withdraw active total under the same timestamp key. Because `Checkpoints.Trace256.push()` overwrites an existing equal-key checkpoint, the period-start snapshot becomes:

   ```
   totalAssetsAt(periodOneStart) = 0
   ```
8. A valid, approved, in-window payout for a `600,000` loss captured in period 1 is executed:

   ```solidity
   paid = vault.payout(payoutRecipient, 600_000, periodOneStart);
   ```
9. The payout calculation becomes:

   ```
   assetsAtCapturePeriod = 0
   activePayableAmount   = 600,000
   paidAmount             = min(600,000, 0, 600,000) = 0
   ```
10. The configured payout recipient receives zero, even though the full `600,000` remains in the delayed withdrawal bucket.
11. After the withdrawal becomes claimable, the staker calls:

    ```solidity
    vault.claimWithdraw(2);
    ```
12. The staker receives the full `600,000`. The valid approved payout is therefore completely bypassed.

### Partial-withdrawal scenario

1. Two ordinary stakers deposit `300,000` each before period 1 starts.
2. The correct period-start exposure is `600,000`.
3. At exactly `periodOneStart`, staker A withdraws `300,000`.
4. The same-timestamp checkpoint overwrite changes the historical exposure snapshot from `600,000` to `300,000`.
5. A valid approved payout for `600,000` is executed.
6. Only `300,000` is paid because the payout is capped by the corrupted snapshot. The remaining value stays claimable through the withdrawal bucket instead of remaining exposed to the full period-1 loss.

### One-second control

The PoC repeats the full-withdrawal scenario with the withdrawal executed at `periodOneStart + 1` instead of exactly at the period boundary.

In the control case:

```
totalAssetsAt(periodOneStart) = 600,000
paidAmount                    = 600,000
withdrawAssets[2]             = 0 after payout
```

This confirms that the issue is caused specifically by the same-timestamp checkpoint overwrite.

### Reproduction command

From the repository root, after checking out commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` and copying the PoC file into `test/foundry/`, run:

```bash
forge test --match-path test/foundry/M1_PeriodStartWithdrawalEscape.t.sol -vv
```

Expected result:

```
[PASS] test_baseline_withdraw_one_second_later_no_escape()
[PASS] test_boundary_withdrawal_can_escape_entire_same_period_slash()
[PASS] test_boundary_withdrawal_escapes_part_of_same_period_slash()
3 passed; 0 failed
```

The attacker is only an ordinary staker using `withdraw()`. No privileged attacker, oracle manipulation, invalid incident, or mainnet transaction is required.
