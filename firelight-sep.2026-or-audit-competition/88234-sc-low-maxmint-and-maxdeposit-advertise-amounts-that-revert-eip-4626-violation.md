# 88234 sc low maxmint and maxdeposit advertise amounts that revert eip 4626 violation&#x20;

**Submitted on Aug 12th 2026 at 10:22:49 UTC by @shadowHunter for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88234
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

EIP-4626 is explicit that the maximum functions must return values the corresponding action will accept:

> `maxMint` — *"MUST return the maximum amount of shares `mint` would allow to be deposited to `receiver` and not cause a revert."* `maxDeposit` — *"MUST return the maximum amount of underlying assets that can be deposited into the Vault for the `receiver`, through a `deposit` call, and not cause a revert."*

`FirelightVault` violates both. The vault is marketed as ERC-4626 compatible in `README.md` and its interface inherits `IERC4626` (`contracts/core/interfaces/IFirelightVault.sol:22`), so the full ERC-4626 selector set is published in the ABI that integrators compile against.

### Defect 1 — `maxMint` derives a share ceiling from the whole deposit limit

`contracts/core/FirelightVault.sol:385-393`

```solidity
function maxMint(address receiver) public view override returns (uint256 amount) {
    uint256 shares = totalSupply();
    uint256 sharesLimit = convertToShares(depositLimit);           // whole limit, floored
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || shares > sharesLimit) {
        return 0;
    } else {
        return sharesLimit - shares;
    }
}
```

`mint`, however, enforces an **asset** ceiling on the remaining headroom:

`contracts/core/FirelightVault.sol:976-991` (`_depositFunds`)

```solidity
_totalSupply += shares;
_totalAssets += assets;

if (_totalAssets > depositLimit) revert DepositLimitExceeded();     // :987
```

`convertToShares` floors while `previewMint` ceils, so once the share price exceeds 1 the two disagree and `maxMint` over-reports.

Worked example with `totalSupply S = 100`, `totalAssets A = 101`, `depositLimit D = 102` (`_decimalsOffset()` is not overridden, so the virtual offset is 1):

```
sharesLimit = floor(102 * (100 + 1) / (101 + 1)) = floor(101.0) = 101
maxMint     = 101 - 100 = 1
previewMint(1) = ceil(1 * (101 + 1) / (100 + 1)) = ceil(1.0099) = 2 assets
_totalAssets   = 101 + 2 = 103 > 102  ->  DepositLimitExceeded
```

The condition `A > S` is the steady state after the first reward distribution, because `VaultRewardDistributor.distributeRewards` forwards assets into the vault without minting shares (`contracts/core/VaultRewardDistributor.sol:146`). The share price is therefore permanently above 1 in normal operation.

### Defect 2 — `maxDeposit` ignores the zero-share guard

`contracts/core/FirelightVault.sol:371-378`

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

It returns the remaining asset headroom without checking that the headroom is large enough to mint at least one share. `deposit` rejects exactly that case:

`contracts/core/FirelightVault.sol:667-669`

```solidity
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();
```

With an empty vault, a 10-asset donation and `depositLimit = 11`: `maxDeposit` returns `1`, but `previewDeposit(1)` is `0` and `deposit(1)` reverts.

### Note on prior review

This code was introduced by commit `29cfa99` ("Fix 4626 max methods implementation"), which was the remediation for the OpenZeppelin client-reported issue CR-01 ("Limit Functions Not Overridden to Enforce Deposit/Withdraw Restrictions"). CR-01 asked only that the maximum functions reflect deposit limits, blocklist status and pause state — it did not concern rounding consistency. The defect is therefore a genuine unintended bug, though it sits in previously reviewed code.

***

## Steps to reproduce

### Case 1 — `maxMint`

**1. A user deposits 100 assets into an empty vault**, receiving 100 shares at the 1:1 bootstrap rate. `depositLimit` is 102.

```solidity
deposit(100, user);      // totalSupply = 100, totalAssets = 100
```

**2. One asset reaches the vault without minting shares.** In production this is a reward distribution (`contracts/core/VaultRewardDistributor.sol:146`); in the PoC it is a direct transfer, which is equivalent from the vault's point of view because `totalAssets()` is balance-derived.

```solidity
token.transfer(address(vault), 1);   // totalAssets = 101, totalSupply = 100
```

**3. The integrator reads the advertised maximum.**

```solidity
maxMint(user);        // returns 1   (FirelightVault.sol:385-393)
previewMint(1);       // returns 2   -- already inconsistent
```

**4. The integrator performs the canonical ERC-4626 call and it reverts.**

```solidity
mint(1, user);
// _depositFunds: _totalAssets = 101 + 2 = 103 > depositLimit 102
// reverts: DepositLimitExceeded   (FirelightVault.sol:987)
```

### Case 2 — `maxDeposit`

**1. An empty vault has `depositLimit = 11`, and 10 assets are donated to it.**

```solidity
token.transfer(address(vault), 10);   // totalAssets = 10, totalSupply = 0
```

**2. The integrator reads the advertised maximum.**

```solidity
maxDeposit(user);      // returns 11 - 10 = 1   (FirelightVault.sol:371-378)
previewDeposit(1);     // returns floor(1 * (0 + 1) / (10 + 1)) = 0
```

**3. The canonical call reverts.**

```solidity
deposit(1, user);
// shares == 0  ->  revert InvalidAmount   (FirelightVault.sol:669)
```

***

## Impact

**No principal is lost — both transactions revert with no state change.** The impact is purely that the contract does not deliver the API behaviour it advertises.

* Any router, zapper, vault aggregator, or front-end "Max" button that follows the canonical ERC-4626 pattern `vault.mint(vault.maxMint(user), user)` or `vault.deposit(vault.maxDeposit(user), user)` will submit a **guaranteed-to-fail transaction**, wasting the user's gas.
* The failures look **non-deterministic** from the integrator's side, because whether they occur depends on the flooring of `depositLimit * (S + 1) / (A + 1)` — so they are hard to diagnose and will present as intermittent bugs in the integrating protocol.
* The `maxMint` case is not an edge condition that resolves itself: the share price sits permanently above 1 after the first reward distribution, so the divergence is the normal operating state rather than a transient one.
* Integrations that automate deposits (yield routers, auto-compounders) may retry indefinitely against a maximum that can never be executed.

The repository's own tests assert the intended invariant `maxMint == previewDeposit(maxDeposit)` (`test/deposit_and_withdraw.js:41-47`, `:61-67`, `:84-90`), but only at a 1:1 share price, where the two formulas coincide — which is why the divergence is untested.

***

## Recommendation

Derive `maxMint` from the remaining **asset** headroom rather than from the whole limit, so it is consistent with the ceiling `mint` actually enforces:

```solidity
function maxMint(address receiver) public view override returns (uint256) {
    return previewDeposit(maxDeposit(receiver));
}
```

And return zero from `maxDeposit` when the remaining headroom cannot mint at least one share:

```solidity
function maxDeposit(address receiver) public view override returns (uint256 amount) {
    uint256 assets = totalAssets();
    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) {
        return 0;
    }
    uint256 remaining = depositLimit - assets;
    if (previewDeposit(remaining) == 0) return 0;
    return remaining;
}
```

Extend the existing limit tests to run at a share price above 1 (for example after a simulated reward distribution), so the two formulas are exercised where they diverge.

## Proof of Concept

## Proof of Concept

### Environment

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

### Create `hardhat.poc.config.js` in the repository root

```js
require('@nomicfoundation/hardhat-ethers')
require('@nomicfoundation/hardhat-chai-matchers')
require('@openzeppelin/hardhat-upgrades')

module.exports = {
  defaultNetwork: 'hardhat',
  networks: { hardhat: { allowUnlimitedContractSize: true } },
  solidity: {
    compilers: [
      { version: '0.8.23', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london' } },
      { version: '0.8.28', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london', viaIR: true } },
      { version: '0.8.29', settings: { optimizer: { enabled: true, runs: 1000 }, evmVersion: 'london' } },
    ],
    overrides: {
      'contracts/core/CoverOrderAllocator.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 100 }, evmVersion: 'london', viaIR: true },
      },
      'contracts/core/FirelightVault.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 800 }, evmVersion: 'london', viaIR: true },
      },
    },
  },
  paths: { sources: './contracts', tests: './test', cache: './.audit-cache', artifacts: './.audit-artifacts' },
}
```

### Save the PoC as `test/poc_low_erc4626_limits.js`

```js
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { deployVault } = require('./setup/fixtures.js')

describe('PoC: ERC-4626 limit methods advertise reverting operations', function () {
  it('maxMint reports one share although minting it exceeds the deposit limit', async function () {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: 102n }))
    const user = ctx.users[0]

    await ctx.utils.mintAndApprove(103n, user)
    await ctx.firelight_vault.connect(user).deposit(100n, user.address)
    await ctx.token_contract.connect(user).transfer(ctx.firelight_vault.target, 1n)

    expect(await ctx.firelight_vault.totalSupply()).to.equal(100n)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(101n)
    expect(await ctx.firelight_vault.maxMint(user.address)).to.equal(1n)
    expect(await ctx.firelight_vault.previewMint(1n)).to.equal(2n)

    await expect(ctx.firelight_vault.connect(user).mint(1n, user.address)).to.be.reverted
  })

  it('maxDeposit reports one asset although it would mint zero shares', async function () {
    const ctx = await loadFixture(deployVault.bind(null, { initial_deposit_limit: 11n }))
    const user = ctx.users[0]

    await ctx.utils.mintAndApprove(11n, user)
    await ctx.token_contract.connect(user).transfer(ctx.firelight_vault.target, 10n)

    expect(await ctx.firelight_vault.maxDeposit(user.address)).to.equal(1n)
    expect(await ctx.firelight_vault.previewDeposit(1n)).to.equal(0n)
    await expect(ctx.firelight_vault.connect(user).deposit(1n, user.address)).to.be.reverted
  })
})
```

### Run

```bash
node node_modules/hardhat/internal/cli/bootstrap.js --config hardhat.poc.config.js test test/poc_low_erc4626_limits.js
```

### Output

```
  PoC: ERC-4626 limit methods advertise reverting operations
    √ maxMint reports one share although minting it exceeds the deposit limit (2360ms)
    √ maxDeposit reports one asset although it would mint zero shares (184ms)


  2 passing (2s)
```

### What the output proves

Both cases use the real `FirelightVault` behind a transparent proxy.

**Case 1 — `maxMint` over-reports.** The state assertions pin the exact scenario: `totalSupply() == 100`, `totalAssets() == 101` (share price above 1), `depositLimit == 102`.

* `maxMint(user) == 1` — the vault advertises that one share can be minted.
* `previewMint(1) == 2` — but minting that one share costs **two** assets. The vault's own two views already contradict each other.
* `mint(1, user)` reverts — the advertised maximum is not executable, because `101 + 2 = 103` exceeds the limit of `102` at `FirelightVault.sol:987`.

This is a direct violation of the EIP-4626 `maxMint` MUST clause.

**Case 2 — `maxDeposit` over-reports.**

* `maxDeposit(user) == 1` — one asset of headroom is advertised.
* `previewDeposit(1) == 0` — depositing it would mint zero shares.
* `deposit(1, user)` reverts on the `shares == 0` guard at `FirelightVault.sol:669`.

In both cases the transaction reverts with no state change and no principal at risk, which is why this is reported as Low: the harm is a guaranteed-failing transaction and wasted gas for any integrator that trusts the standard interface.
