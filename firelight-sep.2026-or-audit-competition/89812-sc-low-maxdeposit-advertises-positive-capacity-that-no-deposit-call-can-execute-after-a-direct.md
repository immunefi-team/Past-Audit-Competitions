# 89812 sc low maxdeposit advertises positive capacity that no deposit call can execute after a direct asset transfer

**Submitted on Aug 21st 2026 at 10:16:42 UTC by @legat for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89812
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`FirelightVault.maxDeposit` returns the raw asset headroom `depositLimit - totalAssets`. It does not check whether depositing that headroom would mint at least one share. Once assets enter the vault without minting shares, a positive headroom can convert to zero shares. Phase 2 correctly rejects such a deposit with `InvalidAmount`, while every larger deposit exceeds the cap and reverts with `DepositLimitExceeded`.

Any underlying-token holder can force this state without a Firelight role. The PoC first proves that a two-atomic-asset `maxDeposit` is executable, then has an unrelated account transfer one atomic FXRP directly to the vault. That permissionless transfer alone changes `maxDeposit` to one atomic FXRP, makes its preview zero, and makes every positive deposit revert. No distributor, administrator, oracle, callback, or victim-selected composition is required.

The result is a state in which `maxDeposit(receiver) > 0`, but there is no positive asset amount up to that advertised maximum that `deposit` will accept. This contradicts both Firelight's stated ERC-4626 integration semantics and ERC-4626's requirement that `maxDeposit` not exceed the amount a same-state `deposit` can accept.

The runnable PoC deploys the exact frozen contracts and proves both the role-free transition and the normal production reward path:

```
role-free pre-transfer maxDeposit       = 0.000002 FXRP
deposit(pre-transfer maxDeposit)        = executable
unprivileged direct transfer            = 0.000001 FXRP
role-free post-transfer maxDeposit      = 0.000001 FXRP
previewDeposit(post-transfer maximum)   = 0 shares
deposit(post-transfer maximum)          = InvalidAmount
every larger positive deposit           = DepositLimitExceeded
true executable positive maximum        = 0

production deposit                      = 100.000000 FXRP
ordinary distributed incentive          = 99.999999 FXRP
production-path maxDeposit              = 0.000001 FXRP
production-path deposit(maxDeposit)      = InvalidAmount
```

## Description

### Intended behavior

Firelight documents the vault as implementing the [ERC-4626 tokenized-vault standard](https://docs.firelight.finance/core-concepts/vault-architecture#erc-4626-foundation), emphasizing standardized accounting and predictable integration. The official [`maxDeposit` specification](https://eips.ethereum.org/EIPS/eip-4626#maxdeposit) requires the getter to return an amount that `deposit` would allow without reverting and says it must not be higher than the actual accepted maximum.

The frozen contract's own NatSpec likewise promises:

```solidity
/**
 * @notice Returns the maximum amount of the underlying asset that can be deposited into the Vault for the receiver,
 * through a deposit call.
 */
```

Therefore, if no positive deposit can pass all same-state vault checks, `maxDeposit` must return zero. ERC-4626 expressly allows the getter to underestimate when exact rounding is difficult; it does not allow an overestimate that reverts.

### Actual behavior

The getter checks the receiver, pause state, incident state, and whether current assets already exceed the cap. Otherwise it returns the raw cap remainder:

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

`deposit`, however, applies one additional acceptance condition. It converts assets to shares using floor rounding, then rejects a zero-share result:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    assets,
    true,
    Math.Rounding.Floor
);
if (shares == 0) revert InvalidAmount();

_depositFunds(_msgSender(), receiver, assets, shares, _totalSupply, _totalAssets);
```

For the frozen vault's zero decimals offset, if `A = totalAssets`, `S = totalSupply`, and `x` is the asset amount, the conversion is:

```
shares(x) = floor(x * (S + 1) / (A + 1))
minimum assets that mint a positive share = ceil((A + 1) / (S + 1))
```

Let `H = depositLimit - A`. Whenever:

```
0 < H < ceil((A + 1) / (S + 1))
```

`maxDeposit` returns `H`, but every amount in `[1, H]` converts to zero shares and reverts `InvalidAmount`. Every amount greater than `H` makes `_totalAssets > depositLimit` and reverts `DepositLimitExceeded`. The actual executable positive maximum is therefore zero.

`totalAssets()` is derived from the vault's underlying-token balance, less pending withdrawals. A standard ERC-20 transfer to the vault therefore increases `A` without minting shares and requires no Firelight function or role. The role-free PoC starts from an executable boundary and shows that one atomic direct transfer is the only state change needed to create the mismatch.

The same state class also arises through the scoped, intended production flow: `VaultRewardDistributor.distributeIncentive` transfers ordinary vault assets to `FirelightVault` and checkpoints `totalAssets` without minting shares. This is an independent normal-operation scenario, not a precondition for exploitation.

## Root Cause

[`maxDeposit`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L371-L377) models only the deposit cap. It omits the zero-share acceptance predicate enforced by [`deposit`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L671).

This mismatch becomes permissionlessly reachable because [`totalAssets`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L422-L429) ultimately reads the vault's ERC-20 balance. A direct transfer grows the conversion denominator independently of share supply. The intended [`distributeIncentive`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176) and [`distributeRewards`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L132-L152) paths provide normal production reachability as well.

## Risk

### Impact

This is Smart Contract **Low**, the listed impact **“Contract fails to deliver promised returns, but doesn't lose value.”**

The contract promises a standards-compatible maximum-deposit query but returns a positive value that the corresponding state-changing function cannot execute. A wallet, aggregator, strategy, or user following the advertised ERC-4626 capacity receives an unusable quote and a reverting deposit. Firelight specifically promises that integrations can rely on standard vault semantics.

This mapping is also consistent with Firelight's published competition result [#58993](https://reports.immunefi.com/firelight/58993-sc-low-incorrect-timestamp-calculation-in-periodattimestamp-leads-to-broken-historical-period), where an incorrect `FirelightVault` view return was accepted under the same listed Low impact.

### Likelihood

The role-free path requires only ordinary vault state and ownership of one atomic underlying-token unit:

* the vault has positive supply;
* assets are two atomic units below the cap in the concrete PoC, where the advertised boundary is initially executable;
* any account transfers one atomic underlying asset directly to the vault;
* the remaining cap headroom is positive but below the minimum asset amount that floors to one share; and
* the receiver is otherwise allowed to deposit.

The attacker calls only the underlying ERC-20 `transfer`; the vault cannot require a role or reject the inbound transfer. The PoC proves causality by checking that `deposit(maxDeposit)` succeeds immediately before that transfer and fails immediately afterward. Normal rewards and incentives independently create the same class of state in production.

## Attack Path / Reproduction Path

### Role-free forced path

1. The vault has a 200 FXRP deposit limit.
2. An ordinary staker deposits 199.999998 FXRP, creating equal asset and share totals.
3. `maxDeposit` returns two atomic FXRP, and a same-state static `deposit(2)` succeeds and returns two shares.
4. An unrelated account transfers one atomic FXRP directly to the vault through the standard ERC-20 `transfer` function.
5. Assets become 199.999999 FXRP while supply remains 199.999998 stXRP.
6. `maxDeposit` now returns one atomic FXRP, but `previewDeposit(1)` returns zero shares.
7. `deposit(1)` reaches the Phase 2 dust guard and reverts `InvalidAmount`.
8. `deposit(2)` and every larger amount exceed the 200 FXRP cap and revert `DepositLimitExceeded`.
9. The permissionless one-atom transfer has changed an executable advertised maximum into a positive maximum that no deposit can execute.

### Normal production path

1. An ordinary staker deposits 100 FXRP under the same 200 FXRP cap.
2. The authorized distributor forwards an ordinary 99.999999 FXRP incentive through the scoped `VaultRewardDistributor`.
3. Assets become 199.999999 FXRP while supply remains 100 stXRP.
4. `maxDeposit` returns one atomic FXRP, its preview returns zero, `deposit(1)` reverts `InvalidAmount`, and every larger deposit exceeds the cap.

## External Preconditions

* A standard 6-decimal FXRP-compatible vault asset is used.
* Any ordinary account holds one atomic unit of the underlying token and can transfer it directly to the vault.
* A receiver that is not blocklisted queries and uses `maxDeposit`.

The authorized distributor is not an external precondition. Its normal incentive path is retained only as independent production reachability.

## Internal Preconditions

* The vault is not paused and has no active incident.
* `totalSupply > 0` and an asset inflow without share minting has raised the asset-per-share ratio.
* `0 < depositLimit - totalAssets < ceil((totalAssets + 1) / (totalSupply + 1))`.

No administrator, distributor, oracle behavior, token callback, or unsafe user-selected composition is required for the forced path.

## Affected Code

* [`FirelightVault.sol#L365-L377`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L365-L377) documents and implements `maxDeposit` but returns only cap headroom.
* [`FirelightVault.sol#L422-L429`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L422-L429) derives active `totalAssets` from the underlying-token balance, so direct transfers change conversions without minting shares.
* [`FirelightVault.sol#L646-L671`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L671) floors the conversion and rejects `shares == 0`.
* [`FirelightVault.sol#L976-L989`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L976-L989) rejects any deposit whose post-operation assets exceed `depositLimit`.
* [`FirelightVault.sol#L1030-L1073`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1030-L1073) defines the snapshot conversion formulas and rounding.
* [`VaultRewardDistributor.sol#L163-L176`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/VaultRewardDistributor.sol#L163-L176) supplies the exact ordinary incentive path used by the PoC.

## Recommended Mitigation

Make `maxDeposit` apply the same conversion acceptance predicate as `deposit`, using one consistent snapshot of supply and assets:

```solidity
function maxDeposit(address receiver) public view override returns (uint256) {
    uint256 supply = totalSupply();
    uint256 assets = totalAssets();

    if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets >= depositLimit) {
        return 0;
    }

    uint256 headroom = depositLimit - assets;
    if (_convertToSharesTotals(headroom, supply, assets, Math.Rounding.Floor) == 0) {
        return 0;
    }

    return headroom;
}
```

Add a regression invariant over asset-skewed states:

```
maxDeposit(receiver) == 0
    OR deposit(maxDeposit(receiver), receiver) passes vault acceptance checks
```

The getter may conservatively return zero at this dust boundary, as ERC-4626 allows underestimation.

## Proof of Concept

The program requires a [runnable PoC](https://immunefi.com/audit-competition/audit-comp-firelight-1/information/#proof-of-concept-poc-requirements). This report is standalone: the complete test and guarded runner are embedded below. The test uses the exact frozen commit and the repository's normal deployment fixture. That fixture deploys the real frozen `FirelightVault` proxy, the repository-bundled `FAsset` implementation behind its real `FAssetProxy`, and—only for the production scenario—the real frozen `VaultRewardDistributor`. `MockContract` is used solely as the local AssetManager authorization/minting shim; it does not replace FXRP transfer, balance, approval, or vault behavior. No vault, reward, conversion, cap, checkpoint, or revert behavior is mocked or replaced.

### Complete PoC source

Save as `poc/max-deposit-zero-share/firelight-vault-max-deposit-zero-share.poc.js`:

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployVault } = require('../../test/setup/fixtures.js')

describe('PoC: maxDeposit advertises a cap remainder that no positive deposit can use', function () {
  const FXRP = 10n ** 6n
  const DEPOSIT_LIMIT = 200n * FXRP
  const INITIAL_DEPOSIT = 100n * FXRP
  const REWARD = 100n * FXRP - 1n

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

  it('returns one atomic FXRP although deposit(1) and every larger amount revert', async function () {
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

    await ctx.utils.mintAndApprove(2n, nextStaker)

    const supplyBefore = await ctx.firelight_vault.totalSupply()
    const assetsBefore = await ctx.firelight_vault.totalAssets()
    const maxDeposit = await ctx.firelight_vault.maxDeposit(nextStaker.address)
    const previewAtMaximum = await ctx.firelight_vault.previewDeposit(maxDeposit)

    expect(supplyBefore).to.equal(INITIAL_DEPOSIT)
    expect(assetsBefore).to.equal(DEPOSIT_LIMIT - 1n)
    expect(maxDeposit).to.equal(1n)
    expect(previewAtMaximum).to.equal(0n)

    await expect(
      ctx.firelight_vault.connect(nextStaker).deposit(maxDeposit, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'InvalidAmount')

    await expect(
      ctx.firelight_vault.connect(nextStaker).deposit(maxDeposit + 1n, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'DepositLimitExceeded')

    expect(await ctx.firelight_vault.totalSupply()).to.equal(supplyBefore)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(assetsBefore)
    expect(await ctx.firelight_vault.balanceOf(nextStaker.address)).to.equal(0n)

    console.log(JSON.stringify({
      depositLimitAtomic: DEPOSIT_LIMIT.toString(),
      totalAssetsAtomic: assetsBefore.toString(),
      totalSupplyAtomic: supplyBefore.toString(),
      advertisedMaxDepositAtomic: maxDeposit.toString(),
      previewSharesAtAdvertisedMaximum: previewAtMaximum.toString(),
      depositAtAdvertisedMaximum: 'InvalidAmount',
      everyLargerDeposit: 'DepositLimitExceeded',
      trueExecutablePositiveMaximumAtomic: '0',
    }))
  })

  it('lets an unprivileged account force the mismatch with one atomic FXRP', async function () {
    const ctx = await deployVault({ initial_deposit_limit: DEPOSIT_LIMIT })
    const initialStaker = ctx.users[0]
    const nextStaker = ctx.users[1]
    const attacker = ctx.users[2]
    const initialDeposit = DEPOSIT_LIMIT - 2n

    await ctx.utils.mintAndApprove(initialDeposit, initialStaker)
    await ctx.firelight_vault
      .connect(initialStaker)
      .deposit(initialDeposit, initialStaker.address)
    await ctx.utils.mintAndApprove(2n, nextStaker)

    const preAttackMaximum = await ctx.firelight_vault.maxDeposit(nextStaker.address)
    expect(preAttackMaximum).to.equal(2n)
    expect(
      await ctx.firelight_vault
        .connect(nextStaker)
        .deposit.staticCall(preAttackMaximum, nextStaker.address),
    ).to.equal(2n)

    await ctx.token_contract.mintTo(attacker.address, 1n)
    await ctx.token_contract.connect(attacker).transfer(ctx.firelight_vault.target, 1n)

    const supplyAfterDonation = await ctx.firelight_vault.totalSupply()
    const assetsAfterDonation = await ctx.firelight_vault.totalAssets()
    const postAttackMaximum = await ctx.firelight_vault.maxDeposit(nextStaker.address)
    const postAttackPreview = await ctx.firelight_vault.previewDeposit(postAttackMaximum)

    expect(supplyAfterDonation).to.equal(initialDeposit)
    expect(assetsAfterDonation).to.equal(DEPOSIT_LIMIT - 1n)
    expect(await ctx.token_contract.balanceOf(attacker.address)).to.equal(0n)
    expect(postAttackMaximum).to.equal(1n)
    expect(postAttackPreview).to.equal(0n)

    await expect(
      ctx.firelight_vault.connect(nextStaker).deposit(postAttackMaximum, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'InvalidAmount')

    await expect(
      ctx.firelight_vault.connect(nextStaker).deposit(postAttackMaximum + 1n, nextStaker.address),
    ).to.be.revertedWithCustomError(ctx.firelight_vault, 'DepositLimitExceeded')

    expect(await ctx.firelight_vault.totalSupply()).to.equal(supplyAfterDonation)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(assetsAfterDonation)
    expect(await ctx.firelight_vault.balanceOf(nextStaker.address)).to.equal(0n)

    console.log(JSON.stringify({
      roleFreeScenario: true,
      attackerDonationAtomic: '1',
      preAttackMaxDepositAtomic: preAttackMaximum.toString(),
      preAttackDepositAtMaximum: 'success',
      postAttackMaxDepositAtomic: postAttackMaximum.toString(),
      postAttackPreviewShares: postAttackPreview.toString(),
      postAttackDepositAtMaximum: 'InvalidAmount',
      everyLargerFeasibleDeposit: 'DepositLimitExceeded',
      trueExecutablePositiveMaximumAtomic: '0',
    }))
  })
})
```

Save as `poc/max-deposit-zero-share/run.sh`:

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
npx hardhat test poc/max-deposit-zero-share/firelight-vault-max-deposit-zero-share.poc.js \
  | tee "${SCRIPT_DIR}/evidence/local-e2e.log"
```

The exact embedded files have these SHA-256 hashes:

```
5c1ada4658b0bc54788a79564a0b6f9bb3470a65dbe65ae17288e9b4d58752b5  firelight-vault-max-deposit-zero-share.poc.js
6c67c985f3de96e8b6b4e8d8a5a3ae989c43202bb61f06ff88fa8186ccbe4319  run.sh
```

### Standalone clean reproduction

Requirements: Git, Node.js, and npm. No `.env` file is required for local tests.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
mkdir -p poc/max-deposit-zero-share
```

Save the two complete code blocks above at these exact repository-relative paths:

```
poc/max-deposit-zero-share/firelight-vault-max-deposit-zero-share.poc.js
poc/max-deposit-zero-share/run.sh
```

Then run:

```bash
chmod +x poc/max-deposit-zero-share/run.sh
./poc/max-deposit-zero-share/run.sh
```

The runner rejects the wrong Git commit or any modification to the vulnerable vault, production distributor, bundled FAsset implementation, or bundled FAsset proxy. It then runs only the two targeted tests on Hardhat's local chain and preserves the complete console output at `poc/max-deposit-zero-share/evidence/local-e2e.log`.

### Captured output

```
PoC: maxDeposit advertises a cap remainder that no positive deposit can use
{"depositLimitAtomic":"200000000","totalAssetsAtomic":"199999999","totalSupplyAtomic":"100000000","advertisedMaxDepositAtomic":"1","previewSharesAtAdvertisedMaximum":"0","depositAtAdvertisedMaximum":"InvalidAmount","everyLargerDeposit":"DepositLimitExceeded","trueExecutablePositiveMaximumAtomic":"0"}
  ✔ returns one atomic FXRP although deposit(1) and every larger amount revert
{"roleFreeScenario":true,"attackerDonationAtomic":"1","preAttackMaxDepositAtomic":"2","preAttackDepositAtMaximum":"success","postAttackMaxDepositAtomic":"1","postAttackPreviewShares":"0","postAttackDepositAtMaximum":"InvalidAmount","everyLargerFeasibleDeposit":"DepositLimitExceeded","trueExecutablePositiveMaximumAtomic":"0"}
  ✔ lets an unprivileged account force the mismatch with one atomic FXRP

2 passing
```

### Determinism

Verified environment: Node.js `v24.18.0`, npm `11.16.0`, Hardhat `2.22.19`. The test ran against commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`; the runner pins the decisive contract hashes shown above. The claimed behavior consists entirely of deterministic EVM state transitions in the frozen contracts: ERC-20 transfer, ERC-4626 accounting, cap checks, and exact custom-error reverts. It does not claim any network, consensus, validator, RPC, mempool, or cross-node behavior. Hardhat's in-process chain therefore supplies the complete execution environment; Docker, a public fork, and a multi-node cluster are neither required nor probative for this finding.

The role-free test fails unless all of the following hold:

* before the unrelated transfer, `maxDeposit` is exactly two atomic FXRP and a static call proves `deposit(2)` succeeds;
* an account with no Firelight role transfers exactly one atomic FXRP directly to the vault and spends its full one-atom balance;
* the transfer changes assets but not supply;
* afterward, `maxDeposit` is exactly one atomic FXRP and `previewDeposit(1)` is zero;
* `deposit(1)` reverts with the target's `InvalidAmount` error;
* `deposit(2)` reverts with the target's `DepositLimitExceeded` error; and
* neither failed call changes vault assets, supply, or the second staker's share balance.

The production-path test independently fails unless a real `VaultRewardDistributor` forwards exactly 99,999,999 atomic FXRP after a 100,000,000-atomic deposit, after which the same one-atom advertised maximum, zero preview, two exact revert reasons, and unchanged-state assertions all hold.
