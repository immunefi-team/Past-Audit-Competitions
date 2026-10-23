# 90093 sc medium a boundary withdrawal changes the period start exposure lookup after capacity is committed causing an approved claim to be underpaid up to zero

**Submitted on Aug 22nd 2026 at 22:59:18 UTC by @v4rvl for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90093
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

The defect exists inside `FirelightVault` independently of allocation logic. An ordinary withdrawal at `P.start` records 100 assets in `withdrawAssets[P + 1]`, a bucket that `payout` expressly includes while the withdrawal remains exposed. The same withdrawal also writes a zero checkpoint at `P.start`, so `assetsAtCapturePeriod` becomes zero and the other term of the function's `min` forces an approved 100-unit payout to zero. Changing only the withdrawal timestamp from `P.start` to `P.start + 1` changes the approved payout from 0 to 100.

In the production incident lifecycle, the order is `MATCHED`, the buyer pays a nonzero premium, and the approved 100-unit loss transfers nothing to the payout receiver. The LP later claims the entire 100-unit withdrawal that the payout had treated as unusable.

`CoverOrderAllocator` demonstrates that paid cover can reach this broken vault state. It reads `totalAssetsAt(P.start)` as 100 and validly commits 200 of matching capacity from that answer. Later in the same block, the ordinary withdrawal changes the identical historical lookup to 0 while the accepted commitment remains usable. At the one-dollar price used by the production pricepath in the proof, the unpaid amount is $100.

The impact pillar uses the first write at the boundary key. An exhaustive ordering property separately proves the same result when the withdrawal overwrites an existing `P.start` checkpoint.

The root cause is the mutable period-start exposure lookup: a later transaction with the same block timestamp changes the answer used by payout after the protocol has already consumed a different answer for the same period.

The vault checkpoints balances under `block.timestamp` at `contracts/core/FirelightVault.sol:1051-1055`, and the payout cap reads a checkpoint at the period-start timestamp at `:911`. Those two keys are equal in the first block of a period, so a withdrawal there changes what the cap lookup returns for a reader that has already used it. The Root cause section shows the three lines that make that possible.

The official [Periods and Coverage Cycles](https://docs.firelight.finance/core-concepts/periods-and-coverage-cycles) documentation states:

> "Period settlement is deterministic with respect to the period start timestamp. Deployments and withdrawals submitted after a period has started do not affect the current period's settlement; they are picked up in the next period's capacity calculation. This makes each period's outcomes reproducible from its starting state."

The contract assigns the exact boundary timestamp to P: `periodAtTimestamp(P.start) == P`. In the executed commit-then-withdraw ordering, matching consumes P's starting value first, then the later P withdrawal rewrites that same starting value. The write affects P immediately instead of only the next period.

The separate [Deployments and Withdrawals](https://docs.firelight.finance/for-stakers/deployments-and-withdrawals) page states that "each period settles based on its starting state, so mid-period flows take effect in the next period." It also states that pending withdrawal capital remains backing coverage for the period in which unstaking began and that, for an incident attributed to that period, "the resulting slash still applies to the pending withdrawal." The withdrawal remains recorded as exposed exactly as documented, but the boundary checkpoint independently reduces the historical exposure cap to zero, preventing that exposed bucket from being used.

The program-linked [Claims Liquidation](https://docs.firelight.finance/protocol-architecture/claims-liquidation) page independently defines payout capacity from the First-Loss Buffer and the market value of staked assets. It says capacity is measured at settlement and that an approved loss within that capacity pays in full. In the boundary case, the vault still holds the withdrawal assets and `activePayableAmount` includes them. The payment shortfall comes from the rewritten historical cap, not from missing vault tokens, an exhausted withdrawal bucket, collateral-price decline, or liquidation slippage.

At `P.start`, the same withdrawal is simultaneously counted as still-exposed capital in `withdrawAssets[P + 1]` and allowed to rewrite `totalAssetsAt(P.start)` to zero, causing that exposed capital to become unusable for the approved payout.

The zero-payout endpoint is the maximal case, not a required empty-vault precondition. A separate production-vault control starts with two independent LPs and 200 total assets. When only one LP withdraws 100 at `P.start`, the vault remains funded, `activePayableAmount` remains 200, but `totalAssetsAt(P.start)` falls from 200 to 100 and the requested 200 payout transfers only 100. Moving that same 100 withdrawal to `P.start + 1` preserves the 200 historical cap and the same payout transfers 200. Therefore an ordinary LP does not need every existing holder to exit: for starting assets `S` and a boundary withdrawal `W`, the mutable lookup reduces the payout cap from `S` to `S - W`.

The canonical measurements on the competition pin are:

1. With 100 vault assets, `totalAssetsAt(P.start)` returns 100 and the allocator computes 200 of matching capacity.
2. The allocator accepts and records a 200-unit commitment from that value.
3. The order settles as `MATCHED` for period P and mints its CoverNFT.
4. The buyer's balance decreases by the order's exact nonzero premium.
5. A later withdrawal in the same block changes `totalAssetsAt(P.start)` to 0 while the 100 assets remain recorded in `withdrawAssets[P + 1]`.
6. After the production incident lifecycle approves a 100-unit loss, `vaultPaidAmount` is 0, the payout receiver's balance is unchanged, and the LP claims the recorded 100 withdrawal.
7. Moving only the withdrawal to `P.start + 1` leaves `totalAssetsAt(P.start)` at 100; the same approved claim pays 100 and leaves 0 in the withdrawal bucket.

The funded two-LP bound test separately measures `200 -> 100` rather than relying on the maximal `100 -> 0` case. It proves that the mechanism produces a proportional payout shortfall in a nonempty pooled vault.

Both production arms create the incident only after settlement, so the proof does not depend on cover becoming claimable after an incident's capture time.

## Who loses what

A curator creates a 100 USD cover order for period P while the vault holds 100 assets. The allocator commits it in the first block of P. The LP withdraws later in that same block.

| Actor                      | Boundary outcome | Control outcome | Evidence                                                                                          |
| -------------------------- | ---------------- | --------------- | ------------------------------------------------------------------------------------------------- |
| Configured payout receiver | **0 paid**       | **100 paid**    | `vaultPaidAmount == 0`; receiver balance unchanged across `approveCurrentAssessment`              |
| LP withdrawal              | **100 claimed**  | **0 remaining** | the boundary withdrawal escapes the haircut; in the control the payout consumes that bucket first |

The control arm is the same withdrawal one second later. The 100 does not vanish: it moves from the approved claim to the LP's withdrawal.

In both production-lifecycle arms, the buyer's balance decreases by the exact premium and the order becomes `MATCHED` for period P before the incident is created. The boundary-arm LP wallet is 0 immediately before `claimWithdraw` and 100 afterward, equal to the 100 funded before the deposit. The proof therefore measures an avoided haircut, not tokens in excess of the LP's principal.

The LP is a relative beneficiary: they claim 100 in the boundary arm, while the control payout consumes that withdrawal bucket. The proof shows the LP recovering only the principal they deposited, not receiving extra tokens. It does not show an attacker-controlled address gaining funds outside its withdrawal claim. What changes is where the covered loss lands: the approved payout receives zero, while the otherwise-payable withdrawal escapes the haircut.

The result does not assume that this LP owns the whole vault. In the two-LP test, one LP withdraws half of a 200-asset pool. The boundary arm pays 100 of the requested 200 and leaves 50 in that LP's withdrawal bucket; the one-second-later control pays 200 and leaves that bucket at zero. This is a funded-pool shortfall and avoided haircut, not an empty-vault rounding condition.

The failure is not a missed return. The protocol had already committed capacity for the cover, the order settles and charges a premium, and a subsequently approved 100-unit claim receives zero solely because the historical exposure value used for underwriting reads differently at payout. The corresponding 100 assets stay claimable by the LP instead of being available to satisfy the covered loss.

The demonstrated harm is an approved covered loss receiving zero payout after the protocol had already committed capacity and collected premium, while the otherwise-payable withdrawal escapes the haircut. That is damage to the protocol under the program's Medium griefing row: the approved assessment transfers zero to the configured incident payout receiver. This is not a claim about later manual distribution because no assets reach the receiver. The LP is better off than in the control through an avoided haircut, although the proof shows no unentitled token gain.

## Root cause

Three facts compose. Each is a line in the pinned source.

**The payout cap is a historical lookup at the period-start timestamp**, at `contracts/core/FirelightVault.sol:911` and `:926-927`:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The withdrawal bucket is inside `activePayableAmount`, so the payment is not blocked by the bucket. It is blocked by `assetsAtCapturePeriod`.

The zero-payout condition exists in `FirelightVault` independently of allocation logic: an exact-boundary withdrawal remains in the payout's included withdrawal bucket while simultaneously changing `assetsAtCapturePeriod` to zero. The allocator demonstrates that paid cover can reach this vault state; it does not create the payout-accounting discrepancy.

**Balance checkpoints are keyed by `block.timestamp`**, at `contracts/core/FirelightVault.sol:1051-1055`:

```solidity
uint48 ts = Time.timestamp();
_traceBalanceOf[owner].push(ts, balance);
_traceTotalSupply.push(ts, _totalSupply);

if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

**The lookup is inclusive, so a write AT the boundary key changes it.** `Checkpoints.upperLookupRecent` returns the value of the last checkpoint whose key is less than or equal to the search key, at `contracts/core/lib/Checkpoints.sol:219-223`:

```solidity
function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
    uint208 idx = self._trace.upperLookupRecent(key);
    return idx > 0 ? self._values[idx] : 0;
}
```

That inclusive lookup allows a later write at the boundary timestamp to change the value returned for `P.start`. Before the withdrawal, the newest checkpoint at or below `P.start` is the one written by an earlier deposit, and the lookup returns 100. The withdrawal then writes a checkpoint keyed exactly at `P.start` carrying the post-withdrawal total. It is now the newest checkpoint at or below `P.start`, so the same lookup returns 0.

The impact pillar's write is an append. The ordering property also executes the overwrite arm. It calls `checkpointTotalAssets` before the withdrawal at the same `P.start` timestamp, then proves the later withdrawal changes the stored value from 100 to 0 while the earlier commitment remains settleable. `Trace256.push` implements that behavior directly at `contracts/core/lib/Checkpoints.sol:203-212`: when the new key equals the last key, it replaces the last stored value.

The allocator is one such reader. `contracts/core/CoverOrderAllocator.sol:710-711` sizes capacity from the same lookup:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
```

It validates and stores a matching capacity, but storing that figure does not freeze the vault value it was derived from.

The issue is not merely that a checkpoint at `P.start` reflects a transaction executed during the opening block. The allocator is allowed to consume `totalAssetsAt(P.start)` earlier in that same block and commit cover from the returned value. The protocol therefore treats the lookup as authoritative before the block is complete, while later transactions carrying the same timestamp can still change what the identical lookup returns. That is a time-of-check/time-of-use inconsistency between underwriting and payout. Block-timestamp granularity on its own would be unremarkable; what makes it a defect is committing underwriting state from a value that later transactions in the same block can still change.

This report does not argue that checkpoint history should be immutable in general, nor that the two terms of the `min` should be reconciled. It shows that one value is consumed by two readers at different times and can change between them, inside a single block.

This conflicts with the product's stated period semantics, not only surprising checkpoint behavior. The official period documentation requires deterministic settlement from a reproducible starting state and defers later P flows to the next period's calculation. Here, a later transaction after the P commit changes the value both the allocator and payout treat as P's starting state. The source and PoC therefore establish `X != Y`: P starts from 100 when capacity is committed, but the same P starting-state query later returns 0 and caps the approved payout at 0.

## Precondition boundary

The single parameter is the withdrawal's block timestamp relative to the period start. The impact and control rows are executed by the production lifecycle, and the ordering property executes all six placements of `commit`, `withdraw`, and `checkpoint`:

| Case                              | Withdrawal timestamp | Ordering                                      | `totalAssetsAt(P.start)` | Payout on the approved 100 claim | Defect triggered |
| --------------------------------- | -------------------- | --------------------------------------------- | -----------------------: | -------------------------------: | ---------------- |
| full-lifecycle one-second control | `P.start + 1`        | control                                       |                  **100** |            **100, paid in full** | **no**           |
| exact-boundary payout             | `P.start`            | withdrawal alone                              |                    **0** |                            **0** | yes              |
| commit then withdraw              | `P.start`            | commit first, withdraw second, same block     |                    **0** |                            **0** | yes              |
| reverse ordering                  | `P.start`            | withdraw first, commit second, same block     |                    **0** |      not reached; commit reverts | **no**           |
| existing-key overwrite            | `P.start`            | checkpoint, commit, then withdraw, same block |                    **0** |       settlement remains enabled | yes              |

One second past the boundary the attack fails outright and the claim is paid in full. That is the measured point at which the claim dies, and it is what attributes the outcome to the timestamp collision rather than to the withdrawal.

The commit-then-withdraw case carries the impact because it proves the two readers disagree within a single block: it asserts both receipts share a block number and that the commitment receipt index is lower than the withdrawal's, then asserts the committed capacity remains recorded while the period-start lookup used to validate it now reads zero.

The property test uses distinct actors and EIP-1559 priority fees, not RPC insertion order alone. In every permutation where commitment precedes withdrawal, the commitment succeeds and the order settles after the basis falls to zero. In every reverse permutation, commitment has receipt status 0 and no root is stored. A checkpoint placed before the withdrawal proves the same-key overwrite case.

Range covered: two timestamp positions across five tests, including six boundary-block action permutations. The production-lifecycle arms use a 100-asset vault, 100 USD cover order, and 100-unit approved claim. The separate funded-pool bound test starts with 200 assets across two LPs, withdraws 100, and requests a 200-unit payout, measuring a 100-unit shortfall. I did not sweep arbitrary deposit sizes, cover sizes, or period durations.

## Reachability on the deployment chain

Final verdict: **probabilistically reachable**. The transaction that changes the lookup is the ordinary `withdraw` entry point. Exploitation additionally requires the allocator commitment to have consumed the period-start value earlier in the same boundary block. The role-supplied amount matches the pinned capacity formula at execution: the first-loss-buffer balance is 0, the period-start vault value is 100, `effectiveLeverage` is 24,000, `minCAR` is 12,000, and `divergenceToleranceBps` is 0, producing a capacity of 200. The submitted `totalAllocated` is 100, equal to the single order and its one-leaf allocation, and the assessment loss is the same 100 allocated to that order.

This path does not require a privileged actor to submit an invalid value. The allocator's 200 matching capacity equals the implemented maximum when committed, and the declared allocation equals the order's 100-unit allocation. An ordinary LP then calls `withdraw`, which changes the same historical lookup to zero. The proof confirms that recommitting the same 200 capacity after that withdrawal reverts with `MatchingCapacityOverflow(200e18, 0)`, while the earlier valid commitment remains usable. The curator and approver later process one settled cover and one loss within its 100-unit allocation; neither supplies the state change that creates the discrepancy.

The boundary is derived from production state, not chosen by the withdrawing user. `FirelightVault.initialize` stores its initialization block timestamp as the first epoch at `FirelightVault.sol:235`. `currentPeriodStart` and `periodAtTimestamp` derive every later boundary by integer division from that epoch. Every permitted duration is a multiple of 86,400 seconds, and later configuration epochs must themselves align to the existing schedule at `FirelightVault.sol:1084-1108`. `initializeV2` does not change this state. The residue therefore comes from a real initialization block and cannot be shifted arbitrarily by a later duration update.

The intended chain is Flare: the system depends on Flare FTSO v2, and the repository includes a Flare FTSO adapter. In the official [`flare-foundation/go-flare`](https://github.com/flare-foundation/go-flare/tree/ac6a9e7706cd4d271b31af2e161e1576217f0d8b) source at commit `ac6a9e7706cd4d271b31af2e161e1576217f0d8b`, Granite is active on Flare from 2026-07-14. The EVM header carries millisecond time, requires `header.Time == TimeMilliseconds / 1000`, takes the next timestamp from current wall-clock time, and enforces a dynamic minimum block delay. A block produced during the boundary second therefore receives exactly the required integer `block.timestamp`; no timestamp congruence rule forbids it.

The competition pack does not provide a resolved production address, initialization block, stored epoch, or active duration, and its rules prohibit testing deployed mainnet or public testnet code. I therefore do not claim a measured historical frequency or a known upcoming boundary. A block timestamp equal to the boundary is permitted by the target chain's timestamp rules, but the supplied materials do not contain enough production telemetry to measure how often one occurs. Likelihood therefore remains Low.

Recovery is discretionary rather than automatic. `CONFIG_ADMIN_ROLE` can cancel a commitment before any settlement, but the withdrawal does not invalidate it. Recommitting the same 200 revalidates fresh capacity and reverts, as the PoC proves. Once the first order settles, both replacement and cancellation revert with `SettlementsAlreadyStarted`. Avoiding the measured outcome therefore requires an operator to detect the boundary rewrite and cancel before the first settlement; no contract transition performs that response.

## Production path

The proof runs the production `FirelightVault`, `CoverOrderAllocator`, `IncidentManager`, and `CoverNFT` through the repository's own `deployVault` fixture. None of those four contracts is mocked or replaced. The impact-carrying arm and its control do not call `payout` directly: they reach it through `createIncident`, `confirmIncident`, `addAssessmentLosses`, `submitCurrentAssessment`, and `approveCurrentAssessment`, so the measured zero payment is the production path's own result.

The fixture uses a mintable test asset for balances and `MockAggregatorV3` to supply a fixed $1 input. Neither implements the checkpoint lookup, commitment validation, order settlement, incident assessment, or payout. The attack and control use the same token and price input, so no mock decides the zero-versus-100 payout result.

## What the surrounding comments say

The `payout` docblock covering `contracts/core/FirelightVault.sol:909-912` carries three sentences that a reviewer will reach for. Each is answered here.

> "This is an exposure cap, not a segregated asset bucket."

Agreed, and the finding does not need it to be a bucket. The claim is about the cap's VALUE changing, not about which assets back it. A cap that limits what a payout may draw is the thing at issue: `contracts/core/FirelightVault.sol:927` takes `Math.min(assetsAtCapturePeriod, activePayableAmount)`, so when the capture-period figure becomes zero the payment becomes zero regardless of how the remaining assets are held.

> "The amount is capped by the active assets at the capture-period start (an exposure cap, not a segregated bucket) and by currently payable assets, and is drawn proportionally from active assets and still-exposed withdrawals."

This describes what the cap means. It does not say when the capture-period start value may be written. The defect is that the value read at `:911` is not fixed at the period start: it is a checkpoint lookup that a later transaction in the same block can change, after `CoverOrderAllocator.sol:710-711` has already sized capacity from it.

> "The vault applies only these per-call limits; ensuring aggregate payouts stay within underwritten exposure is the responsibility of the curator and assessor roles."

This assigns aggregate exposure across multiple payouts to the roles, and this report makes no aggregate claim. One incident, one payout, one call. The proof shows a single approved claim paying zero, so no accumulation across calls is involved and nothing here is asked of the curator or assessor.

`contracts/core/FirelightVault.sol:914-915`:

> "Withdrawals requested during the capture period are assigned to capturePeriod + 1 and remain payable while payout is allowed."

The proof executes the state this sentence describes. The withdrawal is requested during P, it lands in `withdrawAssets[P + 1]`, and `:916` includes that bucket in `activePayableAmount`. The other binding term of the same `min` subsequently reads as zero after the withdrawal writes the boundary checkpoint, so the payout is still zero.

No comment at any cited line records a decision to allow a boundary-timestamp checkpoint write to change a value that underwriting has already consumed.

The docblock at `contracts/core/CoverOrderAllocator.sol:388-392` states that a commitment stores the exact capacity used by the matcher rather than recomputing capacity at settlement, because the off-chain tree is derived from that committed input.

The finding does not require settlement to replace that committed value. It shows that the historical input used to validate the committed value can change after commitment, while `payout` later reads the changed input as the cap on the same period's approved loss. Preserving the matcher's exact input and preserving one answer to the period-start exposure lookup are different requirements.

## Distinction from prior findings

Macro I-9 is the closest prior item. It reported that an allocation commitment did not bind the capacity used by the matcher. The pin contains its fix: `commitAllocation` stores `matchingCapacity` and rejects it when it exceeds the recomputed capacity. This report exercises that bound successfully, then shows a later ordinary transaction in the same block changes the historical input that the recomputation consumed. The stored capacity remains 200 while the same formula now returns zero while the earlier commitment remains usable. This leaves the capacity accepted by the I-9 remediation dependent on a period-start input that can still change later in the boundary block; it is not the original missing-capacity field. The vector persists after the `matchingCapacity` field and its validation have both been added.

The disclosed reward-forwarding checkpoint item is also distinct. It concerns a value that was never written, while this report concerns a value changing after a reader has already used it. Searches across the supplied audit corpus, known issues, commit history, and tests found no disclosed item about a same-timestamp overwrite, an inclusive `upperLookupRecent` boundary, or a first-block withdrawal changing a period-start payout cap.

## Proof of Concept

## Proof of concept

Save as `test/period-start-exposure-lookup.test.js` in the competition repository at pin `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, then run `npx hardhat test test/period-start-exposure-lookup.test.js`.

```javascript
const { expect } = require("chai");
const { ethers, network, upgrades } = require("hardhat");
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");

const { deployVault, computeMarketId } = require("./setup/fixtures.js");

const DEPOSIT = ethers.parseUnits("100", 6);
const COVER = ethers.parseUnits("100", 18);
const MATCHING_CAPACITY = ethers.parseUnits("200", 18);
const MARKET = ethers.encodeBytes32String("market-A");
const MARKET_ID = computeMarketId(1, "Morpho", MARKET);

async function fundedVault() {
  const ctx = await deployVault({
    decimals: 6,
    initial_deposit_limit: ethers.parseUnits("1000000", 6),
    period_configuration_duration: 24 * 60 * 60,
  });

  const lp = ctx.users[0];
  await ctx.utils.mintAndApprove(DEPOSIT, lp);
  expect(await ctx.token_contract.balanceOf(lp.address)).to.equal(DEPOSIT);
  await ctx.firelight_vault.connect(lp).deposit(DEPOSIT, lp.address);
  expect(await ctx.token_contract.balanceOf(lp.address)).to.equal(0n);

  return { ...ctx, lp };
}

async function fundedVaultWithTwoLps() {
  const ctx = await fundedVault();
  const secondLp = ctx.users[1];

  await ctx.utils.mintAndApprove(DEPOSIT, secondLp);
  await ctx.firelight_vault.connect(secondLp).deposit(DEPOSIT, secondLp.address);

  expect(await ctx.firelight_vault.totalAssets()).to.equal(2n * DEPOSIT);
  return { ...ctx, secondLp };
}

async function fundedVaultAndAllocator() {
  const ctx = await fundedVault();
  const buyer = ctx.users[1];
  const firstLossBuffer = ctx.users[2];
  const signers = await ethers.getSigners();
  const assessmentApprover = signers[10];
  const assessmentRejecter = signers[11];
  const incidentInvalidator = signers[12];
  const payoutAdmin = signers[13];
  const priceFeedAdmin = signers[14];

  const PriceFeed = await ethers.getContractFactory("MockAggregatorV3");
  const priceFeed = await PriceFeed.deploy(18, ethers.parseUnits("1", 18));

  const CoverNFT = await ethers.getContractFactory("CoverNFT");
  const coverNFT = await upgrades.deployProxy(
    CoverNFT,
    [
      "Firelight Cover",
      "FLCOVER",
      "",
      ctx.deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
    ],
    { kind: "transparent" },
  );

  const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
  const allocator = await upgrades.deployProxy(
    Allocator,
    [
      {
        vault: await ctx.firelight_vault.getAddress(),
        premiumCollector: ctx.period_configuration_updater.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 60 * 60,
        premiumTokens: [await ctx.token_contract.getAddress()],
        admin: ctx.deployer.address,
        adminRole: ctx.rescuer.address,
        curatorRole: ctx.blocklister.address,
        allocatorRole: ctx.pauser.address,
        configAdminRole: ctx.limit_updater.address,
        initialProtocolConcentrations: [
          { chainId: 1, protocol: "Morpho", maxProtocolConcentrationBps: 10_000 },
        ],
        newMarkets: [{ chainId: 1, protocol: "Morpho", market: MARKET }],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: await ctx.token_contract.getAddress(),
          firstLossBuffer: firstLossBuffer.address,
          effectiveLeverage: 24_000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await coverNFT
    .connect(ctx.deployer)
    .grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress());

  const IncidentManager = await ethers.getContractFactory("IncidentManager");
  const incidentManager = await upgrades.deployProxy(
    IncidentManager,
    [
      ctx.deployer.address,
      ctx.blocklister.address,
      assessmentApprover.address,
      assessmentRejecter.address,
      incidentInvalidator.address,
      ctx.limit_updater.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      ctx.payout_receiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      7 * 24 * 60 * 60,
    ],
    { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
  );

  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), await incidentManager.getAddress());
  await ctx.firelight_vault
    .connect(ctx.deployer)
    .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), await incidentManager.getAddress());

  await ctx.token_contract.mintTo(buyer.address, ethers.parseUnits("100", 6));
  await ctx.token_contract.connect(buyer).approve(await allocator.getAddress(), ethers.MaxUint256);

  const markets = [{ marketId: MARKET_ID, coverRateAnnual: 500, coverAmount: COVER }];
  const orderId = await allocator
    .connect(ctx.blocklister)
    .createCoverOrder.staticCall(
      buyer.address,
      buyer.address,
      buyer.address,
      await ctx.token_contract.getAddress(),
      markets,
      0,
    );
  await allocator
    .connect(ctx.blocklister)
    .createCoverOrder(
      buyer.address,
      buyer.address,
      buyer.address,
      await ctx.token_contract.getAddress(),
      markets,
      0,
    );

  const tree = StandardMerkleTree.of(
    [[orderId, [[MARKET_ID, COVER]]]],
    ["uint256", "(bytes32,uint256)[]"],
  );

  return {
    ...ctx,
    allocator,
    assessmentApprover,
    buyer,
    coverNFT,
    incidentManager,
    orderId,
    tree,
  };
}

async function mineBoundaryActions(ctx, orderedActions) {
  const vault = ctx.firelight_vault;
  const nextStart = await vault.currentPeriodEnd();
  const commitmentPeriod = await vault.periodAtTimestamp(nextStart);
  const transactions = {};

  const send = {
    checkpoint: (priorityFee) =>
      vault.connect(ctx.deployer).checkpointTotalAssets({
        gasLimit: 5_000_000,
        maxFeePerGas: ethers.parseUnits("100", "gwei"),
        maxPriorityFeePerGas: priorityFee,
      }),
    commit: (priorityFee) =>
      ctx.allocator
        .connect(ctx.pauser)
        .commitAllocation(commitmentPeriod, ctx.tree.root, COVER, MATCHING_CAPACITY, {
          gasLimit: 5_000_000,
          maxFeePerGas: ethers.parseUnits("100", "gwei"),
          maxPriorityFeePerGas: priorityFee,
        }),
    withdraw: (priorityFee) =>
      vault.connect(ctx.lp).withdraw(DEPOSIT, ctx.lp.address, ctx.lp.address, {
        gasLimit: 5_000_000,
        maxFeePerGas: ethers.parseUnits("100", "gwei"),
        maxPriorityFeePerGas: priorityFee,
      }),
  };

  await network.provider.send("evm_setAutomine", [false]);
  try {
    await network.provider.send("evm_setNextBlockTimestamp", [Number(nextStart)]);

    for (const [index, action] of orderedActions.entries()) {
      // Distinct actors submit all actions. Priority fees, rather than RPC
      // insertion order alone, determine their order inside the boundary block.
      const priorityFee = ethers.parseUnits(String(orderedActions.length - index + 1), "gwei");
      transactions[action] = await send[action](priorityFee);
    }

    await network.provider.send("evm_mine");
  } finally {
    await network.provider.send("evm_setAutomine", [true]);
  }

  const receipts = {};
  for (const [action, transaction] of Object.entries(transactions)) {
    receipts[action] = await ethers.provider.getTransactionReceipt(transaction.hash);
  }

  return { commitmentPeriod, nextStart, receipts };
}

describe("period-start exposure lookup changes after capacity commitment", function () {
  it("a withdrawal at the exact period start removes the withdrawal from the payout cap", async function () {
    const ctx = await loadFixture(fundedVault);
    const vault = ctx.firelight_vault;
    const nextStart = await vault.currentPeriodEnd();

    expect(await vault.totalAssetsAt(nextStart)).to.equal(DEPOSIT);

    // The withdrawal is in the new period because periodAtTimestamp(nextStart)
    // resolves to that period. Its checkpoint is nevertheless stored at the
    // exact key later used by the period-start exposure lookup.
    await time.setNextBlockTimestamp(nextStart);
    await vault.connect(ctx.lp).withdraw(DEPOSIT, ctx.lp.address, ctx.lp.address);

    const capturePeriod = await vault.currentPeriod();
    expect(await vault.periodAtTimestamp(nextStart)).to.equal(capturePeriod);
    expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(DEPOSIT);
    expect(await vault.totalAssetsAt(nextStart)).to.equal(0n);

    await expect(
      vault
        .connect(ctx.payout_signer)
        .payout(ctx.payout_receiver.address, DEPOSIT, nextStart),
    )
      .to.emit(vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, DEPOSIT, 0n, nextStart);

    // The payout did not haircut the withdrawal bucket. The LP can claim the
    // full amount after the normal delay.
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n);
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n);
    const before = await ctx.token_contract.balanceOf(ctx.lp.address);
    expect(before).to.equal(0n);
    await vault.connect(ctx.lp).claimWithdraw(capturePeriod + 1n);
    expect(await ctx.token_contract.balanceOf(ctx.lp.address)).to.equal(DEPOSIT);
  });

  it("reduces payout in a funded two-LP vault without requiring the vault to become empty", async function () {
    async function run(withdrawalOffset) {
      const ctx = await loadFixture(fundedVaultWithTwoLps);
      const vault = ctx.firelight_vault;
      const nextStart = await vault.currentPeriodEnd();
      const requestedPayout = 2n * DEPOSIT;

      expect(await vault.totalAssetsAt(nextStart)).to.equal(requestedPayout);

      await time.setNextBlockTimestamp(nextStart + withdrawalOffset);
      await vault.connect(ctx.lp).withdraw(DEPOSIT, ctx.lp.address, ctx.lp.address);

      const capturePeriod = await vault.periodAtTimestamp(nextStart);
      const historicalCap = await vault.totalAssetsAt(nextStart);
      expect(await ctx.token_contract.balanceOf(await vault.getAddress())).to.equal(requestedPayout);
      expect(await vault.totalAssets()).to.equal(DEPOSIT);
      expect(await vault.withdrawAssets(capturePeriod + 1n)).to.equal(DEPOSIT);
      const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address);

      await vault
        .connect(ctx.payout_signer)
        .payout(ctx.payout_receiver.address, requestedPayout, nextStart);

      return {
        historicalCap,
        paid:
          (await ctx.token_contract.balanceOf(ctx.payout_receiver.address)) - receiverBefore,
        remainingWithdrawal: await vault.withdrawAssets(capturePeriod + 1n),
      };
    }

    const boundary = await run(0n);
    expect(boundary.historicalCap).to.equal(DEPOSIT);
    expect(boundary.paid).to.equal(DEPOSIT);
    expect(boundary.remainingWithdrawal).to.equal(DEPOSIT / 2n);

    const oneSecondLater = await run(1n);
    expect(oneSecondLater.historicalCap).to.equal(2n * DEPOSIT);
    expect(oneSecondLater.paid).to.equal(2n * DEPOSIT);
    expect(oneSecondLater.remainingWithdrawal).to.equal(0n);
  });

  it("changes the period-start exposure lookup after capacity was committed earlier in the boundary block", async function () {
    const ctx = await loadFixture(fundedVaultAndAllocator);
    const vault = ctx.firelight_vault;
    const nextStart = await vault.currentPeriodEnd();
    const commitmentPeriod = await vault.periodAtTimestamp(nextStart);

    expect(await vault.totalAssetsAt(nextStart)).to.equal(DEPOSIT);

    let commitTx;
    let withdrawTx;
    await network.provider.send("evm_setAutomine", [false]);
    try {
      await network.provider.send("evm_setNextBlockTimestamp", [Number(nextStart)]);

      commitTx = await ctx.allocator
        .connect(ctx.pauser)
        .commitAllocation(commitmentPeriod, ctx.tree.root, COVER, MATCHING_CAPACITY, {
          gasLimit: 5_000_000,
          maxFeePerGas: ethers.parseUnits("100", "gwei"),
          maxPriorityFeePerGas: ethers.parseUnits("3", "gwei"),
        });
      withdrawTx = await vault.connect(ctx.lp).withdraw(DEPOSIT, ctx.lp.address, ctx.lp.address, {
        gasLimit: 5_000_000,
        maxFeePerGas: ethers.parseUnits("100", "gwei"),
        maxPriorityFeePerGas: ethers.parseUnits("2", "gwei"),
      });

      await network.provider.send("evm_mine");
    } finally {
      await network.provider.send("evm_setAutomine", [true]);
    }

    const [commitReceipt, withdrawReceipt] = await Promise.all([commitTx.wait(), withdrawTx.wait()]);
    expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber);
    expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index);

    const commitment = await ctx.allocator.getAllocationCommitment(commitmentPeriod);
    expect(commitment.totalAvailableCapacity).to.equal(MATCHING_CAPACITY);
    expect(commitment.totalDeclaredAllocated).to.equal(COVER);

    // The commitment passed the allocator's capacity validation while
    // totalAssetsAt(P.start) returned 100. The later transaction in the same
    // block changes what that identical lookup returns to zero.
    expect(await vault.totalAssetsAt(nextStart)).to.equal(0n);
    expect(await vault.withdrawAssets(commitmentPeriod + 1n)).to.equal(DEPOSIT);

    // The same capacity value now fails the allocator's own formula
    // after the withdrawal. Settlement does not perform this revalidation.
    await expect(
      ctx.allocator
        .connect(ctx.limit_updater)
        .recommitAllocation(commitmentPeriod, ctx.tree.root, COVER, MATCHING_CAPACITY),
    )
      .to.be.revertedWithCustomError(ctx.allocator, "MatchingCapacityOverflow")
      .withArgs(MATCHING_CAPACITY, 0n);

    const proof = ctx.tree.getProof(0);
    const premiumCollectorBefore = await ctx.token_contract.balanceOf(
      ctx.period_configuration_updater.address,
    );
    const buyerBefore = await ctx.token_contract.balanceOf(ctx.buyer.address);
    await ctx.allocator
      .connect(ctx.pauser)
      .settleCoverOrder(ctx.orderId, [{ marketId: MARKET_ID, allocatedCover: COVER }], proof);

    const order = await ctx.allocator.getCoverOrder(ctx.orderId);
    expect(order.status).to.equal(1n);
    expect(order.period).to.equal(commitmentPeriod);
    expect(order.allocatedCoverAmount).to.equal(COVER);
    expect(order.allocatedPremiumAmount).to.be.greaterThan(0n);
    const premiumNative = (order.allocatedPremiumAmount + 10n ** 12n - 1n) / 10n ** 12n;
    expect(
      (await ctx.token_contract.balanceOf(ctx.period_configuration_updater.address)) -
        premiumCollectorBefore,
    ).to.equal(premiumNative);
    expect(buyerBefore - (await ctx.token_contract.balanceOf(ctx.buyer.address))).to.equal(
      premiumNative,
    );
    expect(await ctx.coverNFT.ownerOf(ctx.orderId)).to.equal(ctx.buyer.address);

    const captureTimestamp = await time.latest();
    expect(captureTimestamp).to.be.greaterThan(nextStart);
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .createIncident(
        captureTimestamp,
        "Boundary checkpoint incident",
        ethers.id("boundary-incident"),
      );
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .confirmIncident(1n, "ipfs://boundary-incident");
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .addAssessmentLosses(1n, [{ coverTokenId: ctx.orderId, marketId: MARKET_ID, amount: COVER }]);
    await ctx.incidentManager.connect(ctx.blocklister).submitCurrentAssessment(1n);

    const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address);
    await expect(ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1n))
      .to.emit(vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, DEPOSIT, 0n, captureTimestamp);

    const [incident] = await ctx.incidentManager.getIncident(1n);
    expect(incident.vaultPaidAmount).to.equal(0n);
    expect(incident.status).to.equal(4n);
    expect(await ctx.token_contract.balanceOf(ctx.payout_receiver.address)).to.equal(receiverBefore);

    await time.increaseTo((await vault.currentPeriodEnd()) + 1n);
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n);
    const before = await ctx.token_contract.balanceOf(ctx.lp.address);
    expect(before).to.equal(0n);
    await vault.connect(ctx.lp).claimWithdraw(commitmentPeriod + 1n);
    expect(await ctx.token_contract.balanceOf(ctx.lp.address)).to.equal(DEPOSIT);
  });

  it("ordering property: only a commitment mined before the boundary withdrawal survives the mutable checkpoint", async function () {
    const permutations = [
      ["commit", "withdraw", "checkpoint"],
      ["commit", "checkpoint", "withdraw"],
      ["checkpoint", "commit", "withdraw"],
      ["withdraw", "commit", "checkpoint"],
      ["withdraw", "checkpoint", "commit"],
      ["checkpoint", "withdraw", "commit"],
    ];

    for (const orderedActions of permutations) {
      const ctx = await loadFixture(fundedVaultAndAllocator);
      const vault = ctx.firelight_vault;
      const checkpointRole = await vault.CHECKPOINT_ROLE();
      await vault.connect(ctx.deployer).grantRole(checkpointRole, ctx.deployer.address);

      const { commitmentPeriod, nextStart, receipts } = await mineBoundaryActions(
        ctx,
        orderedActions,
      );

      const blockNumbers = new Set(Object.values(receipts).map((receipt) => receipt.blockNumber));
      expect(blockNumbers.size, orderedActions.join(" -> ")).to.equal(1);
      expect(receipts.withdraw.status, orderedActions.join(" -> ")).to.equal(1);
      expect(receipts.checkpoint.status, orderedActions.join(" -> ")).to.equal(1);

      for (let index = 1; index < orderedActions.length; index += 1) {
        expect(
          receipts[orderedActions[index - 1]].index,
          orderedActions.join(" -> "),
        ).to.be.lessThan(receipts[orderedActions[index]].index);
      }

      const commitPrecedesWithdrawal =
        orderedActions.indexOf("commit") < orderedActions.indexOf("withdraw");
      expect(receipts.commit.status, orderedActions.join(" -> ")).to.equal(
        commitPrecedesWithdrawal ? 1 : 0,
      );
      expect(await vault.totalAssetsAt(nextStart), orderedActions.join(" -> ")).to.equal(0n);

      const commitment = await ctx.allocator.getAllocationCommitment(commitmentPeriod);
      if (commitPrecedesWithdrawal) {
        // This falsifies the desired stability property. A commitment accepted
        // against 100 remains settleable after a later same-key write lowers
        // the payout-side period-start lookup to zero.
        expect(commitment.root, orderedActions.join(" -> ")).to.equal(ctx.tree.root);
        expect(commitment.totalAvailableCapacity, orderedActions.join(" -> ")).to.equal(
          MATCHING_CAPACITY,
        );

        await ctx.allocator
          .connect(ctx.pauser)
          .settleCoverOrder(
            ctx.orderId,
            [{ marketId: MARKET_ID, allocatedCover: COVER }],
            ctx.tree.getProof(0),
          );
        expect((await ctx.allocator.getCoverOrder(ctx.orderId)).status).to.equal(1n);
      } else {
        // Reverse ordering establishes the boundary dependency: once the
        // withdrawal writes zero first, the same commitment is rejected.
        expect(commitment.root, orderedActions.join(" -> ")).to.equal(ethers.ZeroHash);
      }
    }
  });

  it("negative control: the full lifecycle pays the claim when withdrawal lands one second later", async function () {
    const ctx = await loadFixture(fundedVaultAndAllocator);
    const vault = ctx.firelight_vault;
    const nextStart = await vault.currentPeriodEnd();
    const commitmentPeriod = await vault.periodAtTimestamp(nextStart);

    await time.setNextBlockTimestamp(nextStart);
    await ctx.allocator
      .connect(ctx.pauser)
      .commitAllocation(commitmentPeriod, ctx.tree.root, COVER, MATCHING_CAPACITY);

    await time.setNextBlockTimestamp(nextStart + 1n);
    await vault.connect(ctx.lp).withdraw(DEPOSIT, ctx.lp.address, ctx.lp.address);

    expect(await vault.totalAssetsAt(nextStart)).to.equal(DEPOSIT);
    expect(await vault.withdrawAssets(commitmentPeriod + 1n)).to.equal(DEPOSIT);

    const proof = ctx.tree.getProof(0);
    const buyerBefore = await ctx.token_contract.balanceOf(ctx.buyer.address);
    await ctx.allocator
      .connect(ctx.pauser)
      .settleCoverOrder(ctx.orderId, [{ marketId: MARKET_ID, allocatedCover: COVER }], proof);

    const order = await ctx.allocator.getCoverOrder(ctx.orderId);
    expect(order.status).to.equal(1n);
    expect(order.period).to.equal(commitmentPeriod);
    expect(order.allocatedCoverAmount).to.equal(COVER);
    expect(order.allocatedPremiumAmount).to.be.greaterThan(0n);
    const premiumNative = (order.allocatedPremiumAmount + 10n ** 12n - 1n) / 10n ** 12n;
    expect(buyerBefore - (await ctx.token_contract.balanceOf(ctx.buyer.address))).to.equal(
      premiumNative,
    );
    expect(await ctx.coverNFT.ownerOf(ctx.orderId)).to.equal(ctx.buyer.address);

    const captureTimestamp = await time.latest();
    expect(captureTimestamp).to.be.greaterThan(nextStart + 1n);
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .createIncident(
        captureTimestamp,
        "Control checkpoint incident",
        ethers.id("control-incident"),
      );
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .confirmIncident(1n, "ipfs://control-incident");
    await ctx.incidentManager
      .connect(ctx.blocklister)
      .addAssessmentLosses(1n, [{ coverTokenId: ctx.orderId, marketId: MARKET_ID, amount: COVER }]);
    await ctx.incidentManager.connect(ctx.blocklister).submitCurrentAssessment(1n);

    const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address);
    await expect(ctx.incidentManager.connect(ctx.assessmentApprover).approveCurrentAssessment(1n))
      .to.emit(vault, "PayoutExecuted")
      .withArgs(ctx.payout_receiver.address, DEPOSIT, DEPOSIT, captureTimestamp);

    const [incident] = await ctx.incidentManager.getIncident(1n);
    expect(incident.vaultPaidAmount).to.equal(DEPOSIT);
    expect(incident.status).to.equal(4n);
    expect(
      (await ctx.token_contract.balanceOf(ctx.payout_receiver.address)) - receiverBefore,
    ).to.equal(DEPOSIT);
    expect(await vault.withdrawAssets(commitmentPeriod + 1n)).to.equal(0n);
    expect(await ctx.token_contract.balanceOf(ctx.lp.address)).to.equal(0n);
  });
});
```

## What the PoC proves

The numbered measurements in the Summary are the evidence spine. The tests divide the proof as follows:

* `a withdrawal at the exact period start...` isolates the boundary write and the zero payout.
* `reduces payout in a funded two-LP vault...` proves the defect scales without an empty vault: one LP's 100 withdrawal reduces a 200 payout to 100, while the one-second control pays 200.
* `changes the period-start exposure lookup...` proves the commitment executes first in the same block, then settlement, premium payment, CoverNFT minting, and the zero-paid production incident lifecycle all complete.
* `ordering property...` exhaustively places `checkpoint`, `commit`, and `withdraw` within the boundary block. It proves reverse ordering rejects the commitment, same-key overwrite does not protect the historical basis, and every commit-first permutation remains settleable.
* `negative control...` moves the withdrawal one second later; the approved claim pays 100 and the withdrawal bucket ends at 0.

## Measured output

```
  period-start exposure lookup changes after capacity commitment
    ✔ a withdrawal at the exact period start removes the withdrawal from the payout cap
    ✔ reduces payout in a funded two-LP vault without requiring the vault to become empty
    ✔ changes the period-start exposure lookup after capacity was committed earlier in the boundary block
    ✔ ordering property: only a commitment mined before the boundary withdrawal survives the mutable checkpoint
    ✔ negative control: the full lifecycle pays the claim when withdrawal lands one second later


  5 passing (1s)
```

## Recommendation

Once an allocation commitment has consumed the period-start exposure value, keep the value used by payout for that period stable. One implementation is to materialize the period exposure before accepting allocation commitments and use that stored value for both capacity validation and payout. The project can preserve ordinary boundary withdrawals as long as those writes do not change the already-consumed exposure value.
