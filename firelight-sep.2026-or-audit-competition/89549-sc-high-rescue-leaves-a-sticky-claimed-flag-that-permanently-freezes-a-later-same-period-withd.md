# 89549 sc high rescue leaves a sticky claimed flag that permanently freezes a later same period withdrawal

**Submitted on Aug 19th 2026 at 20:19:06 UTC by @Kuzomo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89549
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted` moves a blocklisted account's pending withdrawal to a safe address and permanently marks the source account as having claimed that period. The flag is never cleared: both assignments to it in `FirelightVault` set it to `true`, and no function in the repository clears the mapping. If that account is later removed from the blocklist and makes an ordinary new withdrawal request while the originating period is still current, the request is accepted and credited, but `claimWithdraw` reverts `AlreadyClaimedPeriod` and a second rescue reverts on the same flag. The entitlement is unreachable by every path the contract offers. In the run below the account ends with 0 shares, 0 tokens, and 500.000000 FXRP that `withdrawalsOf()` still reports as owed; `pendingWithdrawAssets` retains the same amount forever, which also permanently understates `totalAssets()` for every other staker. No role misbehaves at any point: blocklist, rescue and un-blocklist are the documented compliance workflow performed correctly, and the step that traps the funds is an ordinary `redeem` by the now-cleared account.

## Vulnerability Details

`isWithdrawClaimed` has exactly two writers in `FirelightVault` and both set it to `true`. No function in any in-scope contract clears the mapping, and there is no setter or admin override:

```
FirelightVaultStorage.sol:71   mapping(uint256 period => mapping(address account => bool value)) public isWithdrawClaimed;
FirelightVault.sol:794         if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
FirelightVault.sol:806         isWithdrawClaimed[period][sender] = true;
FirelightVault.sol:864         if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
FirelightVault.sol:865         if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
FirelightVault.sol:870         isWithdrawClaimed[periods[i]][from] = true;
```

The rescue empties the source account and then marks it claimed:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L868-L870>

`true` here means "this account has been settled for this period", but the account was emptied, not paid. The write path never consults the flag. `_requestWithdraw` targets `currentPeriod() + 1`, so a second request made before the originating period rolls lands in the same period and is credited normally:

```solidity
uint256 period = currentPeriod() + 1;
...
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1008-L1019>

Both release paths then reject it. `claimWithdraw` reverts at line 794 before it ever reaches the conversion, and a second `rescueWithdrawFromBlocklisted` reverts at line 864 on the identical flag. `rescueSharesFromBlocklisted` (line 818) moves share balances, not withdrawal entitlements, and the account's `balanceOf` is already zero, so it cannot help either. Recovery requires a contract upgrade.

`withdrawalsOf` performs the same conversion as `claimWithdraw` but does not consult the flag, so the view surface keeps reporting the amount as owed while both release paths revert: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L466-L474>

## Impact Details

Permanent freezing of user funds. A staker who is blocklisted, rescued and subsequently cleared loses the entire value of any new withdrawal request they make before that period ends, with no on-chain recovery. In the run below the amount is 500.000000 FXRP, but the mechanism scales with the position: it strands whatever the second request is worth, not a rounding residue. The loss is silent, because `withdrawalsOf()` continues to report the funds as claimable, and it is triggered by an ordinary user action that the contract accepts without warning. `pendingWithdrawAssets` is decremented only inside the now-unreachable `claimWithdraw` and inside `payout`, so the stranded amount stays subtracted from `totalAssets()` (<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L427-L429>) permanently, understating the vault for every remaining staker. `FirelightVault` is the V2 implementation for the live predeposit vault at `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`, which returns `totalAssets() = 56,601,078,263,347` (56.6M FXRP at 6 decimals) on a read taken 2026-08-19, so this lands on a vault holding real staker capital at upgrade time.

Feasibility Limitations check. No trusted role misbehaves, exceeds its privilege or is misconfigured anywhere in this sequence. Blocklisting, rescuing and un-blocklisting are fixed documented operations, not configuration values or discretionary parameters, and each is performed correctly here; nothing requires a trusted party to act maliciously or incorrectly, and no privilege is exceeded. The protocol breaks while every party behaves exactly as intended. The action that loses the funds is a permissionless `redeem` by an account that is not blocklisted at that moment. The precondition is a blocklisting that is later reversed, which is the ordinary lifecycle of a screening false positive. That path is provisioned and live today, not hypothetical: on the same vault `hasRole(BLOCKLIST_ROLE, 0x189993f3B53284c7e5C02b058EFC6618F78D0cEA)` and `hasRole(RESCUER_ROLE, 0xC115757266b902d4b1284eA24DBbE557C15E963E)` both return `true`. Attack investment: none. Chain rollbacks and pre-impact monitoring: not cited; the state transition is committed in the user's own transaction and there is no pause or guardian that inspects it. An informed user could avoid it by waiting for the next period, but nothing on-chain or in the view surface tells them to.

## Recommendation

Clear the flag when the rescue empties the source account, since the account has not been paid:

```solidity
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = false;
```

The flag's purpose is to prevent double-claiming a balance, and after the rescue the source balance is zero, so `claimWithdraw`'s own `assets == 0` check at line 803 already provides that protection. If the marker is needed for accounting, track rescued periods in a separate mapping that `_requestWithdraw` resets on a fresh credit, and make `withdrawalsOf` return zero whenever `isWithdrawClaimed` is set so the view cannot report unreachable funds.

## References

* rescue sets the terminal flag: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L876>
* claimWithdraw rejects on it: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L811>
* \_requestWithdraw credits without consulting it: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028>
* storage declaration, no other writer: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L71>

## Proof of Concept

## Proof of Concept

Local Hardhat only; no mainnet or public-testnet transaction was sent. Clone the in-scope repository at the audited revision, install, save the file below as `test/poc_sticky_claim.js`, and run it. It uses only the repository's own fixtures.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight && cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
npx hardhat test test/poc_sticky_claim.js
```

```javascript
// PoC — rescueWithdrawFromBlocklisted leaves a terminal isWithdrawClaimed flag that permanently
// locks a later same-period withdrawal by the same, now-unblocklisted account.
//
// Run:  npx hardhat test test/poc_sticky_claim.js
const { expect } = require('chai')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures')

describe('PoC: sticky isWithdrawClaimed permanently freezes a later withdrawal', function () {
  this.timeout(120000)

  it('victim ends with 0 shares, 0 tokens and 500.000000 FXRP that nobody can ever release', async () => {
    const f = await deployVault({ decimals: 6 })
    const { firelight_vault: vault, token_contract, users, utils, blocklister, rescuer, config } = f
    const [victim, safeAddr] = users

    // 1. Victim stakes 1000.000000 and redeems half, creating a request for the next period.
    await utils.mintAndApprove(1_000_000_000n, victim)
    await vault.connect(victim).deposit(1_000_000_000n, victim.address)
    const half = (await vault.balanceOf(victim.address)) / 2n
    const rc1 = await (await vault.connect(victim).redeem(half, victim.address, victim.address)).wait()
    const period = rc1.logs.map(l => { try { return vault.interface.parseLog(l) } catch { return null } })
      .find(p => p && p.name === 'WithdrawRequest').args[3]
    console.log('  request 1 -> period', period.toString(),
      '| shares', (await vault.withdrawSharesOf(period, victim.address)).toString())

    // 2. Compliance ops blocklist the victim and rescue the entitlement. Both are the documented,
    //    correct use of these roles.
    await vault.connect(blocklister).addToBlocklist(victim.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safeAddr.address, [period])
    console.log('  after rescue: isWithdrawClaimed[p][victim] =',
      await vault.isWithdrawClaimed(period, victim.address))

    // 3. Victim is cleared and un-blocklisted.
    await vault.connect(blocklister).removeFromBlocklist(victim.address)

    // 4. Victim redeems the rest. Still inside the originating period, so it lands in the SAME period.
    const rc2 = await (await vault.connect(victim).redeem(half, victim.address, victim.address)).wait()
    const period2 = rc2.logs.map(l => { try { return vault.interface.parseLog(l) } catch { return null } })
      .find(p => p && p.name === 'WithdrawRequest').args[3]
    expect(period2).to.equal(period)
    const owed = await vault.withdrawalsOf(period, victim.address)
    console.log('  request 2 -> period', period2.toString(), '| withdrawalsOf() reports owed:', owed.toString())

    // 5. Neither release path works, permanently.
    await time.increase(config.period_configuration_duration * 2)
    await expect(vault.connect(victim).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
    console.log('  claimWithdraw  -> revert AlreadyClaimedPeriod')

    await vault.connect(blocklister).addToBlocklist(victim.address)
    await expect(vault.connect(rescuer).rescueWithdrawFromBlocklisted(victim.address, safeAddr.address, [period]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
    console.log('  second rescue  -> revert AlreadyClaimedPeriod')
    await vault.connect(blocklister).removeFromBlocklist(victim.address)

    // 6. Attribute the loss. The rescued half is fine and the safe address claims it; only the
    //    victim's second request is destroyed.
    const safeOwed = await vault.withdrawalsOf(period, safeAddr.address)
    const before = await token_contract.balanceOf(safeAddr.address)
    await vault.connect(safeAddr).claimWithdraw(period)
    const safeGot = (await token_contract.balanceOf(safeAddr.address)) - before

    console.log('  --- loss attribution ---')
    console.log('  rescued leg claimed by safe address :', safeGot.toString())
    console.log('  victim irrecoverable entitlement    :', owed.toString())
    console.log('  pendingWithdrawAssets never released:', (await vault.pendingWithdrawAssets()).toString())
    console.log('  victim shares', (await vault.balanceOf(victim.address)).toString(),
      '| victim tokens', (await token_contract.balanceOf(victim.address)).toString())

    expect(safeGot).to.equal(safeOwed)
    expect(owed).to.equal(500_000_000n)
    expect(await vault.balanceOf(victim.address)).to.equal(0n)
    expect(await token_contract.balanceOf(victim.address)).to.equal(0n)
    expect(await vault.pendingWithdrawAssets()).to.equal(500_000_000n)
  })
})
```

Observed output:

```
  PoC: sticky isWithdrawClaimed permanently freezes a later withdrawal
  request 1 -> period 1 | shares 500000000
  after rescue: isWithdrawClaimed[p][victim] = true
  request 2 -> period 1 | withdrawalsOf() reports owed: 500000000
  claimWithdraw  -> revert AlreadyClaimedPeriod
  second rescue  -> revert AlreadyClaimedPeriod
  --- loss attribution ---
  rescued leg claimed by safe address : 500000000
  victim irrecoverable entitlement    : 500000000
  pendingWithdrawAssets never released: 500000000
  victim shares 0 | victim tokens 0
    ✔ victim ends with 0 shares, 0 tokens and 500.000000 FXRP that nobody can ever release

  1 passing
```

The rescued leg is deliberately excluded from the loss: the safe address claims it in full, and the assertion `safeGot == safeOwed` proves it. The 500.000000 FXRP attributed as lost is only the second request, which the account made after being cleared, and the final assertions confirm the account holds 0 shares and 0 tokens while `pendingWithdrawAssets` still carries the full amount.
