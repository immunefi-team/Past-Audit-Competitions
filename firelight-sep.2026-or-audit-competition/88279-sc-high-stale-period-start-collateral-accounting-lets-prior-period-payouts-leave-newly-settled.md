# 88279 sc high stale period start collateral accounting lets prior period payouts leave newly settled cover fully unbacked

**Submitted on Aug 12th 2026 at 15:19:30 UTC by @Porcupine17126 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88279
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator._computeAvailableCapacity()` values vault collateral exclusively through:

```solidity
$.vault.totalAssetsAt($.vault.currentPeriodStart())
```

That historical value is not reduced when a claim from the previous period is paid during the current period. Firelight explicitly permits a claim to be paid in its capture period or the following period. Consequently, a legitimate period `P` payout executed in period `P + 1` can drain the vault, while a later period `P + 1` allocation commitment continues to count the pre-payout balance recorded at the start of `P + 1`.

The allocator therefore accepts and settles new cover against collateral that has already left the vault. The buyer is charged a premium and receives a `MATCHED` CoverNFT, but a valid claim for that cover can receive a zero vault payout. The PoC funds and approves the protocol's external first-loss buffer, then exhausts it through the legitimate prior claim before demonstrating the stale vault accounting.

Suggested severity: **Critical — protocol insolvency**. The discrepancy can reach 100% of the vault-backed cover sold for the new period.

## Affected code

Scope commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

In `contracts/core/CoverOrderAllocator.sol:696-724`, capacity always uses the historical period-start balance:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

In `contracts/core/FirelightVault.sol:903-927`, payouts are intentionally allowed in the capture period or the following period and are paid from currently available vault assets:

```solidity
if (!_isPeriodInPayoutWindow(capturePeriod, _currentPeriod)) {
    revert InvalidCapturePeriod();
}

uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The payout records the reduced active balance only at the payout's current timestamp (`FirelightVault.sol:952`):

```solidity
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

Because `totalAssetsAt(periodStart)` performs an inclusive historical lookup at the earlier period-start key (`FirelightVault.sol:456-457`), the later checkpoint cannot reduce that value.

## Root cause

The protocol uses two incompatible meanings of vault collateral:

1. Allocation capacity treats the period-start asset checkpoint as immutable for the whole period.
2. A previous-period claim is allowed to consume those same assets after the new period has started.

There is no reservation for the unresolved previous-period liability, and no post-payout adjustment to the current period's matching capacity.

The contract's own documentation confirms that this validation is meant to reflect current backing: `commitAllocation()` describes capacity as "recomputed live," and `recommitAllocation()` says that fresh validation keeps commitments bound to the period's "real collateral." However, both ultimately validate through `_computeAvailableCapacity()`, which combines the live first-loss-buffer balance with the immutable period-start vault checkpoint—even after those vault assets have already been paid out. The snapshot may be intentional; counting already-spent collateral as "real collateral" in a fresh commitment is the failure.

This is especially likely in the intended workflow because the matching engine pins its state to the period-start block for deterministic matching. Even if operators resolve the old incident before committing the new allocation—the safest possible ordering—the on-chain capacity check still accepts the stale period-start balance.

## Impact

* New cover can be fully matched with no remaining vault collateral.
* The buyer pays the premium and receives a valid CoverNFT, but an approved loss can close with `vaultPaidAmount == 0`.
* The protocol records liabilities exceeding its remaining assets: protocol insolvency.
* The maximum discrepancy is the amount paid to prior-period incidents during the new period, up to the entire vault balance.

This does **not** rely on oracle manipulation, an asset depeg, a fee-on-transfer token, a malicious privileged role, or an unfunded/unapproved first-loss buffer.

The first-loss-buffer wallet starts with 1,000 USDC and unlimited approval to `IncidentManager`, satisfying the documented operational assumption. The first valid 11,000 USD claim legitimately consumes that 1,000 USDC buffer plus 10,000 vault assets; the approval remains sufficient afterward. At the next commitment, the allocator correctly observes the buffer's live zero balance and attributes zero capacity to it, but still attributes 10,000 to the already-paid vault checkpoint. Only then does it settle a different cover for the next period. The root cause is therefore stale `FirelightVault` capacity, not first-loss-buffer custody. It is also distinct from aggregate duplicate claims because the PoC uses two different settled covers in consecutive periods and one valid incident for each.

## Recommended remediation

Capacity must reflect prior-period claims that consume collateral during the current period.

At minimum:

1. Prevent commitment/recommitment and settlement while a previous-period incident remains unresolved, or reserve its maximum payable liability when calculating capacity.
2. After prior-period incidents are resolved, cap the period-start snapshot by the vault assets that remain exposed to a current-period incident. Conceptually, this is the lesser of:
   * the original period-start snapshot; and
   * current active assets plus still-exposed withdrawals for the current period.

For example, the vault can expose a dedicated `remainingBackingForPeriod(period)` view that starts from the period-start snapshot and is reduced by every prior-period payout affecting that period. Both the off-chain matcher and `_validateMatchingCapacity()` should use that same value.

Add an invariant test asserting that after any valid previous-period payout in period `P`, a new period-`P` commitment cannot exceed the vault assets that remain payable to a period-`P` incident.

## Link to Proof of Concept

<https://gist.github.com/aviad12g/c4837d265dfa23eab07ef38aff4535e1>

## Proof of Concept

## Proof of concept

The runnable PoC is:

`test/research/overlappingPeriodCollateral.test.js`

Run from the repository root:

```bash
npx hardhat test test/research/overlappingPeriodCollateral.test.js
```

Expected result:

```
Research PoC: prior-period payout leaves stale next-period capacity
    ✔ reuses vault collateral after a funded FLB and the vault are legitimately exhausted

1 passing
```

### PoC sequence

1. A staker deposits 10,000 vault assets. The external first-loss-buffer wallet is separately funded with 1,000 USDC and grants `IncidentManager` unlimited approval.
2. Cover order `0`, for period 1, is settled for 11,000 USD of cover, exactly backed by the 1,000 USDC buffer plus 10,000 vault assets.
3. While period 1 is healthy, renewal order `1` is created for period 2.
4. A valid period-1 incident occurs and is fully assessed.
5. Period 2 begins. `totalAssetsAt(period2Start)` is 10,000.
6. The period-1 incident is approved in its valid following-period payout window. It first consumes the properly funded and approved 1,000 USDC buffer, then pays all 10,000 vault assets. It closes normally with the buffer balance and `totalAssets()` both equal to zero.
7. The allocator commits and settles order `1` after the old incident is closed. It observes the live first-loss-buffer balance of zero, but the commitment for 10,000 still succeeds because `_computeAvailableCapacity()` reads `totalAssetsAt(period2Start) == 10,000` for the already-paid vault assets.
8. The buyer is charged and receives CoverNFT `1` with status `MATCHED`.
9. A valid period-2 incident is assessed and approved for that cover. It closes with `vaultPaidAmount == 0` because no vault collateral remains.

All calls use the production contracts together: `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`.

## Runnable evidence

Secret Gist: <https://gist.github.com/aviad12g/c4837d265dfa23eab07ef38aff4535e1>

The Gist contains the exact test/research/overlappingPeriodCollateral.test.js file and the captured focused/full-suite execution log (1 passing; 455 passing).

## Complete runnable test

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')

const { deployVault, computeMarketId } = require('../setup/fixtures.js')

describe('Research PoC: prior-period payout leaves stale next-period capacity', function () {
  const DECIMALS = 6
  const PERIOD = 24 * 60 * 60
  const DEPOSIT = ethers.parseUnits('10000', DECIMALS)
  const FIRST_LOSS_BUFFER = ethers.parseUnits('1000', DECIMALS)
  const OLD_COVER = ethers.parseUnits('11000', 18)
  const NEW_COVER = ethers.parseUnits('10000', 18)
  const MARKET = ethers.encodeBytes32String('market-A')
  const MARKET_ID = computeMarketId(1, 'Aave', MARKET)

  async function fixture() {
    const ctx = await deployVault({
      decimals: DECIMALS,
      initial_deposit_limit: ethers.parseUnits('50000', DECIMALS),
      period_configuration_duration: PERIOD,
    })

    const staker = ctx.users[0]
    const buyer = ctx.users[1]
    const firstLossBuffer = ctx.users[2]
    const curator = ctx.limit_updater
    const allocatorRole = ctx.blocklister
    const configAdmin = ctx.pauser
    const adminRole = ctx.period_configuration_updater
    const assessmentApprover = ctx.pauser
    const assessmentRejecter = ctx.period_configuration_updater
    const incidentInvalidator = ctx.rescuer

    await ctx.utils.mintAndApprove(DEPOSIT, staker)
    await ctx.firelight_vault.connect(staker).deposit(DEPOSIT, staker.address)

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.deploy('MockUSDC', 'mUSDC', DECIMALS)
    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, 10n ** 18n)

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      [
        'Firelight Cover',
        'FLCOVER',
        '',
        ctx.deployer.address,
        ctx.deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
      ],
      { kind: 'transparent' },
    )

    const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Allocator,
      [{
        vault: await ctx.firelight_vault.getAddress(),
        premiumCollector: ctx.rescuer.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * PERIOD,
        premiumTokens: [await usdc.getAddress()],
        admin: ctx.deployer.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [{
          protocol: 'Aave',
          chainId: 1,
          maxProtocolConcentrationBps: 10_000,
        }],
        newMarkets: [{ chainId: 1, protocol: 'Aave', market: MARKET }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await coverNFT
      .connect(ctx.deployer)
      .grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())

    const IncidentManager = await ethers.getContractFactory('IncidentManager')
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        ctx.deployer.address,
        curator.address,
        assessmentApprover.address,
        assessmentRejecter.address,
        incidentInvalidator.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ctx.payout_receiver.address,
        await allocator.getAddress(),
        await priceFeed.getAddress(),
        7 * PERIOD,
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), await incidentManager.getAddress())
    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), await incidentManager.getAddress())

    // Satisfy the documented operational assumption: the external first-loss
    // buffer is funded and has approved IncidentManager before underwriting.
    await usdc.mint(firstLossBuffer.address, FIRST_LOSS_BUFFER)
    await usdc.connect(firstLossBuffer).approve(await incidentManager.getAddress(), ethers.MaxUint256)

    await usdc.mint(buyer.address, ethers.parseUnits('1000', DECIMALS))
    await usdc.connect(buyer).approve(await allocator.getAddress(), ethers.MaxUint256)

    const createOrder = async (coverAmount) => allocator.connect(curator).createCoverOrder(
      buyer.address,
      ctx.payout_receiver.address,
      'covered-beneficiary',
      await usdc.getAddress(),
      [{ marketId: MARKET_ID, coverRateAnnual: 500, coverAmount }],
      0,
    )

    return {
      ...ctx,
      buyer,
      firstLossBuffer,
      curator,
      allocatorRole,
      assessmentApprover,
      usdc,
      priceFeed,
      coverNFT,
      allocator,
      incidentManager,
      createOrder,
    }
  }

  function treeFor(orderId, coverAmount) {
    return StandardMerkleTree.of(
      [[orderId, [[MARKET_ID, coverAmount]]]],
      ['uint256', '(bytes32,uint256)[]'],
    )
  }

  async function settlePeriod(ctx, orderId, coverAmount) {
    const tree = treeFor(orderId, coverAmount)
    const period = await ctx.firelight_vault.currentPeriod()
    await ctx.priceFeed.setAnswer(10n ** 18n)
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .commitAllocation(period, tree.root, coverAmount, coverAmount)
    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
      orderId,
      [{ marketId: MARKET_ID, allocatedCover: coverAmount }],
      tree.getProof(0),
    )
  }

  async function openSubmittedIncident(ctx, captureTimestamp, orderId, coverAmount, ref) {
    await ctx.incidentManager
      .connect(ctx.curator)
      .createIncident(captureTimestamp, `incident-${ref}`, ethers.id(ref))
    const incidentId = BigInt(orderId + 1)
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(incidentId, `ipfs://${ref}`)
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(incidentId, [{
      coverTokenId: orderId,
      marketId: MARKET_ID,
      amount: coverAmount,
    }])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(incidentId)
    return incidentId
  }

  it('reuses vault collateral after a funded FLB and the vault are legitimately exhausted', async function () {
    const ctx = await loadFixture(fixture)

    // Order 0 is authored in period 0 for period 1.
    await ctx.createOrder(OLD_COVER)
    await time.increaseTo(await ctx.firelight_vault.currentPeriodEnd())
    expect(await ctx.firelight_vault.currentPeriod()).to.equal(1n)
    await settlePeriod(ctx, 0, OLD_COVER)

    // The period-2 renewal is authored while period 1 is healthy. The curator
    // therefore does not need to create an order after learning of an incident.
    await ctx.createOrder(NEW_COVER)

    // A valid period-1 incident then occurs and is fully assessed, but approval
    // is delayed into its explicitly supported following-period payout window.
    const capture1 = await time.latest()
    const incident1 = await openSubmittedIncident(ctx, capture1, 0, OLD_COVER, 'period-1')

    expect(await ctx.firelight_vault.maxDeposit(ctx.buyer.address)).to.equal(0n)

    await time.increaseTo(await ctx.firelight_vault.currentPeriodEnd())
    expect(await ctx.firelight_vault.currentPeriod()).to.equal(2n)
    const period2Start = await ctx.firelight_vault.currentPeriodStart()
    expect(await ctx.firelight_vault.totalAssetsAt(period2Start)).to.equal(DEPOSIT)

    // The legitimate 11,000 claim first consumes the properly funded 1,000 FLB,
    // then drains the 10,000 vault. No operational assumption is violated.
    expect(await ctx.usdc.balanceOf(ctx.firstLossBuffer.address)).to.equal(FIRST_LOSS_BUFFER)
    expect(
      await ctx.usdc.allowance(ctx.firstLossBuffer.address, await ctx.incidentManager.getAddress()),
    ).to.be.greaterThanOrEqual(FIRST_LOSS_BUFFER)
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(incident1)
    expect(await ctx.usdc.balanceOf(ctx.firstLossBuffer.address)).to.equal(0n)
    expect(
      await ctx.usdc.allowance(ctx.firstLossBuffer.address, await ctx.incidentManager.getAddress()),
    ).to.be.greaterThanOrEqual(FIRST_LOSS_BUFFER)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(0n)
    expect(await ctx.usdc.balanceOf(ctx.payout_receiver.address)).to.equal(FIRST_LOSS_BUFFER)
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(DEPOSIT)
    expect(await ctx.incidentManager.activeIncidentCount(1)).to.equal(0n)

    // Nevertheless, the historical period-start checkpoint remains 10,000.
    // commitAllocation reads that stale value instead of live post-payout
    // collateral, so period-2 cover settles, charges a premium, and mints an
    // NFT even though the vault already holds no active assets.
    expect(await ctx.firelight_vault.totalAssetsAt(period2Start)).to.equal(DEPOSIT)
    const premiumBefore = await ctx.usdc.balanceOf(ctx.rescuer.address)
    await settlePeriod(ctx, 1, NEW_COVER)
    expect(await ctx.coverNFT.ownerOf(1)).to.equal(ctx.buyer.address)
    expect(await ctx.usdc.balanceOf(ctx.rescuer.address)).to.be.greaterThan(premiumBefore)
    expect((await ctx.allocator.getCoverOrder(1)).status).to.equal(1n)
    expect(await ctx.firelight_vault.totalAssets()).to.equal(0n)

    // A valid period-2 claim against the paid and MATCHED cover closes with a
    // zero vault payout because the collateral was double-underwritten.
    const capture2 = await time.latest()
    const incident2 = await openSubmittedIncident(ctx, capture2, 1, NEW_COVER, 'period-2')
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(incident2)

    const [secondIncident] = await ctx.incidentManager.getIncident(incident2)
    expect(secondIncident.vaultPaidAmount).to.equal(0n)
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(DEPOSIT)
  })
})
```

Run:

```bash
npx hardhat test test/research/overlappingPeriodCollateral.test.js
```

Observed: 1 passing (979ms). Full repository suite: 455 passing (10s).
