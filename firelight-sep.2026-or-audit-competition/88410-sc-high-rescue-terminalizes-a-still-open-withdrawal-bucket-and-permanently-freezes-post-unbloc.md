# 88410 sc high rescue terminalizes a still open withdrawal bucket and permanently freezes post unblock principal

**Submitted on Aug 13th 2026 at 03:16:37 UTC by @Kopi for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88410
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted` marks a blocked user's future withdrawal bucket as claimed even though that bucket can still receive new withdrawals. If the account is legitimately unblocked before the current period ends, a later `withdraw` or `redeem` succeeds, burns the user's shares, and appends fresh principal to the already-claimed bucket. Once mature, both normal claim and another rescue permanently revert, leaving the fresh principal trapped in the vault with no supported recovery path.

## Vulnerability Details

### Root cause

Firelight's delayed withdrawal flow burns shares immediately and records withdrawal shares in `withdrawSharesOf[period][receiver]`, where the target period is `currentPeriod() + 1`.

`rescueWithdrawFromBlocklisted` supports future periods. It transfers the source bucket to a rescue receiver, clears the source amount, and sets `isWithdrawClaimed[period][from] = true`. This terminal marker is safe only if the source bucket can never receive another withdrawal.

The source bucket can be reused. If the account is unblocked before the current period changes, its next withdrawal targets the same future period. `_requestWithdraw` does not check `isWithdrawClaimed[period][receiver]` before it:

1. adds new withdrawal shares to the bucket;
2. increases `withdrawAssets`, `withdrawShares`, and `pendingWithdrawAssets`; and
3. burns the owner's vault shares.

At maturity, `claimWithdraw` checks `isWithdrawClaimed` first and reverts `AlreadyClaimedPeriod`. Re-blocking the account and attempting another rescue reaches the same source-side flag and also reverts. No in-scope function clears the flag or moves the new pending entitlement.

### Reachable sequence

1. In period `P`, an account has a withdrawal entitlement in period `P + 1`.
2. The account is blocklisted through the intended compliance role.
3. An authorized rescuer moves the entitlement to an allowed receiver. The vault sets `isWithdrawClaimed[P + 1][account] = true`.
4. Still in period `P`, the blocklist role legitimately unblocks the account.
5. The restored user owns or acquires shares and calls self-`withdraw` or self-`redeem`.
6. The request succeeds and burns the shares into the same `P + 1` bucket despite its terminal flag.
7. At maturity, the rescue receiver claims the original entitlement. The restored user's claim reverts, and a repeat rescue also reverts.

Neither privileged actor exceeds its named authority. The unsafe result is produced by the interaction of valid rescue, unblock, and ordinary withdrawal transitions.

### Recommended remediation

The narrow prospective fix is to stop setting `isWithdrawClaimed[period][from]` during rescue after the source entitlement is fully moved and zeroed. A repeat claim/rescue then fails on zero amount, while a later request remains claimable. As defense in depth, `_requestWithdraw` should reject an already-claimed target bucket before changing accounting or burning shares. Existing rescue-created flags require targeted migration or user compensation; the mapping is not enumerable, so candidate keys must be reconstructed from complete `WithdrawRescuedFromBlocklisted` logs rather than cleared indiscriminately.

## Impact Details

The exact in-scope impact is **Permanent freezing of funds**. The PoC freezes `1,000` six-decimal vault-asset units:

* The user's second withdrawal succeeds.
* The user's vault shares are irreversibly burned.
* The fresh entitlement remains recorded in `withdrawSharesOf` and `pendingWithdrawAssets`.
* The same amount remains in the vault's raw token balance.
* `claimWithdraw` permanently rejects the entitlement.
* `rescueWithdrawFromBlocklisted` also permanently rejects it.

The trapped amount is excluded from active ERC-4626 `totalAssets`, so it is neither claimable principal nor reusable active capital. Recovery requires a contract upgrade, storage migration, or exceptional compensation. The issue does not depend on attacker profit; it is a permanent loss of access to the restored user's principal.

The amount is not dust-bounded. Any shares the restored account owns or acquires before the period changes can be burned into the poisoned bucket. Likelihood is low because rescue and unblock must occur in the same source period, but each operation is supported and Firelight permits one-day minimum periods.

## References

* [`FirelightVault` blocklist administration](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L546-L563)
* [`FirelightVault.withdraw` and `redeem`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L777)
* [`FirelightVault.claimWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L810)
* [`FirelightVault.rescueWithdrawFromBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L875)
* [`FirelightVault._requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1027)
* [Immunefi Firelight competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)
* [Related 2025 OpenZeppelin/Sentora audit](https://firelight.finance/audit.pdf)
* [Destination-claimed rescue fix](https://github.com/immunefi-team/audit-comp-firelight/commit/a2def47716389330c0ebc1f2006fca5c588c9469)

The prior audit's M-02 finding has a similar final symptom but a different producer: period-duration reconfiguration could route a request into an old claimed period. That mechanism was repaired with monotonic period configuration. This finding requires no reconfiguration; the new rescue function itself terminalizes the still-open `P + 1` source bucket. The later destination-claimed rescue fix checks the recipient, not reuse of the rescued source bucket.

## Link to Proof of Concept

<https://gist.github.com/sebastian-osec/920898b55fbb4c216effe93696ca6d5e>

## Proof of Concept

## Proof of Concept

The PoC runs only on a local Hardhat network and uses the repository's real proxy `FirelightVault`, role checks, withdrawal accounting, and six-decimal fixture asset.

### Steps

1. Clone `https://github.com/immunefi-team/audit-comp-firelight.git` and check out revision `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.
2. Run `npm ci`.
3. Create `test/security/`.
4. Attach/copy [`poc.test.js`](https://gist.githubusercontent.com/sebastian-osec/920898b55fbb4c216effe93696ca6d5e/raw/872212a4cacd4f4315a556ef3306ca22ba5714b4/poc.js) into `test/security/poc.test.js`.
5. Run:

```sh
npx hardhat test test/security/poc.test.js
```

The test has SHA-256 `d577317947652fc6c0871213a914f456f55d065076cc134c2b92ab58e120b05a` and passed at the pinned revision with Node `v24.16.0`, npm `11.13.0`, and Hardhat `2.22.19`:

```
FL-012: rescued claimed bucket can be reused and strand a later withdrawal
  3 passing (656ms)
```

The exploit case observes:

```
old rescued entitlement: 1000000000
new inaccessible entitlement: 1000000000
pending withdrawal assets after maturity: 1000000000
raw vault balance after maturity: 1000000000
source claim: AlreadyClaimedPeriod
repeat rescue: AlreadyClaimedPeriod
```
