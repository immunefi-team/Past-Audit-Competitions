# 88688 sc high coverorderallocator validates capacity against a stale period start snapshot letting cover settle after its collateral backing is paid out and causing protocol insolvency

**Submitted on Aug 14th 2026 at 14:30:34 UTC by @Jornason for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88688
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/ICoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator` validates the capacity behind a cover allocation commitment against `vault.totalAssetsAt(vault.currentPeriodStart())` — a period-start checkpoint — while the contract's own interface documentation specifies that the matching capacity must be computed from live `vault.totalAssets()`. A valid prior-period incident payout that executes during the target period reduces live vault collateral but leaves the period-start checkpoint unchanged. Because `settleCoverOrder` loads the stored commitment and never revalidates live collateral, cover for the target period remains settleable after the assets backing it have already left the vault. A later valid incident against that newly settled cover is then closed by `IncidentManager` with an underpaid payout, leaving the protocol with settled cover liabilities that exceed its remaining assets: protocol insolvency.

## Vulnerability Details

### Affected Asset

* Program: Audit Comp | Firelight
* Asset: `contracts/core/CoverOrderAllocator.sol` (interacting with `contracts/core/FirelightVault.sol` and `contracts/core/IncidentManager.sol`, both in scope)
* Repo: <https://github.com/immunefi-team/audit-comp-firelight>
* Branch/commit: `v1_audit_ready` @ `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

### Root Cause

The documented off-chain matching flow in `ICoverOrderAllocator.sol` instructs the matcher to compute capacity from the vault's live asset balance:

```solidity
// ICoverOrderAllocator.sol (step 4 of the documented matching flow):
//   Compute capacity: (config.firstLossBufferToken.balanceOf(config.firstLossBuffer)
//      + vault.totalAssets() * assetPriceUSD / 10**priceFeedDecimals) * config.effectiveLeverage / config.minCAR
```

The on-chain validation of `commitAllocation` / `recommitAllocation`, however, derives capacity from the period-start checkpoint instead of the live balance (`CoverOrderAllocator.sol`, `_computeAvailableCapacity`):

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),   // period-start checkpoint, not live totalAssets()
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

Settlement then trusts the stored commitment:

* `settleCoverOrder` loads `$.allocationCommitments[order.period]` and checks the Merkle proof, the per-market concentration caps (computed from `commit.totalAvailableCapacity`), and `commit.totalSettledCover > commit.totalDeclaredAllocated`. At no point does it compare the remaining cover to be settled against the vault's live collateral.
* `FirelightVault.payout()` pays out of current live active assets and checkpoints the lower post-payout value (`_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive)`), but nothing propagates this reduction to the allocator's current-period commitment.

So the sequence "commit at healthy snapshot → valid payout drains live assets → settle against stale snapshot" settles cover whose collateral backing no longer exists. `IncidentManager.approveCurrentAssessment` then pays a later valid claim via `FirelightVault.payout`, which caps `paidAmount` by live assets, and closes the incident regardless of the shortfall:

```solidity
uint256 vaultPaidAmount = _executePayout(...);
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;   // closed even when vaultPaidAmount < totalAssessmentLoss
```

### Exploit Flow

{% stepper %}
{% step %}

## LP deposits collateral and earlier cover settles

LP deposits 1000 vault-asset units. An earlier cover order is created and settled for period P. All values below are the PoC scale; the mechanism scales linearly with vault TVL.
{% endstep %}

{% step %}

## Create a target-period cover order

A new cover order for 1000 cover units targeting period P+1 is created before P+1 starts.
{% endstep %}

{% step %}

## Commit allocation at the healthy snapshot

At the start of P+1, the allocator commits the allocation root with `matchingCapacity = 1000`. On-chain validation passes because `totalAssetsAt(P+1 start) == 1000`.
{% endstep %}

{% step %}

## Execute a valid prior-period payout

A valid incident from period P is approved during P+1, as the payout window explicitly allows, and pays 900 out of the vault. Live assets drop to 100, while `totalAssetsAt(P+1 start)` remains 1000.
{% endstep %}

{% step %}

## Settle underbacked cover

The committed P+1 cover still settles for the full 1000 because settlement performs no live collateral revalidation.
{% endstep %}

{% step %}

## Close a later incident with an underpaid claim

A later valid incident against that newly settled cover is assessed for exactly its settled cover amount, 1000. The vault pays only the remaining 100 and the incident is terminally closed. 900 of a valid covered loss is unpaid.
{% endstep %}
{% endstepper %}

### Triggerability

No role acts maliciously, incorrectly, or outside its documented permissions. Every step is a routine, intended protocol action: the curator creates cover orders on agreed terms, the allocator commits and settles a matching result that passes every on-chain guard, and the assessor/approver approves only valid, correctly sized losses. The vulnerable state transition is produced entirely by the contracts: the documented capacity constraint (live `vault.totalAssets()`, per the interface specification) is bypassed because the enforcement point uses a stale period-start snapshot and settlement never revalidates. This is a protocol-constraint bypass, not a trusted-party misbehavior report: even a perfectly operated underwriting desk settles underbacked cover whenever an ordinary valid payout lands between commitment and settlement.

### Why Existing Checks Do Not Prevent It

* `MatchingCapacityOverflow` at commit time does not prevent this because it compares `matchingCapacity` against the period-start checkpoint, which is unaffected by payouts executed after the period starts.
* The settlement-time checks (`InvalidProof`, `ProtocolConcentrationOverflow`, `TotalSettledOverflow`) do not prevent this because all of them operate on the stored commitment values, never on live `vault.totalAssets()`.
* `FirelightVault.payout()` bounding each payout by live assets does not restore solvency; it is precisely what produces the shortfall — it silently pays less than the assessed covered loss and the incident is closed anyway.
* The first-loss-buffer waterfall does not cover the shortfall in general; the PoC demonstrates the failure with the buffer leg at zero, so the underpayment is attributable solely to the vault-side stale capacity accounting.

## Impact Details

Selected impact:

```
Protocol insolvency
```

Impact:

* Affected funds/assets: the vault's staked collateral (FXRP on Flare) and the cover buyers' valid claims.
* Per occurrence: up to the full settled cover of the target period minus the live collateral remaining after the earlier payout. In the executable PoC, a valid 1000-unit covered loss is paid 100 and closed with 900 unpaid — 90% of the valid claim is lost.
* Cumulative exposure: repeatable every period; every period in which a prior-period payout executes before that period's cover finishes settling can mint underbacked cover. The vault this code upgrades held approximately 60.16M FXRP of predeposit capital on Flare mainnet at the time of the previous audit (July 2026).
* Attacker cost: none — no attacker action is required; ordinary valid protocol operation triggers the insolvency.
* Recovery status: none on-chain. The underpaid incident is terminally `CLOSED`; there is no function to reopen it, top it up, or compensate the cover buyer. The protocol's settled-cover liabilities exceed its remaining assets with no normal recovery path.

Why this is not merely theoretical:

* The full sequence runs against the in-scope contracts at the audit commit and is demonstrated by the executable PoC attached in the PoC field: three tests covering the attack cell, a solvency control where sufficient remaining collateral pays the same claim in full, and a boundary control showing the same commit is correctly rejected when the payout happens before the period starts.
* The trigger condition — a valid prior-period incident paying out during the following period — is the normal, documented operation of the payout window (`FirelightVault.payout`: "Payout can execute during the incident period or the following period").
* All three components are explicitly listed as in-scope assets, and the program's stated priority areas include commitment/settlement correctness, capacity/solvency constraints, and interaction between multiple incidents in the same period.

Distinction from the program's known issues:

* Not an FLB depeg report: the PoC uses zero FLB balance and a constant 1 USD price; no depeg is involved.
* Not an unescrowed-FLB funding report: the underpayment occurs in the vault-asset leg after a valid vault payout; no FLB funding or allowance failure is used.
* Not an aggregate-claims-over-cover report: each incident is assessed for no more than its own settled cover, and each cover is claimed at most once. The failure is stale collateral accounting at settlement, not duplicate or excessive claims.
* Not a collateral-price-drawdown report: live assets decrease because of a valid payout, not because of any USD price movement; the price feed is constant in the PoC.

## Recommendation

Revalidate settlement against live collateral, or invalidate stale commitments, so that settled cover never exceeds live backing:

{% stepper %}
{% step %}

## Revalidate live capacity at settlement

In `settleCoverOrder` / `batchSettleCoverOrder`, check that `commit.totalSettledCover + allocatedCover` remains within a capacity recomputed from live `vault.totalAssets()` and adjusted by the same leverage/minCAR formula, reverting otherwise.
{% endstep %}

{% step %}

## Invalidate or recommit allocations after payouts

When `FirelightVault.payout()` checkpoints lower live assets during a period, invalidate or force a recommit of that period's allocation commitment before further settlement.
{% endstep %}

{% step %}

## Add regression tests

Add regression tests for both directions of the boundary: a post-period-start payout must reduce settleable capacity, while a pre-period-start payout must keep the existing commit-validation behavior.
{% endstep %}
{% endstepper %}

## References

* Official scope: <https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/>
* Affected source (audit commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`):
  * `contracts/core/interfaces/ICoverOrderAllocator.sol` — documented matcher capacity formula using live `vault.totalAssets()`
  * `contracts/core/CoverOrderAllocator.sol` — `_computeAvailableCapacity` using `totalAssetsAt(currentPeriodStart())`; `settleCoverOrder` trusting stored commitment values
  * `contracts/core/FirelightVault.sol` — `payout` capped by live assets; post-payout checkpoint
  * `contracts/core/IncidentManager.sol` — `approveCurrentAssessment` closing incidents after partial payout
* PoC package and raw runtime output: attached in the Proof of Concept field

## Link to Proof of Concept

<https://gist.github.com/Jornason/472933656254b8dc7f91a14cd4d7070e>

## Proof of Concept

### Summary

This is an executable runtime PoC, not static analysis. It runs on the project's own Hardhat setup against the in-scope contracts at the audit commit and demonstrates the full insolvency path: cover for a target period is committed while the period-start snapshot shows 1000 collateral units, a valid prior-period payout then reduces live collateral to 100, the committed cover still settles for 1000, and a later valid 1000-unit covered loss is closed with only 100 paid — 900 units of a valid claim are never paid and the incident is terminally closed.

Two control cells isolate the root cause:

* The same sequence with sufficient remaining collateral pays in full (C2).
* The same commit is correctly rejected when the payout happens before the target period starts (C3).

### Environment

```
Repository: https://github.com/immunefi-team/audit-comp-firelight
Commit/tag: v1_audit_ready @ 42f9ea5e43d88b35197b901acc2a8c24b314fa62
Framework: Hardhat (the repository's own toolchain; no .env required)
Network: local Hardhat runtime
Mainnet transactions: none
Read-only live checks: not required (no live dependency)
```

The PoC uses only the repository's own test helpers (`test/setup/fixtures.js`) and the repository's `MockAggregatorV3` with a constant 1 USD answer for deterministic accounting. The bug is price-independent: the underpayment occurs in native vault-asset units with a constant price, so no oracle or depeg assumption is involved. The first-loss-buffer wallet holds a zero balance throughout, so the shortfall is attributable solely to vault-side stale capacity accounting.

### Files

```
test/aecgiv_post_payout_capacity.test.js   (supplied below; drop into the repo's test/ directory)
```

### Reproduction Steps

{% stepper %}
{% step %}

## Clone the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

{% endstep %}

{% step %}

## Copy the PoC file

```bash
cp aecgiv_post_payout_capacity.test.js test/
```

{% endstep %}

{% step %}

## Run the attack cell

```bash
npx hardhat test test/aecgiv_post_payout_capacity.test.js --grep 'AECGIV-C1'
```

{% endstep %}

{% step %}

## Run the solvency control

```bash
npx hardhat test test/aecgiv_post_payout_capacity.test.js --grep 'AECGIV-C2'
```

{% endstep %}

{% step %}

## Run the boundary control

```bash
npx hardhat test test/aecgiv_post_payout_capacity.test.js --grep 'AECGIV-C3'
```

{% endstep %}
{% endstepper %}

### Expected Output

C1 (attack / impact cell):

```
  AECGIV independent post-payout capacity verifier
AECGIV_EVIDENCE {"cell":"AECGIV-C1","targetPeriod":"2","assetsAtTargetStart":"1000000000","liveAssetsAtCommit":"1000000000","liveAssetsAfterPriorPayout":"100000000","historicalAssetsAfterPriorPayout":"1000000000","committedCover":"1000000000000000000000","settledCover":"1000000000000000000000","newIncidentApprovedLoss":"1000000000000000000000","newIncidentPaid":"100000000","underpaidNative":"900000000","newIncidentStatus":"4"}
    ✔ AECGIV-C1: target cover committed before a prior payout settles after live collateral falls and later closes 900 underpaid

  1 passing
```

C2 (solvency control):

```
AECGIV_EVIDENCE {"cell":"AECGIV-C2","targetPeriod":"2","assetsAtTargetStart":"2000000000","liveAssetsAfterPriorPayout":"1100000000","committedCover":"1000000000000000000000","approvedLoss":"1000000000000000000000","actualPaid":"1000000000","underpaidNative":"0","liveAssetsAfterLaterIncident":"100000000"}
    ✔ AECGIV-C2: same sequence with 2000 starting assets fully pays the later 1000-cover incident

  1 passing
```

C3 (boundary control):

```
AECGIV_EVIDENCE {"cell":"AECGIV-C3","targetPeriod":"2","assetsAtTargetStart":"100000000","rejectedMatchingCapacity":"1000000000000000000000","maxAcceptedCapacity":"100000000000000000000","settledCover":"100000000000000000000","priorPayoutBeforeTargetStart":"900000000"}
    ✔ AECGIV-C3: payout before the target period start snapshots only 100 and rejects a 1000-capacity commit

  1 passing
```

Key invariants (C1):

```
totalAssetsAt(targetPeriodStart) == 1000e6   // stale snapshot still high after payout
totalAssets()                    == 100e6    // live collateral drained by valid prior payout
settledCover                     == 1000e18  // cover settled against the stale snapshot
newIncidentPaid                  == 100e6    // valid 1000-cover loss underpaid by 900
newIncidentStatus                == 4        // incident terminally CLOSED despite shortfall
```

### What the PoC Proves

1. A target-period allocation commitment passes on-chain validation against the period-start checkpoint while live collateral equals the snapshot.
2. A valid prior-period incident approved during the target period pays 900 out of the vault through the normal `approveCurrentAssessment` → `FirelightVault.payout` path, reducing live assets to 100 while `totalAssetsAt(targetPeriodStart)` remains 1000.
3. `settleCoverOrder` still settles the committed 1000 cover because no settlement-time check compares the settled amount against live collateral.
4. A later valid incident assessed for exactly the settled cover amount, 1000, is paid only 100 and is terminally closed (`IncidentStatus.CLOSED`), leaving 900 of a valid covered loss permanently unpaid.
5. C2 proves the shortfall is caused by collateral depletion, not by the assessment path: with 2000 starting assets the identical sequence pays the later claim in full.
6. C3 proves the vulnerable window is precisely the post-period-start payout: when the payout happens before the period starts, the checkpoint is low and `commitAllocation` reverts with `MatchingCapacityOverflow` for the same 1000 capacity.

### PoC Source

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId } = require('./setup/fixtures.js')

const PERIOD = 7 * 24 * 60 * 60
const VAULT_DECIMALS = 6
const CANONICAL_DECIMALS = 18
const A100 = ethers.parseUnits('100', VAULT_DECIMALS)
const A900 = ethers.parseUnits('900', VAULT_DECIMALS)
const A1000 = ethers.parseUnits('1000', VAULT_DECIMALS)
const A1100 = ethers.parseUnits('1100', VAULT_DECIMALS)
const A2000 = ethers.parseUnits('2000', VAULT_DECIMALS)
const C100 = ethers.parseUnits('100', CANONICAL_DECIMALS)
const C900 = ethers.parseUnits('900', CANONICAL_DECIMALS)
const C1000 = ethers.parseUnits('1000', CANONICAL_DECIMALS)
const MARKET = ethers.encodeBytes32String('AECGIV-market')
const PROTOCOL = 'AECGIV-protocol'
const CHAIN_ID = 814
const MARKET_ID = computeMarketId(CHAIN_ID, PROTOCOL, MARKET)

function treeFor(orderId, allocatedCover) {
  return StandardMerkleTree.of(
    [[orderId, [[MARKET_ID, allocatedCover]]]],
    ['uint256', '(bytes32,uint256)[]']
  )
}

function alloc(allocatedCover) {
  return [{ marketId: MARKET_ID, allocatedCover }]
}

function evidence(cell, fields) {
  console.log(`AECGIV_EVIDENCE ${JSON.stringify({ cell, ...fields })}`)
}

async function deploySystem(initialAssets) {
  const ctx = await deployVault({
    decimals: VAULT_DECIMALS,
    initial_deposit_limit: ethers.parseUnits('1000000', VAULT_DECIMALS),
    period_configuration_duration: PERIOD
  })
  const signers = await ethers.getSigners()
  const lp = ctx.users[0]
  const buyer = ctx.users[1]
  const curator = signers[9]
  const allocatorRole = signers[10]
  const configAdmin = signers[11]
  const adminRole = signers[12]
  const premiumCollector = signers[13]
  const flbWallet = signers[14]
  const incidentCurator = signers[15]
  const assessmentApprover = signers[16]
  const assessmentRejecter = signers[17]
  const incidentInvalidator = signers[18]
  const payoutAdmin = signers[8]
  const priceFeedAdmin = signers[7]
  const incidentPayoutReceiver = signers[6]

  await ctx.utils.mintAndApprove(initialAssets, lp)
  await ctx.firelight_vault.connect(lp).deposit(initialAssets, lp.address)

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ['AECGIV Cover', 'AECGIV', '', ctx.deployer.address, ctx.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, ethers.parseUnits('1', 18))

  const CoverOrderAllocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocator,
    [{
      vault: ctx.firelight_vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: PERIOD * 10,
      premiumTokens: [ctx.token_contract.target],
      admin: ctx.deployer.address,
      adminRole: adminRole.address,
      curatorRole: curator.address,
      allocatorRole: allocatorRole.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{
        chainId: CHAIN_ID,
        protocol: PROTOCOL,
        maxProtocolConcentrationBps: 10_000
      }],
      newMarkets: [{ chainId: CHAIN_ID, protocol: PROTOCOL, market: MARKET }],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: ctx.token_contract.target,
        firstLossBuffer: flbWallet.address,
        effectiveLeverage: 12_000,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0
      }
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await coverNFT.connect(ctx.deployer).grantRole(await coverNFT.MINTER_ROLE(), allocator.target)

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      ctx.deployer.address,
      incidentCurator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      ctx.deployer.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      incidentPayoutReceiver.address,
      allocator.target,
      priceFeed.target,
      PERIOD * 10
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await ctx.firelight_vault.connect(ctx.deployer).grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target)
  await ctx.firelight_vault.connect(ctx.deployer).grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target)
  await ctx.firelight_vault.connect(ctx.payout_allowlister).addToPayoutAllowlist(incidentPayoutReceiver.address)

  await ctx.token_contract.mintTo(buyer.address, ethers.parseUnits('100', VAULT_DECIMALS))
  await ctx.token_contract.connect(buyer).approve(allocator.target, ethers.MaxUint256)

  return {
    ...ctx,
    lp,
    buyer,
    curator,
    allocatorRole,
    configAdmin,
    flbWallet,
    incidentCurator,
    assessmentApprover,
    coverNFT,
    priceFeed,
    allocator,
    incidentManager,
    incidentPayoutReceiver
  }
}

async function advanceOnePeriod(ctx) {
  const before = await ctx.firelight_vault.currentPeriod()
  await time.increaseTo((await ctx.firelight_vault.currentPeriodEnd()) + 1n)
  expect(await ctx.firelight_vault.currentPeriod()).to.equal(before + 1n)
  await ctx.priceFeed.setAnswer(ethers.parseUnits('1', 18))
}

async function createOrder(ctx, cover = C1000) {
  const orderId = await ctx.allocator.nextCoverOrderId()
  await ctx.allocator.connect(ctx.curator).createCoverOrder(
    ctx.buyer.address,
    ctx.buyer.address,
    ctx.buyer.address,
    ctx.token_contract.target,
    [{ marketId: MARKET_ID, coverRateAnnual: 500, coverAmount: cover }],
    0
  )
  return orderId
}

async function commitAndSettle(ctx, orderId, cover = C1000, matchingCapacity = cover) {
  const period = await ctx.firelight_vault.currentPeriod()
  const merkle = treeFor(orderId, cover)
  await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(period, merkle.root, cover, matchingCapacity)
  await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(orderId, alloc(cover), merkle.getProof(0))
  return merkle
}

async function approveIncident(ctx, captureTimestamp, label, coverTokenId, lossCanonical) {
  const incidentId = await ctx.incidentManager.nextIncidentId()
  await ctx.incidentManager.connect(ctx.incidentCurator).createIncident(captureTimestamp, label, ethers.id(label))
  await ctx.incidentManager.connect(ctx.incidentCurator).confirmIncident(incidentId, `ipfs://${label}`)
  await ctx.incidentManager.connect(ctx.incidentCurator).addAssessmentLosses(incidentId, [{
    coverTokenId,
    marketId: MARKET_ID,
    amount: lossCanonical
  }])
  await ctx.incidentManager.connect(ctx.incidentCurator).submitCurrentAssessment(incidentId)
  const before = await ctx.token_contract.balanceOf(ctx.incidentPayoutReceiver.address)
  const tx = await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(incidentId)
  const receipt = await tx.wait()
  const after = await ctx.token_contract.balanceOf(ctx.incidentPayoutReceiver.address)
  const [incident] = await ctx.incidentManager.getIncident(incidentId)
  const [recipientLoss, roundExists] = await ctx.incidentManager.getCurrentPayoutRecipientAssessmentLoss(incidentId, ctx.buyer.address)
  return { incidentId, receipt, paid: after - before, incident, recipientLoss, roundExists }
}

async function committedTargetThenPriorPayout(initialAssets) {
  const ctx = await deploySystem(initialAssets)
  expect(await ctx.token_contract.balanceOf(ctx.flbWallet.address)).to.equal(0n)

  const oldOrderId = await createOrder(ctx)
  await advanceOnePeriod(ctx)
  await commitAndSettle(ctx, oldOrderId)
  expect(await ctx.coverNFT.ownerOf(oldOrderId)).to.equal(ctx.buyer.address)

  const newOrderId = await createOrder(ctx)
  const oldCaptureTimestamp = BigInt(await time.latest())
  await advanceOnePeriod(ctx)

  const targetPeriod = await ctx.firelight_vault.currentPeriod()
  const targetStart = await ctx.firelight_vault.currentPeriodStart()
  const startAssets = await ctx.firelight_vault.totalAssetsAt(targetStart)
  const liveAtCommit = await ctx.firelight_vault.totalAssets()
  const targetTree = treeFor(newOrderId, C1000)
  await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(targetPeriod, targetTree.root, C1000, C1000)

  const oldResult = await approveIncident(ctx, oldCaptureTimestamp, `AECGIV-old-${initialAssets.toString()}`, oldOrderId, C900)
  expect(oldResult.paid).to.equal(A900)

  return { ctx, oldOrderId, newOrderId, targetTree, oldCaptureTimestamp, targetPeriod, targetStart, startAssets, liveAtCommit, oldResult }
}

describe('AECGIV independent post-payout capacity verifier', function () {
  this.timeout(180000)

  it('AECGIV-C1: target cover committed before a prior payout settles after live collateral falls and later closes 900 underpaid', async function () {
    const s = await loadFixture(committedTargetThenPriorPayout.bind(null, A1000))
    const { ctx, newOrderId, targetTree } = s
    expect(s.startAssets).to.equal(A1000)
    expect(s.liveAtCommit).to.equal(A1000)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(A100)
    expect(await ctx.firelight_vault.totalAssetsAt(s.targetStart)).to.equal(A1000)

    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(newOrderId, alloc(C1000), targetTree.getProof(0))
    expect((await ctx.allocator.getCoverOrder(newOrderId)).allocatedCoverAmount).to.equal(C1000)

    const newCaptureTimestamp = BigInt(await time.latest())
    await ctx.priceFeed.setAnswer(ethers.parseUnits('1', 18))
    const newResult = await approveIncident(ctx, newCaptureTimestamp, 'AECGIV-new-underpaid', newOrderId, C1000)

    expect(newResult.roundExists).to.equal(true)
    expect(newResult.recipientLoss).to.equal(C1000)
    expect(newResult.paid).to.equal(A100)
    expect(newResult.incident.status).to.equal(4n)
    expect(newResult.incident.vaultPaidAmount).to.equal(A100)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(0n)

    evidence('AECGIV-C1', {
      targetPeriod: s.targetPeriod.toString(),
      targetStart: s.targetStart.toString(),
      assetsAtTargetStart: s.startAssets.toString(),
      liveAssetsAtCommit: s.liveAtCommit.toString(),
      liveAssetsAfterPriorPayout: A100.toString(),
      historicalAssetsAfterPriorPayout: (await ctx.firelight_vault.totalAssetsAt(s.targetStart)).toString(),
      committedCover: C1000.toString(),
      settledCover: (await ctx.allocator.getAllocationCommitment(s.targetPeriod)).totalSettledCover.toString(),
      newIncidentApprovedLoss: C1000.toString(),
      newIncidentPaid: newResult.paid.toString(),
      underpaidNative: (A1000 - newResult.paid).toString(),
      newIncidentStatus: newResult.incident.status.toString()
    })
  })

  it('AECGIV-C2: same sequence with 2000 starting assets fully pays the later 1000-cover incident', async function () {
    const s = await loadFixture(committedTargetThenPriorPayout.bind(null, A2000))
    const { ctx, newOrderId, targetTree } = s
    expect(s.startAssets).to.equal(A2000)
    expect(s.liveAtCommit).to.equal(A2000)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(A1100)

    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(newOrderId, alloc(C1000), targetTree.getProof(0))
    const newCaptureTimestamp = BigInt(await time.latest())
    await ctx.priceFeed.setAnswer(ethers.parseUnits('1', 18))
    const newResult = await approveIncident(ctx, newCaptureTimestamp, 'AECGIV-control-paid', newOrderId, C1000)

    expect(newResult.roundExists).to.equal(true)
    expect(newResult.recipientLoss).to.equal(C1000)
    expect(newResult.paid).to.equal(A1000)
    expect(newResult.incident.status).to.equal(4n)
    expect(newResult.incident.vaultPaidAmount).to.equal(A1000)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(A100)

    evidence('AECGIV-C2', {
      targetPeriod: s.targetPeriod.toString(),
      assetsAtTargetStart: s.startAssets.toString(),
      liveAssetsAfterPriorPayout: A1100.toString(),
      committedCover: C1000.toString(),
      approvedLoss: C1000.toString(),
      actualPaid: newResult.paid.toString(),
      underpaidNative: '0',
      liveAssetsAfterLaterIncident: (await ctx.firelight_vault.totalAssets()).toString()
    })
  })

  it('AECGIV-C3: payout before the target period start snapshots only 100 and rejects a 1000-capacity commit', async function () {
    const ctx = await loadFixture(deploySystem.bind(null, A1000))
    const oldOrderId = await createOrder(ctx)
    await advanceOnePeriod(ctx)
    await commitAndSettle(ctx, oldOrderId)

    const newOrderId = await createOrder(ctx)
    const oldCaptureTimestamp = BigInt(await time.latest())
    const oldResult = await approveIncident(ctx, oldCaptureTimestamp, 'AECGIV-preperiod-old', oldOrderId, C900)
    expect(oldResult.paid).to.equal(A900)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(A100)

    await advanceOnePeriod(ctx)
    const targetPeriod = await ctx.firelight_vault.currentPeriod()
    const targetStart = await ctx.firelight_vault.currentPeriodStart()
    expect(await ctx.firelight_vault.totalAssetsAt(targetStart)).to.equal(A100)

    const fullTree = treeFor(newOrderId, C1000)
    await expect(ctx.allocator.connect(ctx.allocatorRole).commitAllocation(targetPeriod, fullTree.root, C1000, C1000))
      .to.be.revertedWithCustomError(ctx.allocator, 'MatchingCapacityOverflow')
      .withArgs(C1000, C100)

    const boundedTree = treeFor(newOrderId, C100)
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(targetPeriod, boundedTree.root, C100, C100)
    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(newOrderId, alloc(C100), boundedTree.getProof(0))
    expect((await ctx.allocator.getCoverOrder(newOrderId)).allocatedCoverAmount).to.equal(C100)

    evidence('AECGIV-C3', {
      targetPeriod: targetPeriod.toString(),
      assetsAtTargetStart: A100.toString(),
      rejectedMatchingCapacity: C1000.toString(),
      maxAcceptedCapacity: C100.toString(),
      settledCover: C100.toString(),
      priorPayoutBeforeTargetStart: oldResult.paid.toString()
    })
  })
})
```

### Raw Runtime Evidence

Raw passing output for each isolated cell run is included with this report's local package (`evidence/aecgiv_c1_runtime.raw.txt`, `evidence/aecgiv_c2_runtime.raw.txt`, `evidence/aecgiv_c3_runtime.raw.txt`), each with `EXIT_CODE=0` and exactly one `1 passing`. The commands above reproduce this output from a clean checkout.
