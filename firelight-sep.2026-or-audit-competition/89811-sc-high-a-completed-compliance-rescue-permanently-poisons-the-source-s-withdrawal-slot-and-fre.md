# 89811 sc high a completed compliance rescue permanently poisons the source s withdrawal slot and freezes fresh principal after unblocking

**Submitted on Aug 21st 2026 at 10:11:53 UTC by @legat for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89811
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted()` moves a blocklisted account's current pending withdrawal to a clean recipient, zeros the source entitlement, and permanently sets `isWithdrawClaimed[period][from] = true`.

That flag is never cleared when the account is removed from the blocklist. If the restored account makes a fresh deposit and uses the ordinary `redeem()` or `withdraw()` flow before the current period changes, `_requestWithdraw()` credits the independently funded entitlement into the same `(period, account)` slot without checking or clearing its already-claimed flag. The transaction succeeds, the fresh shares are burned, and the fresh assets enter `pendingWithdrawAssets`.

After the delay, `claimWithdraw()` rejects the positive fresh entitlement on the stale flag. Re-blocking the account and invoking the authorized rescue fallback rejects it on the same flag. No scoped method clears the tombstone, cancels the accepted request, restores the burned shares, or transfers the poisoned position. The fresh principal is permanently frozen.

## Description

### Intended Behavior

Firelight documents unstaking as a two-step process. The staker initiates, then receives the underlying after the defined unstaking window. The documentation states that the protocol adds no additional waiting beyond that window.

The documented blocklist lifecycle is equally specific: blocklisted addresses cannot use the vault, while an authorized role may reassign their assets to a non-blocklisted address, **preserving the position**. Removing an address from the blocklist restores ordinary vault operations; the repository's own blocklist regression test confirms that transfers work again after removal.

Accordingly:

* rescue authority concerns the current blocklisted position;
* the source is empty after a complete pending-withdrawal and active-share rescue;
* clearing the blocklist restores the account;
* an accepted new withdrawal must become payable after the ordinary delay.

Relevant public documentation:

* [Deployments and Withdrawals — withdrawal timing](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals.md#step-2-withdraw)
* [Deployments and Withdrawals — blocklist and rescues](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals.md#blocklist-and-rescues)
* [Vault Architecture](https://docs.firelight.finance/core-concepts/vault-architecture.md)

### Actual Behavior

The rescue correctly moves and zeros the old source position, but also writes a terminal flag against the reusable `(period, source)` key:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

`removeFromBlocklist()` clears only the blocklist mapping:

```solidity
function removeFromBlocklist(address account) external onlyRole(BLOCKLIST_ROLE) onlyBlocklisted(account) {
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

The restored account can deposit normally. `maxRedeem()` advertises its entire fresh share balance, and `previewRedeem()` returns the expected asset amount. A full self-redeem then reaches `_requestWithdraw()`:

```solidity
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(
    assets,
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

There is no `isWithdrawClaimed[period][receiver]` validation. The fresh credit and burned shares therefore coexist with a terminal flag.

When the period becomes claimable, `claimWithdraw()` checks the flag before calculating or transferring the positive amount:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender],
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
```

The rescue fallback also checks the source flag before moving the fresh entitlement:

```solidity
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

The old rescued position remains payable to its clean recipient. Only the restored source's later fresh position is inaccessible, isolating the defect to the source tombstone lifecycle.

## Root Cause

`isWithdrawClaimed` is a terminal boolean keyed only by `(period, account)`, but that key can hold multiple sequential entitlement generations.

`rescueWithdrawFromBlocklisted()` uses the boolean to close the source's old generation even after zeroing it. Later, `_requestWithdraw()` treats the same zeroed key as reusable and adds a new generation without reconciling the terminal flag. The code therefore violates this required invariant:

```
withdrawSharesOf[period][account] > 0
    implies
isWithdrawClaimed[period][account] == false
```

The vulnerable combination is:

1. `rescueWithdrawFromBlocklisted()` writes `isWithdrawClaimed[period][from] = true` after emptying the source.
2. `removeFromBlocklist()` does not reconcile per-period withdrawal state.
3. `_requestWithdraw()` credits a positive amount to a terminal slot and irreversibly burns the fresh active shares.
4. Both claim and rescue reject the positive amount before transferring it.

## Risk

### Impact

This maps directly to the program's Critical impact, **Permanent freezing of funds**.

The PoC permanently freezes `2,000 FXRP` of independently supplied fresh principal and proves:

* the historical `1,000 FXRP` rescue recipient claims successfully;
* the restored account's new entitlement remains exactly `2,000 FXRP`;
* the restored account has zero active vault shares and zero liquid FXRP after its accepted full redeem;
* direct claim always reverts;
* an authorized second rescue always reverts;
* ten additional periods do not repair the state; and
* `pendingWithdrawAssets` remains elevated by exactly `2,000 FXRP`.

The demonstrated amount is arbitrary. Any amount admitted and redeemed during the remaining portion of the same current period can be burned into the poisoned next-period key. Because pending assets are excluded from active `totalAssets()`, the poisoned amount also remains reserved in withdrawal accounting rather than returning to the user's active position.

### Likelihood

The trigger uses only documented operations and ordinary values:

* a real pending withdrawal is rescued from a blocklisted account;
* all remaining active shares are rescued, leaving the source completely empty;
* the compliance authority removes the empty source from the blocklist before the current period changes;
* the restored account deposits fresh FXRP and self-redeems normally.

No malicious recipient, custom callback, unsafe multicall, period reconfiguration, token anomaly, storage edit, stolen credential, or unusual parameter is required. The only timing condition is that rescue, removal, and the new request occur before the current period advances, so the new request targets the same `currentPeriod() + 1` key. Firelight documents current one-day periods and intended feature-complete thirty-day periods, making this a normal operational window rather than a constructed timestamp.

The existence of a different-period workaround does not recover an accepted poisoned request. The contract exposes no warning or restriction: it reports the full fresh balance as redeemable, previews the expected output, accepts the self-redeem, and burns the shares.

### Privileged-role applicability

The homepage states:

> Any bug report that requires a privileged role holder to act within the permissions explicitly granted to that role is out of scope, unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint.

This finding falls within that literal exception. The roles do not act maliciously or incorrectly: `BLOCKLIST_ROLE` blocks a source, `RESCUER_ROLE` moves its complete current position to a clean recipient, and `BLOCKLIST_ROLE` later clears the empty source.

The documented authority of `RESCUER_ROLE` is to move and preserve the **current** blocklisted position. It is not authority to install a permanent disability on future, independently funded principal after the account is restored. The rescue call silently exceeds its attributed effect and bypasses the intended constraints that only currently blocklisted accounts are disabled, clearing restores ordinary use, accepted withdrawals become payable after the window, and rescue preserves rather than destroys a position. The rescuer does not select or control the later fresh deposit, redeem, or frozen amount.

## Attack Path

1. The user deposits `3,000 FXRP` and requests a normal `1,000 FXRP` withdrawal for period `q = currentPeriod() + 1`.
2. `BLOCKLIST_ROLE` correctly blocklists the account.
3. `RESCUER_ROLE` correctly moves the period-`q` withdrawal to clean recipient B. This zeroes A's amount and sets `isWithdrawClaimed[q][A] = true`.
4. `RESCUER_ROLE` moves every remaining active share from A to B. A now has no old entitlement and no active shares.
5. `BLOCKLIST_ROLE` correctly removes A from the blocklist before the current period changes.
6. A supplies an independent fresh `2,000 FXRP` deposit and receives fresh shares.
7. `maxRedeem(A)` reports every fresh share as redeemable, and `previewRedeem()` reports `2,000 FXRP`.
8. A self-redeems 100% of the fresh shares. `_requestWithdraw()` credits `withdrawSharesOf[q][A]`, adds `2,000 FXRP` to pending accounting, and burns every fresh share despite the stale true flag.
9. After the full delay, B claims the historical `1,000 FXRP`, proving the pool and period are functional.
10. A's claim for the positive fresh entitlement reverts `AlreadyClaimedPeriod(q)`.
11. A is re-blocklisted, but an authorized rescue to clean recipient C also reverts `AlreadyClaimedPeriod(q)`.
12. A is unblocked and ten more periods pass. Claim still reverts; the fresh amount remains pending, while A retains neither shares nor FXRP.

## External Preconditions

* The source account has an ordinary pending withdrawal and active vault shares.
* Compliance operations legitimately block the account and perform a complete rescue.
* The account is legitimately cleared before the current period changes.
* The restored user owns and approves fresh FXRP for a new deposit.

No role must select the later fresh amount or initiate the later redeem. No role needs to exceed a call-level access check; the excess is the rescue call's undocumented persistent effect on post-restoration funds.

## Internal Preconditions

* The rescued and fresh requests resolve to the same `currentPeriod() + 1` period key.
* Rescue has left `withdrawSharesOf[q][source] == 0` and `isWithdrawClaimed[q][source] == true`.
* The account is no longer blocklisted when it deposits and redeems.
* The vault is unpaused, no active incident blocks the deposit, and the deposit remains within the configured limit.

These are ordinary protocol states. No storage corruption, invalid token return value, period-duration change, or fabricated balance is used.

## Affected Code

Frozen target: [`audit-comp-firelight @ 42f9ea5e`](https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62).

* [`FirelightVault.sol#L556-L563`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L556-L563) removes only the blocklist flag.
* [`FirelightVault.sol#L709-L744`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L709-L744) accepts the fresh redeem and routes it to `_requestWithdraw()`.
* [`FirelightVault.sol#L780-L811`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L780-L811) rejects claim on the stale flag before calculating the positive entitlement.
* [`FirelightVault.sol#L839-L876`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L839-L876) marks the source claimed and later rejects re-rescue on that same flag.
* [`FirelightVault.sol#L994-L1028`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028) credits a terminal slot, increases pending assets, and burns the fresh shares without checking the flag.
* [`FirelightVaultStorage.sol#L61-L74`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L61-L74) defines the reusable per-period amount and terminal boolean mappings.

## Recommended Mitigation

Do not set a permanent claimed flag on the emptied source during rescue. Zeroing the old source amount already prevents the old entitlement from being claimed or rescued twice:

```diff
 withdrawSharesOf[period][to] += amount;
 withdrawSharesOf[period][from] = 0;
-isWithdrawClaimed[period][from] = true;
```

Keep the destination-already-claimed guard introduced by `a2def477…`.

As defense in depth, enforce the invariant before every `_requestWithdraw()` credit. If a claimed slot has a zero existing account amount, safely reinitialize it for the new generation. If it has a nonzero amount, revert **before** pending accounting changes or share burning. A generation counter or explicit withdrawal-position state machine would avoid overloading one sticky boolean across sequential positions.

Add regression coverage for this exact lifecycle: complete pending and active-share rescue, unblock, fresh deposit, full redeem and fixed-asset withdraw into the same next-period slot, successful later claim, and successful rescue fallback.

## Proof of Concept

### Scenario

This report is standalone: the complete deterministic test and guarded runner are embedded below. The test deploys the exact frozen scoped `FirelightVault` through the repository's official fixture and performs the complete twelve-step path above. The fixture deploys the repository-bundled `FAsset` implementation behind its real `FAssetProxy`; `MockContract` is used solely as the local AssetManager authorization/minting shim and does not replace FXRP transfer, balance, approval, or vault behavior. The helper supplies only the `5,000 FXRP` used by the two deposits. Consequently, after the fresh full redeem, the restored account has exactly zero liquid FXRP and zero active vault shares, making the frozen principal balance-aligned and directly measurable.

The test also executes both possible exits after maturity: direct claim and authorized re-rescue. Both fail. The historical recipient's successful claim is the control proving that the period and vault are otherwise operating normally.

### Determinism

The exact frozen and embedded-file hashes are:

```
Repository commit: 42f9ea5e43d88b35197b901acc2a8c24b314fa62
FirelightVault.sol SHA-256: 778608250e5893fe908c6ebf7255cb559094066157f4e70b88950c94e509ab5f
FirelightVaultStorage.sol SHA-256: 24ce74c93880aba0ebfa6fec4c8c9759a5e8b5907aa88669ae985a8c6f246513
FAsset.sol SHA-256: 75194713c07ea3c99a62f22a60f40729bb884ba1c18f3fce4a725595dddcb380
FAssetProxy.sol SHA-256: 87d561360f1fd9ec192aa3dc8d17b9c317477d40b1e4c05ac8eb7eecbb27a958
PoC SHA-256: e7d383c5407901b2f3fe77278c31d915dc768e1df88c9fc6f17a8520d6e63a11
Runner SHA-256: 424819ee2737a1cfa1b9fc85559ef05f7b7a3914da9afcd5059d1f56468d4f06
Node.js: v24.18.0
npm: 11.16.0
Hardhat: 2.22.19
```

`run-poc.sh` refuses to run if the commit, either scoped vault source, bundled FAsset implementation, or bundled FAsset proxy differs. The test edits no storage, subclasses no vault, replaces no vault logic, and contacts no external network. Its claims are exclusively frozen-contract EVM state transitions: role-authorized calls, share burning, mapping writes, maturity-time advancement, claims, and exact reverts. It does not claim any network, consensus, validator, RPC, mempool, or cross-node behavior. Hardhat's in-process chain supplies the complete execution environment, including atomic rollback and deterministic time advancement; Docker, a public fork, and a multi-node cluster are neither required nor probative.

### Logs

```
PoC: rescue permanently poisons a restored account future withdrawal slot
{"poisonedPeriod":"1","rescuedHistoricalAssets":"1000000000","freshAssetsPermanentlyFrozen":"2000000000","victimActiveSharesAfterFullRedeem":"0","victimLiquidFxrp":"0","staleClaimedFlag":true,"vaultPendingWithdrawalAssets":"2000000000","finalCurrentPeriod":"12"}
  ✔ completes rescue, unblocks, accepts a fresh full redeem, then makes claim and re-rescue impossible forever (362ms)

1 passing (364ms)
```

The token uses six decimals, so `2000000000` atomic units equals `2,000 FXRP`.

### Source — `poc/rescue-source-tombstone-fresh-redeem/firelight-vault-rescue-source-tombstone.e2e.poc.js`

```javascript
// Local-only deterministic PoC against the exact frozen scoped implementation.
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const { deployVault } = require('../../test/setup/fixtures.js')

describe('PoC: rescue permanently poisons a restored account future withdrawal slot', function () {
  const FXRP = 10n ** 6n
  const INITIAL_DEPOSIT = 3_000n * FXRP
  const INITIAL_WITHDRAWAL = 1_000n * FXRP
  const FRESH_DEPOSIT = 2_000n * FXRP
  const DEPOSIT_LIMIT = 100_000n * FXRP

  it('completes rescue, unblocks, accepts a fresh full redeem, then makes claim and re-rescue impossible forever', async function () {
    const {
      token_contract: fxrp,
      firelight_vault: vault,
      blocklister,
      rescuer,
      users,
      utils,
      config,
    } = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })

    const restoredUser = users[0]
    const rescueRecipient = users[1]
    const secondRescueRecipient = users[2]

    // The user funds only the two deposits used below. At the end of the fresh
    // deposit, their liquid fXRP balance is therefore exactly zero.
    await utils.mintAndApprove(INITIAL_DEPOSIT + FRESH_DEPOSIT, restoredUser)
    await vault.connect(restoredUser).deposit(INITIAL_DEPOSIT, restoredUser.address)

    // Create an ordinary pending withdrawal in the next period.
    const poisonedPeriod = (await vault.currentPeriod()) + 1n
    await expect(
      vault
        .connect(restoredUser)
        .withdraw(INITIAL_WITHDRAWAL, restoredUser.address, restoredUser.address),
    )
      .to.emit(vault, 'WithdrawRequest')
      .withArgs(
        restoredUser.address,
        restoredUser.address,
        restoredUser.address,
        poisonedPeriod,
        INITIAL_WITHDRAWAL,
        INITIAL_WITHDRAWAL,
      )

    // Perform the complete, documented compliance lifecycle: block the source,
    // move its pending position, and move every remaining active share.
    await vault.connect(blocklister).addToBlocklist(restoredUser.address)
    await vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(restoredUser.address, rescueRecipient.address, [poisonedPeriod])
    await vault
      .connect(rescuer)
      .rescueSharesFromBlocklisted(restoredUser.address, rescueRecipient.address)

    expect(await vault.withdrawSharesOf(poisonedPeriod, restoredUser.address)).to.equal(0n)
    expect(await vault.balanceOf(restoredUser.address)).to.equal(0n)
    expect(await vault.isWithdrawClaimed(poisonedPeriod, restoredUser.address)).to.equal(true)
    expect(await vault.withdrawalsOf(poisonedPeriod, rescueRecipient.address)).to.equal(INITIAL_WITHDRAWAL)

    // The compliance authority deliberately restores the account. No stale
    // balance or entitlement remains at the source when it is restored.
    await vault.connect(blocklister).removeFromBlocklist(restoredUser.address)
    expect(await vault.isBlocklisted(restoredUser.address)).to.equal(false)

    // The restored user supplies independent fresh principal. The vault accepts
    // a redeem of 100% of those fresh shares into the still-current next period,
    // burns every share, and records a positive withdrawal despite the stale flag.
    await vault.connect(restoredUser).deposit(FRESH_DEPOSIT, restoredUser.address)
    const freshShares = await vault.balanceOf(restoredUser.address)
    expect(await vault.maxRedeem(restoredUser.address)).to.equal(freshShares)
    expect(await vault.previewRedeem(freshShares)).to.equal(FRESH_DEPOSIT)
    const freshAssets = await vault
      .connect(restoredUser)
      .redeem.staticCall(freshShares, restoredUser.address, restoredUser.address)

    expect(freshAssets).to.equal(FRESH_DEPOSIT)
    await expect(
      vault.connect(restoredUser).redeem(freshShares, restoredUser.address, restoredUser.address),
    )
      .to.emit(vault, 'WithdrawRequest')
      .withArgs(
        restoredUser.address,
        restoredUser.address,
        restoredUser.address,
        poisonedPeriod,
        FRESH_DEPOSIT,
        freshShares,
      )

    expect(await vault.balanceOf(restoredUser.address)).to.equal(0n)
    expect(await fxrp.balanceOf(restoredUser.address)).to.equal(0n)
    expect(await vault.withdrawalsOf(poisonedPeriod, restoredUser.address)).to.equal(FRESH_DEPOSIT)
    expect(await vault.isWithdrawClaimed(poisonedPeriod, restoredUser.address)).to.equal(true)

    // After the full delay, the rescued historical position pays normally. This
    // isolates the failure to the restored source's newly funded entitlement.
    await time.increase(config.period_configuration_duration * 2)
    await expect(vault.connect(rescueRecipient).claimWithdraw(poisonedPeriod))
      .to.emit(vault, 'CompleteWithdraw')
      .withArgs(rescueRecipient.address, INITIAL_WITHDRAWAL, poisonedPeriod)

    expect(await fxrp.balanceOf(rescueRecipient.address)).to.equal(INITIAL_WITHDRAWAL)
    expect(await vault.pendingWithdrawAssets()).to.equal(FRESH_DEPOSIT)

    // The newly funded principal cannot be claimed: claim checks the tombstone
    // before reading the positive entitlement.
    await expect(vault.connect(restoredUser).claimWithdraw(poisonedPeriod))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(poisonedPeriod)

    // The only compliance recovery path is poisoned by the same source flag.
    await vault.connect(blocklister).addToBlocklist(restoredUser.address)
    await expect(
      vault
        .connect(rescuer)
        .rescueWithdrawFromBlocklisted(
          restoredUser.address,
          secondRescueRecipient.address,
          [poisonedPeriod],
        ),
    )
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(poisonedPeriod)

    // Neither restoration nor arbitrary additional periods clear the flag.
    await vault.connect(blocklister).removeFromBlocklist(restoredUser.address)
    await time.increase(config.period_configuration_duration * 10)
    await expect(vault.connect(restoredUser).claimWithdraw(poisonedPeriod))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(poisonedPeriod)

    expect(await vault.withdrawalsOf(poisonedPeriod, restoredUser.address)).to.equal(FRESH_DEPOSIT)
    expect(await vault.pendingWithdrawAssets()).to.equal(FRESH_DEPOSIT)
    expect(await vault.balanceOf(restoredUser.address)).to.equal(0n)
    expect(await fxrp.balanceOf(restoredUser.address)).to.equal(0n)

    console.log(
      JSON.stringify({
        poisonedPeriod: poisonedPeriod.toString(),
        rescuedHistoricalAssets: INITIAL_WITHDRAWAL.toString(),
        freshAssetsPermanentlyFrozen: FRESH_DEPOSIT.toString(),
        victimActiveSharesAfterFullRedeem: (await vault.balanceOf(restoredUser.address)).toString(),
        victimLiquidFxrp: (await fxrp.balanceOf(restoredUser.address)).toString(),
        staleClaimedFlag: await vault.isWithdrawClaimed(poisonedPeriod, restoredUser.address),
        vaultPendingWithdrawalAssets: (await vault.pendingWithdrawAssets()).toString(),
        finalCurrentPeriod: (await vault.currentPeriod()).toString(),
      }),
    )
  })
})
```

### Source — `poc/rescue-source-tombstone-fresh-redeem/run-poc.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

POC_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
REPO_DIR="$(cd -- "${POC_DIR}/../.." && pwd)"
EXPECTED_COMMIT="42f9ea5e43d88b35197b901acc2a8c24b314fa62"
EXPECTED_VAULT_HASH="778608250e5893fe908c6ebf7255cb559094066157f4e70b88950c94e509ab5f"
EXPECTED_STORAGE_HASH="24ce74c93880aba0ebfa6fec4c8c9759a5e8b5907aa88669ae985a8c6f246513"
EXPECTED_FASSET_HASH="75194713c07ea3c99a62f22a60f40729bb884ba1c18f3fce4a725595dddcb380"
EXPECTED_FASSET_PROXY_HASH="87d561360f1fd9ec192aa3dc8d17b9c317477d40b1e4c05ac8eb7eecbb27a958"

sha256_file() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | awk '{print $1}'
  elif command -v shasum >/dev/null 2>&1; then
    shasum -a 256 "$1" | awk '{print $1}'
  else
    echo "sha256sum or shasum is required" >&2
    exit 1
  fi
}

ACTUAL_COMMIT="$(git -C "${REPO_DIR}" rev-parse HEAD)"
ACTUAL_VAULT_HASH="$(sha256_file "${REPO_DIR}/contracts/core/FirelightVault.sol")"
ACTUAL_STORAGE_HASH="$(sha256_file "${REPO_DIR}/contracts/core/FirelightVaultStorage.sol")"
ACTUAL_FASSET_HASH="$(sha256_file "${REPO_DIR}/contracts/fasset/fassetToken/implementation/FAsset.sol")"
ACTUAL_FASSET_PROXY_HASH="$(sha256_file "${REPO_DIR}/contracts/fasset/fassetToken/implementation/FAssetProxy.sol")"

if [[ "${ACTUAL_COMMIT}" != "${EXPECTED_COMMIT}" ]]; then
  echo "wrong repository commit: expected ${EXPECTED_COMMIT}, got ${ACTUAL_COMMIT}" >&2
  exit 1
fi

if [[ "${ACTUAL_VAULT_HASH}" != "${EXPECTED_VAULT_HASH}" ]]; then
  echo "FirelightVault.sol hash mismatch" >&2
  exit 1
fi

if [[ "${ACTUAL_STORAGE_HASH}" != "${EXPECTED_STORAGE_HASH}" ]]; then
  echo "FirelightVaultStorage.sol hash mismatch" >&2
  exit 1
fi

if [[ "${ACTUAL_FASSET_HASH}" != "${EXPECTED_FASSET_HASH}" ]]; then
  echo "FAsset.sol hash mismatch" >&2
  exit 1
fi

if [[ "${ACTUAL_FASSET_PROXY_HASH}" != "${EXPECTED_FASSET_PROXY_HASH}" ]]; then
  echo "FAssetProxy.sol hash mismatch" >&2
  exit 1
fi

cd "${REPO_DIR}"
mkdir -p "${POC_DIR}/evidence"
npx hardhat test "${POC_DIR}/firelight-vault-rescue-source-tombstone.e2e.poc.js" \
  | tee "${POC_DIR}/evidence/local-e2e.log"
```

### Standalone clean reproduction

Requirements: Git, Node.js, and npm. No `.env` file, external RPC, or live account is required.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
mkdir -p poc/rescue-source-tombstone-fresh-redeem
```

Save the two complete code blocks above at these exact repository-relative paths:

```
poc/rescue-source-tombstone-fresh-redeem/firelight-vault-rescue-source-tombstone.e2e.poc.js
poc/rescue-source-tombstone-fresh-redeem/run-poc.sh
```

Then run:

```bash
chmod +x poc/rescue-source-tombstone-fresh-redeem/run-poc.sh
./poc/rescue-source-tombstone-fresh-redeem/run-poc.sh
```

The wrapper performs every commit and decisive-source hash check, runs only the targeted local Hardhat test, and writes the full console output to `poc/rescue-source-tombstone-fresh-redeem/evidence/local-e2e.log`.
