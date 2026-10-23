# 89051 sc high rescue marker collision in firelightvault permanently strands a cleared account s later withdrawal

**Submitted on Aug 16th 2026 at 19:16:53 UTC by @Dead\_Aizen for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89051
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Summary

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` as an "already rescued" marker. That is the same flag `claimWithdraw` reads to prevent double claims. The two meanings are conflated on one storage slot.

The flag is a one-way latch. It is written in exactly two places in the codebase and both write `true`; nothing writes `false`. The ledger it guards keeps accepting writes: `_requestWithdraw` credits `withdrawSharesOf[period][receiver]` unconditionally, without consulting the flag.

An account rescued for period `p` and subsequently un-blocklisted can therefore request another withdrawal that also lands in period `p`. The shares are burned, the assets enter `pendingWithdrawAssets`, `withdrawalsOf(p, account)` reports the full amount owed — and `claimWithdraw(p)` reverts `AlreadyClaimedPeriod`.

The assets are not transferred to another party. They are unreachable by every party: not the account, not the other participants in the same withdrawal bucket, and not the remaining share holders.

The rescue does not need this flag. Line 869 already zeroes `withdrawSharesOf[periods[i]][from]`, which alone prevents the rescued balance from being claimed or rescued twice. Line 870 adds no protection and is the sole cause.

**Severity / impact:** High — Temporary freezing of funds for at least 24 hours\
**Commit:** `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (`v1_audit_ready`)\
**File:** `contracts/core/FirelightVault.sol`

The freeze is demonstrated below to be permanent, with every recovery path enumerated and shown to revert. This impact is selected conservatively as the lowest category the evidence supports.

## Root cause

`FirelightVault.sol:868-870`, inside `rescueWithdrawFromBlocklisted`:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;   // <-- written as "already rescued"
```

`FirelightVault.sol:794`, inside `claimWithdraw`, reads the same slot with the other meaning:

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

`FirelightVault.sol:1008` and `1017`, inside `_requestWithdraw`, credit the guarded ledger without consulting it:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
```

The complete reference set for the flag across the in-scope tree:

```
794   read    claimWithdraw guard
806   write   true    claimWithdraw
864   read    rescue guard, `from`
865   read    rescue guard, `to`
870   write   true    rescue
```

There is no write of `false` anywhere in the codebase. Two writers, both setting; no clearing path.

## Invariants broken

**Shares burned for a withdrawal must remain claimable by someone.** `_requestWithdraw` burns via `_update(owner, address(0), shares)` and moves the assets into `pendingWithdrawAssets`, from which `claimWithdraw` is the only exit. After the collision those assets are claimable by nobody.

**Un-blocklisting restores normal use.** The repository's own test suite states this expectation:

```
✔ removes a user from the blocklist; emits RemovedFromBlocklist
✔ allows user to transfer again after removing from blocklist
```

Restoration is incomplete. A cleared account remains permanently unable to claim for the colliding period, and permanently unable to receive a rescue for it.

**The view surface and the state machine disagree.** `withdrawalsOf(p, account)` returns the full amount owed for a claim the contract will always reject, so off-chain accounting and user-facing balances report a debt that will never be honoured.

## Sequence

Every step is a legitimate action by the role entitled to perform it. No role acts maliciously or incorrectly, and none exceeds its authority.

{% stepper %}
{% step %}

### Period `p-1` — Request a withdrawal

The user requests a withdrawal. `_requestWithdraw` assigns it to bucket `p = currentPeriod() + 1`.
{% endstep %}

{% step %}

### Period `p-1` — Blocklist the account

`BLOCKLIST_ROLE` blocklists the account. A screening false positive is the ordinary case.
{% endstep %}

{% step %}

### Period `p-1` — Rescue the pending withdrawal

`RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(user, safeHaven, [p])`. This is the function's documented purpose. It sets `isWithdrawClaimed[p][user] = true`.
{% endstep %}

{% step %}

### Period `p-1` — Clear the account

The account is cleared and `BLOCKLIST_ROLE` un-blocklists it.
{% endstep %}

{% step %}

### Period `p-1` — Request another withdrawal

The user requests another withdrawal from shares they still hold. It lands in bucket `p` again. Shares burned, assets moved to `pendingWithdrawAssets`, `withdrawSharesOf[p][user]` credited.
{% endstep %}

{% step %}

### Period `p+1` onward — Attempt to claim

`claimWithdraw(p)` reverts `AlreadyClaimedPeriod`.
{% endstep %}
{% endstepper %}

Steps 1 to 5 occur inside period `p-1`, a window of one full period. `SMALLEST_PERIOD_DURATION` is one day; the repository fixtures use seven.

No re-deposit is required. Because the first request was partial, the user simply still holds shares.

Per the Macro audit's on-chain verification, `BLOCKLIST_ROLE` is held by a 1-of-1 Safe (`0x189993f3B53284c7e5C02b058EFC6618F78D0cEA`) and `RESCUER_ROLE` by a separate 1-of-1 Safe (`0xC115757266b902d4b1284eA24DBbE557C15E963E`). The sequence requires no multisig quorum.

## Proof of concept

A single self-contained file, `test/integration/firelight-rescue-flag-poc.js`. It depends only on the repository's own helper (`test/setup/fixtures.js`) and needs no other new file. No mock contract was authored for this report.

Contract provenance is asserted by the first test case rather than claimed in prose. It reads the proxy's ERC-1967 implementation address, fetches the on-chain runtime bytecode, and asserts byte-for-byte equality with `artifacts/contracts/core/FirelightVault.sol/FirelightVault.json`. It also asserts the vault asset is the project's own FAsset (`fXRP`), the asset type the live V1 vault holds, and that all twelve functions the PoC calls are declared on the in-scope artifact. A mocked, modified or substituted contract fails that case and the run stops.

Executing in the defective path: `FirelightVault` (in scope, unmodified) and `FAsset` / `FAssetProxy` from `contracts/fasset/`.

One mock is instantiated by the fixture: `contracts/test/MockContract.sol`, which the repository's own `deployFAsset` helper installs as the FAsset `assetManager` so the test can mint `fXRP`. The vault never calls it, it holds no protocol state, and it appears in no code path in this finding.

```bash
npx hardhat test test/integration/firelight-rescue-flag-poc.js
```

```
  Rescue marker collision permanently strands a cleared account's later withdrawal
        implementation ................... 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
        runtime bytecode ................. 48658 chars, exact match to the
                                           contracts/core/FirelightVault.sol artifact
        asset() .......................... fXRP (contracts/fasset FAsset)
        every function called below is declared on that artifact
    ✔ runs against the unmodified in-scope FirelightVault
        P-1  second withdrawal -> bucket 1
        P-1  shares burned ................... 500000000
        P-1  assets into pendingWithdrawAssets 500000000
        P+2  withdrawalsOf(p, victim) ........ 500000000   <-- reported as owed
        P+2  claimWithdraw(p) ................ reverted AlreadyClaimedPeriod
        P+2  victim token balance ............ unchanged
    ✔ burns the shares and leaves no path to claim them
        other      owed 1000000000  received 1000000000
        safeHaven  owed 500000000  received 500000000
        pendingWithdrawAssets still holds .... 500000000
        totalAssets() unchanged at ........... 0
    ✔ does not redistribute the stranded assets to any other party
        inbound rescue into victim ........... reverted AlreadyClaimedPeriod
        victim claim ......................... reverted AlreadyClaimedPeriod
        re-rescue to a fresh address ......... reverted AlreadyClaimedPeriod
        self-rescue back to the victim ....... reverted
        external functions scanned ........... 84, able to clear the flag: 0
    ✔ offers no recovery path to the user, the rescuer, or an admin
        second request -> bucket 2, not 1
        claimWithdraw(2) .................... succeeded, victim received 500000000
        the period collision is the sole cause

  5 passing
```

The fourth and fifth cases are written to disprove the finding rather than confirm it.

Asserted, in order: the proxy implementation's runtime bytecode is byte-identical to the compiled in-scope `FirelightVault` artifact, the vault asset is `fXRP`, and every function the PoC calls is declared on that artifact; `isWithdrawClaimed(p, victim)` is `true` and `withdrawSharesOf(p, victim)` is `0` immediately after the rescue; the second request burns the victim's entire remaining share balance to `0` and raises `pendingWithdrawAssets` by exactly `500000000`; `withdrawalsOf(p, victim)` equals that amount three periods later; `claimWithdraw(p)` reverts `AlreadyClaimedPeriod` and the victim's token balance is unchanged; both other participants in bucket `p` receive exactly their `withdrawalsOf` amounts, so none of the stranded funds reach them; `pendingWithdrawAssets` still holds the stranded amount after every other claim settles and `totalAssets()` is unchanged across them; an inbound rescue from a different blocklisted account into the victim for period `p`, the victim's own claim, a re-rescue to a fresh address, and a self-rescue all revert; all 84 external functions are enumerated from the ABI and none can reset the flag; and, as a control, the identical sequence with the second request landing in period `p+1` claims successfully.

## No mitigation exists

`isWithdrawClaimed` has no setter. The only two writes both set `true` (`FirelightVault.sol:806`, `FirelightVault.sol:870`).

The rescue path cannot repair its own damage. `rescueWithdrawFromBlocklisted` reverts on `isWithdrawClaimed[periods[i]][from]` at line 864, so the account cannot be rescued again, and on `isWithdrawClaimed[periods[i]][to]` at line 865, so it cannot receive an inbound rescue for that period either. The account is unusable in both directions for the affected period. The PoC exercises each path and asserts the revert.

Re-blocklisting and un-blocklisting have no effect on the flag.

The only remaining route is a proxy upgrade adding a reset function. Per the program's terms, *"Emergency actions may be used to fix or mitigate an issue, but they are not intended as an automatic reason to downgrade an otherwise valid report."*

## Impact

The affected account permanently loses the assets backing its second request for that period. The shares are already burned, so the position cannot be reconstructed.

The loss is bounded to that account and that period. Other periods behave normally, and other participants in the same bucket are unaffected, because the stranded shares remain counted in both `withdrawShares[p]` and `withdrawAssets[p]`, leaving their pro-rata unchanged.

The stranded assets permanently distort protocol accounting. They remain in `pendingWithdrawAssets` for the life of the vault, and since `totalAssets()` is `super.totalAssets() - pendingWithdrawAssets`, the vault's reported assets are permanently understated by that amount. The share price, the deposit-limit check, and every period-start snapshot consumed by `CoverOrderAllocator` all read from that understated figure.

The user receives no signal. `redeem` succeeds normally and `withdrawalsOf` continues to report the amount as owed. The failure surfaces only on the claim attempt, at least two periods later, by which point the shares are long burned.

The live V1 vault holds approximately 60.16 million FXRP of staker capital, and the audited V2 code is the intended upgrade for it.

## Remediation

Delete line 870. `withdrawSharesOf[periods[i]][from]` is already zeroed on line 869, which is sufficient to prevent the rescued balance from being claimed or rescued twice. The claimed flag adds no protection the zeroing does not already provide.

If an explicit record of the rescue is wanted, track it on its own slot:

```solidity
mapping(uint256 period => mapping(address account => bool)) public isWithdrawRescued;
```

and set that in `rescueWithdrawFromBlocklisted` instead of `isWithdrawClaimed`.

Either fix also removes the inbound-rescue restriction at line 865, which currently blocks an unrelated legitimate rescue into a previously rescued account.

## Proof of Concept

```javascript
// test/integration/firelight-rescue-flag-poc.js

const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const fs = require('fs')
const path = require('path')

const { deployVault } = require('../setup/fixtures.js')

const E6 = 10n ** 6n
const STAKE = 1000n * E6

const log = (m) => console.log('        ' + m)

const artifactOf = (rel, name) =>
  JSON.parse(fs.readFileSync(path.join(__dirname, '../../artifacts', rel, `${name}.json`)))

const advancePeriods = async (vault, n) => {
  const cfg = await vault.currentPeriodConfiguration()
  await ethers.provider.send('evm_increaseTime', [Number(cfg.duration) * n + 10])
  await ethers.provider.send('evm_mine', [])
}

// Drives the collision and returns the poisoned period plus the stranded amount.
// The first redeem is partial, so the victim still holds shares for the second one and
// no re-deposit step is needed.
async function reachCollision(ctx, { partial = true } = {}) {
  const { firelight_vault: v, rescuer, blocklister, users, utils } = ctx
  const [victim, safeHaven] = users

  await utils.mintAndApprove(STAKE, victim)
  await v.connect(victim).deposit(STAKE, victim.address)

  const p = (await v.currentPeriod()) + 1n

  const first = partial ? STAKE / 2n : STAKE
  await v.connect(victim).redeem(first, victim.address, victim.address)

  // Compliance flow: blocklist, rescue the pending withdrawal, then clear the account.
  await v.connect(blocklister).addToBlocklist(victim.address)
  await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safeHaven.address, [p])
  await v.connect(blocklister).removeFromBlocklist(victim.address)

  expect(await v.isWithdrawClaimed(p, victim.address)).to.equal(true)
  expect(await v.withdrawSharesOf(p, victim.address)).to.equal(0n)

  // Second request, from the shares still held. Lands in the same bucket p.
  const pendingBefore = await v.pendingWithdrawAssets()
  const remaining = await v.balanceOf(victim.address)
  await v.connect(victim).redeem(remaining, victim.address, victim.address)
  const stranded = (await v.pendingWithdrawAssets()) - pendingBefore

  return { p, stranded, victim, safeHaven }
}

describe('Rescue marker collision permanently strands a cleared account\'s later withdrawal', function () {
  it('runs against the unmodified in-scope FirelightVault', async () => {
    const ctx = await loadFixture(deployVault)
    const v = ctx.firelight_vault

    // The proxy implementation must be byte-identical to the compiled in-scope
    // artifact, so a mocked or modified contract cannot pass this test.
    const impl = await upgrades.erc1967.getImplementationAddress(await v.getAddress())
    const onchain = await ethers.provider.getCode(impl)
    const artifact = artifactOf('contracts/core/FirelightVault.sol', 'FirelightVault')

    expect(onchain).to.equal(artifact.deployedBytecode)
    log(`implementation ................... ${impl}`)
    log(`runtime bytecode ................. ${onchain.length} chars, exact match to the`)
    log(`                                   contracts/core/FirelightVault.sol artifact`)

    // Vault asset is the project's own FAsset (fXRP), the asset type the live V1 vault
    // holds, rather than a stand-in ERC20.
    expect(await v.asset()).to.equal(ctx.token_contract.target)
    expect(await ctx.token_contract.symbol()).to.equal('fXRP')
    log(`asset() .......................... ${await ctx.token_contract.symbol()} (contracts/fasset FAsset)`)

    const abi = new Set(artifact.abi.filter((f) => f.type === 'function').map((f) => f.name))
    for (const fn of [
      'deposit', 'redeem', 'claimWithdraw', 'withdrawalsOf', 'withdrawSharesOf',
      'isWithdrawClaimed', 'pendingWithdrawAssets', 'totalAssets', 'balanceOf',
      'addToBlocklist', 'removeFromBlocklist', 'rescueWithdrawFromBlocklisted',
    ]) {
      expect(abi.has(fn), `${fn} missing from in-scope ABI`).to.equal(true)
    }
    log(`every function called below is declared on that artifact`)
  })

  it('burns the shares and leaves no path to claim them', async () => {
    const ctx = await loadFixture(deployVault)
    const { firelight_vault: v, token_contract } = ctx

    const { p, stranded, victim } = await reachCollision(ctx)

    log(`P-1  second withdrawal -> bucket ${p}`)
    log(`P-1  shares burned ................... ${await v.withdrawSharesOf(p, victim.address)}`)
    log(`P-1  assets into pendingWithdrawAssets ${stranded}`)
    expect(await v.balanceOf(victim.address)).to.equal(0n)
    expect(stranded).to.be.gt(0n)

    await advancePeriods(v, 3)

    // ---- DEFECT: the vault reports the debt, then refuses to honour it ----
    const owed = await v.withdrawalsOf(p, victim.address)
    expect(owed).to.equal(stranded)
    log(`P+2  withdrawalsOf(p, victim) ........ ${owed}   <-- reported as owed`)

    const before = await token_contract.balanceOf(victim.address)
    await expect(v.connect(victim).claimWithdraw(p))
      .to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')
    expect(await token_contract.balanceOf(victim.address)).to.equal(before)
    log(`P+2  claimWithdraw(p) ................ reverted AlreadyClaimedPeriod`)
    log(`P+2  victim token balance ............ unchanged`)
  })

  it('does not redistribute the stranded assets to any other party', async () => {
    const ctx = await loadFixture(deployVault)
    const { firelight_vault: v, token_contract, users, utils } = ctx
    const [, safeHaven, other] = users

    // A second staker in the same bucket, so over-payment would be visible.
    await utils.mintAndApprove(STAKE, other)
    await v.connect(other).deposit(STAKE, other.address)

    const { p, stranded } = await reachCollision(ctx)
    await v.connect(other).redeem(STAKE, other.address, other.address)

    const totalAssetsAfter = await v.totalAssets()
    await advancePeriods(v, 3)

    for (const [name, who] of [['other', other], ['safeHaven', safeHaven]]) {
      const owed = await v.withdrawalsOf(p, who.address)
      const before = await token_contract.balanceOf(who.address)
      await v.connect(who).claimWithdraw(p)
      const got = (await token_contract.balanceOf(who.address)) - before
      log(`${name.padEnd(10)} owed ${owed}  received ${got}`)
      expect(got).to.equal(owed) // exactly pro-rata, no slice of the stranded funds
    }

    // Still counted in pendingWithdrawAssets, so excluded from totalAssets() for the
    // life of the vault. Remaining share holders never receive it either.
    expect(await v.pendingWithdrawAssets()).to.be.gte(stranded)
    expect(await v.totalAssets()).to.equal(totalAssetsAfter)
    log(`pendingWithdrawAssets still holds .... ${await v.pendingWithdrawAssets()}`)
    log(`totalAssets() unchanged at ........... ${await v.totalAssets()}`)
  })

  it('offers no recovery path to the user, the rescuer, or an admin', async () => {
    const ctx = await loadFixture(deployVault)
    const { firelight_vault: v, rescuer, blocklister, users, utils } = ctx
    const thirdParty = (await ethers.getSigners())[7]

    await utils.mintAndApprove(STAKE, thirdParty)
    await v.connect(thirdParty).deposit(STAKE, thirdParty.address)

    const { p, victim } = await reachCollision(ctx)
    const [, , other] = users

    // Line 865 also blocks the victim as a rescue destination for this period.
    await v.connect(thirdParty).redeem(STAKE, thirdParty.address, thirdParty.address)
    await v.connect(blocklister).addToBlocklist(thirdParty.address)
    await expect(v.connect(rescuer).rescueWithdrawFromBlocklisted(thirdParty.address, victim.address, [p]))
      .to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')
    await v.connect(blocklister).removeFromBlocklist(thirdParty.address)
    log('inbound rescue into victim ........... reverted AlreadyClaimedPeriod')

    await advancePeriods(v, 3)

    await expect(v.connect(victim).claimWithdraw(p))
      .to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')
    log('victim claim ......................... reverted AlreadyClaimedPeriod')

    await v.connect(blocklister).addToBlocklist(victim.address)
    await expect(v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, other.address, [p]))
      .to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')
    log('re-rescue to a fresh address ......... reverted AlreadyClaimedPeriod')

    await expect(v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, victim.address, [p]))
      .to.be.reverted
    log('self-rescue back to the victim ....... reverted')
    await v.connect(blocklister).removeFromBlocklist(victim.address)

    // No administrative reset exists on the external surface.
    const fns = v.interface.fragments.filter((f) => f.type === 'function').map((f) => f.name)
    const resetters = fns.filter((n) => /reset|clear|unclaim|setWithdrawClaimed|forceClaim/i.test(n))
    expect(resetters).to.have.lengthOf(0)
    log(`external functions scanned ........... ${fns.length}, able to clear the flag: ${resetters.length}`)
  })

  it('control: the identical sequence in a different period claims successfully', async () => {
    const ctx = await loadFixture(deployVault)
    const { firelight_vault: v, token_contract, rescuer, blocklister, users, utils } = ctx
    const [victim, safeHaven] = users

    await utils.mintAndApprove(STAKE, victim)
    await v.connect(victim).deposit(STAKE, victim.address)

    const p = (await v.currentPeriod()) + 1n
    await v.connect(victim).redeem(STAKE / 2n, victim.address, victim.address)

    await v.connect(blocklister).addToBlocklist(victim.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safeHaven.address, [p])
    await v.connect(blocklister).removeFromBlocklist(victim.address)

    // Only difference from the run above: one period passes first, so the second
    // request lands in p+1 rather than colliding with p.
    await advancePeriods(v, 1)
    const p2 = (await v.currentPeriod()) + 1n
    expect(p2).to.not.equal(p)
    await v.connect(victim).redeem(await v.balanceOf(victim.address), victim.address, victim.address)

    await advancePeriods(v, 3)
    const before = await token_contract.balanceOf(victim.address)
    await v.connect(victim).claimWithdraw(p2)
    const got = (await token_contract.balanceOf(victim.address)) - before

    expect(got).to.be.gt(0n)
    log(`second request -> bucket ${p2}, not ${p}`)
    log(`claimWithdraw(${p2}) .................... succeeded, victim received ${got}`)
    log('the period collision is the sole cause')
  })
})
```
