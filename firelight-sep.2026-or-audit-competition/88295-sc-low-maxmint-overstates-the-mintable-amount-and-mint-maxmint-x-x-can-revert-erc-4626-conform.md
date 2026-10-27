# 88295 sc low maxmint overstates the mintable amount and mint maxmint x x can revert erc 4626 conformance break near the deposit cap

**Submitted on Aug 12th 2026 at 16:43:51 UTC by @Oxb4b for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88295
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`FirelightVault.sol` exposes ERC-4626-style `maxDeposit()` and `maxMint()` bounds, but `maxMint()` is not round-trip safe with `mint()`. In certain valid vault states, `maxMint(receiver)` returns a share amount that `mint(maxMint(receiver), receiver)` rejects with `DepositLimitExceeded()`. This is a spec conformance issue and an integration nuisance, not a fund-loss bug.

## Vulnerability Details

`maxMint()` is implemented by converting the asset deposit limit to shares with floor rounding and subtracting current supply:

* `sharesLimit = convertToShares(depositLimit)`
* return `sharesLimit - totalSupply()`

`mint()` uses a different path:

* it converts requested shares back to assets with **ceil** rounding via `_previewTotals(..., Math.Rounding.Ceil)`
* `_depositFunds()` then enforces `totalAssets() + assets <= depositLimit`

Because the bound is computed with floor rounding in one direction and mint execution uses ceil rounding in the opposite direction, the result can be off by 1 asset unit. That is enough to make `mint(maxMint(x), x)` revert even though `maxMint()` claimed the call was valid.

A concrete counterexample exists with `_decimalsOffset() == 0`:

* `totalAssets() = 4`
* `totalSupply() = 3`
* `depositLimit = 10`

Then:

* `maxMint()` computes `sharesLimit = floor(10 × (3+1) / (4+1)) = floor(40/5) = 8`, so it returns `5`
* `mint(5)` computes `assets = ceil(5 × (4+1) / (3+1)) = ceil(25/4) = 7`
* `_depositFunds()` sees `4 + 7 = 11 > 10` and reverts with `DepositLimitExceeded()`

A related dust-edge case exists for `maxDeposit()`: it returns the remaining asset headroom, but if that headroom floors to zero shares, `deposit(maxDeposit(...))` reverts on the intentional zero-share guard with `InvalidAmount()`.

ERC‑4626 requires that `maxMint/maxDeposit` “MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary).” In the current implementation, `mint(maxMint(receiver), receiver)` and `deposit(maxDeposit(receiver), receiver)` can revert due to rounding‑induced overstatement, violating this requirement.

## Impact and Likelihood Details

This does not steal funds or corrupt accounting. The vault fails closed before taking assets, so the impact is limited to:

* broken ERC-4626 conformance
* failed integrations for routers, zappers, aggregators, and vault-of-vaults that rely on the canonical `mint(maxMint(...))` / `deposit(maxDeposit(...))` pattern
* wasted gas on reverted transactions
* This creates a griefing surface: repeated reverts and wasted gas can occur for integrators and users that rely on mint(maxMint(...)) / deposit(maxDeposit(...)) when the vault is near its cap, even without any attacker activity.

Likelihood is **medium as a nuisance**. The state is realistic because the vault’s asset/share ratio changes through ordinary deposits, rewards, and payout-related asset movements. It is not a governance bug and does not require malicious use of privileged roles.

## References

* `FirelightVault.sol` `maxDeposit()` and `maxMint()` implementation
* `FirelightVault.sol` `mint()` and `_depositFunds()` implementation
* `FirelightVault.sol` `deposit()` zero-share guard
* `scope.md` in-scope categories for Low / Medium griefing / theft of gas
* ERC-4626 standard requirements for `maxDeposit` / `maxMint` not to overstate the safe amount

## Proof of Concept

## Proof of Concept

add the below testcase to the existing test file `test/deposit_and_withdraw.js` and run

```js
describe('[LOW-05] maxMint()/maxDeposit() overstate what mint()/deposit() accept', function () {
  const DECIMALS = 6,
    ONE_TOKEN = ethers.parseUnits('1', DECIMALS),
    DEPOSIT_LIMIT = ethers.parseUnits('50000', DECIMALS),
    SEED_DEPOSIT = ethers.parseUnits('1000', DECIMALS),
    SWEEP_TOKENS = 120

  const stakedVaultFixture = async () => {
    const ctx = await deployVault({ decimals: DECIMALS, initial_deposit_limit: DEPOSIT_LIMIT })
    const { firelight_vault, token_contract, deployer, users, utils } = ctx

    await utils.mintAndApprove(SEED_DEPOSIT, users[0])
    await firelight_vault.connect(users[0]).deposit(SEED_DEPOSIT, users[0].address)

    const Distributor = await ethers.getContractFactory('VaultRewardDistributor')
    const distributor = await upgrades.deployProxy(Distributor, [
      firelight_vault.target, deployer.address, deployer.address, ethers.ZeroAddress
    ])
    await firelight_vault.connect(deployer)
      .grantRole(await firelight_vault.CHECKPOINT_ROLE(), distributor.target)

    await token_contract.mintTo(deployer.address, ONE_TOKEN * BigInt(SWEEP_TOKENS))
    await token_contract.connect(deployer).approve(distributor.target, ONE_TOKEN * BigInt(SWEEP_TOKENS))
    await utils.mintAndApprove(DEPOSIT_LIMIT, users[1])

    ctx.distributor = distributor
    ctx.accruePremium = async (amount) => distributor.connect(deployer).distributeRewards(
      amount, token_contract.target, amount, (await time.latest())
    )
    return ctx
  }

  it('mint(maxMint(u), u) reverts at states discovered by ordinary premium accrual', async () => {
    const { firelight_vault, users, accruePremium } = await loadFixture(stakedVaultFixture)
    const integrator = users[1]
    const violations = []

    for (let i = 1; i <= SWEEP_TOKENS; i++) {
      await accruePremium(ONE_TOKEN)

      const total_assets = await firelight_vault.totalAssets(),
        max_mint = await firelight_vault.maxMint(integrator.address),
        max_deposit = await firelight_vault.maxDeposit(integrator.address),
        mint_cost = await firelight_vault.previewMint(max_mint)

      if (total_assets + mint_cost <= DEPOSIT_LIMIT) {
        expect(max_mint).to.equal(await firelight_vault.previewDeposit(max_deposit))
        continue
      }

      violations.push(i)

      await expect(firelight_vault.connect(integrator).mint(max_mint, integrator.address))
        .to.be.revertedWithCustomError(firelight_vault, 'DepositLimitExceeded')

      expect(max_mint).to.equal((await firelight_vault.previewDeposit(max_deposit)) + 1n)

      const snapshot = await takeSnapshot()
      await expect(firelight_vault.connect(integrator).mint(max_mint - 1n, integrator.address))
        .to.emit(firelight_vault, 'Deposit')
      await snapshot.restore()
    }

    expect(violations.length).to.be.greaterThan(0)
    expect(violations[0]).to.equal(26)
  })

  it('reproduces the first violating state with exact figures', async () => {
    const { firelight_vault, users, accruePremium } = await loadFixture(stakedVaultFixture)
    const integrator = users[1]

    await accruePremium(ONE_TOKEN * 26n)

    const TOTAL_SUPPLY = 1_000_000_000n,
      TOTAL_ASSETS = 1_026_000_000n
    expect(await firelight_vault.totalSupply()).to.equal(TOTAL_SUPPLY)
    expect(await firelight_vault.totalAssets()).to.equal(TOTAL_ASSETS)
    expect(await firelight_vault.depositLimit()).to.equal(DEPOSIT_LIMIT)

    const max_mint = await firelight_vault.maxMint(integrator.address)
    expect(max_mint).to.equal(47_732_943_471n)

    const assets_needed = await firelight_vault.previewMint(max_mint)
    expect(assets_needed).to.equal(48_974_000_001n)
    expect(TOTAL_ASSETS + assets_needed).to.equal(DEPOSIT_LIMIT + 1n)

    const max_deposit = await firelight_vault.maxDeposit(integrator.address)
    expect(max_deposit).to.equal(48_974_000_000n)
    expect(TOTAL_ASSETS + max_deposit).to.equal(DEPOSIT_LIMIT)
    expect(await firelight_vault.previewDeposit(max_deposit)).to.equal(max_mint - 1n)

    await expect(firelight_vault.connect(integrator).mint(max_mint, integrator.address))
      .to.be.revertedWithCustomError(firelight_vault, 'DepositLimitExceeded')

    await expect(firelight_vault.connect(integrator).mint(max_mint - 1n, integrator.address))
      .to.emit(firelight_vault, 'Deposit')
      .withArgs(integrator.address, integrator.address, 48_973_999_999n, max_mint - 1n)

    expect(await firelight_vault.balanceOf(integrator.address)).to.equal(max_mint - 1n)
    expect(await firelight_vault.totalAssets()).to.equal(DEPOSIT_LIMIT - 1n)
  })

  it('maxDeposit(receiver) returns dust that deposit() then rejects as zero shares', async () => {
    const { firelight_vault, users, accruePremium } = await loadFixture(stakedVaultFixture)
    const integrator = users[1]

    await accruePremium(ONE_TOKEN * 26n)

    const fill_shares = await firelight_vault.previewDeposit(await firelight_vault.maxDeposit(integrator.address))
    await firelight_vault.connect(integrator).mint(fill_shares, integrator.address)
    expect(await firelight_vault.totalAssets()).to.equal(DEPOSIT_LIMIT - 1n)
    expect(await firelight_vault.totalSupply()).to.equal(48_732_943_470n)

    const max_deposit = await firelight_vault.maxDeposit(integrator.address)
    expect(max_deposit).to.equal(1n)
    expect(await firelight_vault.previewDeposit(max_deposit)).to.equal(0n)
    await expect(firelight_vault.connect(integrator).deposit(max_deposit, integrator.address))
      .to.be.revertedWithCustomError(firelight_vault, 'InvalidAmount')

    const max_mint_after = await firelight_vault.maxMint(integrator.address)
    expect(max_mint_after).to.equal(1n)
    expect(await firelight_vault.previewMint(max_mint_after)).to.equal(2n)
    await expect(firelight_vault.connect(integrator).mint(max_mint_after, integrator.address))
      .to.be.revertedWithCustomError(firelight_vault, 'DepositLimitExceeded')
  })
})
```

### Scenario 1: `mint(maxMint(...))` reverts near the cap

1. Bring the vault to a state where total assets and total supply are not in a clean ratio and the vault is close to `depositLimit`.
2. Call `maxMint(receiver)`.
3. Submit `mint(maxMint(receiver), receiver)`.
4. The call reverts with `DepositLimitExceeded()` even though `maxMint()` returned that share amount as safe.

### Scenario 2: dust edge case on `maxDeposit()`

1. Bring the vault into a high share-price state where the remaining asset headroom is 1 wei or similarly tiny.
2. Call `maxDeposit(receiver)`.
3. Submit `deposit(maxDeposit(receiver), receiver)`.
4. The call reverts with `InvalidAmount()` because the deposit floors to zero shares.

### Reproduction note

The included `deposit_and_withdraw.js` testcase demonstrates both behaviors with concrete numbers and a deterministic state progression. It does not invent the bug; it reproduces a reachable edge state that causes the mismatch.
