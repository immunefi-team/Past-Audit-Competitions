# 89876 sc low rounding inconsistent maxdeposit and maxmint values can advertise vault entry that necessarily reverts

**Submitted on Aug 21st 2026 at 16:09:07 UTC by @Nescient for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89876
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

### Brief/Intro

`FirelightVault.maxDeposit` reports the raw asset headroom below `depositLimit`, without checking whether that amount converts to a nonzero number of shares. After ordinary rewards increase the vault's asset-per-share exchange rate, the remaining headroom can be too small to mint even one share base unit. `maxDeposit` nevertheless reports that headroom as available, while `deposit` deliberately rejects it with `InvalidAmount`.

`FirelightVault.maxMint` takes a different inconsistent route: it converts the total deposit limit to shares and only then subtracts the existing share supply. Executable mint capacity must instead be derived from the remaining asset headroom. Because conversion rounds down while `mint` rounds its asset cost up, the current calculation can advertise one share base unit more than the remaining headroom can purchase. Calling `mint(maxMint(receiver), receiver)` then reverts with `DepositLimitExceeded`.

The defect is reachable in a nonempty V2 vault through normal protocol operations. The fully end-to-end PoC:

1. deposits 100 FXRP through the real upgradeable `FirelightVault`;
2. forwards 1 FXRP through the real upgradeable `VaultRewardDistributor`, increasing assets without minting shares;
3. queries and executes each advertised maximum in the same transaction; and
4. proves that the quote is rejected without any intervening state change.

This conflicts with ERC-4626, which requires `maxDeposit` and `maxMint` not to overestimate the amount accepted by their corresponding entry function. The standard expressly permits these functions to underestimate when necessary. EIP-4626 — <https://eips.ethereum.org/EIPS/eip-4626>

Firelight also documents that its vault is ERC-4626-based for DeFi compatibility and composability, and that a normal deposit or mint transfers the underlying asset and returns vault shares. Firelight vault documentation — <https://docs.firelight.finance/introducing-firelight/core-features/vaults>, Firelight common workflows — <https://docs.firelight.finance/technical-documents/common-workflows>

### Vulnerability Details

The maximum asset quote subtracts live assets from the limit:

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

This checks only the asset limit. It omits the execution constraint added to `deposit`: the asset amount must mint at least one share unit.

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    assets,
    true,
    Math.Rounding.Floor
);
if (shares == 0) revert InvalidAmount();
```

For the six-decimal FXRP vault, the conversion is:

```
depositShares(x) = floor(x * (totalSupply + 1) / (totalAssets + 1))
```

Let:

```
S = totalSupply
A = totalAssets
L = depositLimit
H = L - A
```

`maxDeposit` returns `H`. If:

```
H * (S + 1) < A + 1
```

then `previewDeposit(H) == 0`. Because conversion is monotonic, every positive amount at or below the advertised maximum also converts to zero. The actual executable maximum is therefore zero, not `H`.

The maximum share quote is calculated as follows:

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

The returned value is:

```
reportedMaxMint = floor(L * (S + 1) / (A + 1)) - S
```

However, `mint` rounds the asset requirement upward and `_depositFunds` rejects any result above `L`:

```solidity
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Ceil
);

_totalAssets += assets;
if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

Accordingly, the actual maximum number of mintable shares is:

```
actualMaxMint = floor((L - A) * (S + 1) / (A + 1))
```

When rewards have made `A > S`, converting `L` before subtracting `S` can preserve a fractional component from the existing assets. It can round `reportedMaxMint` one share base unit above `actualMaxMint`. `previewMint(reportedMaxMint)` then exceeds `H`, so the advertised maximum necessarily reverts.

### Concrete `maxDeposit` Failure

The first PoC case uses six-decimal FXRP units:

```
Ordinary deposit                         100.000000 FXRP
Ordinary incentive distribution           1.000000 FXRP
totalSupply                              100,000,000 share units
totalAssets                              101,000,000 asset units
depositLimit                             101,000,001 asset units
```

The vault reports:

```
maxDeposit(receiver) = 1 asset unit
previewDeposit(1)    = floor(1 * 100,000,001 / 101,000,001)
                     = 0 shares
```

The PoC's integration caller invokes `maxDeposit` and passes its result directly to `deposit` within the same transaction. `deposit(1, receiver)` reverts `InvalidAmount`. The vault balance and caller balance remain unchanged and the receiver obtains no shares.

Because the advertised maximum is one unit, no smaller positive amount exists. `maxDeposit` therefore advertises positive capacity while the actual maximum executable deposit is zero.

### Concrete `maxMint` Failure

The second case starts from the same ordinary deposit and incentive state but uses this configured limit:

```
totalSupply                              100,000,000 share units
totalAssets                              101,000,000 asset units
depositLimit                             101,101,001 asset units
remaining asset headroom                     101,001 asset units
```

The current calculation returns:

```
maxMint(receiver)                        100,001 share units
previewMint(100,001)                     101,002 asset units
remaining headroom                       101,001 asset units
result                                   reverts by 1 asset unit
```

The same-transaction quote-and-mint call reverts `DepositLimitExceeded`. The PoC then proves the precise executable boundary:

```
previewMint(100,000)                     101,000 asset units
mint(100,000, receiver)                  succeeds
```

Thus the failure is not caused by pause, blocklisting, insufficient approval, an active incident, a changing price, or a concurrent deposit. It is the rounding order in `maxMint` itself.

### Fully End-to-End Failure Sequence

1. The vault is initialized with its deposit limit. The limit is not changed after the test state is created.
2. A normal staker deposits 100 FXRP and receives 100 stFXRP.
3. An authorized distributor forwards an ordinary 1-FXRP incentive through the real `VaultRewardDistributor`. This increases `totalAssets` to 101 FXRP while `totalSupply` remains 100 stFXRP, as intended for share-accruing rewards.
4. There is no active incident, the vault is not paused, and neither the integration caller nor receiver is blocklisted.
5. For the `maxDeposit` configuration, the remaining headroom is one asset base unit. `maxDeposit` returns one even though that amount produces zero shares.
6. A test-only integration caller queries `maxDeposit` and immediately calls `deposit` with the returned amount in one transaction. The real vault reverts `InvalidAmount`.
7. In an independent fixture for `maxMint`, the remaining headroom is 101,001 asset units. `maxMint` returns 100,001 shares, whose rounded-up cost is 101,002 assets.
8. The integration caller queries `maxMint` and immediately calls `mint` with the returned amount in one transaction. The real vault reverts `DepositLimitExceeded`.
9. Minting one share base unit less succeeds, confirming the true boundary and isolating the faulty maximum quote.

### Affected Parties and Practical Impact

* **Direct staker:** pays gas for a transaction that the vault's own maximum view represented as executable and receives none of the expected stFXRP. The FXRP principal remains safe because the revert rolls back the transfer.
* **ERC-4626 router or vault aggregator:** a standard `max*`-bounded entry flow can revert atomically at the Firelight leg. If Firelight is one operation in a larger multicall, strategy deployment, or rebalance, the whole transaction can revert and the other operations are rolled back.
* **Automated strategy/user interface:** a client using `maxDeposit` or `maxMint` to enable its final-capacity action can repeatedly propose a transaction that can never succeed in the unchanged state. It must implement Firelight-specific retry logic or reduce the quote heuristically.
* **Firelight/stakers:** the last quoted capacity cannot be consumed through the affected maximum-based path. The protocol may miss a small amount of capital deployment and the associated prospective fee participation. The demonstrated amount is dust; no material protocol insolvency or liveness impact is claimed.
* **Attacker:** none is required and nobody receives the caller's lost gas. The inconsistent state arises naturally after reward distribution as the vault approaches its configured cap. Front-running is not part of the PoC.

This report deliberately does not claim theft, freezing, or principal loss. Both operations fail before assets are transferred. The proposed Low impact is limited to failure to deliver the standardized, currently executable vault entry returned by `maxDeposit`/`maxMint`, the resulting failed-transaction gas, and integration disruption.

## Known-Issue Distinction

Macro's audit reported that an earlier `deposit` implementation could accept dust assets while minting zero shares, causing direct asset loss. That issue is fixed in the audited code by:

```solidity
if (shares == 0) revert InvalidAmount();
```

The `maxDeposit` manifestation is a residual inconsistency after that fix. The guard correctly prevents the old loss, but `maxDeposit` was not updated to return zero when all remaining deposit headroom is rejected by the new guard. This report does not claim the fixed asset-loss impact; it demonstrates that the maximum view still promises an entry the fixed execution path rejects.

Macro expressly stated that `mint` was unaffected by its zero-share deposit issue. The `maxMint` manifestation here is different: a positive share quote has a rounded-up asset cost above the deposit limit because conversion occurs before existing supply is subtracted.

Neither manifestation relies on:

* zero total supply or a fresh empty V2 deployment;
* fee-on-transfer, rebasing, or otherwise nonstandard tokens;
* donation by an attacker;
* V1-to-V2 storage state;
* a malicious or incorrect privileged action;
* incident processing; or
* a state change between quotation and execution.

The supplied competition known issues do not disclose either maximum-quote inconsistency. The supplied Macro audit does not disclose `maxMint`, and its dust-deposit finding is distinguished above.

## Impact Details

The best available in-scope classification is **Low — Contract fails to deliver promised returns, but doesn't lose value**.

The relevant promise is concrete and machine-consumed: the ERC-4626 maximum functions return the greatest amount the corresponding entry function should accept in the current state. Firelight advertises ERC-4626 compatibility specifically for broader DeFi integration. An immediate call using either returned maximum fails, so the receiver does not obtain the quoted vault position.

The category's qualifier is what makes it the correct fit. "But doesn't lose value" is not incidental — a failure that loses value belongs in a higher tier, and this tier exists for functional failures where nothing is lost. Both halves are satisfied here: an ERC-4626 entry function rejects the exact amount the contract's own maximum view advertised, and the revert is atomic so no principal moves.

The Insight categories do not fit. EIP-4626 states that `maxDeposit` and `maxMint` MUST NOT overestimate the amount their corresponding entry function accepts; a normative requirement in a standard the project advertises compliance with is not a "best practice". Nothing here is a code optimization, an architectural decentralization concern, or a documentation gap — the documentation is correct and the implementation contradicts it.

In the `maxDeposit` case the executable maximum is not merely lower than advertised, it is zero: no positive deposit within the reported headroom mints a nonzero share, so the vault advertises available capacity while accepting none, and remains in that state until assets leave or the limit is raised.

The value-preservation qualifier is also satisfied: the vault reverts before transferring the caller's FXRP or minting shares. The report therefore does not elevate the issue to Medium griefing. There is no attacker profit, no third-party principal damage, and no sustained protocol outage. The practical losses are transaction gas, failure of a composed transaction, delayed vault entry, and potentially missed prospective participation while the client diagnoses or works around the inaccurate quote.

## Recommended Remediation

Derive both maximums from the remaining asset headroom and the same conversion constraints used by execution.

Conceptually:

```solidity
function maxDeposit(address receiver) public view override returns (uint256) {
    uint256 assets = totalAssets();
    if (
        isBlocklisted[receiver] ||
        paused() ||
        _hasActiveIncident() ||
        assets >= depositLimit
    ) return 0;

    uint256 headroom = depositLimit - assets;
    return previewDeposit(headroom) == 0 ? 0 : headroom;
}

function maxMint(address receiver) public view override returns (uint256) {
    uint256 maxAssets = maxDeposit(receiver);
    return maxAssets == 0 ? 0 : previewDeposit(maxAssets);
}
```

`previewDeposit(headroom)` rounds down to the greatest share amount whose rounded-up `previewMint` cost remains within the headroom. Returning zero for zero-share asset dust also keeps `maxDeposit` aligned with the explicit `deposit` guard.

Add invariants covering reward-accrued exchange rates and every deposit-limit remainder:

```
If maxDeposit(receiver) > 0 and state is unchanged,
deposit(maxDeposit(receiver), receiver) does not revert.

If maxMint(receiver) > 0 and state is unchanged,
mint(maxMint(receiver), receiver) does not revert.
```

## References

* FirelightVault.maxDeposit — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L371-L378>
* FirelightVault.maxMint — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L393>
* FirelightVault.deposit — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L674>
* FirelightVault.mint — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L706>
* FirelightVault.\_depositFunds — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L992>
* FirelightVault conversion helpers — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1058-L1074>
* VaultRewardDistributor.distributeIncentive — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L149-L162>
* EIP-4626 — <https://eips.ethereum.org/EIPS/eip-4626>
* Firelight vault documentation — <https://docs.firelight.finance/introducing-firelight/core-features/vaults>
* Firelight common workflows — <https://docs.firelight.finance/technical-documents/common-workflows>
* Supplied Macro audit — <https://library-pr-440.onrender.com/library/audits/firelight-1>

## Proof of Concept

## Proof of Concept

Create `contracts/test/MaxQuoteExecutor.sol` and `test/max-quote-rounding-poc.js` with the contents below.

This is a fully end-to-end V2 quotation/execution PoC. It deploys the real upgradeable `FirelightVault` and the real upgradeable `VaultRewardDistributor`. The vault state is established through an ordinary deposit and ordinary incentive distribution. The only helper is a minimal ERC-4626 integration caller that queries a maximum and calls the corresponding function in the same transaction. The external FAsset dependency remains mocked locally.

Run:

```bash
npx hardhat test test/max-quote-rounding-poc.js
```

Expected result:

```
  E2E: ERC-4626 maximum quotes exceed executable deposit capacity
    ✔ maxDeposit advertises the entire positive headroom although every allowed positive deposit reverts
    ✔ maxMint advertises one share unit more than the deposit-limit headroom can buy

  2 passing
```

`contracts/test/MaxQuoteExecutor.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/// @dev Test-only integration caller that quotes and executes an ERC-4626 maximum
/// in one transaction, so no intervening state change can explain a failure.
contract MaxQuoteExecutor {
    using SafeERC20 for IERC20;

    IERC4626 public immutable vault;
    IERC20 public immutable asset;

    constructor(IERC4626 vault_) {
        vault = vault_;
        asset = IERC20(vault_.asset());
    }

    function depositAdvertisedMax(address receiver) external returns (uint256 shares) {
        uint256 assets = vault.maxDeposit(receiver);
        asset.forceApprove(address(vault), assets);
        return vault.deposit(assets, receiver);
    }

    function mintAdvertisedMax(address receiver) external returns (uint256 assets) {
        uint256 shares = vault.maxMint(receiver);
        asset.forceApprove(address(vault), type(uint256).max);
        return vault.mint(shares, receiver);
    }

    function mintExact(uint256 shares, address receiver) external returns (uint256 assets) {
        asset.forceApprove(address(vault), type(uint256).max);
        return vault.mint(shares, receiver);
    }
}
```

`test/max-quote-rounding-poc.js`:

```javascript
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployVault } = require('./setup/fixtures.js')

const E6 = 10n ** 6n
const DEPOSIT = 100n * E6
const REWARD = 1n * E6

const deployQuoteFixture = async (depositLimit) => {
  const ctx = await deployVault({
    decimals: 6,
    initial_deposit_limit: depositLimit,
  })

  const depositor = ctx.users[0]
  const receiver = ctx.users[1]
  const distributor = ctx.users[2]
  const vault = ctx.firelight_vault
  const asset = ctx.token_contract

  // Establish a nonempty, ordinary vault state.
  await ctx.utils.mintAndApprove(DEPOSIT, depositor)
  await vault.connect(depositor).deposit(DEPOSIT, depositor.address)

  // Raise the share price through the real production reward path. Rewards add
  // assets without minting shares, which makes rounding relevant at the limit.
  const RewardDistributor = await ethers.getContractFactory('VaultRewardDistributor')
  const rewarder = await upgrades.deployProxy(
    RewardDistributor,
    [vault.target, ctx.deployer.address, distributor.address, ethers.ZeroAddress],
    { kind: 'transparent' },
  )
  await vault.connect(ctx.deployer).grantRole(await vault.CHECKPOINT_ROLE(), rewarder.target)
  await asset.mintTo(distributor.address, REWARD)
  await asset.connect(distributor).approve(rewarder.target, REWARD)
  await rewarder.connect(distributor).distributeIncentive(REWARD, ethers.id('ordinary-incentive'))

  const Executor = await ethers.getContractFactory('MaxQuoteExecutor')
  const executor = await Executor.deploy(vault.target)
  await asset.mintTo(executor.target, 1n * E6)

  expect(await vault.totalSupply()).to.equal(DEPOSIT)
  expect(await vault.totalAssets()).to.equal(DEPOSIT + REWARD)

  return { ...ctx, vault, asset, receiver, executor }
}

describe('E2E: ERC-4626 maximum quotes exceed executable deposit capacity', function () {
  it('maxDeposit advertises the entire positive headroom although every allowed positive deposit reverts', async function () {
    // After the ordinary deposit and reward, only one asset base unit remains.
    const limit = DEPOSIT + REWARD + 1n
    const { vault, asset, receiver, executor } = await loadFixture(
      deployQuoteFixture.bind(null, limit),
    )

    const advertisedAssets = await vault.maxDeposit(receiver.address)
    expect(advertisedAssets).to.equal(1n)
    expect(await vault.previewDeposit(advertisedAssets)).to.equal(0n)

    const vaultBalanceBefore = await asset.balanceOf(vault.target)
    const executorBalanceBefore = await asset.balanceOf(executor.target)

    // The helper reads maxDeposit and immediately deposits that exact amount in
    // the same transaction. There is no intervening block or state mutation.
    await expect(executor.depositAdvertisedMax(receiver.address))
      .to.be.revertedWithCustomError(vault, 'InvalidAmount')

    expect(await asset.balanceOf(vault.target)).to.equal(vaultBalanceBefore)
    expect(await asset.balanceOf(executor.target)).to.equal(executorBalanceBefore)
    expect(await vault.balanceOf(receiver.address)).to.equal(0n)
  })

  it('maxMint advertises one share unit more than the deposit-limit headroom can buy', async function () {
    // The 101,001-unit headroom can mint at most 100,000 share units. Computing
    // convertToShares(limit) - totalSupply instead advertises 100,001.
    const limit = DEPOSIT + REWARD + 101_001n
    const { vault, asset, receiver, executor } = await loadFixture(
      deployQuoteFixture.bind(null, limit),
    )

    const advertisedShares = await vault.maxMint(receiver.address)
    const executableShares = advertisedShares - 1n
    expect(advertisedShares).to.equal(100_001n)
    expect(await vault.previewMint(advertisedShares)).to.equal(101_002n)
    expect((await vault.totalAssets()) + (await vault.previewMint(advertisedShares))).to.equal(limit + 1n)
    expect(await vault.previewMint(executableShares)).to.equal(101_000n)

    const vaultBalanceBefore = await asset.balanceOf(vault.target)
    const executorBalanceBefore = await asset.balanceOf(executor.target)

    // The helper reads maxMint and immediately mints that exact amount in the
    // same transaction. The vault rejects its own advertised maximum.
    await expect(executor.mintAdvertisedMax(receiver.address))
      .to.be.revertedWithCustomError(vault, 'DepositLimitExceeded')

    expect(await asset.balanceOf(vault.target)).to.equal(vaultBalanceBefore)
    expect(await asset.balanceOf(executor.target)).to.equal(executorBalanceBefore)
    expect(await vault.balanceOf(receiver.address)).to.equal(0n)

    // One share base unit less is the true executable maximum. It succeeds and
    // leaves one unusable asset unit, proving that the failure is maxMint rounding.
    await expect(executor.mintExact(executableShares, receiver.address))
      .to.emit(vault, 'Deposit')
      .withArgs(executor.target, receiver.address, 101_000n, executableShares)
    expect(await vault.totalAssets()).to.equal(limit - 1n)
    expect(await vault.balanceOf(receiver.address)).to.equal(executableShares)
  })
})
```
