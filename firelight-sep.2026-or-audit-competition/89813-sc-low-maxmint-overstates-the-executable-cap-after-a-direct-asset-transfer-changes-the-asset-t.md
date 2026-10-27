# 89813 sc low maxmint overstates the executable cap after a direct asset transfer changes the asset to share ratio

**Submitted on Aug 21st 2026 at 10:23:52 UTC by @legat for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89813
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`FirelightVault.maxMint` converts the entire asset cap to shares with floor rounding, then subtracts existing share supply. `mint`, by contrast, converts only the newly requested shares to assets with ceiling rounding and rejects the operation if those assets exceed the remaining cap headroom. Those are not equivalent calculations once assets enter the vault without minting shares.

Any underlying-token holder can force the mismatch without a Firelight role. The PoC first proves that `maxMint = 100 stXRP` is executable in a one-to-one state, then has an unrelated account transfer 0.007071 FXRP directly to the vault. That permissionless transfer alone changes `maxMint` to 99.985859 stXRP, one atomic share above the actual maximum. Calling `mint(maxMint(receiver), receiver)` then needs one atomic asset more than the remaining cap and reverts `DepositLimitExceeded`, while minting exactly one atomic share less succeeds from the same state.

No distributor, administrator, oracle, callback, or victim-selected composition is required. The runnable PoC deploys the exact frozen contracts and proves both the role-free transition and the normal production reward path:

```
role-free pre-transfer maxMint      = 100.000000 stXRP
mint(pre-transfer maxMint)          = executable
unprivileged direct transfer        = 0.007071 FXRP
role-free post-transfer maxMint     = 99.985859 stXRP
remaining asset capacity            = 99.992929 FXRP
previewMint(post-transfer maxMint)  = 99.992930 FXRP
mint(post-transfer maxMint)         = DepositLimitExceeded
true executable maximum             = 99.985858 stXRP
mint(true executable maximum)       = success

ordinary distributed incentive     = 0.007071 FXRP
production-path maxMint             = 99.985859 stXRP
production-path mint(maxMint)       = DepositLimitExceeded
```

Although the arithmetic overstatement is one atomic share, it makes the entire advertised boundary operation—approximately 99.985859 stXRP in the reproduction—unexecutable.

## Description

### Intended behavior

Firelight documents the vault as implementing the [ERC-4626 tokenized-vault standard](https://docs.firelight.finance/core-concepts/vault-architecture#erc-4626-foundation), emphasizing standardized accounting and predictable integration. The official [`maxMint` specification](https://eips.ethereum.org/EIPS/eip-4626#maxmint) requires the getter to return an amount that `mint` would allow without reverting and says it must not be higher than the actual accepted maximum.

The frozen contract's own NatSpec likewise promises:

```solidity
/**
 * @notice Returns the maximum amount of the Vault shares that can be minted for the receiver, through a mint call.
 */
```

The maximum must therefore be derived from the assets still available under the cap and the same conversion relation used by `mint`.

### Actual behavior

The getter converts the total `depositLimit` at the current exchange rate, then subtracts current supply:

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

`mint` instead ceiling-converts only the newly requested shares to assets, then adds those assets to current assets before checking the cap:

```solidity
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares,
    false,
    Math.Rounding.Ceil
);

_depositFunds(_msgSender(), receiver, assets, shares, _totalSupply, _totalAssets);
```

```solidity
_totalSupply += shares;
_totalAssets += assets;

if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

For the frozen vault's zero decimals offset, let:

```
A = totalAssets
S = totalSupply
L = depositLimit
H = L - A
```

The getter returns:

```
advertised M = floor(L * (S + 1) / (A + 1)) - S
```

The actual maximum that satisfies `mint`'s ceiling conversion and cap check is:

```
true T = floor(H * (S + 1) / (A + 1))
```

The getter's “convert the total cap, then subtract supply” expression is not distributive through integer floor rounding and the ERC-4626 virtual-asset/share terms. After an asset inflow increases `A` without increasing `S`, `M` can equal `T + 1`. `previewMint(M)` then ceiling-rounds to `H + 1` assets and `_depositFunds` reverts, while `previewMint(T) <= H` and the mint succeeds.

`totalAssets()` is derived from the vault's underlying-token balance, less pending withdrawals. A standard ERC-20 transfer to the vault therefore increases `A` without minting shares and requires no Firelight function or role. The role-free PoC starts from an executable boundary and shows that a direct transfer of 7,071 atomic FXRP is the only state change needed to create the exact overstatement.

The same state also arises through `VaultRewardDistributor.distributeIncentive`, an intended scoped production path that transfers assets into the vault without minting new shares. That scenario confirms ordinary protocol reachability but is not a precondition for the role-free forced path.

## Root Cause

[`maxMint`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L385-L392) converts the full absolute asset cap and subtracts supply. The executable limit must instead convert the remaining asset headroom `depositLimit - totalAssets` using the same snapshotted totals.

The incorrect expression appears correct only in the one-to-one states covered by the historical regression tests. Because [`totalAssets`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L422-L429) ultimately reads the vault's ERC-20 balance, any holder can change the ratio through a direct transfer. Ordinary [`distributeIncentive`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176) or [`distributeRewards`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L132-L152) independently exposes the same error during normal production operation.

## Risk

### Impact

This is Smart Contract **Low**, the listed impact **“Contract fails to deliver promised returns, but doesn't lose value.”**

The contract promises a standards-compatible maximum-mint query but advertises a share amount its corresponding state-changing function rejects. Wallets, aggregators, strategies, and users conventionally use `maxMint` to size a boundary operation. Here the approximately 99.985859 stXRP operation returned by the contract reverts, even though the immediately adjacent true maximum succeeds.

This mapping is also consistent with Firelight's published competition result [#58993](https://reports.immunefi.com/firelight/58993-sc-low-incorrect-timestamp-calculation-in-periodattimestamp-leads-to-broken-historical-period), where an incorrect `FirelightVault` view return was accepted under the same listed Low impact.

### Likelihood

The role-free path requires only ordinary vault state and ownership of 0.007071 FXRP:

* the vault has positive supply and assets below the deposit cap;
* `maxMint` is initially executable in the concrete one-to-one state;
* any account transfers 7,071 atomic underlying units directly to the vault, making `totalAssets != totalSupply`;
* the integer boundary falls on a state where the full-cap formula rounds one share atom above the remaining-headroom formula; and
* the receiver is otherwise allowed to mint.

The attacker calls only the underlying ERC-20 `transfer`; the vault cannot require a role or reject the inbound transfer. The PoC proves causality by checking that `mint(maxMint)` succeeds immediately before that transfer and fails immediately afterward. Normal rewards and incentives independently create the same state in production. The arithmetic condition is deterministic and can recur whenever any asset inflow changes the asset/share ratio or the cap changes relative to that ratio.

## Attack Path / Reproduction Path

### Role-free forced path

1. The vault has a 200 FXRP deposit limit.
2. An ordinary staker deposits 100 FXRP and receives 100 stXRP.
3. `maxMint` returns 100 stXRP, and a same-state static `mint(100 stXRP)` succeeds and requires 100 FXRP.
4. An unrelated account transfers 0.007071 FXRP directly to the vault through the standard ERC-20 `transfer` function.
5. Vault assets become 100.007071 FXRP while supply remains 100 stXRP, leaving 99.992929 FXRP of cap headroom.
6. `maxMint` now returns 99.985859 stXRP.
7. `previewMint(99.985859 stXRP)` returns 99.992930 FXRP, one atomic FXRP more than the remaining cap.
8. `mint(maxMint, receiver)` reverts `DepositLimitExceeded`.
9. From the unchanged state, `previewMint(99.985858 stXRP)` returns 99.992928 FXRP and `mint(99.985858 stXRP)` succeeds.

### Normal production path

1. Start from the same 100 FXRP deposit and 200 FXRP cap.
2. The authorized distributor forwards an ordinary 0.007071 FXRP incentive through the scoped `VaultRewardDistributor`.
3. The resulting assets, supply, remaining capacity, advertised maximum, required assets, revert, and true successful maximum are identical to the role-free post-transfer state.

## External Preconditions

* A standard 6-decimal FXRP-compatible vault asset is used.
* Any ordinary account holds 0.007071 FXRP and can transfer it directly to the vault.
* A receiver that is not blocklisted queries and uses `maxMint`.

The authorized distributor is not an external precondition. Its normal incentive path is retained only as independent production reachability.

## Internal Preconditions

* The vault is not paused and has no active incident.
* `totalAssets < depositLimit`.
* The asset-skewed totals make the full-cap floor expression one share atom greater than the remaining-headroom expression.

No administrator, distributor, oracle behavior, token callback, or unsafe user-selected composition is required for the forced path.

## Affected Code

* [`FirelightVault.sol#L380-L392`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L380-L392) documents and implements `maxMint` using `convertToShares(depositLimit) - totalSupply`.
* [`FirelightVault.sol#L422-L429`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L422-L429) derives active `totalAssets` from the underlying-token balance, so direct transfers change conversions without minting shares.
* [`FirelightVault.sol#L682-L704`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L682-L704) ceiling-converts the newly requested shares to assets.
* [`FirelightVault.sol#L976-L989`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989) adds those assets to existing assets and enforces `depositLimit`.
* [`FirelightVault.sol#L1030-L1073`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1030-L1073) defines the snapshot conversion formulas and rounding.
* [`VaultRewardDistributor.sol#L163-L176`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176) supplies the exact ordinary incentive path used by the PoC.

## Recommended Mitigation

Calculate the maximum from the remaining asset capacity using one consistent snapshot:

```solidity
function maxMint(address receiver) public view override returns (uint256) {
    uint256 supply = totalSupply();
    uint256 assets = totalAssets();

    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets >= depositLimit) {
        return 0;
    }

    return _convertToSharesTotals(
        depositLimit - assets,
        supply,
        assets,
        Math.Rounding.Floor
    );
}
```

Add asset-skewed regression invariants:

```
mint(maxMint(receiver), receiver) passes vault acceptance checks when maxMint(receiver) > 0
previewMint(maxMint(receiver)) <= depositLimit - totalAssets
previewMint(maxMint(receiver) + 1) > depositLimit - totalAssets
```

## Proof of Concept

The program requires a [runnable PoC](https://immunefi.com/audit-competition/audit-comp-firelight-1/information/#proof-of-concept-poc-requirements). This report is standalone: the complete test and guarded runner are embedded below. The test uses the exact frozen commit and the repository's normal deployment fixture. That fixture deploys the real frozen `FirelightVault` proxy, the repository-bundled `FAsset` implementation behind its real `FAssetProxy`, and—only for the production scenario—the real frozen `VaultRewardDistributor`. `MockContract` is used solely as the local AssetManager authorization/minting shim; it does not replace FXRP transfer, balance, approval, or vault behavior. No vault, reward, conversion, cap, checkpoint, or revert behavior is mocked or replaced.

### Complete PoC source

Save as `poc/max-mint-cap-rounding/firelight-vault-max-mint-cap-rounding.poc.js`:

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployVault } = require('../../test/setup/fixtures.js')

describe('PoC: maxMint converts the full cap and overstates the executable maximum', function () {
  const FXRP = 10n ** 6n
  const DEPOSIT_LIMIT = 200n * FXRP
  const INITIAL_DEPOSIT = 100n * FXRP
  const REWARD = 7_071n // 0.007071 FXRP

  async function deployExactFlow () {
    const ctx = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })
    const distributor = (await ethers.getSigners())[9]

    const Rewarder = await ethers.getContractFactory('VaultRewardDistributor')
    const rewarder = await upgrades.deployProxy(Rewarder, [
      ctx.firelight_vault.target,
      ctx.deployer.address,
      distributor.address,
      ethers.ZeroAddress,
    ])

    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.CHECKPOINT_ROLE(), rewarder.target)

    return { ...ctx, distributor, rewarder }
  }

  it('reverts mint(maxMint) but accepts exactly one atomic share less in the same state', async function () {
    const ctx = await deployExactFlow()
    const initialStaker = ctx.users[0]
    const nextStaker = ctx.users[1]

    await ctx.utils.mintAndApprove(INITIAL_DEPOSIT, initialStaker)
    await ctx.firelight_vault
      .connect(initialStaker)
      .deposit(INITIAL_DEPOSIT, initialStaker.address)

    await ctx.token_contract.mintTo(ctx.distributor.address, REWARD)
    await ctx.token_contract.connect(ctx.distributor).approve(ctx.rewarder.target, REWARD)
    await ctx.rewarder
      .connect(ctx.distributor)
      .distributeIncentive(REWARD, ethers.id('ordinary-firelight-incentive'))

    await ctx.utils.mintAndApprove(DEPOSIT_LIMIT, nextStaker)

    const supplyBefore = await ctx.firelight_vault.totalSupply()
    const assetsBefore = await ctx.firelight_vault.totalAssets()
    const remainingAssets = DEPOSIT_LIMIT - assetsBefore
    const advertisedMaximum = await ctx.firelight_vault.maxMint(nextStaker.address)
    const assetsForAdvertisedMaximum = await ctx.firelight_vault.previewMint(advertisedMaximum)
    // This is the exact maximum satisfying the cap inequality for this frozen
    // ERC-4626 conversion, derived independently of maxMint().
    const trueMaximum = remainingAssets * (supplyBefore + 1n) / (assetsBefore + 1n)
    const assetsForTrueMaximum = await ctx.firelight_vault.previewMint(trueMaximum)

    expect(supplyBefore).to.equal(INITIAL_DEPOSIT)
    expect(assetsBefore).to.equal(INITIAL_DEPOSIT + REWARD)
    expect(advertisedMaximum).to.equal(99_985_859n)
    expect(trueMaximum).to.equal(99_985_858n)
    expect(advertisedMaximum).to.equal(trueMaximum + 1n)
    expect(assetsForAdvertisedMaximum).to.be.greaterThan(remainingAssets)
    expect(assetsForTrueMaximum).to.be.at.most(remainingAssets)

    await expect(
      ctx.firelight_vault.connect(nextStaker).mint(advertisedMaximum, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'DepositLimitExceeded')

    expect(await ctx.firelight_vault.totalSupply()).to.equal(supplyBefore)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(assetsBefore)
    expect(await ctx.firelight_vault.balanceOf(nextStaker.address)).to.equal(0n)

    await expect(
      ctx.firelight_vault.connect(nextStaker).mint(trueMaximum, nextStaker.address),
    )
      .to.emit(ctx.firelight_vault, 'Deposit')
      .withArgs(nextStaker.address, nextStaker.address, assetsForTrueMaximum, trueMaximum)

    expect(await ctx.firelight_vault.balanceOf(nextStaker.address)).to.equal(trueMaximum)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(
      assetsBefore + assetsForTrueMaximum,
    )

    console.log(JSON.stringify({
      depositLimitAtomic: DEPOSIT_LIMIT.toString(),
      totalAssetsBeforeAtomic: assetsBefore.toString(),
      totalSupplyBeforeAtomic: supplyBefore.toString(),
      remainingCapacityAtomic: remainingAssets.toString(),
      advertisedMaxMintAtomic: advertisedMaximum.toString(),
      assetsRequiredAtAdvertisedMaximumAtomic: assetsForAdvertisedMaximum.toString(),
      resultAtAdvertisedMaximum: 'DepositLimitExceeded',
      trueMaxMintAtomic: trueMaximum.toString(),
      assetsRequiredAtTrueMaximumAtomic: assetsForTrueMaximum.toString(),
      resultAtTrueMaximum: 'success',
    }))
  })

  it('lets an unprivileged account force the mismatch with a direct FXRP transfer', async function () {
    const ctx = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })
    const initialStaker = ctx.users[0]
    const nextStaker = ctx.users[1]
    const attacker = ctx.users[2]

    await ctx.utils.mintAndApprove(INITIAL_DEPOSIT, initialStaker)
    await ctx.firelight_vault
      .connect(initialStaker)
      .deposit(INITIAL_DEPOSIT, initialStaker.address)
    await ctx.utils.mintAndApprove(DEPOSIT_LIMIT, nextStaker)

    const preAttackMaximum = await ctx.firelight_vault.maxMint(nextStaker.address)
    expect(preAttackMaximum).to.equal(INITIAL_DEPOSIT)
    expect(
      await ctx.firelight_vault
        .connect(nextStaker)
        .mint.staticCall(preAttackMaximum, nextStaker.address),
    ).to.equal(INITIAL_DEPOSIT)

    await ctx.token_contract.mintTo(attacker.address, REWARD)
    await ctx.token_contract.connect(attacker).transfer(ctx.firelight_vault.target, REWARD)

    const supplyAfterDonation = await ctx.firelight_vault.totalSupply()
    const assetsAfterDonation = await ctx.firelight_vault.totalAssets()
    const remainingAssets = DEPOSIT_LIMIT - assetsAfterDonation
    const advertisedMaximum = await ctx.firelight_vault.maxMint(nextStaker.address)
    const assetsForAdvertisedMaximum = await ctx.firelight_vault.previewMint(advertisedMaximum)
    const trueMaximum = remainingAssets * (supplyAfterDonation + 1n) / (assetsAfterDonation + 1n)
    const assetsForTrueMaximum = await ctx.firelight_vault.previewMint(trueMaximum)

    expect(await ctx.token_contract.balanceOf(attacker.address)).to.equal(0n)
    expect(advertisedMaximum).to.equal(99_985_859n)
    expect(trueMaximum).to.equal(99_985_858n)
    expect(advertisedMaximum).to.equal(trueMaximum + 1n)
    expect(assetsForAdvertisedMaximum).to.be.greaterThan(remainingAssets)
    expect(assetsForTrueMaximum).to.be.at.most(remainingAssets)

    await expect(
      ctx.firelight_vault.connect(nextStaker).mint(advertisedMaximum, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'DepositLimitExceeded')

    expect(await ctx.firelight_vault.totalSupply()).to.equal(supplyAfterDonation)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(assetsAfterDonation)
    expect(await ctx.firelight_vault.balanceOf(nextStaker.address)).to.equal(0n)

    await expect(
      ctx.firelight_vault.connect(nextStaker).mint(trueMaximum, nextStaker.address),
    )
      .to.emit(ctx.firelight_vault, 'Deposit')
      .withArgs(nextStaker.address, nextStaker.address, assetsForTrueMaximum, trueMaximum)

    console.log(JSON.stringify({
      roleFreeScenario: true,
      attackerDonationAtomic: REWARD.toString(),
      preAttackMaxMintAtomic: preAttackMaximum.toString(),
      preAttackMintAtMaximum: 'success',
      postAttackMaxMintAtomic: advertisedMaximum.toString(),
      assetsRequiredAtAdvertisedMaximumAtomic: assetsForAdvertisedMaximum.toString(),
      remainingCapacityAtomic: remainingAssets.toString(),
      postAttackMintAtMaximum: 'DepositLimitExceeded',
      trueMaxMintAtomic: trueMaximum.toString(),
      trueMaximumMint: 'success',
    }))
  })
})
```

Save as `poc/max-mint-cap-rounding/run.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
REPO_DIR="$(cd -- "${SCRIPT_DIR}/../.." && pwd)"

cd "${REPO_DIR}"

EXPECTED_COMMIT="42f9ea5e43d88b35197b901acc2a8c24b314fa62"
EXPECTED_VAULT_SHA256="778608250e5893fe908c6ebf7255cb559094066157f4e70b88950c94e509ab5f"
EXPECTED_DISTRIBUTOR_SHA256="bd3287bbe2cec8d6f56c9cde2a8c2f8697524071b3b1acdb6d335d0ceb02c1aa"
EXPECTED_FASSET_SHA256="75194713c07ea3c99a62f22a60f40729bb884ba1c18f3fce4a725595dddcb380"
EXPECTED_FASSET_PROXY_SHA256="87d561360f1fd9ec192aa3dc8d17b9c317477d40b1e4c05ac8eb7eecbb27a958"

sha256_file() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | awk '{print $1}'
  elif command -v shasum >/dev/null 2>&1; then
    shasum -a 256 "$1" | awk '{print $1}'
  else
    echo "sha256sum or shasum is required" >&2
    exit 1
  fi
}

[[ "$(git rev-parse HEAD)" == "${EXPECTED_COMMIT}" ]] || {
  echo "unexpected repository commit" >&2
  exit 1
}
[[ "$(sha256_file contracts/core/FirelightVault.sol)" == "${EXPECTED_VAULT_SHA256}" ]] || {
  echo "unexpected FirelightVault.sol hash" >&2
  exit 1
}
[[ "$(sha256_file contracts/core/VaultRewardDistributor.sol)" == "${EXPECTED_DISTRIBUTOR_SHA256}" ]] || {
  echo "unexpected VaultRewardDistributor.sol hash" >&2
  exit 1
}
[[ "$(sha256_file contracts/fasset/fassetToken/implementation/FAsset.sol)" == "${EXPECTED_FASSET_SHA256}" ]] || {
  echo "unexpected FAsset.sol hash" >&2
  exit 1
}
[[ "$(sha256_file contracts/fasset/fassetToken/implementation/FAssetProxy.sol)" == "${EXPECTED_FASSET_PROXY_SHA256}" ]] || {
  echo "unexpected FAssetProxy.sol hash" >&2
  exit 1
}

mkdir -p "${SCRIPT_DIR}/evidence"
npx hardhat test poc/max-mint-cap-rounding/firelight-vault-max-mint-cap-rounding.poc.js \
  | tee "${SCRIPT_DIR}/evidence/local-e2e.log"
```

The exact embedded files have these SHA-256 hashes:

```
f44bff7f6725199699330d9b38baf958a0f7303962484e830f2f117bcabb9215  firelight-vault-max-mint-cap-rounding.poc.js
db0084990d0bf115c5884b4f0d942669711e40af89e6a83697aa3d5a7ba5c357  run.sh
```

### Standalone clean reproduction

Requirements: Git, Node.js, and npm. No `.env` file is required for local tests.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
mkdir -p poc/max-mint-cap-rounding
```

Save the two complete code blocks above at these exact repository-relative paths:

```
poc/max-mint-cap-rounding/firelight-vault-max-mint-cap-rounding.poc.js
poc/max-mint-cap-rounding/run.sh
```

Then run:

```bash
chmod +x poc/max-mint-cap-rounding/run.sh
./poc/max-mint-cap-rounding/run.sh
```

The runner rejects the wrong Git commit or any modification to the vulnerable vault, production distributor, bundled FAsset implementation, or bundled FAsset proxy. It then runs only the two targeted tests on Hardhat's local chain and preserves the complete console output at `poc/max-mint-cap-rounding/evidence/local-e2e.log`.

### Captured output

```
PoC: maxMint converts the full cap and overstates the executable maximum
{"depositLimitAtomic":"200000000","totalAssetsBeforeAtomic":"100007071","totalSupplyBeforeAtomic":"100000000","remainingCapacityAtomic":"99992929","advertisedMaxMintAtomic":"99985859","assetsRequiredAtAdvertisedMaximumAtomic":"99992930","resultAtAdvertisedMaximum":"DepositLimitExceeded","trueMaxMintAtomic":"99985858","assetsRequiredAtTrueMaximumAtomic":"99992928","resultAtTrueMaximum":"success"}
  ✔ reverts mint(maxMint) but accepts exactly one atomic share less in the same state
{"roleFreeScenario":true,"attackerDonationAtomic":"7071","preAttackMaxMintAtomic":"100000000","preAttackMintAtMaximum":"success","postAttackMaxMintAtomic":"99985859","assetsRequiredAtAdvertisedMaximumAtomic":"99992930","remainingCapacityAtomic":"99992929","postAttackMintAtMaximum":"DepositLimitExceeded","trueMaxMintAtomic":"99985858","trueMaximumMint":"success"}
  ✔ lets an unprivileged account force the mismatch with a direct FXRP transfer

2 passing
```

### Determinism

Verified environment: Node.js `v24.18.0`, npm `11.16.0`, Hardhat `2.22.19`. The test ran against commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`; the runner pins the decisive contract hashes shown above. The claimed behavior consists entirely of deterministic EVM state transitions in the frozen contracts: ERC-20 transfer, ERC-4626 conversion rounding, cap checks, and exact custom-error reverts. It does not claim any network, consensus, validator, RPC, mempool, or cross-node behavior. Hardhat's in-process chain therefore supplies the complete execution environment; Docker, a public fork, and a multi-node cluster are neither required nor probative for this finding.

The role-free test fails unless all of the following hold:

* before the unrelated transfer, `maxMint` is exactly 100,000,000 atomic stXRP and a static call proves `mint(maxMint)` succeeds;
* an account with no Firelight role transfers exactly 7,071 atomic FXRP directly to the vault and spends its full transferred balance;
* the transfer changes assets but not supply;
* afterward, remaining cap capacity is exactly 99,992,929 atomic FXRP and `maxMint` is exactly 99,985,859 atomic stXRP;
* `previewMint(maxMint)` requires 99,992,930 atomic FXRP;
* `mint(maxMint)` reverts with the target's `DepositLimitExceeded` error and changes no state;
* the independently derived true maximum is exactly 99,985,858 atomic stXRP; and
* `mint(trueMaximum)` succeeds and emits the exact `Deposit` event.

The production-path test independently fails unless a real `VaultRewardDistributor` forwards exactly 7,071 atomic FXRP after a 100,000,000-atomic deposit, after which the same post-transfer totals, advertised maximum, required assets, exact revert, true maximum, and successful mint assertions all hold.
