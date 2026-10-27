# 89856 sc high rescue path permanently freezes a user s funds by marking a future period as already claimed

## #89856 \[SC-High] Rescue path permanently freezes a user's funds by marking a future period as already claimed

**Submitted on Aug 21st 2026 at 14:33:18 UTC by @pindarev for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89856
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of funds

### Description

### Brief / Intro

`rescueWithdrawFromBlocklisted` sets `isWithdrawClaimed[period][from] = true` for a period that has not started yet and is still accepting new withdrawal requests. The flag is permanent — no function in the contract ever clears it — and `claimWithdraw` checks it *before* it looks at the user's balance. If the account is later un-blocklisted (the normal end of a compliance hold) and submits a fresh withdrawal that lands in that same period, its shares are burned and the assets can never be claimed by anyone. The stranded assets also stay inside `pendingWithdrawAssets` forever, permanently understating `totalAssets()` for every other depositor.

No privileged party acts maliciously or incorrectly anywhere in this flow. The rescuer performs a correct rescue and the blocklister performs a correct un-blocklisting; the loss is caused entirely by the contract writing a flag whose meaning ("this account has been paid for this period") is false.

### Vulnerability Details

## Root cause

A permanent, per-`(period, account)` latch is written for a period whose share balance is still mutable, and the latch is checked before the balance it was meant to stand in for.

`FirelightVault.rescueWithdrawFromBlocklisted` ([L845-L873](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845)):

```solidity
withdrawSharesOf[periods[i]][to] += shares;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;   // L870
```

`FirelightVault.claimWithdraw` ([L788-L800](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788)):

```solidity
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);   // L794
```

`FirelightVault._requestWithdraw` ([L994](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994)) credits into `currentPeriod() + 1` and **never reads `isWithdrawClaimed`**:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);   // shares are burned here
```

The rescue targets `currentPeriod() + 1` — the only period that can hold a pending withdrawal — which is precisely the period that is still accepting new credits for the remainder of the current period. The implicit assumption is *"a rescued address will never again hold withdraw shares for this period."* Nothing enforces it, and `removeFromBlocklist` makes violating it routine.

The write is also **redundant against its own stated purpose**: line 869 already zeroes `withdrawSharesOf[periods[i]][from]`, which alone makes `claimWithdraw` compute `assets == 0` and revert `NoWithdrawalAmount`, and makes a second rescue revert too. Its only observable effect is the poisoning.

There is no recovery path. Re-blocklisting and re-rescuing hits the identical `isWithdrawClaimed` guard at the top of the rescue and reverts. No function anywhere writes `isWithdrawClaimed = false`.

### Impact Details

**Permanent freezing of user funds.** The victim's shares are burned by `_update(owner, address(0), shares)` at request time, and the corresponding assets can never be claimed by the victim, by the rescuer, or by anyone else. This is unrecoverable without a contract upgrade.

**Permanent accounting corruption affecting all other depositors.** The stranded assets remain in `pendingWithdrawAssets`, and `totalAssets()` is defined as `super.totalAssets() - pendingWithdrawAssets`. Every future share-price computation, every `payout` exposure cap (`totalAssetsAt`), and the cover capacity that `CoverOrderAllocator._computeAvailableCapacity` reads via `totalAssetsAt(currentPeriodStart())` are all permanently understated by the frozen amount. The damage is not contained to the victim.

**Trigger is a routine compliance operation, not an attack.** Blocklisting a user on a screening hit, rescuing their pending withdrawal, and then clearing the hit when it proves to be a false positive is exactly the workflow these functions exist to support. Any user who withdraws again in the same period after being cleared loses those funds.

### References

* [`FirelightVault.rescueWithdrawFromBlocklisted` L845-L873](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L845) — sets the permanent flag at L870
* [`FirelightVault.claimWithdraw` L788](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L788) — checks the flag at L794, before the balance
* [`FirelightVault._requestWithdraw` L994](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994) — re-credits the same period without consulting the flag

### Mitigation

Delete the flag write. Zeroing the share balance on the preceding line is already sufficient to prevent both a double-claim and a double-rescue:

```diff
  withdrawSharesOf[periods[i]][to] += shares;
  withdrawSharesOf[periods[i]][from] = 0;
- isWithdrawClaimed[periods[i]][from] = true;
```

If a latch is still wanted for auditability, only set it for periods that are already closed (`periods[i] < currentPeriod()`), which can no longer receive credits.

### Proof of Concept

### Proof of Concept

Add the following test as `test/FirelightVault/rescueWithdrawPermanentFreeze.js` and run it with `npx hardhat test test/FirelightVault/rescueWithdrawPermanentFreeze.js`

PoC:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('rescueWithdrawFromBlocklisted permanently bricks a still-open withdrawal pool', function () {
  it('a routine blocklist -> rescue -> unblocklist cycle permanently strands a legitimate later withdrawal into the same period', async () => {
    const { firelight_vault, blocklister, rescuer, users, utils, config } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: ethers.parseUnits('100000', 6) })
    )
    const DEPOSIT = ethers.parseUnits('1000', 6)
    const WITHDRAW = ethers.parseUnits('400', 6)
    const alice = users[0]
    const clean = users[1]

    await utils.mintAndApprove(DEPOSIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)

    // 1) Alice requests a withdrawal; it lands in the *next*, still-open period's pool.
    const currentPeriod = await firelight_vault.currentPeriod()
    const targetPeriod = currentPeriod + 1n
    await firelight_vault.connect(alice).withdraw(WITHDRAW, alice.address, alice.address)
    expect(await firelight_vault.withdrawSharesOf(targetPeriod, alice.address)).to.equal(WITHDRAW)

    // 2) Alice is blocklisted for a routine reason (KYC/AML hold) and RESCUER_ROLE moves her
    //    pending withdrawal for that still-open period to a clean address -- ordinary compliance
    //    operations, no malice or contract misuse involved.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(
      alice.address, clean.address, [targetPeriod]
    )

    // isWithdrawClaimed[targetPeriod][alice] is now permanently set to true, even though
    // targetPeriod hasn't even started yet and Alice hasn't claimed anything.
    expect(await firelight_vault.isWithdrawClaimed(targetPeriod, alice.address)).to.equal(true)
    expect(await firelight_vault.withdrawSharesOf(targetPeriod, alice.address)).to.equal(0)

    // 3) Alice is cleared -- the normal end of a compliance hold -- and, still within the same
    //    current period, submits a brand-new, fully legitimate withdrawal request.
    await firelight_vault.connect(blocklister).removeFromBlocklist(alice.address)
    await utils.mintAndApprove(DEPOSIT, alice)
    await firelight_vault.connect(alice).deposit(DEPOSIT, alice.address)
    expect(await firelight_vault.currentPeriod()).to.equal(currentPeriod) // still the same period -> same target pool
    await firelight_vault.connect(alice).withdraw(WITHDRAW, alice.address, alice.address)

    // Her new shares are correctly credited into the pool...
    expect(await firelight_vault.withdrawSharesOf(targetPeriod, alice.address)).to.equal(WITHDRAW)

    // 4) Roll forward until the target period has fully elapsed and become claimable.
    await time.increase(Number(config.period_configuration_duration) * 2)
    expect(await firelight_vault.currentPeriod()).to.be.gt(targetPeriod)

    // 5) Her second, entirely legitimate withdrawal can never be claimed -- permanently -- because
    //    the flag set in step 2 was never meant to apply to a request that didn't exist yet.
    await expect(firelight_vault.connect(alice).claimWithdraw(targetPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(targetPeriod)

    // No function anywhere in the contract ever writes isWithdrawClaimed back to false, and there
    // is no sweep path for the stranded assets -- re-blocklisting/re-rescuing her hits the same
    // flag and reverts identically.
    await firelight_vault.connect(blocklister).addToBlocklist(alice.address)
    await expect(
      firelight_vault.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, clean.address, [targetPeriod])
    ).to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod').withArgs(targetPeriod)
  })
})
```
