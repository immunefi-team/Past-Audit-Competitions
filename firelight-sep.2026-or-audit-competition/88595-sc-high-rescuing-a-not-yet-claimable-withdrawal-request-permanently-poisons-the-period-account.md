# 88595 sc high rescuing a not yet claimable withdrawal request permanently poisons the period account pair freezing any later request on that period

**Submitted on Aug 14th 2026 at 02:02:31 UTC by @Kalila95146 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88595
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted` marks the source account as *already claimed* for the rescued period. It accepts any period, including one that is not yet claimable — which is the normal case, since withdrawal requests always target `currentPeriod() + 1`.

`_requestWithdraw` never reads that flag. `claimWithdraw` does, and reverts on it.

So after a rescue, if the account files a new withdrawal request that lands on the same period, the contract **accepts the request and burns the shares for a payout it can never honour**. The flag is never cleared anywhere in the contract, so the assets are frozen permanently.

## Vulnerability Details

`FirelightVault` uses delayed withdrawals: `withdraw`/`redeem` burn the caller's shares and create a pending request targeting `currentPeriod() + 1`, claimable later via `claimWithdraw(period)`.

### 1 — The rescue sets a permanent flag, on any period

`FirelightVault.sol:845-876`:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;   // :870 — no constraint on periods[i]
```

There is no `periods[i] < currentPeriod()` check. Rescuing a request for a period that has not yet become claimable therefore sets `isWithdrawClaimed[futurePeriod][from] = true`.

### 2 — A new request never consults that flag

`FirelightVault.sol:994-1028`:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;   // no isWithdrawClaimed check
```

### 3 — But the claim does

`FirelightVault.sol:788-811`:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // :794
```

`isWithdrawClaimed` is **never set back to `false`** anywhere in the contract (verified: the only writes are `:806` and `:870`, both to `true`). The pair `(period, account)` is poisoned permanently.

### Why this is a contract defect, not an operational choice

The most likely objection is *"the rescuer could simply have waited until the period became claimable"*. Four reasons that does not hold:

1. **The function explicitly accepts any period.** `rescueWithdrawFromBlocklisted` takes an arbitrary `periods[]` array with **no guard whatsoever** on its values. The contract permits the dangerous path and signals nothing.
2. **Waiting defeats the purpose of the rescue.** It would leave a blocklisted account's value in limbo for one to two full periods — the opposite of what a compliance rescue exists to do.
3. **Nothing communicates a timing constraint.** Not the signature, not the NatSpec, not any revert. The only way to know is to have already found this bug.
4. **A batch rescue cannot avoid it.** An account with requests spread over several periods is rescued in a single call (trigger condition #5), and such a call necessarily includes the not-yet-claimable period.

A rescuer following the obvious, documented behaviour produces the broken state.

### Exploit scenario

Every role action below is legitimate and squarely within its documented privileges — this is the normal compliance workflow, not misuse.

| # | Action                                                             | Resulting state                                                                  |
| - | ------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| 1 | Victim requests a withdrawal during period `C`                     | `withdrawSharesOf[C+1][victim] > 0`                                              |
| 2 | Victim is blocklisted (compliance suspicion)                       | —                                                                                |
| 3 | Rescuer calls `rescueWithdrawFromBlocklisted(victim, safe, [C+1])` | `withdrawSharesOf[C+1][victim] = 0`, **`isWithdrawClaimed[C+1][victim] = true`** |
| 4 | Suspicion cleared, victim unblocklisted (still period `C`)         | —                                                                                |
| 5 | Victim files a new withdrawal → targets `currentPeriod()+1 = C+1`  | `withdrawSharesOf[C+1][victim] > 0`, **shares burned**                           |
| 6 | Period advances, victim calls `claimWithdraw(C+1)`                 | **`revert AlreadyClaimedPeriod`**                                                |

Step 3 is not an odd operator choice: **a blocklisted user cannot claim** (`claimWithdraw` carries `notBlocklisted(_msgSender())`, reverting `BlocklistedAddress`), so rescuing is the *only* mechanism available to move that pending value. Verified in test R1.

**No recovery path exists:**

* `claimWithdraw` reverts forever — the flag is never cleared;
* a second rescue requires `isWithdrawClaimed[period][from] == false` → also reverts;
* the shares are already burned, so the amount cannot be re-requested.

### Exact trigger conditions

The bug fires **only** when a rescue targets a not-yet-claimable period **and** a later request lands on that same period. Both edges are asserted:

| # | Condition                                       | Result                                                                              |
| - | ----------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1 | Rescue a period that is **already claimable**   | **No bug** — a new request can only target `currentPeriod()+1`, never a past period |
| 2 | Re-request **one period later**                 | **No bug** — lands on a fresh, unpoisoned period                                    |
| 3 | Re-request via **`redeem()`**                   | **Reproduces identically** — same `_requestWithdraw`                                |
| 4 | **`rescueSharesFromBlocklisted`** (shares only) | **No poisoning** — never touches `isWithdrawClaimed`                                |
| 5 | Rescue passing **several periods in one call**  | **Every period is poisoned at once** — damage scales from one transaction           |

Rows 1, 2 and 4 are included to show the finding is precisely scoped rather than over-generalised. Row 5 matters for severity: cleaning up an account with requests spread over several periods turns each of them into a permanent trap.

### Recommended fix

Either option suffices:

1. In `_requestWithdraw`, revert if `isWithdrawClaimed[period][receiver]` is already `true`; **or**
2. In `rescueWithdrawFromBlocklisted`, set the claimed flag only for periods that are already claimable — for a not-yet-claimable period, zeroing `withdrawSharesOf` is enough, since the flag guards nothing once the balance is zero.

### Why this was not caught earlier

* `test/rescue.js` covers rescuing shares and rescuing withdrawals, but never files a new withdrawal request **after** a rescue on the same period — the exact trigger.
* `FirelightVault.sol` line coverage is **91.2%** (`npx hardhat coverage`); `_requestWithdraw`'s interaction with a pre-existing `isWithdrawClaimed` flag is among the uncovered behaviours.

This is an oversight, not an accepted design trade-off.

## Impact Details

**Severity: Critical — Permanent freezing of funds** (impact id 17). Fallback: High — Temporary freezing of funds ≥ 24h (id 6261).

1. **Permanent loss for the user.** Shares burned, assets unclaimable forever. The victim can lose their **entire remaining position** in a single call — test R2 redeems the full balance into the poisoned period and ends with `balanceOf == 0` and `maxWithdraw == 0`.
2. **Assets permanently stranded, owned by nobody.** `pendingWithdrawAssets` is incremented at step 5 and never decremented (only `claimWithdraw` decrements it, and it reverts). Since `totalAssets() = super.totalAssets() - pendingWithdrawAssets` (`:427-429`), the tokens physically remain in the vault while `totalAssets()` excludes them forever: not claimable by the victim, not redistributed to anyone else. Asserted: `asset.balanceOf(vault) - vault.totalAssets() >= frozenAmount`, and it does not heal over subsequent periods.
3. **Reduced cover capacity.** `CoverOrderAllocator._computeAvailableCapacity` (`CoverOrderAllocator.sol:710-714`) values the vault leg as `vault.totalAssetsAt(vault.currentPeriodStart())` — a checkpoint of `totalAssets() = super.totalAssets() - pendingWithdrawAssets`, which permanently excludes the stranded amount. The capacity computation therefore values the vault **below the collateral it actually holds**, for good.

   Asserted in test R5, on the exact quantity the allocator reads: the checkpoint returned by `totalAssetsAt(currentPeriodStart())` **equals** the live `totalAssets()` (so there is no temporal skew between the two readings), and `asset.balanceOf(vault) - checkpoint` **equals the frozen amount exactly** — still short five periods later.
4. **The withdrawal path gives no warning.** `maxWithdraw`, `maxRedeem`, `balanceOf`, `previewWithdraw` and `withdrawalsOf` all return healthy values, and `withdraw()` succeeds and emits `WithdrawRequest` normally. The raw `isWithdrawClaimed` mapping *is* a public getter and does return `true` — but **nothing on the withdrawal path consults or surfaces it**: the request is accepted and the shares burned regardless. A user following the documented ERC-4626 flow has no signal. This is why it is a protocol defect, not user error.

**Bounds of the claim (test C).** The victim is not locked out of withdrawing in general: a later request naming a **different** receiver claims normally. What is permanently frozen is the amount already committed to the doomed request — shares burned, assets unclaimable, no way back. And since nothing on the withdrawal path signals the poisoning (Impact 4), the victim has no reason to name a different receiver until after the loss has already occurred.

**Explicitly NOT claimed (verified, stated for accuracy):** the share price is **not** distorted for other depositors. When the doomed request was filed, the victim's shares were burned at the same moment `pendingWithdrawAssets` grew, so `totalAssets/totalSupply` is preserved — `previewDeposit` returns an identical quote before and after the freeze. Remaining holders are neither diluted nor enriched; the loss falls entirely on the victim.

## Scope justification

* **No role acts maliciously or incorrectly.** Blocklist → rescue → unblocklist is the *expected* operational sequence of a compliance system. The victim is a legitimate user.
* **The rescue is forced, not optional.** A blocklisted account cannot call `claimWithdraw` (reverts `BlocklistedAddress`), so its pending value can only be moved by `rescueWithdrawFromBlocklisted`.
* **Rescuing a not-yet-claimable period is the normal case.** Requests target `currentPeriod() + 1`, so a request stays non-claimable throughout the period in which it was filed and the next one. A rescuer reacting promptly necessarily lands in that window; nothing in the signature, NatSpec or revert conditions suggests waiting, and the function explicitly accepts any `periods[i]`.
* **It bypasses an intended constraint.** The implicit invariant *"an accepted withdrawal request remains claimable"* is broken even though `_requestWithdraw` accepted the request, burned the shares and took the value.

## References

* `contracts/core/FirelightVault.sol:845-876` — `rescueWithdrawFromBlocklisted`, the write at `:870`
* `contracts/core/FirelightVault.sol:994-1028` — `_requestWithdraw`, no read of the flag
* `contracts/core/FirelightVault.sol:788-811` — `claimWithdraw`, the revert at `:794`
* `contracts/core/FirelightVault.sol:427-429` — `totalAssets()`
* `contracts/core/CoverOrderAllocator.sol:710-714` — `_computeAvailableCapacity`

## Proof of Concept

## Proof of Concept

### Step-by-step reproduction

Every role acts strictly within its documented privileges. This is the normal compliance workflow of the protocol, not misuse.

1. **A user files a normal withdrawal request.** During period `C`, the victim calls `withdraw(amount, victim, victim)`. `_requestWithdraw` burns the shares and credits `withdrawSharesOf[C+1][victim]`, because a request always targets `currentPeriod() + 1`.
2. **Compliance blocklists the account.** The blocklister calls `addToBlocklist(victim)` on suspicion. From this moment the victim can no longer call `claimWithdraw` — it carries `notBlocklisted(_msgSender())` and reverts `BlocklistedAddress`. The pending value can now only be moved by a rescue.
3. **The rescuer moves the pending request to a safe address.** `rescueWithdrawFromBlocklisted(victim, safe, [C+1])` zeroes `withdrawSharesOf[C+1][victim]` and credits `safe`. It *also* sets `isWithdrawClaimed[C+1][victim] = true` (`FirelightVault.sol:870`), with no check that `C+1` is already claimable — and it is not, since we are still inside period `C`.
4. **The suspicion is cleared and the account is un-blocklisted.** `removeFromBlocklist(victim)`, still within period `C`.
5. **The victim files a new withdrawal request.** It targets `currentPeriod() + 1 = C+1` — the poisoned period. `_requestWithdraw` never reads `isWithdrawClaimed`, so the request is accepted, the shares are burned, `pendingWithdrawAssets` grows, and `WithdrawRequest` is emitted normally. `maxWithdraw`, `maxRedeem`, `balanceOf` and `previewWithdraw` all keep reporting healthy values.
6. **The period advances and the claim reverts, permanently.** `claimWithdraw(C+1)` hits `if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period)` (`:794`). There is no way out: the flag is never written back to `false` anywhere in the contract (the only two writes, `:806` and `:870`, both set `true`); a second rescue requires the flag to be `false` and therefore reverts too; and the shares are already burned, so the amount cannot be re-requested.

The assets stay physically in the vault while `totalAssets()` excludes them forever — claimable by nobody, redistributed to nobody.

### Running the tests

The suite is a **single self-contained file** written against the project's **own Hardhat fixtures** (`test/setup/fixtures.js`). No external harness, no mocks beyond what the repository already ships. Drop it into `test/` and run:

```bash
npx hardhat test test/FL02_full_suite.js
```

### Test file — `test/FL02_full_suite.js`

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * FL-02 — Permanent freezing of funds
 *
 * `rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` for a period
 * that is NOT yet claimable (rescuing a pending request is its intended use, and requests
 * always target `currentPeriod() + 1`).
 *
 * `_requestWithdraw` never reads that flag: it just does
 * `withdrawSharesOf[period][receiver] += sharesWithdraw` and burns the shares.
 *
 * `claimWithdraw` does read it, and reverts.
 *
 * => The flag permanently poisons the (period, address) pair. If the address is later
 *    un-blocklisted and files a new withdrawal request landing in that same period, the
 *    shares are burned and the assets can never be claimed. The flag is never reset to
 *    false anywhere in the contract, a second rescue requires it to be false, and no
 *    shares remain to re-request with.
 *
 * Every role acts strictly within its documented privileges: blocklisting a suspicious
 * account, rescuing its funds, then lifting the blocklist once cleared is the expected
 * operational sequence of a compliance system.
 */

describe('FL-02 — rescue poisons a future period, freezing a later withdrawal', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    WITHDRAW_AMOUNT = ethers.parseUnits('2000', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  async function setup() {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))
    const { firelight_vault, utils, users } = ctx
    // victim = users[0], rescue destination = users[1]
    await utils.mintAndApprove(DEPOSIT_LIMIT, users[0])
    await firelight_vault.connect(users[0]).deposit(DEPOSIT_AMOUNT, users[0].address)
    return ctx
  }

  it('POC: assets are permanently frozen after rescue + re-request in the same period', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0]
    const safeDestination = users[1]

    // --- 1. Victim files a normal withdrawal request. It targets currentPeriod() + 1.
    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    expect(await firelight_vault.withdrawSharesOf(period, victim.address)).to.be.gt(0)

    // --- 2. Compliance flags the account.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)

    // --- 3. Rescuer moves the pending request to a safe address.
    //        This is the intended use of the function: the period is NOT yet claimable.
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      victim.address,
      safeDestination.address,
      [period]
    )

    // The rescue moved the balance AND flagged the victim as "already claimed" for a
    // period that has not even started being claimable.
    expect(await firelight_vault.withdrawSharesOf(period, victim.address)).to.equal(0)
    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(true)

    // --- 4. The suspicion is cleared, the account is un-blocklisted. Still in the same period.
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // --- 5. Victim files a NEW withdrawal request. It lands in the same period.
    const pendingBefore = await firelight_vault.pendingWithdrawAssets()
    const sharesBefore = await firelight_vault.balanceOf(victim.address)

    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)

    expect(await firelight_vault.currentPeriod()).to.equal(period - 1n) // same period as step 1
    expect(await firelight_vault.withdrawSharesOf(period, victim.address)).to.be.gt(0)

    // The shares are really gone.
    const sharesAfter = await firelight_vault.balanceOf(victim.address)
    expect(sharesAfter).to.be.lt(sharesBefore)

    // --- 6. Move past the period so the withdrawal becomes claimable.
    await time.increase(config.period_configuration_duration * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(period)

    // --- 7. The claim reverts. Permanently.
    await expect(
      firelight_vault.connect(victim).claimWithdraw(period)
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    // No recovery path exists:
    //  - the flag is never reset to false anywhere in the contract;
    //  - a second rescue requires isWithdrawClaimed[period][from] == false, so it reverts too;
    //  - the shares were burned, so the amount cannot be re-requested.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
        victim.address,
        safeDestination.address,
        [period]
      )
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    // --- Collateral damage --------------------------------------------------------
    // pendingWithdrawAssets was incremented in step 5 and is only ever decremented by
    // claimWithdraw, which now reverts forever. The tokens stay in the vault while
    // totalAssets() excludes them permanently, so _computeAvailableCapacity (cover
    // capacity) is reduced for good. The share price itself is NOT distorted (the
    // shares were burned at the same moment) — see test R4.
    const pendingAfter = await firelight_vault.pendingWithdrawAssets()
    expect(pendingAfter).to.equal(pendingBefore + WITHDRAW_AMOUNT)
  })

  it('CONTROL: the very same flow without the rescue claims successfully', async () => {
    const { firelight_vault, users, config } = await setup()
    const victim = users[0]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)

    // Without the poisoned flag the claim goes through and the assets are delivered.
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.emit(firelight_vault, 'CompleteWithdraw')
      .withArgs(victim.address, WITHDRAW_AMOUNT, period)
  })
})

describe('FL-02 hardening — attempts to invalidate the finding', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    WITHDRAW_AMOUNT = ethers.parseUnits('2000', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  async function poisoned() {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))
    const { firelight_vault, utils, users, blocklister, rescuer } = ctx
    const victim = users[0]
    const safe = users[1]

    await utils.mintAndApprove(DEPOSIT_LIMIT, victim)
    await firelight_vault.connect(victim).deposit(DEPOSIT_AMOUNT, victim.address)

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    return { ...ctx, victim, safe, period }
  }

  it('A — the poisoned flag survives and the funds sit in limbo (nobody can get them)', async () => {
    const { firelight_vault, token_contract, victim, period, config } = await poisoned()

    const vaultBalBefore = await token_contract.balanceOf(firelight_vault.target)
    const totalAssetsBefore = await firelight_vault.totalAssets()

    // Victim re-requests: shares burned, request accepted silently.
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    // The assets never left the vault...
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(vaultBalBefore)
    // ...but totalAssets() excludes them forever, so they are not redistributed either.
    // Limbo: not claimable by the victim, not owned by the remaining depositors.
    expect(await firelight_vault.totalAssets()).to.be.lte(totalAssetsBefore - WITHDRAW_AMOUNT)
    expect(await firelight_vault.pendingWithdrawAssets()).to.be.gte(WITHDRAW_AMOUNT)
  })

  it('B — no admin/role path clears the flag (rescue, re-blocklist, pause cycle all fail)', async () => {
    const { firelight_vault, blocklister, rescuer, pauser, victim, safe, period, config } = await poisoned()

    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration * 2)

    // A second rescue cannot clear it.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // A pause/unpause cycle does not reset anything.
    await firelight_vault.connect(pauser).pause()
    await firelight_vault.connect(pauser).unpause()

    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(true)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
  })

  it('C — EXPOSURE: re-requesting to a DIFFERENT receiver does work (bounds the claim)', async () => {
    const { firelight_vault, victim, users, period, config } = await poisoned()
    const alt = users[2] // clean address, never poisoned

    // Same poisoned period, but the credit goes to a different receiver.
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, alt.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)

    // This one claims fine => the victim is not locked out of withdrawing in general.
    await expect(firelight_vault.connect(alt).claimWithdraw(period))
      .to.emit(firelight_vault, 'CompleteWithdraw')

    // But the victim themselves is still permanently barred for that period.
    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(true)
  })

  it('D — the withdrawal path gives no warning: the doomed request is accepted silently', async () => {
    const { firelight_vault, victim, period } = await poisoned()

    // maxWithdraw still advertises capacity, and the call succeeds without any revert
    // or event signalling that this request can never be claimed.
    expect(await firelight_vault.maxWithdraw(victim.address)).to.be.gt(0)

    await expect(firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address))
      .to.emit(firelight_vault, 'WithdrawRequest')

    // Shares are gone, and the flag that dooms the request was already set beforehand.
    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(true)
    expect(await firelight_vault.withdrawSharesOf(period, victim.address)).to.be.gt(0)
  })
})

describe('FL-02 boundaries — exact trigger conditions', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    WITHDRAW_AMOUNT = ethers.parseUnits('1000', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  async function setup() {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))
    await ctx.utils.mintAndApprove(DEPOSIT_LIMIT, ctx.users[0])
    await ctx.firelight_vault.connect(ctx.users[0]).deposit(DEPOSIT_AMOUNT, ctx.users[0].address)
    return ctx
  }

  it('1 — rescuing an ALREADY-CLAIMABLE period does NOT create the bug', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    const oldPeriod = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)

    // Let the period become claimable BEFORE rescuing.
    await time.increase(config.period_configuration_duration * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(oldPeriod)

    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [oldPeriod])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // A new request can no longer land on that past period, so no collision is possible.
    const newPeriod = (await firelight_vault.currentPeriod()) + 1n
    expect(newPeriod).to.be.gt(oldPeriod)
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(newPeriod))
      .to.emit(firelight_vault, 'CompleteWithdraw')
  })

  it('2 — re-requesting in a LATER period is safe (defines the trigger window)', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    const poisoned = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [poisoned])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Wait one period before re-requesting => lands on a fresh, unpoisoned period.
    await time.increase(config.period_configuration_duration)
    const fresh = (await firelight_vault.currentPeriod()) + 1n
    expect(fresh).to.be.gt(poisoned)

    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration * 2)

    await expect(firelight_vault.connect(victim).claimWithdraw(fresh))
      .to.emit(firelight_vault, 'CompleteWithdraw')
  })

  it('3 — the bug reproduces identically through redeem()', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Same period, but exit via redeem (shares) instead of withdraw (assets).
    const shares = await firelight_vault.balanceOf(victim.address)
    await firelight_vault.connect(victim).redeem(shares / 4n, victim.address, victim.address)

    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
  })

  it('4 — rescueSharesFromBlocklisted alone does NOT poison anything', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    // Rescue SHARES (not withdrawals): must not touch isWithdrawClaimed.
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueSharesFromBlocklisted(victim.address, safe.address)
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    const period = (await firelight_vault.currentPeriod()) + 1n
    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(false)

    // safe now holds the shares and can withdraw + claim normally.
    await firelight_vault.connect(safe).withdraw(WITHDRAW_AMOUNT, safe.address, safe.address)
    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(safe).claimWithdraw(period))
      .to.emit(firelight_vault, 'CompleteWithdraw')
  })

  it('5 — a multi-period rescue poisons every period passed in one call', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    // Two requests in two consecutive periods.
    const p1 = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration)
    const p2 = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    expect(p2).to.be.gt(p1)

    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [p1, p2])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Both periods are now poisoned => the damage scales with the number of rescued periods.
    expect(await firelight_vault.isWithdrawClaimed(p1, victim.address)).to.equal(true)
    expect(await firelight_vault.isWithdrawClaimed(p2, victim.address)).to.equal(true)

    // p2 is still the upcoming period here, so a re-request lands on it and is doomed.
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(p2))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
  })
})

describe('FL-02 refutations — closing every escape route', function () {
  const DECIMALS = 6,
    DEPOSIT_AMOUNT = ethers.parseUnits('6000', DECIMALS),
    WITHDRAW_AMOUNT = ethers.parseUnits('1000', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)

  async function setup() {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT }))
    await ctx.utils.mintAndApprove(DEPOSIT_LIMIT, ctx.users[0])
    await ctx.utils.mintAndApprove(DEPOSIT_LIMIT, ctx.users[2])
    await ctx.firelight_vault.connect(ctx.users[0]).deposit(DEPOSIT_AMOUNT, ctx.users[0].address)
    return ctx
  }

  it('R1 — "the rescuer had other options": no. A blocklisted user CANNOT claim, so rescuing is forced', async () => {
    const { firelight_vault, blocklister, users, config } = await setup()
    const victim = users[0]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)

    await time.increase(config.period_configuration_duration * 2)

    // While blocklisted the user cannot claim: the pending value is stranded unless rescued.
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'BlocklistedAddress')

    // => rescueWithdrawFromBlocklisted is the ONLY mechanism to move that value.
    // The scenario is therefore not an odd operator choice, it is the intended remedy.
  })

  it('R2 — "only part of the funds is lost": the victim can lose their ENTIRE remaining balance', async () => {
    const { firelight_vault, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Victim exits fully — the natural thing to do after a compliance scare.
    const allShares = await firelight_vault.balanceOf(victim.address)
    expect(allShares).to.be.gt(0)
    await firelight_vault.connect(victim).redeem(allShares, victim.address, victim.address)

    expect(await firelight_vault.balanceOf(victim.address)).to.equal(0) // nothing left

    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    // Zero shares, zero claimable assets: the victim's entire position is gone.
    expect(await firelight_vault.balanceOf(victim.address)).to.equal(0)
    expect(await firelight_vault.maxWithdraw(victim.address)).to.equal(0)
  })

  it('R3 — "the user could have seen it coming": no ERC-4626 surface warns them', async () => {
    const { firelight_vault, blocklister, rescuer, users } = await setup()
    const victim = users[0], safe = users[1]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Everything the standard ERC-4626 surface reports looks perfectly healthy.
    expect(await firelight_vault.maxWithdraw(victim.address)).to.be.gt(0)
    expect(await firelight_vault.maxRedeem(victim.address)).to.be.gt(0)
    expect(await firelight_vault.balanceOf(victim.address)).to.be.gt(0)
    expect(await firelight_vault.withdrawalsOf(period, victim.address)).to.equal(0)

    // previewWithdraw quotes a normal amount for a request that can never be claimed.
    expect(await firelight_vault.previewWithdraw(WITHDRAW_AMOUNT)).to.be.gt(0)

    // NOTE: the raw `isWithdrawClaimed` mapping IS a public getter and does return true
    // here. The point is not that the state is hidden, but that nothing on the
    // withdrawal path consults or surfaces it: `withdraw()` accepts the request and
    // burns the shares regardless.
    expect(await firelight_vault.isWithdrawClaimed(period, victim.address)).to.equal(true)
  })

  it('R5 — Impact 3 proven: the capacity input (totalAssetsAt) is permanently reduced', async () => {
    const { firelight_vault, token_contract, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1]

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Freeze it.
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    // Settle the legitimate side first, so the remaining gap isolates the FROZEN amount.
    await firelight_vault.connect(safe).claimWithdraw(period)

    // `_computeAvailableCapacity` (CoverOrderAllocator.sol:710-714) values the vault leg as
    // `vault.totalAssetsAt(vault.currentPeriodStart())` — a checkpoint of
    // `totalAssets() = super.totalAssets() - pendingWithdrawAssets`.
    const heldByVault = await token_contract.balanceOf(firelight_vault.target)
    const counted = await firelight_vault.totalAssets()
    const checkpoint = await firelight_vault.totalAssetsAt(await firelight_vault.currentPeriodStart())

    expect(await firelight_vault.pendingWithdrawAssets()).to.be.gte(WITHDRAW_AMOUNT)

    // No temporal skew between the readings: a legitimate claim decrements the token
    // balance and `pendingWithdrawAssets` by the same amount, so `totalAssets()` is
    // invariant under it and the checkpoint stays current. Asserted, not assumed.
    expect(checkpoint).to.equal(counted)

    // The gap is exactly the frozen amount: the capacity computation values the vault
    // one full WITHDRAW_AMOUNT below the collateral it actually holds.
    expect(heldByVault - checkpoint).to.equal(WITHDRAW_AMOUNT)

    // Still true several periods later — it never heals.
    await time.increase(config.period_configuration_duration * 5)
    const heldLater = await token_contract.balanceOf(firelight_vault.target)
    const countedLater = await firelight_vault.totalAssets()
    expect(heldLater - countedLater).to.be.gte(WITHDRAW_AMOUNT)
  })

  it('R4 — the stranded assets belong to NOBODY (share price is NOT affected)', async () => {
    const { firelight_vault, token_contract, blocklister, rescuer, users, config } = await setup()
    const victim = users[0], safe = users[1], newcomer = users[2]

    // Baseline share price quote on a healthy vault.
    const healthyQuote = await firelight_vault.previewDeposit(DEPOSIT_AMOUNT)

    const period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await firelight_vault.connect(blocklister).addToBlocklist(victim.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safe.address, [period])
    await firelight_vault.connect(blocklister).removeFromBlocklist(victim.address)

    // Trigger the freeze.
    await firelight_vault.connect(victim).withdraw(WITHDRAW_AMOUNT, victim.address, victim.address)
    await time.increase(config.period_configuration_duration * 2)
    await expect(firelight_vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')

    const pendingStuck = await firelight_vault.pendingWithdrawAssets()
    expect(pendingStuck).to.be.gte(WITHDRAW_AMOUNT)

    // IMPORTANT — the share price is NOT distorted: when the request was filed, the
    // victim's shares were burned at the same time pendingWithdrawAssets grew, so
    // totalAssets/totalSupply is preserved. Other depositors are neither diluted nor
    // enriched. Asserting this explicitly keeps the report honest.
    const poisonedQuote = await firelight_vault.previewDeposit(DEPOSIT_AMOUNT)
    expect(poisonedQuote).to.equal(healthyQuote)

    await firelight_vault.connect(newcomer).deposit(DEPOSIT_AMOUNT, newcomer.address)

    // What IS permanent: the tokens physically sit in the vault while totalAssets()
    // excludes them forever. They are owned by nobody — the victim cannot claim them
    // and they are not redistributed to remaining holders.
    const heldByVault = await token_contract.balanceOf(firelight_vault.target)
    const counted = await firelight_vault.totalAssets()
    expect(heldByVault - counted).to.be.gte(WITHDRAW_AMOUNT)

    // And it never heals: more time, more periods, same stranded value.
    await time.increase(config.period_configuration_duration * 5)
    expect(await firelight_vault.pendingWithdrawAssets()).to.be.gte(pendingStuck)
  })
})
```

### Output — 16 passing, 0 failing

```
FL-02 — rescue poisons a future period, freezing a later withdrawal
  ✔ POC: assets are permanently frozen after rescue + re-request in the same period
  ✔ CONTROL: the very same flow without the rescue claims successfully

FL-02 hardening — attempts to invalidate the finding
  ✔ A — the poisoned flag survives and the funds sit in limbo (nobody can get them)
  ✔ B — no admin/role path clears the flag (rescue, re-blocklist, pause cycle all fail)
  ✔ C — EXPOSURE: re-requesting to a DIFFERENT receiver does work (bounds the claim)
  ✔ D — the withdrawal path gives no warning: the doomed request is accepted silently

FL-02 boundaries — exact trigger conditions
  ✔ 1 — rescuing an ALREADY-CLAIMABLE period does NOT create the bug
  ✔ 2 — re-requesting in a LATER period is safe (defines the trigger window)
  ✔ 3 — the bug reproduces identically through redeem()
  ✔ 4 — rescueSharesFromBlocklisted alone does NOT poison anything
  ✔ 5 — a multi-period rescue poisons every period passed in one call

FL-02 refutations — closing every escape route
  ✔ R1 — "the rescuer had other options": no. A blocklisted user CANNOT claim, so rescuing is forced
  ✔ R2 — "only part of the funds is lost": the victim can lose their ENTIRE remaining balance
  ✔ R3 — "the user could have seen it coming": no ERC-4626 surface warns them
  ✔ R5 — Impact 3 proven: the capacity input (totalAssetsAt) is permanently reduced
  ✔ R4 — the stranded assets belong to NOBODY (share price is NOT affected)

16 passing
```

### What each test establishes

* **POC / CONTROL** — the freeze reproduces, and the identical flow *without* the rescue claims successfully. The defect is attributable to the rescue, not to the delayed-withdrawal mechanism.
* **A, B** — the funds are in limbo and no admin or role path clears the flag (second rescue, re-blocklist, pause/unpause cycle all fail).
* **C** — bounds the claim honestly: a later request naming a *different* receiver does claim. What is frozen is the amount already committed.
* **D** — the doomed request is accepted silently, with `WithdrawRequest` emitted normally.
* **1, 2, 4** — the negative controls. Rescuing an already-claimable period, re-requesting one period later, and `rescueSharesFromBlocklisted` alone all produce no bug. The finding is precisely scoped rather than over-generalised.
* **3** — reproduces identically through `redeem()`.
* **5** — a multi-period rescue poisons every period passed in one call, so the damage scales from a single transaction.
* **R1–R5** — each closes one anticipated objection, including R4 which states explicitly what is *not* claimed (the share price is unaffected).
