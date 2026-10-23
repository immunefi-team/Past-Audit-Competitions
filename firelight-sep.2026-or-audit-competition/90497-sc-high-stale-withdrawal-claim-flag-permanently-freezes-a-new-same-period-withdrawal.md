# 90497 sc high stale withdrawal claim flag permanently freezes a new same period withdrawal

**Submitted on Aug 25th 2026 at 09:59:35 UTC by @RustyLock for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90497
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A blocklisted user whose pending withdrawal is rescued and then unblocked can create a new withdrawal for the same period, but the vault permanently refuses both claim and rescue operations for the new withdrawal due to a stale terminal flag. The user's principal-derived funds remain frozen in the pending-withdrawal bucket with no in-contract recovery path.

## Vulnerability Details

`FirelightVault` implements compliance controls via `BLOCKLIST_ROLE` and `RESCUER_ROLE`. When a blocklisted account has a pending withdrawal, `rescueWithdrawFromBlocklisted` transfers the withdrawal shares to an unblocked recipient and marks the slot as claimed:

```solidity
// FirelightVault.sol:869-870
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

The `removeFromBlocklist` function clears only the blocklist status:

```solidity
// FirelightVault.sol:560-563
function removeFromBlocklist(address account) external onlyRole(BLOCKLIST_ROLE) onlyBlocklisted(account) {
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

It does not reset `isWithdrawClaimed` for any period. When the unblocked user deposits and withdraws again, `_requestWithdraw` targets `currentPeriod() + 1`. If the current period has not advanced since the rescue, the same period `P` is targeted again and `_requestWithdraw` credits the `(period, account)` slot:

```solidity
// FirelightVault.sol:1015-1017
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
```

No flag check or reset occurs. The user now has a nonzero `withdrawSharesOf[period][account]` balance alongside `isWithdrawClaimed[period][account] = true`.

When the period becomes claimable, `claimWithdraw` checks the flag before computing assets:

```solidity
// FirelightVault.sol:794-799
address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender],
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
```

The call reverts with `AlreadyClaimedPeriod(period)` despite the positive withdrawal balance.

Re-blocklisting the account and invoking `rescueWithdrawFromBlocklisted` also fails at the same flag check:

```solidity
// FirelightVault.sol:864
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

An exhaustive search of `FirelightVault.sol` confirms only two writes to `isWithdrawClaimed` (both `= true` in `claimWithdraw` and `rescueWithdrawFromBlocklisted`), with no reset, delete, or administrative setter. The frozen withdrawal has no in-contract recovery mechanism.

## Impact Details

The affected user's principal-derived withdrawal is permanently inaccessible. The new withdrawal amount remains in `pendingWithdrawAssets`; when the old rescued recipient successfully claims their unrelated withdrawal, the newly created amount remains retained in the pending-withdrawal accounting, while the affected user receives zero from that new withdrawal. While the user can avoid future losses by targeting later periods, the already-created withdrawal for period `P` cannot be claimed, rescued, rerouted, or refunded through any public or permissioned vault function.

This maps to the Critical impact **"Permanent freezing of funds"** under the competition scope. The root cause is a code-level invariant failure in `_requestWithdraw`, which permits `withdrawSharesOf[P][account] > 0` alongside `isWithdrawClaimed[P][account] = true`, violating the intended protocol constraint that any nonzero withdrawal balance must remain claimable or rescuable.

**Scope eligibility note:** The blocklist, rescue, and unblock operations act within their explicitly documented authority (`BLOCKLIST_ROLE` and `RESCUER_ROLE`). The defect is not privilege overreach or a malicious compliance decision. The report argues eligibility under the scope's "bypass of an intended protocol constraint" exception: the protocol constraint is that a newly created, nonzero withdrawal for a compliant receiver must remain accessible, but the stale flag permanently blocks all recovery paths after a correct role sequence completes. Historical audit finding M-02 addressed period-duration remapping and claim-flag reuse through append-only period configuration; this finding's trigger is the rescue operation's direct write to `isWithdrawClaimed[period][from] = true`, followed by same-period re-entry after unblocklisting, with no period-duration change involved.

## References

* Frozen source: `https://github.com/immunefi-team/audit-comp-firelight.git`, branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Affected contract: `contracts/core/FirelightVault.sol`\
  Direct blob: `https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol`
* Storage: `contracts/core/FirelightVaultStorage.sol`\
  Direct blob: `https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVaultStorage.sol`
* Key functions: `rescueWithdrawFromBlocklisted` (lines 845-876), `removeFromBlocklist` (lines 560-563), `_requestWithdraw` (lines 994-1028), `claimWithdraw` (lines 788-811)
* Storage mappings: `FirelightVaultStorage.sol` lines 68 and 71 (`withdrawSharesOf`, `isWithdrawClaimed`)

## Proof of Concept

## Proof of Concept

Add the following as a new test file at `test/poc/FirelightVaultWithdrawalRegression.test.js`. It uses the existing repository fixture at `test/setup/fixtures.js` and exercises only the vault's public interface and documented role calls. Run it with:

```shell
npx hardhat test test/poc/FirelightVaultWithdrawalRegression.test.js
```

The equivalent test was executed against the frozen source and passed with `1 passing (2s)`. It proves that the rescue operation sets the old `(period, account)` claim flag, a later compliant same-period withdrawal receives a positive balance without clearing that flag, the user's claim and a subsequent rescue both revert with `AlreadyClaimedPeriod`, and the rescued recipient's original withdrawal remains claimable.

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("../setup/fixtures.js");
const { expect } = require("chai");

describe("C1 rescued withdrawal slot poisoning", function () {
  it("strands a new same-period withdrawal after a rescued account is unblocked", async function () {
    const ctx = await loadFixture(
      deployVault.bind(null, {
        decimals: 0n,
        initial_deposit_limit: 100n,
      }),
    );

    const { firelight_vault, token_contract, blocklister, rescuer, users, utils, config } = ctx;
    const victim = users[0];
    const rescueRecipient = users[1];
    const amount = 10n;

    // The victim first has a normal withdrawal scheduled for period P.
    await utils.mintAndApprove(amount, victim);
    await firelight_vault.connect(victim).deposit(amount, victim.address);
    const withdrawalPeriod = (await firelight_vault.currentPeriod()) + 1n;
    await firelight_vault.connect(victim).withdraw(amount, victim.address, victim.address);
    expect(await firelight_vault.withdrawalsOf(withdrawalPeriod, victim.address)).to.equal(amount);

    // A standard compliance rescue transfers the old pending withdrawal and marks
    // the victim's P slot claimed.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address);
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      victim.address,
      rescueRecipient.address,
      [withdrawalPeriod],
    );
    expect(await firelight_vault.withdrawalsOf(withdrawalPeriod, victim.address)).to.equal(0n);
    expect(await firelight_vault.withdrawalsOf(withdrawalPeriod, rescueRecipient.address)).to.equal(amount);
    expect(await firelight_vault.isWithdrawClaimed(withdrawalPeriod, victim.address)).to.equal(true);

    // Once unblocked, the victim can re-enter normally. Because the current
    // period has not advanced, the new withdrawal targets the same P slot.
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address);
    await utils.mintAndApprove(amount, victim);
    await firelight_vault.connect(victim).deposit(amount, victim.address);
    expect(await firelight_vault.maxWithdraw(victim.address)).to.equal(amount);
    await firelight_vault.connect(victim).withdraw(amount, victim.address, victim.address);

    expect(await firelight_vault.withdrawalsOf(withdrawalPeriod, victim.address)).to.equal(amount);
    expect(await firelight_vault.isWithdrawClaimed(withdrawalPeriod, victim.address)).to.equal(true);
    expect(await firelight_vault.withdrawAssets(withdrawalPeriod)).to.equal(amount * 2n);

    // P becomes claimable. The rescue recipient receives the old withdrawal,
    // while the victim's new withdrawal is rejected as already claimed.
    await time.increase(config.period_configuration_duration);
    await time.increase(config.period_configuration_duration);
    await expect(firelight_vault.connect(victim).claimWithdraw(withdrawalPeriod))
      .to.be.revertedWithCustomError(firelight_vault, "AlreadyClaimedPeriod")
      .withArgs(withdrawalPeriod);
    await expect(firelight_vault.connect(rescueRecipient).claimWithdraw(withdrawalPeriod))
      .to.emit(firelight_vault, "CompleteWithdraw")
      .withArgs(rescueRecipient.address, amount, withdrawalPeriod);

    expect(await token_contract.balanceOf(victim.address)).to.equal(0n);
    expect(await token_contract.balanceOf(rescueRecipient.address)).to.equal(amount);
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(amount);

    // Re-blocklisting cannot move the victim's new P withdrawal because the
    // same stale flag makes rescueWithdrawFromBlocklisted reject it as claimed.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address);
    await expect(firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      victim.address,
      rescueRecipient.address,
      [withdrawalPeriod],
    ))
      .to.be.revertedWithCustomError(firelight_vault, "AlreadyClaimedPeriod")
      .withArgs(withdrawalPeriod);
  });
});
```
