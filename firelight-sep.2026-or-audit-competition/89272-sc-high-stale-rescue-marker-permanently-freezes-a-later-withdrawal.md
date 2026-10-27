# 89272 sc high stale rescue marker permanently freezes a later withdrawal

**Submitted on Aug 18th 2026 at 11:13:38 UTC by @Paludo0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89272
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Firelight can permanently lock a user's later withdrawal after rescue moves an earlier withdrawal from a blocklisted account and the account is restored. The issue comes from the interaction between the rescue marker and the ordinary withdrawal accounting, because each path uses a different rule for when a withdrawal key is final. The user loses access to the assets represented by burned shares, and normal claim or rescue cannot recover them.

## Vulnerability Details

FirelightVault stores withdrawal shares by period and receiver. Rescue marks the source entry as claimed, while a normal withdrawal always writes to the next period and does not check that marker. A blocklist removal only changes the blocklist status. These three paths can therefore create a withdrawal entry that contains assets but is already marked as claimed.

### Step by step vulnerability explanation

1. `FirelightVault::claimWithdraw()` permits a claim only after the selected period ends. It then treats `isWithdrawClaimed` as a final state:

```solidity
// contracts/core/FirelightVault.sol:791-806
if (period >= currentPeriod()) revert InvalidPeriod();

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

pendingWithdrawAssets -= assets;
isWithdrawClaimed[period][sender] = true;
```

The period check normally protects the marker because a normal withdrawal writes only to `currentPeriod() + 1`. A future key should remain open until the period ends.

2. `FirelightVault::rescueWithdrawFromBlocklisted()` does not require the selected period to have ended:

```solidity
// contracts/core/FirelightVault.sol:861-870
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

An authorized rescue can therefore mark `currentPeriod() + 1` as claimed while the normal withdrawal path can still write to that key. Rescue moves the old amount to another receiver, but it does not prevent a later request from using the source key again.

3. `FirelightVault::removeFromBlocklist()` restores the account without checking its withdrawal state:

```solidity
// contracts/core/FirelightVault.sol:560-562
function removeFromBlocklist(address account)
    external onlyRole(BLOCKLIST_ROLE) onlyBlocklisted(account)
{
    isBlocklisted[account] = false;
    emit RemovedFromBlocklist(account);
}
```

The account can now call the ordinary withdrawal functions again. No state transition clears or validates the claimed marker.

4. `FirelightVault::_requestWithdraw()` writes to the next period and burns shares without checking the receiver marker:

```solidity
// contracts/core/FirelightVault.sol:1008-1025
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

If the restored account requests another withdrawal to itself, the contract adds value to a key that already reads as claimed and burns the related shares.

5. After the period ends, `claimWithdraw()` sees the old marker and reverts. A later rescue also sees the same marker and reverts. The new pending amount remains in the withdrawal accounting, but no normal function can settle it.

### Attack path

1. A shareholder calls `FirelightVault::withdraw()` during period `c`. The request creates an entry for period `p = c + 1`.
2. `BLOCKLIST_ROLE` calls `FirelightVault::addToBlocklist()` for the shareholder.
3. Before period `p` ends, `RESCUER_ROLE` calls `FirelightVault::rescueWithdrawFromBlocklisted()` for period `p`.
4. The rescue moves the old shares and sets `isWithdrawClaimed[p][shareholder]` to true.
5. `BLOCKLIST_ROLE` calls `FirelightVault::removeFromBlocklist()`.
6. During period `c`, the shareholder calls `FirelightVault::withdraw()` again to itself.
7. `_requestWithdraw()` adds the new shares to period `p` and burns the shareholder's shares.
8. After period `p` ends, `FirelightVault::claimWithdraw(p)` and a new rescue both revert with `AlreadyClaimedPeriod`.

The two role holders perform their documented operations. The defect is that those valid operations create a claimed future key, and the public withdrawal path bypasses the intended final state by writing to it again.

## Impact Details

* Funds at risk: any later withdrawal amount written to the already claimed key. The amount can reach the user's remaining share position.
* Who loses: the rehabilitated shareholder loses access to the assets represented by the newly burned shares.
* Repeatability: the sequence can affect each account that follows the rescue and rehabilitation order before the next period ends.
* No self healing: claim and rescue both reject the marker. Recovery requires an upgrade, migration, or a dedicated state repair function.
* Accounting effect: the stranded amount remains included in `pendingWithdrawAssets` after the period matures.
* Existing defenses: pausing can stop new withdrawal requests, but it cannot restore shares already burned into the claimed key.
* Severity: Critical. The live competition lists permanent freezing of funds as a Critical impact. This path permanently blocks access to user funds through the normal withdrawal lifecycle.
* Scope and actor status: the root is in `contracts/core/FirelightVault.sol`. The role holders only perform the documented rescue and account restoration operations. The public shareholder then bypasses the intended final marker because `_requestWithdraw()` accepts a key that is already claimed. The issue does not require a role holder to use an unauthorized input or permission.
* Known issue and duplicate status: the current public known issues concern the FLB token assumptions, external FLB custody, and aggregate claim limits. The public Macro audit describes rescue as a trusted compliance operation, but it does not describe a future withdrawal key marked as claimed and then reused after the account is restored. I found no matching issue in the published findings.

## Recommended Fix

Reject a request before changing accounting or burning shares when its destination key is already claimed:

```solidity
uint256 period = currentPeriod() + 1;
if (isWithdrawClaimed[period][receiver]) {
    revert AlreadyClaimedPeriod(period);
}
```

The rescue path should also either require a matured period or use a separate rescue state that cannot block later writes to the same account and period key.

## Proof of Concept

## PoC Explanation

* The PoC is `test/FirelightVault/staleRescueWithdrawal.poc.js`.
* It imports and calls the original `deployVault()` fixture from `test/setup/fixtures.js`.
* That fixture deploys the actual `FirelightVault` through the project upgrade proxy, creates the developer FAsset mock, and provides the role accounts used by the original tests.
* The PoC also uses the original `utils.mintAndApprove()` helper, Hardhat network helpers, and the same Chai assertions used by `test/rescue.js`.
* The test creates an ordinary withdrawal, performs the documented blocklist and rescue operations before maturity, restores the account, and creates another withdrawal for the same period key.
* It leaves one day between the first request and rescue, then another day before the second request. Both requests remain in the same seven day period. No exact timestamp or period boundary is forced.
* It confirms that the second request burns shares and adds pending assets even though the key is already claimed. After maturity, both claim and rescue reject the second amount.

Run:

```
npx hardhat test test/FirelightVault/staleRescueWithdrawal.poc.js
```

Actual Hardhat output:

```
  FirelightVault stale rescue withdrawal PoC
claimed marker after the second withdrawal: true
elapsed time between withdrawal requests: 172800 seconds
locked withdrawal assets: 50000.0
pending assets after the rescued withdrawal is claimed: 50000.0
    ✔ keeps a later withdrawal unclaimable after rescue reuses its period key (1143ms)

  1 passing (1s)
```

```
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('FirelightVault stale rescue withdrawal PoC', function () {
  const DECIMALS = 6
  const DEPOSIT = ethers.parseUnits('1000000', DECIMALS)
  const FIRST_WITHDRAWAL = ethers.parseUnits('100000', DECIMALS)
  const SECOND_WITHDRAWAL = ethers.parseUnits('50000', DECIMALS)
  const ONE_DAY = 24 * 60 * 60

  async function deployDeveloperFixture() {
    const context = await deployVault({ initial_deposit_limit: DEPOSIT })
    await context.utils.mintAndApprove(DEPOSIT, context.users[0])
    await context.firelight_vault.connect(context.users[0]).deposit(DEPOSIT, context.users[0].address)
    return context
  }

  it('keeps a later withdrawal unclaimable after rescue reuses its period key', async function () {
    const { firelight_vault, blocklister, rescuer, users, config } = await loadFixture(deployDeveloperFixture)
    const shareholder = users[0]
    const rescueRecipient = users[1]
    const currentPeriod = await firelight_vault.currentPeriod()
    const withdrawalPeriod = currentPeriod + 1n

    await firelight_vault
      .connect(shareholder)
      .withdraw(FIRST_WITHDRAWAL, shareholder.address, shareholder.address)

    await time.increase(ONE_DAY)
    expect(await firelight_vault.currentPeriod()).to.eq(currentPeriod)

    await firelight_vault.connect(blocklister).addToBlocklist(shareholder.address)
    await firelight_vault
      .connect(rescuer)
      .rescueWithdrawFromBlocklisted(shareholder.address, rescueRecipient.address, [withdrawalPeriod])

    expect(await firelight_vault.isWithdrawClaimed(withdrawalPeriod, shareholder.address)).to.eq(true)
    expect(await firelight_vault.withdrawSharesOf(withdrawalPeriod, shareholder.address)).to.eq(0)
    expect(await firelight_vault.withdrawSharesOf(withdrawalPeriod, rescueRecipient.address)).to.eq(FIRST_WITHDRAWAL)

    await firelight_vault.connect(blocklister).removeFromBlocklist(shareholder.address)

    await time.increase(ONE_DAY)
    expect(await firelight_vault.currentPeriod()).to.eq(currentPeriod)

    const sharesBefore = await firelight_vault.balanceOf(shareholder.address)
    await firelight_vault
      .connect(shareholder)
      .withdraw(SECOND_WITHDRAWAL, shareholder.address, shareholder.address)

    const claimedAfterSecondWithdrawal = await firelight_vault.isWithdrawClaimed(
      withdrawalPeriod,
      shareholder.address
    )
    expect(await firelight_vault.balanceOf(shareholder.address)).to.eq(sharesBefore - SECOND_WITHDRAWAL)
    expect(await firelight_vault.withdrawSharesOf(withdrawalPeriod, shareholder.address)).to.eq(SECOND_WITHDRAWAL)
    expect(claimedAfterSecondWithdrawal).to.eq(true)
    expect(await firelight_vault.pendingWithdrawAssets()).to.eq(FIRST_WITHDRAWAL + SECOND_WITHDRAWAL)

    await time.increase(config.period_configuration_duration * 2)
    expect(await firelight_vault.currentPeriod()).to.be.greaterThan(withdrawalPeriod)

    await firelight_vault.connect(rescueRecipient).claimWithdraw(withdrawalPeriod)
    expect(await firelight_vault.pendingWithdrawAssets()).to.eq(SECOND_WITHDRAWAL)

    await expect(firelight_vault.connect(shareholder).claimWithdraw(withdrawalPeriod))
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)

    await firelight_vault.connect(blocklister).addToBlocklist(shareholder.address)
    await expect(
      firelight_vault
        .connect(rescuer)
        .rescueWithdrawFromBlocklisted(shareholder.address, rescueRecipient.address, [withdrawalPeriod])
    )
      .to.be.revertedWithCustomError(firelight_vault, 'AlreadyClaimedPeriod')
      .withArgs(withdrawalPeriod)

    expect(await firelight_vault.withdrawSharesOf(withdrawalPeriod, shareholder.address)).to.eq(SECOND_WITHDRAWAL)
    expect(await firelight_vault.withdrawAssets(withdrawalPeriod)).to.be.greaterThan(0)

    const lockedWithdrawalAssets = await firelight_vault.withdrawalsOf(withdrawalPeriod, shareholder.address)
    console.log('claimed marker after the second withdrawal:', claimedAfterSecondWithdrawal)
    console.log('elapsed time between withdrawal requests:', `${ONE_DAY * 2} seconds`)
    console.log('locked withdrawal assets:', ethers.formatUnits(lockedWithdrawalAssets, DECIMALS))
    console.log(
      'pending assets after the rescued withdrawal is claimed:',
      ethers.formatUnits(await firelight_vault.pendingWithdrawAssets(), DECIMALS)
    )
  })
})

```
