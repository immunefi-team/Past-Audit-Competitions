# 90313 sc low firelightvault can return positive erc 4626 entry maxima that cannot be executed

**Submitted on Aug 24th 2026 at 11:30:41 UTC by @go4ko for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90313
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

After ordinary rewards raise the FXRP-per-share exchange rate near the vault's deposit cap, `FirelightVault.maxDeposit` and `maxMint` can return positive ERC-4626 entry limits that their corresponding `deposit` and `mint` functions reject in the same unchanged state. `maxDeposit` can report positive asset capacity even though every permitted positive amount rounds to zero shares, while `maxMint` can report one share unit more than the remaining asset headroom can fund. A user or integration trusting either quote spends gas and receives no stFXRP, and a composed router or strategy transaction can revert entirely; the caller's FXRP principal remains safe because execution reverts atomically.

## Vulnerability Details

The vault enforces its deposit cap in asset units, but its entry functions convert between FXRP assets and stFXRP shares using virtual units and asymmetric rounding.

`maxDeposit` ignores the share-side constraint. It returns `depositLimit - totalAssets()` even when that complete remainder rounds down to zero shares. The current `deposit` implementation specifically rejects such an amount with `InvalidAmount`.

`maxMint` performs the asset/share calculation in the wrong order for a remaining-capacity query. It converts the entire deposit limit into shares and subtracts the existing supply afterward. The executable maximum must instead be computed by converting only the remaining asset headroom. Moving the subtraction across a floor-rounded conversion can add one share base unit. Because `mint` rounds the asset cost upward, that extra share can cost one asset unit more than the vault has room to accept.

Both failures are reproduced against the repository's normal configuration:

```
Vault asset decimals:       6
Deposit limit:              50,000 FXRP
Initial vault state:        established by an ordinary deposit
Exchange-rate increase:     1 FXRP sent through VaultRewardDistributor
Incident active:            no
Vault paused:               no
Caller/receiver blocked:    no
Token behavior:             standard test FAsset, no transfer fee or rebase
```

The PoC uses a minimal integration contract that reads each maximum and calls its corresponding entry function inside the same transaction. There is no opportunity for a deposit, reward, limit update, or other state change between quotation and execution.

ERC-4626 requires these maximum functions not to return more than the amount their corresponding entry operation would accept for the receiver. Underestimation is permitted when a vault needs to be conservative. [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)

Firelight documents its ERC-4626 design as a composability feature and describes `deposit`/`mint` as the normal path by which an asset holder enters the vault and receives shares. [Firelight vault documentation](https://docs.firelight.finance/introducing-firelight/core-features/vaults), [Firelight common workflows](https://docs.firelight.finance/technical-documents/common-workflows)

### Root Cause

`maxDeposit` checks global entry restrictions but otherwise exposes all unused asset capacity:

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

The actual deposit path applies an additional restriction after converting the assets with floor rounding:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    assets,
    true,
    Math.Rounding.Floor
);

if (shares == 0) revert InvalidAmount();
```

For this six-decimal vault, `_decimalsOffset() == 0`, so the conversion is:

```
sharesFromAssets(x) = floor(x × (S + 1) / (A + 1))
```

where `S` is the live total share supply and `A` is the live total asset balance. If the remaining asset headroom `H = L - A` satisfies:

```
H × (S + 1) < A + 1
```

then even `H`, the largest amount returned by `maxDeposit`, produces zero shares. Every smaller positive amount does as well. The correct maximum is therefore zero.

`maxMint` converts the whole cap and then subtracts supply:

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

That produces:

```
current quote = floor(L × (S + 1) / (A + 1)) - S
```

The maximum shares whose rounded-up cost fits the asset headroom is instead:

```
executable maximum = floor((L - A) × (S + 1) / (A + 1))
```

These expressions are not equivalent after rounding. The current quote may be one share unit larger. Execution then calculates the assets using ceiling rounding:

```solidity
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Ceil
);
```

and `_depositFunds` rejects the resulting asset total:

```solidity
_totalAssets += assets;

if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

### Failure A: `maxDeposit` Reports Capacity When the Executable Maximum Is Zero

The first test begins with the repository's 50,000-FXRP cap. A normal user deposits `49,998.999999` FXRP, and the real reward distributor forwards a 1-FXRP incentive:

```
depositLimit                       50,000,000,000 asset units
totalSupply                        49,998,999,999 share units
totalAssets                        49,999,999,999 asset units
remaining asset headroom                        1 asset unit
```

The view reports:

```
maxDeposit(receiver) = 1
```

But its corresponding conversion is:

```
previewDeposit(1)
= floor(1 × 49,999,000,000 / 50,000,000,000)
= 0 shares
```

The integration contract obtains the value from `maxDeposit` and immediately calls `deposit(1, receiver)`. The real vault reverts `InvalidAmount` because the advertised maximum cannot mint a share.

There is no smaller positive value to try. Therefore:

```
advertised maxDeposit = 1 asset unit
actual maxDeposit     = 0 asset units
```

### Failure B: `maxMint` Is One Share Unit Above the Executable Boundary

The second test again uses the unmodified 50,000-FXRP cap. This time a user deposits `49,998.95` FXRP, followed by an ordinary 1-FXRP reward:

```
depositLimit                       50,000,000,000 asset units
totalSupply                        49,998,950,000 share units
totalAssets                        49,999,950,000 asset units
remaining asset headroom                   50,000 asset units
```

The asset-side maximum remains usable:

```
maxDeposit(receiver)               50,000 assets
previewDeposit(50,000)             49,998 shares
```

However, the share-side maximum is overstated:

```
maxMint(receiver)                  49,999 shares
previewMint(49,999)                50,001 assets
available headroom                 50,000 assets
```

The quoted mint would make total assets equal `depositLimit + 1`, so an atomic `maxMint` → `mint` call reverts `DepositLimitExceeded`.

The PoC then attempts one share unit less:

```
previewMint(49,998)                49,999 assets
mint(49,998, receiver)             succeeds
```

This proves that a positive executable maximum exists and that the failure is specifically the extra share unit introduced by `maxMint`'s rounding order.

### End-to-End Execution Sequence

1. Deploy the real upgradeable `FirelightVault` using the repository's default six-decimal asset and 50,000-FXRP deposit limit.
2. Deposit normally until the vault is close to its configured cap. The cap is not changed after the state is established.
3. Deploy and authorize the real upgradeable `VaultRewardDistributor` with `CHECKPOINT_ROLE`.
4. Forward a normal 1-FXRP incentive. This increases `totalAssets` without minting shares, creating a legitimate exchange rate above one.
5. Fund a minimal ERC-4626 integration caller with standard vault assets.
6. In the first fixture, the caller reads `maxDeposit(receiver)` and supplies that exact value to `deposit` before returning control. The call reverts `InvalidAmount`.
7. Confirm that neither the caller's underlying balance nor the vault's balance changed and the receiver received no shares.
8. In a separate fixture, the caller reads `maxMint(receiver)` and supplies the result directly to `mint`. The call reverts `DepositLimitExceeded`.
9. Confirm again that balances remain unchanged after the revert.
10. Mint one share base unit less and observe a successful real `Deposit` event, establishing the precise executable limit.

### Known-Issue Distinction

Macro identified an earlier `deposit` behavior in which a dust asset amount could be transferred even though it minted zero shares. The audited implementation remediates that loss by reverting when `shares == 0`.

This report does not claim that the old direct-loss bug remains. Instead, it shows that `maxDeposit` was not synchronized with the remediation: it still returns the zero-share amount as executable even though the corrected deposit path rejects it.

The `maxMint` case does not depend on zero shares at all. The true maximum is a positive 49,998 share units, while `maxMint` returns 49,999. Macro described `mint` as unaffected by its dust-deposit finding and did not report this limit-conversion ordering issue.

The competition's supplied known issues do not disclose either maximum-view mismatch. The PoC additionally avoids the assumptions that commonly invalidate token-accounting reports:

* it uses a standard FAsset without transfer fees;
* total supply is large and nonzero;
* it does not require an empty V2 deployment;
* it does not require a donation from an attacker;
* the configured limit is the repository default;
* it does not rely on an active incident or a privileged mistake; and
* quotation and execution happen in the same transaction.

### Recommended Remediation

Calculate both maximums from the unused asset capacity and ensure `maxDeposit` reflects the zero-share guard.

For example:

```solidity
function maxDeposit(address receiver) public view override returns (uint256) {
    uint256 currentAssets = totalAssets();
    if (
        isBlocklisted[receiver] ||
        paused() ||
        _hasActiveIncident() ||
        currentAssets >= depositLimit
    ) return 0;

    uint256 remainingAssets = depositLimit - currentAssets;
    if (previewDeposit(remainingAssets) == 0) return 0;
    return remainingAssets;
}

function maxMint(address receiver) public view override returns (uint256) {
    uint256 remainingAssets = maxDeposit(receiver);
    if (remainingAssets == 0) return 0;
    return previewDeposit(remainingAssets);
}
```

This makes `maxMint` equal to:

```
floor(remainingAssets × (S + 1) / (A + 1))
```

which is the greatest share amount whose ceiling-rounded mint cost can fit the remaining assets.

Regression properties should assert:

```
When state does not change and maxDeposit(receiver) is positive,
deposit(maxDeposit(receiver), receiver) succeeds.

When state does not change and maxMint(receiver) is positive,
mint(maxMint(receiver), receiver) succeeds.
```

Test these properties after direct deposits, reward forwarding, payouts, and every small remainder below the deposit limit.

## Impact Details

The closest listed impact is **Low - Contract fails to deliver promised returns, but doesn't lose value**.

### Affected Parties and Losses

**The depositing user** is the immediate affected party. A user or wallet that relies on the vault's maximum quote submits an operation that is guaranteed to fail in that state. They pay the failed transaction's gas and receive no stFXRP. Their approved FXRP principal is not lost.

**ERC-4626 routers and aggregators** may use the standardized maximum views to size vault entry. A revert at the Firelight call can unwind a complete atomic route, batch, rebalance, or multicall, including unrelated operations earlier in the same transaction.

**Automated strategy operators** cannot safely treat the reported limits as executable. They need Firelight-specific simulation, decrement-and-retry logic, or a conservative local calculation, weakening the interoperability the maximum functions are designed to provide.

**Firelight and its existing stakers** may miss a small final capital inflow and the prospective rewards associated with it until the caller works around the quote. The demonstrated boundary error is small and does not threaten vault solvency or existing balances.

**No attacker needs to act and no attacker profits.** The state arises from an ordinary deposit, an ordinary reward, and natural use of the configured cap. The PoC has no transaction-ordering dependency.

The direct realized loss is limited to the gas consumed by the reverted call. Any earlier operations in the same atomic transaction are reverted rather than economically lost, although the user or integration also loses the immediate opportunity to acquire the quoted stFXRP position.

### Why the Selected Low Impact Applies

The Low category expressly contemplates a contract failing to deliver promised behavior without losing user value. Both elements are demonstrated here. The maximum functions return an explicit protocol result intended for machine consumption and state that a positive amount can enter the vault, but the corresponding operations cannot deliver the represented stFXRP position. At the same time, the caller's FXRP principal remains safe because execution reverts.

This is not merely a best-practice recommendation or documentation improvement. ERC-4626 normatively requires `maxDeposit` and `maxMint` not to overestimate the amounts their corresponding entry functions accept, and Firelight advertises ERC-4626 compatibility as the basis for integration with the wider DeFi ecosystem. The defect changes observable execution: a standards-based integration receives a positive limit and then reverts while consuming that limit in the unchanged state. Correcting it changes contract behavior rather than optimizing gas or improving explanatory documentation.

The issue also does not concern architectural decentralization or code optimization. No role distribution, governance model, or performance characteristic is implicated. The inconsistency is between public state-returning functions and the state-changing operations those values are defined to bound. It therefore does not fit any of the program's four Insight categories: Security best practices, Code Optimizations and Enhancements, Architectural decentralization, or Documentation Improvements.

The `maxDeposit` manifestation is stronger than a quote that is merely slightly too high. Its returned value is positive while every positive amount at or below it converts to zero shares and reverts. The actual executable maximum remains zero for as long as the same state persists; it becomes positive only if vault assets decrease, the deposit limit increases, or another state transition materially changes the exchange-rate/cap relationship. A caller cannot recover by subtracting one unit because that produces the prohibited zero-asset call.

Value is nevertheless preserved: the revert prevents any asset transfer or share minting from surviving. The user loses transaction gas and the immediate opportunity to enter at the quoted amount, but not their FXRP principal. This limited consequence is why the report selects Low rather than Medium griefing.

### Likelihood and Severity Boundary

The likelihood is limited to states near the asset cap after assets have accrued without proportional share issuance. That is nevertheless a normal vault lifecycle: rewards are intentionally forwarded directly into the vault, and users can progressively fill the remaining capacity.

The issue should not be classified above Low because both state-changing calls revert atomically, no FXRP is retained, no shares are burned or frozen, and no attacker receives value. The `maxMint` caller can succeed by reducing the request to the true maximum; the `maxDeposit` caller must wait for a relevant state change because its actual executable maximum is zero.

The claimed impact is therefore restricted to failure to deliver the entry represented by the standardized return value, failed gas expenditure, delayed share acquisition, and composability failures. No theft, permanent freezing, or protocol insolvency is claimed.

## References

* [`FirelightVault.maxDeposit`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L371-L378)
* [`FirelightVault.maxMint`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L393)
* [`FirelightVault.deposit`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L674)
* [`FirelightVault.mint`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L706)
* [`FirelightVault._depositFunds`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L992)
* [`FirelightVault` conversion helpers](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1074)
* [`VaultRewardDistributor.distributeIncentive`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L149-L162)
* [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)
* [Firelight vault documentation](https://docs.firelight.finance/introducing-firelight/core-features/vaults)
* [Firelight common workflows](https://docs.firelight.finance/technical-documents/common-workflows)
* [Supplied Macro audit](https://library-pr-440.onrender.com/library/audits/firelight-1)

## Proof of Concept

## Proof of Concept

Create:

```
contracts/test/VaultMaxEntryCaller.sol
test/poc_max_entry_quote_consistency.js
```

The PoC is end-to-end for the vulnerable ERC-4626 entry path. It uses the real upgradeable `FirelightVault` and `VaultRewardDistributor`. The helper contract only represents a standard integration that consumes a maximum quote atomically. The local FAsset is mocked because it is an external dependency.

Run:

```bash
npx hardhat test test/poc_max_entry_quote_consistency.js
```

Expected output:

```
  E2E: FirelightVault maximum entry quotes are not executable
    ✔ returns a positive maxDeposit although no positive deposit fits the share-rounding guard
    ✔ returns a maxMint one share unit above the amount affordable by the remaining headroom

  2 passing
```

`contracts/test/VaultMaxEntryCaller.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/// @dev Minimal test integration that consumes a live ERC-4626 maximum in the
/// same transaction in which it is queried.
contract VaultMaxEntryCaller {
    using SafeERC20 for IERC20;

    IERC4626 public immutable vault;
    IERC20 public immutable underlying;

    constructor(IERC4626 vault_) {
        vault = vault_;
        underlying = IERC20(vault_.asset());
        underlying.forceApprove(address(vault_), type(uint256).max);
    }

    function depositUsingLiveMaximum(address receiver) external returns (uint256 shares) {
        uint256 assets = vault.maxDeposit(receiver);
        return vault.deposit(assets, receiver);
    }

    function mintUsingLiveMaximum(address receiver) external returns (uint256 assets) {
        uint256 shares = vault.maxMint(receiver);
        return vault.mint(shares, receiver);
    }

    function mintKnownExecutableAmount(uint256 shares, address receiver) external returns (uint256 assets) {
        return vault.mint(shares, receiver);
    }
}
```

`test/poc_max_entry_quote_consistency.js`:

```javascript
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployVault } = require('./setup/fixtures.js')

const E6 = 10n ** 6n
const DEFAULT_LIMIT = 50_000n * E6

const prepareRewardedVault = async (depositAmount, rewardAmount) => {
  // Use the repository's default six-decimal, 50,000-FXRP deposit limit.
  const ctx = await deployVault()
  const vault = ctx.firelight_vault
  const asset = ctx.token_contract
  const depositor = ctx.users[0]
  const receiver = ctx.users[1]
  const rewardDistributor = ctx.users[2]

  expect(await vault.depositLimit()).to.equal(DEFAULT_LIMIT)

  await ctx.utils.mintAndApprove(depositAmount, depositor)
  await vault.connect(depositor).deposit(depositAmount, depositor.address)

  // The production reward path increases totalAssets without issuing shares.
  const Rewarder = await ethers.getContractFactory('VaultRewardDistributor')
  const rewarder = await upgrades.deployProxy(
    Rewarder,
    [vault.target, ctx.deployer.address, rewardDistributor.address, ethers.ZeroAddress],
    { kind: 'transparent' },
  )
  await vault.connect(ctx.deployer).grantRole(await vault.CHECKPOINT_ROLE(), rewarder.target)
  await asset.mintTo(rewardDistributor.address, rewardAmount)
  await asset.connect(rewardDistributor).approve(rewarder.target, rewardAmount)
  await rewarder
    .connect(rewardDistributor)
    .distributeIncentive(rewardAmount, ethers.id('max-entry-quote-reward'))

  const Caller = await ethers.getContractFactory('VaultMaxEntryCaller')
  const caller = await Caller.deploy(vault.target)
  await asset.mintTo(caller.target, 2n * E6)

  expect(await vault.totalSupply()).to.equal(depositAmount)
  expect(await vault.totalAssets()).to.equal(depositAmount + rewardAmount)

  return { ...ctx, vault, asset, receiver, caller }
}

describe('E2E: FirelightVault maximum entry quotes are not executable', function () {
  it('returns a positive maxDeposit although no positive deposit fits the share-rounding guard', async function () {
    const reward = 1n * E6
    const deposit = DEFAULT_LIMIT - reward - 1n
    const { vault, asset, receiver, caller } = await loadFixture(
      prepareRewardedVault.bind(null, deposit, reward),
    )

    expect(await vault.totalAssets()).to.equal(DEFAULT_LIMIT - 1n)
    expect(await vault.maxDeposit(receiver.address)).to.equal(1n)
    expect(await vault.previewDeposit(1n)).to.equal(0n)

    const vaultBalanceBefore = await asset.balanceOf(vault.target)
    const callerBalanceBefore = await asset.balanceOf(caller.target)

    // maxDeposit is read inside this call and immediately supplied to deposit.
    // The quote and execution therefore observe exactly the same vault state.
    await expect(caller.depositUsingLiveMaximum(receiver.address))
      .to.be.revertedWithCustomError(vault, 'InvalidAmount')

    expect(await asset.balanceOf(vault.target)).to.equal(vaultBalanceBefore)
    expect(await asset.balanceOf(caller.target)).to.equal(callerBalanceBefore)
    expect(await vault.balanceOf(receiver.address)).to.equal(0n)
  })

  it('returns a maxMint one share unit above the amount affordable by the remaining headroom', async function () {
    const reward = 1n * E6
    const remainingHeadroom = 50_000n // 0.05 FXRP
    const deposit = DEFAULT_LIMIT - reward - remainingHeadroom
    const { vault, asset, receiver, caller } = await loadFixture(
      prepareRewardedVault.bind(null, deposit, reward),
    )

    const quotedShares = await vault.maxMint(receiver.address)
    const actualMaximumShares = 49_998n

    expect(await vault.totalAssets()).to.equal(DEFAULT_LIMIT - remainingHeadroom)
    expect(await vault.maxDeposit(receiver.address)).to.equal(remainingHeadroom)
    expect(await vault.previewDeposit(remainingHeadroom)).to.equal(actualMaximumShares)
    expect(quotedShares).to.equal(49_999n)
    expect(await vault.previewMint(quotedShares)).to.equal(50_001n)
    expect((await vault.totalAssets()) + (await vault.previewMint(quotedShares)))
      .to.equal(DEFAULT_LIMIT + 1n)

    const vaultBalanceBefore = await asset.balanceOf(vault.target)
    const callerBalanceBefore = await asset.balanceOf(caller.target)

    // maxMint is read and consumed atomically. Its rounded-up asset cost is one
    // base unit above the limit, so the real vault rejects its own live quote.
    await expect(caller.mintUsingLiveMaximum(receiver.address))
      .to.be.revertedWithCustomError(vault, 'DepositLimitExceeded')

    expect(await asset.balanceOf(vault.target)).to.equal(vaultBalanceBefore)
    expect(await asset.balanceOf(caller.target)).to.equal(callerBalanceBefore)
    expect(await vault.balanceOf(receiver.address)).to.equal(0n)

    // One share unit less is the true maximum and succeeds in the same state.
    expect(await vault.previewMint(actualMaximumShares)).to.equal(49_999n)
    await expect(caller.mintKnownExecutableAmount(actualMaximumShares, receiver.address))
      .to.emit(vault, 'Deposit')
      .withArgs(caller.target, receiver.address, 49_999n, actualMaximumShares)
    expect(await vault.totalAssets()).to.equal(DEFAULT_LIMIT - 1n)
    expect(await vault.balanceOf(receiver.address)).to.equal(actualMaximumShares)
  })
})
```
