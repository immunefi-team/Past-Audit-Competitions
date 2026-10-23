# 88292 sc low maxmint can overstate remaining mint capacity and return an amount that mint immediately rejects

**Submitted on Aug 12th 2026 at 16:36:04 UTC by @CarlosMB for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88292
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`FirelightVault.maxMint()` derives the remaining mint capacity by converting the **absolute** `depositLimit` to shares and then subtracting the current `totalSupply()`:

```solidity
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 shares = totalSupply();
    uint256 sharesLimit = convertToShares(depositLimit);
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || shares > sharesLimit) {
        return 0;
    } else {
        return sharesLimit - shares;
    }
}
```

This is not equivalent to converting the **remaining asset capacity** once the vault exchange rate is not exactly 1:1.

Firelight's ERC-4626 conversions include the virtual terms used by OpenZeppelin (`+1` asset and `+1` virtual share with the default decimals offset). Therefore:

```
convertToShares(depositLimit) - totalSupply()
```

can be larger than:

```
convertToShares(depositLimit - totalAssets())
```

As a result, `maxMint(receiver)` can return a share amount whose `previewMint()` asset requirement exceeds the remaining deposit capacity. A fully funded and approved user calling:

```solidity
mint(maxMint(receiver), receiver)
```

then immediately reverts with `DepositLimitExceeded`.

This is reachable through normal protocol operation. A legitimate reward distribution increases `totalAssets()` without minting vault shares, causing `totalAssets() != totalSupply()`.

## Vulnerability Details

The deposit limit is enforced in `_depositFunds()` against the post-mint asset balance:

```solidity
function _depositFunds(
    address caller,
    address receiver,
    uint256 assets,
    uint256 shares,
    uint256 _totalSupply,
    uint256 _totalAssets
) private {
    _totalSupply += shares;
    _totalAssets += assets;

    if (_totalAssets > depositLimit) revert DepositLimitExceeded();

    _deposit(caller, receiver, assets, shares);

    _logTrace(receiver, balanceOf(receiver), _totalSupply, _totalAssets, true);
}
```

Therefore the actual mint capacity is determined by the remaining asset room:

```
remainingAssets = depositLimit - totalAssets()
```

and the maximum executable share amount is:

```
convertToShares(remainingAssets)
```

However, `maxMint()` currently computes:

```
convertToShares(depositLimit) - totalSupply()
```

These expressions are not equivalent because the conversion is not a simple linear `assets * totalSupply / totalAssets` mapping. Firelight inherits the OpenZeppelin virtual-asset/share terms.

For the default decimals offset, the relevant conversion is effectively:

```
shares = floor(
    assets * (totalSupply + 1)
           / (totalAssets + 1)
)
```

Subtracting `totalSupply()` **after** converting the absolute limit therefore does not correctly represent the number of shares that can be minted using only the remaining asset capacity.

## Concrete Example

Using the repository's 6-decimal asset:

```
depositLimit = 24,000,000 raw assets = 24 tokens
totalSupply  = 10,000,000 raw shares = 10 shares
totalAssets  = 11,000,000 raw assets = 11 tokens
```

The `totalAssets > totalSupply` state is reached normally by depositing 10 tokens and then forwarding 1 token of legitimate rewards through `VaultRewardDistributor`, which transfers vault assets directly to the vault without minting shares.

The current implementation computes:

```
convertToShares(24,000,000)
= floor(
    24,000,000 * 10,000,001
               / 11,000,001
  )
= 21,818,182
```

and therefore:

```
maxMint()
= 21,818,182 - 10,000,000
= 11,818,182 shares
```

However, minting those shares requires rounding the asset requirement up:

```
previewMint(11,818,182)
= ceil(
    11,818,182 * 11,000,001
               / 10,000,001
  )
= 13,000,001 assets
```

The vault only has 13,000,000 raw assets of remaining capacity:

```
depositLimit - totalAssets
= 24,000,000 - 11,000,000
= 13,000,000
```

Therefore:

```
11,000,000 + 13,000,001
= 24,000,001
> 24,000,000
```

and:

```solidity
mint(maxMint(user), user)
```

reverts with:

```
DepositLimitExceeded
```

The actual maximum executable amount is:

```
convertToShares(13,000,000)
= 11,818,181 shares
```

Thus `maxMint()` overstates the protocol's real mint capacity by one raw share in this reproducible state.

## Relation to the Published ERC-4626 Max-Method Fix

That public change contains the same `maxMint()` implementation:

```solidity
uint256 shares = totalSupply();
uint256 sharesLimit = convertToShares(depositLimit);
...
return sharesLimit - shares;
```

Its tests compare `maxMint()` against the shares corresponding to `maxDeposit()`, but only exercise states where the exchange rate remains 1:1. In those states the incorrect formula happens to produce the expected result.

The audit snapshot contains the same implementation, but a normal reward distribution moves the vault away from 1:1 and exposes the remaining inconsistency.

This report therefore demonstrates an incomplete remediation / edge case in the published max-method fix rather than merely restating the previously fixed absence of ERC-4626 max methods.

## Impact

`maxMint()` is intended to report the maximum number of shares that the vault can currently accept for a receiver.

In the demonstrated state, it returns a value that the vault itself rejects due to its own `depositLimit`, even when the caller:

* is not blocklisted;
* the vault is not paused;
* there is no active incident;
* the caller has sufficient underlying assets; and
* the caller has sufficient allowance.

Integrations or users that rely on `maxMint()` to construct the largest valid mint transaction can therefore receive a protocol-provided value that deterministically reverts.

There is no loss of funds, so **Low — Contract fails to deliver promised returns without loss of value** is the appropriate impact.

## Proof of Concept

## Proof of Concept

Create:

```
test/low_maxMint_incorrect.js
```

with:

```javascript
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { deployVault } = require('./setup/fixtures.js')

describe('Low: maxMint can return an amount that mint immediately rejects', function () {
  const DECIMALS = 6
  const DEPOSIT_LIMIT = ethers.parseUnits('24', DECIMALS)
  const INITIAL_DEPOSIT = ethers.parseUnits('10', DECIMALS)
  const REWARD = ethers.parseUnits('1', DECIMALS)

  it('overstates mint capacity after a legitimate reward changes the exchange rate', async function () {
    const {
      token_contract: token,
      firelight_vault: vault,
      deployer,
      users,
      utils
    } = await loadFixture(
      deployVault.bind(null, { initial_deposit_limit: DEPOSIT_LIMIT })
    )

    const [user, distributor] = users

    // Start from 10 assets / 10 shares.
    await utils.mintAndApprove(INITIAL_DEPOSIT, user)
    await vault.connect(user).deposit(INITIAL_DEPOSIT, user.address)

    expect(await vault.totalSupply()).to.equal(INITIAL_DEPOSIT)
    expect(await vault.totalAssets()).to.equal(INITIAL_DEPOSIT)

    // Deploy the protocol's legitimate reward-forwarding contract.
    const RewardDistributor = await ethers.getContractFactory(
      'VaultRewardDistributor'
    )

    const rewardDistributor = await upgrades.deployProxy(
      RewardDistributor,
      [
        await vault.getAddress(),
        deployer.address,
        distributor.address,
        ethers.ZeroAddress
      ]
    )

    // Reward distribution atomically checkpoints the vault, so the distributor
    // needs the vault's CHECKPOINT_ROLE as intended by the implementation.
    await vault.connect(deployer).grantRole(
      await vault.CHECKPOINT_ROLE(),
      await rewardDistributor.getAddress()
    )

    // Forward 1 token of legitimate rewards into the vault.
    await token.mintTo(distributor.address, REWARD)
    await token.connect(distributor).approve(
      await rewardDistributor.getAddress(),
      REWARD
    )

    await rewardDistributor.connect(distributor).distributeIncentive(
      REWARD,
      ethers.id('maxMint-capacity-test')
    )

    // Rewards increase assets without minting shares.
    expect(await vault.totalSupply()).to.equal(
      ethers.parseUnits('10', DECIMALS)
    )
    expect(await vault.totalAssets()).to.equal(
      ethers.parseUnits('11', DECIMALS)
    )

    const reportedMaxMint = await vault.maxMint(user.address)

    // Current implementation overstates capacity by one raw share.
    expect(reportedMaxMint).to.equal(11_818_182n)

    const assetsRequired = await vault.previewMint(reportedMaxMint)
    expect(assetsRequired).to.equal(13_000_001n)

    const remainingCapacity =
      DEPOSIT_LIMIT - (await vault.totalAssets())

    expect(remainingCapacity).to.equal(13_000_000n)
    expect(assetsRequired).to.be.gt(remainingCapacity)

    // Ensure balance/allowance are not the reason for the revert.
    const mintFunding = ethers.parseUnits('14', DECIMALS)
    await utils.mintAndApprove(mintFunding, user)

    // Calling mint(maxMint()) fails solely because maxMint overstated
    // the protocol's own remaining deposit-limit capacity.
    await expect(
      vault.connect(user).mint(reportedMaxMint, user.address)
    ).to.be.revertedWithCustomError(vault, 'DepositLimitExceeded')

    // The actual maximum executable amount is one raw share lower.
    const actualMaxMint = await vault.convertToShares(remainingCapacity)
    expect(actualMaxMint).to.equal(11_818_181n)

    expect(
      await vault.previewMint(actualMaxMint)
    ).to.be.lte(remainingCapacity)

    await expect(
      vault.connect(user).mint(actualMaxMint, user.address)
    ).to.emit(vault, 'Deposit')
  })
})
```

Run:

```bash
npx hardhat test test/low_maxMint_incorrect.js
```

Expected security-relevant result:

```
maxMint(user)               = 11,818,182
previewMint(maxMint(user))  = 13,000,001
remaining asset capacity    = 13,000,000
mint(maxMint(user), user)   -> DepositLimitExceeded

actual executable maxMint   = 11,818,181
mint(actualMaxMint, user)   -> succeeds
```

## Root Cause

`maxMint()` applies the asset-to-share conversion to the absolute deposit limit and only then subtracts the existing share supply:

```solidity
convertToShares(depositLimit) - totalSupply()
```

This assumes a linear conversion where subtracting the existing position after conversion is equivalent to converting only the remaining capacity.

That assumption is false once `totalAssets() != totalSupply()` because ERC-4626 conversion includes virtual assets/shares and integer rounding.

## Recommended Mitigation

Derive mint capacity from the remaining asset capacity:

```solidity
function maxMint(address receiver) public view override returns (uint256) {
    return convertToShares(maxDeposit(receiver));
}
```

or equivalently, after applying the same blocklist/pause/incident checks:

```solidity
uint256 assets = totalAssets();

if (assets > depositLimit) {
    return 0;
}

return convertToShares(depositLimit - assets);
```

This calculates the maximum shares whose corresponding asset requirement fits within the vault's actual remaining deposit capacity.
