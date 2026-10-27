# 90183 sc high rescue tombstone permanently freezes newly funded same bucket withdrawals after unblock

**Submitted on Aug 23rd 2026 at 14:11:18 UTC by @kryoka for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90183
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief / Intro

`FirelightVault.rescueWithdrawFromBlocklisted` correctly moves a blocklisted account's old pending withdrawal to a clean beneficiary, but it also permanently sets `isWithdrawClaimed[period][source] = true`. The shipped `removeFromBlocklist` function clears only the address blocklist bit.

If the cleared account deposits new principal and requests another withdrawal before the current period rolls, `_requestWithdraw` accepts the new request into the same `(period, receiver)` record without checking or resetting the terminal flag. The deposit transfers new assets into vault custody, and the accepted request burns the user's new shares while recording a positive funded pending entitlement. At maturity, `claimWithdraw` checks the stale flag first and reverts with `AlreadyClaimedPeriod`. Re-blocking the account and invoking the only pending-position rescue path fails on the same flag. No function in the assessed implementation resets, cancels, or transfers the poisoned record.

The PoC freezes exactly 300 FXRP of wholly new post-unblock principal. A second receiver-keyed witness freezes 250 FXRP after an unrelated owner burns its own shares into the poisoned receiver. Clean receivers and later buckets remain live, proving that this is a funded key-specific freeze rather than a global vault halt or a view-only error.

## Vulnerability Details

The withdrawal state is keyed by `(period, account)` and combines reusable per-account withdrawal shares with a terminal-looking claimed boolean:

* [`FirelightVaultStorage.sol:61-74`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L61-L74) stores aggregate assets/shares, per-account shares, and `isWithdrawClaimed[period][account]`.
* [`rescueWithdrawFromBlocklisted`, lines 845-875](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L875) moves the old per-account shares to a clean destination, zeros the source, and sets the source's claimed flag to `true` at line 870.
* [`removeFromBlocklist`, lines 556-563](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L556-L563) changes only `isBlocklisted`; it leaves the period/account withdrawal state untouched.
* [`_requestWithdraw`, lines 994-1028](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028) targets `currentPeriod() + 1`, adds assets and shares to the receiver, increments pending accounting, and burns the owner's active shares. It never reads or resets `isWithdrawClaimed`.
* [`claimWithdraw`, lines 788-810](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L810) reverts on the stale boolean before computing or transferring the new positive entitlement.

Across the complete scoped source, this mapping is assigned only to `true` in the normal claim and rescue paths. There is no reset, pending-right receiver replacement, cancellation, or administrator claim-on-behalf function.

The violated invariant is: a completed rescue of an old pending position must not make later, independently funded withdrawal rights of an unblocked account unclaimable. Equivalently, `_requestWithdraw` must not add positive rights to a `(period, receiver)` record that `claimWithdraw` already treats as terminal.

## Impact Details

This maps directly to the program-listed **Critical — Permanent freezing of funds** impact:

* the affected amount is newly deposited claimant principal, not yield, an NFT, dust, or a display-only artifact;
* the ordinary withdrawal request burns value-bearing active shares;
* the corresponding assets remain in vault custody and in `pendingWithdrawAssets`;
* both the ordinary claim and the only pending-position reassignment path fail on the same historical flag; and
* waiting, pause/blocklist cycling, role replacement, configuration changes, payout attempts, and successful later buckets do not repair the historical key.

The amount scales with the valid fresh request subject to normal vault limits. The witnesses use 250 and 300 FXRP, not dust. A custom implementation upgrade or off-chain reimbursement could remediate the state, but neither is a shipped recovery path in the assessed code.

## Preconditions and Attack Sequence

1. During current period `P`, an ordinary user deposits and requests withdrawal of an old amount. The request targets future bucket `P+1`.
2. For a legitimate compliance reason, `BLOCKLIST_ROLE` blocklists the account.
3. `RESCUER_ROLE` moves the old `P+1` position to a clean beneficiary. This correctly preserves the old position but sets the source's `isWithdrawClaimed[P+1]` flag to `true`.
4. After compliance clearance, `BLOCKLIST_ROLE` calls the shipped `removeFromBlocklist` function.
5. Before period `P` ends, the source deposits entirely new principal and makes an ordinary withdrawal or redeem request to itself. The vault accepts this request into `P+1`, accounts for it as pending, and burns the new shares.
6. After `P+1` ends, the clean beneficiary claims the old rescued position normally. The source still has a positive, fully funded new entitlement, but its claim reverts `AlreadyClaimedPeriod(P+1)`.
7. Re-blocking the source and calling pending-withdrawal rescue again also reverts `AlreadyClaimedPeriod(P+1)`.

No role must act maliciously or outside its documented authority. The claimed harm is not the authorized movement of the old compliance position; it is the silent acceptance and share burn of wholly new principal after the shipped unblock operation. The ordinary user chooses the later deposit and withdrawal.

The rescue, unblock, deposit, and second request must complete before the period rolls so both requests target the same future bucket. This timing affects feasibility, not the permanence of an already accepted and funded request. The retained official documentation describes one-day current periods and planned 30-day feature-complete periods; current deployed configuration and compliance history were not queried.

## Negative Controls and Recovery Boundary

* Waiting one period before the new request targets a clean later bucket and pays the expected amount at maturity.
* A clean receiver in the same period succeeds. An unrelated clean owner can instead burn its own shares into the poisoned receiver and reproduce the receiver-keyed freeze.
* The beneficiary of the old rescue claims its complete entitlement normally; exactly the fresh poisoned amount remains pending and physically held.
* The primary witness uses no incident payout, haircut, donation, saturation, or material rounding.
* Re-block plus pending rescue fails on the same flag. Active-share rescue cannot touch already-burned pending shares.
* Pause/unpause, deposit-limit and period-configuration changes, rescuer-role replacement, a valid zero-paid in-window payout whose capture-start exposure is zero, unblock/reblock, and waiting do not clear the historical marker. A later mature payout is rejected with `InvalidCapturePeriod` and is not a recovery route.

These controls distinguish the issue from a view bug, withdrawal rounding, aggregate denominator saturation, a currently blocklisted receiver, or a global vault shutdown.

## Scope and Prior-Art Qualification

The issue was compared against the program's known issues, the 18 Macro findings and six accepted risks, the indexed 2025 Firelight reports, public repository issues and pull requests, and retained source history. No exact match was found.

The closest public records are distinct: report 59027 concerns a stale view of an already completed claim; report 59288 concerns rescue-loop gas efficiency; PR #5 adds blocklist checks; and historical commit `a2def47` prevents rescue to an already-claimed destination. None prevents a rescued source from accepting new rights after unblock. This issue does not require synthetic-share saturation, payout haircuts, or rounding; it freezes exact material value on a stale boolean.

The privileged-role exclusion should not remove the finding. Regulatory blocklisting, rescue to a clean address, and later unblock are shipped protocol workflows. All privileged calls use their intended inputs. The intended- constraint violation occurs when the contract accepts wholly new post-unblock principal into a record it already treats as permanently claimed.

## Recommended Remediation

Do not overload one boolean for both “old rights were moved by rescue” and “this account can never receive rights in this bucket again.”

The smallest repair is to avoid marking the rescued source as claimed after its per-account withdrawal shares are zeroed. Zero shares already prevent the source from claiming the moved old position, while future newly funded shares remain claimable. Keep guards that reject genuinely paid sources and already-claimed destinations.

If the tombstone is intentional, `_requestWithdraw` must instead reject a receiver whose `isWithdrawClaimed[currentPeriod()+1][receiver]` is already true, before accounting changes or share burn. The user can then wait for the next bucket or select a clean receiver without losing shares.

Add regression tests for rescue → unblock → same-bucket self-request, clean and poisoned third-party receivers, actually paid records versus moved rights, payout-adjusted buckets, pause/blocklist cycling, and later-bucket liveness. Blindly clearing every historical flag on unblock would be unsafe because a truly paid record could become claimable twice.

## Evidence Limitations

The PoC runs locally against the exact competition pin `42f9ea5e43d88b35197b901acc2a8c24b314fa62`. It does not claim that a current production deployment already contains the required compliance history or same-bucket state. No mainnet, public testnet, fork, external oracle, or third-party system was contacted. Private or rejected bounty reports are not public, so the novelty result is limited to the retained public corpus.

The complete runnable PoC source is provided in the Proof of Concept field as `test/NR3-S21-rescue-reopen-bucket.js`, SHA-256 `cc60173d88376c2a4ace2408ee36466eb4b2206ac296205ff805be94be4b69e1`. The exact bytes passed `2/2` twice in the final clean exact-pin replay, together with `2/2` clean-path negative controls.

## Link to Proof of Concept

<https://gist.github.com/ferbiy/4c66c14cc3555bddf3084062e8fe1542>

## Proof of Concept

## Proof of Concept

This PoC is local-only and runs against the exact Firelight competition pin:

```
Repository: https://github.com/immunefi-team/audit-comp-firelight
Commit: 42f9ea5e43d88b35197b901acc2a8c24b314fa62
Tree: 6c38527edd6f8d28b4b8fce5c5cdbad331c86124
```

Attach the unchanged file as `NR3-S21-rescue-reopen-bucket.js`:

```
SHA-256: cc60173d88376c2a4ace2408ee36466eb4b2206ac296205ff805be94be4b69e1
```

The retained source file is 264 lines and uses the repository's real proxy fixture and `FirelightVault` implementation. It does not use storage injection, a fork, an external RPC, or a live system.

### Reproduction

From a clean checkout:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62

# npm 12 blocks root Git dependencies unless explicitly allowed. The lockfile
# also pins two public repositories with SSH URLs; rewrite those URLs to HTTPS
# for this command without changing global Git configuration.
GIT_CONFIG_COUNT=2 \
GIT_CONFIG_KEY_0=url.https://github.com/.insteadOf \
GIT_CONFIG_VALUE_0=ssh://git@github.com/ \
GIT_CONFIG_KEY_1=url.https://gitlab.com/.insteadOf \
GIT_CONFIG_VALUE_1=ssh://git@gitlab.com/ \
npm ci --allow-git=root

# A fresh machine may download the pinned Solidity compiler during this step.
npx hardhat compile --network hardhat

# Copy the attached file into the checkout:
cp /path/to/attached/NR3-S21-rescue-reopen-bucket.js \
  test/NR3-S21-rescue-reopen-bucket.js

env -u NODE_RPC_URL -u DEPLOYMENT_ACCOUNT_KEY -u EXECUTION_KEYS \
  -u EXTRA_KEYS -u HARDHAT_CHAIN_ID -u ETHERSCAN_API_KEY \
  CI=true FORK=0 HARDHAT_NETWORK=hardhat \
  HARDHAT_TELEMETRY_DISABLED=1 \
  HARDHAT_DISABLE_TELEMETRY_PROMPT=true \
  XDG_CONFIG_HOME=/tmp/nr3-s21-hardhat \
  npx hardhat test --network hardhat --no-compile \
    test/NR3-S21-rescue-reopen-bucket.js
```

Expected terminal result:

```
NR3-S21 retained/new shares {
  "poisonedBucket":"1",
  "stuckEntitlementFXRP":"300",
  "bucketAssetsFXRP":"500",
  "pendingFXRP":"300",
  "activeFXRP":"1800",
  "physicalFXRP":"2100"
}
NR3-S21 third-party receiver {
  "bucket":"1",
  "thirdPartySharesBurnedFXRP":"500",
  "poisonedReceiverEntitlementFXRP":"250",
  "cleanReceiverPaidFXRP":"350",
  "pendingFXRP":"250",
  "activeFXRP":"1400",
  "physicalFXRP":"1650"
}
2 passing
```

### What the two tests prove

The first test performs the complete sequence:

1. An ordinary account deposits and requests an old withdrawal into bucket `P+1`.
2. Authorized blocklist and rescue move that old position to a clean beneficiary and set the source's claimed marker.
3. Authorized unblock clears only the address blocklist bit.
4. The source deposits 300 FXRP of new principal and makes a second ordinary request before `P` rolls. The request succeeds, records 300 FXRP pending, and burns 300 FXRP of shares.
5. At maturity, the clean beneficiary receives the old 200 FXRP, while the source's new 300-FXRP claim reverts `AlreadyClaimedPeriod`.
6. Re-block/rescue, active-share rescue, role replacement, pause cycling, configuration changes, payout attempts, repeated unblock/reblock, and waiting do not recover the 300 FXRP.
7. A later non-tombstoned bucket remains claimable, proving the failure is specific to the poisoned historical key.

The second test shows that the failure is receiver-keyed: an unrelated clean owner burns 500 FXRP of its own shares into two receivers in the same bucket. The clean receiver receives 350 FXRP; the poisoned receiver retains a funded 250-FXRP entitlement that cannot be claimed.

Both tests assert exact physical conservation:

```
vault token balance == totalAssets() + pendingWithdrawAssets()
```

The assets are therefore still held by the vault and attributed to the pending position; the failure is not missing test funding or an accounting-only view.

### Expected Result

After the old position is rescued and the account is unblocked, either the vault must reject a new request targeting a terminal `(period, receiver)` key before burning shares, or a successfully accepted and funded request must be claimable at maturity.

### Actual Result

The vault accepts the new request, burns the new shares, records 250 or 300 FXRP of funded pending entitlement, and retains the matching assets. At maturity, both `claimWithdraw` and the only pending-position rescue path revert on the stale `AlreadyClaimedPeriod` marker. No shipped recovery clears or moves that record.

### Independent execution record

The exact attached bytes passed repeatedly:

* producer: `2/2` twice;
* independent final package reconstruction: producer and independent controls passed twice;
* final clean exact-pin replay: `2/2` twice plus `2/2` clean-path negative controls;
* adjacent original Firelight regression suite: `448/448` passing in the final clean replay.

The decisive negative controls prove that a later bucket and a clean third-party receiver remain payable. The same receiver and same future bucket are required for the stale-marker collision.
