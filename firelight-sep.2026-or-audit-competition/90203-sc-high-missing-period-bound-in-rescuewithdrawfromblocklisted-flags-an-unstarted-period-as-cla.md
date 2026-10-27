# 90203 sc high missing period bound in rescuewithdrawfromblocklisted flags an unstarted period as claimed permanently freezing the account s next withdrawal

**Submitted on Aug 23rd 2026 at 17:11:08 UTC by @illusi1 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90203
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal to a clean beneficiary and, as part of that, sets `isWithdrawClaimed[period][from] = true`. The period it marks has not started yet, and never can have, because a pending withdrawal only ever exists for `currentPeriod() + 1`. Nothing in the contract ever sets that flag back to false. If the sanction is later lifted and the account withdraws again before that period ends, the new credit lands on a key already marked as paid: the shares are burned, the assets are added to `pendingWithdrawAssets`, and `claimWithdraw` reverts `AlreadyClaimedPeriod` for as long as the contract exists. On mainnet this destroys the affected LP's entire remaining position — 750,000 fXRP in the proof of concept — with no recovery path in the deployed implementation.

## Vulnerability Details

`claimWithdraw` uses `isWithdrawClaimed[period][account]` as a one-way latch that refuses a second payout for a period the account already collected:

```solidity
// contracts/core/FirelightVault.sol:791-806
if (period >= currentPeriod()) revert InvalidPeriod();

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
...
pendingWithdrawAssets -= assets;
isWithdrawClaimed[period][sender] = true;
```

The latch is safe here because of the line above it. `claimWithdraw` accepts only a period that has already closed (`:791`), and a closed period can never receive a new credit, so once the flag is set nothing can arrive under that key again.

The rescue path sets the same flag with no such restriction:

```solidity
// contracts/core/FirelightVault.sol:862-870
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

`periods` is caller-supplied and unbounded, but the `_withdrawOf == 0` check at `:866` means the only periods that can be rescued are the ones the account actually has a booking in — and a booking is only ever created for the period after the current one:

```solidity
// contracts/core/FirelightVault.sol:1008-1019
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
...
pendingWithdrawAssets += assets;
```

So the rescue always latches a period that has not started. This is not an unusual way to use the function; it is the only way. `_requestWithdraw` keeps accepting credits into that period until it ends and never consults `isWithdrawClaimed`. Once the account is off the blocklist, `withdraw` and `redeem` accept it like any other holder, and the `+=` at `:1017` writes to the same `(period, account)` key the rescue marked.

From that moment the position is unrecoverable:

* `claimWithdraw` reverts at `:794` before it ever reads `withdrawSharesOf`.
* A second rescue is refused by the symmetric check at `:864`, on the identical flag.
* `rescueSharesFromBlocklisted` (`:818`) only moves `balanceOf` shares and never touches `withdrawSharesOf` — and the shares are already burned.
* `isWithdrawClaimed` is written at exactly two places in the contract, `:806` and `:870`, and set to false at none.

That the developers wrote the mirrored check for `to` at `:865` shows they understood the flag has to stay paired with `withdrawSharesOf`. The pairing is broken because the write at `:870` outlives the entitlement it was meant to describe.

## Impact Details

**Permanent freezing of funds.** The affected LP loses its entire next withdrawal, and the assets it leaves behind are unreachable by anyone.

In the proof of concept: Alice deposits 1,000,000 fXRP and requests 250,000 back. She is blocklisted and the rescuer moves that 250,000 to a clean address — the rescue feature used exactly as documented. The sanction is lifted. Alice then redeems her remaining 750,000 fXRP of shares in the ordinary way, inside the same period. The shares are burned, `pendingWithdrawAssets` rises by 750,000e6, and `claimWithdraw` reverts `AlreadyClaimedPeriod`. The vault's token balance is 1,750,000e6 while `totalAssets()` reports 1,000,000e6; the 750,000e6 difference belongs to nobody. Bob, who now holds 100% of the supply, cannot reach it either, because `totalAssets()` subtracts `pendingWithdrawAssets` (`:428`). Alice's fXRP balance is zero.

The exposure window is the remainder of the period in which the rescue happened — one week under the default configuration in the project's own fixture, and at minimum one day (`SMALLEST_PERIOD_DURATION`). Any withdrawal or redemption the account books in that window is lost. The account has no way to see it coming: `withdrawalsOf(period, alice)` keeps returning 750,000e6, so any interface reading the vault shows a claimable balance that will never pay out.

**Preconditions, stated up front.** The sequence needs two privileged calls, and both are the feature working as documented, not an abuse: `RESCUER_ROLE` rescues a pending withdrawal, and `BLOCKLIST_ROLE` later removes the account from the blocklist. Neither role exceeds its permissions, nothing is stolen, and no attacker profits — the loss is pure destruction of an LP's principal. The harmful variant is the ordinary one, because every pending withdrawal an account has is for a period that has not started. The final step is taken by the LP itself through a plain `withdraw` or `redeem`.

**On permanence, separating what is shown from what is argued.** The PoC shows the refusal surviving a year of block time, a repeat rescue attempt and a re-blocklist cycle, and `isWithdrawClaimed` has no write-false site anywhere in the contract, so within the deployed implementation the freeze has no exit. `FirelightVault` sits behind a proxy and carries an `initializeV2` reinitializer, so an upgrade could add a clearing path; that recovery does not exist in the audited code.

## References

* `contracts/core/FirelightVault.sol:862-870` — `rescueWithdrawFromBlocklisted`, the unbounded period and the flag write
* `contracts/core/FirelightVault.sol:791-806` — `claimWithdraw`, the closed-period bound that makes the same flag safe there
* `contracts/core/FirelightVault.sol:1008-1019` — `_requestWithdraw`, booking into `currentPeriod() + 1`
* `contracts/core/FirelightVault.sol:428` — `totalAssets()` subtracting `pendingWithdrawAssets`
* `contracts/core/FirelightVault.sol:818` — `rescueSharesFromBlocklisted`, which does not touch `withdrawSharesOf`
* Branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

## Proof of Concept

## Proof of Concept

A runnable Hardhat test is included below. It uses the repository's own fixture (`test/setup/fixtures.js`), the real `FirelightVault` behind its usual proxy, and only public entrypoints. No state is planted with cheatcodes.

### Steps

1. Deploy the vault through the project fixture. Two LPs, Alice and Bob, each deposit 1,000,000 fXRP. Bob is present so the vault still has holders once Alice is gone.
2. Advance one period so the vault is in normal operation.
3. Alice calls `withdraw(250_000e6)`. `_requestWithdraw` books it into `period = currentPeriod() + 1` — a period that has not started.
4. `BLOCKLIST_ROLE` calls `addToBlocklist(alice)`. This is an ordinary compliance action.
5. `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted(alice, recipient, [period])` to move the pending 250,000 to a clean address. This is the rescue feature used as documented, and `period` is the only period Alice has a booking in. **Side effect:** `isWithdrawClaimed[period][alice]` is now `true` for a period that has not started. Assert it directly.
6. `BLOCKLIST_ROLE` calls `removeFromBlocklist(alice)` — the sanction is lifted.
7. Still inside the same period, Alice calls `redeem` for her whole share balance (750,000 fXRP). It succeeds: her shares are burned, `withdrawSharesOf[period][alice]` is credited again, and `pendingWithdrawAssets` rises by 750,000e6.
8. Advance past the end of `period` so it closes.
9. The claim path is healthy for everyone else: `recipient` calls `claimWithdraw(period)` and receives the rescued 250,000e6.
10. Alice calls `claimWithdraw(period)` and it reverts `AlreadyClaimedPeriod(period)`, even though `withdrawalsOf(period, alice)` reports 750,000e6.
11. Re-running the rescue for Alice reverts on the same flag at `:864`. Waiting a year and calling `claimWithdraw` again still reverts. There is no other path: `isWithdrawClaimed` is set to false nowhere in the contract.
12. Final state: the vault holds 1,750,000e6 fXRP, `totalAssets()` reports 1,000,000e6, Alice holds zero shares and zero fXRP, and the 750,000e6 difference is reachable by nobody — `totalAssets()` subtracts `pendingWithdrawAssets` at `:428`, so it is not in Bob's NAV either.

### Test file

`test/poc/rescue_withdraw_flag_strands_next_withdrawal.js`

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { deployVault } = require('../setup/fixtures.js')

const E6 = 1_000_000n // the vault asset (fXRP) has 6 decimals

describe('PoC: a rescued period stays flagged claimed and strands the next withdrawal', function () {
  it('the de-blocklisted LP burns 750,000 fXRP of shares into a period it can never claim', async () => {
    const s = await loadFixture(deployVault.bind(null, { initial_deposit_limit: (10_000_000n * E6).toString() }))
    const vault = s.firelight_vault
    const [alice, bob, recipient] = s.users

    await s.utils.mintAndApprove(1_000_000n * E6, alice)
    await s.utils.mintAndApprove(1_000_000n * E6, bob)
    await vault.connect(alice).deposit(1_000_000n * E6, alice.address)
    await vault.connect(bob).deposit(1_000_000n * E6, bob.address)

    await time.increase(s.config.period_configuration_duration)

    // Alice requests a 250,000 withdrawal. It books into the next period, which has not started.
    await vault.connect(alice).withdraw(250_000n * E6, alice.address, alice.address)
    const period = (await vault.currentPeriod()) + 1n
    expect(await vault.withdrawalsOf(period, alice.address)).to.equal(250_000n * E6)

    // Compliance blocklists Alice; the rescuer moves her pending withdrawal to a clean address.
    await vault.connect(s.blocklister).addToBlocklist(alice.address)
    await vault.connect(s.rescuer).rescueWithdrawFromBlocklisted(alice.address, recipient.address, [period])

    // The side effect: a period that has not started is flagged claimed for Alice.
    expect(await vault.isWithdrawClaimed(period, alice.address)).to.equal(true)
    expect(period).to.be.greaterThan(await vault.currentPeriod())
    expect(await vault.withdrawSharesOf(period, alice.address)).to.equal(0n)

    await vault.connect(s.blocklister).removeFromBlocklist(alice.address)

    const pendingBefore = await vault.pendingWithdrawAssets()

    // Alice exits the rest of her position, still inside the same period.
    await vault.connect(alice).redeem(await vault.balanceOf(alice.address), alice.address, alice.address)
    expect(await vault.balanceOf(alice.address)).to.equal(0n)
    expect(await vault.withdrawalsOf(period, alice.address)).to.equal(750_000n * E6)
    expect((await vault.pendingWithdrawAssets()) - pendingBefore).to.equal(750_000n * E6)

    await time.increase(2n * BigInt(s.config.period_configuration_duration))
    expect(await vault.currentPeriod()).to.be.greaterThan(period)

    // The claim path itself is healthy: the rescue beneficiary collects for the same period.
    await expect(vault.connect(recipient).claimWithdraw(period)).to.changeTokenBalance(
      s.token_contract, recipient, 250_000n * E6,
    )

    // Alice cannot. The vault says she is owed 750,000 and refuses to pay it.
    expect(await vault.withdrawalsOf(period, alice.address)).to.equal(750_000n * E6)
    await expect(vault.connect(alice).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(period)

    // Re-running the rescue does not help — it refuses on the same flag.
    await vault.connect(s.blocklister).addToBlocklist(alice.address)
    await expect(vault.connect(s.rescuer).rescueWithdrawFromBlocklisted(alice.address, recipient.address, [period]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(period)
    await vault.connect(s.blocklister).removeFromBlocklist(alice.address)

    // A year later, nothing has changed.
    await time.increase(365 * 24 * 3600)
    await expect(vault.connect(alice).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod').withArgs(period)

    const vaultBalance = await s.token_contract.balanceOf(vault.target)
    console.log('    vault token balance              :', vaultBalance.toString())
    console.log('    pendingWithdrawAssets            :', (await vault.pendingWithdrawAssets()).toString())
    console.log('    totalAssets() now                :', (await vault.totalAssets()).toString())
    console.log('    Alice shares                     :', (await vault.balanceOf(alice.address)).toString())
    console.log('    Alice claimable per withdrawalsOf:', (await vault.withdrawalsOf(period, alice.address)).toString())
    console.log('    Alice fXRP balance               :', (await s.token_contract.balanceOf(alice.address)).toString())

    expect(await vault.pendingWithdrawAssets()).to.equal(750_000n * E6)
    expect(await s.token_contract.balanceOf(alice.address)).to.equal(0n)
    expect(await vault.balanceOf(bob.address)).to.equal(await vault.totalSupply())
    expect(await vault.totalAssets()).to.equal(1_000_000n * E6)
    expect(vaultBalance).to.equal(1_750_000n * E6)
  })
})
```

### Running it

From the repository root at branch `v1_audit_ready` (commit `42f9ea5`), after `npm ci`:

```
npx hardhat test test/poc/rescue_withdraw_flag_strands_next_withdrawal.js
```

Output:

```
  PoC: a rescued period stays flagged claimed and strands the next withdrawal
    vault token balance              : 1750000000000
    pendingWithdrawAssets            : 750000000000
    totalAssets() now                : 1000000000000
    Alice shares                     : 0
    Alice claimable per withdrawalsOf: 750000000000
    Alice fXRP balance               : 0
    ✔ the de-blocklisted LP burns 750,000 fXRP of shares into a period it can never claim

  1 passing
```

### Recommended mitigation

Drop the flag write in the rescue loop. `withdrawSharesOf[periods[i]][from] = 0` at `:869` already removes the entitlement, and the `_withdrawOf == 0` check at `:866` already blocks a second rescue of the same slice, so the write at `:870` buys nothing and is the entire cause of the defect. Keep the read at `:864`, which still correctly refuses to rescue a period the account has genuinely claimed.

```diff
--- a/contracts/core/FirelightVault.sol
+++ b/contracts/core/FirelightVault.sol
@@ -868,4 +868,3 @@
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;

             rescuedShares[i] = _withdrawOf;
```

I checked the change rather than proposing it blind. With that single line removed the PoC above fails at its first assertion — `isWithdrawClaimed(period, alice)` is no longer set — and the repository's own suite (`test/rescue.js`, `test/blocklist.js`, `test/deposit_and_withdraw.js`) passes 65 of 65. Double-claim protection is unaffected: `claimWithdraw` never zeroes `withdrawSharesOf`, so the flag it sets at `:806` remains the only thing standing between an account and a second payout.

Bounding the rescue with `periods[i] >= currentPeriod()` instead would be worse: it would make it impossible to rescue exactly the withdrawal a compliance action needs to intercept, since the only pending withdrawal an account ever has is for a period that has not started.
