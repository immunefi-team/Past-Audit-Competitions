# 89238 sc medium unblocking after rescue can permanently freeze same period withdrawals

**Submitted on Aug 18th 2026 at 02:24:17 UTC by @chiranz for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89238
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` in addition to zeroing the rescued user's withdraw shares. No code path anywhere in the repository ever sets that flag back to `false`. If the address is subsequently un-blocklisted via `removeFromBlocklist` — the intended outcome of a false positive or a lifted sanction — and the user calls `withdraw()` again while still in the same period, the new request is credited to the same poisoned `(period, user)` slot. Their vault shares are burned, but the resulting assets can never be claimed by the user nor recovered by the rescuer. They are permanently frozen in the vault.

### Vulnerability Details

Withdrawals are batched per period. A `withdraw()` during period `N` burns the caller's vault shares immediately and credits `withdrawSharesOf[N+1][receiver]`; after period `N+1` elapses the user calls `claimWithdraw(N+1)` to receive assets. Two storage slots track this per `(period, account)`: `withdrawSharesOf` (amount owed) and `isWithdrawClaimed` (a bool).

The rescue function writes both (`FirelightVault.sol:868-870`):

```solidity
withdrawSharesOf[periods[i]][to]   += _withdrawOf;  // :868
withdrawSharesOf[periods[i]][from]  = 0;            // :869
isWithdrawClaimed[periods[i]][from] = true;         // :870  <-- the defect
```

Line `:869` alone is already sufficient to prevent `from` double-claiming, because `claimWithdraw` reverts with `NoWithdrawalAmount` at `:803` when the computed assets are zero. Line `:870` is therefore redundant for its apparent purpose, but it permanently poisons the `(period, from)` slot.

The flag is never cleared. Repository-wide, `isWithdrawClaimed` has exactly two writers — `FirelightVault.sol:806` (inside `claimWithdraw`) and `FirelightVault.sol:870` (inside the rescue) — and both assign `true`. There is no clearer, no admin reset, and no hook. `removeFromBlocklist` (`:560-563`) only mutates `isBlocklisted`.

`_requestWithdraw` (`:994-1028`) never reads the flag. It unconditionally credits the slot and burns the shares:

```solidity
withdrawSharesOf[period][receiver] += sharesWithdraw;  // :1017
...
_update(owner, address(0), shares);                    // :1025  shares burned
```

Both recovery paths are then closed by the same guard:

* `claimWithdraw` checks the claimed bit at `:794`, **before** it reads the share balance at `:797`, so it reverts with `AlreadyClaimedPeriod` regardless of the non-zero balance.
* `rescueWithdrawFromBlocklisted` checks the identical bit at `:864`, so re-blocklisting and re-rescuing also reverts with `AlreadyClaimedPeriod`.

The assets remain accounted in `pendingWithdrawAssets`, so they are unusable by the user and simultaneously excluded from `totalAssets()` for everyone else.

#### Timing

The dangerous window is the remainder of period `N`, after a rescue was performed for bucket `N+1` during period `N`. A withdrawal in that window lands in the poisoned bucket and is destroyed. Waiting until period `N+1` targets bucket `N+2` and is unaffected. The user receives no on-chain signal: `withdraw()` succeeds, emits a normal `WithdrawRequest`, and burns the shares. The loss only surfaces a period later when `claimWithdraw` reverts, at which point it is irreversible.

### Impact Details

Permanent freezing of funds. The affected user's vault shares are burned and the corresponding assets are locked in the vault with no function capable of releasing them — not by the user, not by the rescuer, not by any admin role. The amount is bounded by whatever the un-blocklisted user withdraws to themselves during the remainder of that one period.

Note that no privileged role behaves maliciously or exceeds its documented permissions at any point. Block to rescue to unblock is the protocol's intended compliance flow, executed correctly. The freeze is a state-machine defect that materialises afterwards, against a user in good standing, through an ordinary `withdraw()` call.

### Recommended fix

Remove the redundant write:

```diff
  withdrawSharesOf[periods[i]][to]   += _withdrawOf;
  withdrawSharesOf[periods[i]][from]  = 0;
- isWithdrawClaimed[periods[i]][from] = true;
```

Zeroing the shares on `:869` already blocks the double-claim the flag was presumably intended to prevent. If the flag must be retained for accounting reasons, then `claimWithdraw` should evaluate the share balance before the claimed bit, and the rescue should clear the flag whenever it zeroes the shares.

## References

All line numbers refer to branch `v1_audit_ready` of `immunefi-team/audit-comp-firelight`.

* [`rescueWithdrawFromBlocklisted` — defect at L870](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845-L876)
* [`claimWithdraw` — claimed-bit guard at L794 precedes the share read at L797](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L811)
* [`_requestWithdraw` — credits the slot at L1017, burns shares at L1025](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028)
* [`removeFromBlocklist` — clears only `isBlocklisted`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L560-L563)
* [`isWithdrawClaimed` storage declaration](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVaultStorage.sol#L71)

## Proof of Concept

### Runnable PoC

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const DECIMALS = 6
const PERIOD = 604800
const U = (n) => ethers.parseUnits(String(n), DECIMALS)

async function seededVault() {
  const ctx = await deployVault({
    decimals: DECIMALS,
    initial_deposit_limit: U(1_000_000),
    period_configuration_duration: PERIOD
  })
  const [alice, bob, carol] = ctx.users
  await ctx.utils.mintAndApprove(U(10_000), alice)
  await ctx.utils.mintAndApprove(U(10_000), bob)
  await ctx.utils.mintAndApprove(U(10_000), carol)
  return ctx
}

describe('H-V-001', function () {
  it('post-rescue same-period withdraw is permanently unclaimable', async () => {
    const ctx = await loadFixture(seededVault)
    const { firelight_vault: vault, token_contract, blocklister, rescuer, users } = ctx
    const alice = users[0]
    const beneficiary = users[1]

    // 1. Alice deposits 1000 and requests a withdrawal of 400 during period N.
    await vault.connect(alice).deposit(U(1000), alice.address)

    const p0 = await vault.currentPeriod()
    await vault.connect(alice).withdraw(U(400), alice.address, alice.address)
    const withdrawPeriod = p0 + 1n

    // 2. Intended compliance flow: blocklist Alice, rescue her pending withdrawal.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      alice.address,
      beneficiary.address,
      [withdrawPeriod]
    )

    // FirelightVault.sol:869-870 — shares zeroed AND the claimed flag latched true.
    expect(await vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(0n)
    expect(await vault.isWithdrawClaimed(withdrawPeriod, alice.address)).to.equal(true)

    // 3. Alice is cleared. Only isBlocklisted is reset (FirelightVault.sol:560-563).
    await vault.connect(blocklister).removeFromBlocklist(alice.address)

    // 4. She still holds 600 shares and does the obvious thing: withdraw them.
    //    Still period N, so this targets the same (poisoned) bucket.
    await vault.connect(alice).withdraw(U(600), alice.address, alice.address)

    // Shares burned, slot credited, flag still true.
    expect(await vault.withdrawSharesOf(withdrawPeriod, alice.address)).to.equal(U(600))
    expect(await vault.isWithdrawClaimed(withdrawPeriod, alice.address)).to.equal(true)
    expect(await vault.balanceOf(alice.address)).to.equal(0n)

    await time.increase(PERIOD * 2)

    // 5. Recovery path 1 closed: claimWithdraw checks the flag at :794 before
    //    ever reading the share balance at :797.
    await expect(vault.connect(alice).claimWithdraw(withdrawPeriod))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawPeriod)

    // 6. Recovery path 2 closed: re-blocklisting and re-rescuing hits the
    //    identical guard at :864.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        alice.address,
        users[2].address,
        [withdrawPeriod]
      )
    )
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawPeriod)

    // 7. The rescued 400 is claimable by the beneficiary, as intended.
    //    The 600 is not claimable by anyone, ever.
    expect(await vault.pendingWithdrawAssets()).to.equal(U(1000))
    await vault.connect(beneficiary).claimWithdraw(withdrawPeriod)
    expect(await token_contract.balanceOf(beneficiary.address)).to.equal(U(10_000) + U(400))

    // 600 assets permanently stranded in the vault.
    expect(await vault.pendingWithdrawAssets()).to.equal(U(600))
    expect(await token_contract.balanceOf(vault.target)).to.equal(U(600))
  })
})
```

### What the PoC demonstrates

| Assertion                                                                         | Result    |
| --------------------------------------------------------------------------------- | --------- |
| Rescue latches `isWithdrawClaimed[N+1][alice] = true`                             | confirmed |
| `removeFromBlocklist` does not clear it                                           | confirmed |
| Post-clearance `withdraw(600)` burns Alice's shares and credits the poisoned slot | confirmed |
| `claimWithdraw(N+1)` reverts `AlreadyClaimedPeriod`                               | confirmed |
| Re-blocklist + re-rescue reverts `AlreadyClaimedPeriod`                           | confirmed |
| 600 assets remain in the vault, unclaimable by any party                          | confirmed |
