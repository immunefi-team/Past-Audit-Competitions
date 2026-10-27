# 89486 sc low incorrect rounding in maxmint causes mint maxmint to revert with depositlimitexceeded violating erc 4626 spec

**Submitted on Aug 19th 2026 at 15:45:58 UTC by @omsanto for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89486
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

FirelightVault.maxMint() uses Floor rounding to compute the maximum mintable shares, but mint() uses Ceil rounding to convert those shares back to assets. After yield distribution via VaultRewardDistributor (normal protocol operation), this rounding mismatch causes mint(maxMint()) to revert with DepositLimitExceeded, violating the ERC-4626 guarantee that mint() MUST NOT revert when called with the value returned by maxMint().

## Vulnerability Details

```
maxMint() calls convertToShares(depositLimit) which uses Math.Rounding.Floor:

    function convertToShares(uint256 assets) public view virtual returns (uint256) {
        return _convertToShares(assets, Math.Rounding.Floor);
    }


    function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
        return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
    }


mint() converts those shares back to assets using Math.Rounding.Ceil inside _previewTotals():

    (uint256 assets, ...) = _previewTotals(shares, false, Math.Rounding.Ceil);


_depositFunds() then checks:

    if (_totalAssets > depositLimit) revert DepositLimitExceeded();


The problem is the rounding mismatch. maxMint() computes the share limit using Floor. mint() converts those same shares back to assets using Ceil. Ceil always produces a result equal to or greater than Floor. This means mint() can require 1 wei more assets than maxMint() assumed was safe, pushing totalAssets 1 wei above depositLimit and causing the revert.

This bug triggers whenever two normal ongoing protocol conditions exist simultaneously:

Condition 1 — totalAssets > totalSupply (share price above 1:1)
This happens every time VaultRewardDistributor.distributeRewards() is called, which is the normal mechanism for distributing yield to stakers. After every yield distribution, share price rises above 1:1. This is expected and intended protocol behavior.

Condition 2 — Vault is near its depositLimit
Admins routinely set depositLimit close to current TVL as a risk management measure. As the vault grows, the admin periodically raises the limit. At any point where the limit is set close to current totalAssets, this condition is met.

Both conditions are normal ongoing states of the protocol. They are not edge cases or attack setups, they describe the vault during routine operation. Whenever both conditions exist at the same time, any user calling the standard ERC-4626 pattern of mint(vault.maxMint(receiver)) will receive an unexpected revert.

Concrete example from passing Hardhat test:

- totalAssets = 1100 FXRP (1000 deposited by stakers + 100 yield distributed by VaultRewardDistributor)
- totalSupply = 1000 shares
- depositLimit = 1101 FXRP (only 1 FXRP of capacity remaining)

At this point the share price is 1.1 FXRP per share (1100/1000).
So with only 1 FXRP of remaining capacity, a user can mint approximately
0.909 shares (1 / 1.1 = 0.909). With 6 decimal precision this is 909091 raw share units.

maxMint() computes this using Floor rounding and returns 909091 shares advertising this as the safe maximum.

mint(909091) converts those shares back to FXRP using Ceil rounding:
assets needed = ceil(909091 * 1100000001 / 1000000001) = 1000001 raw units (1.000001 FXRP)

_depositFunds then checks:
totalAssets + assets = 1100000000 + 1000001 = 1101000001
depositLimit = 1101000000
1101000001 > 1101000000 → REVERT DepositLimitExceeded

The Floor rounding in maxMint() assumed the user needed 1000000 assets (1 FXRP exactly).

The Ceil rounding in mint() actually charged 1000001 assets (1.000001 FXRP).
That 1 wei difference causes the revert.
```

## Impact Details

No funds are lost, the transaction reverts. However the vault breaks the ERC-4626 guarantee that mint() MUST NOT revert when called with maxMint() value. Any protocol or user using the standard ERC-4626 integration pattern of mint(vault.maxMint(receiver)) will get an unexpected revert during normal protocol operation after yield distribution. The impact matches "Contract fails to deliver promised returns, but doesn't lose value."

## References

ERC-4626 specification: <https://eips.ethereum.org/EIPS/eip-4626>

Relevant section : maxMint(): "MUST return the maximum amount of shares mint would allow to be deposited to receiver and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted."

FirelightVault intentionally deviates from ERC-4626 withdrawal semantics (documented), but the maxMint() violation is not listed as intentional.

## Proof of Concept

## Proof of Concept

Step-by-step reproduction:

```
Step 1: Deploy FirelightVault with depositLimit = 1101 FXRP

Step 2: User1 deposits 1000 FXRP
        totalAssets = 1000 FXRP, totalSupply = 1000 shares
        Share price = 1 FXRP per share (1:1 ratio)

Step 3: VaultRewardDistributor distributes 100 FXRP yield to the vault
        totalAssets = 1100 FXRP, totalSupply = 1000 shares
        Share price = 1.1 FXRP per share (normal yield accrual)
        Remaining capacity = 1101 - 1100 = 1 FXRP

Step 4: User2 calls vault.maxMint(user2.address)
        With 1 FXRP remaining and share price 1.1, User2 can mint ~0.909 shares
        maxMint() computes this using Floor rounding
        Returns: 909091 shares (0.909091 shares with 6 decimal precision)
        ERC-4626 spec guarantees mint() MUST NOT revert with this value

Step 5: User2 calls vault.mint(909091, user2.address)
        mint() converts 909091 shares to assets using Ceil rounding
        assets needed = ceil(909091 * 1100000001 / 1000000001) = 1000001 (1.000001 FXRP)
        Floor assumed 1.000000 FXRP, Ceil charges 1.000001 FXRP — 1 wei difference
        _depositFunds checks: 1100000000 + 1000001 = 1101000001 > 1101000000
        REVERT: DepositLimitExceeded

Hardhat test:   test/poc/maxMintViolation.js

const { ethers } = require('hardhat')
const { expect } = require('chai')
const { deployVault } = require('../setup/fixtures.js')

describe("PoC: maxMint() ERC-4626 violation in FirelightVault", function () {
    it("mint(maxMint()) reverts with DepositLimitExceeded", async function () {
        const result = await deployVault({
            initial_deposit_limit: '1101000000' // 1101 FXRP with 6 decimals
        })
        const vault = result.firelight_vault
        const token = result.token_contract
        const utils = result.utils
        const [deployer, , , , , , user1, user2] = await ethers.getSigners()

        // Step 2: User1 deposits 1000 FXRP — share price starts at 1:1
        const initialDeposit = ethers.parseUnits("1000", 6)
        await utils.mintAndApprove(initialDeposit, user1)
        await vault.connect(user1).deposit(initialDeposit, user1.address)
        console.log("totalAssets after deposit:", ethers.formatUnits(await vault.totalAssets(), 6))
        console.log("totalSupply after deposit:", ethers.formatUnits(await vault.totalSupply(), 6))

        // Step 3: VaultRewardDistributor distributes 100 FXRP yield
        // This is normal protocol operation — share price goes above 1:1
        const yieldAmount = ethers.parseUnits("100", 6)
        await token.mintTo(deployer.address, yieldAmount)
        await token.transfer(await vault.getAddress(), yieldAmount)
        console.log("totalAssets after yield:", ethers.formatUnits(await vault.totalAssets(), 6))
        console.log("totalSupply after yield:", ethers.formatUnits(await vault.totalSupply(), 6))
        console.log("Share price:", Number(await vault.totalAssets()) / Number(await vault.totalSupply()), "FXRP per share")

        // Step 4: maxMint() returns safe maximum using Floor rounding
        const maxMintShares = await vault.maxMint(user2.address)
        console.log("maxMint() returns:", maxMintShares.toString(), "shares")
        console.log("ERC-4626 spec: mint(maxMint()) MUST NOT revert")

        // Step 5: mint(maxMint()) reverts — Ceil rounding needs 1 wei more than Floor assumed
        const previewAssets = await vault.previewMint(maxMintShares)
        console.log("previewMint says mint needs:", ethers.formatUnits(previewAssets, 6), "FXRP")
        console.log("totalAssets after mint would be:", ethers.formatUnits(await vault.totalAssets() + previewAssets, 6))
        console.log("depositLimit:", ethers.formatUnits(await vault.depositLimit(), 6))
        console.log("Exceeds limit:", await vault.totalAssets() + previewAssets > await vault.depositLimit())

        await utils.mintAndApprove(await vault.depositLimit(), user2)
        await expect(
            vault.connect(user2).mint(maxMintShares, user2.address)
        ).to.be.revertedWithCustomError(vault, "DepositLimitExceeded")

        console.log("=== BUG CONFIRMED ===")
        console.log("maxMint() returned", maxMintShares.toString(), "shares as safe maximum")
        console.log("mint(maxMint()) REVERTS with DepositLimitExceeded")
        console.log("ERC-4626 violation: mint() MUST NOT revert when called with maxMint() value")
    })
})

Test output:  npx hardhat test test/poc/maxMintViolation.js

totalAssets after deposit: 1000.0
  totalSupply after deposit: 1000.0
  totalAssets after yield: 1100.0
  totalSupply after yield: 1000.0
  Share price: 1.1 FXRP per share
  maxMint() returns: 909091 shares
  ERC-4626 spec: mint(maxMint()) MUST NOT revert
  previewMint says mint needs: 1.000001 FXRP
  totalAssets after mint would be: 1101.000001
  depositLimit: 1101.0
  Exceeds limit: true
  === BUG CONFIRMED ===
  maxMint() returned 909091 shares as safe maximum
  mint(maxMint()) REVERTS with DepositLimitExceeded
  ERC-4626 violation: mint() MUST NOT revert when called with maxMint() value
  ✔ mint(maxMint()) reverts with DepositLimitExceeded (549ms)
  1 passing (552ms)
```
