# 90175 sc medium exact period start withdrawal redefines opening exposure and lets an lp evade a valid slash

**Submitted on Aug 23rd 2026 at 13:01:43 UTC by @Jornason for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90175
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An ordinary Firelight Vault LP can request a withdrawal in a block whose timestamp is exactly the start of a new cover period. The withdrawal remains in the pending bucket and, according to both the implementation and the official staking documentation, remains exposed to incidents captured in that period. However, `withdraw()` records post-withdraw active assets at the transaction timestamp, and `payout()` later uses an inclusive historical lookup at the same `periodStart` timestamp as the maximum exposure it will pay. The equal-key checkpoint semantics therefore let the boundary withdrawal redefine the period's opening exposure after moving most capital into a bucket that is still payable.

In the full product PoC, the LP withdraws 900 of 1,000 asset units at the exact boundary, making the opening checkpoint 100 while 900 units remain pending and slashable. With the allowed 2x capacity ratio used by the repository's default allocator fixture, the normal allocator can commit and settle 200 units one hour later. A valid 200-unit loss is approved, a funded and fully approved first-loss buffer pays 50, and the Vault is asked for the remaining 150. The exact-boundary branch pays only 100, while an otherwise identical `periodStart + 1 second` control pays 150. The configured payout receiver is counterfactually underpaid by 50 Vault asset units; the withdrawing LP actually claims 45 more units and retains shares that the Vault values 5 units higher, for a total 50-unit position difference. The incident is then terminally `CLOSED`, with no normal reopen, retry, or shortfall-top-up path.

## Vulnerability Details

### 1. The withdrawal remains slashable

`withdraw()` is a public LP entrypoint. It computes the shares to burn, calls `_requestWithdraw()`, and then logs post-withdraw totals:

```solidity
// contracts/core/FirelightVault.sol:755-775
function withdraw(uint256 assets, address receiver, address owner) public ... returns (uint256) {
    if (assets == 0) revert InvalidAmount();

    (uint256 shares, uint256 _totalSupply, uint256 _totalAssets) =
        _previewTotals(assets, true, Math.Rounding.Ceil);

    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
    return shares;
}
```

`_requestWithdraw()` assigns the request to `currentPeriod() + 1` and increases `pendingWithdrawAssets`:

```solidity
// contracts/core/FirelightVault.sol:1008-1019
uint256 period = currentPeriod() + 1;
...
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

For a transaction mined at `periodStart(P)`, `currentPeriod()` is already `P`. The request is therefore stored in bucket `P + 1`, which `payout()` expressly treats as payable for an incident captured in `P`.

The official staking documentation confirms the intended exposure: after unstaking, capital is still backing coverage for the remainder of the period in which unstaking was initiated, and a pending withdrawal takes the same pro-rata slash as active positions if an incident occurs in that period. An exact-boundary transaction is already in period `P` under the Vault's own `currentPeriod()` calculation, so its pending capital must remain exposed throughout `P`.

### 2. The same transaction redefines opening exposure

`_logTrace()` keys the post-withdraw active-asset value by the current timestamp:

```solidity
// contracts/core/FirelightVault.sol:1044-1055
function _logTrace(..., uint256 _totalAssets, bool isLogAssets) private {
    uint48 ts = Time.timestamp();
    ...
    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
}
```

The custom `Trace256.push()` replaces the latest value when the key equals the latest timestamp, and its lookup returns the latest key lower than or equal to the requested key:

```solidity
// contracts/core/lib/Checkpoints.sol:198-223
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
...
uint208 idx = self._trace.upperLookupRecent(key);
return idx > 0 ? self._values[idx] : 0;
```

It is not necessary that a checkpoint already exist at the boundary: the first write at the exact key is also included by `upperLookupRecent(periodStart)`. If multiple writes occur at that timestamp, the final equal-key value controls; allocator commitment and settlement do not write the Vault's total-assets checkpoint.

### 3. Payout caps itself by the corrupted value while including the pending assets

`payout()` resolves the incident's capture-period start, reads the inclusive checkpoint, separately includes the capture-period withdrawal bucket in payable assets, and takes the minimum:

```solidity
// contracts/core/FirelightVault.sol:900-927
(uint256 capturePeriod, uint48 capturePeriodStart) = _periodAndStartAt(captureTimestamp);
...
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The 900 pending units are therefore recognized as currently slashable but excluded from the opening cap because the same withdrawal wrote `100` as the period-start active value.

### 4. The short payment becomes terminal

`approveCurrentAssessment()` executes the payout and then unconditionally closes the incident:

```solidity
// contracts/core/IncidentManager.sol:423-438
uint256 vaultPaidAmount = _executePayout(...);
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
_decreaseIncident(incident.period);
emit IncidentClosed(incidentId, vaultPaidAmount, totalAssessmentLoss);
```

No normal entrypoint reopens the incident or pays the difference. The LP's pending claim remains live and is later paid through `claimWithdraw()`.

### Attack Sequence

The primary path uses an allowed 2x capacity configuration (`effectiveLeverage=24000`, `minCAR=12000`) mirrored by the repository's default allocator fixture and does not require allocator ordering in the boundary block. The Phase 2 allocator is a new deployment, so this report does not claim that 2x is already a deployed parameter; the separate strict-1x cell covers the same root with a boundary-block ordering requirement.

1. A normal LP holds the Vault's 1,000 asset units. A normal 200-unit cover order is pending.
2. The LP broadcasts `withdraw(900, lp, lp)` for inclusion at the known next period start.
3. The transaction is mined at `periodStart(P)`. Active assets become 100, pending assets become 900, and `totalAssetsAt(periodStart(P))` becomes 100.
4. One hour later, the normal allocator commits 200 units. This is valid because 100 units at the configured 2x ratio support 200 units. The allocator then settles the valid cover order.
5. A real incident is captured in `P`, and the normal incident roles approve a valid 200-unit assessment. No role makes an incorrect business decision.
6. The funded and fully approved first-loss buffer transfers 50 units. The IncidentManager asks the Vault for the remaining 150.
7. Although the Vault holds 100 active plus 900 pending payable units, the corrupted opening checkpoint caps the Vault transfer at 100.
8. The payout receiver is short 50 Vault asset units. The incident transitions to `CLOSED`.
9. After the normal delay, the LP claims 810 pending units and retains shares worth 90 under the Vault's conversion function. In the `periodStart + 1 second` control, it claims 765 and retains shares worth 85. The boundary transaction therefore pays the LP 45 extra units immediately and leaves 5 additional units of redeemable share value, for a 50-unit contract-valued position advantage.

At a strict 1x ratio, the PoC separately proves the same bug with the normal allocator commitment first and the LP withdrawal second in the boundary block; settlement still occurs in a later block. Leverage changes only the timing requirement, not the root cause.

### Scope And Known-Issue Distinctions

* `FirelightVault.sol` and `Checkpoints.sol` are listed in scope, and Medium griefing is an accepted impact.
* This does not use aggregate or duplicate claims: one valid cover and one valid assessment are used in both branches.
* This does not use the FLB funding/allowance known issue: the payer is funded for 50 units and grants the full allowance; that transfer succeeds in both branches.
* This does not rely on trusted roles making an intentionally incorrect decision. The allocator commits within the enforced capacity, settlement is valid, and the incident roles approve a valid loss. The value-changing step is the ordinary LP's boundary withdrawal.
* This is distinct from stale post-payout capacity. Here, a boundary withdrawal writes the low opening checkpoint that caps payout; a live-settlement revalidation would not repair the already corrupted opening exposure. Conversely, making the opening checkpoint immutable would not repair a later payout that reduces live backing after a healthy commitment.
* Upstream PRs `#1-#10`, issue `#11`, the Macro report, and the published known issues contain no exact matching root. PR `#9` concerns ERC4626 max views, while merged PR `#10` leaves this checkpoint/payout composition unchanged.

### Recommended Mitigation

Store a dedicated immutable opening-exposure snapshot for each period and prevent ordinary state transitions at or after the boundary from redefining it. Alternatively, make payout and capacity use the last total-assets checkpoint strictly before `periodStart`, while retaining the current timestamped trace for general historical queries. Add regressions for exact-boundary `withdraw` and `redeem`, later normal matching at ratios above 1x, strict-1x commitment ordering, a funded first-loss buffer, and actual mature claims.

## Impact Details

The selected impact is Medium griefing. The PoC demonstrates a concrete on-chain value differential rather than only a control or API inconsistency:

```
same valid assessment loss:                 200
same fully funded/approved FLB payment:       50

exact period start:
  Vault requested / paid:               150 / 100
  payout receiver Vault-asset delta:          100
  LP mature claim + retained value:      810 + 90 = 900

periodStart + 1 second control:
  Vault requested / paid:               150 / 150
  payout receiver Vault-asset delta:          150
  LP mature claim + retained value:      765 + 85 = 850

receiver shortfall:                            50
LP additional mature claim actually paid:       45
LP additional retained-share value:              5
LP total contract-valued position difference:   50
```

The advantage scales with the amount moved into the pending bucket and the valid Vault-funded loss, up to the corrupted opening cap. The attacker does not need to fabricate a claim or control a role; it opportunistically avoids a valid slash if an incident occurs during the period. If no incident occurs, the request follows the ordinary delayed-withdrawal lifecycle.

The PoC does not claim that 50 units are debited from the payout receiver's pre-existing balance. It proves that a valid Vault leg terminally closes 50 units below the `+1 second` control, while the LP is actually paid 45 additional units and retains shares worth 5 additional units. The impact is therefore framed as Medium slash evasion/griefing rather than Critical direct theft, and does not rely on asserting harm to a downstream off-chain beneficiary.

### Feasibility

The period boundary is public through `currentPeriodEnd()`. A saved read-only Flare observation found block `67313847` with timestamp `1786628569`, exactly equal to the deployed Vault's period start `1786628569`; the block contained 60 transactions, with adjacent blocks at `T-1` and `T+1`. Exact-boundary blocks are therefore a real chain state, not a Hardhat-only timestamp.

The primary 2x path requires the LP withdrawal to be included at the exact timestamp and requires that no later active-increasing Vault checkpoint at that same timestamp restore the opening cap above the Vault request. It does not require allocator commitment or settlement in the boundary block; those calls occur later and do not write the Vault's total-assets trace. A later same-timestamp write that records the same or a lower active balance does not remove the effect, while a sufficiently large deposit or reward checkpoint could. The strict-1x alternative additionally requires the LP to follow a normal pending commitment in the boundary block. Neither path requires a privileged key, malicious role action, unusual token behavior, low share supply, or a first-loss-buffer funding failure.

## References

* FirelightVault: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol>
* Checkpoints library: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol>
* CoverOrderAllocator: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol>
* IncidentManager: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol>
* Official staking deposits and withdrawals documentation: <https://docs.firelight.finance/for-stakers/deployments-and-withdrawals>

## Link to Proof of Concept

<https://gist.github.com/Jornason/0cd2b8ea47b0a49de539f8a5ac2853c3>

## Proof of Concept

## Proof of Concept

### Summary

This executable Hardhat PoC runs against the pinned competition source and deploys the real FirelightVault, CoverOrderAllocator, CoverNFT, IncidentManager, price-feed integration, ERC20 transfers, first-loss-buffer waterfall, and delayed withdrawal claims. It compares exact `periodStart` inclusion against an otherwise identical `periodStart + 1 second` control from the same EVM snapshot.

The four cells prove:

1. strict 1x capacity: a normal commitment precedes the LP withdrawal in the boundary block, settlement occurs later, and the exact branch pays 100 instead of 200;
2. allowed/default-fixture 2x capacity: the LP withdraws at the boundary and the normal allocator commits and settles in later blocks, while the exact branch pays 100 instead of 200;
3. funded-buffer 2x capacity: the FLB payer holds 50 tokens and grants the full allowance, but the exact branch still pays only 100 of the remaining 150 Vault units; the LP actually claims 45 more units and retains shares worth 5 more; and
4. a strict-1x negative control showing that later matching cannot exceed the reduced capacity, which isolates the exploitability condition from the checkpoint primitive.

### Environment

```
Repository: https://github.com/immunefi-team/audit-comp-firelight
Commit: 42f9ea5e43d88b35197b901acc2a8c24b314fa62
Framework: Hardhat using the repository's existing dependencies and mocks
Network: local Hardhat network only
Mainnet/public-testnet transactions: none
```

### Files

Copy the supplied test file into the repository checkout:

```
poc/FirelightBoundarySlashEvasion.test.js -> test/FirelightBoundarySlashEvasion.test.js
```

No in-scope source file, project configuration, or dependency is modified, and the test uses no direct storage injection.

### Reproduction

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install

cp /path/to/FirelightBoundarySlashEvasion.test.js test/FirelightBoundarySlashEvasion.test.js
npx hardhat test test/FirelightBoundarySlashEvasion.test.js
```

Expected result for the supplied test alone:

```
  4 passing
```

The saved producer run also executed the relevant official Vault payout and allocator commitment regressions and ended with `87 passing`. The package also includes the separately structured verifier source, its command record, and a saved fresh run that compiled 133 Solidity files before ending with `2 passing`.

### Decisive Funded-Buffer Output

```
source SHA:                       42f9ea5e43d88b35197b901acc2a8c24b314fa62
asset decimals:                  6
FLB balance and allowance:       50
valid settled cover/loss:        200

exact periodStart:
  opening checkpoint:            100.000000
  active / pending:              100.000000 / 900.000000
  commit block:                  later than withdrawal block
  Vault requested / paid:        150.000000 / 100.000000
  LP mature claim:               810.000000
  LP retained-share value:       90.000000
  LP final position:             900.000000

periodStart + 1 second:
  opening checkpoint:            1000.000000
  active / pending:              100.000000 / 900.000000
  commit block:                  later than withdrawal block
  Vault requested / paid:        150.000000 / 150.000000
  LP mature claim:               765.000000
  LP retained-share value:       85.000000
  LP final position:             850.000000

receiver Vault-asset shortfall:  50.000000
LP additional mature claim paid: 45.000000
LP additional retained value:     5.000000
LP total position difference:    50.000000
```

### Assertions That Prevent A Vacuous Pass

* The LP, buyer, payout receiver, premium collector, and FLB payer are distinct accounts.
* The Vault starts with ordinary 1:1 assets and shares; there is no low-supply or donation setup.
* The cover order is created, committed, settled, and represented by a real CoverNFT.
* The allocator commitment and settlement occur after the withdrawal in both fixture-2x branches.
* The incident follows `createIncident -> confirmIncident -> addAssessmentLosses -> submitCurrentAssessment -> approveCurrentAssessment`.
* The FLB payer is funded and grants the full allowance before the funded-buffer branches.
* The payout receiver's ERC20 balance delta equals `vaultPaidAmount`.
* The LP attempts no premature claim; after maturity the actual wallet delta equals the recorded claim.
* The exact and control branches restore the same deployment snapshot; the withdrawal timestamp is the isolated input.
* A strict-1x no-impact cell confirms that a later commitment limited to 100 produces no suppressed payout.

### Saved Evidence

The submission directory includes:

```
evidence/producer-and-regressions.txt
evidence/independent-verifier.txt
evidence/independent-verifier-fresh-compile.txt
evidence/commands.txt
evidence/flare-boundary-rpc.jsonl
```

The producer output contains the full machine-readable 1x, 2x, and funded-buffer state objects and ends with `87 passing`. The verifier outputs independently reproduce funded-buffer 1x and 2x differentials; the fresh output records compilation of 133 Solidity files and ends with `2 passing`. The RPC transcript records the deployed Vault address and the read-only calls used for the timestamp anchor.

### Production Timestamp Anchor

A saved read-only observation of Flare block `67313847` records:

```
block timestamp:       1786628569
Vault period start:    1786628569
transaction count:     60
previous block:        1786628568
following block:       1786628570
```

This proves exact period-start timestamps occur on the target chain. The PoC itself remains fully local and broadcasts no public-network transaction.

### Complete PoC Source

The following is the complete content of `test/FirelightBoundarySlashEvasion.test.js`:

```javascript
const { expect } = require('chai')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { ethers, upgrades } = require('hardhat')

const DAY = 24n * 60n * 60n
const ONE = 10n ** 18n
const ASSET_UNIT = 10n ** 6n

function canonical(amount) {
  return amount * ONE
}

function assetAmount(amount) {
  return amount * ASSET_UNIT
}

function parseEvent(contract, receipt, name) {
  for (const log of receipt.logs) {
    try {
      const event = contract.interface.parseLog(log)
      if (event && event.name === name) return event
    } catch {}
  }
  throw new Error(`${name} event missing`)
}

function stringify(value) {
  if (typeof value === 'bigint') return value.toString()
  if (Array.isArray(value)) return value.map(stringify)
  if (value && typeof value === 'object') {
    return Object.fromEntries(Object.entries(value).map(([key, nested]) => [key, stringify(nested)]))
  }
  return value
}

async function deployProduct({ effectiveLeverage = 24_000n, orderCover = 200n, flbBalance = 0n } = {}) {
  const [admin, lp, buyer, payoutReceiver, premiumCollector, flbPayer] = await ethers.getSigners()
  const asset = await (await ethers.getContractFactory('MockERC20')).deploy('V7H2M9 Asset', 'V7A', 6)
  const premium = await (await ethers.getContractFactory('MockERC20')).deploy('V7H2M9 Premium', 'V7P', 0)
  const flb = await (await ethers.getContractFactory('MockERC20')).deploy('V7H2M9 FLB', 'V7F', 0)

  const vaultInit = ethers.AbiCoder.defaultAbiCoder().encode(
    ['address', 'address', 'address', 'address', 'address', 'address', 'uint256', 'uint48'],
    [
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      assetAmount(100_000n),
      DAY
    ]
  )
  const vault = await upgrades.deployProxy(
    await ethers.getContractFactory('FirelightVault'),
    [asset.target, 'V7H2M9 Vault', 'V7V', vaultInit]
  )
  const priceFeed = await (await ethers.getContractFactory('MockPriceFeed')).deploy(18, ONE)
  const coverNFT = await upgrades.deployProxy(
    await ethers.getContractFactory('CoverNFT'),
    ['V7H2M9 Cover', 'V7C', '', admin.address, admin.address, ethers.ZeroAddress, ethers.ZeroAddress],
    { kind: 'transparent' }
  )

  const market = { chainId: 1, protocol: 'V7H2M9', market: ethers.encodeBytes32String('market') }
  const marketId = ethers.keccak256(
    ethers.AbiCoder.defaultAbiCoder().encode(
      ['uint64', 'string', 'bytes32'],
      [market.chainId, market.protocol, market.market]
    )
  )
  const allocator = await upgrades.deployProxy(
    await ethers.getContractFactory('CoverOrderAllocator'),
    [{
      vault: vault.target,
      premiumCollector: premiumCollector.address,
      coverNFT: coverNFT.target,
      priceFeedAdapter: priceFeed.target,
      maxPriceAge: Number(2n * DAY),
      premiumTokens: [premium.target],
      admin: admin.address,
      adminRole: admin.address,
      curatorRole: admin.address,
      allocatorRole: admin.address,
      configAdminRole: admin.address,
      initialProtocolConcentrations: [{
        protocol: market.protocol,
        chainId: market.chainId,
        maxProtocolConcentrationBps: 10_000
      }],
      newMarkets: [market],
      capacityConfig: {
        minCAR: 12_000,
        firstLossBufferToken: flb.target,
        firstLossBuffer: flbPayer.address,
        effectiveLeverage,
        minOrderMarketCoverAmount: 1,
        divergenceToleranceBps: 0
      }
    }],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )
  await coverNFT.grantRole(await coverNFT.MINTER_ROLE(), allocator.target)

  const incidentManager = await upgrades.deployProxy(
    await ethers.getContractFactory('IncidentManager'),
    [
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      admin.address,
      payoutReceiver.address,
      allocator.target,
      priceFeed.target,
      Number(2n * DAY)
    ],
    { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
  )
  await vault.grantRole(await vault.PAYOUT_ROLE(), incidentManager.target)
  await vault.grantRole(await vault.INCIDENT_ROLE(), incidentManager.target)
  await vault.grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), admin.address)
  await vault.addToPayoutAllowlist(payoutReceiver.address)

  if (flbBalance > 0n) {
    await flb.mint(flbPayer.address, flbBalance)
    await flb.connect(flbPayer).approve(incidentManager.target, flbBalance)
  }

  await asset.mint(lp.address, assetAmount(1_000n))
  await asset.connect(lp).approve(vault.target, assetAmount(1_000n))
  await vault.connect(lp).deposit(assetAmount(1_000n), lp.address)

  await premium.mint(buyer.address, 100n)
  await premium.connect(buyer).approve(allocator.target, 100n)
  await allocator.createCoverOrder(
    buyer.address,
    buyer.address,
    'V7H2M9 beneficiary',
    premium.target,
    [{ marketId, coverRateAnnual: 500, coverAmount: canonical(orderCover) }],
    0
  )

  return {
    admin,
    lp,
    buyer,
    payoutReceiver,
    asset,
    premium,
    flb,
    vault,
    priceFeed,
    coverNFT,
    allocator,
    incidentManager,
    marketId,
    orderCover,
    effectiveLeverage,
    flbBalance
  }
}

async function runBranch(ctx, withdrawalOffset) {
  const {
    admin,
    lp,
    payoutReceiver,
    asset,
    vault,
    priceFeed,
    coverNFT,
    allocator,
    incidentManager,
    marketId,
    orderCover
  } = ctx
  const boundary = await vault.currentPeriodEnd()

  await time.setNextBlockTimestamp(boundary + BigInt(withdrawalOffset))
  const withdrawalReceipt = await (
    await vault.connect(lp).withdraw(assetAmount(900n), lp.address, lp.address)
  ).wait()
  const snapshotAfterWithdrawal = await vault.totalAssetsAt(boundary)
  const activeAfterWithdrawal = await vault.totalAssets()
  const pendingAfterWithdrawal = await vault.pendingWithdrawAssets()
  const commitmentPeriod = await vault.currentPeriod()

  // The matcher operates later in the period, after observing the finalized boundary state.
  await time.setNextBlockTimestamp(boundary + 3_600n)
  await priceFeed.setLatestRoundData(2, ONE, boundary + 3_600n, boundary + 3_600n, 2)

  const allocation = [[marketId, canonical(orderCover)]]
  const tree = StandardMerkleTree.of([[0n, allocation]], ['uint256', '(bytes32,uint256)[]'])
  const matchingCapacity = canonical(orderCover)
  const commitReceipt = await (
    await allocator.connect(admin).commitAllocation(
      commitmentPeriod,
      tree.root,
      matchingCapacity,
      matchingCapacity
    )
  ).wait()
  const settleReceipt = await (
    await allocator.connect(admin).settleCoverOrder(0n, [{ marketId, allocatedCover: canonical(orderCover) }], tree.getProof(0))
  ).wait()

  expect(commitReceipt.blockNumber).to.be.greaterThan(withdrawalReceipt.blockNumber)
  expect(settleReceipt.blockNumber).to.be.greaterThan(commitReceipt.blockNumber)
  expect((await allocator.getCoverOrder(0n)).allocatedCoverAmount).to.equal(canonical(orderCover))
  expect(await coverNFT.ownerOf(0n)).to.equal(ctx.buyer.address)

  const captureTimestamp = BigInt(await time.latest())
  await incidentManager.createIncident(captureTimestamp, 'V7H2M9 incident', ethers.id('V7H2M9-incident'))
  await incidentManager.confirmIncident(1n, 'ipfs://v7h2m9')
  await incidentManager.addAssessmentLosses(
    1n,
    [{ coverTokenId: 0n, marketId, amount: canonical(orderCover) }]
  )
  await incidentManager.submitCurrentAssessment(1n)

  const receiverBefore = await asset.balanceOf(payoutReceiver.address)
  const payoutReceipt = await (await incidentManager.connect(admin).approveCurrentAssessment(1n)).wait()
  const payoutEvent = parseEvent(incidentManager, payoutReceipt, 'IncidentPayoutExecuted').args
  const receiverDelta = (await asset.balanceOf(payoutReceiver.address)) - receiverBefore
  expect(receiverDelta).to.equal(payoutEvent.vaultPaidAmount)
  expect(payoutEvent.firstLossTokenAmount).to.equal(ctx.flbBalance)

  const withdrawalPeriod = commitmentPeriod + 1n
  const pendingAfterPayout = await vault.pendingWithdrawAssets()
  const activeAfterPayout = await vault.totalAssets()
  await time.increaseTo((await vault.nextPeriodEnd()) + 1n)
  const lpWalletBeforeClaim = await asset.balanceOf(lp.address)
  await vault.connect(lp).claimWithdraw(withdrawalPeriod)
  const claimed = (await asset.balanceOf(lp.address)) - lpWalletBeforeClaim
  const retainedShares = await vault.balanceOf(lp.address)
  const retainedShareValue = await vault.convertToAssets(retainedShares)
  const finalLpPosition = (await asset.balanceOf(lp.address)) + retainedShareValue

  return {
    withdrawalOffset,
    boundary,
    withdrawalBlock: withdrawalReceipt.blockNumber,
    commitBlock: commitReceipt.blockNumber,
    settleBlock: settleReceipt.blockNumber,
    commitmentPeriod,
    snapshotAfterWithdrawal,
    activeAfterWithdrawal,
    pendingAfterWithdrawal,
    matchingCapacity,
    allocatedCover: (await allocator.getCoverOrder(0n)).allocatedCoverAmount,
    captureTimestamp,
    firstLossTokenAmount: payoutEvent.firstLossTokenAmount,
    vaultRequestedAmount: payoutEvent.vaultRequestedAmount,
    vaultPaidAmount: payoutEvent.vaultPaidAmount,
    receiverDelta,
    pendingAfterPayout,
    activeAfterPayout,
    claimed,
    retainedShares,
    retainedShareValue,
    finalLpPosition
  }
}

async function runCommitFirstBranch(ctx, exactBoundary) {
  const {
    admin,
    lp,
    payoutReceiver,
    asset,
    vault,
    priceFeed,
    coverNFT,
    allocator,
    incidentManager,
    marketId,
    orderCover
  } = ctx
  const boundary = await vault.currentPeriodEnd()
  const commitmentPeriod = await vault.periodAtTimestamp(boundary)
  const allocation = [[marketId, canonical(orderCover)]]
  const tree = StandardMerkleTree.of([[0n, allocation]], ['uint256', '(bytes32,uint256)[]'])

  await time.setNextBlockTimestamp(boundary)
  await ethers.provider.send('evm_setAutomine', [false])
  let commitTx
  let withdrawalTx
  try {
    commitTx = await allocator.connect(admin).commitAllocation(
      commitmentPeriod,
      tree.root,
      canonical(orderCover),
      canonical(1_000n),
      { gasLimit: 8_000_000, gasPrice: ethers.parseUnits('2', 'gwei') }
    )
    if (exactBoundary) {
      withdrawalTx = await vault.connect(lp).withdraw(
        assetAmount(900n),
        lp.address,
        lp.address,
        { gasLimit: 8_000_000, gasPrice: ethers.parseUnits('1', 'gwei') }
      )
    }
    await ethers.provider.send('evm_mine', [])
  } finally {
    await ethers.provider.send('evm_setAutomine', [true])
  }
  const commitReceipt = await commitTx.wait()

  if (!exactBoundary) {
    await time.setNextBlockTimestamp(boundary + 1n)
    withdrawalTx = await vault.connect(lp).withdraw(assetAmount(900n), lp.address, lp.address)
  }
  const withdrawalReceipt = await withdrawalTx.wait()
  const boundarySnapshot = await vault.totalAssetsAt(boundary)
  const activeAfterWithdrawal = await vault.totalAssets()
  const pendingAfterWithdrawal = await vault.pendingWithdrawAssets()

  await time.setNextBlockTimestamp(boundary + 3_600n)
  await priceFeed.setLatestRoundData(2, ONE, boundary + 3_600n, boundary + 3_600n, 2)
  const settleReceipt = await (
    await allocator.connect(admin).settleCoverOrder(0n, [{ marketId, allocatedCover: canonical(orderCover) }], tree.getProof(0))
  ).wait()
  expect(settleReceipt.blockNumber).to.be.greaterThan(commitReceipt.blockNumber)
  expect((await allocator.getCoverOrder(0n)).allocatedCoverAmount).to.equal(canonical(orderCover))
  expect(await coverNFT.ownerOf(0n)).to.equal(ctx.buyer.address)

  const captureTimestamp = BigInt(await time.latest())
  await incidentManager.createIncident(captureTimestamp, 'V7H2M9 commit-first incident', ethers.id('V7H2M9-commit-first'))
  await incidentManager.confirmIncident(1n, 'ipfs://v7h2m9-commit-first')
  await incidentManager.addAssessmentLosses(
    1n,
    [{ coverTokenId: 0n, marketId, amount: canonical(orderCover) }]
  )
  await incidentManager.submitCurrentAssessment(1n)

  const receiverBefore = await asset.balanceOf(payoutReceiver.address)
  const payoutReceipt = await (await incidentManager.connect(admin).approveCurrentAssessment(1n)).wait()
  const payoutEvent = parseEvent(incidentManager, payoutReceipt, 'IncidentPayoutExecuted').args
  const receiverDelta = (await asset.balanceOf(payoutReceiver.address)) - receiverBefore
  expect(receiverDelta).to.equal(payoutEvent.vaultPaidAmount)
  expect(payoutEvent.firstLossTokenAmount).to.equal(ctx.flbBalance)

  const withdrawalPeriod = commitmentPeriod + 1n
  const pendingAfterPayout = await vault.pendingWithdrawAssets()
  await time.increaseTo((await vault.nextPeriodEnd()) + 1n)
  const walletBeforeClaim = await asset.balanceOf(lp.address)
  await vault.connect(lp).claimWithdraw(withdrawalPeriod)
  const claimed = (await asset.balanceOf(lp.address)) - walletBeforeClaim
  const retainedShares = await vault.balanceOf(lp.address)
  const retainedShareValue = await vault.convertToAssets(retainedShares)
  const finalLpPosition = (await asset.balanceOf(lp.address)) + retainedShareValue

  return {
    exactBoundary,
    boundary,
    commitBlock: commitReceipt.blockNumber,
    commitIndex: commitReceipt.index,
    withdrawalBlock: withdrawalReceipt.blockNumber,
    withdrawalIndex: withdrawalReceipt.index,
    settleBlock: settleReceipt.blockNumber,
    commitmentPeriod,
    committedMatchingCapacity: (await allocator.getAllocationCommitment(commitmentPeriod)).totalAvailableCapacity,
    boundarySnapshot,
    activeAfterWithdrawal,
    pendingAfterWithdrawal,
    allocatedCover: (await allocator.getCoverOrder(0n)).allocatedCoverAmount,
    captureTimestamp,
    firstLossTokenAmount: payoutEvent.firstLossTokenAmount,
    vaultRequestedAmount: payoutEvent.vaultRequestedAmount,
    vaultPaidAmount: payoutEvent.vaultPaidAmount,
    receiverDelta,
    pendingAfterPayout,
    claimed,
    retainedShares,
    retainedShareValue,
    finalLpPosition
  }
}

describe('V7H2M9 exact-boundary checkpoint with later matching', function () {
  this.timeout(300_000)

  it('needs only commit before the boundary withdrawal; settlement remains in a later block at 1x', async function () {
    const ctx = await deployProduct({ effectiveLeverage: 12_000n, orderCover: 200n })
    const snapshotId = await ethers.provider.send('evm_snapshot', [])

    const exact = await runCommitFirstBranch(ctx, true)
    expect(await ethers.provider.send('evm_revert', [snapshotId])).to.equal(true)
    const plusOne = await runCommitFirstBranch(ctx, false)

    expect(exact.commitBlock).to.equal(exact.withdrawalBlock)
    expect(exact.commitIndex).to.be.lessThan(exact.withdrawalIndex)
    expect(exact.settleBlock).to.be.greaterThan(exact.withdrawalBlock)
    expect(plusOne.commitBlock).to.be.lessThan(plusOne.withdrawalBlock)
    expect(plusOne.settleBlock).to.be.greaterThan(plusOne.withdrawalBlock)
    expect(exact.committedMatchingCapacity).to.equal(canonical(1_000n))
    expect(plusOne.committedMatchingCapacity).to.equal(canonical(1_000n))
    expect(exact.boundarySnapshot).to.equal(assetAmount(100n))
    expect(plusOne.boundarySnapshot).to.equal(assetAmount(1_000n))
    expect(exact.activeAfterWithdrawal).to.equal(assetAmount(100n))
    expect(plusOne.activeAfterWithdrawal).to.equal(assetAmount(100n))
    expect(exact.pendingAfterWithdrawal).to.equal(assetAmount(900n))
    expect(plusOne.pendingAfterWithdrawal).to.equal(assetAmount(900n))
    expect(exact.allocatedCover).to.equal(canonical(200n))
    expect(plusOne.allocatedCover).to.equal(canonical(200n))
    expect(exact.vaultRequestedAmount).to.equal(assetAmount(200n))
    expect(plusOne.vaultRequestedAmount).to.equal(assetAmount(200n))
    expect(exact.vaultPaidAmount).to.equal(assetAmount(100n))
    expect(plusOne.vaultPaidAmount).to.equal(assetAmount(200n))
    expect(exact.claimed).to.equal(assetAmount(810n))
    expect(plusOne.claimed).to.equal(assetAmount(720n))
    expect(exact.finalLpPosition).to.equal(assetAmount(900n))
    expect(plusOne.finalLpPosition).to.equal(assetAmount(800n))

    console.log(`V7H2M9_COMMIT_FIRST_EVIDENCE=${JSON.stringify(stringify({
      sourceSha: '42f9ea5e43d88b35197b901acc2a8c24b314fa62',
      assetDecimals: 6,
      exact,
      plusOne,
      leverageRatio: '1x',
      isolatedInput: 'withdrawal timestamp: periodStart versus periodStart + 1',
      exactOrdering: 'commit then withdrawal in the boundary block; settlement later',
      payoutSuppression: plusOne.vaultPaidAmount - exact.vaultPaidAmount,
      lpEconomicAdvantage: exact.finalLpPosition - plusOne.finalLpPosition
    }))}`)
  })

  it('uses a later normal commit yet suppresses payout only at the exact boundary', async function () {
    const ctx = await deployProduct({ effectiveLeverage: 24_000n, orderCover: 200n })
    const snapshotId = await ethers.provider.send('evm_snapshot', [])

    const exact = await runBranch(ctx, 0)
    expect(await ethers.provider.send('evm_revert', [snapshotId])).to.equal(true)
    const plusOne = await runBranch(ctx, 1)

    expect(exact.snapshotAfterWithdrawal).to.equal(assetAmount(100n))
    expect(plusOne.snapshotAfterWithdrawal).to.equal(assetAmount(1_000n))
    expect(exact.activeAfterWithdrawal).to.equal(assetAmount(100n))
    expect(plusOne.activeAfterWithdrawal).to.equal(assetAmount(100n))
    expect(exact.pendingAfterWithdrawal).to.equal(assetAmount(900n))
    expect(plusOne.pendingAfterWithdrawal).to.equal(assetAmount(900n))
    expect(exact.matchingCapacity).to.equal(canonical(200n))
    expect(plusOne.matchingCapacity).to.equal(canonical(200n))
    expect(exact.allocatedCover).to.equal(canonical(200n))
    expect(plusOne.allocatedCover).to.equal(canonical(200n))
    expect(exact.vaultRequestedAmount).to.equal(assetAmount(200n))
    expect(plusOne.vaultRequestedAmount).to.equal(assetAmount(200n))
    expect(exact.vaultPaidAmount).to.equal(assetAmount(100n))
    expect(plusOne.vaultPaidAmount).to.equal(assetAmount(200n))
    expect(exact.pendingAfterPayout).to.equal(assetAmount(810n))
    expect(plusOne.pendingAfterPayout).to.equal(assetAmount(720n))
    expect(exact.claimed).to.equal(assetAmount(810n))
    expect(plusOne.claimed).to.equal(assetAmount(720n))
    expect(exact.finalLpPosition).to.equal(assetAmount(900n))
    expect(plusOne.finalLpPosition).to.equal(assetAmount(800n))

    console.log(`V7H2M9_EVIDENCE=${JSON.stringify(stringify({
      sourceSha: '42f9ea5e43d88b35197b901acc2a8c24b314fa62',
      assetDecimals: 6,
      exact,
      plusOne,
      isolatedInput: 'withdrawal timestamp: periodStart versus periodStart + 1',
      matcherTiming: 'commit and settle occur in later blocks after withdrawal in both branches',
      payoutSuppression: plusOne.vaultPaidAmount - exact.vaultPaidAmount,
      lpEconomicAdvantage: exact.finalLpPosition - plusOne.finalLpPosition
    }))}`)
  })

  it('still suppresses the vault leg when the first-loss buffer is funded and fully approved', async function () {
    const ctx = await deployProduct({ effectiveLeverage: 24_000n, orderCover: 200n, flbBalance: 50n })
    const snapshotId = await ethers.provider.send('evm_snapshot', [])

    const exact = await runBranch(ctx, 0)
    expect(await ethers.provider.send('evm_revert', [snapshotId])).to.equal(true)
    const plusOne = await runBranch(ctx, 1)

    expect(exact.firstLossTokenAmount).to.equal(50n)
    expect(plusOne.firstLossTokenAmount).to.equal(50n)
    expect(exact.vaultRequestedAmount).to.equal(assetAmount(150n))
    expect(plusOne.vaultRequestedAmount).to.equal(assetAmount(150n))
    expect(exact.vaultPaidAmount).to.equal(assetAmount(100n))
    expect(plusOne.vaultPaidAmount).to.equal(assetAmount(150n))
    expect(exact.pendingAfterPayout).to.equal(assetAmount(810n))
    expect(plusOne.pendingAfterPayout).to.equal(assetAmount(765n))
    expect(exact.finalLpPosition).to.equal(assetAmount(900n))
    expect(plusOne.finalLpPosition).to.equal(assetAmount(850n))

    console.log(`V7H2M9_FUNDED_FLB_EVIDENCE=${JSON.stringify(stringify({
      sourceSha: '42f9ea5e43d88b35197b901acc2a8c24b314fa62',
      flbFunding: '50 tokens, fully approved to IncidentManager',
      exact,
      plusOne,
      vaultPayoutSuppression: plusOne.vaultPaidAmount - exact.vaultPaidAmount,
      lpEconomicAdvantage: exact.finalLpPosition - plusOne.finalLpPosition
    }))}`)
  })

  it('at 1x leverage the post-withdraw capacity cannot create an amount above the payout cap', async function () {
    const ctx = await deployProduct({ effectiveLeverage: 12_000n, orderCover: 100n })
    const exact = await runBranch(ctx, 0)

    expect(exact.snapshotAfterWithdrawal).to.equal(assetAmount(100n))
    expect(exact.matchingCapacity).to.equal(canonical(100n))
    expect(exact.vaultRequestedAmount).to.equal(assetAmount(100n))
    expect(exact.vaultPaidAmount).to.equal(assetAmount(100n))
    expect(exact.receiverDelta).to.equal(assetAmount(100n))
    expect(exact.finalLpPosition).to.equal(assetAmount(900n))
  })
})
```
