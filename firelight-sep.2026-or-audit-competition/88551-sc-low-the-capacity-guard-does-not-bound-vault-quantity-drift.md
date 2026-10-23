# 88551 sc low the capacity guard does not bound vault quantity drift

**Submitted on Aug 13th 2026 at 19:03:19 UTC by @whitehorse for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88551
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`CoverOrderAllocator._validateMatchingCapacity` is the onchain bound on the capacity figure the offchain matching engine submits, widened by `divergenceToleranceBps` to absorb drift between the engine's snapshot and commit inclusion. It reads two of its three collateral inputs live and the third, vault collateral, at `totalAssetsAt(currentPeriodStart())`. Collateral leaving the vault mid period is therefore invisible to the bound at any magnitude, and the accepted figure is then pinned into the commitment and consumed by settlement without being requeried. Cover binds against collateral the vault no longer holds, at real backing below the floor the contract declares for its own configuration bounds.

## Vulnerability Details

`_computeAvailableCapacity` builds the period ceiling from three collateral terms:

```solidity
// CoverOrderAllocator.sol:700 to :715
uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);          // live
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer), ...       // live
);
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()), ...                 // period start
);
```

`totalAssetsAt` resolves through `Checkpoints.upperLookupRecent`, which returns the last key lower than or equal to the search key. `Checkpoints.push` appends at a strictly greater key or overwrites the tail on an equal one, and `pop` has no caller in the tree. The read therefore reflects the vault as of the period's first second, and `payout` (FirelightVault.sol:952), `_logTrace` (:1055) and `checkpointTotalAssets` (:525) all write strictly later keys that it cannot reach.

Two orderings reach the defect.

**Payout before commit.** The engine runs step 4 of the matching flow documented in `ICoverOrderAllocator.sol`, which reads live `vault.totalAssets()`, and submits the figure that was true when it ran. A payout lands before inclusion. An equivalent fall in the price or buffer leg is detected and rejected; this one is not, because the vault comparand cannot move.

**Payout after commit.** `commitAllocation` pins the accepted figure at :352. `_computeAvailableCapacity` has one call site, :400 inside `_validateMatchingCapacity`. `_settleCoverOrder` (:495 to :520) never reaches it and scales the per protocol concentration caps at :506 to :516 off the stored value. In this ordering the engine cannot have observed the payout at any point.

`recommitAllocation` reenters the same read, so the assurance at :375 to :377 that a recommit "stays bound to the period's real collateral within tolerance" does not hold across a payout.

Recommended fix, reading the vault leg live like its two siblings:

```solidity
// CoverOrderAllocator.sol:711
$.vault.totalAssets(),
```

That closes the commit path only. Settlement needs its own live floor before the concentration caps, `min(commit.totalAvailableCapacity, _capacityFromCollateral($, $.vault.totalAssets()))`, or the second ordering survives the fix.

## Impact Details

Selected impact: "Contract fails to deliver promised returns, but doesn't lose value."

`MAX_LEVERAGE_FACTOR` (:79 to :82) declares real backing behind sold cover floored at `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` "regardless of the configured `minCAR`", which is 18.18% at the contract's own ceilings. Vault quantity drift falsifies that floor.

Measured at the contract's permitted maxima, with a collateral loss of 30% of the vault's USD value:

| Quantity                                          | Value                                     |
| ------------------------------------------------- | ----------------------------------------- |
| capacity submitted by the matching engine         | $293,231,863                              |
| guard ceiling applied, unchanged by the payout    | $322,555,050                              |
| guard ceiling the remaining collateral supports   | $227,438,535                              |
| admitted above what remaining collateral supports | $65,793,328                               |
| real backing behind the settled cover             | 14.10% against a declared floor of 18.18% |

The tolerance the other two legs are held to is 10% and the loss applied here is 30%, so a bounded leg rejects it. The price leg does reject it, on identical collateral. Paying the vault leg out in full leaves backing at 0.34% with the ceiling unmoved, which establishes the drift as unlimited rather than merely oversized.

The under delivered promise lands on the cover purchasers settling in that window: they hold policies backed below the declared floor and pay the premium computed at :495 for cover priced as though it held.

No funds are lost. The defect admits and settles a commitment, it does not move assets, cover already settled is unaffected, and the next period start snapshot restores the correct basis. No role acts outside its documented authority.

## References

* `contracts/core/CoverOrderAllocator.sol:711`, vault collateral read at the period start key
* `contracts/core/CoverOrderAllocator.sol:75` to `:82`, the tolerance and the declared backing floor
* `contracts/core/CoverOrderAllocator.sol:328` to `:330`, the guard's documented purpose
* `contracts/core/CoverOrderAllocator.sol:352`, the accepted ceiling pinned into the commitment
* `contracts/core/CoverOrderAllocator.sol:394` to `:403`, `_validateMatchingCapacity`
* `contracts/core/CoverOrderAllocator.sol:495` to `:520`, settlement consuming the stored ceiling
* `contracts/core/interfaces/ICoverOrderAllocator.sol`, step 4 of the offchain matching flow
* `contracts/core/FirelightVault.sol:952`, the payout checkpoint push
* `contracts/core/lib/Checkpoints.sol:221` to `:224`, `upperLookupRecent`

## Proof of Concept

`poc_unbounded_vault_leg_divergence.js`, four tests, all passing on unmodified source. Build and run steps, parameters and per test expected output are in the file header.

Tests 1 and 2 route the same 30% collateral loss through the vault leg and the price leg: identical remaining collateral, identical honest ceiling of $227,438,535, accepted versus reverted `MatchingCapacityOverflow`. Test 3 is the unbounded case. Test 4 is the settle path ordering, where the concentration cap holds at $175,939,118 against $124,057,382 of real capacity and cover settles at 14.10% backing.

An isolation control patches line 711 alone to `$.vault.totalAssets()`. Tests 1 and 3 then revert, test 2 is unchanged, and test 4 passes identically, which is the evidence that the one line change is necessary and not sufficient.

The vault is the real `FirelightVault` behind its proxy, not `MockCoverOrderAllocatorVault`. Capacity is read without adding a getter, from the `maxCapacity` argument of the `MatchingCapacityOverflow` revert. Phase 2 is not deployed, so the tests run against the in scope source at the competition commit on the program's mandated harness.

## Proof of Concept

```
/*
 * PoC: CoverOrderAllocator._validateMatchingCapacity is documented as the bound on the
 * off-chain matcher's `matchingCapacity`, widened by `divergenceToleranceBps` to absorb
 * drift between the matcher's snapshot and commit inclusion. It bounds the price leg and
 * the first loss buffer leg, both of which it reads live. It does not bound the vault leg,
 * which it reads at `totalAssetsAt(currentPeriodStart())`: that resolves to the vault as of
 * the period's first second, and every write at a strictly later timestamp is unreachable
 * from it. Vault quantity drift is therefore admitted at any magnitude.
 *
 * HOW TO RUN
 *   git clone https://github.com/immunefi-team/audit-comp-firelight.git
 *   cd audit-comp-firelight && git checkout v1_audit_ready
 *   npm install
 *   cp <this file> test/poc_unbounded_vault_leg_divergence.js
 *   npx hardhat test test/poc_unbounded_vault_leg_divergence.js
 *
 * No contract is modified. Every call is a production external function on a production
 * contract. The vault is the real FirelightVault behind its proxy, not the repository's
 * MockCoverOrderAllocatorVault, so the payout, checkpoint and stale read run end to end
 * through production code.
 *
 * `_computeAvailableCapacity` is internal, so the guard's own comparand is read without
 * adding a getter: the test calls commitAllocation with an out of range matchingCapacity
 * and takes `maxCapacity` out of the MatchingCapacityOverflow(matchingCapacity, maxCapacity)
 * revert. That is the exact value the guard compares against.
 *
 * The matcher's number is computed by the formula the interface documents for it
 * (ICoverOrderAllocator.sol step 4 of the off-chain matching flow), which reads live
 * `vault.totalAssets()`. The matcher in these tests is honest: it always submits the
 * capacity that was true at the moment it ran.
 *
 * EXPERIMENT DESIGN
 *   Tests 1 and 2 apply the SAME collateral loss, 30% of the vault's USD value, through
 *   two different legs of the same expression, and compare the guard's verdict.
 *     Test 1  loss delivered as vault quantity (an approved incident payout)  -> ACCEPTED
 *     Test 2  loss delivered as vault price (the feed moves)                  -> REVERTED
 *   Real collateral behind the commitment is identical in both. Only the leg differs.
 *   Test 3 shows the vault leg admits drift of unbounded size, not merely oversized drift.
 *   Test 4 reverses the ordering: the commitment is made first and legitimately, with no
 *   payout yet, and the payout lands afterwards. Settlement then consumes the ceiling pinned
 *   at :352 without requerying the vault. Test 4 carries NO control branch and passes under
 *   BOTH arms, which is the evidence that reading :711 live is necessary but not sufficient.
 *
 * Parameters mirror Flare mainnet on 2026-08-13:
 *   vault proxy 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3, totalAssets() 57,359,574.888550 FXRP
 *   FXRP decimals 6, period duration 86400 s, XRP $1.005
 * Capacity config uses the contract's own hard ceilings for effectiveLeverage / minCAR and
 * divergenceToleranceBps. The defect is independent of that choice: the vault leg is frozen
 * at every admissible configuration. The configuration sets only the size of the tolerance
 * the other two legs enjoy, which is what tests 1 and 2 compare against.
 *
 * ISOLATION CONTROL (built into this file, both arms reproduce):
 *   TREATMENT, unmodified source:
 *       npx hardhat test test/poc_unbounded_vault_leg_divergence.js
 *   CONTROL, one line of source changed and nothing else:
 *       sed -i '711s|.*|            $.vault.totalAssets(),|' contracts/core/CoverOrderAllocator.sol
 *       FIXED=1 npx hardhat test test/poc_unbounded_vault_leg_divergence.js
 *     Under the control the vault leg becomes bounded exactly like the price leg: tests 1 and
 *     3 revert and the guard capacity tracks the outflow, while test 2 is unchanged, which
 *     confirms the patch touches only the vault leg. Test 4 passes under both arms with
 *     identical figures, because settlement never calls _computeAvailableCapacity (its only
 *     call site is :400, inside _validateMatchingCapacity).
 *
 * EXPECTED OUTPUT
 *   Unmodified source, `npx hardhat test`, 4 passing:
 *     Test 1  matcher submitted $293,231,863 | guard ceiling $322,555,050 unchanged by the
 *             payout | ceiling on remaining collateral $227,438,535 | commit ACCEPTED and
 *             both orders settled | real backing 14.10% vs declared floor 18.18%
 *     Test 2  same submitted $293,231,863 | guard ceiling $227,438,535, tracked the loss |
 *             commit REVERTED MatchingCapacityOverflow
 *     Test 3  vault FXRP balance 0 | guard ceiling $322,555,050 | ceiling on remaining
 *             collateral $5,500,000 | commit ACCEPTED | real backing 0.34%
 *     Test 4  commitment pinned $293,231,863 | stored ceiling unchanged after the payout |
 *             concentration cap from stored $175,939,118 | cap had it used live collateral
 *             $124,057,382 | cover settled $293,231,863 | real backing 14.10%
 *   Control arm, `FIXED=1` with line 711 patched, 4 passing:
 *     Tests 1 and 3 REVERT, test 2 unchanged, test 4 identical to the treatment arm.
 *
 * SCOPE OF WHAT THIS PROVES
 *   It proves the declared bound is absent on one leg of the expression. It does not
 *   demonstrate a loss of funds, and no test here claims one. Cover already committed is
 *   unaffected; the effect is confined to commitments admitted while the drift stands.
 */

const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault } = require('./setup/fixtures.js')

const FXRP_DECIMALS = 6n
const CANONICAL_DECIMALS = 18n
const PERIOD_DURATION = 86400
const VAULT_ASSETS = 57_359_574_888550n
const XRP_USD_1E18 = 1_005_000_000_000_000_000n
const FLB_BALANCE = 1_000_000_000000n

const MIN_CAR = 12_000n
const EFFECTIVE_LEVERAGE = 60_000n
const TOLERANCE_BPS = 1_000n
const BPS = 10_000n

// The collateral loss applied in tests 1 and 2. Chosen above TOLERANCE_BPS so that a leg the
// guard actually bounds is required to reject it.
const DRIFT_BPS = 3_000n

const CONTROL = process.env.FIXED === '1'

const PROTOCOL_MORPHO = 'Morpho'
const PROTOCOL_AAVE = 'Aave'
const MARKET_A = ethers.encodeBytes32String('market-A')
const MARKET_B = ethers.encodeBytes32String('market-B')

const abiCoder = ethers.AbiCoder.defaultAbiCoder()
const marketIdOf = (chainId, protocol, market) =>
  ethers.keccak256(abiCoder.encode(['uint64', 'string', 'bytes32'], [chainId, protocol, market]))

const scaleUp = (v) => v * 10n ** (CANONICAL_DECIMALS - FXRP_DECIMALS)

// The formula ICoverOrderAllocator documents for the off-chain matcher, step 4. Live inputs,
// no tolerance: the tolerance is the guard's allowance, not part of the matcher's number.
const matcherCapacity = (assets6, flb6, price) =>
  ((scaleUp(flb6) + (scaleUp(assets6) * price) / 10n ** 18n) * EFFECTIVE_LEVERAGE) / MIN_CAR

// What the guard admits: the same expression widened by divergenceToleranceBps.
const guardCeiling = (assets6, flb6, price) =>
  (matcherCapacity(assets6, flb6, price) * (BPS + TOLERANCE_BPS)) / BPS

const fmtUSD = (canonical) => '$' + (canonical / 10n ** 18n).toLocaleString('en-US')
const pct = (num, den) => (Number((num * 10000n) / den) / 100).toFixed(2) + '%'

// Canonical USD value of collateral the vault and buffer actually hold.
const realCollateral = (assets6, flb6, price) =>
  scaleUp(flb6) + (scaleUp(assets6) * price) / 10n ** 18n

// The floor CoverOrderAllocator.sol:79-82 declares the config bounds guarantee:
//   minCAR / (effectiveLeverage * (1 + tolerance)), in bps.
const DECLARED_BACKING_FLOOR_BPS = (MIN_CAR * BPS * BPS) / (EFFECTIVE_LEVERAGE * (BPS + TOLERANCE_BPS))

describe('CoverOrderAllocator divergence guard does not bound the vault leg', function () {
  this.timeout(600000)

  let ctx

  beforeEach(async () => {
    const v = await deployVault({
      decimals: Number(FXRP_DECIMALS),
      period_configuration_duration: PERIOD_DURATION,
      initial_deposit_limit: (VAULT_ASSETS * 2n).toString()
    })

    const [deployer] = await ethers.getSigners()

    const rand = async () => {
      const w = ethers.Wallet.createRandom().connect(ethers.provider)
      await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
      return w
    }
    const admin = await rand()
    const adminRole = await rand()
    const curator = await rand()
    const allocatorRole = await rand()
    const configAdmin = await rand()
    const premiumCollector = await rand()
    const firstLossBuffer = await rand()
    const buyer = await rand()

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.connect(deployer).deploy(18, XRP_USD_1E18)

    const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFTFactory,
      ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' }
    )

    const markets = [
      { chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A },
      { chainId: 10, protocol: PROTOCOL_AAVE, market: MARKET_B }
    ]

    const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Allocator,
      [
        {
          vault: await v.firelight_vault.getAddress(),
          premiumCollector: premiumCollector.address,
          coverNFT: await coverNFT.getAddress(),
          priceFeedAdapter: await priceFeed.getAddress(),
          maxPriceAge: 7 * 24 * 3600,
          premiumTokens: [await usdc.getAddress()],
          admin: admin.address,
          adminRole: adminRole.address,
          curatorRole: curator.address,
          allocatorRole: allocatorRole.address,
          configAdminRole: configAdmin.address,
          initialProtocolConcentrations: [
            { protocol: PROTOCOL_MORPHO, chainId: 1, maxProtocolConcentrationBps: 6000 },
            { protocol: PROTOCOL_AAVE, chainId: 10, maxProtocolConcentrationBps: 4000 }
          ],
          newMarkets: markets,
          capacityConfig: {
            minCAR: MIN_CAR,
            firstLossBufferToken: await usdc.getAddress(),
            firstLossBuffer: firstLossBuffer.address,
            effectiveLeverage: EFFECTIVE_LEVERAGE,
            minOrderMarketCoverAmount: 1,
            divergenceToleranceBps: TOLERANCE_BPS
          }
        }
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
    )

    await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())
    await usdc.mint(firstLossBuffer.address, FLB_BALANCE)

    ctx = {
      ...v,
      allocator,
      coverNFT,
      usdc,
      priceFeed,
      admin,
      curator,
      allocatorRole,
      configAdmin,
      premiumCollector,
      firstLossBuffer,
      buyer,
      marketIdA: marketIdOf(1, PROTOCOL_MORPHO, MARKET_A),
      marketIdB: marketIdOf(10, PROTOCOL_AAVE, MARKET_B)
    }
  })

  // Reads the guard's own comparand out of the MatchingCapacityOverflow revert.
  const readGuardCapacity = async () => {
    const period = await ctx.firelight_vault.currentPeriod()
    try {
      await ctx.allocator
        .connect(ctx.allocatorRole)
        .commitAllocation.staticCall(period, ethers.id('probe'), 0, 2n ** 200n)
      throw new Error('probe did not revert: capacity is unbounded')
    } catch (e) {
      const parsed = ctx.allocator.interface.parseError(e.data)
      expect(parsed.name).to.equal('MatchingCapacityOverflow')
      return parsed.args[1]
    }
  }

  // Funds the vault and advances until the deposit checkpoint sits strictly before the
  // current period start, so totalAssetsAt(periodStart) is the funded figure.
  const fundAndOpenPeriod = async () => {
    const vault = ctx.firelight_vault
    await ctx.utils.mintAndApprove(VAULT_ASSETS, ctx.users[0])
    await vault.connect(ctx.users[0]).deposit(VAULT_ASSETS, ctx.users[0].address)
    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS)
    await time.increaseTo(BigInt(await vault.currentPeriodEnd()) + 60n)
    return { P: await vault.currentPeriod(), startP: await vault.currentPeriodStart() }
  }

  it('admits a commitment whose collateral basis left the vault mid period, at drift far beyond divergenceToleranceBps', async () => {
    const vault = ctx.firelight_vault
    const asset = ctx.token_contract

    const { startP } = await fundAndOpenPeriod()

    // Orders are authored for the period that the commitment below will cover.
    const cover = matcherCapacity(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18)
    const coverMorpho = (cover * 6000n) / BPS
    const coverAave = (cover * 4000n) / BPS

    await ctx.allocator
      .connect(ctx.curator)
      .createCoverOrder(ctx.buyer.address, ctx.buyer.address, 'beneficiary', await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdA, coverRateAnnual: 500, coverAmount: coverMorpho }], 0)
    await ctx.allocator
      .connect(ctx.curator)
      .createCoverOrder(ctx.buyer.address, ctx.buyer.address, 'beneficiary', await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdB, coverRateAnnual: 500, coverAmount: coverAave }], 0)

    await ctx.usdc.mint(ctx.buyer.address, 10_000_000_000000n)
    await ctx.usdc.connect(ctx.buyer).approve(await ctx.allocator.getAddress(), 10_000_000_000000n)

    await time.increaseTo(BigInt(await vault.currentPeriodEnd()) + 60n)
    const P = await vault.currentPeriod()
    const startOfP = await vault.currentPeriodStart()

    // The honest matcher runs here, on live inputs, per the documented step 4.
    const submitted = matcherCapacity(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18)
    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS)

    // An approved incident is paid between the matcher's snapshot and commit inclusion.
    const D = (VAULT_ASSETS * DRIFT_BPS) / BPS
    const vaultBalBefore = await asset.balanceOf(await vault.getAddress())
    await vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, D, startOfP)

    // The FXRP is gone: a real ERC20 transfer out, not an accounting move.
    expect(vaultBalBefore - (await asset.balanceOf(await vault.getAddress()))).to.equal(D)
    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS - D)
    // The payout checkpoint went in at a key strictly greater than the period start.
    expect(await vault.totalAssetsAt(startOfP)).to.equal(VAULT_ASSETS)

    const ceiling = await readGuardCapacity()
    const honestCeiling = guardCeiling(VAULT_ASSETS - D, FLB_BALANCE, XRP_USD_1E18)

    // The submitted number now exceeds what the remaining collateral supports even after
    // the full tolerance is applied. A leg the guard bounds would have to reject it.
    expect(submitted).to.be.gt(honestCeiling)

    const commitTx = () =>
      ctx.allocator
        .connect(ctx.allocatorRole)
        .commitAllocation(P, buildRoot(coverMorpho, coverAave, ctx), coverMorpho + coverAave, submitted)

    if (CONTROL) {
      expect(ceiling).to.equal(honestCeiling)
      await expect(commitTx())
        .to.be.revertedWithCustomError(ctx.allocator, 'MatchingCapacityOverflow')
        .withArgs(submitted, honestCeiling)
      console.log('\n  CONTROL ARM, vault leg read live: commitAllocation REVERTED as the price leg does\n')
      return
    }

    // Treatment: the comparand did not move at all across the payout.
    expect(ceiling).to.equal(guardCeiling(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18))
    await commitTx()
    expect((await ctx.allocator.getAllocationCommitment(P)).totalAvailableCapacity).to.equal(submitted)

    // The commitment is not inert: cover is written against it.
    const tree = treeOf(coverMorpho, coverAave, ctx)
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0, [{ marketId: ctx.marketIdA, allocatedCover: coverMorpho }], proofFor(tree, 0))
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(1, [{ marketId: ctx.marketIdB, allocatedCover: coverAave }], proofFor(tree, 1))
    expect(await ctx.coverNFT.ownerOf(0)).to.equal(ctx.buyer.address)
    expect(await ctx.coverNFT.ownerOf(1)).to.equal(ctx.buyer.address)

    // The declared backing floor at :79-82 is a property of the config bounds alone. Measure
    // it against the cover actually settled and the collateral actually remaining.
    const settled = (await ctx.allocator.getAllocationCommitment(P)).totalSettledCover
    const remaining = realCollateral(VAULT_ASSETS - D, FLB_BALANCE, XRP_USD_1E18)
    const backingBps = (remaining * BPS) / settled
    expect(backingBps).to.be.lt(DECLARED_BACKING_FLOOR_BPS)

    console.log('')
    console.log('  TEST 1  loss delivered through the VAULT QUANTITY leg')
    console.log('    collateral loss applied            ', pct(DRIFT_BPS, BPS), 'of vault USD value')
    console.log('    divergence tolerance               ', pct(TOLERANCE_BPS, BPS))
    console.log('    matcher submitted                  ', fmtUSD(submitted))
    console.log('    guard ceiling used                 ', fmtUSD(ceiling), ' (unchanged by the payout)')
    console.log('    ceiling on remaining collateral    ', fmtUSD(honestCeiling))
    console.log('    submitted over honest ceiling      ', fmtUSD(submitted - honestCeiling))
    console.log('    commitAllocation                    ACCEPTED, cover settled')
    console.log('    cover settled                      ', fmtUSD(settled))
    console.log('    real backing behind settled cover  ', pct(backingBps, BPS))
    console.log('    declared floor at :79-82           ', pct(DECLARED_BACKING_FLOOR_BPS, BPS))
    console.log('')
  })

  it('rejects the identical collateral loss when it arrives through the price leg', async () => {
    const vault = ctx.firelight_vault
    await fundAndOpenPeriod()
    await time.increaseTo(BigInt(await vault.currentPeriodEnd()) + 60n)
    const P = await vault.currentPeriod()

    // Same honest matcher, same snapshot, same submitted number as test 1.
    const submitted = matcherCapacity(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18)

    // The same 30% collateral loss, delivered through price instead of quantity.
    const drifted = (XRP_USD_1E18 * (BPS - DRIFT_BPS)) / BPS
    await ctx.priceFeed.setAnswer(drifted)

    const ceiling = await readGuardCapacity()
    const honestCeiling = guardCeiling(VAULT_ASSETS, FLB_BALANCE, drifted)

    // Real collateral behind the commitment is identical to test 1: the vault holds the
    // same USD value in both, and the guard's honest ceiling is the same number.
    expect(honestCeiling).to.equal(guardCeiling((VAULT_ASSETS * (BPS - DRIFT_BPS)) / BPS, FLB_BALANCE, XRP_USD_1E18))
    expect(ceiling).to.equal(honestCeiling)

    await expect(
      ctx.allocator.connect(ctx.allocatorRole).commitAllocation(P, ethers.id('root'), 0, submitted)
    )
      .to.be.revertedWithCustomError(ctx.allocator, 'MatchingCapacityOverflow')
      .withArgs(submitted, honestCeiling)

    console.log('')
    console.log('  TEST 2  identical loss delivered through the PRICE leg')
    console.log('    matcher submitted                  ', fmtUSD(submitted))
    console.log('    guard ceiling used                 ', fmtUSD(ceiling), ' (tracked the loss)')
    console.log('    commitAllocation                    REVERTED MatchingCapacityOverflow')
    console.log('')
    console.log('    Same collateral, same magnitude, opposite verdict from test 1.')
    console.log('')
  })

  it('admits vault leg drift of unbounded size', async () => {
    const vault = ctx.firelight_vault
    const asset = ctx.token_contract
    await fundAndOpenPeriod()
    await time.increaseTo(BigInt(await vault.currentPeriodEnd()) + 60n)
    const P = await vault.currentPeriod()
    const startOfP = await vault.currentPeriodStart()

    const submitted = matcherCapacity(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18)

    // payout caps at min(assetsAt(capturePeriodStart), live payable), so the whole vault leg
    // is payable in one call.
    await vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, VAULT_ASSETS, startOfP)
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(0n)
    expect(await vault.totalAssets()).to.equal(0n)
    expect(await vault.totalAssetsAt(startOfP)).to.equal(VAULT_ASSETS)

    const ceiling = await readGuardCapacity()
    const honestCeiling = guardCeiling(0n, FLB_BALANCE, XRP_USD_1E18)

    if (CONTROL) {
      expect(ceiling).to.equal(honestCeiling)
      await expect(
        ctx.allocator.connect(ctx.allocatorRole).commitAllocation(P, ethers.id('root'), 0, submitted)
      )
        .to.be.revertedWithCustomError(ctx.allocator, 'MatchingCapacityOverflow')
        .withArgs(submitted, honestCeiling)
      console.log('\n  CONTROL ARM, empty vault: commitAllocation REVERTED\n')
      return
    }

    expect(ceiling).to.equal(guardCeiling(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18))
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(P, ethers.id('root'), 0, submitted)
    expect((await ctx.allocator.getAllocationCommitment(P)).totalAvailableCapacity).to.equal(submitted)

    // Backing behind the ADMITTED ceiling, the first loss buffer being all that remains.
    const remaining = realCollateral(0n, FLB_BALANCE, XRP_USD_1E18)
    const backingBps = (remaining * BPS) / submitted
    expect(backingBps).to.be.lt(DECLARED_BACKING_FLOOR_BPS)

    console.log('')
    console.log('  TEST 3  vault leg drift is unbounded, not merely oversized')
    console.log('    vault FXRP balance                 ', (await asset.balanceOf(await vault.getAddress())).toString())
    console.log('    guard ceiling used                 ', fmtUSD(ceiling), ' (still the pre payout figure)')
    console.log('    ceiling on remaining collateral    ', fmtUSD(honestCeiling), ' (first loss buffer alone)')
    console.log('    commitAllocation                    ACCEPTED at', fmtUSD(submitted))
    console.log('    real backing behind admitted cap   ', pct(backingBps, BPS))
    console.log('    declared floor at :79-82           ', pct(DECLARED_BACKING_FLOOR_BPS, BPS))
    console.log('')
  })

  // This test carries NO control branch on purpose. It must pass identically under the
  // unmodified source and under FIXED=1, because settlement never calls
  // _computeAvailableCapacity: the only call site is :400, inside _validateMatchingCapacity.
  // Passing under both arms is what shows the read at :711 is not the whole defect, and that
  // reading it live does not close this ordering.
  it('settles against the pinned ceiling after a payout, under both arms', async () => {
    const vault = ctx.firelight_vault
    const asset = ctx.token_contract

    await fundAndOpenPeriod()

    // The matching engine's number, computed on live inputs per the documented step 4.
    const C = matcherCapacity(VAULT_ASSETS, FLB_BALANCE, XRP_USD_1E18)
    const coverMorpho = (C * 6000n) / BPS
    const coverAave = (C * 4000n) / BPS

    await ctx.allocator
      .connect(ctx.curator)
      .createCoverOrder(ctx.buyer.address, ctx.buyer.address, 'beneficiary', await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdA, coverRateAnnual: 500, coverAmount: coverMorpho }], 0)
    await ctx.allocator
      .connect(ctx.curator)
      .createCoverOrder(ctx.buyer.address, ctx.buyer.address, 'beneficiary', await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdB, coverRateAnnual: 500, coverAmount: coverAave }], 0)

    await ctx.usdc.mint(ctx.buyer.address, 10_000_000_000000n)
    await ctx.usdc.connect(ctx.buyer).approve(await ctx.allocator.getAddress(), 10_000_000_000000n)

    await time.increaseTo(BigInt(await vault.currentPeriodEnd()) + 60n)
    const P = await vault.currentPeriod()
    const startOfP = await vault.currentPeriodStart()

    // Commit happens FIRST, with no payout yet. Live and period start collateral agree here,
    // so this commitment is legitimate under either arm and the guard is not being evaded.
    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS)
    expect(await vault.totalAssetsAt(startOfP)).to.equal(VAULT_ASSETS)

    await ctx.allocator
      .connect(ctx.allocatorRole)
      .commitAllocation(P, buildRoot(coverMorpho, coverAave, ctx), C, C)

    const commitBefore = await ctx.allocator.getAllocationCommitment(P)
    expect(commitBefore.totalAvailableCapacity).to.equal(C)

    // Wait out the settlement grace, staying inside period P.
    if (BigInt(await time.latest()) < BigInt(commitBefore.graceExpiresAt)) {
      await time.increaseTo(BigInt(commitBefore.graceExpiresAt))
    }

    // The approved incident is paid AFTER the commitment is already pinned. In this ordering
    // the matching engine could not have observed the payout at any point.
    const D = (VAULT_ASSETS * DRIFT_BPS) / BPS
    const vaultBalBefore = await asset.balanceOf(await vault.getAddress())
    await vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, D, startOfP)
    expect(vaultBalBefore - (await asset.balanceOf(await vault.getAddress()))).to.equal(D)
    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS - D)

    // Settlement proceeds against the stored commitment.
    const tree = treeOf(coverMorpho, coverAave, ctx)
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0, [{ marketId: ctx.marketIdA, allocatedCover: coverMorpho }], proofFor(tree, 0))
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(1, [{ marketId: ctx.marketIdB, allocatedCover: coverAave }], proofFor(tree, 1))
    expect(await ctx.coverNFT.ownerOf(0)).to.equal(ctx.buyer.address)
    expect(await ctx.coverNFT.ownerOf(1)).to.equal(ctx.buyer.address)

    const commitAfter = await ctx.allocator.getAllocationCommitment(P)

    // The per protocol concentration cap at :506-516 is protocolConcentrationBps applied to
    // commit.totalAvailableCapacity. The stored ceiling is unchanged by the payout, so the cap
    // is still the pre payout figure and the Morpho leg settles exactly at it.
    expect(commitAfter.totalAvailableCapacity).to.equal(C)
    const capFromStored = (6000n * commitAfter.totalAvailableCapacity) / BPS
    expect(capFromStored).to.equal(coverMorpho)

    // Had the cap been scaled off collateral the vault actually holds, the same leg would
    // have exceeded it and settlement would have reverted ProtocolConcentrationOverflow.
    const honestAfter = matcherCapacity(VAULT_ASSETS - D, FLB_BALANCE, XRP_USD_1E18)
    const capIfLive = (6000n * honestAfter) / BPS
    expect(coverMorpho).to.be.gt(capIfLive)

    // Real backing behind the cover that settled, against the floor declared at :79-82.
    const settled = commitAfter.totalSettledCover
    expect(settled).to.equal(C)
    const remaining = realCollateral(VAULT_ASSETS - D, FLB_BALANCE, XRP_USD_1E18)
    const backingBps = (remaining * BPS) / settled
    expect(backingBps).to.be.lt(DECLARED_BACKING_FLOOR_BPS)

    console.log('')
    console.log('  TEST 4  payout lands AFTER the commitment', CONTROL ? '(CONTROL ARM)' : '(TREATMENT ARM)')
    console.log('    commitment pinned at               ', fmtUSD(C), ' (legitimate, no payout yet)')
    console.log('    payout executed after the commit   ', D.toString(), 'FXRP(6d) transferred out')
    console.log('    stored ceiling after the payout    ', fmtUSD(commitAfter.totalAvailableCapacity), ' (unchanged)')
    console.log('    concentration cap from stored      ', fmtUSD(capFromStored))
    console.log('    cap had it used live collateral    ', fmtUSD(capIfLive), ' (settlement would have reverted)')
    console.log('    cover settled                      ', fmtUSD(settled))
    console.log('    real backing behind settled cover  ', pct(backingBps, BPS))
    console.log('    declared floor at :79-82           ', pct(DECLARED_BACKING_FLOOR_BPS, BPS))
    console.log('    identical under both arms: reading :711 live does not close this ordering')
    console.log('')
  })
})

function treeOf(coverMorpho, coverAave, ctx) {
  return StandardMerkleTree.of(
    [
      [0, [[ctx.marketIdA, coverMorpho]]],
      [1, [[ctx.marketIdB, coverAave]]]
    ],
    ['uint256', '(bytes32,uint256)[]']
  )
}
function buildRoot(coverMorpho, coverAave, ctx) {
  return treeOf(coverMorpho, coverAave, ctx).root
}
function proofFor(tree, orderId) {
  for (const [i, leaf] of tree.entries()) {
    if (BigInt(leaf[0]) === BigInt(orderId)) return tree.getProof(i)
  }
  throw new Error('leaf not found')
}


```
