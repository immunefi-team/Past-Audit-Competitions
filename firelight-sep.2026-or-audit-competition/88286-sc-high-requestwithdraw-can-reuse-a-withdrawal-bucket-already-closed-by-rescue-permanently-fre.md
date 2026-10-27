# 88286 sc high requestwithdraw can reuse a withdrawal bucket already closed by rescue permanently freezing fresh user funds

**Submitted on Aug 12th 2026 at 15:44:53 UTC by @CarlosMB for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88286
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted()` moves a blocklisted user's pending withdrawal to another address and marks the source `(period, account)` bucket as claimed:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

This can happen while that withdrawal period is still in the future.

If the account is legitimately removed from the blocklist before the current period ends, it can deposit fresh assets and request another withdrawal. `_requestWithdraw()` assigns the new request to the same `currentPeriod() + 1` bucket but does not check `isWithdrawClaimed[period][receiver]`:

```solidity
uint256 period = currentPeriod() + 1;

withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
```

The new withdrawal succeeds and burns the user's shares even though the destination bucket is already terminal.

The resulting state is:

```
isWithdrawClaimed[P+1][user] == true
withdrawSharesOf[P+1][user] > 0
```

From that point:

* `claimWithdraw(P+1)` always reverts with `AlreadyClaimedPeriod`;
* re-blocklisting the user and calling `rescueWithdrawFromBlocklisted()` also reverts on the same flag;
* the newly withdrawn vault shares have already been burned; and
* there is no cancellation, reassignment, or flag-reset path.

Therefore the fresh withdrawal is permanently inaccessible under the current implementation.

## Vulnerability Details

Assume the vault is currently in period `P`.

1. Alice creates a normal withdrawal targeting `P+1`.
2. Alice is legitimately blocklisted.
3. `RESCUER_ROLE` rescues Alice's pending withdrawal for `P+1`.
4. The rescue leaves:

```
withdrawSharesOf[P+1][Alice] = 0
isWithdrawClaimed[P+1][Alice] = true
```

5. The rescuer may also rescue all of Alice's remaining active shares.
6. Alice is legitimately removed from the blocklist while the vault is still in `P`.
7. Alice makes a completely fresh deposit.
8. Alice calls `withdraw(..., Alice, Alice)` before `P` ends.
9. `_requestWithdraw()` again selects `P+1` and credits new withdrawal shares to Alice despite `isWithdrawClaimed[P+1][Alice] == true`.
10. Alice's fresh vault shares are burned.
11. Once `P+1` matures, Alice cannot claim the withdrawal.
12. Re-blocklisting Alice does not help because the rescue path also rejects the already-marked source bucket.

No privileged role needs to behave maliciously. The privileged calls only perform supported compliance actions; the freeze occurs when the restored user later performs a normal permissionless deposit/withdrawal flow.

## Root Cause

`isWithdrawClaimed[period][account]` is treated as a terminal state by both available settlement paths, but `_requestWithdraw()` does not enforce that terminal state before writing new withdrawal shares.

There should never be a state where:

```
isWithdrawClaimed[period][receiver] == true
&&
withdrawSharesOf[period][receiver] > 0
```

but rescue can mark the future `P+1` bucket as claimed while the vault is still in `P`, and `_requestWithdraw()` can subsequently recreate value in that same bucket.

This invariant is already enforced in `rescueWithdrawFromBlocklisted()` for the destination account:

```solidity
if (isWithdrawClaimed[periods[i]][to])
    revert AlreadyClaimedPeriod(periods[i]);
```

but the equivalent check is missing from `_requestWithdraw()`.

## Impact

**Permanent freezing of user funds.**

The amount is not bounded to dust. After being removed from the blocklist, the account can make a fresh deposit of any amount permitted by normal vault limits and then burn those shares into the unusable withdrawal bucket.

An upgrade could theoretically modify storage or contract logic, but there is no recovery path available through the current protocol implementation.

## Proof of Concept

## Proof of Concept

Create `test/critical_rescue_claimed_bucket_reuse.js`:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('Critical: rescued claimed bucket can be reused and permanently freeze funds', function () {
  it('accepts a fresh withdrawal into an already-claimed bucket that cannot be claimed or rescued', async function () {
    const {
      token_contract: token,
      firelight_vault: vault,
      users,
      utils,
      blocklister,
      rescuer,
      config
    } = await loadFixture(deployVault)

    const [alice, recovery, secondRecovery] = users

    const INITIAL_DEPOSIT = ethers.parseUnits('100', 6)
    const FIRST_WITHDRAW = ethers.parseUnits('40', 6)
    const FRESH_DEPOSIT = ethers.parseUnits('60', 6)

    await utils.mintAndApprove(INITIAL_DEPOSIT + FRESH_DEPOSIT, alice)

    await vault.connect(alice).deposit(INITIAL_DEPOSIT, alice.address)

    const targetPeriod = (await vault.currentPeriod()) + 1n

    // Initial withdrawal targets P+1.
    await vault.connect(alice).withdraw(
      FIRST_WITHDRAW,
      alice.address,
      alice.address
    )

    // Legitimate compliance flow.
    await vault.connect(blocklister).addToBlocklist(alice.address)

    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      alice.address,
      recovery.address,
      [targetPeriod]
    )

    expect(
      await vault.withdrawSharesOf(targetPeriod, alice.address)
    ).to.equal(0n)

    expect(
      await vault.isWithdrawClaimed(targetPeriod, alice.address)
    ).to.equal(true)

    // Rescue all remaining active shares as well.
    await vault.connect(rescuer).rescueSharesFromBlocklisted(
      alice.address,
      recovery.address
    )

    expect(await vault.balanceOf(alice.address)).to.equal(0n)

    // Alice is legitimately restored while still in the same period P.
    await vault.connect(blocklister).removeFromBlocklist(alice.address)

    // Completely fresh capital after restoration.
    await vault.connect(alice).deposit(FRESH_DEPOSIT, alice.address)

    expect(await vault.balanceOf(alice.address)).to.equal(FRESH_DEPOSIT)

    // Still in P, so this again targets P+1.
    // The request succeeds despite Alice's P+1 bucket already being terminal.
    await expect(
      vault.connect(alice).withdraw(
        FRESH_DEPOSIT,
        alice.address,
        alice.address
      )
    ).to.emit(vault, 'WithdrawRequest')

    expect(await vault.balanceOf(alice.address)).to.equal(0n)

    expect(
      await vault.withdrawSharesOf(targetPeriod, alice.address)
    ).to.equal(FRESH_DEPOSIT)

    expect(
      await vault.isWithdrawClaimed(targetPeriod, alice.address)
    ).to.equal(true)

    // Mature P+1. A claim requires targetPeriod < currentPeriod().
    await time.increase(config.period_configuration_duration * 2)

    // The fresh withdrawal cannot be claimed.
    await expect(
      vault.connect(alice).claimWithdraw(targetPeriod)
    )
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(targetPeriod)

    // Re-blocklisting does not create a recovery path.
    await vault.connect(blocklister).addToBlocklist(alice.address)

    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        alice.address,
        secondRecovery.address,
        [targetPeriod]
      )
    )
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(targetPeriod)

    // The original legitimately rescued withdrawal remains independently claimable.
    const recoveryBalanceBefore = await token.balanceOf(recovery.address)

    await vault.connect(recovery).claimWithdraw(targetPeriod)

    expect(await token.balanceOf(recovery.address)).to.equal(
      recoveryBalanceBefore + FIRST_WITHDRAW
    )

    // Alice's fresh withdrawal remains accounted as pending but has no claim/rescue path.
    expect(await vault.pendingWithdrawAssets()).to.equal(FRESH_DEPOSIT)

    expect(
      await vault.withdrawSharesOf(targetPeriod, alice.address)
    ).to.equal(FRESH_DEPOSIT)
  })
})
```

Run:

```bash
npx hardhat test test/critical_rescue_claimed_bucket_reuse.js
```

The PoC passes and demonstrates that fresh user shares are burned into a pending withdrawal for which both `claimWithdraw()` and `rescueWithdrawFromBlocklisted()` are permanently disabled.

## Recommended Mitigation

Reject withdrawal creation when the receiver's target-period bucket is already marked claimed:

```solidity
uint256 period = currentPeriod() + 1;

if (isWithdrawClaimed[period][receiver]) {
    revert AlreadyClaimedPeriod(period);
}
```

This check should execute before updating withdrawal accounting or burning the owner's shares.
