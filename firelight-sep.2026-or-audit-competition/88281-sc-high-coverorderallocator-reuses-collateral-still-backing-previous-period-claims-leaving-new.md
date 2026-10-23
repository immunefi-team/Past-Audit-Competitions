# 88281 sc high coverorderallocator reuses collateral still backing previous period claims leaving new cover insolvent

**Submitted on Aug 12th 2026 at 15:34:55 UTC by @Tradi3 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88281
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator` treats the vault balance at the start of the current period as available collateral. The same assets can still be paid to a valid incident from the previous period.

This allows the protocol to fully match a new cover order and charge its premium using collateral that remains liable for an older claim. If that older incident is reported after the new order settles, it consumes the backing already counted for the new cover. A later valid claim on the new order then receives only what remains, which can be zero.

The PoC uses the intended 1x configuration. The order curator creates two valid orders, the allocator commits and settles them, the incident curator reports two genuine incidents, and the assessment approver approves both losses within their respective allocations. None of those roles behaves maliciously or exceeds its permissions.

## Root cause

`CoverOrderAllocator._computeAvailableCapacity()` derives current-period capacity from the vault's opening checkpoint:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 stakedAssetsValueUSDCanonical =
    totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
uint256 availableCollateralCanonical =
    firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

uint256 strictCapacity =
    availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
totalAvailableCapacity = strictCapacity.mulDiv(
    BPS_DENOMINATOR + config.divergenceToleranceBps,
    BPS_DENOMINATOR
);
```

`commitAllocation()` stores that capacity, and settlement later increases the amount of settled cover against it:

```solidity
commit.totalAvailableCapacity = matchingCapacity;
...
commit.totalSettledCover += allocatedCover;
```

However, `IncidentManager` permits an incident from period P-1 to be created and approved during period P. `FirelightVault.payout()` then pays it from the vault's live assets:

```solidity
if (!_isPeriodInPayoutWindow(capturePeriod, _currentPeriod)) {
    revert InvalidCapturePeriod();
}

uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

No previous-period liability is reserved when period-P capacity is committed. Settlement also does not prevent an older, still-valid incident from being registered afterwards. The result is that the same vault collateral backs two consecutive periods of cover.

## Reproduction sequence

1. An LP deposits 1,000 units into the vault.
2. A 1,000-unit cover order is created and settled for period 1.
3. A second 1,000-unit order is created for period 2.
4. At the start of period 2, `totalAssetsAt(currentPeriodStart())` and the live vault balance are both 1,000. At 1x leverage, the allocator commits and settles exactly 1,000 units of new cover. The buyer receives the Cover NFT and pays the premium.
5. Only after that settlement, a valid period-1 incident is reported and approved within the permitted payout window. It consumes the assets counted for period-2 capacity.
6. A valid period-2 incident is then approved. With no first-loss buffer it closes with `vaultPaidAmount == 0`. With a real 100-unit first-loss buffer funded and fully approved, the period-2 claim receives only 100 of the promised 1,000 units.

## Impact

A buyer can pay premium for fully matched cover that is not actually backed when its claim is processed. The PoC demonstrates a 100% shortfall with no first-loss buffer and a 90% shortfall with a funded 100-unit buffer.

This is protocol insolvency: settled cover exceeds the collateral still available to honor it. The problem occurs during capacity accounting, before either incident is approved.

This is separate from the disclosed aggregate-claims issue. The PoC uses two different orders in two different periods and one valid loss against each order. It follows FIFO and never claims the same cover twice. Adding a per-order `remainingCoverage` cap would not reserve collateral for the still-live period-1 liability before period-2 capacity is committed.

## Recommended fix

Do not make collateral available for new-period matching while it can still be claimed by the previous period. The protocol can reserve the maximum unresolved previous-period liability, segregate collateral by coverage period, or delay new-period settlement until the previous-period reporting window closes.

A one-time live-balance check during settlement is not sufficient because the older incident can be registered after settlement, which is the ordering used in the PoC.

## Preconditions

The buyer cannot approve its own incident. The loss requires two genuine incidents to be processed by the intended roles in FIFO order. No compromised role, invalid assessment, excessive leverage, repeated claim, or first-loss-buffer failure is required.

## Proof of Concept

## Steps to reproduce

1. Save the test below as `test/PoC_ConsecutivePeriodDoubleUse.js`.
2. From the frozen `v1_audit_ready` checkout at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, run:

```bash
npx hardhat test test/PoC_ConsecutivePeriodDoubleUse.js --no-compile
```

Expected result:

```
PoC: consecutive-period collateral is used twice
  ✔ charges and settles period-2 cover before a late period-1 payout drains its backing (FLB=0)
  ✔ charges and settles period-2 cover before a late period-1 payout drains its backing (FLB=100)

2 passing
```

The first case leaves the new period-2 claim with zero vault payment. The second funds and fully approves a real 100-unit first-loss buffer; the new claim still receives only 100 of the promised 1,000 units.

PoC SHA-256: `CDE79D636A90AB8CC83A9BF4F7BA12A509452246E42874B078D73C6E783B929D`

The two focused cases pass. Running this PoC together with the relevant allocator, incident-manager, and vault-payout regression suites passes 189/189.

## Full runnable test

```javascript
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId } = require('./setup/fixtures.js')

const e6 = (value) => ethers.parseUnits(String(value), 6)
const e18 = (value) => ethers.parseUnits(String(value), 18)

const BPS = 10_000n
const SECONDS_PER_YEAR = 365n * 24n * 60n * 60n
const PERIOD_DURATION = 24n * 60n * 60n
const COVER_RATE_ANNUAL = 500n
const SCALE_18_TO_6 = 10n ** 12n

function ceilDiv(numerator, denominator) {
  return (numerator + denominator - 1n) / denominator
}

function expectedPremiumNative(cover) {
  const canonical = ceilDiv(
    cover * COVER_RATE_ANNUAL * PERIOD_DURATION,
    BPS * SECONDS_PER_YEAR,
  )
  return ceilDiv(canonical, SCALE_18_TO_6)
}

function allocationTree(orderId, marketId, cover) {
  return StandardMerkleTree.of(
    [[BigInt(orderId), [[marketId, cover]]]],
    ['uint256', '(bytes32,uint256)[]'],
  )
}

describe('PoC: consecutive-period collateral is used twice', function () {
  this.timeout(180_000)

  for (const fundedFlbUnits of [0, 100]) {
    it(`charges and settles period-2 cover before a late period-1 payout drains its backing (FLB=${fundedFlbUnits})`, async function () {
    const ctx = await deployVault({
      initial_deposit_limit: e6(1_000_000),
      period_configuration_duration: Number(PERIOD_DURATION),
    })

    const signers = await ethers.getSigners()
    const deployer = signers[0]
    const lp = ctx.users[0]
    const curator = signers[10]
    const allocatorRole = signers[11]
    const oldBuyer = signers[12]
    const newBuyer = signers[13]
    const premiumCollector = signers[14]
    const firstLossBuffer = signers[15]
    const incidentCurator = signers[16]
    const assessmentApprover = signers[17]

    const vault = ctx.firelight_vault
    const vaultAsset = ctx.token_contract
    const cover = e18(1_000)
    const collateral = e6(1_000)
    const fundedFlb = e6(fundedFlbUnits)

    await ctx.utils.mintAndApprove(collateral, lp)
    await vault.connect(lp).deposit(collateral, lp.address)
    expect(await vault.currentPeriod()).to.equal(0n)
    expect(await vault.totalAssets()).to.equal(collateral)

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, e18(1))

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      [
        'Firelight Cover',
        'FLCOVER',
        '',
        deployer.address,
        deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
      ],
      { kind: 'transparent' },
    )

    const chainId = 1
    const protocol = 'PoC Protocol'
    const marketName = ethers.encodeBytes32String('PoC Market')
    const marketId = computeMarketId(chainId, protocol, marketName)

    const CoverOrderAllocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      CoverOrderAllocator,
      [{
        vault: vault.target,
        premiumCollector: premiumCollector.address,
        coverNFT: coverNFT.target,
        premiumTokens: [vaultAsset.target],
        admin: deployer.address,
        adminRole: deployer.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: deployer.address,
        initialProtocolConcentrations: [{
          protocol,
          chainId,
          maxProtocolConcentrationBps: 10_000,
        }],
        newMarkets: [{ chainId, protocol, market: marketName }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: vaultAsset.target,
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 7 * 24 * 60 * 60,
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await coverNFT.grantRole(ethers.id('MINTER_ROLE'), allocator.target)

    const IncidentManager = await ethers.getContractFactory('IncidentManager')
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        deployer.address,
        incidentCurator.address,
        assessmentApprover.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        ctx.payout_receiver.address,
        allocator.target,
        priceFeed.target,
        7 * 24 * 60 * 60,
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await vault.grantRole(await vault.PAYOUT_ROLE(), incidentManager.target)
    await vault.grantRole(await vault.INCIDENT_ROLE(), incidentManager.target)

    if (fundedFlb > 0n) {
      await vaultAsset.mintTo(firstLossBuffer.address, fundedFlb)
      await vaultAsset.connect(firstLossBuffer).approve(incidentManager.target, fundedFlb)
    }

    for (const buyer of [oldBuyer, newBuyer]) {
      await vaultAsset.mintTo(buyer.address, e6(10))
      await vaultAsset.connect(buyer).approve(allocator.target, ethers.MaxUint256)
    }

    const createOrder = async (buyer, beneficiary) => allocator.connect(curator).createCoverOrder(
      buyer.address,
      buyer.address,
      beneficiary,
      vaultAsset.target,
      [{
        marketId,
        coverRateAnnual: Number(COVER_RATE_ANNUAL),
        coverAmount: cover,
      }],
      0,
    )

    const settleOrder = async (orderId) => {
      const tree = allocationTree(orderId, marketId, cover)
      const period = await vault.currentPeriod()
      await allocator.connect(allocatorRole).commitAllocation(period, tree.root, cover, cover)
      await allocator.connect(allocatorRole).settleCoverOrder(
        orderId,
        [{ marketId, allocatedCover: cover }],
        tree.getProof(0),
      )
    }

    const submitIncident = async (incidentId, captureTimestamp, orderId, ref) => {
      await incidentManager.connect(incidentCurator).createIncident(
        captureTimestamp,
        ref,
        ethers.id(ref),
      )
      await incidentManager.connect(incidentCurator).confirmIncident(incidentId, `ipfs://${ref}`)
      await incidentManager.connect(incidentCurator).addAssessmentLosses(
        incidentId,
        [{ coverTokenId: orderId, marketId, amount: cover }],
      )
      await incidentManager.connect(incidentCurator).submitCurrentAssessment(incidentId)
    }

    // Order 0 is legitimate period-1 cover backed by the same 1,000 vault assets.
    await createOrder(oldBuyer, 'period-1-beneficiary')
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.equal(1n)
    await settleOrder(0)
    const period1CaptureTimestamp = BigInt(await time.latest())

    // While no incident has been reported, create the next order for period 2.
    await createOrder(newBuyer, 'period-2-beneficiary')
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.equal(2n)

    const period2Start = await vault.currentPeriodStart()
    expect(await vault.totalAssetsAt(period2Start)).to.equal(collateral)
    expect(await vault.totalAssets()).to.equal(collateral)
    expect(await vaultAsset.balanceOf(firstLossBuffer.address)).to.equal(fundedFlb)

    // The period-2 commitment uses exactly the 1,000 start snapshot at 1x leverage.
    // It succeeds and settlement charges premium before the old incident is reported.
    const collectorBefore = await vaultAsset.balanceOf(premiumCollector.address)
    const newBuyerBefore = await vaultAsset.balanceOf(newBuyer.address)
    await settleOrder(1)
    const expectedPremium = expectedPremiumNative(cover)

    const period2Commitment = await allocator.getAllocationCommitment(2)
    expect(period2Commitment.totalAvailableCapacity).to.equal(cover)
    expect(period2Commitment.totalDeclaredAllocated).to.equal(cover)
    expect(period2Commitment.totalSettledCover).to.equal(cover)
    expect(await coverNFT.ownerOf(1)).to.equal(newBuyer.address)
    expect(await vaultAsset.balanceOf(premiumCollector.address) - collectorBefore).to.equal(expectedPremium)
    expect(newBuyerBefore - await vaultAsset.balanceOf(newBuyer.address)).to.equal(expectedPremium)
    expect(expectedPremium).to.equal(136_987n)

    const newOrder = await allocator.getCoverOrder(1)
    expect(newOrder.status).to.equal(1n) // MATCHED
    expect(newOrder.period).to.equal(2n)
    expect(newOrder.allocatedCoverAmount).to.equal(cover)

    // A valid period-1 incident can be reported and approved during period 2.
    // It consumes the same assets that have just backed the period-2 cover.
    await submitIncident(1, period1CaptureTimestamp, 0, 'late-period-1-incident')
    const receiverBeforeOldPayout = await vaultAsset.balanceOf(ctx.payout_receiver.address)
    await incidentManager.connect(assessmentApprover).approveCurrentAssessment(1)
    const receiverAfterOldPayout = await vaultAsset.balanceOf(ctx.payout_receiver.address)

    expect(receiverAfterOldPayout - receiverBeforeOldPayout).to.equal(collateral)
    expect(await vault.totalAssets()).to.equal(fundedFlb)
    expect(await vault.totalAssetsAt(period2Start)).to.equal(collateral)
    expect(await vaultAsset.balanceOf(firstLossBuffer.address)).to.equal(0n)

    const [oldIncident] = await incidentManager.getIncident(1)
    expect(oldIncident.status).to.equal(4n) // CLOSED
    expect(oldIncident.period).to.equal(1n)
    expect(oldIncident.vaultPaidAmount).to.equal(collateral - fundedFlb)

    // The period-2 buyer's independently valid claim is approved next. It receives only
    // whatever vault capital remains after the previous-period claim (zero without FLB,
    // or 100/1,000 with the funded FLB), leaving the new cover materially underpaid.
    const period2CaptureTimestamp = BigInt(await time.latest())
    await submitIncident(2, period2CaptureTimestamp, 1, 'period-2-incident')
    await incidentManager.connect(assessmentApprover).approveCurrentAssessment(2)

    const receiverAfterNewPayout = await vaultAsset.balanceOf(ctx.payout_receiver.address)
    const [newIncident] = await incidentManager.getIncident(2)
    expect(newIncident.status).to.equal(4n) // CLOSED
    expect(newIncident.period).to.equal(2n)
    expect(newIncident.vaultPaidAmount).to.equal(fundedFlb)
    expect(receiverAfterNewPayout - receiverAfterOldPayout).to.equal(fundedFlb)
    expect(await vault.totalAssets()).to.equal(0n)
    })
  }
})
```
