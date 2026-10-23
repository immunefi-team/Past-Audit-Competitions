# 88321 sc high stale period start collateral snapshot lets the allocator sell cover against assets already paid out as claims

**Submitted on Aug 12th 2026 at 18:49:41 UTC by @pawelschubert for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88321
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Bug Description

`CoverOrderAllocator._computeAvailableCapacity()` values the staked collateral backing new cover sales at a **historical snapshot anchored to the start of the current period**:

```solidity
// CoverOrderAllocator.sol:710-716
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

However, `FirelightVault.payout()` - the function that pays approved incident claims - is explicitly allowed to execute during the incident's capture period **or the following period**:

```solidity
// FirelightVault.sol:1130-1132
function _isPeriodInPayoutWindow(uint256 period, uint256 current) internal pure returns (bool) {
    return current == period || current == period + 1;
}
```

When a payout executes, it pushes the post-slash `totalAssets` checkpoint at the **current** timestamp:

```solidity
// FirelightVault.sol:952
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

That timestamp is always strictly greater than `currentPeriodStart()`. Therefore, when a period-P-1 incident is paid out early in period P (the normal, documented settlement flow), `totalAssetsAt(currentPeriodStart())` keeps returning the **pre-slash** balance for the rest of period P.

Every `commitAllocation` / `recommitAllocation` in period P then validates `matchingCapacity` against collateral that no longer exists:

```solidity
// CoverOrderAllocator.sol:394-403
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
```

The check is fully binding - one wei above the stale bound reverts - but it is bound to phantom collateral. The only documented slack, `divergenceToleranceBps`, can only **widen** the bound (`strictCapacity * (BPS + tolerance) / BPS`); it cannot tighten it after a slash. Even with `divergenceToleranceBps = 0` (strict mode), a commit sized against a 90%-drained vault succeeds.

There is also no on-chain recovery path: `recommitAllocation` re-uses the same stale `_computeAvailableCapacity`, so even a diligent operator cannot re-derive post-slash capacity for the remainder of the period. Notably, the protocol's own interface documentation specifies the off-chain capacity formula with the **live** balance (`ICoverOrderAllocator.sol:40-41`: `vault.totalAssets() * assetPriceUSD / ...`), which contradicts the on-chain snapshot anchor exactly in this scenario - the on-chain ceiling is supposed to track the same real collateral.

The asymmetry makes the design intent explicit: **deposits are blocked during and after active incidents** precisely so that fresh assets cannot back unresolved exposure (`_hasActiveIncident`, FirelightVault.sol:1115-1118), yet **commits are never blocked** and the snapshot pretends destroyed collateral still exists.

### Step-by-step failure path (PoC-verified numbers)

Setup: 6-decimal vault asset at $1.00, `minCAR = 1.2x`, `effectiveLeverage = 2.0x` (well within the post-M-1 5x cap), `divergenceToleranceBps = 0`, first-loss buffer empty (isolates the vault leg).

1. Period 0: depositor deposits 10,000 fXRP.
2. Period 1: curator creates a 12,000 USD cover order targeting period 2. An incident is captured in period 1 (payout window = periods 1-2).
3. Period 2 begins. Early in period 2 - inside the legal payout window - the period-1 incident pays out 9,000 fXRP. Live `totalAssets() = 1,000`. `totalAssetsAt(period2Start)` still returns **10,000**.
4. `commitAllocation(2, root, 12_000, matchingCapacity = 20_000 USD)`:
   * On-chain bound: 10,000 × 2.0 = **20,000 USD (stale)** - commit succeeds.
   * Honest bound from live collateral: 1,000 × 2.0 = **2,000 USD**.
   * `20,000 + 1 wei` reverts `MatchingCapacityOverflow` - the check is binding, just on collateral that no longer exists.
5. `settleCoverOrder` succeeds: premium pulled, CoverNFT minted, 12,000 USD of live period-2 cover.
6. A period-2 incident claims the covered 12,000 USD → `payout()` caps at live assets → pays only **1,000**. **91.7% of the approved claim is unpayable.**

## Impact

**Protocol insolvency.** Cover is sold at `effectiveLeverage/minCAR` against collateral already paid out as claims, and subsequent approved claims on that cover cannot be paid. The on-chain solvency invariant - the *only* on-chain constraint binding the privileged allocation flow - silently stops constraining for the remainder of any period in which a payout executes. Cover buyers hold NFTs backing unpayable claims; stakers face total slash; the vault's remaining assets are a fraction of outstanding cover.

No misbehavior is required to trigger this: incident payouts executing in the period after capture, and once-per-period commits inside the period, are both the normal documented operating flow. The trigger is timing-inevitable in any period following a large incident, which is exactly when continued cover sales matter most.

This is not any known/accepted issue:

* **Not M-1** (unbounded leverage): leverage is within the enforced cap; the flaw is the collateral *input* to the capacity formula, not the multiplier.
* **Not the accepted "aggregate claims not capped vs cover sold" risk**: that is claims exceeding *sold cover*; this is *sold cover* exceeding *actual collateral* - the inverse direction, independent of assessment honesty.
* **Not the unescrowed-buffer accepted risk**: the first-loss buffer is empty in the PoC; the bug is entirely in the vault-asset leg.
* **Not L-2/L-4**: all caps and CAR bounds are respected and enforced.

## Recommendation

Value the vault collateral with the minimum of snapshot and live balance, which can never overstate:

```solidity
uint256 vaultAssets = Math.min(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vault.totalAssets()
);
```

and/or block `commitAllocation`/`recommitAllocation` while any incident captured in the current or previous period has executed a payout during the current period.

## Link to Proof of Concept

<https://gist.github.com/schvbertt/c636d7224c2f0481e7a6cbf4ce24769b>

## Proof of Concept

Runnable Hardhat test (uses the **real** `FirelightVault` via the repo's own `test/setup/fixtures.js`, real `CoverOrderAllocator`, real `CoverNFT`, real OZ `StandardMerkleTree`; only tokens/price feed are mocks).

Setup:

```bash
git clone https://github.com/firelight-protocol/firelight-core.git
cd firelight-core
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62   # frozen competition commit (branch v1_audit_ready)
npm install
mkdir -p test/poc
# save the file below as test/poc/poc_capacity_stale_snapshot.js
npx hardhat test test/poc/poc_capacity_stale_snapshot.js
```

Expected output: `✔ sells 2x-leveraged cover against collateral already paid out as claims` (1 passing).

```js
// PoC: CoverOrderAllocator capacity check ignores same-period vault slashes.
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId } = require('../setup/fixtures.js')

const e18 = (n) => ethers.parseUnits(n.toString(), 18)
const e6 = (n) => ethers.parseUnits(n.toString(), 6)

const randomSigner = async () => {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
  return wallet
}

const PROTOCOL = 'Morpho'
const MARKET_A = ethers.encodeBytes32String('market-A')

const deploySystem = async () => {
  // Real vault: 6-decimals fXRP, 1-week periods, 50k deposit limit.
  const ctx = await deployVault()
  const { firelight_vault, token_contract } = ctx

  const admin = await randomSigner()
  const adminRole = await randomSigner()
  const curator = await randomSigner()
  const allocatorRole = await randomSigner()
  const configAdmin = await randomSigner()
  const premiumCollector = await randomSigner()
  const flbWallet = await randomSigner() // stays empty: isolate the vault-asset leg

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const premiumToken = await MockERC20.deploy('MockUSDC', 'mUSDC', 6)
  const flbToken = await MockERC20.deploy('MockFLB', 'mFLB', 6)

  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, e18(1)) // $1.00

  const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ['Firelight Cover', 'FLCOVER', '', admin.address, ctx.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const markets = [{ chainId: 1, protocol: PROTOCOL, market: MARKET_A }]
  const initialProtocolConcentrations = [
    { protocol: PROTOCOL, chainId: 1, maxProtocolConcentrationBps: 6000 } // 60% concentration cap
  ]
  const capacityConfig = {
    minCAR: 12000,            // 1.2x (MIN_CAR_BPS floor)
    firstLossBufferToken: await flbToken.getAddress(),
    firstLossBuffer: flbWallet.address,
    effectiveLeverage: 24000, // capacity multiplier = 24000/12000 = 2.0x
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0 // STRICT: zero tolerance - any drift should be rejected
  }

  const initParams = {
    vault: await firelight_vault.getAddress(),
    premiumCollector: premiumCollector.address,
    coverNFT: await coverNFT.getAddress(),
    priceFeedAdapter: await priceFeed.getAddress(),
    maxPriceAge: 7 * 24 * 3600,
    premiumTokens: [await premiumToken.getAddress()],
    admin: admin.address,
    adminRole: adminRole.address,
    curatorRole: curator.address,
    allocatorRole: allocatorRole.address,
    configAdminRole: configAdmin.address,
    initialProtocolConcentrations,
    newMarkets: markets,
    capacityConfig
  }

  const AllocatorFactory = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(AllocatorFactory, [initParams], {
    kind: 'transparent',
    unsafeAllow: ['missing-initializer-call']
  })

  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  return {
    ...ctx,
    allocator, coverNFT, priceFeed, premiumToken, flbToken,
    admin, adminRole, curator, allocatorRole, configAdmin, premiumCollector, flbWallet,
    marketIdA: computeMarketId(1, PROTOCOL, MARKET_A)
  }
}

describe('PoC: capacity commit ignores same-period vault slash', function () {
  it('sells 2x-leveraged cover against collateral already paid out as claims', async function () {
    const ctx = await loadFixture(deploySystem)
    const { firelight_vault, token_contract, allocator, priceFeed, premiumToken, coverNFT } = ctx
    const [depositor] = ctx.users
    const buyer = ctx.users[1]

    // ---- Period 0: user deposits 10,000 fXRP (vault epoch = deployment time) ----
    const DEPOSIT = e6(10_000)
    await ctx.utils.mintAndApprove(DEPOSIT, depositor)
    await firelight_vault.connect(depositor).deposit(DEPOSIT, depositor.address)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(DEPOSIT)

    // Buyer funds premium allowance (generous, like a real cover buyer approval)
    await premiumToken.mint(buyer.address, e6(1_000))
    await premiumToken.connect(buyer).approve(allocator.target, e6(1_000))

    // ---- Warp into period 1; curator creates the cover order targeting period 2 ----
    await time.increaseTo((await firelight_vault.currentPeriodEnd()) + 1n)
    expect(await firelight_vault.currentPeriod()).to.equal(1n)

    const COVER = e18(12_000) // 12,000 USD canonical = 60% concentration cap of expected capacity
    const RATE = 500          // 5% annual
    await allocator.connect(ctx.curator).createCoverOrder(
      buyer.address, buyer.address, buyer.address,
      await premiumToken.getAddress(),
      [{ marketId: ctx.marketIdA, coverRateAnnual: RATE, coverAmount: COVER }],
      0 // NEW
    )
    const orderId = 0n

    // An incident is captured in period 1 (its payout window covers periods 1 and 2).
    const captureTimestamp = (await ethers.provider.getBlock('latest')).timestamp

    // ---- Warp into period 2 ----
    await time.increaseTo((await firelight_vault.currentPeriodEnd()) + 1n)
    expect(await firelight_vault.currentPeriod()).to.equal(2n)
    const startOfP2 = await firelight_vault.currentPeriodStart()

    // ---- Early in period 2: the period-1 incident pays out 9,000 of the 10,000 fXRP ----
    await firelight_vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, e6(9_000), captureTimestamp)
    const liveAssets = await firelight_vault.totalAssets()
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(e6(1_000))
    expect(liveAssets).to.equal(e6(1_000))

    // The crux: the period-2 capacity anchor still reports the PRE-SLASH balance.
    const snapshotAssets = await firelight_vault.totalAssetsAt(startOfP2)
    expect(snapshotAssets).to.equal(DEPOSIT) // 10,000 - slash invisible to the snapshot
    expect(snapshotAssets).to.equal(10n * liveAssets)

    // Refresh the (fresh) oracle answer: price is NOT the issue here.
    await priceFeed.setAnswer(e18(1))

    // ---- Commit for period 2 ----
    // On-chain bound: capacity = 10,000 USD x 2.0 leverage = 20,000 USD (stale).
    // Honest bound from LIVE collateral: 1,000 USD x 2.0 = 2,000 USD.
    const STALE_CAPACITY = e18(20_000)
    const LIVE_CAPACITY = e18(2_000)
    const tree = StandardMerkleTree.of(
      [[orderId, [[ctx.marketIdA, COVER]]]],
      ['uint256', '(bytes32,uint256)[]']
    )
    const proof = tree.getProof(0)

    // The capacity check IS binding - 1 wei above the stale bound reverts...
    await expect(
      allocator.connect(ctx.allocatorRole).commitAllocation(2, tree.root, COVER, STALE_CAPACITY + 1n)
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')

    // ...but it is bound to collateral that no longer exists: the full 20,000 USD
    // commit succeeds although only 1,000 USD of collateral remains (tolerance = 0!).
    await expect(
      allocator.connect(ctx.allocatorRole).commitAllocation(2, tree.root, COVER, STALE_CAPACITY)
    ).to.emit(allocator, 'AllocationCommitted')

    const commit = await allocator.getAllocationCommitment(2)
    expect(commit.totalAvailableCapacity).to.equal(STALE_CAPACITY)
    expect(commit.totalAvailableCapacity).to.equal(10n * LIVE_CAPACITY)

    // ---- Settle the 12,000 USD cover order against the drained vault ----
    await expect(
      allocator.connect(ctx.allocatorRole).settleCoverOrder(
        orderId,
        [{ marketId: ctx.marketIdA, allocatedCover: COVER }],
        proof
      )
    ).to.emit(allocator, 'CoverOrderSettled')

    // Buyer holds a live CoverNFT for 12,000 USD of period-2 cover; premium was pulled.
    expect(await coverNFT.ownerOf(orderId)).to.equal(buyer.address)
    const premiumPaid = await premiumToken.balanceOf(ctx.premiumCollector.address)
    expect(premiumPaid).to.be.gt(0n)
    const order = await allocator.getCoverOrder(orderId)
    expect(order.status).to.equal(1n) // MATCHED
    expect(order.allocatedCoverAmount).to.equal(COVER)

    // ---- Impact: a period-2 incident claim against the sold cover is unpayable ----
    // Claim the full 12,000 USD covered loss (= 12,000 fXRP at $1).
    const capture2 = (await ethers.provider.getBlock('latest')).timestamp
    const receiverBefore = await token_contract.balanceOf(ctx.payout_receiver.address)
    await firelight_vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, e6(12_000), capture2)
    const receiverAfter = await token_contract.balanceOf(ctx.payout_receiver.address)

    // Only the 1,000 fXRP actually left in the vault can be paid out:
    // 12,000 USD of cover was sold, 1,000 USD is recoverable -> 91.7% of the claim fails.
    expect(receiverAfter - receiverBefore).to.equal(e6(1_000))
  })
})
```
