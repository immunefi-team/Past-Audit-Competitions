# 88425 sc high rescue marks an open withdrawal bucket claimed and permanently freezes later post unblock principal

**Submitted on Aug 13th 2026 at 05:34:52 UTC by @rtw0x1 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88425
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`rescueWithdrawFromBlocklisted()` sets `isWithdrawClaimed[period][from] = true`, but the same account/period withdrawal bucket remains writable after the account is unblocked. A later ordinary withdrawal is added to that already-claimed bucket. Once the period matures, `claimWithdraw()` reverts with `AlreadyClaimedPeriod`; after the user is blocklisted again, `rescueWithdrawFromBlocklisted()` reverts on the same stale marker. This leaves the newly deposited principal permanently inaccessible through the current implementation.

The fresh-funds test below first rescues the user's entire old balance, then unblocks the user, deposits a new 1,000 units, and requests withdrawal into the same still-open destination period. This proves that the frozen principal did not exist when the rescue was performed.

## Severity Rationale

* Suggested severity: Critical
* Impacted component: FirelightVault delayed-withdrawal and compliance-rescue state machine
* Preconditions: A user is blocklisted, their existing withdrawal is rescued, they are unblocked while that destination period remains open, and they later deposit and request another withdrawal into the same period.

The later 1,000-unit principal remains recorded in `withdrawalsOf` and `pendingWithdrawAssets`, but every current-contract release path rejects it. Time does not clear the state. Recovery requires a proxy upgrade or exceptional state migration, matching the program's “Permanent freezing of funds” impact.

## Impact

An ordinary restored user can lose access to all principal placed into a later same-account/same-period withdrawal after the supported compliance sequence. The funds remain reserved and excluded from active vault assets, but neither the user nor the rescue role can release them under the current code. There is no time-based recovery; remediation requires new implementation code or exceptional migration.

## Remediation

Make a claimed withdrawal key immutable. When a new withdrawal would target an account/period whose claim marker is already set, route it to a new unclaimed bucket or revert before accepting shares. A more robust design is to track claimed share amounts or a per-bucket generation instead of a permanent boolean, so rescue authority applies only to the exact shares that existed when rescue executed. Add a regression test for rescue, unblock, fresh deposit, later withdrawal, maturity, and successful recovery.

## Proof of Concept

## Steps To Reproduce

1. Clone the competition repository and check out the frozen revision:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-firelight.git firelight-poc
   cd firelight-poc
   git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
   npm ci
   mkdir -p test/poc
   ```
2. Save the following first test as `test/poc/rescue-open-bucket.poc.js`:

```javascript
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const { deployVault } = require('../setup/fixtures.js')

describe('PoC: rescue marker poisons a later withdrawal in the same open bucket', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const DEPOSIT = ethers.parseUnits('6000', DECIMALS)
  const FIRST_REQUEST = ethers.parseUnits('2000', DECIMALS)
  const LATER_REQUEST = ethers.parseUnits('1000', DECIMALS)

  async function deployFundedVault() {
    const ctx = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })
    const user = ctx.users[0]
    const receiver = ctx.users[1]

    await ctx.utils.mintAndApprove(DEPOSIT, user)
    await ctx.firelight_vault.connect(user).deposit(DEPOSIT, user.address)

    return { ...ctx, user, receiver }
  }

  async function request(vault, user, assets) {
    const tx = await vault.connect(user).withdraw(assets, user.address, user.address)
    const receipt = await tx.wait()
    const event = receipt.logs.find((log) => log.fragment && log.fragment.name === 'WithdrawRequest')
    if (!event) throw new Error('WithdrawRequest event not found')
    return event.args[3]
  }

  it('permanently strands later post-unblock principal after rescue writes the old marker', async function () {
    const {
      firelight_vault: vault,
      token_contract: asset,
      user,
      receiver,
      blocklister,
      rescuer,
      config
    } = await deployFundedVault()

    const period = await request(vault, user, FIRST_REQUEST)

    await vault.connect(blocklister).addToBlocklist(user.address)
    await vault.connect(rescuer).rescueWithdrawFromBlocklisted(user.address, receiver.address, [period])
    await vault.connect(blocklister).removeFromBlocklist(user.address)

    const laterPeriod = await request(vault, user, LATER_REQUEST)
    expect(laterPeriod).to.equal(period)

    const poisonedState = {
      period: period.toString(),
      currentPeriod: (await vault.currentPeriod()).toString(),
      userWithdrawShares: (await vault.withdrawSharesOf(period, user.address)).toString(),
      userWithdrawalAssets: (await vault.withdrawalsOf(period, user.address)).toString(),
      userClaimMarker: await vault.isWithdrawClaimed(period, user.address),
      receiverWithdrawShares: (await vault.withdrawSharesOf(period, receiver.address)).toString(),
      pendingWithdrawAssets: (await vault.pendingWithdrawAssets()).toString()
    }
    console.log('FIRELIGHT_RESCUE_POC_POISONED=' + JSON.stringify(poisonedState))

    expect(await vault.withdrawSharesOf(period, user.address)).to.equal(LATER_REQUEST)
    expect(await vault.withdrawalsOf(period, user.address)).to.equal(LATER_REQUEST)
    expect(await vault.isWithdrawClaimed(period, user.address)).to.equal(true)
    expect(await vault.withdrawSharesOf(period, receiver.address)).to.equal(FIRST_REQUEST)
    expect(await vault.pendingWithdrawAssets()).to.equal(FIRST_REQUEST + LATER_REQUEST)

    await time.increase(config.period_configuration_duration * 2)
    expect(await vault.currentPeriod()).to.be.greaterThan(period)

    const receiverBefore = await asset.balanceOf(receiver.address)
    await expect(vault.connect(receiver).claimWithdraw(period))
      .to.emit(vault, 'CompleteWithdraw')
      .withArgs(receiver.address, FIRST_REQUEST, period)
    expect(await asset.balanceOf(receiver.address)).to.equal(receiverBefore + FIRST_REQUEST)

    await expect(vault.connect(user).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(period)

    await vault.connect(blocklister).addToBlocklist(user.address)
    await expect(vault.connect(rescuer).rescueWithdrawFromBlocklisted(user.address, receiver.address, [period]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(period)

    const vaultAddress = await vault.getAddress()
    const finalState = {
      userWithdrawalAssets: (await vault.withdrawalsOf(period, user.address)).toString(),
      userWithdrawShares: (await vault.withdrawSharesOf(period, user.address)).toString(),
      userClaimMarker: await vault.isWithdrawClaimed(period, user.address),
      pendingWithdrawAssets: (await vault.pendingWithdrawAssets()).toString(),
      rawUnderlyingInVault: (await asset.balanceOf(vaultAddress)).toString(),
      activeTotalAssets: (await vault.totalAssets()).toString(),
      receiverAssetBalance: (await asset.balanceOf(receiver.address)).toString()
    }
    console.log('FIRELIGHT_RESCUE_POC_FINAL=' + JSON.stringify(finalState))

    expect(await vault.withdrawalsOf(period, user.address)).to.equal(LATER_REQUEST)
    expect(await vault.pendingWithdrawAssets()).to.equal(LATER_REQUEST)
    expect(await asset.balanceOf(vaultAddress)).to.equal(DEPOSIT - FIRST_REQUEST)
    expect(await vault.totalAssets()).to.equal(DEPOSIT - FIRST_REQUEST - LATER_REQUEST)
  })

  it('negative control: the same block/unblock sequence without rescue pays both requests', async function () {
    const {
      firelight_vault: vault,
      token_contract: asset,
      user,
      blocklister,
      config
    } = await deployFundedVault()

    const period = await request(vault, user, FIRST_REQUEST)
    await vault.connect(blocklister).addToBlocklist(user.address)
    await vault.connect(blocklister).removeFromBlocklist(user.address)
    expect(await request(vault, user, LATER_REQUEST)).to.equal(period)

    expect(await vault.isWithdrawClaimed(period, user.address)).to.equal(false)
    expect(await vault.withdrawalsOf(period, user.address)).to.equal(FIRST_REQUEST + LATER_REQUEST)

    await time.increase(config.period_configuration_duration * 2)

    const before = await asset.balanceOf(user.address)
    await expect(vault.connect(user).claimWithdraw(period))
      .to.emit(vault, 'CompleteWithdraw')
      .withArgs(user.address, FIRST_REQUEST + LATER_REQUEST, period)
    expect(await asset.balanceOf(user.address)).to.equal(before + FIRST_REQUEST + LATER_REQUEST)
    expect(await vault.pendingWithdrawAssets()).to.equal(0n)

    const controlState = {
      period: period.toString(),
      paidAssets: (FIRST_REQUEST + LATER_REQUEST).toString(),
      pendingWithdrawAssets: (await vault.pendingWithdrawAssets()).toString(),
      userClaimMarker: await vault.isWithdrawClaimed(period, user.address)
    }
    console.log('FIRELIGHT_RESCUE_POC_CONTROL=' + JSON.stringify(controlState))
  })
})
```

Save the following second test as `test/poc/rescue-fresh-funds.poc.js`:

```javascript
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const { deployVault } = require('../setup/fixtures.js')

describe('Validator control: rescue marker reaches genuinely future funds', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS)
  const FIRST_PRINCIPAL = ethers.parseUnits('2000', DECIMALS)
  const FRESH_PRINCIPAL = ethers.parseUnits('1000', DECIMALS)

  async function request(vault, user, assets) {
    const receipt = await (await vault.connect(user).withdraw(assets, user.address, user.address)).wait()
    const event = receipt.logs.find((log) => log.fragment && log.fragment.name === 'WithdrawRequest')
    if (!event) throw new Error('WithdrawRequest event not found')
    return event.args[3]
  }

  it('poisons principal deposited only after the original rescue and unblock', async function () {
    const ctx = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })
    const user = ctx.users[0]
    const receiver = ctx.users[1]
    const vault = ctx.firelight_vault
    const asset = ctx.token_contract

    await ctx.utils.mintAndApprove(FIRST_PRINCIPAL, user)
    await vault.connect(user).deposit(FIRST_PRINCIPAL, user.address)
    const period = await request(vault, user, FIRST_PRINCIPAL)

    await vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(user.address, receiver.address, [period])
    await vault.connect(ctx.blocklister).removeFromBlocklist(user.address)

    expect(await vault.balanceOf(user.address)).to.equal(0n)
    await ctx.utils.mintAndApprove(FRESH_PRINCIPAL, user)
    await vault.connect(user).deposit(FRESH_PRINCIPAL, user.address)
    expect(await request(vault, user, FRESH_PRINCIPAL)).to.equal(period)
    expect(await vault.withdrawalsOf(period, user.address)).to.equal(FRESH_PRINCIPAL)
    expect(await vault.isWithdrawClaimed(period, user.address)).to.equal(true)

    await time.increase(ctx.config.period_configuration_duration * 2)
    await vault.connect(receiver).claimWithdraw(period)
    await expect(vault.connect(user).claimWithdraw(period))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(period)

    await vault.connect(ctx.blocklister).addToBlocklist(user.address)
    await expect(vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(user.address, receiver.address, [period]))
      .to.be.revertedWithCustomError(vault, 'AlreadyClaimedPeriod')
      .withArgs(period)

    const finalState = {
      userWithdrawalAssets: (await vault.withdrawalsOf(period, user.address)).toString(),
      pendingWithdrawAssets: (await vault.pendingWithdrawAssets()).toString(),
      rawUnderlyingInVault: (await asset.balanceOf(await vault.getAddress())).toString(),
      activeTotalAssets: (await vault.totalAssets()).toString()
    }
    console.log('FIRELIGHT_RESCUE_VALIDATOR_FRESH_FUNDS=' + JSON.stringify(finalState))

    expect(await vault.withdrawalsOf(period, user.address)).to.equal(FRESH_PRINCIPAL)
    expect(await vault.pendingWithdrawAssets()).to.equal(FRESH_PRINCIPAL)
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(FRESH_PRINCIPAL)
    expect(await vault.totalAssets()).to.equal(0n)
  })
})
```

3. Run both tests:

   ```bash
   npx hardhat test test/poc/rescue-open-bucket.poc.js test/poc/rescue-fresh-funds.poc.js
   ```
4. Observe that the post-rescue bucket contains a funded later withdrawal while its old claim marker is already true. After maturity, the test asserts that normal claim reverts; it then re-blocklists the user and proves repeat rescue also reverts. The fresh-funds comparator reports 1,000 units still pending with 1,000 raw units in the vault and zero active assets. The matched no-rescue control pays all 3,000 queued units and clears pending accounting.

   See `step-04-rescue-marker-freezes-fresh-funds.png` for the command and terminal result.

## Proof Of Impact

```
FIRELIGHT_RESCUE_POC_POISONED={"userWithdrawalAssets":"1000000000","userClaimMarker":true,"pendingWithdrawAssets":"3000000000"}
FIRELIGHT_RESCUE_POC_FINAL={"userWithdrawalAssets":"1000000000","userClaimMarker":true,"pendingWithdrawAssets":"1000000000","rawUnderlyingInVault":"4000000000","activeTotalAssets":"3000000000"}
FIRELIGHT_RESCUE_POC_CONTROL={"paidAssets":"3000000000","pendingWithdrawAssets":"0"}
FIRELIGHT_RESCUE_VALIDATOR_FRESH_FUNDS={"userWithdrawalAssets":"1000000000","pendingWithdrawAssets":"1000000000","rawUnderlyingInVault":"1000000000","activeTotalAssets":"0"}
3 passing
```

The relevant code path is:

```solidity
// FirelightVault.sol
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

// rescueWithdrawFromBlocklisted()
isWithdrawClaimed[periods[i]][from] = true;
```

The rescue transfers only the shares present at rescue time, but its boolean marker continues authorizing rejection of principal added later to the same key.
