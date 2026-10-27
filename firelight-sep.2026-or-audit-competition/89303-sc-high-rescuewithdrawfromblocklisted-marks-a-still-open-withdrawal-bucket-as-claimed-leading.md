# 89303 sc high rescuewithdrawfromblocklisted marks a still open withdrawal bucket as claimed leading to permanent freezing of funds

**Submitted on Aug 18th 2026 at 14:15:32 UTC by @c0mpos3r for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89303
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted` marks a withdrawal bucket as claimed for the rescued address, but the bucket it marks is `currentPeriod() + 1`, which is still open for new requests. Nothing clears the mark. If the address is un-blocklisted and withdraws again before that period rolls, `redeem` burns its shares and writes a credit into the same bucket, which `claimWithdraw` then refuses permanently. The assets stay in the vault, owed to an address that can never claim them.

## Vulnerability Details

`isWithdrawClaimed[period][account]` carries two meanings: "already paid out", set by `claimWithdraw`, and "rescued away", set by the rescue. The second is written to a bucket that can still receive credits.

```solidity
// :870   rescue marks the bucket claimed for `from`
isWithdrawClaimed[periods[i]][from] = true;

// :1008  every new withdrawal targets currentPeriod() + 1 ...
uint256 period = currentPeriod() + 1;

// :1017  ... and is credited without checking the flag
withdrawSharesOf[period][receiver] += sharesWithdraw;

// :1025  the user's shares are burned regardless
_update(owner, address(0), shares);

// :794   but claiming checks the flag, and reverts forever
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

The rescue validates `periods[i]` only for a non-zero balance and an unset claim flag (`:864-866`). Nothing requires the period to be closed, and `currentPeriod() + 1` is the natural argument, since that is where a live pending withdrawal sits.

Sequence:

1. Alice redeems, booking into bucket `B = currentPeriod() + 1`.
2. Alice is blocklisted pending a compliance review.
3. `rescueWithdrawFromBlocklisted(Alice, Bob, [B])` sets `isWithdrawClaimed[B][Alice] = true`.
4. The review clears her and she is un-blocklisted, still inside the same period. The repo fixtures use a 1-week period (`test/setup/fixtures.js:9`).
5. Alice redeems again. Her shares are burned at `:1025` and the credit lands in `B`.
6. `claimWithdraw(B)` reverts `AlreadyClaimedPeriod` permanently.

## Scope

This report does not dispute either role's authority. Both privileged calls only write state, and neither destroys value. `RESCUER_ROLE` may already seize an account's entire share balance through `rescueSharesFromBlocklisted` (`:818`), so nothing here turns on a role doing more than it may.

The value is destroyed by an unprivileged call. `redeem` burns the user's own shares (`:1025`) in exchange for a credit written (`:1017`) to a `(period, account)` pair the contract has already marked settled (`:870`), and that credit is one `claimWithdraw` is hard-coded to refuse (`:794`). `_requestWithdraw` never reads the flag, so nothing reverts and nothing warns. The broken invariant is internal to the withdrawal state machine: `_requestWithdraw` and `claimWithdraw` disagree about whether an account may hold a balance in a bucket. The fix asks for those two functions to be made consistent, not for any role to be constrained.

## Impact Details

In the PoC, Alice deposits 6,000, has 2,000 rescued, and redeems 3,000. The 3,000 is burned from supply and unclaimable.

Nothing recovers it. A second rescue is impossible: the function reverts on the very flag it set (`:864`) before reaching any transfer, asserted in the PoC. `rescueSharesFromBlocklisted` (`:818`) moves vault shares, not withdrawal-bucket entries, so it cannot reach the stranded balance. The vault moves the underlying in only two places, `claimWithdraw` (`:808`) and the incident payout (`:962`); the first is the call that reverts, and the second pays an allowlisted incident receiver. Meanwhile `withdrawalsOf(B, Alice)` (`:466`) still reports the full amount as claimable, so the vault publicly states a debt it will never pay.

`pendingWithdrawAssets` never drains, so `totalAssets() = super.totalAssets() - pendingWithdrawAssets` (`:428`) is understated permanently. That flows into every reader of it: allocator capacity (`CoverOrderAllocator.sol:711`) and the incident-payout exposure cap (`FirelightVault.sol:911`) are biased down, while `maxDeposit` (`:371-377`) returns `depositLimit - totalAssets()` and so over-reports headroom by the frozen amount.

Stating the bound honestly: the freeze is scoped to the one rescued bucket. The address withdraws and claims normally from the next period onward. The frozen amount is whatever it books into that bucket before the period rolls, which with a 1-week period can be an entire position, and nothing in the code caps it.

## Recommended fix

Use a separate `isWithdrawRescued[period][account]` marker and leave `isWithdrawClaimed` meaning only "claimed and paid". This is the minimal change and leaves the rescue's behaviour otherwise identical.

Rejecting `periods[i] > currentPeriod()` also closes it, since `_requestWithdraw` always targets `currentPeriod() + 1` (`:1008`) and so a closed bucket can never be credited again. Nothing escapes while the rescuer waits for the period to roll, because a blocklisted account cannot claim (`:790`). I applied this guard locally and ran `test/rescue.js`: 22 of 23 pass, and the one failure is an expectation change rather than a functional regression, since `test/rescue.js:148` passes `[2,3,4]` and asserts `NoWithdrawalAmount(4)` where the guard now reverts `InvalidPeriod` first. Reverting on the same call for a different reason, so that assertion would need updating.

## References

* `contracts/core/FirelightVault.sol` — `:371-377`, `:428`, `:466`, `:790-794`, `:808`, `:818`, `:840`, `:845-876`, `:864-866`, `:870`, `:911`, `:962`, `:1008`, `:1017`, `:1025`
* `contracts/core/CoverOrderAllocator.sol:711`
* `test/rescue.js:148`, `test/setup/fixtures.js:9`
* Commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (`v1_audit_ready`)

## Proof of Concept

Runs against the unmodified repo, using the project's own fixtures. Local hardhat network only, no deployed contract touched.

```
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
# save the test at the end of this section as test/poc/v06-rescue-strand.js
npx hardhat test test/poc/v06-rescue-strand.js
```

Steps:

1. Alice deposits 6,000 (6-decimal asset), then redeems 2,000, credited to `B = currentPeriod() + 1`.
2. `addToBlocklist(alice)`.
3. `rescueWithdrawFromBlocklisted(alice, bob, [B])`. Now `isWithdrawClaimed(B, alice) == true` and `withdrawSharesOf(B, alice) == 0`, while `B` is still `currentPeriod() + 1`.
4. `removeFromBlocklist(alice)`. `currentPeriod()` is still `B - 1`, so her next request goes to the same bucket.
5. Alice redeems 3,000. Her shares are burned and the credit lands in `B` again, because `_requestWithdraw` never reads the flag.
6. Advance past `B`. `alice.claimWithdraw(B)` reverts `AlreadyClaimedPeriod`. Bob claims the same bucket successfully, which isolates the defect to the flag rather than to the bucket.

Output:

```
stranded shares          : 3000000000
pendingWithdrawAssets    : 5000000000 -> 3000000000
totalAssets before/after : 1000000000 -> 1000000000
```

Retrying `rescueWithdrawFromBlocklisted(alice, bob, [B])` after the fact reverts `AlreadyClaimedPeriod(B)` at the guard on line 864, so the rescue path cannot be used to undo its own effect.

`test/rescue.js` covers the rescue in isolation. No existing test credits a bucket after rescuing it. I reviewed the published reports from the prior Firelight competition and the 0xMacro report; neither raises the reuse of `isWithdrawClaimed` as a rescue marker, nor its interaction with a still-creditable bucket.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('PoC — rescue permanently strands a later withdrawal', function () {
  const D = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', D)
  const DEPOSIT = ethers.parseUnits('6000', D)
  const FIRST_REDEEM = ethers.parseUnits('2000', D)
  const SECOND_REDEEM = ethers.parseUnits('3000', D)

  it('strands Alice permanently and understates totalAssets forever', async () => {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))
    const vault = ctx.firelight_vault
    const { blocklister, rescuer, users, utils, config } = ctx
    const alice = users[0]
    const bob = users[1]

    await utils.mintAndApprove(DEPOSIT_LIMIT, alice)
    await vault.connect(alice).deposit(DEPOSIT, alice.address)

    const bucket = (await vault.currentPeriod()) + 1n

    // (1) Alice books a withdrawal into `bucket`.
    await vault.connect(alice).redeem(FIRST_REDEEM, alice.address, alice.address)
    expect(await vault.withdrawSharesOf(bucket, alice.address)).to.be.gt(0n)

    // (2) Alice is blocklisted, e.g. a compliance flag under investigation.
    await vault.connect(blocklister).addToBlocklist(alice.address)

    // (3) RESCUER moves her pending withdrawal to Bob. This marks the bucket claimed for Alice
    //     even though the bucket is still `currentPeriod()+1` and can still be credited.
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [bucket])
    expect(await vault.isWithdrawClaimed(bucket, alice.address)).to.eq(true)
    expect(await vault.withdrawSharesOf(bucket, alice.address)).to.eq(0n)

    // (4) Investigation clears Alice and she is un-blocklisted, still in the SAME period.
    await vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await vault.currentPeriod()).to.eq(bucket - 1n)

    // (5) Alice redeems again. `_requestWithdraw` does not consult isWithdrawClaimed,
    //     so the credit lands in the very bucket already flagged claimed for her.
    await vault.connect(alice).redeem(SECOND_REDEEM, alice.address, alice.address)
    const stranded = await vault.withdrawSharesOf(bucket, alice.address)
    expect(stranded).to.be.gt(0n)

    const pendingBefore = await vault.pendingWithdrawAssets()
    const totalAssetsBefore = await vault.totalAssets()

    // (6) Advance past the bucket so it becomes claimable.
    await time.increase(config.period_configuration_duration * 2)
    expect(await vault.currentPeriod()).to.be.gt(bucket)

    // (7) Alice can never claim. The flag set by the rescue blocks her permanently.
    await expect(vault.connect(alice).claimWithdraw(bucket))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')

    // Bob, the rescue destination, claims normally, proving the bucket itself is healthy.
    await vault.connect(bob).claimWithdraw(bucket)

    // (8) The rescue cannot undo its own effect: retrying reverts on the flag it set (:864).
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [bucket])
    ).to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
    await vault.connect(blocklister).removeFromBlocklist(alice.address)

    // (9) Alice's assets stay in pendingWithdrawAssets, so totalAssets() is permanently
    //     understated for every reader of it.
    const pendingAfter = await vault.pendingWithdrawAssets()
    expect(pendingAfter).to.be.gt(0n)
    await expect(vault.connect(alice).claimWithdraw(bucket))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')

    console.log('        stranded shares          :', stranded.toString())
    console.log('        pendingWithdrawAssets    :', pendingBefore.toString(), '->', pendingAfter.toString())
    console.log('        totalAssets before/after :', totalAssetsBefore.toString(), '->', (await vault.totalAssets()).toString())
  })
})
```
