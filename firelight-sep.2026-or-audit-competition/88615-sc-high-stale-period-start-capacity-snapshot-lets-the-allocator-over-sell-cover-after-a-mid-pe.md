# 88615 sc high stale period start capacity snapshot lets the allocator over sell cover after a mid period payout causing protocol insolvency

**Submitted on Aug 14th 2026 at 06:44:34 UTC by @hunter67 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88615
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator._computeAvailableCapacity` values the vault's staked assets at `vault.totalAssetsAt(vault.currentPeriodStart())`, the asset total checkpointed at the start of the current period, instead of the live `vault.totalAssets()`. The documented off-chain matcher computes capacity from the live `vault.totalAssets()`, and the on-chain error and parameter docs describe the commit ceiling as the "live capacity". So the on-chain check in `commitAllocation` is the live-capacity enforcement bound that is supposed to stop a commitment from exceeding what current collateral can back.

Payouts reduce collateral mid-period. `FirelightVault.payout` pushes the lowered asset total at the block timestamp of the payout, which is after the period start, so `totalAssetsAt(currentPeriodStart())` keeps returning the pre-payout value for the rest of the period. When a legitimate incident payout is included before that period's `commitAllocation`, the enforcement bound is computed from the stale, higher snapshot and accepts a commitment the live value would reject. Cover is then committed and settled against collateral that no longer exists, and a later covered claim on that cover is underpaid.

Every actor in the reproduction behaves honestly and within its role. No attacker needs to hold or abuse a privileged role.

## Target

* Program: Immunefi "Audit Comp | Firelight".
* Repo (frozen contest source): `immunefi-team/audit-comp-firelight` at `42f9ea5` (tree `6c38527`).
* In-scope contracts involved: `contracts/core/CoverOrderAllocator.sol`, `contracts/core/FirelightVault.sol`, `contracts/core/IncidentManager.sol`, `contracts/core/interfaces/ICoverOrderAllocator.sol`.

## Severity

Critical, protocol insolvency. The protocol underwrites and settles more cover than its collateral can back at the configured `minCAR` / `effectiveLeverage`, so covered claims cannot be fully paid and cover buyers take a direct, permanent loss. The insolvency impact is in-scope Critical per the program rules.

The severity is isolated from the protocol's intended leverage by a 1x-leverage control (see Impact, and the supplied `sccap-case4d-1x-leverage-control.test.js`): with `effectiveLeverage == minCAR`, a correct live-capacity guard would pay the claim in full and lose nothing, while the buggy guard leaves 2000 of a valid claim permanently unpaid. That unpaid amount is caused only by this bug, not by fractional-reserve leverage.

## Root cause (exact lines)

1. `CoverOrderAllocator.sol:711`, inside `_computeAvailableCapacity` (`:696-725`):

   ```solidity
   uint256 totalAssetsCanonical = Decimals.convert(
       $.vault.totalAssetsAt($.vault.currentPeriodStart()),   // period-START snapshot
       $.vaultAssetDecimals,
       CANONICAL_DECIMALS,
       Math.Rounding.Floor
   );
   ```
2. `_computeAvailableCapacity` is the sole capacity ceiling consumed by the commit guard `_validateMatchingCapacity` (`:394-403`), reached from `commitAllocation` (`:348`) and `recommitAllocation` (`:378`). Within that computation the vault-backed component is read from the period-start snapshot above rather than live `totalAssets()` (the first-loss-buffer balance is added separately, so the defect is specifically the vault-backed term):

   ```solidity
   uint256 maxCapacity = _computeAvailableCapacity($, period);
   if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
   ```
3. The interface documents this bound as the live capacity. The off-chain matcher computes capacity from `vault.totalAssets()` (`ICoverOrderAllocator.sol:40-41`), and the error / parameter docs call the on-chain ceiling the "live capacity" (`ICoverOrderAllocator.sol:417-421` and `:528-529`). The implementation reads the period-start snapshot instead, so implementation and specification disagree.
4. The snapshot goes stale within the period because `FirelightVault.payout` pushes the reduced total at the current block timestamp, not at the period start (`FirelightVault.sol:952`):

   ```solidity
   _traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
   ```

   `totalAssetsAt(t)` is `upperLookupRecent(t)` (`FirelightVault.sol:456`) and `currentPeriodStart()` floors to the period boundary (`FirelightVault.sol:342-345`), so a payout at `now > periodStart` never moves the period-start lookup.
5. Settlement does not re-check live collateral. `_settleCoverOrder` (`CoverOrderAllocator.sol:457-540`) enforces the per-market cap against the stored `commit.totalAvailableCapacity` (`:507`) and the merkle proof; it never re-reads the vault, so the stale-inflated capacity flows straight into settled cover.

## Preconditions

* Honest roles only: `CURATOR_ROLE`, `ALLOCATOR_ROLE`, `ASSESSMENT_APPROVER_ROLE`, and `IncidentManager` holding the vault `PAYOUT_ROLE` / `INCIDENT_ROLE`. No role acts outside its documented privileges.
* A legitimate incident whose payout is included during the period but before that period's `commitAllocation`. The clean, fully-reachable form is a previous-period incident paid in the current period: the vault's payout window is the capture period or the one after it (`FirelightVault.sol:1130-1131`), so a period P-1 incident can be approved and paid during period P, before the P commit. A same-period-P payout before the first P commit is not reachable, because an incident of P must reference a settled cover of P, which requires the P commit to already exist, after which `commitAllocation(P)` reverts `PeriodAlreadyCommitted`.
* The off-chain matcher snapshotted capacity from the pre-payout collateral, which is normal: the payout can be included between the matcher's snapshot block and the commit-inclusion block.

## Timing (why an honest allocator submits the pre-payout capacity)

```
t0: the off-chain matcher reads live vault.totalAssets() = 10000 and constructs a 20000 matching commitment
t1: the allocator commit transaction is prepared and broadcast
t2: a valid previous-period incident approval is included first and drains collateral to 4000
t3: the allocator commit transaction is included afterward, still carrying the 20000 from t0
t4: the on-chain live-capacity bound should now reject the stale 20000, but it reads the period-start
    snapshot (still 10000) and accepts it
```

This ordering is exactly what the on-chain recompute and the per-period `divergenceToleranceBps` exist for. The interface states the on-chain recompute is "live" and is used to bound the matcher input, but the implementation reads the period-start checkpoint, so the bound does not do its job.

## Steps to reproduce

The supplied PoC `sccap-case4c-production-path.test.js` drives the whole chain through the real `FirelightVault`, `CoverOrderAllocator` and `IncidentManager` (no mock vault). Numbers below are from the run.

{% stepper %}
{% step %}

## Period P0

A user deposits 10,000 fXRP. A cover buyer's order for 6,000 USD of Morpho cover is created, targeting period P1.
{% endstep %}

{% step %}

## Period P1: settle old cover

`ALLOCATOR_ROLE` runs `commitAllocation(P1)` and `settleCoverOrder`, minting the 6,000 "old" CoverNFT. Live collateral at commit time is 10,000, so this commit is correct.
{% endstep %}

{% step %}

## Period P1: create incident and victim order

`CURATOR_ROLE` creates and assesses (but does not yet approve) an incident on the old CoverNFT. A second cover buyer's order for 12,000 USD of Morpho cover is created, targeting period P2.
{% endstep %}

{% step %}

## Advance to period P2

The off-chain matcher reads live collateral, still 10,000, and derives a 20,000 P2 capacity from it.
{% endstep %}

{% step %}

## Period P2: approve prior-period incident

`ASSESSMENT_APPROVER_ROLE` approves the period-P1 incident. `IncidentManager._executePayout` calls `vault.payout`, paying the 6,000 assessed loss and draining live collateral from 10,000 to 4,000.

`totalAssetsAt(currentPeriodStart(P2))` still returns 10,000.
{% endstep %}

{% step %}

## Period P2: commit stale capacity

`ALLOCATOR_ROLE` commits the matcher's pre-payout 20,000. The guard recomputes its ceiling from the stale 10,000 snapshot, gets 20,000, and accepts. The boundary is exact: committing `20,000 + 1` reverts `MatchingCapacityOverflow(20001e18, 20000e18)`, proving the accepted ceiling equals capacity(10,000), not capacity(4,000).
{% endstep %}

{% step %}

## Period P2: settle victim cover

`ALLOCATOR_ROLE` settles the 12,000 victim CoverNFT against the stale-inflated commitment. A correct live commit of 8,000 would cap Morpho cover at 60% \* 8,000 = 4,800, so 7,200 USD of this cover should never have been sellable.
{% endstep %}

{% step %}

## Period P2: pay victim claim

A covered incident on the victim CoverNFT is driven through the full `IncidentManager`.

`_executePayout` requests 12,000 fXRP from the vault; `vault.payout` pays only the remaining 4,000 and the incident is closed terminally with no debt-retry path. The cover buyer is short 8,000 fXRP.
{% endstep %}
{% endstepper %}

## Expected vs actual

* **Expected:** `commitAllocation(P2)` binds `matchingCapacity` to the live capacity, so after the drain it rejects any commitment above 8,000 USD, and no more than 4,800 USD of Morpho cover can be settled.
* **Actual:** it binds to the period-start snapshot, accepts 20,000 USD, and lets 12,000 USD of Morpho cover settle. The excess is not backed by live collateral, and a covered claim on it is underpaid.

## Impact

The bug-attributable harm is the cover the guard allows beyond the live-collateral limit:

* Committed capacity: 20,000 USD accepted vs 8,000 USD that live collateral supports, so 12,000 USD excess capacity.
* Morpho cover: 12,000 USD settled vs a live-correct per-market cap of 4,800 USD, so 7,200 USD of excess cover that a correct guard would have refused.

The same 7,200 falls out of the claim shortfall as a counterfactual:

```
Bug path:                12,000 cover - 4,000 live assets = 8,000 unpaid
Correct live-guard path:  4,800 cover - 4,000 live assets =   800 unpaid  (by-design 2x leverage)
Incremental unpaid liability caused by H-CAP-1 = 8,000 - 800 = 7,200
    = the excess cover:  12,000 - 4,800                    = 7,200
```

So the observed 8,000 fXRP shortfall is not all attributable to the bug; the by-design 2x leverage accounts for 800, and the remaining 7,200 is caused by the stale-capacity over-issuance.

The 1x-leverage control (`sccap-case4d-1x-leverage-control.test.js`) removes the leverage entirely. With `effectiveLeverage == minCAR`, a correct guard would allow at most 2,400 Morpho cover on 4,000 collateral, which pays a full claim 100% and loses nothing; the buggy guard allows 6,000, so a full claim pays only 4,000 and leaves 2,000 permanently unpaid. Under a correct guard the by-design shortfall is zero, so the entire 2,000 is caused by the bug. This shows the finding is genuine insolvency, not merely a higher expected leverage.

Scope note: these figures are the impact demonstrated in the PoC's harness config, not a claim that a specific amount has already been lost on a live deployment. The finding is a source-level mechanism that over-issues cover and leaves approved liability underfunded; on any deployment the magnitude scales with the drained collateral and the configured `effectiveLeverage / minCAR`.

## On the trusted-role responsibility note

`FirelightVault.payout`'s NatSpec states that keeping aggregate payouts within underwritten exposure is the responsibility of the curator and assessor roles. This finding does not contradict that and does not require either role to over-assess or act maliciously. The reproduction has honest roles: the matcher snapshots live collateral correctly, a valid prior-period claim is approved, and the allocator commits the value the matcher produced. The independent allocator invariant still explicitly requires the submitted matching capacity to be bounded by live collateral at commit time (`ICoverOrderAllocator.sol:40-41`, `:417-421`, `:528-529`); that invariant is what fails, because the implementation reads the period-start checkpoint instead of live assets.

## Development intent (commit I-8)

The on-chain bound and its "live capacity" wording were introduced in the same commit. Upstream commit `411e7ca3a6ba705434a170aa39ccd3db0ac54ce6` ("feat: \[I-8] add matchingCapacity to allocation commitments", 2026-07-21) added the `matchingCapacity` parameter and the `_validateMatchingCapacity` check to `commitAllocation`. The same commit documented that check as bounding the caller against the live capacity: the `MatchingCapacityOverflow` NatSpec it added reads "Thrown when the caller's matching capacity exceeds the live capacity", and the `commitAllocation` parameter doc it added says `matchingCapacity` "Must not exceed the live capacity widened by the period's divergence tolerance". But `_validateMatchingCapacity` reuses the pre-existing `_computeAvailableCapacity`, which reads `totalAssetsAt(currentPeriodStart())` rather than live `totalAssets()`; that capacity function is not modified by the commit. So the check that was added and documented as a live-capacity bound is wired to a period-start read. A claim that the period-start snapshot is intentional has to explain why the check introduced specifically to enforce live capacity does not read live collateral.

## Recommended fix

Value staked assets from live collateral, matching the documented behaviour. In `_computeAvailableCapacity` (`CoverOrderAllocator.sol:711`) read `$.vault.totalAssets()` instead of `$.vault.totalAssetsAt($.vault.currentPeriodStart())`. The off-chain matcher already uses `totalAssets()` (`ICoverOrderAllocator.sol:40-41`), so this aligns the on-chain bound with the value the matcher committed against, and the period's `divergenceToleranceBps` continues to absorb the small snapshot-to-inclusion drift it was designed for.

Defence in depth: re-check the committed capacity against live collateral at settlement time in `_settleCoverOrder`, so a payout included after a correct commit but before settlement cannot leave the sold cover under-collateralised either.

## References and how this differs

* The protocol's own divergence tolerance (`CoverOrderAllocator.sol:75-81`) is documented to absorb only the small drift between the off-chain snapshot and commit inclusion, and is capped at 10% (`MAX_DIVERGENCE_TOLERANCE_BPS`). A payout can erode collateral far beyond that band, so this is not the tolerated drift the design accounts for.
* The listed known issues for the contest (permissioned MVP, off-chain matching, external FLB wallet, USD peg) do not cover the period-start-vs-live capacity read. This is a concrete implementation-violates-interface defect in an in-scope contract, not the off-chain-matching known issue: the on-chain bound that is supposed to make over-commit impossible is the thing computing on stale data.

## Limitations

* Source-lane only: the PoC runs against the frozen contest source in a local Hardhat EVM; I did not test against a live Flare deployment. Every claim here is a mechanism claim reproducible in the harness.
* The trigger needs the payout to be included before the period's commit (the previous-period-incident path in the PoC). I did not measure how frequently that interleaving occurs in production operations; it is reachable under normal operation, and the reproduction shows the full reachable path.
* Impact is quantified for the harness config. The general magnitude scales with the drained collateral times `effectiveLeverage / minCAR`; the 1x control isolates the bug's own contribution from the intended leverage.

## Proof of Concept

Production source under review: upstream `immunefi-team/audit-comp-firelight` at `42f9ea5`. The PoC is a Hardhat test supplied with this report and applied on top of that frozen source; it adds no contract and changes no production code. It uses only contracts and helpers already present in the frozen source (`test/setup/fixtures.js`, `contracts/test/MockERC20.sol`, `contracts/test/MockAggregatorV3.sol`, the in-scope `CoverNFT`, `IncidentManager`, `CoverOrderAllocator`, `FirelightVault`) plus the `@openzeppelin/merkle-tree` dev dependency.

{% stepper %}
{% step %}

## Clone the source

Clone `immunefi-team/audit-comp-firelight` and check out `42f9ea5`.
{% endstep %}

{% step %}

## Install dependencies

```bash
npm ci
```

{% endstep %}

{% step %}

## Save the PoC

Save the file below as `test/audit/sccap-case4c-production-path.test.js`.
{% endstep %}

{% step %}

## Run the test

```bash
npx hardhat test test/audit/sccap-case4c-production-path.test.js
```

{% endstep %}
{% endstepper %}

Environment used: Windows 11, Node v24.16.0, Hardhat 2.22.19, solc 0.8.28, in-process Hardhat EVM (london), no fork. Two supplementary tests are supplied the same way: `sccap-case1-capacity-staleness.test.js` (mechanism plus a no-payout control) and `sccap-case4d-1x-leverage-control.test.js` (the 1x-leverage severity control).

Full primary PoC file (`sccap-case4c-production-path.test.js`):

```javascript
// SC-CAP Case 4c - Family 1 (F-CAP), H-CAP-1 - FULL production path (previous-period drain)
// ---------------------------------------------------------------------------
// The honest production path uses a PREVIOUS-PERIOD incident paid during the current
// period (the vault's payout window is capturePeriod OR capturePeriod+1):
//
//   P1: commit(1) + settle "old" cover; create+assess (not approve) an incident on it;
//       create the "victim" cover order targeting P2.
//   -> advance to P2. Live collateral = A0 = 10000 (matcher snapshot).
//   P2: APPROVE the P1 incident through the FULL IncidentManager -> vault.payout drains
//       A0->A1=4000 (a legitimate prior-period claim, executed before the P2 commit).
//   P2: commit(2) the pre-payout 20000 capacity. totalAssetsAt(periodStart(P2)) is still
//       the stale 10000, so the buggy guard ACCEPTS 20000; a correct live guard (A1->8000)
//       would REJECT. No PeriodAlreadyCommitted: the drain incident referenced a period-1
//       cover, the victim commit is period 2.
//   P2: settle the victim CoverNFT; then a FULL IncidentManager incident on it is approved
//       and the covered claim is underpaid (capped at live collateral).
// Environment: source 42f9ea5 / Hardhat EVM (london). No live mutation.
// ---------------------------------------------------------------------------

const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { upgrades, ethers } = require('hardhat')
const { expect } = require('chai')
const { deployVault, computeMarketId } = require('../setup/fixtures.js')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const ONE_E18 = 10n ** 18n
const E12 = 10n ** 12n
const BPS = 10_000n
const ZERO = ethers.ZeroAddress
const NEW = 0
const Status = { PENDING: 0, MATCHED: 1, PARTIAL: 2, CANCELLED: 3 }
const MIN_CAR = 12000n
const EFF_LEV = 24000n
const MORPHO_BPS = 6000n

const capacityOf = (assets6) => (((assets6 * E12) * EFF_LEV) / MIN_CAR) // FLB=0, $1, tol=0
const buildLeaf = (orderId, marketId, cover) => {
  const tree = StandardMerkleTree.of([[orderId, [[marketId, cover]]]], ['uint256', '(bytes32,uint256)[]'])
  return { root: tree.root, proof: tree.getProof(0) }
}
const refOf = (s) => ethers.keccak256(ethers.toUtf8Bytes(s))

async function wire(vaultCtx) {
  const s = await ethers.getSigners()
  const [admin, adminRole, curator, allocatorRole, configAdmin, premiumCollector, flbWallet] = s.slice(9, 16)
  const buyer = s[16], imCurator = s[17], imApprover = s[18], imPayoutReceiver = s[19]

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const flb = await MockERC20.deploy('FLB', 'FLB', 6)
  const usdc = await MockERC20.deploy('MockUSDC', 'mUSDC', 6)
  const usdt = await MockERC20.deploy('MockUSDT', 'mUSDT', 6)
  const MockAggregatorV3 = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregatorV3.deploy(18, ONE_E18)
  const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory, ['Firelight Cover', 'FLCOVER', '', admin.address, s[0].address, ZERO, ZERO],
    { kind: 'transparent' })
  const MARKET_A = ethers.encodeBytes32String('market-A')
  const MARKET_B = ethers.encodeBytes32String('market-B')
  const initParams = {
    vault: await vaultCtx.firelight_vault.getAddress(), premiumCollector: premiumCollector.address,
    coverNFT: await coverNFT.getAddress(), priceFeedAdapter: await priceFeed.getAddress(),
    maxPriceAge: 3650 * 24 * 3600, premiumTokens: [await usdc.getAddress(), await usdt.getAddress()],
    admin: admin.address, adminRole: adminRole.address, curatorRole: curator.address,
    allocatorRole: allocatorRole.address, configAdminRole: configAdmin.address,
    initialProtocolConcentrations: [
      { protocol: 'Morpho', chainId: 1, maxProtocolConcentrationBps: 6000 },
      { protocol: 'Aave', chainId: 10, maxProtocolConcentrationBps: 4000 }],
    newMarkets: [{ chainId: 1, protocol: 'Morpho', market: MARKET_A }, { chainId: 10, protocol: 'Aave', market: MARKET_B }],
    capacityConfig: {
      minCAR: Number(MIN_CAR), firstLossBufferToken: await flb.getAddress(), firstLossBuffer: flbWallet.address,
      effectiveLeverage: Number(EFF_LEV), minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0 }
  }
  const allocator = await upgrades.deployProxy(await ethers.getContractFactory('CoverOrderAllocator'), [initParams],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  const im = await upgrades.deployProxy(await ethers.getContractFactory('IncidentManager'),
    [admin.address, imCurator.address, imApprover.address, admin.address, admin.address, ZERO, ZERO, ZERO,
      imPayoutReceiver.address, await allocator.getAddress(), await priceFeed.getAddress(), 3650 * 24 * 3600],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })

  const vault = vaultCtx.firelight_vault
  await vault.connect(vaultCtx.deployer).grantRole(await vault.PAYOUT_ROLE(), await im.getAddress())
  await vault.connect(vaultCtx.deployer).grantRole(await vault.INCIDENT_ROLE(), await im.getAddress())
  await vault.connect(vaultCtx.payout_allowlister).addToPayoutAllowlist(imPayoutReceiver.address)

  return { allocator, coverNFT, priceFeed, usdc, curator, allocatorRole, im, imCurator, imApprover, imPayoutReceiver, buyer,
    marketIdA: computeMarketId(1, 'Morpho', MARKET_A) }
}

describe('SC-CAP Case 4c / H-CAP-1 - full production path (prev-period drain, payout-before-commit)', function () {
  const DEPOSIT = ethers.parseUnits('10000', 6) // A0 at period 2 start
  const OLD_COVER = ethers.parseUnits('6000', 18) // legit cover, period 1
  const OLD_LOSS = ethers.parseUnits('6000', 18) // its incident drains 6000 fXRP
  const VICTIM_COVER = ethers.parseUnits('12000', 18) // over-sold cover, period 2 (Morpho 60% of 20000)
  const VICTIM_LOSS = ethers.parseUnits('12000', 18)
  const A1 = ethers.parseUnits('4000', 6)

  it('prev-period incident drains via full IncidentManager BEFORE the P2 commit; stale guard accepts 20k; victim claim underpaid', async () => {
    const vaultCtx = await deployVault()
    const w = await wire(vaultCtx)
    const { allocator, coverNFT, priceFeed, usdc, curator, allocatorRole, im, imCurator, imApprover, imPayoutReceiver, buyer, marketIdA } = w
    const vault = vaultCtx.firelight_vault
    const asset = vaultCtx.token_contract
    const DUR = vaultCtx.config.period_configuration_duration

    // --- Period 0: fund; create the OLD cover order (targets period 1) ---
    await vaultCtx.utils.mintAndApprove(DEPOSIT, vaultCtx.users[0])
    await vault.connect(vaultCtx.users[0]).deposit(DEPOSIT, vaultCtx.users[0].address)
    await usdc.mint(buyer.address, ethers.parseUnits('100000', 6))
    await usdc.connect(buyer).approve(await allocator.getAddress(), ethers.parseUnits('100000', 6))
    await allocator.connect(curator).createCoverOrder(buyer.address, buyer.address, buyer.address,
      await usdc.getAddress(), [{ marketId: marketIdA, coverRateAnnual: 500, coverAmount: OLD_COVER }], NEW) // orderId 0

    // --- Period 1 (P-1): commit + settle the OLD cover; create+assess (not approve) its incident;
    //     create the VICTIM order targeting period 2 ---
    await time.increase(DUR + 100)
    const p1 = await vault.currentPeriod()
    await priceFeed.setAnswer(ONE_E18)
    const old = buildLeaf(0n, marketIdA, OLD_COVER)
    await allocator.connect(allocatorRole).commitAllocation(p1, old.root, OLD_COVER, capacityOf(DEPOSIT)) // live=10k -> cap 20k (correct here)
    await allocator.connect(allocatorRole).settleCoverOrder(0, [{ marketId: marketIdA, allocatedCover: OLD_COVER }], old.proof)
    expect(await coverNFT.ownerOf(0)).to.equal(buyer.address)

    const drainTs = await time.latest() // captured in period 1
    await im.connect(imCurator).createIncident(drainTs, 'prior incident P1', refOf('drain')) // incidentId 1
    await im.connect(imCurator).confirmIncident(1, 'ipfs://d')
    await im.connect(imCurator).addAssessmentLosses(1, [{ coverTokenId: 0, marketId: marketIdA, amount: OLD_LOSS }])
    await im.connect(imCurator).submitCurrentAssessment(1) // NOT approved yet

    await allocator.connect(curator).createCoverOrder(buyer.address, buyer.address, buyer.address,
      await usdc.getAddress(), [{ marketId: marketIdA, coverRateAnnual: 500, coverAmount: VICTIM_COVER }], NEW) // orderId 1

    // --- Period 2 (P): the off-chain matcher snapshots LIVE collateral and builds its commitment ---
    await time.increase(DUR + 100)
    const p2 = await vault.currentPeriod()
    // The matcher reads live totalAssets() (per the documented off-chain flow) and derives capacity from it.
    // Capturing it here proves the committed value is the PRE-payout snapshot, not a tester-invented constant.
    const matcherSnapshotAssets = await vault.totalAssets()
    expect(matcherSnapshotAssets).to.equal(DEPOSIT) // 10000 live at snapshot time, nothing drained yet
    const capBuggy = capacityOf(matcherSnapshotAssets) // 20000e18, computed from the pre-payout live read
    await priceFeed.setAnswer(ONE_E18)

    // (1) A valid P1 incident approval is included FIRST and drains collateral 10000->4000, before the P2 commit tx.
    await im.connect(imApprover).approveCurrentAssessment(1)
    expect(await vault.totalAssets()).to.equal(A1) // live now 4000
    expect(await vault.totalAssetsAt(await vault.currentPeriodStart())).to.equal(DEPOSIT) // periodStart(P2) still 10000

    // (2) The allocator commits the matcher's pre-payout 20000. The period-start guard reads the stale 10000 and
    //     ACCEPTS; a live guard reading 4000 (capacity 8000) would REJECT.
    const capLive = capacityOf(A1) // 8000e18, what live collateral now supports
    expect(capBuggy).to.be.gt(capLive)
    const victim = buildLeaf(1n, marketIdA, VICTIM_COVER)
    await expect(
      allocator.connect(allocatorRole).commitAllocation(p2, ethers.id('probe'), 0n, capBuggy + 1n)
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow').withArgs(capBuggy + 1n, capBuggy)
    await allocator.connect(allocatorRole).commitAllocation(p2, victim.root, VICTIM_COVER, capBuggy)
    expect((await allocator.getAllocationCommitment(p2)).totalAvailableCapacity).to.equal(capBuggy)

    // (3) Settle the victim CoverNFT (12k Morpho) against the stale-inflated commit.
    await allocator.connect(allocatorRole).settleCoverOrder(1, [{ marketId: marketIdA, allocatedCover: VICTIM_COVER }], victim.proof)
    expect((await allocator.getCoverOrder(1)).status).to.equal(Status.MATCHED)
    expect(await coverNFT.ownerOf(1)).to.equal(buyer.address)
    const liveMorphoCap = (capLive * MORPHO_BPS) / BPS // 4800e18
    const excessMorphoCover = VICTIM_COVER - liveMorphoCap // 7200e18
    expect(excessMorphoCover).to.be.gt(0n)

    // (4) FULL IncidentManager claim on the victim CoverNFT -> underpaid.
    await priceFeed.setAnswer(ONE_E18)
    const claimTs = await time.latest()
    await im.connect(imCurator).createIncident(claimTs, 'victim claim P2', refOf('claim')) // incidentId 2
    await im.connect(imCurator).confirmIncident(2, 'ipfs://c')
    await im.connect(imCurator).addAssessmentLosses(2, [{ coverTokenId: 1, marketId: marketIdA, amount: VICTIM_LOSS }])
    await im.connect(imCurator).submitCurrentAssessment(2)

    const before = await asset.balanceOf(imPayoutReceiver.address)
    await im.connect(imApprover).approveCurrentAssessment(2)
    const paidClaim = (await asset.balanceOf(imPayoutReceiver.address)) - before

    const vaultRequested = VICTIM_LOSS / E12 // 12000e6
    const shortfall = vaultRequested - paidClaim
    expect(paidClaim).to.equal(A1) // 4000e6 (all that remained)
    expect(shortfall).to.equal(ethers.parseUnits('8000', 6))
    expect(await vault.totalAssets()).to.equal(0n)

    console.log('\n  === SC-CAP Case 4c (H-CAP-1) - full production path (prev-period drain) ===')
    console.log(`  P1: old cover ${ethers.formatUnits(OLD_COVER,18)} settled; its incident drains in P2 via IncidentManager`)
    console.log(`  P2 start live A0                : ${ethers.formatUnits(DEPOSIT,6)} fXRP`)
    console.log(`  after prev-period payout (A1)   : ${ethers.formatUnits(A1,6)} fXRP  (periodStart snapshot still 10000)`)
    console.log(`  buggy guard committed capacity  : ${ethers.formatUnits(capBuggy,18)} USD  (live-correct ${ethers.formatUnits(capLive,18)} would reject)`)
    console.log(`  victim cover sold (Morpho)      : ${ethers.formatUnits(VICTIM_COVER,18)} USD  (live-correct cap ${ethers.formatUnits(liveMorphoCap,18)})`)
    console.log(`  bug-attributable EXCESS cover   : ${ethers.formatUnits(excessMorphoCover,18)} USD  (excess capacity ${ethers.formatUnits(capBuggy-capLive,18)})`)
    console.log(`  victim claim vault-requested    : ${ethers.formatUnits(vaultRequested,6)} fXRP`)
    console.log(`  victim claim paid               : ${ethers.formatUnits(paidClaim,6)} fXRP`)
    console.log(`  observed shortfall (incl. leverage): ${ethers.formatUnits(shortfall,6)} fXRP\n`)
  })
})
```

Raw output of the primary PoC:

```
  SC-CAP Case 4c / H-CAP-1 - full production path (prev-period drain, payout-before-commit)

  === SC-CAP Case 4c (H-CAP-1) - full production path (prev-period drain) ===
  P1: old cover 6000.0 settled; its incident drains in P2 via IncidentManager
  P2 start live A0                : 10000.0 fXRP
  after prev-period payout (A1)   : 4000.0 fXRP  (periodStart snapshot still 10000)
  buggy guard committed capacity  : 20000.0 USD  (live-correct 8000.0 would reject)
  victim cover sold (Morpho)      : 12000.0 USD  (live-correct cap 4800.0)
  bug-attributable EXCESS cover   : 7200.0 USD  (excess capacity 12000.0)
  victim claim vault-requested    : 12000.0 fXRP
  victim claim paid               : 4000.0 fXRP
  observed shortfall (incl. leverage): 8000.0 fXRP

    ok prev-period incident drains via full IncidentManager BEFORE the P2 commit; stale guard accepts 20k; victim claim underpaid

  1 passing
```

Raw output of the 1x-leverage control (`sccap-case4d-1x-leverage-control.test.js`):

```
  === SC-CAP Case 4d (H-CAP-1) - 1x-leverage severity control ===
  leverage                         : 1x (minCAR==effectiveLeverage)
  stale capacity / live capacity   : 10000.0 / 4000.0 USD
  Morpho cover: buggy / live-correct: 6000.0 / 2400.0 USD
  claim requested / paid           : 6000.0 / 4000.0 fXRP
  unpaid by-design (correct guard) : 0.0 fXRP
  unpaid caused SOLELY by the bug  : 2000.0 fXRP

    ok at 1x leverage a correct guard would pay the claim in full; the stale guard causes 2000 unpaid solely due to the bug

  1 passing
```
