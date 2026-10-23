# 89929 sc high inclusive period start lookup lets exact boundary withdrawals reduce incident payouts and potentially cause protocol insolvency

**Submitted on Aug 21st 2026 at 22:24:06 UTC by @satyaorz for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89929
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Descriptive Summary

When `withdraw()` executes at the exact timestamp of a new period, it records post-withdrawal active assets under the same timestamp that `payout()` later queries as the period-start exposure baseline. Because that historical lookup is inclusive, it returns the reduced post-withdrawal value. If an approved incident exceeds the first-loss buffer, the vault leg can therefore be underpaid even though the pending withdrawal remains physically in the vault; the incident is then permanently closed with the shortfall.

## Brief/Intro

`FirelightVault` expressly keeps withdrawals requested during an incident's capture period exposed to that incident. At the exact timestamp at which a new period begins, however, an ordinary shareholder withdrawal can make the post-withdrawal `totalAssets` value become the historical exposure cap for the new period.

If the withdrawal is included in a block whose timestamp equals the new period start, `withdraw()` records the post-withdrawal active assets with key `T`. `payout()` uses an inclusive historical lookup at `T`, so it treats that post-withdrawal state as the assets committed at period start. If the user owns and withdraws all active assets, the vault leg is capped at zero even though the assets remain in the pending-withdrawal bucket and are later claimed in full. A partial boundary withdrawal reduces the cap by the withdrawn amount and can cause a proportional shortfall.

This lets an unprivileged holder reduce a period's effective incident exposure without removing the underlying tokens from the vault. When an approved loss exceeds the first-loss-buffer payment and the resulting vault request exceeds the corrupted cap, beneficiaries receive less than the approved amount. At sufficient scale this leaves the cover system undercollateralized or insolvent.

## Vulnerability Details

#### 1. Withdrawal logs a post-withdrawal checkpoint at the current timestamp

Both withdrawal entry points call `_logTrace` with the post-withdrawal asset total:

```solidity
// contracts/core/FirelightVault.sol:755-777
function withdraw(
    uint256 assets,
    address receiver,
    address owner
)
    public
    override
    whenNotPaused
    notBlocklisted(_msgSender())
    notBlocklisted(owner)
    notBlocklisted(receiver)
    nonReentrant
    returns (uint256)
{
    if (assets == 0) revert InvalidAmount();

    (uint256 shares, uint256 _totalSupply, uint256 _totalAssets) =
        _previewTotals(assets, true, Math.Rounding.Ceil);

    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);

    return shares;
}
```

`_logTrace` uses `Time.timestamp()` as the checkpoint key:

```solidity
// contracts/core/FirelightVault.sol:1044-1056
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

The withdrawal is assigned to `currentPeriod() + 1` and the assets remain in the pending-withdrawal accounting:

```solidity
// contracts/core/FirelightVault.sol:994-1018
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
pendingWithdrawAssets += assets;
```

Therefore, at a period boundary `T`, withdrawing all active vault assets can produce both:

```
_traceTotalAssets[T] = 0
withdrawAssets[capturePeriod + 1] = withdrawn amount
asset.balanceOf(vault) = withdrawn amount
```

#### 2. The inclusive lookup treats the boundary transaction's post-state as the period-start baseline

`totalAssetsAt()` uses `upperLookupRecent`, which returns the checkpoint with a key less than or equal to the requested timestamp:

```solidity
// contracts/core/FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

Same-key replacement is not required for the attack. Ordinarily the last asset checkpoint has a key below `T`; the boundary withdrawal appends the first checkpoint at `T`. Because `upperLookupRecent(T)` is inclusive, it returns that post-withdrawal checkpoint rather than the pre-boundary value. If another vault write also occurs at `T`, `Trace256.push` replaces same-key values, making the last vault state-changing transaction at that timestamp decisive.

#### 3. `payout()` applies the reduced snapshot before pending withdrawals

The payout code computes the current active assets and includes the relevant pending withdrawal bucket, but the final amount is still capped by `totalAssetsAt(capturePeriodStart)`:

```solidity
// contracts/core/FirelightVault.sol:900-927
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals;
paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

With a full boundary withdrawal:

```
assetsAtCapturePeriod = 0
activePayableAmount    = withdrawn amount
paidAmount             = min(requested amount, 0, withdrawn amount) = 0
```

The vault payout is therefore processed with zero payment. The withdrawal bucket is not consumed by this zero payout, and the sole shareholder can later claim the full amount through `claimWithdraw()`.

For partial ownership, let `A` be active assets immediately before the boundary and `X` the boundary withdrawal. The lookup returns `A - X`, so the maximum vault payment is reduced by up to `X`. The payout waterfall still charges the pending withdrawal proportionally; it does not generally let a minority withdrawer claim all of `X`.

The PoC's partial case starts with `A = 100`, withdraws `X = 20`, and requests 100. The vault pays only 80, leaving 16 active units and 4 pending units. The withdrawer later claims 4. This proves both the beneficiary shortfall and the correctly bounded personal benefit for a minority withdrawer.

#### 4. The first-loss buffer is paid first, but `IncidentManager` permanently closes after a partial or zero vault payment

`IncidentManager` first transfers the available first-loss buffer and requests only the remaining loss from the vault:

```solidity
// contracts/core/IncidentManager.sol:643-684
uint256 remainderAmount = totalAssessmentLoss;
...
remainderAmount = flbAmountCanonical >= totalAssessmentLoss
    ? 0
    : totalAssessmentLoss - flbAmountCanonical;
...
vaultPaidAmount = $.vault.payout(receiver, vaultRequestedAmount, captureTimestamp);
```

Accordingly, this finding affects incidents whose loss exceeds the available first-loss-buffer contribution and whose vault remainder exceeds the reduced period-start cap. That is a normal large-loss condition, not an assumption that the buffer role acts incorrectly.

After `vault.payout()` returns, `approveCurrentAssessment()` closes the incident without requiring `vaultPaidAmount` to equal the requested vault amount or the approved loss:

```solidity
// contracts/core/IncidentManager.sol:423-438
uint256 vaultPaidAmount = _executePayout(...);
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
_decreaseIncident(incident.period);
emit IncidentClosed(incidentId, vaultPaidAmount, totalAssessmentLoss);
```

There is no remaining on-chain liability or retry path for the unpaid portion.

### Attack Path

1. The attacker owns a material amount of vault shares and anticipates or can coordinate a covered incident after the next period boundary `T`.
2. The attacker monitors `T` and, after the preceding block, submits a normal `withdraw()` transaction with sufficient priority for inclusion in the block whose EVM timestamp is exactly `T`.
3. The withdrawal burns the shares, records the assets in `withdrawAssets[currentPeriod + 1]`, leaves the ERC-20 assets physically in the vault, and writes the post-withdrawal `totalAssets` checkpoint under key `T`.
4. A valid incident is captured in the new period. The normal payout flow pays the available first-loss buffer and requests the remaining approved loss from the vault.
5. If that vault remainder exceeds the reduced `A - X` snapshot, `payout()` underpays by up to `X`. If `X == A`, its vault payment is zero despite the pending withdrawal being included in `activePayableAmount`.
6. `IncidentManager` closes the incident using the partial or zero return.
7. After the payout window, the attacker claims the amount remaining in their pending-withdrawal position. In the all-assets case this is the full amount; in a partial case the payout waterfall reduces it proportionally.

No privileged role needs to make an invalid assessment, bypass a role check, or intentionally process an incorrect payout. The attacker only uses the ordinary public `withdraw()` function. The payout role performs its normal operation for a valid incident.

## Impact Details

The vault's historical exposure cap no longer represents the assets exposed immediately before the capture period. For pre-boundary assets `A` and boundary withdrawal `X`, the cap becomes `A - X`. If the post-buffer vault request is at least `A - X`, the beneficiary shortfall can be as large as `X`.

At meaningful scale, this can leave approved incident beneficiaries unpaid and the cover system undercollateralized or insolvent. The withdrawer's personal benefit is smaller than the beneficiary shortfall when the withdrawer owns only part of the vault, because the payout is charged proportionally to active and pending assets. For example, the PoC's 20-of-100 withdrawal causes a 20-unit underpayment but leaves only 4 units claimable by the withdrawer. A zero vault payout and full withdrawal claim require the user to control and withdraw all active assets.

The issue is not generic withdrawal behavior: the one-second-late control preserves the 100-unit period-start cap and pays the requested 100 units. The selected in-scope impact is **Protocol insolvency** because an approved covered loss can be permanently closed with less than its valid payout while collateral that should have backed that loss remains in the vault.

### Exploitability Caveat

The accounting behavior is deterministic once the transaction is included at the exact boundary timestamp. Exact-block inclusion is a timing condition, not a guaranteed outcome for every attempt.

Read-only Flare RPC revalidation found actual boundary blocks for the deployed daily configuration. In a 14-period sample ending at period 262, 12 boundaries had a block timestamp exactly equal to the configured period start. All 12 exact-boundary blocks contained transactions and none contained a call to the deployed vault proxy. Ten of the 12 were immediately preceded by a block at `T - 1`, providing an observable broadcast point. Examples include:

```
periodConfigurations(0): epoch = 1764682969, duration = 86400
block 67921346: timestamp = 1787319769 (current period boundary)
block 67843343: timestamp = 1787233369 (previous period boundary)

Exact-boundary blocks in the 14-period sample:
67921346, 67843343, 67764965, 67689328, 67613933, 67538763,
67464427, 67313847, 67238441, 67162912, 67087558, 67012090

Skipped boundaries:
target 1786714969 -> block 67389042 at 1786714976
target 1786196569 -> block 66937153 at 1786196570
```

Thus the exact boundary is observable on the target chain and is not only a local-test artifact. The remaining condition is that the attacker's transaction is included in the exact `T` block. Flare's proposer-controlled transaction ordering makes this plausible with carefully timed submission and sufficient priority, but inclusion cannot be guaranteed.

The withdrawal does not need to be first in the block: all transactions in a block share the same timestamp. A later `FirelightVault` transaction in the same block could write another same-key checkpoint, although none of the 12 verified exact-boundary blocks contained a call to the deployed vault proxy.

The attacker must also anticipate or coordinate a covered incident after `T`; this is not a reactive withdrawal performed after learning of a historical loss. A meaningful personal benefit requires a material vault position. The first-loss buffer may absorb smaller incidents entirely, so the corrupted vault cap matters only for the loss remaining after the buffer contribution.

### Why Existing Guards Do Not Prevent It

* `withdraw()` has normal pause, blocklist, allowance, share-balance, and reentrancy checks, but no check preventing execution at the period boundary.
* `payout()` intentionally allows `paidAmount == 0`; it does not reject a zero exposure snapshot or preserve an unpaid liability for the pending withdrawal.
* The withdrawal delay makes the assets claimable only later, but does not make them fully available to the incident payout after the snapshot has been reduced.
* The payout window check only validates the incident period; it does not distinguish a genuine period-start exposure snapshot from a post-withdrawal checkpoint written at the same timestamp.
* The first-loss buffer reduces the amount requested from the vault but does not repair the corrupted cap for a larger remainder.
* `IncidentManager` accepts the partial or zero vault return and transitions the incident to `CLOSED`, so the unpaid portion is not retried.

## References

* [`FirelightVault.sol:755-777`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L755-L777) — withdrawal records post-withdrawal assets.
* [`FirelightVault.sol:456-458`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L456-L458) — inclusive historical `totalAssetsAt` lookup.
* [`FirelightVault.sol:900-927`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L900-L927) — payout cap and pending-withdrawal accounting.
* [`FirelightVault.sol:994-1018`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1018) — pending withdrawal bucket.
* [`FirelightVault.sol:1044-1056`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1044-L1056) — checkpoint key is the current EVM timestamp.
* [`Checkpoints.sol:198-223`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L198-L223) — inclusive lookup and secondary same-key ordering behavior.
* [`IncidentManager.sol:621-688`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L621-L688) — first-loss-buffer waterfall and vault remainder request.
* [`IncidentManager.sol:403-438`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L403-L438) — incident is closed after the partial or zero vault return.
* `poc_period_boundary_snapshot.js` — runnable full-withdrawal, partial-ownership, and timing-control tests (included inline in the PoC section).
* [Flare Network overview](https://dev.flare.network/network/overview) — block timing and proposer-controlled transaction ordering.
* [Firelight competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/) — `FirelightVault.sol`, protocol insolvency impact, and trusted-role rules.

## Suggested Mitigation

Do not let a user operation executed at timestamp `T` define the pre-existing exposure baseline for the period that begins at `T`. Suitable fixes include:

1. Store an explicit period-start active-assets snapshot before any user operation can write a checkpoint for that period, and use that immutable value in `payout()`; or
2. Query the last checkpoint strictly before the period start (for example, the checkpoint at or before `T - 1`) and separately account for assets that are intentionally admitted at the boundary.

The implementation should also preserve an outstanding incident liability when `paidAmount < amount`; closing after a partial or zero payment while allowing the remaining pending assets to be claimed makes the shortfall permanent.

## Proof of Concept

The attached PoC is a runnable Hardhat test using the production `FirelightVault` implementation and its production `withdraw`, `payout`, and `claimWithdraw` paths. It uses the deployed configuration's daily period duration (`86400` seconds). Because the defect is in the vault leg, the test calls the same `onlyRole(PAYOUT_ROLE)` entry point that `IncidentManager` calls after the first-loss-buffer transfer; the production `IncidentManager` call and permanent close behavior are traced above from source.

### Step-by-step reproduction

1. Deploy the production `FirelightVault` implementation through the existing test fixture and configure its period duration to one day.
2. Mint and approve 100 asset units for an ordinary user, then deposit them into the vault.
3. Advance to one second before the next period boundary and confirm that the pre-boundary historical `totalAssetsAt` value is 100.
4. Set the next block timestamp to the exact boundary and have the ordinary user call `withdraw(100, user, user)`.
5. Confirm that the assets remain in the vault and in `withdrawAssets(currentPeriod + 1)`, but `totalAssetsAt(periodStart)` is now zero.
6. Call the normal vault `payout()` entry point for a valid incident captured one second into the new period. The emitted `PayoutExecuted` event reports `paidAmount == 0`.
7. Advance beyond the payout window and call `claimWithdraw()`. The user receives the full 100 asset units that escaped the incident payout.
8. Run the partial-ownership case: deposit 80 units for one user and 20 for the boundary withdrawer. A 100-unit request pays only 80, leaves 16 active and 4 pending, and the withdrawer later claims 4.
9. Run the control case with the withdrawal included one second after the boundary. The period-start snapshot remains 100 and the payout transfers the requested 100 units.

Run from the repository's `repo` directory:

```bash
npx hardhat test test/exploration/poc_period_boundary_snapshot.js
```

Expected result:

```
3 passing
```

The first test proves the all-assets sequence: the exact-boundary withdrawal leaves the assets in the vault and pending bucket, makes `totalAssetsAt(periodStart) == 0`, causes `paidAmount == 0`, and later claims the full amount. The second test proves the bounded partial-ownership impact with explicit before/after balances. The third is the one-second-late control and proves that the effect depends on the exact period-start timestamp.

### Attached PoC: `test/exploration/poc_period_boundary_snapshot.js`

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { deployVault } = require('../setup/fixtures.js')

const E6 = 10n ** 6n
const DEPOSIT = 100n * E6
// Match the deployed Flare vault's currently configured daily period. The bug
// is duration-independent, but using 86400 makes the reproduction mirror the
// live configuration.
const boundaryFixture = async () => deployVault({ period_configuration_duration: 86400 })

describe('Period-start withdrawal exposure snapshot', function () {
  it('makes an exact-boundary withdrawal of all vault assets the period-start snapshot and caps payout at zero', async function () {
    const ctx = await loadFixture(boundaryFixture)
    const { firelight_vault: vault, token_contract: asset, users, payout_signer: payoutSigner, payout_receiver: receiver } = ctx
    const user = users[0]

    await ctx.utils.mintAndApprove(DEPOSIT, user)
    await vault.connect(user).deposit(DEPOSIT, user.address)

    const firstPeriodEnd = await vault.currentPeriodEnd()
    // Mine immediately before the boundary, then force the withdrawal transaction
    // itself into the first block whose timestamp equals the next period start.
    await time.increaseTo(firstPeriodEnd - 1n)
    expect(await vault.totalAssetsAt(firstPeriodEnd - 1n)).to.equal(DEPOSIT)
    await ethers.provider.send('evm_setNextBlockTimestamp', [firstPeriodEnd.toString()])
    await vault.connect(user).withdraw(DEPOSIT, user.address, user.address)

    const capturePeriod = await vault.currentPeriod()
    expect(capturePeriod).to.equal(1n)
    expect(await ethers.provider.getBlock('latest').then((b) => BigInt(b.timestamp))).to.equal(firstPeriodEnd)

    // The withdrawal remains physically in the vault and in the capture-period
    // exposure bucket, but the inclusive period-start lookup selects the
    // post-withdrawal checkpoint written at the boundary.
    expect(await asset.balanceOf(vault.target)).to.equal(DEPOSIT)
    expect(await vault.pendingWithdrawAssets()).to.equal(DEPOSIT)
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(DEPOSIT)
    expect(await vault.totalAssets()).to.equal(0n)
    expect(await vault.totalAssetsAt(firstPeriodEnd)).to.equal(0n)

    const captureTimestamp = firstPeriodEnd + 1n
    await expect(
      vault.connect(payoutSigner).payout(receiver.address, DEPOSIT, captureTimestamp)
    ).to.emit(vault, 'PayoutExecuted').withArgs(receiver.address, DEPOSIT, 0n, captureTimestamp)

    expect(await asset.balanceOf(vault.target)).to.equal(DEPOSIT)
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(DEPOSIT)

    // Once the P1/P2 payout window closes, the same pending withdrawal becomes
    // claimable, proving that the boundary checkpoint let the collateral escape.
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.equal(capturePeriod + 2n)
    await vault.connect(user).claimWithdraw(capturePeriod + 1n)
    expect(await asset.balanceOf(user.address)).to.equal(DEPOSIT)
  })

  it('reduces payout proportionally when the boundary withdrawer owns only part of the vault', async function () {
    const ctx = await loadFixture(boundaryFixture)
    const { firelight_vault: vault, token_contract: asset, users, payout_signer: payoutSigner, payout_receiver: receiver } = ctx
    const activeUser = users[0]
    const withdrawer = users[1]
    const activeDeposit = 80n * E6
    const boundaryWithdrawal = 20n * E6

    await ctx.utils.mintAndApprove(activeDeposit, activeUser)
    await vault.connect(activeUser).deposit(activeDeposit, activeUser.address)
    await ctx.utils.mintAndApprove(boundaryWithdrawal, withdrawer)
    await vault.connect(withdrawer).deposit(boundaryWithdrawal, withdrawer.address)

    const firstPeriodEnd = await vault.currentPeriodEnd()
    await time.increaseTo(firstPeriodEnd - 1n)
    await ethers.provider.send('evm_setNextBlockTimestamp', [firstPeriodEnd.toString()])
    await vault.connect(withdrawer).withdraw(boundaryWithdrawal, withdrawer.address, withdrawer.address)

    const capturePeriod = await vault.currentPeriod()
    expect(await vault.totalAssetsAt(firstPeriodEnd)).to.equal(activeDeposit)
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(boundaryWithdrawal)

    // The 20-unit boundary withdrawal lowers the cap from 100 to 80. The
    // proportional waterfall consumes 64 active units and 16 pending units,
    // leaving 16 active and 4 claimable instead of paying the requested 100.
    const captureTimestamp = firstPeriodEnd + 1n
    await expect(
      vault.connect(payoutSigner).payout(receiver.address, DEPOSIT, captureTimestamp)
    ).to.emit(vault, 'PayoutExecuted').withArgs(receiver.address, DEPOSIT, activeDeposit, captureTimestamp)

    expect(await asset.balanceOf(receiver.address)).to.equal(activeDeposit)
    expect(await vault.totalAssets()).to.equal(16n * E6)
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(4n * E6)

    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await vault.connect(withdrawer).claimWithdraw(capturePeriod + 1n)
    expect(await asset.balanceOf(withdrawer.address)).to.equal(4n * E6)
  })

  it('keeps the pre-period snapshot when the withdrawal is one second after the boundary', async function () {
    const ctx = await loadFixture(boundaryFixture)
    const { firelight_vault: vault, users, payout_signer: payoutSigner, payout_receiver: receiver } = ctx
    const user = users[0]

    await ctx.utils.mintAndApprove(DEPOSIT, user)
    await vault.connect(user).deposit(DEPOSIT, user.address)

    const firstPeriodEnd = await vault.currentPeriodEnd()
    await time.increaseTo(firstPeriodEnd)
    await ethers.provider.send('evm_setNextBlockTimestamp', [(firstPeriodEnd + 1n).toString()])
    await vault.connect(user).withdraw(DEPOSIT, user.address, user.address)

    expect(await vault.totalAssetsAt(firstPeriodEnd)).to.equal(DEPOSIT)
    const captureTimestamp = firstPeriodEnd + 2n
    await expect(
      vault.connect(payoutSigner).payout(receiver.address, DEPOSIT, captureTimestamp)
    ).to.emit(vault, 'PayoutExecuted').withArgs(receiver.address, DEPOSIT, DEPOSIT, captureTimestamp)
  })
})
```
