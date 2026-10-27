# 90104 sc high rescuing a blocklisted account s withdrawal permanently bricks that account s next withdrawal

**Submitted on Aug 23rd 2026 at 02:07:55 UTC by @koshak01 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90104
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` for an account that receives **no assets at all**. That flag is declared, and read everywhere else, as "this account has already been paid for this period". It is never cleared: no code path in scope writes `false` to it and no clearing function exists.

Consequently, once an account has had a pending withdrawal rescued, its **next legitimate withdrawal request into the same period bucket becomes permanently unclaimable**. The assets stay inside the vault, permanently counted in `pendingWithdrawAssets`, and are therefore permanently subtracted from `totalAssets()` — lost to the account and to every other depositor alike.

Every actor in the path acts strictly inside its documented powers, and the rescue itself is correct behaviour — moving a blocklisted account's queued withdrawal to a safe address is exactly what the function exists to do. The defect is not that the rescue happens. It is that the rescue leaves behind an irreversible mark which the vault reads as "this account was paid", breaking the invariant that an account which has not claimed can still claim. No role can restore that state and no additional role is needed to reach it.

## Root cause: one storage slot carrying two incompatible meanings

The slot is declared as a record of **payment received**:

```solidity
// FirelightVaultStorage.sol:71
/// @notice Indicates whether an account has claimed their withdrawal for a given period.
mapping(uint256 period => mapping(address account => bool value)) public isWithdrawClaimed;
```

and `claimWithdraw` documents its revert on exactly that basis:

```solidity
// FirelightVault.sol:782-784
 * Transfers the corresponding assets to the caller if not already claimed.
 * Can only be called after the specified period has ended.
 * Reverts if the withdrawal has already been claimed or if no withdrawal amount is available for the period.
```

Across the whole scope the flag is written in exactly two places:

```solidity
// FirelightVault.sol:806-808  — a real claim: flag set, then assets transferred
isWithdrawClaimed[period][sender] = true;

IERC20(asset()).safeTransfer(sender, assets);
```

```solidity
// FirelightVault.sol:868-870  — the rescue: shares moved to `to`, NOTHING transferred to `from`
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

and it is read as a hard stop in two places:

```solidity
// FirelightVault.sol:794
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

```solidity
// FirelightVault.sol:864
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

`grep -rn "isWithdrawClaimed" contracts/core/` returns exactly seven sites: the declaration (`FirelightVaultStorage.sol:71`), the interface getter (`interfaces/IFirelightVault.sol:171`), the two reads that revert (`:794`, `:864`), the read guarding the receiver (`:865`), and the two writes (`:806`, `:870`). There is no `= false`, no `delete`, and no function that clears it anywhere. Zeroing `withdrawSharesOf[periods[i]][from]` on `:869` only prevents the *same* balance from being rescued twice; it does nothing for shares credited to that account afterwards.

The immediately adjacent line shows the authors were aware of this precise failure mode and guarded only the other side of the transfer:

```solidity
// FirelightVault.sol:865  — the receiver is protected from inheriting an unclaimable balance
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
```

The mirror case — the sender left with a permanent, false "already paid" mark — is not guarded anywhere.

## Vulnerability path

Every step below is a documented operation performed correctly by the actor entitled to it.

1. A user requests a withdrawal. `_requestWithdraw` credits bucket `currentPeriod() + 1` (`FirelightVault.sol:1008`, `:1017`) and adds the assets to `pendingWithdrawAssets` (`:1019`).
2. `BLOCKLIST_ROLE` places the account on hold — `addToBlocklist`, `FirelightVault.sol:550`. A compliance hold is the stated purpose of the blocklist.
3. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted` — `FirelightVault.sol:845`, documented at `:840` as "Rescues pending withdrawals from a blocklisted address". The queued shares move to a safe address; **the blocklisted account receives nothing**; `:870` marks it as having claimed.
4. The hold is lifted — `removeFromBlocklist`, `FirelightVault.sol:560`. This is a separate, deliberately provided function of the same role, used whenever a hold was precautionary or mistaken.
5. The account still holds its vault shares — the rescue moves queued withdrawal shares, not the ERC20 balance; moving the balance is a different function, `rescueSharesFromBlocklisted` at `:818` — and it makes an ordinary new withdrawal request. It lands in the same bucket, and `withdrawalsOf(bucket, account)` now reports a positive amount owed. This is the single condition the path needs: the new request must fall in the same period as the rescue, because a request always targets `currentPeriod() + 1`. The window is therefore the remainder of the current period, which on the deployed vault this upgrade targets is **one day**: its single period configuration carries `duration = 86400` (read live at Flare block 68,000,000 from `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`, currently in period 263). A user whose hold has just been lifted and who goes to withdraw walks into that window by the shortest path, especially since the vault itself reports the amount as owed.
6. When the bucket matures, `claimWithdraw` reverts `AlreadyClaimedPeriod` (`:794`), and a second rescue reverts on the same flag (`:864`). No further path exists.

### The loss is committed at request time, and the vault says nothing is wrong

The flag is readable — `isWithdrawClaimed` has a public getter (`IFirelightVault.sol:171`) — so it could be argued the user should have checked it before step 5. Three things make that argument fail.

First, nothing else in the contract reflects the flag. `maxWithdraw` (`:401-407`) and `maxRedeem` (`:415-421`) gate only on blocklist and pause, so the ERC4626 surface reports the full balance as withdrawable. `withdrawalsOf` (`:466-473`) reads `withdrawSharesOf` alone and reports the amount as owed. The one accessor that would reveal the problem is the one nothing points at, while three standard accessors say the funds are available.

Second, the request itself succeeds. `_requestWithdraw` performs no flag check and burns the shares immediately — `_update(owner, address(0), shares)` (`:1025`).

Third, there is no way back: the contract has no function to cancel a withdrawal request (`grep -n "cancel" contracts/core/FirelightVault.sol` returns nothing). The moment `withdraw` returns successfully the shares are gone and the assets are committed to a bucket that can never be claimed. The failure only becomes visible one period later, at `claimWithdraw`.

So the user cannot mitigate by "requesting in a later period" — by the time the defect is observable, the shares that would have funded that later request no longer exist.

## Impact

The assets of step 5 can never leave the vault:

* `claimWithdraw` — reverts on `:794`.
* `rescueWithdrawFromBlocklisted` — reverts on `:864`, including after re-blocklisting.
* The only other outgoing asset transfer in the contract is `payout` (`:962`), restricted to `PAYOUT_ROLE` and to an incident payout window; it pays a configured receiver and never releases a specific account's queued shares.

Because `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets` (`:428`), the stuck amount is permanently excluded from the vault's active assets. It is lost to its owner, and it permanently depresses `totalAssets()` for every other depositor, for the underwriting capacity computed from it, and for any future incident payout.

This is a permanent freezing of user funds produced by the contract itself: no role exceeds its powers, no role makes an incorrect decision, and no key is compromised. What is violated is a system invariant the contract states in three places — `isWithdrawClaimed` records that the account **has claimed** its withdrawal (`FirelightVaultStorage.sol:71`), and a claim is refused only when the withdrawal "has already been claimed" (`FirelightVault.sol:784`, repeated in `IFirelightVault.sol:230`). Here the claim is refused on precisely that ground when nothing was ever claimed. No additional role is required for the outcome: the rescuer cannot undo it, and the victim is an ordinary depositor holding no role at all.

### The deployment this lands on

`FirelightVault.sol` is the V2 implementation upgrade for the legacy pre-deposit vault deployed on Flare at `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`. Read live at block 68,000,000, that vault holds **56,613,733.94 FXRP**, with **393,423.29 FXRP** already sitting in the withdrawal queue, against a 60,168,000 FXRP deposit cap.

Its storage layout is compatible with V2 slot for slot — the two new mappings consume two words of the reserved gap and `__gap` drops from 50 to 48 — so every `isWithdrawClaimed` entry and every queued withdrawal carries into V2 unchanged. The defect therefore applies to a live vault and its existing withdrawal queue from the moment the upgrade executes, not to a fresh deployment.

## Recommendation

Stop overloading one slot with two meanings. The rescue is not a payment, so it must not write the payment flag for the sender. Either:

* drop `isWithdrawClaimed[periods[i]][from] = true` on `:870` — zeroing `withdrawSharesOf[periods[i]][from]` on `:869` already prevents the same balance from being rescued twice, and `:866` already rejects a zero balance; or
* keep a separate `isWithdrawRescued[period][account]` marker for the rescue and leave `isWithdrawClaimed` meaning strictly what its NatSpec says.

Either change keeps the rescue idempotent while leaving the account able to claim withdrawals it requests afterwards.

## Link to Proof of Concept

<https://gist.github.com/koshak01/0a1329fa6fd2fe4b77cd09d82745569d>

## Proof of Concept

Runnable against the audit snapshot with the project's own fixtures, unmodified. Save the file below as `test/FL-02-poc.js` and run:

```
npm install
npx hardhat test test/FL-02-poc.js
```

```javascript
// PoC: rescueWithdrawFromBlocklisted marks the sender as "already claimed" without
// paying them anything, and the flag can never be cleared. The account's next
// legitimate withdrawal request into the same period bucket becomes permanently
// unclaimable, and the assets stay locked in the vault forever.
//
// Overloaded storage slot:
//   FirelightVaultStorage.sol:71  "Indicates whether an account has claimed their withdrawal for a given period"
//   FirelightVault.sol:806        set after a real payout (assets transferred on :808)
//   FirelightVault.sol:870        set by the rescue, with NO transfer to `from`
//   FirelightVault.sol:794        claimWithdraw reverts on it
//   FirelightVault.sol:864        rescueWithdrawFromBlocklisted reverts on it
// The flag is never set to false anywhere in scope; no clearing function exists.
//
// The mirror guard on :865 shows the authors were aware of this exact class for the
// receiver side and left the sender side unguarded.
//
// Run:
//   cd <repo> && npx hardhat test test/FL-02-poc.js

const { expect } = require('chai')
const { ethers } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures')

const U = (n) => BigInt(n) * 10n ** 6n            // vault asset has 6 decimals
const fmt = (v) => (Number(v) / 1e6).toFixed(6)
const PERIOD = 604800n                             // 1 week, the fixture default

describe('FL-02: rescue permanently bricks the account\'s next withdrawal', () => {
  it('locks user assets forever with every role acting inside its documented powers', async () => {
    const d = await deployVault()                  // project fixture, unmodified
    const { firelight_vault: vault, token_contract: token, utils, users, blocklister, rescuer } = d
    const [alice] = users
    const safeAddr = ethers.Wallet.createRandom().address
    const safeAddr2 = ethers.Wallet.createRandom().address
    const vaultAddr = await vault.getAddress()

    await utils.mintAndApprove(U(1000), alice)
    await vault.connect(alice).deposit(U(1000), alice.address)

    const N = await vault.currentPeriod()
    const bucket = N + 1n                           // _requestWithdraw always targets currentPeriod()+1
    console.log(`period ${N}: alice deposited 1000; withdrawal requests land in bucket ${bucket}`)

    // STEP 1 - ordinary withdrawal request by the user
    await vault.connect(alice).withdraw(U(200), alice.address, alice.address)
    const firstShares = await vault.withdrawSharesOf(bucket, alice.address)
    console.log(`request #1: withdrawSharesOf(${bucket}, alice) = ${firstShares}, pendingWithdrawAssets = ${fmt(await vault.pendingWithdrawAssets())}`)

    // STEP 2 - BLOCKLIST_ROLE puts the account on hold. Documented power, FirelightVault.sol:550
    await vault.connect(blocklister).addToBlocklist(alice.address)

    // STEP 3 - RESCUER_ROLE rescues the stuck request. Exactly what the function is for,
    //          FirelightVault.sol:840 "Rescues pending withdrawals from a blocklisted address"
    const aliceAssetsBefore = await token.balanceOf(alice.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, safeAddr, [bucket])
    const aliceAssetsAfter = await token.balanceOf(alice.address)

    expect(await vault.withdrawSharesOf(bucket, alice.address)).to.equal(0n)
    expect(await vault.withdrawSharesOf(bucket, safeAddr)).to.equal(firstShares)
    expect(aliceAssetsAfter, 'alice received nothing').to.equal(aliceAssetsBefore)
    expect(await vault.isWithdrawClaimed(bucket, alice.address), 'yet she is flagged as having claimed').to.equal(true)
    console.log(`rescue: shares moved to the safe address, alice received 0 tokens,`)
    console.log(`        but isWithdrawClaimed(${bucket}, alice) == true  <-- flag set for a payout that never happened`)

    // STEP 4 - the hold turns out to be a mistake, BLOCKLIST_ROLE lifts it. FirelightVault.sol:560
    await vault.connect(blocklister).removeFromBlocklist(alice.address)
    console.log('blocklist lifted (same role, separate documented function :560)')

    // STEP 5 - alice still holds vault shares (the rescue moved queued shares, not her balance)
    //          and makes a NEW, perfectly ordinary withdrawal request into the same bucket.
    const pendingBefore = await vault.pendingWithdrawAssets()
    await vault.connect(alice).withdraw(U(300), alice.address, alice.address)
    const secondShares = await vault.withdrawSharesOf(bucket, alice.address)
    const lockedAssets = (await vault.pendingWithdrawAssets()) - pendingBefore
    expect(secondShares).to.be.greaterThan(0n)
    console.log(`request #2: withdrawSharesOf(${bucket}, alice) = ${secondShares}, pendingWithdrawAssets grew by ${fmt(lockedAssets)}`)

    // the vault itself tells her she is owed the money
    const owed = await vault.withdrawalsOf(bucket, alice.address)
    expect(owed).to.be.greaterThan(0n)
    console.log(`vault reports withdrawalsOf(${bucket}, alice) = ${fmt(owed)}  <-- the vault says she is owed this`)

    // STEP 6 - the bucket matures. Claiming is now permanently impossible.
    await time.increase(PERIOD * 2n)
    console.log(`now in period ${await vault.currentPeriod()}; bucket ${bucket} has been claimable for a full period`)

    await expect(vault.connect(alice).claimWithdraw(bucket))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(bucket)
    console.log(`claimWithdraw(${bucket}) -> reverts AlreadyClaimedPeriod, although no payout was ever received`)

    // STEP 7 - the rescue path cannot recover it either: same flag, FirelightVault.sol:864
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, safeAddr2, [bucket]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(bucket)
    console.log('rescue retry -> reverts on the same flag (:864). No exit remains.')

    // STEP 8 - quantify the permanent loss
    const bal = await token.balanceOf(vaultAddr)
    const pending = await vault.pendingWithdrawAssets()
    const total = await vault.totalAssets()
    expect(total + pending).to.equal(bal)
    expect(await vault.withdrawSharesOf(bucket, alice.address)).to.equal(secondShares)
    expect(await vault.isWithdrawClaimed(bucket, alice.address)).to.equal(true)

    console.log(`vault token balance ${fmt(bal)} = totalAssets ${fmt(total)} + pendingWithdrawAssets ${fmt(pending)}`)
    console.log(`${fmt(lockedAssets)} of that queue is unreachable by ANY function:`)
    console.log('  - claimWithdraw  reverts (:794)')
    console.log('  - rescueWithdrawFromBlocklisted reverts (:864)')
    console.log('  - the only other asset transfer out is payout() (:962), PAYOUT_ROLE only, and it never')
    console.log('    releases a specific account\'s queued shares')
    console.log('  - totalAssets() = balance - pendingWithdrawAssets (:428), so the amount is permanently')
    console.log('    subtracted from every other depositor\'s share value as well')
  })
})
```

### Output

```
  FL-02: rescue permanently bricks the account's next withdrawal
period 0: alice deposited 1000; withdrawal requests land in bucket 1
request #1: withdrawSharesOf(1, alice) = 200000000, pendingWithdrawAssets = 200.000000
rescue: shares moved to the safe address, alice received 0 tokens,
        but isWithdrawClaimed(1, alice) == true  <-- flag set for a payout that never happened
blocklist lifted (same role, separate documented function :560)
request #2: withdrawSharesOf(1, alice) = 300000000, pendingWithdrawAssets grew by 300.000000
vault reports withdrawalsOf(1, alice) = 300.000000  <-- the vault says she is owed this
now in period 2; bucket 1 has been claimable for a full period
claimWithdraw(1) -> reverts AlreadyClaimedPeriod, although no payout was ever received
rescue retry -> reverts on the same flag (:864). No exit remains.
vault token balance 1000.000000 = totalAssets 500.000000 + pendingWithdrawAssets 500.000000
300.000000 of that queue is unreachable by ANY function:
  - claimWithdraw  reverts (:794)
  - rescueWithdrawFromBlocklisted reverts (:864)
  - the only other asset transfer out is payout() (:962), PAYOUT_ROLE only, and it never
    releases a specific account's queued shares
  - totalAssets() = balance - pendingWithdrawAssets (:428), so the amount is permanently
    subtracted from every other depositor's share value as well
    ✔ locks user assets forever with every role acting inside its documented powers (817ms)


  1 passing (819ms)
```
