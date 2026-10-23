# 88237 sc high previous period payouts do not reduce current period cover capacity allowing undercollateralized cover to be sold

**Submitted on Aug 12th 2026 at 10:28:41 UTC by @CarlosMB for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88237
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Summary

`CoverOrderAllocator` computes the amount of cover that can be allocated during the current period using the vault's asset balance at the **start of that period**:

```
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);

```

However, `FirelightVault` explicitly allows incidents captured during the **previous period** to be paid during the current period.

Such a payout removes real collateral from the vault and checkpoints the reduced balance, but it does not modify the historical `totalAssetsAt(currentPeriodStart())` value used by `CoverOrderAllocator`.

Consequently, the following sequence is valid:

1. Period `P` begins with `$X` of vault collateral.
2. An unresolved incident from `P-1` is paid during `P`, consuming some or all of that `$X`.
3. `commitAllocation(P)` is called afterwards.
4. `_computeAvailableCapacity()` still counts the collateral that existed at the beginning of `P`, including assets that have already been paid out.
5. New cover for `P` can therefore be committed and settled against collateral that no longer exists.

In the extreme case, a previous-period payout can reduce the vault from `$100` to `$0`, while the allocator still allows `$100` of new current-period cover to be committed.

The protocol therefore permits outstanding cover liabilities to exceed the collateral actually available to satisfy them.

## Vulnerability Details

The issue results from an inconsistency between the accounting horizon used for underwriting and the payout window supported by the vault.

### 1. Current-period capacity uses a historical period-start snapshot

In `CoverOrderAllocator._computeAvailableCapacity()`:

```
function _computeAvailableCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period
) internal view returns (uint256 totalAvailableCapacity) {
    uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);

    CapacityConfig storage config = _getEffectiveCapacityConfig(period);

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

    totalAvailableCapacity = strictCapacity.mulDiv(
        BPS_DENOMINATOR + config.divergenceToleranceBps,
        BPS_DENOMINATOR
    );
}

```

Relevant location:

`contracts/core/CoverOrderAllocator.sol:696-725`

The important line is:

```
$.vault.totalAssetsAt($.vault.currentPeriodStart())

```

The resulting value is used to bound `matchingCapacity` in `_validateMatchingCapacity()`:

```
uint256 maxCapacity = _computeAvailableCapacity($, period);

if (matchingCapacity > maxCapacity)
    revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);

if (totalAllocated > matchingCapacity)
    revert TotalAllocationOverflow(totalAllocated, matchingCapacity);

```

Relevant location:

`contracts/core/CoverOrderAllocator.sol:394-403`

Once accepted, the historical capacity is stored in the commitment:

```
commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;

```

Relevant location:

`contracts/core/CoverOrderAllocator.sol:350-354`

### 2. Previous-period incidents remain payable during the current period

`FirelightVault.payout()` intentionally accepts payouts from both the current period and the immediately preceding period:

```
// Payout can execute during the incident period or the following period.
if (!_isPeriodInPayoutWindow(capturePeriod, _currentPeriod)) {
    revert InvalidCapturePeriod();
}

```

The payout window is:

```
function _isPeriodInPayoutWindow(
    uint256 period,
    uint256 current
) internal pure returns (bool) {
    return current == period || current == period + 1;
}

```

Relevant locations:

`contracts/core/FirelightVault.sol:900-907`

`contracts/core/FirelightVault.sol:1130-1132`

Therefore, after period `P` starts, an incident from `P-1` can legitimately consume collateral that was present when period `P` began.

### 3. The payout reduces live assets but cannot change the historical period-start snapshot

The vault computes the payout using currently payable collateral:

```
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();

...

uint256 activePayableAmount =
    currentActiveAssets +
    capturePeriodWithdrawals +
    nextPeriodWithdrawals;

paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);

```

It then reduces current active assets:

```
_traceTotalAssets.push(
    Time.timestamp(),
    currentActiveAssets - paidFromActive
);

```

and transfers the collateral:

```
IERC20(asset()).safeTransfer(to, paidAmount);

```

Relevant location:

`contracts/core/FirelightVault.sol:909-962`

The new checkpoint is created at the payout timestamp, which is **after** the start of the current period.

It therefore does not alter:

```
totalAssetsAt(currentPeriodStart())

```

As a result, after the payout it is possible to simultaneously have:

```
vault.totalAssets() = 0

vault.totalAssetsAt(vault.currentPeriodStart()) = 100

```

`CoverOrderAllocator` uses the second value.

### 4. New cover can consequently settle using collateral that has already been paid out

`commitAllocation()` validates the submitted allocation against the stale period-start value:

```
_validateMatchingCapacity(
    $,
    currentPeriod,
    matchingCapacity,
    totalAllocated
);

```

and stores it:

```
commit.totalAvailableCapacity = matchingCapacity;

```

Settlement subsequently uses that stored capacity rather than the vault's remaining collateral:

```
uint256 protocolConcentrationCap =
    (
        _getEffectiveProtocolConcentration(
            concHash,
            order.period
        ) * commit.totalAvailableCapacity
    ) / BPS_DENOMINATOR;

```

The order can therefore become `MATCHED` even though the vault assets supposedly backing the cover have already been consumed by the previous-period incident.

The ordering can also be reversed: if `commitAllocation(P)` executes **before** the previous-period payout, the commitment stores the period-start capacity in `commit.totalAvailableCapacity`. A later `P-1` payout can then remove the live collateral before settlement, and `settleCoverOrder()` still does not re-check the vault's live backing. Therefore the issue is not limited to the narrow `payout -> commit -> settle` ordering.

## Impact

The issue can cause **protocol insolvency**.

Consider a configuration with no funded FLB and a 1x effective vault-backed capacity for simplicity.

At the beginning of period `P`:

```
Vault collateral:                  $100
Current-period calculated capacity: $100

```

A valid incident captured during `P-1` is subsequently approved during `P`:

```
Previous-period payout: $100
Vault collateral after payout: $0

```

Nevertheless:

```
totalAssetsAt(currentPeriodStart) = $100

```

A current-period allocation can therefore still be committed for:

```
New cover sold during P: $100
Actual remaining vault backing: $0

```

The resulting state is:

```
Outstanding current-period cover liabilities: $100
Available vault collateral:                     $0

```

This is not merely an incorrect displayed capacity. The inflated value passes the on-chain allocation constraints and can be used to settle real cover orders.

If a legitimate incident subsequently affects that newly-issued cover, the vault can pay zero because `payout()` caps payment by currently payable assets:

```
paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);

```

When:

```
activePayableAmount = 0

```

the result is:

```
paidAmount = 0

```

This zero payout is explicitly permitted by the vault.

Furthermore, `IncidentManager._executePayout()` accepts the returned zero payment:

```
vaultPaidAmount = $.vault.payout(
    receiver,
    vaultRequestedAmount,
    captureTimestamp
);

```

and `approveCurrentAssessment()` subsequently closes the incident:

```
if (vaultPaidAmount > 0)
    incident.vaultPaidAmount = vaultPaidAmount;

incident.status = IncidentStatus.CLOSED;

```

Relevant locations:

`contracts/core/IncidentManager.sol:423-438`

`contracts/core/IncidentManager.sol:672-689`

Therefore a buyer can hold valid, settled cover for which the backing collateral had already been consumed before that cover was committed.

This maps directly to the contest's accepted **Critical — Protocol insolvency** impact.

## Why this does not require a malicious privileged actor

`commitAllocation()` is permissioned, but exploitation does not require the allocator to exceed its documented authority or submit an intentionally invalid allocation.

The allocator can follow the intended workflow and use the maximum capacity returned by the protocol.

The problematic sequence consists exclusively of legitimate operations:

```
1. Existing collateral backs P-1.
2. P begins.
3. P-1 incident is legitimately paid.
4. Matcher calculates P capacity.
5. commitAllocation(P) passes all on-chain checks.
6. Current-period cover orders settle normally.

```

The contract itself reports the already-consumed collateral as available capacity.

The privileged role is therefore not the source of the vulnerability; the incorrect capacity invariant is.

## Proof of Concept

## Proof of Concept

The following PoC uses the repository's production `FirelightVault` and production `CoverOrderAllocator` together in a single end-to-end test. No allocator-vault state is injected through `MockCoverOrderAllocatorVault`.

The timeline is intentionally aligned with the real vault's period numbering:

1. The real vault starts in period `0`.
2. `$100` is deposited during `P0`, so it is already present at the start of `P1`.
3. A legitimate incident is captured during `P1`, making the `$100` part of that incident's exposure cap.
4. During `P1`, a `$100` cover order is created for `P2`.
5. `P2` begins with the same `$100`, so `totalAssetsAt(P2Start) == $100`.
6. The unresolved `P1` incident is paid during `P2`, draining the live vault to `$0`.
7. `totalAssetsAt(P2Start)` remains `$100` because the payout checkpoint is written after `P2Start`.
8. `commitAllocation(P2)` therefore accepts `$100` of capacity and the `P2` order settles as `MATCHED` while `vault.totalAssets() == 0`.

Create:

```
test/critical_previous_period_payout_capacity.js
```

with:

```javascript
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { StandardMerkleTree } = require('@openzeppelin/merkle-tree')

const {
  deployVault,
  computeMarketId
} = require('./setup/fixtures.js')

describe('Critical: previous-period payout is reused as current-period capacity', function () {
  const VAULT_DECIMALS = 6
  const ONE_HUNDRED_VAULT = ethers.parseUnits('100', VAULT_DECIMALS)
  const ONE_HUNDRED_USD = ethers.parseUnits('100', 18)
  const PERIOD_DURATION = 86400

  it('settles P2 cover after a P1 payout has already drained the same P2-start collateral', async () => {
    // -----------------------------------------------------------------------
    // 1. Deploy the real FirelightVault and fund it before P1 starts.
    // -----------------------------------------------------------------------
    const vaultCtx = await deployVault({
      decimals: VAULT_DECIMALS,
      initial_deposit_limit: ethers.parseUnits('1000', VAULT_DECIMALS),
      period_configuration_duration: PERIOD_DURATION
    })

    const vault = vaultCtx.firelight_vault
    const depositor = vaultCtx.users[0]

    // The production vault starts in P0.
    expect(await vault.currentPeriod()).to.equal(0n)

    await vaultCtx.utils.mintAndApprove(ONE_HUNDRED_VAULT, depositor)
    await vault.connect(depositor).deposit(ONE_HUNDRED_VAULT, depositor.address)

    expect(await vault.totalAssets()).to.equal(ONE_HUNDRED_VAULT)

    // -----------------------------------------------------------------------
    // 2. Deploy the real CoverOrderAllocator pointing at the real vault.
    // -----------------------------------------------------------------------
    const [deployer] = await ethers.getSigners()

    async function fundedRandomSigner() {
      const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
      await ethers.provider.send('hardhat_setBalance', [
        wallet.address,
        '0x3635C9ADC5DEA00000'
      ])
      return wallet
    }

    const admin = await fundedRandomSigner()
    const curator = await fundedRandomSigner()
    const allocatorRole = await fundedRandomSigner()
    const configAdmin = await fundedRandomSigner()
    const adminRole = await fundedRandomSigner()
    const premiumCollector = await fundedRandomSigner()
    const firstLossBufferWallet = await fundedRandomSigner()
    const buyer = await fundedRandomSigner()

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const premiumToken = await MockERC20.deploy('MockUSDC', 'mUSDC', 6)

    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.deploy(18, 10n ** 18n) // vault asset = $1

    const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFTFactory,
      [
        'Firelight Cover',
        'FLCOVER',
        '',
        admin.address,
        deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress
      ],
      { kind: 'transparent' }
    )

    const protocol = 'Morpho'
    const chainId = 1
    const market = ethers.encodeBytes32String('market-A')
    const marketId = computeMarketId(chainId, protocol, market)

    const AllocatorFactory = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      AllocatorFactory,
      [{
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 3600,
        premiumTokens: [await premiumToken.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [{
          protocol,
          chainId,
          maxProtocolConcentrationBps: 10000
        }],
        newMarkets: [{ chainId, protocol, market }],
        capacityConfig: {
          // 1x vault-backed capacity and an unfunded FLB.
          minCAR: 12000,
          firstLossBufferToken: await premiumToken.getAddress(),
          firstLossBuffer: firstLossBufferWallet.address,
          effectiveLeverage: 12000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0
        }
      }],
      {
        kind: 'transparent',
        unsafeAllow: ['missing-initializer-call']
      }
    )

    await coverNFT.connect(admin).grantRole(
      ethers.id('MINTER_ROLE'),
      await allocator.getAddress()
    )

    // -----------------------------------------------------------------------
    // 3. Enter P1. The P0 deposit is part of P1's exposure snapshot.
    // -----------------------------------------------------------------------
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.equal(1n)

    const p1Start = await vault.currentPeriodStart()
    expect(await vault.totalAssetsAt(p1Start)).to.equal(ONE_HUNDRED_VAULT)

    // Capture a legitimate P1 incident.
    const p1CaptureTimestamp = await time.latest()

    // While in P1, create a legitimate $100 cover order targeting P2.
    await premiumToken.mint(buyer.address, ethers.parseUnits('1', 6))
    await premiumToken.connect(buyer).approve(
      await allocator.getAddress(),
      ethers.MaxUint256
    )

    await allocator
      .connect(curator)
      .createCoverOrder(
        buyer.address,
        buyer.address,
        'beneficiary',
        await premiumToken.getAddress(),
        [{
          marketId,
          coverRateAnnual: 500,
          coverAmount: ONE_HUNDRED_USD
        }],
        0
      )

    const createdOrder = await allocator.getCoverOrder(0)
    expect(createdOrder.period).to.equal(2n)

    // -----------------------------------------------------------------------
    // 4. Enter P2, then pay the unresolved P1 incident.
    // -----------------------------------------------------------------------
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    expect(await vault.currentPeriod()).to.equal(2n)

    const p2Start = await vault.currentPeriodStart()

    expect(await vault.totalAssetsAt(p2Start)).to.equal(ONE_HUNDRED_VAULT)
    expect(await vault.totalAssets()).to.equal(ONE_HUNDRED_VAULT)

    // A P1 incident is intentionally still payable during P2.
    await expect(
      vault
        .connect(vaultCtx.payout_signer)
        .payout(
          vaultCtx.payout_receiver.address,
          ONE_HUNDRED_VAULT,
          p1CaptureTimestamp
        )
    )
      .to.emit(vault, 'PayoutExecuted')
      .withArgs(
        vaultCtx.payout_receiver.address,
        ONE_HUNDRED_VAULT,
        ONE_HUNDRED_VAULT,
        p1CaptureTimestamp
      )

    // Live backing is gone.
    expect(await vault.totalAssets()).to.equal(0n)

    // BUG PRECONDITION: P2's historical capacity input still counts the spent $100.
    expect(await vault.totalAssetsAt(p2Start)).to.equal(ONE_HUNDRED_VAULT)

    // -----------------------------------------------------------------------
    // 5. Commit and settle P2 after the collateral has already left the vault.
    // -----------------------------------------------------------------------
    const allocation = [marketId, ONE_HUNDRED_USD]
    const tree = StandardMerkleTree.of(
      [[0n, [allocation]]],
      ['uint256', '(bytes32,uint256)[]']
    )

    // BUG: _computeAvailableCapacity() reads totalAssetsAt(P2Start) == $100.
    await allocator
      .connect(allocatorRole)
      .commitAllocation(
        2,
        tree.root,
        ONE_HUNDRED_USD,
        ONE_HUNDRED_USD
      )

    const commitment = await allocator.getAllocationCommitment(2)
    expect(commitment.totalAvailableCapacity).to.equal(ONE_HUNDRED_USD)
    expect(await vault.totalAssets()).to.equal(0n)

    // Settlement trusts the stored commitment and does not re-check live backing.
    await allocator
      .connect(allocatorRole)
      .settleCoverOrder(
        0,
        [{
          marketId,
          allocatedCover: ONE_HUNDRED_USD
        }],
        tree.getProof(0)
      )

    const settledOrder = await allocator.getCoverOrder(0)

    expect(settledOrder.status).to.equal(1n) // MATCHED
    expect(settledOrder.allocatedCoverAmount).to.equal(ONE_HUNDRED_USD)

    // Final vulnerable state.
    expect(await vault.totalAssets()).to.equal(0n)
  })
})
```

Run:

```
npx hardhat test test/critical_previous_period_payout_capacity.js
```

The important final state is:

```
Current-period settled cover:             $100
commit.totalAvailableCapacity:            $100
Live vault assets backing that cover:       $0
```

The `$100` counted by `commitAllocation()` is the same collateral that the previous-period payout has already transferred out of the vault.

## Recommended Mitigation

The protocol must prevent collateral that is still exposed to unresolved liabilities from `P-1` from simultaneously being counted as backing for new `P` cover.

Simply subtracting previous-period payouts when `commitAllocation()` runs is **not sufficient**, because the unsafe ordering can also be:

```
commitAllocation(P)
    -> previous-period payout removes collateral
    -> settleCoverOrder(P) uses the already-stored capacity
```

A robust fix should therefore enforce the invariant for the entire matching/settlement lifecycle, not only at commit time.

Possible approaches include:

1. **Reserve unresolved previous-period exposure from current-period capacity.** Compute the amount of vault collateral that can still be consumed by `P-1` incidents and subtract that reserved amount from the collateral available to underwrite `P`.
2. **Block current-period commitment/settlement while previous-period liabilities remain unresolved.** Since `FirelightVault` already tracks active incidents across the current and previous periods, underwriting could be prevented until the previous-period liability reaches a terminal state.
3. **Invalidate or resize outstanding commitments when a previous-period payout occurs.** If concurrent underwriting is required, a payout that reduces live backing must also prevent pre-existing commitments from settling above the newly available capacity.

The core invariant should be:

```
collateral counted toward current-period capacity
must not simultaneously remain spendable against previous-period liabilities
```

Any fix must cover both `commitAllocation()` and `recommitAllocation()`, and must also ensure that an already-created commitment cannot remain settleable after a later previous-period payout removes its backing.

## Additional Notes

This issue is distinct from the disclosed first-loss-buffer custody issue because the PoC requires no FLB balance at all and relies exclusively on vault collateral.

It is also distinct from cumulative/duplicate assessment-loss issues. The two losses can correspond to separate legitimate cover periods and independently valid incidents.

No oracle depeg, malicious token, malicious privileged role, duplicate claim, or abnormal ERC-20 behavior is required.
