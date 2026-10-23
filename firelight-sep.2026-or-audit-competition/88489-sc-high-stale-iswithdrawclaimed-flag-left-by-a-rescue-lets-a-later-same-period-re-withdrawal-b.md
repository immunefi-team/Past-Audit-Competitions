# 88489 sc high stale iswithdrawclaimed flag left by a rescue lets a later same period re withdrawal burn a user s shares into a slot no code path can honor permanent freezing of funds

**Submitted on Aug 13th 2026 at 12:49:55 UTC by @Rubyglask for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88489
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Severity

High. Impact category: **Permanent freezing of funds** — an enumerated in-scope impact for this program (listed there at Critical). The freeze is permanent and affects principal (not merely unclaimed yield), and it is reachable with trusted operators acting only within their documented authority plus the affected user's own ordinary withdrawal. Assessed at High rather than Critical because the frozen amount is bounded to the affected user's own re-withdrawal for a single period, and the trigger requires a specific blocklist → rescue → un-blocklist sequence together with a same-period re-withdrawal — a constrained scope rather than a systemic freeze.

## Scope check

Not out of scope. This is a code-level accounting defect: the rescue path sets `isWithdrawClaimed[p][account]` and never clears it, and `_requestWithdraw` then burns shares into that flagged slot. It is not an operator-trust or governance issue — every operator action stays within its normal authority, and the loss is caused by the user's own ordinary withdrawal. That places it under the program's honest-operator provision (honest operator actions plus ordinary user actions and ordering).

## Summary

FirelightVault processes withdrawals in two steps. A request burns the owner's shares and credits a claim in the next period's withdrawal batch; after a delay, `claimWithdraw` pays it out. A per-account flag `isWithdrawClaimed[period][account]` marks a slot as collected so it cannot be claimed twice.

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal to another address and sets that same flag, and never clears it. If the account is later un-blocklisted and requests a new withdrawal into the same still-open batch, `_requestWithdraw` burns the shares and credits the slot without checking the flag. From that point both exit paths (`claimWithdraw` and a further rescue) revert on the stale flag. The re-withdrawn funds are paid in but can never be paid out, and the vault stops counting them in `totalAssets()`.

The defect is independent of operator intent: `_requestWithdraw` should never burn shares into a slot that every exit is guaranteed to reject, regardless of how the slot came to be flagged.

## Root cause

The rescue sets the flag and never resets it:

```solidity
// FirelightVault.sol:868-870 (rescueWithdrawFromBlocklisted)
withdrawSharesOf[periods[i]][to]   += _withdrawOf;
withdrawSharesOf[periods[i]][from]  = 0;
isWithdrawClaimed[periods[i]][from] = true;   // also set at :806; cleared nowhere in the contract
```

A fresh withdrawal credits the slot and burns the shares with no check on the flag:

```solidity
// FirelightVault.sol:1008-1025 (_requestWithdraw)
uint256 period = currentPeriod() + 1;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);            // shares burned; no isWithdrawClaimed check
```

Both exits revert on the flag: `claimWithdraw` at `:794` and re-rescue at `:864`, each `if (isWithdrawClaimed[...]) revert AlreadyClaimedPeriod(...)`.

## Only the rescue path can poison a live slot

`claimWithdraw` requires `period < currentPeriod()` (`:791`) while `_requestWithdraw` targets `currentPeriod() + 1` (`:1008`). Period numbers are monotonic and never reused, so the claim path can never have flagged a slot that a new withdrawal is landing in. `rescueWithdrawFromBlocklisted` accepts an arbitrary `periods[]` and can flag a still-pending period, so it is the only way to reach the collision. The flag has two writers (`:806`, `:870`) and no reset.

## Impact

The re-participating user permanently loses the re-withdrawn amount: shares burned, assets locked. The loss is bounded by that user's own re-withdrawn amount for the period; it is their own funds, not other users'. No party gains: the stranded tokens are excluded from `totalAssets()` (`= super.totalAssets() − pendingWithdrawAssets`, `:427-429`) and are unredeemable by anyone.

There is no recovery path in the deployed code — every exit for the poisoned slot is closed:

* `claimWithdraw` reverts on the flag (`:794`);
* `rescueWithdrawFromBlocklisted` reverts on the flag (`:864`);
* the vault's only other rescue, `rescueSharesFromBlocklisted` (`:818`), acts solely on the live share balance (`balanceOf` → `_transfer`) and never touches `withdrawSharesOf`, `isWithdrawClaimed`, or `pendingWithdrawAssets`, so it is structurally incapable of releasing the frozen withdrawal.

`isWithdrawClaimed` is written in exactly two places (`:806`, `:870`) and cleared nowhere in the contract, and no admin or governance function resets it or moves `pendingWithdrawAssets` back out of the slot. Upgrading the implementation could add such a path, but standard bounty practice does not treat upgradeability as a mitigation.

## Reachability and scope

The full sequence — withdraw, blocklist, rescue, un-blocklist, re-withdraw — must complete within one withdrawal period. This is structural: the flag is keyed `[period][account]`, and a new withdrawal re-enters the poisoned slot only while `currentPeriod() == p-1`. Each operator action is within its role and non-malicious; `addToBlocklist` and `removeFromBlocklist` share one `BLOCKLIST_ROLE`, and there is no guard preventing `removeFromBlocklist` after a rescue.

An operator un-blocklists an address after a protective rescue for reasons unrelated to this bug — for example, a mis-identification is corrected — and the loss is then caused by the user's own ordinary withdrawal. Nothing in the contract makes a rescue permanent or forbids un-blocklisting after one.

## Recommended fix

Guard `_requestWithdraw` so it can never burn shares into an already-claimed slot:

```solidity
// FirelightVault.sol — _requestWithdraw, before crediting the slot
uint256 period = currentPeriod() + 1;
if (isWithdrawClaimed[period][receiver]) revert AlreadyClaimedPeriod(period); // ← add
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

It now reverts instead of burning into a dead slot. Alternatives: clear the flag when crediting a fresh position, or give `rescueWithdrawFromBlocklisted` its own bookkeeping instead of overloading `isWithdrawClaimed`.

## Broken invariant

A non-blocklisted account that requested a withdrawal in period `p` can always claim it after `p`. The stale rescue flag breaks this.

## Notes

The finding is not among the program's listed known issues (which concern first-loss-buffer depeg and oracle behavior, unrelated to withdrawal share-accounting). The distinguishing element is the `isWithdrawClaimed` overload between the claim path (`:806`) and the rescue path (`:870`).

## Proof of Concept

`contracts/test/zz9_rescue_freeze.js` — real `FirelightVault`, real 6-decimal fXRP, fee 0. Hardhat. Run: `cd contracts && npx hardhat test test/zz9_rescue_freeze.js`.

Walkthrough: A deposits 30,000 and withdraws 10,000 into pool `p`. BLOCKLIST\_ROLE blocklists A; RESCUER moves A's pending withdrawal to B, setting `isWithdrawClaimed[p][A] = true`. BLOCKLIST\_ROLE un-blocklists A, still in period `q = p-1`. A re-withdraws 10,000 into pool `p`, which burns A's shares and credits `withdrawalsOf(p, A) = 10,000` while the flag stays true. Pool `p` matures: B claims A's original 10,000, but A's `claimWithdraw(p)` reverts `AlreadyClaimedPeriod` and a further rescue reverts too. The 10,000 is stranded in `pendingWithdrawAssets`, excluded from `totalAssets()`, unredeemable. The no-rescue control claims the full 20,000, isolating the flag as the sole cause.

Output (re-run 2026-08-13):

```
  [control] A withdrew 20000 across two requests, claimed=20000.0 (expect 20000)
    ✔ CONTROL: without rescue, an account that withdraws twice into pool p claims the full amount
  step1  A first withdraw -> pool 1: withdrawSharesOf[p][A]=10000000000 owedA=10000.0 pending=10000.0
  step2  rescue A->B: withdrawSharesOf[p][A]=0 withdrawSharesOf[p][B]=10000000000 isWithdrawClaimed[p][A]=true
  step4  A re-withdraw -> pool 1 (still period 0): withdrawSharesOf[p][A]=10000000000 owedA=10000.0 pending=20000.0
  step5  B claimed A's original: 10000.0 (expect 10000)
  step6  A.claimWithdraw(1) REVERTS AlreadyClaimedPeriod -> A's re-withdrawn 10000 is FROZEN
  step7  re-rescue A->C also REVERTS AlreadyClaimedPeriod -> no rescue path either
  step8  STRANDED: pendingWithdrawAssets=10000.0  vaultBalance=20000.0  totalAssets=10000.0
    ✔ FREEZE: rescue(A->B) then un-blocklist A then A re-withdraws into pool p -> A can NEVER claim
  2 passing (470ms)
```

Test (`contracts/test/zz9_rescue_freeze.js`):

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const DEC = 6
const U = (x) => ethers.parseUnits(String(x), DEC)

describe('rescue permanently poisons the (period,account) claim slot', function () {
  it('CONTROL: without rescue, an account that withdraws twice into pool p claims the full amount', async () => {
    const { firelight_vault: v, token_contract: tok, users, utils, config } =
      await loadFixture(deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(1_000_000) }))
    const [A] = users
    await utils.mintAndApprove(U(30_000), A)
    await v.connect(A).deposit(U(30_000), A.address)

    const p = (await v.currentPeriod()) + 1n
    await v.connect(A).withdraw(U(10_000), A.address, A.address) // 1st -> pool p
    await v.connect(A).withdraw(U(10_000), A.address, A.address) // 2nd -> SAME pool p

    await time.increase(config.period_configuration_duration * 2)
    const before = await tok.balanceOf(A.address)
    await v.connect(A).claimWithdraw(p)
    expect((await tok.balanceOf(A.address)) - before).to.equal(U(20_000))
  })

  it('FREEZE: rescue(A->B), un-blocklist A, A re-withdraws into pool p -> A can NEVER claim', async () => {
    const { firelight_vault: v, token_contract: tok, blocklister, rescuer, users, utils, config } =
      await loadFixture(deployVault.bind(null, { decimals: DEC, initial_deposit_limit: U(1_000_000) }))
    const [A] = users
    const B = ethers.Wallet.createRandom().address // rescue destination — a recovery custodian address
    await utils.mintAndApprove(U(30_000), A)
    await v.connect(A).deposit(U(30_000), A.address)

    const q = await v.currentPeriod()
    const p = q + 1n

    // (1) A's first withdrawal lands in pool p.
    await v.connect(A).withdraw(U(10_000), A.address, A.address)
    expect(await v.withdrawalsOf(p, A.address)).to.equal(U(10_000))

    // (2) Blocklist A, rescue A's pending withdrawal to B. Sets isWithdrawClaimed[p][A]=true.
    await v.connect(blocklister).addToBlocklist(A.address)
    await v.connect(rescuer).rescueWithdrawFromBlocklisted(A.address, B, [p])
    expect(await v.isWithdrawClaimed(p, A.address)).to.equal(true)

    // (3) Un-blocklist A — a legitimate, spec'd operation, for reasons unrelated to this bug. Same period q.
    await v.connect(blocklister).removeFromBlocklist(A.address)
    expect(await v.currentPeriod()).to.equal(q)

    // (4) A makes a new, ordinary withdrawal; currentPeriod is still q, so it lands in the SAME pool p.
    await v.connect(A).withdraw(U(10_000), A.address, A.address)
    expect(await v.withdrawalsOf(p, A.address)).to.equal(U(10_000)) // ledger says A is owed 10k again
    expect(await v.isWithdrawClaimed(p, A.address)).to.equal(true)  // ...but the slot is already "claimed"

    // (5) Advance so pool p is claimable. B claims A's original 10k cleanly.
    await time.increase(config.period_configuration_duration * 2)
    const bSigner = await ethers.getImpersonatedSigner(B)
    await ethers.provider.send('hardhat_setBalance', [B, '0x3635C9ADC5DEA00000'])
    const bBefore = await tok.balanceOf(B)
    await v.connect(bSigner).claimWithdraw(p)
    expect((await tok.balanceOf(B)) - bBefore).to.equal(U(10_000))

    // (6) A tries to claim the new 10k she is genuinely owed -> permanently reverts.
    await expect(v.connect(A).claimWithdraw(p)).to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')

    // (7) Recovery attempts fail: re-blocklist + re-rescue also reverts (slot marked claimed at :864).
    await v.connect(blocklister).addToBlocklist(A.address)
    const C = ethers.Wallet.createRandom().address
    await expect(v.connect(rescuer).rescueWithdrawFromBlocklisted(A.address, C, [p]))
      .to.be.revertedWithCustomError(v, 'AlreadyClaimedPeriod')

    // (8) The 10k is stranded in pendingWithdrawAssets, excluded from totalAssets -> dead weight.
    expect(await v.pendingWithdrawAssets()).to.equal(U(10_000))
    expect((await tok.balanceOf(v.target)) - (await v.totalAssets())).to.be.gte(U(10_000))
  })
})
```

(The committed file additionally logs each step to produce the trace above; both tests pass together.)
