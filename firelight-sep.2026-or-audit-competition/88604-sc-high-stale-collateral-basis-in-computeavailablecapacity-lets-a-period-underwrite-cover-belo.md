# 88604 sc high stale collateral basis in computeavailablecapacity lets a period underwrite cover below the protocol s own minimum backing

**Submitted on Aug 14th 2026 at 04:25:38 UTC by @Dead\_Aizen for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88604
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/ICoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`_computeAvailableCapacity` values the vault leg of collateral at `totalAssetsAt(currentPeriodStart())`. `FirelightVault.payout()` records its collateral-reducing checkpoint at `Time.timestamp()`. A payout executed after a period has begun therefore never reaches that period's capacity basis.

Because an incident captured in period `P` stays payable through `P+1`, a claim on period `P`'s cover can be settled during `P+1`, draining collateral that `P+1`'s capacity was already sized against. The cover being sold in `P+1` is not the cover being claimed against — it is new cover for different buyers.

The snapshot itself is deliberate and correct for its intended case. Withdrawal requests raised during a period are assigned to `withdrawAssets[period + 1]`, and `payout()` draws from that bucket, so those assets remain exposed to the period's incidents and must keep counting toward capacity. The snapshot preserves them correctly. It does not distinguish assets that are leaving but still exposed from assets that have already left via payout and are exposed to nothing.

**Severity / impact:** Critical — Protocol insolvency **Commit:** `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (`v1_audit_ready`) **Files:** `contracts/core/CoverOrderAllocator.sol`, `contracts/core/FirelightVault.sol`

## Root cause

`CoverOrderAllocator.sol:711`

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`FirelightVault.sol:456`

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

`upperLookupRecent` returns the latest checkpoint at or before the key. `FirelightVault.sol:952`, inside `payout`:

```solidity
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

That key is strictly greater than `currentPeriodStart()` for any payout after the period boundary, so it is invisible to every capacity computation for the period in flight.

Within the same function the first-loss buffer is read live:

```solidity
config.firstLossBufferToken.balanceOf(config.firstLossBuffer)
```

The two collateral legs use different time bases. If period-start valuation expressed a solvency policy, it would apply to both.

## Invariants broken

`CoverOrderAllocator.sol:79-82`, above `MAX_LEVERAGE_FACTOR`:

> Real backing behind sold cover is `minCAR / (effectiveLeverage × (1 + tolerance))`, so bounding the ratio floors it at `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` regardless of the configured `minCAR`.

`MAX_LEVERAGE_FACTOR = 5`, `MAX_DIVERGENCE_TOLERANCE_BPS = 1000`. Asserted floor: 18.18%. Measured: **12.5%**.

`MIN_CAR_BPS = 12_000` (L84, enforced L662) makes a 1.20 CAR a code invariant. Measured: **0.30**.

`CoverOrderAllocator.sol:75-78`, above `MAX_DIVERGENCE_TOLERANCE_BPS`:

> Kept small: the tolerance is accepted under-collateralization vs the CAR target, only meant to absorb the price/FLB drift between the off-chain matcher snapshot and commit inclusion.

The protocol caps acceptable under-collateralization at 10%. This produces roughly 31%, from a source the tolerance does not cover.

## Sequence

Every role acts correctly. No role exceeds its authority.

1. `P0` — stakers deposit 2,000. A cover order is created (orders target `currentPeriod() + 1`).
2. `P1` — `ALLOCATOR_ROLE` commits the matching tree; the order settles for 1,500 of cover. An incident is captured in `P1`.
3. `P1` — `CURATOR_ROLE` confirms and files an assessment for the full allocated cover. Submitted.
4. `P2` — `ASSESSMENT_APPROVER_ROLE` approves. `IncidentManager` calls `FirelightVault.payout()`; 1,500 leaves the vault. Collateral is now **500**.
5. `P2` — `commitAllocation(P2)` reads `totalAssetsAt(startP2)`, still **2,000**, and accepts 4,000. An order created in `P1` targeting `P2` settles for **3,000 of cover**, minting a CoverNFT.

Ordering in steps 4 and 5 is not load-bearing. Committing before the payout ends in the same state: cover sold against collateral that has since left.

## Proof of concept

A single self-contained file, `test/integration/firelight-capacity-poc.js`. It depends only on the repository's own helpers (`test/setup/fixtures.js`, `test/oracle/fixtures.js`) and needs no other new file.

Production contracts in the path, unmodified: `FirelightVault`, `CoverOrderAllocator`, `IncidentManager`, `CoverNFT`, `FtsoChainlinkAdapter` — five of the fourteen in-scope assets. Stubbed: the ERC20 tokens and the Flare FTSO/registry services, using the repository's own `contracts/test/` mocks and its own `plantFlareMocks()` helper, which stamps `MockFlareRegistry` at the canonical registry address the adapter hard-codes. No mock contract was authored for this report and nothing in the defective path is mocked.

```
npx hardhat test test/integration/firelight-capacity-poc.js
```

```
P0  staked ....................... 2000
P1  cover settled ................ 1500
P2  capacity basis at start ...... 2000
P2  IncidentManager paid out ..... 1500 -> payout receiver
P2  live collateral now .......... 500
P2  capacity basis STILL ......... 2000   <-- drawdown invisible
P2  after checkpointTotalAssets .. 2000   <-- no operator fix
P2  commit of 4000+1 wei ......... reverted MatchingCapacityOverflow (4000 is a hard ceiling)
P2  commit of 4000 ............... ACCEPTED
P2  fresh cover SETTLED .......... 3000 against 500 collateral

CAR vs stale basis:  12000 bps (floor 12000)
CAR vs real basis:   3000 bps  <-- MIN_CAR_BPS floor breached
real backing:        1250 bps
documented floor:    1818 bps  <-- code comment invariant breached
```

Asserted, in order: `totalAssetsAt(startP2) == 2000e6` before the payout; the payout receiver gains exactly `1500e6` and `totalAssets()` falls to `500e6`; `totalAssetsAt(startP2)` is still `2000e6` afterwards, and remains so after `checkpointTotalAssets()`; a commitment of capacity + 1 wei reverts `MatchingCapacityOverflow`, establishing 4,000 as a hard ceiling rather than slack; the `P2` order settles for `3000e18` with `totalAssets()` still `500e6`; and both floors, computed from the live `getCapacityConfigAt(P2)` values.

## No mitigation exists

`checkpointTotalAssets()` (`FirelightVault.sol:523`) pushes at `Time.timestamp()`. Since `totalAssetsAt(startP2)` reads only checkpoints at or before `startP2`, a checkpoint written mid-period cannot repair the basis. The PoC grants `CHECKPOINT_ROLE`, calls it, and asserts `totalAssets() == 500e6` while `totalAssetsAt(startP2) == 2000e6`.

Settlement provides no second check. `_settleCoverOrder` bounds `totalSettledCover` against the commitment's `totalDeclaredAllocated` and checks concentration caps against the stored `totalAvailableCapacity` — both frozen at commit time. Live vault assets are never re-read. The PoC settles a `P2` order for 3,000 of cover after the payout to show this.

`VaultRewardDistributor.sol:148` shows the inflow case was handled deliberately:

> Checkpoint atomically so historical `totalAssetsAt` lookups (period-start snapshots) include the forwarded assets.

The outflow path has no equivalent treatment for the period in flight.

## Impact

Cover obligations for a period can exceed backing by a margin bounded only by how much the protocol pays out during it. At the 2.0× multiplier used in the PoC, a full-loss claim yields a 6× notional-to-collateral ratio. At maximum permitted leverage, a payout of \~9.1% of the vault breaches the 18.18% floor.

Cover buyers in the affected period hold receipts the collateral cannot honour; a later claim is silently short-paid, since `payout` caps at available assets and returns the reduced amount without reverting. Stakers carry claim exposure at a ratio `minCAR` and `MAX_LEVERAGE_FACTOR` exist to prevent.

The protocol is intentionally fractionally reserved. The defect is that it reserves below the minimum ratio its own code asserts it holds regardless of configuration.

## Remediation

Value the vault leg live — replace `totalAssetsAt(currentPeriodStart())` with `totalAssets()` — matching the treatment already given the first-loss buffer. Matcher determinism is already served by `divergenceToleranceBps`, whose stated purpose is absorbing snapshot-to-commit drift.

Alternatives: re-validate capacity in `_settleCoverOrder` against the live ceiling; or have `payout()` additionally write its reduced total at the current period's start key.

## Proof of Concept

```javascript
// test/integration/firelight-capacity-poc.js

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const { deployVault, computeMarketId } = require('../setup/fixtures.js')
const { plantFlareMocks, FEED_ID } = require('../oracle/fixtures.js')

const E6 = 10n ** 6n
const E18 = 10n ** 18n
const ONE_USD = 10n ** 18n
const BPS = 10000n
const NEW = 0 // CoverOrderType.NEW

const PROTOCOL_MORPHO = 'Morpho'
const PROTOCOL_AAVE = 'Aave'
const MARKET_A = ethers.encodeBytes32String('market-A')
const MARKET_B = ethers.encodeBytes32String('market-B')

const buildTree = (leaves) => {
  const tree = StandardMerkleTree.of(leaves, ['uint256', '(bytes32,uint256)[]'])
  return { tree, root: tree.root }
}
const getProof = (tree, orderId) => {
  for (const [i, leaf] of tree.entries()) if (leaf[0] === BigInt(orderId)) return tree.getProof(i)
  throw new Error(`order ${orderId} not in tree`)
}

const randomSigner = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
  return w
}

async function deployStack() {
  const vaultCtx = await deployVault({
    decimals: 6,
    initial_deposit_limit: '50000000000',
    period_configuration_duration: 604800,
  })
  const { firelight_vault, token_contract, deployer } = vaultCtx

  const admin = await randomSigner()
  const adminRole = await randomSigner()
  const curator = await randomSigner()
  const allocatorRole = await randomSigner()
  const configAdmin = await randomSigner()
  const premiumCollector = await randomSigner()
  const flbWallet = await randomSigner()
  const buyer = await randomSigner()
  const payoutRecipient = await randomSigner()
  const imPayoutReceiver = await randomSigner()
  const approver = await randomSigner()
  const rejecter = await randomSigner()
  const invalidator = await randomSigner()
  const payoutAdmin = await randomSigner()
  const priceFeedAdmin = await randomSigner()

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)

  // Real, in-scope FtsoChainlinkAdapter. plantFlareMocks() is the project's own helper;
  // it stamps MockFlareRegistry at the canonical registry address the adapter hard-codes.
  const { ftso } = await plantFlareMocks()
  const AdapterFactory = await ethers.getContractFactory('FtsoChainlinkAdapter')
  const priceFeed = await AdapterFactory.connect(deployer).deploy(
    admin.address,
    3n * 24n * 60n * 60n,
    admin.address,
    FEED_ID,
    'FXRP / USD',
  )
  await priceFeed.waitForDeployment()

  const publishPrice = async (usdWei = ONE_USD) => {
    const now = BigInt((await ethers.provider.getBlock('latest')).timestamp)
    await ftso.setPrice(FEED_ID, usdWei, now)
  }
  await publishPrice()

  const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' },
  )

  const AllocatorFactory = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    AllocatorFactory,
    [
      {
        vault: await firelight_vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 30 * 24 * 3600,
        premiumTokens: [await usdc.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          { protocol: PROTOCOL_MORPHO, chainId: 1, maxProtocolConcentrationBps: 10000 },
          { protocol: PROTOCOL_AAVE, chainId: 10, maxProtocolConcentrationBps: 10000 },
        ],
        newMarkets: [
          { chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A },
          { chainId: 10, protocol: PROTOCOL_AAVE, market: MARKET_B },
        ],
        capacityConfig: {
          minCAR: 12000, // MIN_CAR_BPS, the 1.2x floor
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: flbWallet.address, // left UNFUNDED so the vault pays 100%
          effectiveLeverage: 24000, // 24000/12000 => exactly 2.0x
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0, // exact capacity, no tolerance band
        },
      },
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await coverNFT.connect(admin).grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())

  const IncidentManagerFactory = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManagerFactory,
    [
      admin.address,
      curator.address,
      approver.address,
      rejecter.address,
      invalidator.address,
      configAdmin.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      imPayoutReceiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      30 * 24 * 3600,
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  const im = await incidentManager.getAddress()
  await firelight_vault.connect(deployer).grantRole(await firelight_vault.INCIDENT_ROLE(), im)
  await firelight_vault.connect(deployer).grantRole(await firelight_vault.PAYOUT_ROLE(), im)
  await firelight_vault
    .connect(vaultCtx.payout_allowlister)
    .addToPayoutAllowlist(imPayoutReceiver.address)

  const marketIdA = computeMarketId(1, PROTOCOL_MORPHO, MARKET_A)

  return {
    ...vaultCtx,
    allocator,
    coverNFT,
    incidentManager,
    usdc,
    priceFeed,
    ftso,
    publishPrice,
    admin,
    curator,
    allocatorRole,
    configAdmin,
    approver,
    buyer,
    payoutRecipient,
    imPayoutReceiver,
    marketIdA,
    advanceOnePeriod: async () => {
      const end = await firelight_vault.currentPeriodEnd()
      await time.increaseTo(BigInt(end) + 1n)
    },
    depositAs: async (user, amount) => {
      await token_contract.mintTo(user.address, amount)
      await token_contract.connect(user).approve(firelight_vault.target, amount)
      await firelight_vault.connect(user).deposit(amount, user.address)
    },
    fundBuyer: async (amount) => {
      await usdc.mint(buyer.address, amount)
      await usdc.connect(buyer).approve(await allocator.getAddress(), amount)
    },
    refOf: (s) => ethers.keccak256(ethers.toUtf8Bytes(s)),
  }
}

describe('Stale collateral basis lets a period underwrite cover below the documented minimum backing', function () {
  it('sells fresh P2 cover against collateral the IncidentManager already paid to P1 claimants', async () => {
    const ctx = await loadFixture(deployStack)
    const {
      firelight_vault,
      allocator,
      incidentManager,
      curator,
      allocatorRole,
      configAdmin,
      approver,
      buyer,
      payoutRecipient,
      imPayoutReceiver,
      marketIdA,
      users,
      deployer,
      token_contract,
      depositAs,
      fundBuyer,
      advanceOnePeriod,
      publishPrice,
      refOf,
    } = ctx

    const log = (m) => console.log('        ' + m)

    // ---- P0: stake 2,000; create the P1 order (orders target currentPeriod()+1) ----
    await depositAs(users[0], 2000n * E6)
    log(`P0  staked ....................... ${(await firelight_vault.totalAssets()) / E6}`)
    await fundBuyer(1000n * E6)

    const cover = 1500n * E18
    await allocator
      .connect(curator)
      .createCoverOrder(buyer.address, payoutRecipient.address, 'bene', ctx.usdc.target, [
        { marketId: marketIdA, coverRateAnnual: 500, coverAmount: cover },
      ], NEW)

    // ---- P1: commit + settle, then capture an incident ----
    await advanceOnePeriod()
    const periodP1 = await firelight_vault.currentPeriod()
    await publishPrice()

    const t1 = buildTree([[0n, [[marketIdA, cover]]]])
    await allocator.connect(allocatorRole).commitAllocation(periodP1, t1.root, cover, 4000n * E18)
    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(0, [{ marketId: marketIdA, allocatedCover: cover }], getProof(t1.tree, 0))
    expect((await allocator.getCoverOrder(0)).allocatedCoverAmount).to.equal(cover)
    log(`P1  cover settled ................ ${cover / E18}`)

    // A second order, created in P1, therefore targeting P2. Settled later, after the payout.
    await allocator
      .connect(curator)
      .createCoverOrder(buyer.address, payoutRecipient.address, 'bene2', ctx.usdc.target, [
        { marketId: marketIdA, coverRateAnnual: 500, coverAmount: 3000n * E18 },
      ], NEW)

    // Incident captured in P1. Payout window is {P1, P2}.
    // Incident ids start at 1 (0 is the absent sentinel in incidentRefs); order ids start at 0.
    const captureTs = await firelight_vault.currentPeriodStart()
    const incidentId = await incidentManager
      .connect(curator)
      .createIncident.staticCall(captureTs, 'P1 incident', refOf('inc-1'))
    await incidentManager.connect(curator).createIncident(captureTs, 'P1 incident', refOf('inc-1'))
    await incidentManager.connect(curator).confirmIncident(incidentId, 'ipfs://report')

    // Full-loss claim against the settled cover — an ordinary total loss.
    await incidentManager
      .connect(curator)
      .addAssessmentLosses(incidentId, [{ coverTokenId: 0, marketId: marketIdA, amount: 1500n * E18 }])
    await incidentManager.connect(curator).submitCurrentAssessment(incidentId)

    // ---- P2: approve in the following period ----
    await advanceOnePeriod()
    const periodP2 = await firelight_vault.currentPeriod()
    const startP2 = await firelight_vault.currentPeriodStart()
    expect(periodP2).to.equal(periodP1 + 1n)

    // P2's capacity basis, snapshotted before anything happens in P2.
    expect(await firelight_vault.totalAssetsAt(startP2)).to.equal(2000n * E6)
    log(`P2  capacity basis at start ...... ${(await firelight_vault.totalAssetsAt(startP2)) / E6}`)

    await time.increase(3600)
    await publishPrice()

    const receiverBefore = await token_contract.balanceOf(imPayoutReceiver.address)

    // Production path: IncidentManager holds PAYOUT_ROLE and drives the vault payout.
    // The first-loss buffer wallet is unfunded, so the vault pays the whole loss.
    await incidentManager.connect(approver).approveCurrentAssessment(incidentId)

    expect((await token_contract.balanceOf(imPayoutReceiver.address)) - receiverBefore).to.equal(1500n * E6)
    expect(await firelight_vault.totalAssets()).to.equal(500n * E6)
    log(`P2  IncidentManager paid out ..... ${((await token_contract.balanceOf(imPayoutReceiver.address)) - receiverBefore) / E6} -> payout receiver`)
    log(`P2  live collateral now .......... ${(await firelight_vault.totalAssets()) / E6}`)

    // ---- DEFECT 1: the P2 basis never saw the drawdown ----
    expect(await firelight_vault.totalAssetsAt(startP2)).to.equal(2000n * E6)
    log(`P2  capacity basis STILL ......... ${(await firelight_vault.totalAssetsAt(startP2)) / E6}   <-- drawdown invisible`)

    // No operator mitigation: checkpointTotalAssets pushes at Time.timestamp(), and
    // totalAssetsAt reads only checkpoints at or before startP2.
    await firelight_vault
      .connect(deployer)
      .grantRole(await firelight_vault.CHECKPOINT_ROLE(), deployer.address)
    await firelight_vault.connect(deployer).checkpointTotalAssets()
    expect(await firelight_vault.totalAssets()).to.equal(500n * E6)
    expect(await firelight_vault.totalAssetsAt(startP2)).to.equal(2000n * E6)
    log(`P2  after checkpointTotalAssets .. ${(await firelight_vault.totalAssetsAt(startP2)) / E6}   <-- no operator fix`)

    // ---- Ceiling control: 4,000 is a hard bound, not slack in the formula ----
    await publishPrice()
    await expect(
      allocator
        .connect(allocatorRole)
        .commitAllocation(periodP2, ethers.id('probe'), 4000n * E18 + 1n, 4000n * E18 + 1n),
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')
    log(`P2  commit of 4000+1 wei ......... reverted MatchingCapacityOverflow (4000 is a hard ceiling)`)

    // ---- DEFECT 2: 4,000 of fresh cover accepted against 500 of real collateral ----
    const p2Cover = 3000n * E18
    const t2 = buildTree([[1n, [[marketIdA, p2Cover]]]])
    await allocator.connect(allocatorRole).commitAllocation(periodP2, t2.root, p2Cover, 4000n * E18)

    const commitP2 = await allocator.getAllocationCommitment(periodP2)
    expect(commitP2.totalAvailableCapacity).to.equal(4000n * E18)
    log(`P2  commit of 4000 ............... ACCEPTED`)

    // ---- DEFECT 3: settlement never re-checks live collateral ----
    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(1, [{ marketId: marketIdA, allocatedCover: p2Cover }], getProof(t2.tree, 1))
    expect((await allocator.getCoverOrder(1)).allocatedCoverAmount).to.equal(p2Cover)
    expect(await firelight_vault.totalAssets()).to.equal(500n * E6)
    log(`P2  fresh cover SETTLED .......... ${p2Cover / E18} against ${(await firelight_vault.totalAssets()) / E6} collateral`)
    log('')

    // ---- Both documented floors breached, in the protocol's own units ----
    const cfg = await allocator.getCapacityConfigAt(periodP2)
    const lev = BigInt(cfg.effectiveLeverage)
    const minCAR = BigInt(cfg.minCAR)
    expect(minCAR).to.equal(12000n)

    const staleCollateral18 = 2000n * E18
    const realCollateral18 = 500n * E18
    const coverSold18 = commitP2.totalAvailableCapacity

    // (1) MIN_CAR_BPS floor: CAR = collateral * effectiveLeverage / coverSold
    const carStale = (staleCollateral18 * lev) / coverSold18
    const carReal = (realCollateral18 * lev) / coverSold18
    expect(carStale).to.equal(minCAR) // passes the on-chain check at exactly the floor
    expect(carReal).to.equal(minCAR / 4n) // 3000 bps == 0.30x
    expect(carReal).to.be.lessThan(BPS) // below even the 1.0x absolute floor

    // (2) CoverOrderAllocator.sol:79-82 — "floors it at 1 / (MAX_LEVERAGE_FACTOR x
    //     (1 + tolerance)) regardless of the configured minCAR". 5 and 10% => 1818 bps.
    const documentedFloorBps = (BPS * BPS) / (5n * (BPS + 1000n))
    const realBackingBps = (realCollateral18 * BPS) / coverSold18
    expect(realBackingBps).to.be.lessThan(documentedFloorBps)

    log(`CAR vs stale basis:  ${carStale} bps (floor ${minCAR})`)
    log(`CAR vs real basis:   ${carReal} bps  <-- MIN_CAR_BPS floor breached`)
    log(`real backing:        ${realBackingBps} bps`)
    log(`documented floor:    ${documentedFloorBps} bps  <-- code comment invariant breached`)

  })
})
```
