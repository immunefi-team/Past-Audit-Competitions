# 88343 sc high a payout after the matcher s capacity read lets stale collateral back new cover and creates an unpaid claim

**Submitted on Aug 12th 2026 at 19:46:31 UTC by @rtw0x1 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88343
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator._computeAvailableCapacity()` values vault collateral with `totalAssetsAt(currentPeriodStart())`. That snapshot intentionally excludes later deposits, but it also continues counting collateral already transferred out by a valid incident payout.

If an honest matcher reads capacity, a valid payout occurs before commitment, and the matcher then submits its previously valid result, `commitAllocation()` recomputes the same stale snapshot and accepts cover above the corrected live ceiling. The integrated test below isolates a distinct 8,000 stale-excess order. A later valid 8,000 loss assessed only against that order empties the vault, transfers only 6,000 of the approved loss, produces a 2,000 shortfall, and closes the incident. The identical no-intervening-payout control pays all 8,000 and retains 2,000 vault assets.

## Severity Rationale

* Suggested severity: Critical
* Impacted component: CoverOrderAllocator capacity validation composed with IncidentManager payouts and FirelightVault collateral accounting
* Preconditions: A trusted matcher reads valid capacity; a distinct valid incident payout reduces vault assets before commitment; the matcher submits its previously honest match; a later valid loss affects the stale-excess order.

The insolvency branch funds the first-loss wallet with 1,000 units and approves IncidentManager before any payout. It uses separate prior 1,000 buffer-backed and 4,000 vault-backed cover tokens, then separate 12,000 live-capacity and 8,000 stale-excess cover tokens, across three incidents. Every assessment is no greater than the affected token's allocation. The no-intervening-payout control pays the same later 8,000 loss in full, isolating the stale capacity check as the cause of the shortfall. The final 2,000 shortfall is measured from receiver balance deltas after the incident is closed, which directly demonstrates the program's “Protocol insolvency” impact.

## Impact

The protocol can collect premiums and mint CoverNFTs for obligations that its configured live collateral envelope no longer supports. In the reproduced normal lifecycle, the stale 8,000 excess order later produces a valid covered loss that receives only 6,000; 2,000 remains unpaid, the vault ends at zero, and the incident is finalized as closed. This is a concrete protocol insolvency rather than a projected collateral ratio.

## Remediation

At commit and recommit time, calculate vault collateral conservatively as the lower of the period-start snapshot and current live `totalAssets()`. This preserves the intended exclusion of mid-period deposits while immediately reflecting payouts and other collateral reductions. Consider also versioning matcher capacity against a vault-capital epoch incremented by every payout, so any match read before a collateral-changing event must be recomputed. Add a regression test for matcher read, intervening payout, stale commit rejection, and a no-payout acceptance control.

## Proof of Concept

## Steps To Reproduce

1. Clone the competition repository and check out the frozen revision:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-firelight.git firelight-poc
   cd firelight-poc
   git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
   npm ci
   mkdir -p test/poc
   ```
2. Save the following code as `test/poc/payout-capacity-integrated.js`:

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId } = require('../setup/fixtures.js')

const SOURCE_COMMIT = '42f9ea5e43d88b35197b901acc2a8c24b314fa62'
const PERIOD_DURATION = 24 * 60 * 60
const BPS = 10_000n
const MARKET = ethers.encodeBytes32String('capacity-market')
const PROTOCOL = 'CapacityProtocol'

const units = (value, decimals) => ethers.parseUnits(value, decimals)

function merkleFor(orderId, marketId, cover) {
  const tree = StandardMerkleTree.of(
    [[orderId, [[marketId, cover]]]],
    ['uint256', '(bytes32,uint256)[]'],
  )
  return { tree, root: tree.root, proof: tree.getProof(0) }
}

async function moveToNextPeriod(vault) {
  const prior = await vault.currentPeriod()
  await time.increaseTo(await vault.currentPeriodEnd())
  expect(await vault.currentPeriod()).to.equal(prior + 1n)
}

async function createOrder(ctx, cover) {
  await ctx.allocator.connect(ctx.curator).createCoverOrder(
    ctx.buyer.address,
    ctx.buyer.address,
    ctx.buyer.address,
    await ctx.usdc.getAddress(),
    [{ marketId: ctx.marketId, coverRateAnnual: 500, coverAmount: cover }],
    0,
  )
  return (await ctx.allocator.nextCoverOrderId()) - 1n
}

async function settleOrder(ctx, orderId, cover, matchingCapacity) {
  const { root, proof } = merkleFor(orderId, ctx.marketId, cover)
  const period = await ctx.vault.currentPeriod()
  await ctx.allocator
    .connect(ctx.allocatorRole)
    .commitAllocation(period, root, cover, matchingCapacity)
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
    orderId,
    [{ marketId: ctx.marketId, allocatedCover: cover }],
    proof,
  )
  return ctx.allocator.getAllocationCommitment(period)
}

async function deployIntegratedSystem() {
  const vaultCtx = await deployVault({
    decimals: 6,
    initial_deposit_limit: units('100000', 6),
    period_configuration_duration: PERIOD_DURATION,
  })

  const [deployer, admin, curator, allocatorRole, configAdmin, adminRole,
    premiumCollector, firstLossBufferWallet, buyer, assessmentApprover,
    assessmentRejecter, incidentInvalidator, incidentConfigAdmin,
    incidentPayoutAdmin, incidentPriceAdmin] = await ethers.getSigners()

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.deploy('Mock USDC', 'mUSDC', 6)
  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, units('1', 18))

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address,
      ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' },
  )

  const marketId = computeMarketId(1, PROTOCOL, MARKET)
  const capacityConfig = {
    minCAR: 12_000,
    firstLossBufferToken: await usdc.getAddress(),
    firstLossBuffer: firstLossBufferWallet.address,
    effectiveLeverage: 24_000,
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0,
  }

  const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    Allocator,
    [{
      vault: await vaultCtx.firelight_vault.getAddress(),
      premiumCollector: premiumCollector.address,
      coverNFT: await coverNFT.getAddress(),
      priceFeedAdapter: await priceFeed.getAddress(),
      maxPriceAge: 7 * PERIOD_DURATION,
      premiumTokens: [await usdc.getAddress()],
      admin: admin.address,
      adminRole: adminRole.address,
      curatorRole: curator.address,
      allocatorRole: allocatorRole.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{
        protocol: PROTOCOL,
        chainId: 1,
        maxProtocolConcentrationBps: 10_000,
      }],
      newMarkets: [{ chainId: 1, protocol: PROTOCOL, market: MARKET }],
      capacityConfig,
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      admin.address,
      curator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      incidentConfigAdmin.address,
      incidentPayoutAdmin.address,
      incidentPriceAdmin.address,
      vaultCtx.payout_receiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      7 * PERIOD_DURATION,
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await vaultCtx.firelight_vault.connect(deployer).grantRole(
    await vaultCtx.firelight_vault.PAYOUT_ROLE(),
    await incidentManager.getAddress(),
  )

  // Fund and approve the first-loss buffer as intended. The legitimate
  // payout below consumes it before drawing the remaining loss from the vault.
  await usdc.mint(firstLossBufferWallet.address, units('1000', 6))
  await usdc.connect(firstLossBufferWallet).approve(
    await incidentManager.getAddress(),
    ethers.MaxUint256,
  )
  await vaultCtx.firelight_vault.connect(deployer).grantRole(
    await vaultCtx.firelight_vault.INCIDENT_ROLE(),
    await incidentManager.getAddress(),
  )

  const initialAssets = units('10000', 6)
  await vaultCtx.utils.mintAndApprove(initialAssets, vaultCtx.users[0])
  await vaultCtx.firelight_vault
    .connect(vaultCtx.users[0])
    .deposit(initialAssets, vaultCtx.users[0].address)

  await usdc.mint(buyer.address, units('1000', 6))
  await usdc.connect(buyer).approve(await allocator.getAddress(), ethers.MaxUint256)

  return {
    ...vaultCtx,
    vault: vaultCtx.firelight_vault,
    allocator,
    coverNFT,
    incidentManager,
    usdc,
    priceFeed,
    deployer,
    admin,
    curator,
    allocatorRole,
    configAdmin,
    buyer,
    assessmentApprover,
    firstLossBufferWallet,
    marketId,
    capacityConfig,
    initialAssets,
  }
}

async function preparePriorCoverAndFutureOrder(ctx) {
  const bufferCover = units('1000', 18)
  const vaultCover = units('4000', 18)
  const bufferOrderId = await createOrder(ctx, bufferCover)
  const vaultOrderId = await createOrder(ctx, vaultCover)

  await moveToNextPeriod(ctx.vault)
  expect(await ctx.vault.currentPeriod()).to.equal(1n)

  // Honest pre-payout matcher read: (10,000 vault + 1,000 FLB) * 2 = 22,000.
  const prePayoutCapacity = units('22000', 18)
  const priorTree = StandardMerkleTree.of(
    [
      [bufferOrderId, [[ctx.marketId, bufferCover]]],
      [vaultOrderId, [[ctx.marketId, vaultCover]]],
    ],
    ['uint256', '(bytes32,uint256)[]'],
  )
  await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
    await ctx.vault.currentPeriod(),
    priorTree.root,
    bufferCover + vaultCover,
    prePayoutCapacity,
  )
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
    bufferOrderId,
    [{ marketId: ctx.marketId, allocatedCover: bufferCover }],
    priorTree.getProof(0),
  )
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
    vaultOrderId,
    [{ marketId: ctx.marketId, allocatedCover: vaultCover }],
    priorTree.getProof(1),
  )

  const futureCover = units('20000', 18)
  const futureOrderId = await createOrder(ctx, futureCover)
  return {
    bufferCover,
    vaultCover,
    bufferOrderId,
    vaultOrderId,
    futureCover,
    futureOrderId,
    prePayoutCapacity,
  }
}

async function settleFutureCoversAfterMatcherRead(ctx, covers, matcherCapacity) {
  const tree = StandardMerkleTree.of(
    [
      [covers.baseOrderId, [[ctx.marketId, covers.baseCover]]],
      [covers.excessOrderId, [[ctx.marketId, covers.excessCover]]],
    ],
    ['uint256', '(bytes32,uint256)[]'],
  )
  await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
    await ctx.vault.currentPeriod(),
    tree.root,
    covers.baseCover + covers.excessCover,
    matcherCapacity,
  )
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
    covers.baseOrderId,
    [{ marketId: ctx.marketId, allocatedCover: covers.baseCover }],
    tree.getProof(0),
  )
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
    covers.excessOrderId,
    [{ marketId: ctx.marketId, allocatedCover: covers.excessCover }],
    tree.getProof(1),
  )
}

async function createAndSubmitLoss(ctx, incidentId, captureTimestamp, coverTokenId, amount, tag) {
  await ctx.incidentManager.connect(ctx.curator).createIncident(
    captureTimestamp,
    tag,
    ethers.id(tag),
  )
  await ctx.incidentManager.connect(ctx.curator).confirmIncident(incidentId, 'ipfs://local-' + tag)
  await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(incidentId, [{
    coverTokenId,
    marketId: ctx.marketId,
    amount,
  }])
  await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(incidentId)
}

async function downstreamClaimScenario(withInterveningVaultPayout) {
  const ctx = await deployIntegratedSystem()
  const prepared = await preparePriorCoverAndFutureOrder(ctx)
  const priorCapture = await ctx.vault.currentPeriodStart()

  // Isolate the correct live-capacity tranche from the stale excess. With the
  // payout, live capacity is exactly 12,000, so this distinct 8,000 order is
  // wholly outside the configured envelope rather than part of a 20,000 order
  // that could have been partially allocated.
  const covers = {
    baseCover: units('12000', 18),
    excessCover: units('8000', 18),
    baseOrderId: await createOrder(ctx, units('12000', 18)),
    excessOrderId: await createOrder(ctx, units('8000', 18)),
  }

  await createAndSubmitLoss(
    ctx,
    1,
    priorCapture,
    prepared.bufferOrderId,
    units('1000', 18),
    withInterveningVaultPayout ? 'impact-buffer-positive' : 'impact-buffer-control',
  )

  if (withInterveningVaultPayout) {
    await createAndSubmitLoss(
      ctx,
      2,
      priorCapture + 1n,
      prepared.vaultOrderId,
      units('4000', 18),
      'impact-vault-positive',
    )
  }

  await moveToNextPeriod(ctx.vault)
  expect(await ctx.vault.currentPeriod()).to.equal(2n)
  const coveredPeriodStart = await ctx.vault.currentPeriodStart()

  // Normal first-loss use leaves the honest matcher with 20,000 capacity.
  await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)
  expect(await ctx.usdc.balanceOf(ctx.firstLossBufferWallet.address)).to.equal(0)
  const matcherCapacity = units('20000', 18)

  if (withInterveningVaultPayout) {
    // A distinct valid loss lands after the matcher read but before commitment.
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(2)
    expect(await ctx.vault.totalAssets()).to.equal(units('6000', 6))
  } else {
    expect(await ctx.vault.totalAssets()).to.equal(units('10000', 6))
  }

  await settleFutureCoversAfterMatcherRead(ctx, covers, matcherCapacity)

  expect(await ctx.coverNFT.ownerOf(covers.baseOrderId)).to.equal(ctx.buyer.address)
  expect(await ctx.coverNFT.ownerOf(covers.excessOrderId)).to.equal(ctx.buyer.address)

  // A third, distinct incident is assessed against only the newly sold cover.
  // 8,000 is payable by the 10,000-asset control but not by the stale 6,000-asset state.
  const claimIncidentId = withInterveningVaultPayout ? 3 : 2
  const validCoveredLoss = units('8000', 18)
  await createAndSubmitLoss(
    ctx,
    claimIncidentId,
    coveredPeriodStart,
    covers.excessOrderId,
    validCoveredLoss,
    withInterveningVaultPayout ? 'impact-downstream-positive' : 'impact-downstream-control',
  )

  const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address)
  await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(claimIncidentId)
  const receiverAfter = await ctx.token_contract.balanceOf(ctx.payout_receiver.address)
  const paid = receiverAfter - receiverBefore
  const unpaid = units('8000', 6) - paid
  const [incident, incidentExists] = await ctx.incidentManager.getIncident(claimIncidentId)
  expect(incidentExists).to.equal(true)

  return {
    ctx,
    prepared,
    covers,
    paid,
    unpaid,
    validCoveredLoss,
    incidentStatus: incident.status,
    finalVaultAssets: await ctx.vault.totalAssets(),
  }
}

describe('PoC: same-period payout leaves stale collateral in allocator capacity', function () {
  it('accepts and settles new cover above the post-payout live-collateral ceiling', async function () {
    const ctx = await loadFixture(deployIntegratedSystem)
    const prepared = await preparePriorCoverAndFutureOrder(ctx)

    const incidentCapture = await ctx.vault.currentPeriodStart()
    const bufferLoss = units('1000', 18)
    const vaultLoss = units('4000', 18)

    await ctx.incidentManager.connect(ctx.curator).createIncident(
      incidentCapture,
      'First valid loss consumes the funded buffer',
      ethers.id('payout-capacity-buffer-poc'),
    )
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1, 'ipfs://local-poc')
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(1, [{
      coverTokenId: prepared.bufferOrderId,
      marketId: ctx.marketId,
      amount: bufferLoss,
    }])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1)

    await ctx.incidentManager.connect(ctx.curator).createIncident(
      incidentCapture + 1n,
      'Second valid loss reaches vault collateral',
      ethers.id('payout-capacity-vault-poc'),
    )
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(2, 'ipfs://local-poc-vault')
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(2, [{
      coverTokenId: prepared.vaultOrderId,
      marketId: ctx.marketId,
      amount: vaultLoss,
    }])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(2)

    await moveToNextPeriod(ctx.vault)
    expect(await ctx.vault.currentPeriod()).to.equal(2n)

    const periodStart = await ctx.vault.currentPeriodStart()
    const snapshotBefore = await ctx.vault.totalAssetsAt(periodStart)
    const liveBefore = await ctx.vault.totalAssets()
    const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address)

    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)
    expect(await ctx.usdc.balanceOf(ctx.firstLossBufferWallet.address)).to.equal(0)
    expect(await ctx.vault.totalAssets()).to.equal(ctx.initialAssets)

    // This is the honest matcher read: the first approved incident has consumed
    // the funded FLB, but no vault collateral has left yet.
    const honestPrePayoutMatchingCapacity = units('20000', 18)

    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(2)

    const snapshotAfter = await ctx.vault.totalAssetsAt(periodStart)
    const liveAfter = await ctx.vault.totalAssets()
    const receiverAfter = await ctx.token_contract.balanceOf(ctx.payout_receiver.address)

    expect(snapshotBefore).to.equal(ctx.initialAssets)
    expect(liveBefore).to.equal(ctx.initialAssets)
    expect(snapshotAfter).to.equal(snapshotBefore)
    expect(liveAfter).to.equal(units('6000', 6))
    expect(receiverAfter - receiverBefore).to.equal(units('4000', 6))

    // The funded FLB was consumed normally before the matcher read. A later
    // valid incident then removes 4,000 from the vault between read and commit.
    const snapshotCapacity = honestPrePayoutMatchingCapacity
    const liveCapacity = units('12000', 18)
    const liveCollateral = units('6000', 18)
    const requiredBackingBps = BigInt(ctx.capacityConfig.minCAR) * BPS * BPS /
      (BigInt(ctx.capacityConfig.effectiveLeverage) *
        (BPS + BigInt(ctx.capacityConfig.divergenceToleranceBps)))
    const actualBackingBps = liveCollateral * BPS / prepared.futureCover
    const backingBreachBps = requiredBackingBps - actualBackingBps
    const overLiveCapacity = prepared.futureCover - liveCapacity

    expect(snapshotCapacity).to.equal(prepared.futureCover)
    expect(snapshotCapacity).to.be.greaterThan(liveCapacity)
    expect(prepared.futureCover).to.be.greaterThan(liveCapacity)
    expect(actualBackingBps).to.be.lessThan(requiredBackingBps)

    const { root, proof } = merkleFor(prepared.futureOrderId, ctx.marketId, prepared.futureCover)
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
      await ctx.vault.currentPeriod(),
      root,
      prepared.futureCover,
      snapshotCapacity,
    )

    const commitment = await ctx.allocator.getAllocationCommitment(await ctx.vault.currentPeriod())
    expect(commitment.totalAvailableCapacity).to.equal(snapshotCapacity)
    expect(commitment.totalDeclaredAllocated).to.equal(prepared.futureCover)

    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
      prepared.futureOrderId,
      [{ marketId: ctx.marketId, allocatedCover: prepared.futureCover }],
      proof,
    )

    const order = await ctx.allocator.getCoverOrder(prepared.futureOrderId)
    expect(order.allocatedCoverAmount).to.equal(prepared.futureCover)
    expect(await ctx.coverNFT.ownerOf(prepared.futureOrderId)).to.equal(ctx.buyer.address)

    console.log('POC_RESULT ' + JSON.stringify({
      sourceCommit: SOURCE_COMMIT,
      period: (await ctx.vault.currentPeriod()).toString(),
      periodStart: periodStart.toString(),
      vault: await ctx.vault.getAddress(),
      allocator: await ctx.allocator.getAddress(),
      incidentManager: await ctx.incidentManager.getAddress(),
      snapshotAssets6: snapshotAfter.toString(),
      liveAssets6: liveAfter.toString(),
      payoutAssets6: (receiverAfter - receiverBefore).toString(),
      snapshotCapacity18: snapshotCapacity.toString(),
      liveCapacity18: liveCapacity.toString(),
      settledCover18: order.allocatedCoverAmount.toString(),
      overLiveCapacity18: overLiveCapacity.toString(),
      requiredBackingBps: requiredBackingBps.toString(),
      actualBackingBps: actualBackingBps.toString(),
      backingBreachBps: backingBreachBps.toString(),
      firstLossBufferNative: (await ctx.usdc.balanceOf(ctx.firstLossBufferWallet.address)).toString(),
      nftOwner: await ctx.coverNFT.ownerOf(prepared.futureOrderId),
    }))
  })

  it('turns an otherwise fully payable distinct covered loss into a finalized partial payout', async function () {
    const positive = await downstreamClaimScenario(true)
    const control = await downstreamClaimScenario(false)

    expect(positive.paid).to.equal(units('6000', 6))
    expect(positive.unpaid).to.equal(units('2000', 6))
    expect(positive.finalVaultAssets).to.equal(0)
    expect(positive.incidentStatus).to.equal(4n) // CLOSED despite the shortfall.

    expect(control.paid).to.equal(units('8000', 6))
    expect(control.unpaid).to.equal(0)
    expect(control.finalVaultAssets).to.equal(units('2000', 6))
    expect(control.incidentStatus).to.equal(4n)

    console.log('DOWNSTREAM_IMPACT_RESULT ' + JSON.stringify({
      sourceCommit: SOURCE_COMMIT,
      validCoveredLoss18: positive.validCoveredLoss.toString(),
      configuredLiveCapacity18: positive.covers.baseCover.toString(),
      staleExcessOrderCover18: positive.covers.excessCover.toString(),
      positiveSettledCover18: (positive.covers.baseCover + positive.covers.excessCover).toString(),
      positivePaidAssets6: positive.paid.toString(),
      positiveUnpaidAssets6: positive.unpaid.toString(),
      positiveFinalVaultAssets6: positive.finalVaultAssets.toString(),
      positiveIncidentStatus: positive.incidentStatus.toString(),
      controlPaidAssets6: control.paid.toString(),
      controlUnpaidAssets6: control.unpaid.toString(),
      controlFinalVaultAssets6: control.finalVaultAssets.toString(),
      controlIncidentStatus: control.incidentStatus.toString(),
    }))
  })

  it('negative control: without payout, committed cover remains inside the live ceiling', async function () {
    const ctx = await loadFixture(deployIntegratedSystem)
    const prepared = await preparePriorCoverAndFutureOrder(ctx)
    await moveToNextPeriod(ctx.vault)

    const periodStart = await ctx.vault.currentPeriodStart()
    const snapshotAssets = await ctx.vault.totalAssetsAt(periodStart)
    const liveAssets = await ctx.vault.totalAssets()
    const capacity = units('20000', 18)

    expect(snapshotAssets).to.equal(ctx.initialAssets)
    expect(liveAssets).to.equal(ctx.initialAssets)

    const commitment = await settleOrder(
      ctx,
      prepared.futureOrderId,
      prepared.futureCover,
      capacity,
    )
    expect(commitment.totalAvailableCapacity).to.equal(capacity)
    expect(prepared.futureCover).to.equal(capacity)

    console.log('NO_PAYOUT_CONTROL ' + JSON.stringify({
      snapshotAssets6: snapshotAssets.toString(),
      liveAssets6: liveAssets.toString(),
      liveCapacity18: capacity.toString(),
      settledCover18: commitment.totalSettledCover.toString(),
    }))
  })

  it('negative control: mid-period deposits are conservatively excluded from the snapshot', async function () {
    const ctx = await loadFixture(deployIntegratedSystem)
    await moveToNextPeriod(ctx.vault)

    const periodStart = await ctx.vault.currentPeriodStart()
    const snapshotBefore = await ctx.vault.totalAssetsAt(periodStart)
    const deposit = units('5000', 6)
    await ctx.utils.mintAndApprove(deposit, ctx.users[1])
    await ctx.vault.connect(ctx.users[1]).deposit(deposit, ctx.users[1].address)

    const snapshotAfter = await ctx.vault.totalAssetsAt(periodStart)
    const liveAfter = await ctx.vault.totalAssets()
    // The strengthened normal-lifecycle fixture starts with a funded 1,000 FLB,
    // so the period-start capacity is (10,000 vault + 1,000 FLB) * 2.
    const snapshotCapacity = units('22000', 18)

    expect(snapshotBefore).to.equal(ctx.initialAssets)
    expect(snapshotAfter).to.equal(snapshotBefore)
    expect(liveAfter).to.equal(units('15000', 6))

    const period = await ctx.vault.currentPeriod()
    await expect(ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
      period,
      ethers.id('mid-period-deposit-overflow'),
      0,
      snapshotCapacity + 1n,
    )).to.be.revertedWithCustomError(ctx.allocator, 'MatchingCapacityOverflow')
      .withArgs(snapshotCapacity + 1n, snapshotCapacity)

    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
      period,
      ethers.id('mid-period-deposit-boundary'),
      0,
      snapshotCapacity,
    )

    console.log('MID_PERIOD_DEPOSIT_CONTROL ' + JSON.stringify({
      snapshotAssets6: snapshotAfter.toString(),
      liveAssets6: liveAfter.toString(),
      acceptedSnapshotCapacity18: snapshotCapacity.toString(),
      rejectedCapacity18: (snapshotCapacity + 1n).toString(),
    }))
  })
})
```

3. Run the integrated tests:

   ```bash
   npx hardhat test test/poc/payout-capacity-integrated.js
   ```
4. Observe four passing cases. In the positive branch, the vault falls from 10,000 to 6,000 before commit, so the corrected live capacity is 12,000, but 20,000 cover still settles. The later 8,000 loss against only the distinct excess order pays 6,000, leaves 2,000 unpaid, empties the vault, and closes the incident. The no-payout branch pays the same loss in full.

   See `step-04-stale-capacity-finalizes-unpaid-claim.png` for the command and terminal result.

The test runs only on Hardhat's ephemeral local network; it sends no public-chain transactions and requires no on-chain cleanup.

## Proof Of Impact

```
POC_RESULT {
  "snapshotAssets6":"10000000000",
  "liveAssets6":"6000000000",
  "snapshotCapacity18":"20000000000000000000000",
  "liveCapacity18":"12000000000000000000000",
  "settledCover18":"20000000000000000000000",
  "overLiveCapacity18":"8000000000000000000000",
  "requiredBackingBps":"5000",
  "actualBackingBps":"3000"
}
DOWNSTREAM_IMPACT_RESULT {
  "validCoveredLoss18":"8000000000000000000000",
  "staleExcessOrderCover18":"8000000000000000000000",
  "positivePaidAssets6":"6000000000",
  "positiveUnpaidAssets6":"2000000000",
  "positiveFinalVaultAssets6":"0",
  "positiveIncidentStatus":"4",
  "controlPaidAssets6":"8000000000",
  "controlUnpaidAssets6":"0",
  "controlFinalVaultAssets6":"2000000000"
}
4 passing
```

The stale value enters the commit guard here:

```solidity
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);

// _computeAvailableCapacity()
$.vault.totalAssetsAt($.vault.currentPeriodStart())
```

Because the period-start value remains 10,000 after the vault has paid 4,000, the guard still considers the old 20,000 matcher value valid instead of rejecting it against the 12,000 live ceiling.
