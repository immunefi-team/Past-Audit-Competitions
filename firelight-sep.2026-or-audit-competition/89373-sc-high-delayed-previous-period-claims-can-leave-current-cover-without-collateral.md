# 89373 sc high delayed previous period claims can leave current cover without collateral

**Submitted on Aug 18th 2026 at 20:54:40 UTC by @Paludo0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89373
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Firelight can activate cover for a new period while the same collateral remains payable to a claim from the previous period. Capacity counts vault assets recorded at the new period start and the First Loss Buffer balance read at commit. It does not reserve the liability that remains payable from the prior period. A valid delayed claim can consume that collateral after the new cover becomes active. The new buyer has paid premium and holds valid cover, but the protocol can lack funds for a later valid claim.

## Vulnerability Details

Firelight permits an incident from the previous period to be reported and paid during the current period. CoverOrderAllocator does not include that continuing liability when it calculates current period capacity.

### Step by step vulnerability explanation

1. `IncidentManager::createIncident()` accepts a capture timestamp while its period remains inside the payout window:

```solidity
uint256 period = $.vault.periodAtTimestamp(captureTimestamp);
if (_isPeriodExpired(period) || captureTimestamp > block.timestamp) {
    revert InvalidCaptureTimestamp(captureTimestamp, period, $.vault.currentPeriod());
}
```

An event from period P can therefore be reported during period P plus one. The report may arrive after cover for P plus one was committed and settled.

2. `CoverOrderAllocator::_computeAvailableCapacity()` values the vault only at the start of the current period:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;
```

This value includes collateral that may still pay a period P claim. No value represents prior period cover or its remaining payout exposure.

3. `CoverOrderAllocator::commitAllocation()` stores capacity after checking only that calculated maximum:

```solidity
uint256 maxCapacity = _computeAvailableCapacity($, period);
if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
if (totalAllocated > matchingCapacity) revert TotalAllocationOverflow(totalAllocated, matchingCapacity);
```

The check passes even when period P cover can still consume the same collateral. It therefore approves two valid obligations against one balance.

4. `CoverOrderAllocator::_settleCoverOrder()` uses the stored commitment, collects premium, and mints the receipt NFT:

```solidity
commit.totalSettledCover += allocatedCover;
if (commit.totalSettledCover > commit.totalDeclaredAllocated)
    revert TotalSettledOverflow(commit.totalSettledCover, commit.totalDeclaredAllocated);

IERC20(order.premiumToken).safeTransferFrom(order.buyer, $.premiumCollector, finalPremiumNative);
$.coverNFT.safeMint(order.buyer, orderId);
```

Settlement does not reserve previous period liability. Once settlement starts, the commitment cannot be replaced or cancelled.

5. `FirelightVault::payout()` allows the old claim during the next period and removes assets from the shared vault:

```solidity
if (!_isPeriodInPayoutWindow(capturePeriod, _currentPeriod)) {
    revert InvalidCapturePeriod();
}

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

A valid period P claim can consume the assets that supported capacity for P plus one. Current period cover remains active because payout does not update its stored commitment.

6. A later claim for P plus one is limited by the remaining assets. `IncidentManager::approveCurrentAssessment()` closes the incident even when the vault returns zero:

```solidity
uint256 vaultPaidAmount = _executePayout(
    $, incidentId, assessmentRoundId, incident.period, incident.captureTimestamp, totalAssessmentLoss
);

incident.status = IncidentStatus.CLOSED;
emit IncidentClosed(incidentId, vaultPaidAmount, totalAssessmentLoss);
```

Waiting until period P plus two does not restore the missing collateral. A claim captured during period P plus one remains inside its payout window during P plus two, but still depends on the same depleted shared balance. If approval happens during P plus one, IncidentManager closes the incident even when vault payout is zero and stores no unpaid balance for retry.

No role must submit an invalid loss. Each assessment can stay within its own settled allocation and match its own cover period.

### Attack path

1. A depositor supplies vault collateral before period P. The First Loss Buffer is funded and has granted its normal allowance.
2. The allocator settles valid cover for period P.
3. During P, the curator creates a valid order for period P plus one.
4. Period P plus one starts without a reported incident from P.
5. The allocator commits the opening capacity and settles the new order for P plus one.
6. A valid event from P is reported during its allowed payout window.
7. The approved period P claim consumes the First Loss Buffer and the vault assets.
8. Only after the new order is settled, another valid event occurs during P plus one.
9. The second event is approved against the settled order for P plus one.
10. The protocol pays only the remaining collateral, possibly zero, and closes the incident.

## Impact Details

* Affected asset: `contracts/core/CoverOrderAllocator.sol` is the primary root. `FirelightVault.sol` and `IncidentManager.sol` execute the resulting underpayment.
* Severity: Critical protocol insolvency. Valid current period cover can remain active after its supporting collateral pays a valid previous period claim.
* Funds at risk: the current period settled cover left without backing.
* Who loses: the current buyer pays premium and receives the cover NFT, but the payout receiver can receive less than the approved loss.
* Recovery: IncidentManager closes the incident after the available payment. The unpaid amount has no normal retry path.
* Trigger: no exact timestamp, block ordering, invalid assessment, or malicious role action is required.
* Known issue difference: the public issue says aggregate claims are not capped against total cover sold. Here total losses do not exceed cover sold. Each claim stays within a different settled order and its correct period. Current capacity instead reuses collateral already supporting prior cover.
* Audit difference: the published Macro audit discusses leverage and settlement availability. It does not describe reuse of collateral across adjacent payout windows.

## Recommended Fix

Reserve the maximum remaining liability from the previous period before calculating capacity for the current period. A conservative implementation can start from prior settled cover:

```solidity
uint256 priorLiability;
if (period > 0) {
    priorLiability = $.allocationCommitments[period - 1].totalSettledCover;
}

if (priorLiability >= availableCollateralCanonical) return 0;
availableCollateralCanonical -= priorLiability;
```

A complete implementation should reduce this reserve after payouts and release it when the prior payout window closes. The reserve must exist before cover settlement because an old incident may be reported later.

## Proof of Concept

## PoC Explanation

`deployCrossPeriodCapacity()` imports the original `deployVault` fixture. It deploys the real vault, allocator, CoverNFT, and IncidentManager contracts behind their normal proxies. The test uses the developer MockPriceFeed only to keep the vault asset price at one dollar.

The setup deposits 1,000,000 asset units into the vault. It also funds the First Loss Buffer with 100,000 units and grants IncidentManager its normal allowance.

`fundAndCreateOrder()` creates one valid order for each adjacent period and gives each buyer enough premium approval.

`commitAndSettle()` uses the production Merkle proof format. It commits capacity, collects premium, mints the real CoverNFT, and returns the settlement block timestamp.

`createAndApproveIncident()` follows the normal incident flow. It creates, confirms, assesses, submits, and approves one claim within its settled allocation.

`crossPeriodCapacity.poc.js::activates current period cover that the vault cannot pay()` advances one hour into each period. It settles the current cover before reporting the delayed claim. After that payment, it advances another 60 seconds before recording the current event. The test asserts that this capture timestamp is after settlement and belongs to the current period.

Run:

```
npx hardhat test test/FirelightVault/crossPeriodCapacity.poc.js
```

Actual output:

```
  Cross period capacity ignores a payable previous period claim
current cover settlement timestamp: 1788279385
current incident capture timestamp: 1788279450
previous period valid claim paid: 1100000.0
current period capacity committed: 1100000.0
current period cover settled: 1100000.0
first loss buffer after previous period claim: 0.0
vault assets after previous period claim: 0.0
current period valid claim paid: 0.0
current period incident status: 4
    ✔ activates current period cover that the vault cannot pay (169ms)

  1 passing (171ms)
```

```
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");

const { deployVault } = require("../setup/fixtures.js");

const BPS = 10_000n;
const YEAR = 365n * 24n * 3600n;
const ASSET_DECIMALS = 6n;
const CANONICAL_DECIMALS = 18n;
const SCALE_TO_CANONICAL = 10n ** (CANONICAL_DECIMALS - ASSET_DECIMALS);
const ONE_ASSET = 10n ** ASSET_DECIMALS;

const proratedPremium = (coverAmount, annualRateBps, periodDuration) => {
  const numerator = coverAmount * BigInt(annualRateBps) * BigInt(periodDuration);
  const denominator = BPS * YEAR;
  return (numerator + denominator - 1n) / denominator;
};

const premiumToNative = (canonicalAmount) => (canonicalAmount + SCALE_TO_CANONICAL - 1n) / SCALE_TO_CANONICAL;

const marketIdFor = (chainId, protocol, market) =>
  ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(["uint64", "string", "bytes32"], [chainId, protocol, market]),
  );

async function deployCrossPeriodCapacity() {
  const vaultContext = await deployVault({
    decimals: Number(ASSET_DECIMALS),
    initial_deposit_limit: 2_000_000n * ONE_ASSET,
    period_configuration_duration: 604800,
  });

  const signers = await ethers.getSigners();
  const deployer = signers[0];
  const allocatorAdmin = signers[1];
  const allocatorCurator = signers[2];
  const allocatorRole = signers[3];
  const allocatorConfigAdmin = signers[4];
  const coverAdmin = signers[5];
  const incidentAdmin = signers[6];
  const incidentCurator = signers[7];
  const assessmentApprover = signers[8];
  const assessmentRejecter = signers[9];
  const incidentInvalidator = signers[10];
  const payoutAdmin = signers[11];
  const priceFeedAdmin = signers[12];
  const payoutReceiver = signers[13];
  const premiumCollector = signers[14];
  const firstLossBuffer = signers[15];
  const depositor = signers[16];
  const previousPeriodBuyer = signers[17];
  const currentPeriodBuyer = signers[18];

  const { firelight_vault: vault, token_contract: asset } = vaultContext;

  const MockPriceFeed = await ethers.getContractFactory("MockPriceFeed");
  const priceFeed = await MockPriceFeed.connect(deployer).deploy(8, 10n ** 8n);

  const CoverNFTFactory = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ["Firelight Cover", "FLCOVER", "", coverAdmin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" },
  );

  const market = ethers.encodeBytes32String("market-A");
  const marketId = marketIdFor(1, "Morpho", market);

  const CoverOrderAllocatorFactory = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocatorFactory,
    [
      {
        vault: vault.target,
        premiumCollector: premiumCollector.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 3600,
        premiumTokens: [asset.target],
        admin: allocatorAdmin.address,
        adminRole: allocatorAdmin.address,
        curatorRole: allocatorCurator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: allocatorConfigAdmin.address,
        initialProtocolConcentrations: [{ chainId: 1, protocol: "Morpho", maxProtocolConcentrationBps: 10_000 }],
        newMarkets: [{ chainId: 1, protocol: "Morpho", market }],
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

  await coverNFT.connect(coverAdmin).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  const IncidentManagerFactory = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManagerFactory,
    [
      incidentAdmin.address,
      incidentCurator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      incidentAdmin.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      payoutReceiver.address,
      allocator.target,
      priceFeed.target,
      3600,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), incidentManager.target);
  await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), incidentManager.target);
  await vault.connect(vaultContext.payout_allowlister).addToPayoutAllowlist(payoutReceiver.address);

  return {
    ...vaultContext,
    vault,
    asset,
    coverNFT,
    allocator,
    allocatorCurator,
    allocatorRole,
    incidentManager,
    incidentCurator,
    assessmentApprover,
    payoutReceiver,
    firstLossBuffer,
    priceFeed,
    depositor,
    previousPeriodBuyer,
    currentPeriodBuyer,
    marketId,
    periodDuration: 604800n,
  };
}

async function fundAndCreateOrder(context, buyer, coverAmount, annualRateBps) {
  const premiumCanonical = proratedPremium(coverAmount, annualRateBps, context.periodDuration);
  const premiumNative = premiumToNative(premiumCanonical);

  await context.asset.mintTo(buyer.address, premiumNative);
  await context.asset.connect(buyer).approve(context.allocator.target, premiumNative);

  await context.allocator
    .connect(context.allocatorCurator)
    .createCoverOrder(
      buyer.address,
      context.payoutReceiver.address,
      "covered beneficiary",
      context.asset.target,
      [{ marketId: context.marketId, coverRateAnnual: annualRateBps, coverAmount }],
      0,
    );
}

async function commitAndSettle(context, orderId, period, coverAmount, roundId) {
  const tree = StandardMerkleTree.of(
    [[orderId, [[context.marketId, coverAmount]]]],
    ["uint256", "(bytes32,uint256)[]"],
  );

  const block = await ethers.provider.getBlock("latest");
  await context.priceFeed.setLatestRoundData(roundId, 10n ** 8n, block.timestamp, block.timestamp, roundId);

  await context.allocator.connect(context.allocatorRole).commitAllocation(period, tree.root, coverAmount, coverAmount);
  const settlementTx = await context.allocator
    .connect(context.allocatorRole)
    .settleCoverOrder(orderId, [{ marketId: context.marketId, allocatedCover: coverAmount }], tree.getProof(0));

  const settlementReceipt = await settlementTx.wait();
  const settlementBlock = await ethers.provider.getBlock(settlementReceipt.blockNumber);
  return BigInt(settlementBlock.timestamp);
}

async function createAndApproveIncident(context, incidentId, orderId, captureTimestamp, coverAmount, reference) {
  await context.incidentManager
    .connect(context.incidentCurator)
    .createIncident(captureTimestamp, "valid covered incident", ethers.id(reference));
  await context.incidentManager.connect(context.incidentCurator).confirmIncident(incidentId, "ipfs://assessment");
  await context.incidentManager
    .connect(context.incidentCurator)
    .addAssessmentLosses(incidentId, [{ coverTokenId: orderId, marketId: context.marketId, amount: coverAmount }]);
  await context.incidentManager.connect(context.incidentCurator).submitCurrentAssessment(incidentId);
  await context.incidentManager.connect(context.assessmentApprover).approveCurrentAssessment(incidentId);
}

describe("Cross period capacity ignores a payable previous period claim", function () {
  it("activates current period cover that the vault cannot pay", async function () {
    const context = await loadFixture(deployCrossPeriodCapacity);
    const vaultAssetsNative = 1_000_000n * ONE_ASSET;
    const firstLossBufferNative = 100_000n * ONE_ASSET;
    const coverAmountNative = vaultAssetsNative + firstLossBufferNative;
    const coverAmount = coverAmountNative * SCALE_TO_CANONICAL;
    const annualRateBps = 500;

    await context.utils.mintAndApprove(vaultAssetsNative, context.depositor);
    await context.vault.connect(context.depositor).deposit(vaultAssetsNative, context.depositor.address);

    await context.asset.mintTo(context.firstLossBuffer.address, firstLossBufferNative);
    await context.asset.connect(context.firstLossBuffer).approve(context.incidentManager.target, firstLossBufferNative);

    const initialPeriod = await context.vault.currentPeriod();
    const previousPeriod = initialPeriod + 1n;
    const currentPeriod = previousPeriod + 1n;

    await fundAndCreateOrder(context, context.previousPeriodBuyer, coverAmount, annualRateBps);

    await time.increaseTo((await context.vault.currentPeriodEnd()) + 3600n);
    expect(await context.vault.currentPeriod()).to.equal(previousPeriod);
    const previousCaptureTimestamp = BigInt(await time.latest());

    await commitAndSettle(context, 0n, previousPeriod, coverAmount, 2);
    await fundAndCreateOrder(context, context.currentPeriodBuyer, coverAmount, annualRateBps);

    await time.increaseTo((await context.vault.currentPeriodEnd()) + 3600n);
    expect(await context.vault.currentPeriod()).to.equal(currentPeriod);

    const currentSettlementTimestamp = await commitAndSettle(context, 1n, currentPeriod, coverAmount, 3);

    const currentCommitment = await context.allocator.getAllocationCommitment(currentPeriod);
    expect(currentCommitment.totalAvailableCapacity).to.equal(coverAmount);
    expect(currentCommitment.totalSettledCover).to.equal(coverAmount);
    expect(await context.coverNFT.ownerOf(1n)).to.equal(context.currentPeriodBuyer.address);

    const receiverBeforePreviousClaim = await context.asset.balanceOf(context.payoutReceiver.address);
    await createAndApproveIncident(
      context,
      1n,
      0n,
      previousCaptureTimestamp,
      coverAmount,
      "previous period valid incident",
    );
    const receiverAfterPreviousClaim = await context.asset.balanceOf(context.payoutReceiver.address);
    const previousClaimPaid = receiverAfterPreviousClaim - receiverBeforePreviousClaim;

    expect(previousClaimPaid).to.equal(coverAmountNative);
    expect(await context.vault.totalAssets()).to.equal(0n);
    expect(await context.asset.balanceOf(context.firstLossBuffer.address)).to.equal(0n);

    await time.increase(60);
    const currentCaptureTimestamp = BigInt(await time.latest());
    expect(currentCaptureTimestamp).to.be.greaterThan(currentSettlementTimestamp);
    expect(await context.vault.periodAtTimestamp(currentCaptureTimestamp)).to.equal(currentPeriod);

    await createAndApproveIncident(
      context,
      2n,
      1n,
      currentCaptureTimestamp,
      coverAmount,
      "current period valid incident",
    );

    const receiverAfterCurrentClaim = await context.asset.balanceOf(context.payoutReceiver.address);
    const currentClaimPaid = receiverAfterCurrentClaim - receiverAfterPreviousClaim;
    const [currentIncident] = await context.incidentManager.getIncident(2n);

    expect(currentClaimPaid).to.equal(0n);
    expect(currentIncident.vaultPaidAmount).to.equal(0n);
    expect(currentIncident.status).to.equal(4);

    console.log("current cover settlement timestamp:", currentSettlementTimestamp.toString());
    console.log("current incident capture timestamp:", currentCaptureTimestamp.toString());
    console.log("previous period valid claim paid:", ethers.formatUnits(previousClaimPaid, Number(ASSET_DECIMALS)));
    console.log("current period capacity committed:", ethers.formatUnits(coverAmount, Number(CANONICAL_DECIMALS)));
    console.log(
      "current period cover settled:",
      ethers.formatUnits(currentCommitment.totalSettledCover, Number(CANONICAL_DECIMALS)),
    );
    console.log(
      "first loss buffer after previous period claim:",
      ethers.formatUnits(await context.asset.balanceOf(context.firstLossBuffer.address), Number(ASSET_DECIMALS)),
    );
    console.log(
      "vault assets after previous period claim:",
      ethers.formatUnits(await context.vault.totalAssets(), Number(ASSET_DECIMALS)),
    );
    console.log("current period valid claim paid:", ethers.formatUnits(currentClaimPaid, Number(ASSET_DECIMALS)));
    console.log("current period incident status:", currentIncident.status.toString());
  });
});

```
