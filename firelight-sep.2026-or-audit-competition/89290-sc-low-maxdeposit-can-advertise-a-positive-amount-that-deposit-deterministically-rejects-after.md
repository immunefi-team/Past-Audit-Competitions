# 89290 sc low maxdeposit can advertise a positive amount that deposit deterministically rejects after legitimate rewards

**Submitted on Aug 18th 2026 at 12:52:53 UTC by @CarlosMB for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89290
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`FirelightVault.maxDeposit()` returns the remaining asset capacity:

```solidity
return depositLimit - totalAssets();
```

but `deposit()` has an additional executability constraint:

```solidity
if (shares == 0) revert InvalidAmount();
```

Legitimate rewards increase `totalAssets()` without increasing `totalSupply()`. Near the deposit limit, this can leave a positive remaining capacity that is too small to mint one share.

As a result, `maxDeposit(receiver)` can return a positive value while an immediate `deposit(maxDeposit(receiver), receiver)` reverts with `InvalidAmount`, with no intervening state change.

ERC-4626 requires `maxDeposit()` not to return more assets than `deposit()` would currently accept without reverting.

## Root Cause

`maxDeposit()` only checks the deposit limit and protocol-level guards:

```solidity
function maxDeposit(address receiver) public view override returns (uint256 amount) {
    uint256 assets = totalAssets();
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) {
        return 0;
    } else {
        return depositLimit - assets;
    }
}
```

However, `deposit()` also rejects deposits that round to zero shares:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    assets,
    true,
    Math.Rounding.Floor
);

if (shares == 0) revert InvalidAmount();
```

`VaultRewardDistributor` can naturally increase the share price because it transfers vault assets directly into `FirelightVault` without minting shares.

## Impact

**Low — Contract fails to deliver promised returns without loss of value.**

Integrations using `maxDeposit()` to construct the largest currently valid ERC-4626 deposit can receive a protocol-provided positive value that is guaranteed to revert.

No donation, privileged misuse, or exchange-rate manipulation is required.

## Proof of Concept

## Proof of Concept

Create:

```
test/FirelightVault/maxDepositZeroShare.e2e.js
```

with:

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')

describe('Low: maxDeposit advertises a deposit that deterministically reverts', function () {
  async function fixture () {
    const LIMIT = ethers.parseUnits('20000', 6)
    const INITIAL_DEPOSIT = ethers.parseUnits('19999', 6)
    const REWARD = 999_999n // 0.999999 vault asset

    const ctx = await deployVault({
      initial_deposit_limit: LIMIT.toString()
    })

    const user = ctx.users[0]
    const distributor = (await ethers.getSigners())[9]

    const Rewarder = await ethers.getContractFactory('VaultRewardDistributor')
    const rewarder = await upgrades.deployProxy(Rewarder, [
      ctx.firelight_vault.target,
      ctx.deployer.address,
      distributor.address,
      ethers.ZeroAddress
    ])

    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.CHECKPOINT_ROLE(), rewarder.target)

    // Normal user deposit: 19,999 assets into a vault capped at 20,000.
    await ctx.utils.mintAndApprove(INITIAL_DEPOSIT, user)
    await ctx.firelight_vault
      .connect(user)
      .deposit(INITIAL_DEPOSIT, user.address)

    // Deploy a mock premium token used only as legitimate distribution metadata.
    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const premiumToken = await MockERC20.deploy('Premium Token', 'PREM', 6)

    // Fund the authorized distributor with 0.999999 vault asset and approve the rewarder.
    await ctx.token_contract.mintTo(distributor.address, REWARD)
    await ctx.token_contract.connect(distributor).approve(rewarder.target, REWARD)

    return {
      ...ctx,
      user,
      distributor,
      rewarder,
      premiumToken,
      LIMIT,
      INITIAL_DEPOSIT,
      REWARD
    }
  }

  it('returns maxDeposit == 1 although deposit(1) reverts with InvalidAmount', async function () {
    const ctx = await loadFixture(fixture)

    // Legitimate protocol reward increases totalAssets without minting shares.
    await ctx.rewarder.connect(ctx.distributor).distributeRewards(
      ctx.REWARD,
      ctx.premiumToken.target,
      1n,
      await time.latest()
    )

    expect(await ctx.firelight_vault.totalSupply())
      .to.equal(ctx.INITIAL_DEPOSIT)

    expect(await ctx.firelight_vault.totalAssets())
      .to.equal(ctx.LIMIT - 1n)

    // maxDeposit advertises the final raw asset unit as depositable.
    const advertised = await ctx.firelight_vault.maxDeposit(ctx.user.address)
    expect(advertised).to.equal(1n)

    // At the same state, that amount rounds to zero shares.
    expect(await ctx.firelight_vault.previewDeposit(advertised))
      .to.equal(0n)

    // Give the user the advertised asset and sufficient allowance.
    await ctx.token_contract.mintTo(ctx.user.address, advertised)
    await ctx.token_contract
      .connect(ctx.user)
      .approve(ctx.firelight_vault.target, advertised)

    // No vault state changes between maxDeposit() and deposit().
    await expect(
      ctx.firelight_vault
        .connect(ctx.user)
        .deposit(advertised, ctx.user.address)
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'InvalidAmount')

    // The vault still advertises the same positive maximum afterwards.
    expect(await ctx.firelight_vault.maxDeposit(ctx.user.address))
      .to.equal(1n)
  })
})
```

Run:

```bash
npx hardhat test test/FirelightVault/maxDepositZeroShare.e2e.js
```

Expected security-relevant result:

```
totalSupply                 = 19,999,000,000
totalAssets                 = 19,999,999,999
depositLimit                = 20,000,000,000

maxDeposit(user)            = 1
previewDeposit(1)           = 0
deposit(1, user)            -> InvalidAmount
```

## Recommended Fix

Make `maxDeposit()` account for the same zero-share condition enforced by `deposit()`:

```solidity
uint256 remaining = depositLimit - totalAssets();

if (remaining == 0 || previewDeposit(remaining) == 0) {
    return 0;
}

return remaining;
```

This ensures the advertised maximum is actually executable under the current vault state.
