# 88420 sc high rescuewithdrawfromblocklisted sets the same flag claimwithdraw reads as already claimed and never clears it permanently freezing any later withdrawal the account makes into t&#x20;

**Submitted on Aug 13th 2026 at 04:37:23 UTC by @daivo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88420
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`isWithdrawClaimed[period][account]` carries two different meanings. `claimWithdraw` reads it as *"this account has already claimed this period"*; `rescueWithdrawFromBlocklisted` writes it as *"this account's withdrawal was rescued away"*. It is never cleared. Because a rescue can execute while the target bucket is still the *upcoming* one, the same account can afterwards accrue a fresh, fully legitimate withdrawal for exactly that period — and can then never claim it. No role can recover it either: the rescue path reverts on the very same flag. The assets remain counted in `pendingWithdrawAssets`, so they are also permanently subtracted from `totalAssets()` and from every staker's share price. In the PoC, 600 tokens sit in the vault while `totalAssets()` reads 0.

## Vulnerability Details

`claimWithdraw` treats the flag as the replay guard ([FirelightVault.sol:788-811](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L811)):

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // 794
...
isWithdrawClaimed[period][sender] = true;                                     // 806
```

`rescueWithdrawFromBlocklisted` reuses the same slot as a "moved away" marker ([FirelightVault.sol:861-873](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L861-L873)):

```solidity
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // 864
if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);   // 865
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;                                         // 870
```

There is no path anywhere in the contract that sets this mapping back to `false`.

**Why the normal claim path can never collide, but the rescue path can.** `claimWithdraw(period)` requires `period < currentPeriod()`, and `_requestWithdraw` always books into `currentPeriod() + 1`. So once an account has genuinely claimed period P, any new request necessarily lands in a bucket strictly greater than P. The two can never meet — the flag is safe there.

The rescue has no such constraint. `periods` is arbitrary caller-supplied input, gated only on `withdrawSharesOf[period][from] != 0`. A rescue therefore commonly executes while the bucket is still `currentPeriod() + 1` — i.e. **before** the period in which further requests would also be booked into it. Once the account is removed from the blocklist, its next `withdraw`/`redeem` in that same period books into the poisoned bucket:

```solidity
uint256 period = currentPeriod() + 1;                         // same bucket
...
withdrawSharesOf[period][receiver] += sharesWithdraw;         // credited normally
pendingWithdrawAssets += assets;                              // liability recorded
```

`_requestWithdraw` never consults `isWithdrawClaimed`, so the request succeeds, the shares are burned, and `withdrawalsOf(period, account)` reports a positive owed balance. Only at claim time does line 794 reject it — permanently.

**There is no recovery path for the user.**

* `claimWithdraw` — reverts `AlreadyClaimedPeriod` forever.
* `rescueWithdrawFromBlocklisted` again (after re-blocklisting) — reverts at line 864 on the same flag.
* `rescueSharesFromBlocklisted` — operates on ERC-20 share balances, but the shares were already burned by the withdrawal request. Irrelevant.
* No admin setter, no sweep, no force-claim exists. Nothing short of a contract upgrade returns the assets to the victim.

*One precise caveat, stated so it cannot be used as a rebuttal.* The stranded assets are not literally untouchable by the protocol for the whole of their life: while `capturePeriod == P - 1` and `currentPeriod ∈ {P - 1, P}`, `payout()` can still draw from `withdrawAssets[P]` and decrement `pendingWithdrawAssets`, so an incident payout in that window can consume part of the balance. That is not a recovery — the victim receives nothing either way, and the assets go to the payout receiver. Once `currentPeriod > P` the window closes and the balance is stranded permanently, unreachable by every function in the contract.

**This is live today, not only in the audited version.** The defect is not introduced by V2. `contracts/legacy/FirelightVaultPredeposits.sol:754-760` — the implementation currently behind proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` — contains the identical overload:

```solidity
if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);   // 754
if (isWithdrawClaimed[periods[i]][to])   revert AlreadyClaimedPeriod(periods[i]);   // 755
...
isWithdrawClaimed[periods[i]][from] = true;                                         // 760
```

and the same replay guard in `claimWithdraw` at lines 694/706. The live vault holds 58,284,732 FXRP with `pendingWithdrawAssets = 226,259.11 FXRP` — the withdrawal buckets this bug operates on are non-empty right now. The V1→V2 upgrade carries the defect forward unchanged.

**The triggering sequence uses only documented, in-permission actions.** `BLOCKLIST_ROLE` blocklists a flagged account; `RESCUER_ROLE` rescues the pending withdrawal to a compliant beneficiary — exactly what the function is for; `BLOCKLIST_ROLE` later removes the account once the compliance issue is resolved, which is why `removeFromBlocklist` exists. The user then performs an ordinary withdrawal. **No party misbehaves at any point**, and the loss lands on the innocent user.

The deployed vault runs **1-day periods** (`periodConfigurations(0) = (1764682969, 86400, 0)` on proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`), so "the same period" is a 24-hour window — well within a same-day resolution of a false-positive screening hit.

The project's own `test/rescue.js` contains 30 tests covering this function but **never removes an account from the blocklist after a rescue**, so the state is unreachable by the existing suite.

## Impact Details

Direct, permanent, unrecoverable loss of the victim's withdrawal, plus a permanent reduction of the vault's usable collateral.

Measured in the PoC (1,000 staked, 400 rescued away, 600 withdrawn afterwards):

```
Alice owed but unclaimable : 600.0
vault balance held hostage  : 600.0
pendingWithdrawAssets stuck : 600.0
totalAssets() (excludes it) : 0.0
```

Consequences:

1. **The user's funds are permanently frozen.** The full second withdrawal is unrecoverable by the victim, and by every privileged role, short of a contract upgrade. (See the caveat in *Vulnerability Details*: an incident `payout()` can consume part of the balance while the capture window is still open, but that never returns it to the victim.)
2. **The protocol's collateral is permanently reduced.** Because `totalAssets() = balanceOf(vault) - pendingWithdrawAssets`, the stranded amount is excluded from the share price forever. Other stakers do not absorb it as yield — it is simply removed from the accounting. In the PoC the vault physically holds 600 tokens while reporting `totalAssets() == 0`.
3. **Cover capacity shrinks correspondingly and permanently.** `CoverOrderAllocator._computeAvailableCapacity` values staked collateral via `vault.totalAssetsAt(currentPeriodStart())`, so every stranded amount permanently reduces how much cover the protocol can sell.
4. **The damage is unbounded in principle.** The frozen amount equals whatever the account withdraws after being un-blocklisted — up to its entire remaining stake. Each occurrence is independent and cumulative across accounts and periods.

This is the in-scope `Permanent freezing of funds` impact, which the program classifies as Critical.

**Honest note on feasibility.** This requires a specific but entirely legitimate operational sequence: blocklist → rescue → un-blocklist → withdraw again, all within one vault period. It is not attacker-triggerable at will. If the program weighs that down, `Temporary freezing of funds for at least 24 hours` (High) understates it, since the freeze is genuinely permanent and no emergency action short of a contract upgrade recovers the funds — and the program states emergency actions are not an automatic reason to downgrade.

## References

* `claimWithdraw` replay guard — [FirelightVault.sol:788-811](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788-L811)
* `rescueWithdrawFromBlocklisted` — [FirelightVault.sol:845-876](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845-L876)
* `_requestWithdraw` books into `currentPeriod() + 1` and never checks the flag — [FirelightVault.sol:994-1028](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1028)
* `totalAssets()` excludes `pendingWithdrawAssets` — [FirelightVault.sol:427-429](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L427-L429)
* Identical defect in the currently-deployed V1 implementation — `contracts/legacy/FirelightVaultPredeposits.sol:694, 706, 754-760`
* Capacity reads the same figure — [CoverOrderAllocator.sol:710-717](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L710-L717)
* Existing rescue tests that never un-blocklist — `test/rescue.js`
* Live proxy period configuration — <https://flare-explorer.flare.network/address/0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3>

## Recommended fix

Separate the two meanings. The rescue should zero the balance rather than mark it claimed, since `withdrawSharesOf[period][from] = 0` already prevents a double rescue via the existing `_withdrawOf == 0` guard:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
-   isWithdrawClaimed[periods[i]][from] = true;
```

If an explicit marker is preferred, use a distinct mapping (`isWithdrawRescued`) that `claimWithdraw` does not read. Alternatively, make `claimWithdraw` idempotent against balance rather than a boolean — zero `withdrawSharesOf[period][sender]` on claim and drop the flag entirely.

***

## Proof of Concept

Runnable PoC: `test/integration/poc_rescue_poison.js` (2 tests, both passing against `v1_audit_ready`). It uses the project's own `test/setup/fixtures.js` vault fixture with default settings.

```bash
npm install
npx hardhat test test/integration/poc_rescue_poison.js
```

Observed output:

```
PoC: rescue poisons isWithdrawClaimed and permanently freezes a later withdrawal
      Alice owed but unclaimable : 600.0
      vault balance held hostage  : 600.0
      pendingWithdrawAssets stuck : 600.0
      totalAssets() (excludes it) : 0.0
  ✔ freezes an innocent user's funds with no recovery path
  ✔ control: without the rescue, the identical second withdrawal claims normally
```

### Step-by-step

Let `P = currentPeriod() + 1` — the bucket every withdrawal made this period lands in.

1. **Alice stakes 1,000.** `deposit(1000, alice)`.
2. **Alice requests a partial withdrawal of 400.** `withdraw(400, alice, alice)`. Now `withdrawSharesOf[P][alice] > 0`.
3. **Compliance flags Alice.** `BLOCKLIST_ROLE` calls `addToBlocklist(alice)` — an ordinary in-permission action.
4. **`RESCUER_ROLE` rescues the pending withdrawal to Bob.** `rescueWithdrawFromBlocklisted(alice, bob, [P])` — exactly what the function exists for. State afterwards: `withdrawSharesOf[P][alice] == 0`, `withdrawSharesOf[P][bob] > 0`, and critically **`isWithdrawClaimed[P][alice] == true`**.
5. **The flag is resolved and Alice is un-blocklisted.** `removeFromBlocklist(alice)`. Still the same vault period — assert `currentPeriod() == P - 1`.
6. **Alice withdraws her remaining 600.** `withdraw(600, alice, alice)` **succeeds**: shares are burned, `withdrawSharesOf[P][alice] > 0`, `pendingWithdrawAssets` increases, and `withdrawalsOf(P, alice)` reports 600 owed. `_requestWithdraw` never consults `isWithdrawClaimed`.
7. **Advance two periods** so bucket `P` becomes claimable (`currentPeriod() > P`).
8. **Alice calls `claimWithdraw(P)` → reverts `AlreadyClaimedPeriod(P)`.** She has claimed nothing; the stale rescue marker blocks her.
9. **Confirm there is no recovery.** Re-blocklist Alice and call `rescueWithdrawFromBlocklisted(alice, bob, [P])` again → reverts `AlreadyClaimedPeriod(P)` at line 864, on the same flag. No other function can move the balance.
10. **Confirm the funds are stranded protocol-wide.** Bob claims his rescued portion normally; afterwards the vault still physically holds 600 tokens, `pendingWithdrawAssets` still counts them, and `totalAssets()` reports **0**. The assets are unreachable by anyone and permanently excluded from the share price and from cover-capacity accounting.
11. **Control (test 2).** Without steps 3–5, the identical pair of withdrawals (400 then 600, same period) claims normally and Alice receives the full amount — isolating the rescue as the sole cause.

### PoC source — full file, not present in the repo

These files do **not** exist in `v1_audit_ready`. Create them at the paths shown below (they import only fixtures that already ship with the project), then run the command above. No other change to the repository is required.

**`test/integration/poc_rescue_poison.js`**

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * PoC — `rescueWithdrawFromBlocklisted` permanently poisons the (period, account) pair.
 *
 * `isWithdrawClaimed[period][account]` is overloaded:
 *   - claimWithdraw    reads it as "this account already claimed this period"  (line 794)
 *   - the rescue path  writes it as "this account's withdrawal was rescued away" (line 870)
 *
 * It is never cleared. A normal claim can never collide, because claiming period P requires
 * currentPeriod > P while a new request always lands in currentPeriod + 1 > P.
 *
 * The rescue path CAN collide: it can be executed while the target bucket is still the
 * *upcoming* one, so the same account can afterwards accrue fresh withdrawal shares for
 * exactly that period — which it can then never claim, and which no rescue can recover.
 */
describe('PoC: rescue poisons isWithdrawClaimed and permanently freezes a later withdrawal', function () {
  const DEC = 6
  const amt = (n) => ethers.parseUnits(n.toString(), DEC)

  it('freezes an innocent user\'s funds with no recovery path', async () => {
    const { firelight_vault: vault, token_contract, blocklister, rescuer, users, utils, config } =
      await loadFixture(deployVault)
    const [alice, bob] = users

    // Alice stakes 1,000.
    await utils.mintAndApprove(amt(1000), alice)
    await vault.connect(alice).deposit(amt(1000), alice.address)

    // The bucket every withdrawal made this period will land in.
    const period = (await vault.currentPeriod()) + 1n

    // --- 1. Alice makes a partial withdrawal request (400 of her 1,000).
    await vault.connect(alice).withdraw(amt(400), alice.address, alice.address)
    const firstShares = await vault.withdrawSharesOf(period, alice.address)
    expect(firstShares).to.be.gt(0n)

    // --- 2. Alice is blocklisted and the pending withdrawal is rescued to Bob.
    //        Both are ordinary, in-permission compliance actions.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [period])

    expect(await vault.withdrawSharesOf(period, alice.address)).to.equal(0n)
    expect(await vault.withdrawSharesOf(period, bob.address)).to.equal(firstShares)
    // The rescue marks Alice as having "claimed" this period.
    expect(await vault.isWithdrawClaimed(period, alice.address)).to.equal(true)

    // --- 3. The blocklisting is resolved (false positive / sanctions cleared) and Alice
    //        is removed from the blocklist. Still the same vault period.
    await vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await vault.currentPeriod()).to.equal(period - 1n)

    // --- 4. Alice withdraws her remaining 600. This is a brand-new, fully legitimate
    //        request that lands in the SAME bucket, because it is still the same period.
    await vault.connect(alice).withdraw(amt(600), alice.address, alice.address)

    const strandedShares = await vault.withdrawSharesOf(period, alice.address)
    const owedAssets = await vault.withdrawalsOf(period, alice.address)
    expect(strandedShares).to.be.gt(0n)
    expect(owedAssets).to.be.gt(0n)

    // --- 5. The bucket becomes claimable.
    await time.increase(config.period_configuration_duration * 2)
    expect(await vault.currentPeriod()).to.be.gt(period)

    // --- 6. Alice can never claim. The stale rescue marker blocks her forever.
    await expect(vault.connect(alice).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(period)

    // --- 7. There is no recovery path. Re-blocklisting and rescuing again reverts on the
    //        very same marker, so even the RESCUER cannot retrieve the funds.
    await vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, bob.address, [period])
    ).to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(period)

    // --- 8. The assets are permanently stranded: still counted in pendingWithdrawAssets,
    //        therefore permanently excluded from totalAssets() and from every staker's
    //        share price. Nobody can ever reach them.
    const pending = await vault.pendingWithdrawAssets()
    const vaultBal = await token_contract.balanceOf(vault.target)
    expect(pending).to.be.gte(owedAssets)

    // Bob claims his rescued portion; Alice's share still cannot move.
    await vault.connect(bob).claimWithdraw(period)
    const pendingAfter = await vault.pendingWithdrawAssets()
    const balAfter = await token_contract.balanceOf(vault.target)

    expect(pendingAfter).to.be.gte(owedAssets)
    expect(balAfter).to.be.gte(owedAssets)
    expect(await vault.totalAssets()).to.equal(balAfter - pendingAfter)

    console.log(`        Alice owed but unclaimable : ${ethers.formatUnits(owedAssets, DEC)}`)
    console.log(`        vault balance held hostage  : ${ethers.formatUnits(balAfter, DEC)}`)
    console.log(`        pendingWithdrawAssets stuck : ${ethers.formatUnits(pendingAfter, DEC)}`)
    console.log(`        totalAssets() (excludes it) : ${ethers.formatUnits(await vault.totalAssets(), DEC)}`)
  })

  it('control: without the rescue, the identical second withdrawal claims normally', async () => {
    const { firelight_vault: vault, token_contract, users, utils, config } = await loadFixture(deployVault)
    const [alice] = users

    await utils.mintAndApprove(amt(1000), alice)
    await vault.connect(alice).deposit(amt(1000), alice.address)
    const period = (await vault.currentPeriod()) + 1n

    await vault.connect(alice).withdraw(amt(400), alice.address, alice.address)
    await vault.connect(alice).withdraw(amt(600), alice.address, alice.address)

    await time.increase(config.period_configuration_duration * 2)

    const owed = await vault.withdrawalsOf(period, alice.address)
    await expect(vault.connect(alice).claimWithdraw(period)).to.emit(vault, 'CompleteWithdraw')
    expect(await token_contract.balanceOf(alice.address)).to.equal(owed)
  })
})
```

### Environment

* Branch `v1_audit_ready`, local tree verified byte-identical to the in-scope files.
* Full suite with the PoC added: **478 passing, 0 failing**.
