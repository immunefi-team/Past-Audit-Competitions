# 90066 sc high function rescuewithdrawfromblocklisted permanently freezes a rescued account s future withdrawals

**Submitted on Aug 22nd 2026 at 20:18:18 UTC by @berring for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90066
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Function `rescueWithdrawFromBlocklisted` permanently marks a rescued account's withdrawal slot as claimed for a period that has not yet occurred. If the account is later removed from the blocklist and submits another withdrawal during the same period, the new request is booked into that already-poisoned slot. Once the claim window opens, `claimWithdraw` always reverts with `AlreadyClaimedPeriod`, permanently freezing the user's assets. The frozen amount remains included in `pendingWithdrawAssets`, causing `totalAssets()` to remain understated for all vault participants.

## Vulnerability Details

### Root cause

When a pending withdrawal is rescued, the function both transfers the withdrawal shares to the rescue destination and permanently marks the source account's `(period, account)` slot as claimed:

`contracts/core/FirelightVault.sol:861-871`

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true; // vulnerable write at line 870
    ...
}
```

The function does not restrict `periods[i]` to completed periods. This is important because normal withdrawal requests are always assigned to the next, still-future period:

`contracts/core/FirelightVault.sol:1008-1017`

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
```

Therefore, rescuing a normal pending withdrawal sets `isWithdrawClaimed[futurePeriod][from] = true` before that period occurs. While the current period is still active, the same account can be removed from the blocklist and submit another legitimate withdrawal. That new request is assigned to the same future period, but the previously written claim flag remains set.

### Why the new withdrawal can never be claimed

`claimWithdraw` checks `isWithdrawClaimed` before reading or converting the account's withdrawal-share balance:

`contracts/core/FirelightVault.sol:788-806`

```solidity
if (period >= currentPeriod()) revert InvalidPeriod();

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender],
    ...
);
if (assets == 0) revert NoWithdrawalAmount(period);
```

As a result, even when `withdrawSharesOf[period][sender]` contains a valid, non-zero entitlement from the later withdrawal, execution reverts at the claim-flag check and never reaches the balance conversion.

The condition is permanent:

* `isWithdrawClaimed` has only two writers in the contract: the normal claim path and the affected rescue path.
* Neither code path ever resets the flag to `false`.
* There is no administrative recovery function for the poisoned slot.
* Re-running the rescue cannot recover the new withdrawal because the function first checks the source account's already-latched flag and reverts with `AlreadyClaimedPeriod`, regardless of the supplied destination.

### Step-by-step exploitation scenario

No privileged role needs to exceed its documented permissions:

1. Alice deposits assets and requests a withdrawal during period `P`. The request is booked into period `P+1`, so `withdrawSharesOf[P+1][alice] > 0`.
2. `BLOCKLIST_ROLE` adds Alice to the blocklist.
3. `RESCUER_ROLE` legitimately calls `rescueWithdrawFromBlocklisted(alice, treasury, [P+1])`.
4. The function moves Alice's pending withdrawal to the treasury, zeroes her balance for that request, and sets `isWithdrawClaimed[P+1][alice] = true`.
5. `BLOCKLIST_ROLE` removes Alice from the blocklist using the vault's intended `removeFromBlocklist` function.
6. Before period `P` ends, Alice submits another withdrawal. It is assigned to the same period `P+1`, resulting in a new positive `withdrawSharesOf[P+1][alice]` balance.
7. After period `P+1` passes, Alice calls `claimWithdraw(P+1)`.
8. The call always reverts with `AlreadyClaimedPeriod(P+1)`, despite Alice having a positive withdrawal entitlement.
9. Re-running the rescue also reverts on the same flag, leaving no path through which Alice, the rescuer, or another account can release the assets.

The rescue destination can still claim the originally rescued withdrawal. This confirms that the problem is isolated to the poisoned `(period, account)` slot rather than a general failure of the vault.

### Why the vulnerable write is unnecessary

The preceding assignment already sets `withdrawSharesOf[period][from] = 0`, which is sufficient to prevent the rescued withdrawal from being claimed again:

* A later `claimWithdraw` call sees a zero balance and reverts with `NoWithdrawalAmount`.
* Repeated periods in the same rescue call, such as `[5, 5]`, are rejected because the second iteration reads the balance zeroed by the first iteration.
* The normal withdrawal path retains its independent double-claim protection.

The appropriate fix is to remove the redundant claim-flag write:

```diff
             withdrawSharesOf[periods[i]][to] += _withdrawOf;
             withdrawSharesOf[periods[i]][from] = 0;
-            isWithdrawClaimed[periods[i]][from] = true;

             rescuedShares[i] = _withdrawOf;
```

With this line removed, all 40 relevant existing project tests continue to pass, including the rescue tests and the ordinary double-claim protection test.

## Impact Details

The selected in-scope impact is **permanent freezing of funds**.

After the affected sequence occurs, the victim has a positive withdrawal entitlement that cannot be claimed through any available code path. The condition does not expire, and neither the victim nor an administrator can clear the poisoned claim flag. The loss is therefore permanent rather than a temporary denial of service.

The accounting consequences extend beyond the affected user. The frozen assets remain in `pendingWithdrawAssets`, while `totalAssets()` is calculated as:

```solidity
totalAssets() = super.totalAssets() - pendingWithdrawAssets;
```

Consequently:

* the victim permanently loses access to the assets represented by the poisoned withdrawal slot;
* `pendingWithdrawAssets` remains permanently overstated;
* `totalAssets()` remains understated for every remaining LP; and
* `totalAssetsAt()`, which is used by `CoverOrderAllocator._computeAvailableCapacity`, is also depressed, reducing the protocol's calculated sellable-cover capacity.

The runnable PoC demonstrates **3,000 fXRP permanently unclaimable** after the treasury successfully claims the originally rescued amount. The 3,000 fXRP remains counted in `pendingWithdrawAssets`.

This issue does not require an attacker to obtain a privileged role. The blocklisting, rescue, and removal from the blocklist are all legitimate protocol operations performed within the roles' documented authority. The permanent loss occurs when the unblocked user later uses the ordinary withdrawal flow.

## References

### Relevant code

* Vulnerable write: `contracts/core/FirelightVault.sol:861-871`
* Withdrawal-period assignment: `contracts/core/FirelightVault.sol:1008-1017`
* Fatal claim-flag check: `contracts/core/FirelightVault.sol:788-806`
* Vault accounting affected by the frozen pending amount: `contracts/core/FirelightVault.sol:428`
* In-scope commit: `42f9ea5` (`v1_audit_ready`)

### Fix verification

After removing the redundant `isWithdrawClaimed[periods[i]][from] = true` assignment, run:

```bash
npx hardhat test test/rescue.js test/deposit_and_withdraw.js
```

All 40 existing tests pass, including the rescue suite and the ordinary double-claim guard.

## Proof of Concept

## Proof of Concept

Create the following file:

```
test/poc/frescue-claim-slot-poisoning.js
```

Run it from the repository root with:

```bash
npx hardhat test test/poc/rescue-claim-slot-poisoning.js
```

Soucre code:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

/**
 * PoC
 * `FirelightVault.rescueWithdrawFromBlocklisted` permanently poisons a future claim slot.
 *
 * Root cause
 * ----------
 * FirelightVault.sol:870
 *     isWithdrawClaimed[periods[i]][from] = true;
 *
 * The rescue latches the *claimed* flag for a (period, account) pair, but the period it
 * latches has not happened yet — `_requestWithdraw` books every request into
 * `currentPeriod() + 1` (FirelightVault.sol:1008). The write is also redundant: the line
 * above it already sets `withdrawSharesOf[periods[i]][from] = 0`, which on its own makes
 * `claimWithdraw` revert with `NoWithdrawalAmount`.
 *
 * `claimWithdraw` reads the flag BEFORE it reads the share balance:
 *     FirelightVault.sol:794  if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
 *     FirelightVault.sol:796  assets = _convertToAssetsTotals(withdrawSharesOf[period][sender], ...);
 *
 * So once the flag is latched, any *new* entitlement the same account later acquires in
 * that same period is unclaimable — by anyone, forever. The assets stay counted in
 * `pendingWithdrawAssets`, which `totalAssets()` subtracts (FirelightVault.sol:428), so the
 * share price is permanently depressed for every remaining LP as well.
 */
describe('PoC rescueWithdrawFromBlocklisted permanently poisons a future claim slot', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const DEPOSIT = ethers.parseUnits('6000', DECIMALS)
  const RESCUED_WITHDRAW = ethers.parseUnits('2000', DECIMALS) // moved to the treasury by the rescue
  const STRANDED_WITHDRAW = ethers.parseUnits('3000', DECIMALS) // permanently lost by the victim

  const setup = () => deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })

  it('strands the victim\'s later withdrawal in the same period, permanently', async () => {
    const { firelight_vault, blocklister, rescuer, users, utils, config } = await loadFixture(setup)

    const alice = users[0] // victim
    const treasury = users[1] // rescue destination
    const other = users[2] // used later to prove the `from` flag is what blocks recovery

    await utils.mintAndApprove(DEPOSIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)

    const P = await firelight_vault.currentPeriod()
    const target = P + 1n // every request in period P is booked into P+1

    // ---------------------------------------------------------------- step 1
    // Alice requests a withdrawal. It lands in `target`.
    await firelight_vault.connect(alice).withdraw(RESCUED_WITHDRAW, alice.address, alice.address)
    expect(await firelight_vault.withdrawSharesOf(target, alice.address)).to.be.gt(0n)
    expect(await firelight_vault.isWithdrawClaimed(target, alice.address)).to.eq(false)

    // ---------------------------------------------------------------- step 2
    // Alice is blocklisted; the rescuer moves that pending withdrawal to the treasury.
    // This is the function used exactly as intended.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(alice.address, treasury.address, [target])

    // The rescue zeroed her shares (sufficient on its own) AND latched the claim flag
    // for a period that has not occurred yet. The latch is the defect.
    expect(await firelight_vault.withdrawSharesOf(target, alice.address)).to.eq(0n)
    expect(await firelight_vault.isWithdrawClaimed(target, alice.address)).to.eq(true)

    // ---------------------------------------------------------------- step 3
    // Alice is un-blocklisted. The vault exposes this explicitly; nothing about it is exotic.
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    expect(await firelight_vault.isBlocklisted(alice.address)).to.eq(false)

    // ---------------------------------------------------------------- step 4
    // Still inside period P, so a new request lands in the SAME, now-poisoned, period.
    expect(await firelight_vault.currentPeriod()).to.eq(P)
    await firelight_vault.connect(alice).withdraw(STRANDED_WITHDRAW, alice.address, alice.address)

    // The vault itself reports that Alice is owed assets for `target`.
    const aliceEntitlement = await firelight_vault.withdrawalsOf(target, alice.address)
    expect(aliceEntitlement).to.be.gt(0n)

    const pendingBeforeClaims = await firelight_vault.pendingWithdrawAssets()
    expect(pendingBeforeClaims).to.eq(RESCUED_WITHDRAW + STRANDED_WITHDRAW)

    // ---------------------------------------------------------------- step 5
    // Move past `target` so the claim window opens.
    await time.increase(Number(config.period_configuration_duration) * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(target)

    // ---------------------------------------------------------------- step 6
    // The treasury claims its rescued portion normally — the vault is not broken in general,
    // only Alice's slot is.
    await expect(firelight_vault.connect(treasury).claimWithdraw(target)).to.not.be.reverted

    // ---------------------------------------------------------------- step 7
    // Alice can never claim. The flag is checked before her (non-zero) balance is read.
    await expect(firelight_vault.connect(alice).claimWithdraw(target))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(target)

    // ---------------------------------------------------------------- step 8
    // There is no recovery path. `isWithdrawClaimed` has exactly two writers in the whole
    // contract (FirelightVault.sol:806 and :870) and neither can clear it; re-running the
    // rescue reverts on Alice's own latched flag, which is checked first.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, other.address, [target])
    )
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(target)

    // ---------------------------------------------------------------- damage
    // Alice's assets remain locked in the vault and still counted as pending, so
    // `totalAssets()` — and therefore the share price for every remaining LP — is
    // permanently understated by that amount.
    const pendingAfter = await firelight_vault.pendingWithdrawAssets()
    expect(pendingAfter).to.be.gte(STRANDED_WITHDRAW - 1n)
    expect(await firelight_vault.withdrawalsOf(target, alice.address)).to.be.gt(0n)

    console.log(`        victim entitlement permanently unclaimable: ${ethers.formatUnits(aliceEntitlement, DECIMALS)} fXRP`)
    console.log(`        still counted in pendingWithdrawAssets:     ${ethers.formatUnits(pendingAfter, DECIMALS)} fXRP`)
  })

  it('control: without the rescue the identical sequence lets the victim claim', async () => {
    const { firelight_vault, blocklister, users, utils, config } = await loadFixture(setup)

    const alice = users[0]

    await utils.mintAndApprove(DEPOSIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)

    const P = await firelight_vault.currentPeriod()
    const target = P + 1n

    // Same shape as the PoC — blocklist and un-blocklist — but no rescue in between,
    // so `isWithdrawClaimed[target][alice]` is never latched.
    await firelight_vault.connect(alice).withdraw(RESCUED_WITHDRAW, alice.address, alice.address)
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    await firelight_vault.connect(alice).withdraw(STRANDED_WITHDRAW, alice.address, alice.address)

    expect(await firelight_vault.isWithdrawClaimed(target, alice.address)).to.eq(false)

    await time.increase(Number(config.period_configuration_duration) * 2)

    // The claim succeeds and pays out both requests.
    await expect(firelight_vault.connect(alice).claimWithdraw(target)).to.not.be.reverted
    expect(await firelight_vault.isWithdrawClaimed(target, alice.address)).to.eq(true)
  })
})

```
