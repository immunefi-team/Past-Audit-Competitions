# 88565 sc high capacity uses a period start snapshot a payout cannot lower so cover is sold against collateral already paid out

**Submitted on Aug 13th 2026 at 20:34:18 UTC by @FaithfulLayney for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88565
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`_computeAvailableCapacity` values the vault leg of collateral at `vault.totalAssetsAt(vault.currentPeriodStart())`, while `FirelightVault.payout` records its post-payout total at `Time.timestamp()` and `Checkpoints.upperLookupRecent` returns the last checkpoint with key **≤** the search key. A payout executed at any moment after the period's first second is therefore excluded from the period-start lookup by construction. Assets already transferred out to pay a claim keep counting as collateral for the rest of the period, so `commitAllocation` accepts — and `settleCoverOrder` sells — cover the capital-adequacy floor was written to forbid. The contract's own constants bound the overstatement at **5.5× the amount paid out**, persisting until the next period boundary: one day on the deployed vault. No attacker is involved; two individually-correct operator actions in their natural order are enough.

## Vulnerability Details

Of the three collateral and price inputs to capacity, two are read live and only the vault leg is a snapshot:

```solidity
// contracts/core/CoverOrderAllocator.sol:696-725
        uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);          // :700  LIVE
            ...
        uint256 firstLossBufferUSDCanonical = Decimals.convert(
            config.firstLossBufferToken.balanceOf(config.firstLossBuffer),           // :705  LIVE
            ...
        uint256 totalAssetsCanonical = Decimals.convert(
            $.vault.totalAssetsAt($.vault.currentPeriodStart()),                     // :711  STALE
            ...
        uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

The two terms of `availableCollateralCanonical` are read as of two different times, and they are exactly the two pots the payout waterfall (`IncidentManager._executePayout`, `:621-706`; the buffer draw at `:658`, the vault draw at `:684`) draws from. The first-loss-buffer leg self-corrects after a payout; the vault leg cannot:

```solidity
// contracts/core/FirelightVault.sol:456-458
    function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
        return _traceTotalAssets.upperLookupRecent(timestamp);
    }

// contracts/core/FirelightVault.sol:952  (inside payout)
            _traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

`Checkpoints.upperLookupRecent` (`Checkpoints.sol:217-224`) returns the last checkpoint with a key lower than or equal to the search key, and the payout's key is `Time.timestamp()`, greater than `currentPeriodStart()` for every payout except one mined in the exact first second of the period — which the PoC confirms from the other side, as a control: a payout landing on that exact second *does* lower the snapshot.

Nothing compensates. There is no per-period "already paid out" accumulator anywhere in the allocator, capacity is validated only at commit or recommit, and settlement never revalidates it: the committed figure is stored at `:352` (or `:381` on recommit) and thereafter read only at `:507`, as the base the per-protocol concentration cap is a percentage of. The stale number is consulted exactly once, at `:401` (`matchingCapacity <= maxCapacity`), with `:402` chaining `totalAllocated <= matchingCapacity` onto it — and `:388-392` documents that this is deliberate (*"A commitment stores the capacity THE MATCHER RAN AGAINST … not a capacity recomputed at transaction time"*), which is precisely why the single commit-time check has to be correct.

A period-start valuation is defensible for *exposure*: withdrawal requests made during the period stay slashable, and `payout` handles that explicitly by adding `withdrawAssets[capturePeriod+1]`/`[capturePeriod+2]` back into `activePayableAmount` (`:916-926`). A payout is categorically different — those tokens are transferred out and can never back anything again. `payout`'s own comment says as much, calling the period-start figure *"an exposure cap, not a segregated asset bucket"* (`FirelightVault.sol:909-910`): the vault treats it as exposure a payout consumes, while the allocator keeps selling against the same number as if untouched.

The period-start basis is documented, but in one direction only. The `@dev` at `CoverOrderAllocator.sol:690-691` states it, and the sponsor's tests pin the intent on the deposit side: `test/coverOrderAllocator/commitAllocation.js:230` asserts *"mid-period deposits do NOT inflate capacity"*, and `:248` asserts a commit reverts when it exceeds snapshot-based capacity even if live `totalAssets` would cover it. Both fix the snapshot below live, making it a conservative floor against an increase. The outflow direction is untested: across the four sponsor test files that call `commitAllocation` — `testVectors.js`, `commitAllocation.js`, `capacityConfigCheckpoints.js`, `premiumTokenDecimals.js` — the string `payout` does not appear once.

Both calls are role-gated and neither role misbehaves: a payout during period `P` — its window is exactly `{capturePeriod, capturePeriod+1}` (`FirelightVault.sol:1130-1132`) — followed by the designed one-commit-per-period allocation for `P` is the ordinary sequence. The interface's own documented recipe computes capacity from live `vault.totalAssets()` (`ICoverOrderAllocator.sol:40-46`), so even a matcher that reads correctly is unprotected: the on-chain check accepts anything up to the stale figure. The constraint that gets bypassed is one the contract asserts about itself three lines above the leverage constant: *"Real backing behind sold cover is `minCAR / (effectiveLeverage × (1 + tolerance))`, so bounding the ratio floors it at `1 / (MAX_LEVERAGE_FACTOR × (1 + tolerance))` regardless of the configured `minCAR`"* (`CoverOrderAllocator.sol:79-81`) — an absolute floor of 1/5.5 ≈ 18.18% real backing, enforced only by the check at `:401`, which reads the stale figure. PoC-030 part 2b drives that floor to **0%**. This is the exception the scope page's "Privileged actors" rule carves out; that rule reads, in full: *"Any bug report that requires a privileged role holder to act within the permissions explicitly granted to that role is out of scope, unless the report demonstrates that the role can exceed its intended permissions or bypass an intended protocol constraint."* Neither role exceeds its permissions — acting strictly within them is what bypasses the constraint.

The fix is to value the vault leg at the live figure the sponsor's own recipe already specifies — `$.vault.totalAssets()` at `:711` — or, if a period-start basis must be kept for exposure, to subtract a per-period paid-out accumulator from it before applying leverage.

## Impact Details

Protocol insolvency: cover is issued against collateral that has already left the vault. The overstatement is

```
overstatement = paidFromActiveThisPeriod × assetPriceUSD / 10**priceFeedDecimals
              × (effectiveLeverage / minCAR) × (1 + divergenceToleranceBps / 10_000)
```

bounded by the contract's own constants — `MAX_LEVERAGE_FACTOR = 5` (`:82`, enforced `:665-666`) and `divergenceToleranceBps ≤ 1_000` (`:78`, enforced `:668-669`) — at **5.5× the amount paid out**, and it persists to the next period boundary. Nobody gains: the victims are the LPs whose remaining stake now backs more cover than the capital-adequacy policy allows, and the buyers settled into the over-issued tranche who pay premium for cover the vault cannot pay.

Measured on the real `FirelightVault` wired to the real `CoverOrderAllocator`:

| Run                                           | Config                                                        | Deposited      | Paid out in-period | `totalAssets()` after | Capacity still granted | Result                                                                                                                          |
| --------------------------------------------- | ------------------------------------------------------------- | -------------- | ------------------ | --------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| PoC-010 #1                                    | `minCAR=12000`, `effectiveLeverage=12000` (1.0×), tolerance 0 | 30,000 fXRP    | 15,000             | 15,000                | 30,000e18              | 15,000e18 canonical USD of cover with zero backing; real backing 0.5 against this configuration's floor of 1.0 — 2× over-issued |
| PoC-010 #2 (claim from `P-1` paid during `P`) | same                                                          | 30,000 fXRP    | 20,000             | 10,000                | 30,000e18              | 3× the real collateral                                                                                                          |
| PoC-030 part 2                                | `minCAR=12000`, `effectiveLeverage=24000` (2.0×), tolerance 0 | 1,000,000 fXRP | 500,000            | 500,000               | 2,000,000e18           | 2,000,000e18 of cover settled; real backing 2500 bps against a 5000 bps policy floor                                            |
| PoC-030 part 2b                               | same                                                          | 1,000,000 fXRP | 100% of the vault  | 0                     | 2,000,000e18           | 1,200,000e18 of cover committed and settled against $0 of staked collateral                                                     |

In the first two runs the bound is proved exactly rather than inferred: `commitAllocation` with `matchingCapacity = staleFigure` is accepted, and with `staleFigure + 1` reverts `MatchingCapacityOverflow(staleFigure+1, staleFigure)`. The on-chain maximum *is* the pre-payout snapshot.

No external precondition is involved — no price move, no depeg, no oracle misbehaviour, no third party; the PoC holds the price at $1.00 and the first-loss buffer at zero. The scope page's known-issues list discloses that a USD price decrease reduces capacity, and that severe collateral drawdowns *"can reduce coverage capacity and may cause shortfalls or undercollateralization relative to outstanding cover obligations"*; the price leg is read live at `:700`, so the price case behaves as disclosed, and both disclosures describe capacity that reduces. This is the opposite case: a collateral outflow at constant price against which capacity provably does not reduce — the allocator returns the identical `maxCapacity` before and after (2,000,000e18 both times, with `totalAssets()` halved between them).

The in-scope Phase-2 allocator is not deployed yet; the vault proxy it will be wired to, `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` on Flare, currently runs the out-of-scope Phase-1 implementation. Read-only `eth_call` at block 67152319: `totalAssets() = 58,333,517.103995 fXRP`, `periodConfigurations(0) = (epoch 1764682969, duration 86400, startingPeriod 0)` — a one-day period. At that TVL a single 10% incident payout (≈ 5.83M fXRP) would license up to ≈ 32M fXRP-equivalent of cover with no collateral behind it before the next boundary; 5.5× is the enforced ceiling, not a configured live value.

## References

Repository `immunefi-team/audit-comp-firelight` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`:

* [`CoverOrderAllocator.sol#L696-L725`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L725) — `_computeAvailableCapacity`; the stale read is `:711`
* [`FirelightVault.sol#L456-L458`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L456-L458) — `totalAssetsAt`
* [`FirelightVault.sol#L892-L966`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L966) — `payout`; the checkpoint push at `:952`, the withdrawal add-back at `:916-926`
* [`lib/Checkpoints.sol#L217-L224`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L217-L224) — `upperLookupRecent` returns the last key ≤ the search key
* [`ICoverOrderAllocator.sol#L40-L41`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/ICoverOrderAllocator.sol#L40-L41) — the sponsor's own recipe uses live `vault.totalAssets()`

## Proof of Concept

### Step-by-step

Roles: `CURATOR_ROLE` (books the cover order, `:198`), `PAYOUT_ROLE` (executes an approved claim), `ALLOCATOR_ROLE` (commits the period's matching tree at `:337` and settles at `:446`); the capacity config is set by `CONFIG_ADMIN_ROLE` at initialization. No attacker, and no user behaviour beyond the ordinary product flow — LPs deposit, the buyer approves premium: the capacity overstatement itself is produced by two individually-correct operator actions in their natural order.

1. **LPs stake.** The vault's `totalAssets()` and its period-start checkpoint agree.
2. **Period `P` begins.** `currentPeriodStart()` is fixed for the rest of the period, and `_computeAvailableCapacity` will read the vault leg at exactly that key (`CoverOrderAllocator.sol:711`).
3. **A claim is paid during `P`.** `FirelightVault.payout` transfers assets out and pushes the reduced total at `Time.timestamp()` (`:952`) — a key strictly greater than `currentPeriodStart()`. `totalAssets()` drops; `totalAssetsAt(currentPeriodStart())` does not, because `upperLookupRecent` only considers keys ≤ the search key.
4. **The allocator commits `P`'s matching tree.** `commitAllocation` validates `matchingCapacity` against the stale figure and accepts it. The bound is proved exactly: the stale figure is accepted and `staleFigure + 1` reverts `MatchingCapacityOverflow(staleFigure+1, staleFigure)`.
5. **Cover settles against it.** `settleCoverOrder` consumes `commit.totalAvailableCapacity` with no revalidation (`:507-509`), so the over-issued cover becomes live cover.
6. **Control 1:** with no in-period payout, the same commit is bounded by the real collateral. **Control 2:** a payout mined in the exact period-start second *does* lower the snapshot — which isolates the checkpoint key as the root cause.

Net result: cover in force 30,000e18 canonical USD against 15,000e18 of real collateral — real backing 0.5 against this configuration's floor of 1.0, i.e. twice the cover the policy permits, with no attacker and no external condition.

### Runnable test A — the bound is exactly the stale snapshot

Reproduced from a clean checkout at the competition commit with this single file added. **Local Hardhat only; no transaction was sent to any live or public network.**

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62      # branch v1_audit_ready HEAD
npm install                                                # postinstall runs patch-package
mkdir -p test/poc && cp h010_capacity_stale_period_start_snapshot.js test/poc/
npx hardhat test test/poc/h010_capacity_stale_period_start_snapshot.js
```

`test/poc/h010_capacity_stale_period_start_snapshot.js`:

```javascript
// PoC 010 — CoverOrderAllocator._computeAvailableCapacity values staked collateral from a
// period-START checkpoint (`totalAssetsAt(currentPeriodStart())`, CoverOrderAllocator.sol:710-715)
// while FirelightVault.payout writes its post-payout checkpoint at `Time.timestamp()`
// (FirelightVault.sol:952), i.e. strictly AFTER the period start.
//
// Consequence: assets that have already left the vault through an incident payout executed
// inside the current period are still counted as collateral when the period's cover capacity
// is computed. The first-loss-buffer term of the same sum IS read live
// (`firstLossBufferToken.balanceOf(...)`, :704-709) — the two terms of
// `availableCollateralCanonical` are read at different times.
//
// This test uses the REAL FirelightVault rather than MockCoverOrderAllocatorVault. The mock CAN
// return a period-start value distinct from the live one — `setTotalAssetsAtSnapshot` (:48-51) sets
// `_hasSnapshot`, after which `totalAssetsAt` (:198-199) returns
// `_hasSnapshot ? _totalAssetsAtSnapshot : _totalAssets` — and the sponsor's own suite uses exactly
// that setter at test/coverOrderAllocator/commitAllocation.js:237. But the mock's `payout` (:251) is
// `pure` and moves nothing, so a mock-based test could only restate a hand-set divergence. Only the
// real vault shows that a real `payout()` PRODUCES the divergence, via the checkpoint push at
// FirelightVault.sol:952 keyed on `Time.timestamp()`. That is the actual claim.
//
// Run:  npx hardhat test test/poc/h010_capacity_stale_period_start_snapshot.js

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId, computeProtocolConcentrationHash } = require('../setup/fixtures.js')

const abiCoder = ethers.AbiCoder.defaultAbiCoder()

const PROTOCOL = 'Morpho'
const MARKET = ethers.encodeBytes32String('market-A')

const randomSigner = async () => {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
  return wallet
}

// Deploys the real vault plus a real CoverOrderAllocator wired to it.
async function setup () {
  const ctx = await loadFixture(
    deployVault.bind(null, {
      decimals: 6,                              // FXRP has 6 decimals on Flare
      initial_deposit_limit: '1000000000000',   // 1,000,000 FXRP
      period_configuration_duration: 604800     // 1 week
    })
  )

  const deployer = ctx.deployer
  const admin = await randomSigner()
  const curator = await randomSigner()
  const allocatorRole = await randomSigner()
  const configAdmin = await randomSigner()
  const adminRole = await randomSigner()
  const premiumCollector = await randomSigner()
  const flbWallet = await randomSigner()
  const buyer = await randomSigner()
  const payoutRecipient = await randomSigner()

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)   // premium + FLB token

  // $1.00 per FXRP with 18-dec feed => canonical USD == FXRP units * 1e12. Keeps the
  // arithmetic in the assertions exact and independent of any price move.
  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.connect(deployer).deploy(18, 10n ** 18n)

  const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const capacityConfig = {
    minCAR: 12000,               // MIN_CAR_BPS
    firstLossBufferToken: await usdc.getAddress(),
    firstLossBuffer: flbWallet.address,   // deliberately left at zero balance
    effectiveLeverage: 12000,    // lev/minCAR == 1.0 so capacity == collateral, exactly
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0    // no tolerance: the bound is the raw formula
  }

  const initParams = {
    vault: ctx.firelight_vault.target,
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
    initialProtocolConcentrations: [{ protocol: PROTOCOL, chainId: 1, maxProtocolConcentrationBps: 10000 }],
    newMarkets: [{ chainId: 1, protocol: PROTOCOL, market: MARKET }],
    capacityConfig
  }

  const Factory = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(Factory, [initParams], {
    kind: 'transparent',
    unsafeAllow: ['missing-initializer-call']
  })

  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  return {
    ...ctx, allocator, coverNFT, usdc, priceFeed,
    admin, curator, allocatorRole, configAdmin, premiumCollector, flbWallet, buyer, payoutRecipient,
    marketId: computeMarketId(1, PROTOCOL, MARKET),
    concHash: computeProtocolConcentrationHash(1, PROTOCOL)
  }
}

describe('POC-010 capacity uses a stale period-start assets checkpoint', function () {
  it('sells a full period of cover against collateral that a payout already removed', async () => {
    const c = await setup()
    const vault = c.firelight_vault
    const DEPOSIT = 30_000_000_000n        // 30,000 FXRP (6 dec)
    const PAYOUT  = 15_000_000_000n        // 15,000 FXRP — exactly half
    const E12 = 10n ** 12n                 // FXRP(6) -> canonical USD(18) at $1.00

    // ---- period P0: LPs deposit -------------------------------------------------
    await c.utils.mintAndApprove(DEPOSIT, c.users[0])
    await vault.connect(c.users[0]).deposit(DEPOSIT, c.users[0].address)
    expect(await vault.totalAssets()).to.equal(DEPOSIT)

    // ---- a cover order is booked for the next period ----------------------------
    const p0 = await vault.currentPeriod()
    await c.allocator.connect(c.curator).createCoverOrder(
      c.buyer.address,
      c.payoutRecipient.address,
      'rXRPbeneficiary',
      await c.usdc.getAddress(),
      [{ marketId: c.marketId, coverRateAnnual: 500, coverAmount: DEPOSIT * E12 }], // 30,000 USD of cover
      0 // NEW
    )
    const orderId = 0n
    const order = await c.allocator.getCoverOrder(orderId)
    expect(order.period).to.equal(p0 + 1n)

    // ---- roll into period P1 ----------------------------------------------------
    await time.increase(604800)
    await c.priceFeed.setAnswer(10n ** 18n)   // refresh the feed after the time jump
    const p1 = await vault.currentPeriod()
    expect(p1).to.equal(p0 + 1n)

    const periodStart = await vault.currentPeriodStart()
    const snapshotAtPeriodStart = await vault.totalAssetsAt(periodStart)
    expect(snapshotAtPeriodStart).to.equal(DEPOSIT)

    // ---- an incident captured inside P1 is paid out, inside P1 ------------------
    // (this is the ordinary flow: IncidentManager.approveCurrentAssessment ->
    //  FirelightVault.payout, with capturePeriod == currentPeriod)
    const captureTimestamp = periodStart
    await vault.connect(c.payout_signer).payout(c.payout_receiver.address, PAYOUT, captureTimestamp)

    // Half the collateral is gone.
    expect(await vault.totalAssets()).to.equal(DEPOSIT - PAYOUT)
    expect(await c.token_contract.balanceOf(vault.target)).to.equal(DEPOSIT - PAYOUT)

    // ...but the period-start checkpoint the capacity formula reads is untouched,
    // because payout pushed its checkpoint at Time.timestamp() > currentPeriodStart().
    expect(await vault.totalAssetsAt(periodStart)).to.equal(DEPOSIT)

    // ---- the allocator still grants the FULL pre-payout capacity ----------------
    const staleCapacity  = DEPOSIT * E12                   // 30,000 USD (18 dec)
    const honestCapacity = (DEPOSIT - PAYOUT) * E12        // 15,000 USD (18 dec)

    // one wei above the stale figure is rejected -> proves maxCapacity == staleCapacity
    const leaves = [[orderId, [[c.marketId, DEPOSIT * E12]]]]
    const tree = StandardMerkleTree.of(leaves, ['uint256', '(bytes32,uint256)[]'])

    await expect(
      c.allocator.connect(c.allocatorRole).commitAllocation(p1, tree.root, staleCapacity, staleCapacity + 1n)
    ).to.be.revertedWithCustomError(c.allocator, 'MatchingCapacityOverflow')
      .withArgs(staleCapacity + 1n, staleCapacity)

    // exactly the stale figure is accepted
    await c.allocator.connect(c.allocatorRole).commitAllocation(p1, tree.root, staleCapacity, staleCapacity)

    const commit = await c.allocator.getAllocationCommitment(p1)
    expect(commit.totalAvailableCapacity).to.equal(staleCapacity)
    expect(commit.totalAvailableCapacity).to.equal(honestCapacity * 2n)  // 2x the real collateral

    // ---- and the cover actually settles at that size ---------------------------
    const premium18 = await (async () => {
      const o = await c.allocator.getCoverOrder(orderId)
      return o.totalPremiumAmount
    })()
    const premiumNative = (premium18 + 10n ** 12n - 1n) / 10n ** 12n   // ceil to 6 dec
    await c.usdc.mint(c.buyer.address, premiumNative)
    await c.usdc.connect(c.buyer).approve(await c.allocator.getAddress(), premiumNative)

    await c.allocator.connect(c.allocatorRole).settleCoverOrder(
      orderId,
      [{ marketId: c.marketId, allocatedCover: DEPOSIT * E12 }],
      tree.getProof(0)
    )

    const settled = await c.allocator.getCoverOrder(orderId)
    expect(settled.status).to.equal(1) // MATCHED
    expect(settled.allocatedCoverAmount).to.equal(DEPOSIT * E12)

    // ---- the invariant that is broken ------------------------------------------
    // cover in force for period P1, valued in USD
    const coverInForceUSD = settled.allocatedCoverAmount
    // collateral actually available to pay a P1 claim, valued in USD
    const flbUSD = (await c.usdc.balanceOf(c.flbWallet.address)) * E12
    const realCollateralUSD = (await vault.totalAssets()) * E12 + flbUSD

    console.log('    cover in force  (USD 18d):', coverInForceUSD.toString())
    console.log('    real collateral (USD 18d):', realCollateralUSD.toString())
    console.log('    unbacked cover  (USD 18d):', (coverInForceUSD - realCollateralUSD).toString())

    // The configured backing floor is effectiveLeverage/minCAR = 12000/12000 = 1.0, i.e. the
    // contract promised >= 1.00 USD of collateral per USD of cover. It has 0.50.
    expect(realCollateralUSD).to.equal(honestCapacity)
    expect(coverInForceUSD).to.equal(realCollateralUSD * 2n)
  })

  // The operationally natural variant: an incident captured late in period P-1 is assessed
  // and paid early in period P (payout window = {P-1, P}), and the matcher commits P's
  // allocation afterwards. No incident of period P has to exist at all.
  it('previous-period claim paid inside period P is invisible to period P capacity', async () => {
    const c = await setup()
    const vault = c.firelight_vault
    const DEPOSIT = 30_000_000_000n
    const PAYOUT  = 20_000_000_000n
    const E12 = 10n ** 12n

    await c.utils.mintAndApprove(DEPOSIT, c.users[0])
    await vault.connect(c.users[0]).deposit(DEPOSIT, c.users[0].address)

    // roll to period P-1, then to period P; both period-start snapshots read DEPOSIT
    await time.increase(604800)
    const pPrev = await vault.currentPeriod()
    const prevStart = await vault.currentPeriodStart()

    await time.increase(604800)
    await c.priceFeed.setAnswer(10n ** 18n)
    const p = await vault.currentPeriod()
    const pStart = await vault.currentPeriodStart()
    expect(p).to.equal(pPrev + 1n)
    expect(await vault.totalAssetsAt(pStart)).to.equal(DEPOSIT)

    // claim for the PREVIOUS period, executed now (allowed: current == capturePeriod + 1)
    await vault.connect(c.payout_signer).payout(c.payout_receiver.address, PAYOUT, prevStart)
    expect(await vault.totalAssets()).to.equal(DEPOSIT - PAYOUT)          // 10,000 FXRP left

    // period P capacity is still computed off the untouched period-start checkpoint
    expect(await vault.totalAssetsAt(pStart)).to.equal(DEPOSIT)

    await expect(
      c.allocator.connect(c.allocatorRole).commitAllocation(p, ethers.id('root'), 0, DEPOSIT * E12 + 1n)
    ).to.be.revertedWithCustomError(c.allocator, 'MatchingCapacityOverflow')
      .withArgs(DEPOSIT * E12 + 1n, DEPOSIT * E12)

    await c.allocator.connect(c.allocatorRole).commitAllocation(p, ethers.id('root'), DEPOSIT * E12, DEPOSIT * E12)
    const commit = await c.allocator.getAllocationCommitment(p)
    // 30,000 USD of capacity granted against 10,000 FXRP (= 10,000 USD) of real collateral
    expect(commit.totalAvailableCapacity).to.equal(DEPOSIT * E12)
    expect(commit.totalAvailableCapacity).to.equal((await vault.totalAssets()) * E12 * 3n)
  })

  it('control: with no in-period payout the same commit is bounded by the real collateral', async () => {
    const c = await setup()
    const vault = c.firelight_vault
    const DEPOSIT = 30_000_000_000n
    const E12 = 10n ** 12n

    await c.utils.mintAndApprove(DEPOSIT, c.users[0])
    await vault.connect(c.users[0]).deposit(DEPOSIT, c.users[0].address)

    await time.increase(604800)
    await c.priceFeed.setAnswer(10n ** 18n)
    const p1 = await vault.currentPeriod()

    // no payout this period -> the bound equals the live collateral
    await expect(
      c.allocator.connect(c.allocatorRole).commitAllocation(p1, ethers.id('root'), 0, DEPOSIT * E12 + 1n)
    ).to.be.revertedWithCustomError(c.allocator, 'MatchingCapacityOverflow')

    await c.allocator.connect(c.allocatorRole).commitAllocation(p1, ethers.id('root'), 0, DEPOSIT * E12)
    expect(await vault.totalAssets()).to.equal(DEPOSIT)
  })
})
```

Output (one run; runtimes are machine-dependent, every other value is deterministic):

```
  POC-010 capacity uses a stale period-start assets checkpoint
    cover in force  (USD 18d): 30000000000000000000000
    real collateral (USD 18d): 15000000000000000000000
    unbacked cover  (USD 18d): 15000000000000000000000
    ✔ sells a full period of cover against collateral that a payout already removed (650ms)
    ✔ previous-period claim paid inside period P is invisible to period P capacity (134ms)
    ✔ control: with no in-period payout the same commit is bounded by the real collateral (120ms)


  3 passing (905ms)
```

### Test B — end-to-end commit → settle, the 100%-payout extreme, and the checkpoint-key control

A second test wires the production allocator to the production vault at `minCAR = 12000`, `effectiveLeverage = 24000` (2.0×) and drives the full commit → settle path. Same repository, same commit, same run instructions, substituting this file's name in the `cp` and `npx hardhat test` lines; add as `test/poc/030_capacity_stale_period_start.js`. Full source:

```javascript
// PoC 030 — "Cover capacity is valued at a period-start snapshot that a mid-period payout
//             can never lower, so cover is committed and settled against collateral that has
//             already left the vault."
//
// Root cause:
//   CoverOrderAllocator._computeAvailableCapacity (CoverOrderAllocator.sol:696-725, esp. :710-715)
//   prices the vault leg of collateral as
//        vault.totalAssetsAt(vault.currentPeriodStart())
//   FirelightVault.totalAssetsAt (:456-457) is Checkpoints.upperLookupRecent, i.e. the last
//   checkpoint whose key is <= the search key.
//   FirelightVault.payout (:952) records the post-payout total at Time.timestamp(), which for any
//   payout executed after the period boundary is STRICTLY GREATER than currentPeriodStart().
//   The period-start snapshot therefore cannot be lowered by a payout that happens inside the
//   period, and every capacity check for the rest of that period keeps using pre-payout assets.
//
// Run:  npx hardhat test test/poc/030_capacity_stale_period_start.js

const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault, computeMarketId, computeProtocolConcentrationHash } = require("../setup/fixtures.js");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");
const { ethers } = require("hardhat");

const DAY = 86400;
const E6 = 10n ** 6n;
const E18 = 10n ** 18n;
const BPS = 10_000n;

const PROTOCOL_MORPHO = "Morpho";
const PROTOCOL_AAVE = "Aave";
const MARKET_A = ethers.encodeBytes32String("market-A");
const MARKET_B = ethers.encodeBytes32String("market-B");

const randomSigner = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider);
  await ethers.provider.send("hardhat_setBalance", [w.address, "0x3635C9ADC5DEA00000"]);
  return w;
};

const buildTree = (leaves) => {
  const tree = StandardMerkleTree.of(leaves, ["uint256", "(bytes32,uint256)[]"]);
  return { tree, root: tree.root };
};
const getProof = (tree, orderId) => {
  for (const [i, leaf] of tree.entries()) if (BigInt(leaf[0]) === BigInt(orderId)) return tree.getProof(i);
  throw new Error("order not in tree");
};

// ------------------------------------------------------------------------------------------
// Real stack: the production FirelightVault behind a proxy + the production CoverOrderAllocator
// pointed at it. No MockCoverOrderAllocatorVault — the whole point is the vault<->allocator
// interaction through totalAssetsAt()/currentPeriodStart().
// ------------------------------------------------------------------------------------------
async function deployRealStack() {
  const { upgrades } = require("hardhat");
  // Vault: 1-day periods, matching the live Flare deployment
  // (periodConfigurations(0) on 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 = epoch 1764682969,
  //  duration 86400, startingPeriod 0).
  const ctx = await deployVault({
    period_configuration_duration: DAY,
    initial_deposit_limit: (10n ** 15n).toString(),
  });

  const deployer = ctx.deployer;
  const admin = await randomSigner();
  const curator = await randomSigner();
  const allocatorRole = await randomSigner();
  const configAdmin = await randomSigner();
  const adminRole = await randomSigner();
  const premiumCollector = await randomSigner();
  const firstLossBufferWallet = await randomSigner(); // deliberately holds 0 buffer token
  const buyer1 = await randomSigner();
  const buyer2 = await randomSigner();

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const usdc = await MockERC20.connect(deployer).deploy("MockUSDC", "mUSDC", 6);

  const MockAggregator = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregator.connect(deployer).deploy(18, E18); // 1 fXRP = $1.00

  const CoverNFTFactory = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ["Firelight Cover", "FLCOVER", "", admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" }
  );

  const markets = [
    { chainId: 1, protocol: PROTOCOL_MORPHO, market: MARKET_A },
    { chainId: 10, protocol: PROTOCOL_AAVE, market: MARKET_B },
  ];

  const capacityConfig = {
    minCAR: 12000, // 1.2x  -> protocol policy: cover <= collateral * leverage / minCAR
    firstLossBufferToken: await usdc.getAddress(),
    firstLossBuffer: firstLossBufferWallet.address,
    effectiveLeverage: 24000, // capacity multiplier = 24000/12000 = 2.0x collateral
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0, // no slack: what the contract accepts is exactly its own policy
  };

  const Factory = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Factory,
    [
      {
        vault: await ctx.firelight_vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * DAY,
        premiumTokens: [await usdc.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          { protocol: PROTOCOL_MORPHO, chainId: 1, maxProtocolConcentrationBps: 6000 },
          { protocol: PROTOCOL_AAVE, chainId: 10, maxProtocolConcentrationBps: 4000 },
        ],
        newMarkets: markets,
        capacityConfig,
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] }
  );

  await coverNFT.connect(admin).grantRole(ethers.id("MINTER_ROLE"), await allocator.getAddress());

  return {
    ...ctx,
    allocator,
    coverNFT,
    usdc,
    priceFeed,
    admin,
    curator,
    allocatorRole,
    configAdmin,
    premiumCollector,
    firstLossBufferWallet,
    buyer1,
    buyer2,
    capacityConfig,
    marketIdA: computeMarketId(1, PROTOCOL_MORPHO, MARKET_A),
    marketIdB: computeMarketId(10, PROTOCOL_AAVE, MARKET_B),
  };
}

// Reads the allocator's own `maxCapacity` by over-asking and decoding MatchingCapacityOverflow.
async function readMaxCapacity(ctx) {
  const period = await ctx.firelight_vault.currentPeriod();
  const huge = 2n ** 200n;
  try {
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .commitAllocation.staticCall(period, ethers.id("probe"), 0, huge);
    throw new Error("probe unexpectedly succeeded");
  } catch (e) {
    const data = e.data ?? e.error?.data ?? e.info?.error?.data;
    const parsed = ctx.allocator.interface.parseError(data);
    expect(parsed.name).to.equal("MatchingCapacityOverflow");
    return parsed.args[1]; // maxCapacity
  }
}

const nextPeriod = async (vault) => {
  const end = await vault.currentPeriodEnd();
  await time.increaseTo(Number(end) + 5);
};

describe("PoC 030 — capacity uses a period-start snapshot a mid-period payout cannot lower", function () {
  this.timeout(600000);

  // ------------------------------------------------------------------------------------------
  // Part 1 — the primitive. After a payout, the vault reports two different truths:
  //   totalAssets()                            = post-payout (real)
  //   totalAssetsAt(currentPeriodStart())      = pre-payout  (what the allocator prices)
  // ------------------------------------------------------------------------------------------
  it("part 1: payout lowers totalAssets() but NOT totalAssetsAt(currentPeriodStart())", async () => {
    const ctx = await loadFixture(deployRealStack);
    const v = ctx.firelight_vault;

    const DEPOSIT = 1_000_000n * E6; // 1,000,000 fXRP
    await ctx.utils.mintAndApprove(DEPOSIT, ctx.users[0]);
    await v.connect(ctx.users[0]).deposit(DEPOSIT, ctx.users[0].address);

    await nextPeriod(v); // roll into a fresh period; the deposit checkpoint is now < periodStart

    const periodStart = await v.currentPeriodStart();
    expect(await v.totalAssetsAt(periodStart)).to.equal(DEPOSIT);
    expect(await v.totalAssets()).to.equal(DEPOSIT);

    // PAYOUT_ROLE pays a claim for an incident captured in THIS period.
    const PAY = 500_000n * E6;
    const now = BigInt(await time.latest());
    await v.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, PAY, now);

    const afterTotal = await v.totalAssets();
    const afterSnapshot = await v.totalAssetsAt(periodStart);

    console.log("        totalAssets() after payout      :", afterTotal.toString());
    console.log("        totalAssetsAt(periodStart)      :", afterSnapshot.toString());

    expect(afterTotal).to.equal(DEPOSIT - PAY); // real assets halved
    expect(afterSnapshot).to.equal(DEPOSIT); // the priced snapshot did not move

    // and the payout checkpoint really was written at a key strictly after the period start
    expect(BigInt(await time.latest())).to.be.greaterThan(BigInt(periodStart));
    // reading one second later already sees the reduction — the key is > periodStart, not <=
    expect(await v.totalAssetsAt(BigInt(await time.latest()))).to.equal(DEPOSIT - PAY);
  });

  // ------------------------------------------------------------------------------------------
  // Part 2 — the loss. Full production flow: orders -> payout -> commit -> settle.
  // Cover is sold at 2x the amount the protocol's own capacity policy allows for the
  // collateral that actually remains.
  // ------------------------------------------------------------------------------------------
  it("part 2: cover is committed and settled at 2x the collateral that actually remains", async () => {
    const ctx = await loadFixture(deployRealStack);
    const v = ctx.firelight_vault;
    const a = ctx.allocator;

    const DEPOSIT = 1_000_000n * E6; // 1,000,000 fXRP == $1,000,000
    await ctx.utils.mintAndApprove(DEPOSIT, ctx.users[0]);
    await v.connect(ctx.users[0]).deposit(DEPOSIT, ctx.users[0].address);

    // period P: curator books next-period orders (createCoverOrder targets currentPeriod()+1)
    await nextPeriod(v);
    const orderPeriod = (await v.currentPeriod()) + 1n;

    const COVER_A = 1_200_000n * E18; // 60% of the pre-payout capacity -> Morpho cap
    const COVER_B = 800_000n * E18; // 40% of the pre-payout capacity -> Aave cap
    await a
      .connect(ctx.curator)
      .createCoverOrder(
        ctx.buyer1.address,
        ctx.buyer1.address,
        "beneficiary-1",
        await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdA, coverRateAnnual: 500, coverAmount: COVER_A }],
        0
      );
    await a
      .connect(ctx.curator)
      .createCoverOrder(
        ctx.buyer2.address,
        ctx.buyer2.address,
        "beneficiary-2",
        await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdB, coverRateAnnual: 500, coverAmount: COVER_B }],
        0
      );

    for (const b of [ctx.buyer1, ctx.buyer2]) {
      await ctx.usdc.mint(b.address, 10n ** 12n);
      await ctx.usdc.connect(b).approve(await a.getAddress(), 10n ** 12n);
    }

    // period P+1: the settle window for those orders
    await nextPeriod(v);
    expect(await v.currentPeriod()).to.equal(orderPeriod);
    await ctx.priceFeed.setAnswer(E18); // keep the feed fresh

    const capacityBeforePayout = await readMaxCapacity(ctx);
    expect(capacityBeforePayout).to.equal(2_000_000n * E18); // 2.0x * $1,000,000

    // ---- a claim for an incident captured in THIS period is paid out, mid-period ----
    const PAY = 500_000n * E6; // half the vault leaves
    const nowTs = BigInt(await time.latest());
    await v.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, PAY, nowTs);
    expect(await v.totalAssets()).to.equal(DEPOSIT - PAY);

    // ---- capacity is UNCHANGED even though half the collateral is gone ----
    await ctx.priceFeed.setAnswer(E18);
    const capacityAfterPayout = await readMaxCapacity(ctx);
    console.log("        allocator maxCapacity before payout:", capacityBeforePayout.toString());
    console.log("        allocator maxCapacity after  payout:", capacityAfterPayout.toString());
    expect(capacityAfterPayout).to.equal(capacityBeforePayout);

    // What the protocol's own policy would allow against the collateral that remains:
    const honestCapacity =
      ((DEPOSIT - PAY) * 10n ** 12n * BigInt(ctx.capacityConfig.effectiveLeverage)) /
      BigInt(ctx.capacityConfig.minCAR);
    expect(honestCapacity).to.equal(1_000_000n * E18);

    // ---- commit + settle the full stale capacity ----
    const leaves = [
      [0, [[ctx.marketIdA, COVER_A]]],
      [1, [[ctx.marketIdB, COVER_B]]],
    ];
    const { tree, root } = buildTree(leaves);
    const totalAllocated = COVER_A + COVER_B; // 2,000,000e18

    await a
      .connect(ctx.allocatorRole)
      .commitAllocation(orderPeriod, root, totalAllocated, capacityAfterPayout);

    await a
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0, [{ marketId: ctx.marketIdA, allocatedCover: COVER_A }], getProof(tree, 0));
    await a
      .connect(ctx.allocatorRole)
      .settleCoverOrder(1, [{ marketId: ctx.marketIdB, allocatedCover: COVER_B }], getProof(tree, 1));

    const commit = await a.getAllocationCommitment(orderPeriod);
    const settledCover = commit.totalSettledCover ?? commit[4];
    console.log("        cover actually SOLD (canonical 18d):", settledCover.toString());
    console.log("        policy max for remaining collateral:", honestCapacity.toString());

    expect(settledCover).to.equal(2_000_000n * E18);
    expect(settledCover).to.equal(honestCapacity * 2n); // exactly 2x over-issued

    // Real backing ratio: collateral / cover sold.  Policy floor is minCAR/leverage = 0.5.
    const realCollateral18 = (await v.totalAssets()) * 10n ** 12n;
    const realBackingBps = (realCollateral18 * BPS) / settledCover;
    const policyFloorBps = (BigInt(ctx.capacityConfig.minCAR) * BPS) / BigInt(ctx.capacityConfig.effectiveLeverage);
    console.log("        real backing (bps of cover)        :", realBackingBps.toString());
    console.log("        policy floor (bps of cover)        :", policyFloorBps.toString());
    expect(realBackingBps).to.be.lessThan(policyFloorBps);
    expect(realBackingBps).to.equal(2500n); // 25% backing vs a 50% policy floor
  });

  // ------------------------------------------------------------------------------------------
  // Part 2b — the extreme. A payout that consumes the whole vault still leaves the period's
  // capacity at its pre-payout value: cover is sellable against ZERO vault collateral.
  // ------------------------------------------------------------------------------------------
  it("part 2b: a 100% payout leaves full capacity — cover sellable with zero vault collateral", async () => {
    const ctx = await loadFixture(deployRealStack);
    const v = ctx.firelight_vault;
    const a = ctx.allocator;

    const DEPOSIT = 1_000_000n * E6;
    await ctx.utils.mintAndApprove(DEPOSIT, ctx.users[0]);
    await v.connect(ctx.users[0]).deposit(DEPOSIT, ctx.users[0].address);

    await nextPeriod(v);
    const orderPeriod = (await v.currentPeriod()) + 1n;
    const COVER_A = 1_200_000n * E18;
    await a
      .connect(ctx.curator)
      .createCoverOrder(
        ctx.buyer1.address,
        ctx.buyer1.address,
        "beneficiary-1",
        await ctx.usdc.getAddress(),
        [{ marketId: ctx.marketIdA, coverRateAnnual: 500, coverAmount: COVER_A }],
        0
      );
    await ctx.usdc.mint(ctx.buyer1.address, 10n ** 12n);
    await ctx.usdc.connect(ctx.buyer1).approve(await a.getAddress(), 10n ** 12n);

    await nextPeriod(v);
    await ctx.priceFeed.setAnswer(E18);

    // drain the vault completely
    await v.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, DEPOSIT, BigInt(await time.latest()));
    expect(await v.totalAssets()).to.equal(0n);

    await ctx.priceFeed.setAnswer(E18);
    const capacity = await readMaxCapacity(ctx);
    console.log("        vault totalAssets() = 0, allocator maxCapacity =", capacity.toString());
    expect(capacity).to.equal(2_000_000n * E18); // unchanged

    const { tree, root } = buildTree([[0, [[ctx.marketIdA, COVER_A]]]]);
    await a.connect(ctx.allocatorRole).commitAllocation(orderPeriod, root, COVER_A, capacity);
    await a
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0, [{ marketId: ctx.marketIdA, allocatedCover: COVER_A }], getProof(tree, 0));

    const commit = await a.getAllocationCommitment(orderPeriod);
    expect(commit.totalSettledCover ?? commit[4]).to.equal(COVER_A);
    expect(await v.totalAssets()).to.equal(0n); // $1.2M of cover sold against $0 of staked collateral
  });

  // ------------------------------------------------------------------------------------------
  // Part 3 — control. The same payout executed AT the period-start second IS picked up, which
  // pins the root cause on the checkpoint key rather than on anything else in the flow.
  // ------------------------------------------------------------------------------------------
  it("part 3 (control): a payout landing exactly on the period-start second does lower the snapshot", async () => {
    const ctx = await loadFixture(deployRealStack);
    const v = ctx.firelight_vault;

    const DEPOSIT = 1_000_000n * E6;
    await ctx.utils.mintAndApprove(DEPOSIT, ctx.users[0]);
    await v.connect(ctx.users[0]).deposit(DEPOSIT, ctx.users[0].address);

    const end = await v.currentPeriodEnd();
    // mine the payout in the exact block whose timestamp == the new period's start
    await time.setNextBlockTimestamp(Number(end));
    const PAY = 400_000n * E6;
    await v.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, PAY, BigInt(end));

    const periodStart = await v.currentPeriodStart();
    expect(BigInt(periodStart)).to.equal(BigInt(end));
    expect(await v.totalAssetsAt(periodStart)).to.equal(DEPOSIT - PAY); // reduction IS visible
  });
});
```

Output (one run; runtimes are machine-dependent, every other value is deterministic):

```
  PoC 030 — capacity uses a period-start snapshot a mid-period payout cannot lower
        totalAssets() after payout      : 500000000000
        totalAssetsAt(periodStart)      : 1000000000000
    ✔ part 1: payout lowers totalAssets() but NOT totalAssetsAt(currentPeriodStart()) (602ms)
        allocator maxCapacity before payout: 2000000000000000000000000
        allocator maxCapacity after  payout: 2000000000000000000000000
        cover actually SOLD (canonical 18d): 2000000000000000000000000
        policy max for remaining collateral: 1000000000000000000000000
        real backing (bps of cover)        : 2500
        policy floor (bps of cover)        : 5000
    ✔ part 2: cover is committed and settled at 2x the collateral that actually remains (73ms)
        vault totalAssets() = 0, allocator maxCapacity = 2000000000000000000000000
    ✔ part 2b: a 100% payout leaves full capacity — cover sellable with zero vault collateral (42ms)
    ✔ part 3 (control): a payout landing exactly on the period-start second does lower the snapshot


  4 passing (730ms)
```

Part 1 isolates the mechanism (`totalAssets()` halves, `totalAssetsAt(periodStart)` does not), part 2 shows 2,000,000e18 of cover settled at 2500 bps of real backing against a 5000 bps policy floor, part 2b shows full capacity surviving a payout that empties the vault entirely, and part 3 is the control: a payout mined on the period-start second does lower the snapshot, which pins the checkpoint key as the root cause.
