# 88214 sc high period p 1 cover capacity is computed from collateral a pending period p claim will consume

**Submitted on Aug 12th 2026 at 09:29:19 UTC by @shadowHunter for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88214
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Description

The amount of cover Firelight may sell in a period is bounded on-chain by `_computeAvailableCapacity`. That function values the collateral as the **live first-loss buffer balance** plus the **staked assets snapshot**, and applies the leverage / capital adequacy ratio:

`contracts/core/CoverOrderAllocator.sol:696-725`

```solidity
function _computeAvailableCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period
) internal view returns (uint256 totalAvailableCapacity) {
    uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);

    CapacityConfig storage config = _getEffectiveCapacityConfig(period);

    uint256 firstLossBufferUSDCanonical = Decimals.convert(
        config.firstLossBufferToken.balanceOf(config.firstLossBuffer),   // :705  live balance
        _readDecimals(address(config.firstLossBufferToken)),
        CANONICAL_DECIMALS,
        Math.Rounding.Floor
    );
    uint256 totalAssetsCanonical = Decimals.convert(
        $.vault.totalAssetsAt($.vault.currentPeriodStart()),             // :711  period-start snapshot
        $.vaultAssetDecimals,
        CANONICAL_DECIMALS,
        Math.Rounding.Floor
    );
    uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
    uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

    uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
    totalAvailableCapacity = strictCapacity.mulDiv(
        BPS_DENOMINATOR + config.divergenceToleranceBps,
        BPS_DENOMINATOR
    );
}
```

**Neither term is reduced by exposure that is already owed.** The payout window for a capture period `P` spans periods `P` and `P+1`:

`contracts/core/FirelightVault.sol:1130-1132`

```solidity
function _isPeriodInPayoutWindow(uint256 period, uint256 current) internal pure returns (bool) {
    return current == period || current == period + 1;
}
```

So a period-`P` claim is legitimately paid **during period `P+1`**, drawing on exactly the first-loss buffer and vault assets that `commitAllocation(P+1)` already counted as free capacity for period `P+1`'s cover. **The same collateral backs two periods of cover simultaneously.**

### The protocol already recognises this entanglement — on the wrong path

`FirelightVault` deliberately blocks *deposits* for precisely this reason. The comment is explicit, `contracts/core/FirelightVault.sol:1112-1118`:

```solidity
/// @dev True while an incident is active in the current period or the previous one. The previous period is
/// included because a claim captured then is still payable this period, so deposits stay blocked until that
/// payout window closes; otherwise fresh deposits would back unresolved exposure.
function _hasActiveIncident() internal view returns (bool) {
    uint256 period = currentPeriod();
    return hasActiveIncident[period] || (period > 0 && hasActiveIncident[period - 1]);
}
```

The team identified that fresh capital must not back unresolved exposure, and hard-gated the depositor path on-chain. The **capacity** path — the one that actually determines solvency — was left ungated. `CoverOrderAllocator.sol` contains **zero** references to `hasActiveIncident`; neither `commitAllocation` nor `_settleCoverOrder` consults incident state at all.

### Anticipated objection

`matchingCapacity` is supplied by the `ALLOCATOR_ROLE` and only *upper*-bounded on-chain (`_validateMatchingCapacity`, `contracts/core/CoverOrderAllocator.sol:394-403`), so one could argue the off-chain matcher should net out pending incidents. Three reasons that does not dispose of the finding:

1. The documented off-chain formula the matcher is told to implement, `contracts/core/interfaces/ICoverOrderAllocator.sol:39-41`, contains **no term for unresolved incidents either**. A matcher implementing the specification exactly will over-sell:

   ```
   *   4. Compute capacity: `(config.firstLossBufferToken.balanceOf(config.firstLossBuffer)
   *      + vault.totalAssets() * assetPriceUSD / 10**priceFeedDecimals) * config.effectiveLeverage / config.minCAR`
   ```

   Note in passing that this specification also disagrees with the on-chain implementation on *which* asset figure to use: the spec says `vault.totalAssets()` (**live**), while `_computeAvailableCapacity` at `:711` reads `vault.totalAssetsAt(vault.currentPeriodStart())` (**period-start snapshot**). The two diverge by every deposit, payout and reward distribution that lands inside the period, so a compliant matcher's `matchingCapacity` and the chain's own bound are computed from different collateral figures.
2. The entire purpose of `_validateMatchingCapacity`, `MAX_LEVERAGE_FACTOR`, `MIN_CAR_BPS` and `divergenceToleranceBps` is that the **chain**, not the allocator, enforces capital adequacy. That enforcement evaporates in the one window where it matters.
3. `_settleCoverOrder` has no incident check either, so a gate on `commitAllocation` alone would be insufficient.

### Why this is not the disclosed known issue

The program lists as a known issue:

> "Aggregate claims are not capped on-chain against total cover sold; the MVP relies on trusted incident/assessment roles and the approval process to prevent duplicate, excessive, or otherwise invalid claims."

That disclosure is about the **claims** side: nothing stops assessors from approving losses exceeding the cover that was sold. This finding is on the **capacity** side and concerns a different pair of quantities entirely:

|                      | Known issue                    | This finding                                                                           |
| -------------------- | ------------------------------ | -------------------------------------------------------------------------------------- |
| Quantities compared  | assessed claims vs. cover sold | cover sold vs. **unencumbered collateral**                                             |
| Control              | Trusted assessment roles       | The on-chain guard `_validateMatchingCapacity` / `MIN_CAR_BPS` / `MAX_LEVERAGE_FACTOR` |
| Mitigation relied on | Human approval process         | Nothing — the chain computes and accepts the figure itself                             |

Here the claim is entirely legitimate, correctly sized, and within the cover sold for its period. The defect is that the *collateral* backing period `P+1`'s capacity has not been reduced by that legitimate period-`P` obligation. No assessor misbehaves and no claim is excessive; the numbers the chain itself computes are wrong.

Likewise this is not Macro M-1 (*unbounded `effectiveLeverage`*, Fixed) or L-4 (*CAR floor 1.0 vs. 1.2*, Fixed). Both concern the **multiplier** applied to collateral. This finding concerns the **collateral figure the multiplier is applied to**, and reproduces at a fully compliant 2× leverage with `divergenceToleranceBps = 0`.

### On the privileged-actor exclusion

`matchingCapacity` is submitted by `ALLOCATOR_ROLE`, but this report does not require that role to misbehave — it submits the figure the documented formula produces, and the chain independently computes and accepts it. The report demonstrates a **bypass of an intended protocol constraint**: `_validateMatchingCapacity`, `MIN_CAR_BPS`, `MAX_LEVERAGE_FACTOR` and `divergenceToleranceBps` exist precisely so the chain, not the allocator, enforces capital adequacy, and step 6 below shows the chain approving a capacity figure it rejects one period later on unchanged collateral.

***

## Steps to reproduce

No malicious actor is involved. Every step is a role performing its documented function.

**1. Collateral is in place.** The first-loss buffer holds $1,000,000 and the vault has $4,000,000 of staked assets at the period-start snapshot.

```solidity
// _computeAvailableCapacity, CoverOrderAllocator.sol:704-717
// availableCollateral = $1,000,000 + $4,000,000 = $5,000,000
// effectiveLeverage / minCAR = 24000 / 12000 = 2x
// totalAvailableCapacity = $10,000,000
```

**2. Period 2: the allocator commits $10,000,000 of capacity and settles $5,000,000 of cover.**

```solidity
// CoverOrderAllocator.sol:332-357
commitAllocation(2, merkleRoot, /*totalAllocated*/ 5_000_000e18, /*matchingCapacity*/ 10_000_000e18);
// CoverOrderAllocator.sol:446-448
settleCoverOrder(0, allocations, proof);
```

**3. A period-2 claim is assessed but not yet paid.** This is normal: assessment rounds take time, and the protocol explicitly allows payment during period 3.

**4. Period 3 begins. Nothing is added to the buffer and nothing new is staked.**

**5. The allocator commits period 3 — and the chain grants the identical capacity.** `_computeAvailableCapacity` re-reads the *same* buffer balance and the *same* staked snapshot, with no deduction for the outstanding period-2 claim.

```solidity
commitAllocation(3, merkleRoot2, 5_000_000e18, 10_000_000e18);   // accepted
// getAllocationCommitment(3).totalAvailableCapacity == getAllocationCommitment(2).totalAvailableCapacity
```

At this instant `vault.isPeriodInPayoutWindow(2) == true` — the period-2 claim is still payable from exactly this collateral. **$10,000,000 of live cover now stands against $5,000,000 of collateral.**

**6. The period-2 claim is paid.** `_executePayout` drains the first-loss buffer first, then the vault (`contracts/core/IncidentManager.sol:641-684`). Buffer → $0, staked → $500,000.

**7. Period 4 proves the collateral was never there.** The allocator submits the *same* capacity figure it was granted for period 3, and the chain now rejects it:

```solidity
commitAllocation(4, merkleRoot3, 5_000_000e18, 10_000_000e18);
// reverts: MatchingCapacityOverflow
```

The identical number is **accepted for period 3 and rejected for period 4**. The only thing that changed in between is that the period-2 claim was actually paid — which is precisely the exposure period 3's capacity should have netted out.

***

## Impact

**Protocol insolvency.** Cover is sold against collateral that is already committed to paying an earlier period's claims:

* After the period-2 payout in the worked example, **$5,000,000 of period-3 cover is backed by $500,000 of remaining collateral** — a 10× breach of the committed capital adequacy ratio, produced by parameters the chain itself validated.
* The example uses the **most conservative legal configuration** (`divergenceToleranceBps = 0`). At the repository fixture's own 2× leverage with a non-zero tolerance the overhang is strictly larger.
* No role misbehaves. The allocator submits a capacity number the on-chain guard computed and accepted; the approver executes a payout inside its documented window.
* The condition recurs on every period boundary where an incident is unresolved, and unresolved incidents are the *normal* state during an assessment.

The on-chain capital-adequacy machinery (`MIN_CAR_BPS`, `MAX_LEVERAGE_FACTOR`, `divergenceToleranceBps`, per-protocol concentration caps) provides no protection here, because every one of them is applied to a collateral figure that has not been reduced by outstanding obligations.

***

## Recommendation

1. **Net outstanding assessed-but-unpaid exposure out of both collateral terms.** Have `IncidentManager` expose per-period outstanding exposure and subtract it inside `_computeAvailableCapacity` before applying `effectiveLeverage / minCAR`:

   ```solidity
   uint256 outstanding = $.incidentManager.outstandingExposureCanonical(); // periods P and P-1
   uint256 availableCollateralCanonical =
       firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical - outstanding;
   ```
2. **A `hasActiveIncident` gate alone is not sufficient.** If the commit lands *before* the incident is created, such a gate never fires. Only a reservation/netting model closes it — or deferring settlement of period `P+1` until period `P`'s payout window has closed.
3. **Gate `_settleCoverOrder` as well as `commitAllocation`.** Capacity consumed at settlement must be validated against the same netted figure, otherwise orders committed before an incident can still be settled after it.
4. **Escrow the first-loss buffer in a contract with per-period reservations** instead of reading a live EOA balance at `CoverOrderAllocator.sol:705`. A balance that can be spent between the commit and the period end is not collateral.

## Proof of Concept

## Proof of Concept

### Environment

42f9ea5e43d88b35197b901acc2a8c24b314fa62 is `v1_audit_ready`

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

### Create `hardhat.poc.config.js` in the repository root

```js
require('@nomicfoundation/hardhat-ethers')
require('@nomicfoundation/hardhat-chai-matchers')
require('@openzeppelin/hardhat-upgrades')

module.exports = {
  defaultNetwork: 'hardhat',
  networks: { hardhat: { allowUnlimitedContractSize: true } },
  solidity: {
    compilers: [
      { version: '0.8.23', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london' } },
      { version: '0.8.28', settings: { optimizer: { enabled: true, runs: 10000 }, evmVersion: 'london', viaIR: true } },
      { version: '0.8.29', settings: { optimizer: { enabled: true, runs: 1000 }, evmVersion: 'london' } },
    ],
    overrides: {
      'contracts/core/CoverOrderAllocator.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 100 }, evmVersion: 'london', viaIR: true },
      },
      'contracts/core/FirelightVault.sol': {
        version: '0.8.28',
        settings: { optimizer: { enabled: true, runs: 800 }, evmVersion: 'london', viaIR: true },
      },
    },
  },
  paths: { sources: './contracts', tests: './test', cache: './.audit-cache', artifacts: './.audit-artifacts' },
}
```

### Save the PoC as `test/poc_capacity_double_count.js`

```js
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const { deployCoverOrderAllocator } = require('./setup/fixtures.js')

// Canonical USD inside the allocator is 18 decimals; the vault asset / FLB token is 6.
const usd = (n) => ethers.parseUnits(String(n), 18)
const usd6 = (n) => ethers.parseUnits(String(n), 6)

describe('PoC: period P+1 capacity is computed from collateral a pending period-P claim will consume', function () {
  this.timeout(120000)

  // Helper: create -> commit -> settle one full-cover order for `targetPeriod`.
  const sellCover = async (ctx, orderId, targetPeriod, cover, capacity, buyer) => {
    const { allocator, usdc, curator, allocatorRole, beneficiary, marketIdA } = ctx

    // createCoverOrder always targets _currentPeriod() + 1.
    await allocator
      .connect(curator)
      .createCoverOrder(
        buyer.address,
        beneficiary.address,
        `cover for period ${targetPeriod}`,
        await usdc.getAddress(),
        [{ marketId: marketIdA, coverRateAnnual: 500, coverAmount: cover }],
        0,
      )

    const order = await allocator.getCoverOrder(orderId)
    const nativeScale = 10n ** 12n
    const premiumNative = (order.totalPremiumAmount + nativeScale - 1n) / nativeScale
    await usdc.mint(buyer.address, premiumNative)
    await usdc.connect(buyer).approve(await allocator.getAddress(), premiumNative)

    await ctx.advanceToPeriod(targetPeriod)

    const tree = StandardMerkleTree.of([[BigInt(orderId), [[marketIdA, cover]]]], [
      'uint256',
      '(bytes32,uint256)[]',
    ])
    await allocator
      .connect(allocatorRole)
      .commitAllocation(targetPeriod, tree.root, cover, capacity)

    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(orderId, [{ marketId: marketIdA, allocatedCover: cover }], tree.getProof(0))
  }

  it('sells a second full period of cover against collateral that is still liable for period P', async function () {
    const ctx = await loadFixture(deployCoverOrderAllocator)
    const { allocator, vault, usdc, curator, allocatorRole, buyer1, buyer2, beneficiary, firstLossBufferWallet, marketIdA } =
      ctx

    // ---- Collateral: $1M first-loss buffer + $4M staked assets --------------
    // capacity = ($1M + $4M) * effectiveLeverage/minCAR (24000/12000 = 2x) = $10M
    await usdc.mint(firstLossBufferWallet.address, usd6(1_000_000))
    await vault.setTotalAssetsAtSnapshot(usd6(4_000_000))

    // The fixture only configures periods 1-3; period 4 is used for the final check.
    await vault.setPeriodConfiguration(4, { epoch: 0, duration: ctx.PERIOD_DURATION, startingPeriod: 4 })

    const CAPACITY = usd(10_000_000)
    const COVER = usd(5_000_000) // below the 60% Morpho concentration cap ($6M)

    // ---- Period 2: sell $5M of cover ---------------------------------------
    await sellCover(ctx, 0, 2, COVER, CAPACITY, buyer1)
    const commit2 = await allocator.getAllocationCommitment(2)
    expect(commit2.totalAvailableCapacity).to.equal(CAPACITY)
    expect(commit2.totalSettledCover).to.equal(COVER)

    // ---- Period 3: the SAME collateral is counted again ---------------------
    // Nothing is added to the buffer and nothing is staked between the periods.
    const flbBefore = await usdc.balanceOf(firstLossBufferWallet.address)

    await sellCover(ctx, 1, 3, COVER, CAPACITY, buyer2)

    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(flbBefore)

    // A period-2 claim is STILL payable while we are in period 3 — the payout
    // window for capture period P spans {P, P+1}. So the collateral backing the
    // period-3 commitment is simultaneously liable for period-2 losses.
    expect(await vault.isPeriodInPayoutWindow(2)).to.eq(true)

    const commit3 = await allocator.getAllocationCommitment(3)
    // The chain granted period 3 exactly the same capacity as period 2, with no
    // deduction for the outstanding period-2 exposure.
    expect(commit3.totalAvailableCapacity).to.equal(commit2.totalAvailableCapacity)
    expect(commit3.totalSettledCover).to.equal(COVER)

    // $10M of live cover now stands against $5M of collateral.
    const liveCover = commit2.totalSettledCover + commit3.totalSettledCover
    expect(liveCover).to.equal(usd(10_000_000))

    // ---- The period-2 claim now lands, consuming that same collateral -------
    // _executePayout drains the first-loss buffer first, then the vault.
    await usdc
      .connect(firstLossBufferWallet)
      .transfer(ctx.deployer.address, await usdc.balanceOf(firstLossBufferWallet.address))
    await vault.setTotalAssetsAtSnapshot(usd6(500_000)) // $4M staked - $3.5M paid

    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(0n)

    // ---- Period 4 proves the collateral was never there --------------------
    // Remaining collateral is $0.5M, so the chain now refuses the very capacity
    // it granted for period 3 — while period 3's $5M of cover is still live.
    await allocator
      .connect(curator)
      .createCoverOrder(
        buyer1.address,
        beneficiary.address,
        'cover for period 4',
        await usdc.getAddress(),
        [{ marketId: marketIdA, coverRateAnnual: 500, coverAmount: COVER }],
        0,
      )
    await ctx.advanceToPeriod(4)

    const tree4 = StandardMerkleTree.of([[2n, [[marketIdA, COVER]]]], ['uint256', '(bytes32,uint256)[]'])
    await expect(
      allocator.connect(allocatorRole).commitAllocation(4, tree4.root, COVER, CAPACITY),
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')

    // Same capacity number: accepted for period 3, rejected for period 4. The
    // only difference is that the period-2 claim has now actually been paid —
    // which is exactly the exposure period 3's capacity should have netted out.
  })
})
```

### Run

```bash
node node_modules/hardhat/internal/cli/bootstrap.js --config hardhat.poc.config.js test test/poc_capacity_double_count.js
```

### Output

```
  PoC: period P+1 capacity is computed from collateral a pending period-P claim will consume
    √ sells a second full period of cover against collateral that is still liable for period P (2525ms)


  1 passing (3s)
```

### What the output proves

The passing assertions establish the following chain of facts against the real `CoverOrderAllocator`:

1. `commit2.totalAvailableCapacity == $10,000,000` — the chain computed $10M of capacity from $5M of collateral at the configured 2× multiplier, and `$5,000,000` of cover settled.
2. `usdc.balanceOf(firstLossBufferWallet) == flbBefore` after the period-3 sale — proof that **no new collateral was added** between the two periods.
3. `vault.isPeriodInPayoutWindow(2) == true` while committing period 3 — proof that the period-2 claim is still payable from that same collateral at that exact moment.
4. `commit3.totalAvailableCapacity == commit2.totalAvailableCapacity` — the decisive assertion. Period 3 received the **identical** capacity grant, with no deduction for the outstanding period-2 exposure.
5. `liveCover == $10,000,000` — two periods of cover, each $5M, live simultaneously against $5M of collateral.
6. The final `commitAllocation(4, ..., CAPACITY)` reverts `MatchingCapacityOverflow`. This is the same capacity number the chain accepted at step 4. The only intervening change is that the period-2 claim was paid, draining the buffer to zero and reducing staked assets to $500,000.

Step 6 is the clearest statement of the bug: the protocol's own capacity guard rejects, one period later, the exact figure it approved — because the obligation it should have netted out has now been settled in cash.
