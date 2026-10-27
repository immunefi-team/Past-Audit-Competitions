# 88620 sc medium cover capacity in computeavailablecapacity is sized on a stale period start asset snapshot so a prior period payout lets the allocator commit cover against collateral that has&#x20;

**Submitted on Aug 14th 2026 at 07:40:46 UTC by @enowk for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88620
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator._computeAvailableCapacity` mixes two clocks: the first-loss buffer is read **live** via `balanceOf`, but the staked collateral — the dominant term — is read from the **period-start checkpoint** via `vault.totalAssetsAt(vault.currentPeriodStart())`. A payout for a *prior* period's incident may legitimately execute during the current period, because the payout window is `{capturePeriod, capturePeriod + 1}`. That permanently removes assets from the vault but does not move the current period's start checkpoint, so for the remainder of the period the allocator sizes cover against collateral that has already left. `_validateMatchingCapacity` — the on-chain capital adequacy guard applied by both `commitAllocation` and `recommitAllocation` — passes on the inflated figure. On mainnet this lets cover be sold above what the vault can actually back, by the payout amount multiplied by the leverage ratio, with the resulting shortfall landing on cover beneficiaries as a reduced payout.

## Vulnerability Details

### Two clocks in one function

[`CoverOrderAllocator.sol#L696-L725`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L696-L725):

```solidity
function _computeAvailableCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period
) internal view returns (uint256 totalAvailableCapacity) {
    uint256 assetPriceUSD = $.priceFeedAdapter.getPrice($.maxPriceAge);      // LIVE
    CapacityConfig storage config = _getEffectiveCapacityConfig(period);

    uint256 firstLossBufferUSDCanonical = Decimals.convert(
        config.firstLossBufferToken.balanceOf(config.firstLossBuffer),        // LIVE
        ...
    );
    uint256 totalAssetsCanonical = Decimals.convert(
        $.vault.totalAssetsAt($.vault.currentPeriodStart()),                  // PERIOD-START SNAPSHOT
        ...
    );
    uint256 stakedAssetsValueUSDCanonical = totalAssetsCanonical.mulDiv(assetPriceUSD, 10 ** $.priceFeedDecimals);
    uint256 availableCollateralCanonical = firstLossBufferUSDCanonical + stakedAssetsValueUSDCanonical;

    uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
    totalAvailableCapacity = strictCapacity.mulDiv(BPS_DENOMINATOR + config.divergenceToleranceBps, BPS_DENOMINATOR);
}
```

Both the price leg and the buffer leg track reality in real time. The staked-collateral leg — which in normal operation is the largest input — is frozen at the period boundary.

### Why the snapshot goes stale within the period

`payout` pushes its new total at `Time.timestamp()` ([`FirelightVault.sol#L952`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L952)):

```solidity
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

Because `totalAssetsAt` resolves via `upperLookupRecent(timestamp)` ([`FirelightVault.sol#L456-L458`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L456-L458)), a checkpoint pushed *after* the period start is invisible to a lookup *at* the period start. The snapshot therefore keeps reporting the pre-payout figure until the next period boundary.

The payout that causes this is entirely legitimate. `payout` explicitly permits execution one period after capture ([`FirelightVault.sol#L1130-L1132`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1130-L1132)):

```solidity
function _isPeriodInPayoutWindow(uint256 period, uint256 current) internal pure returns (bool) {
    return current == period || current == period + 1;
}
```

So an incident captured in period `C-1` can, by design, be paid during period `C` — after period `C`'s collateral snapshot has already been taken.

### Why this is specific to payouts and not to withdrawals

Withdrawal *requests* also reduce `totalAssets()` mid-period, and the snapshot ignores those too. That is correct behaviour: requested-but-unclaimed capital is still slashable, and `payout` explicitly adds `capturePeriodWithdrawals` (and, in the following period, `nextPeriodWithdrawals`) to `activePayableAmount` ([`FirelightVault.sol#L914-L926`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L914-L926)). Counting that capital as backing is right.

Assets paid out are simply gone. Counting them is not. The snapshot cannot distinguish the two cases, and no compensating adjustment exists anywhere in the allocator.

Note also the direction of the error is one-sided. Mid-period *deposits* are likewise invisible to the snapshot, which is conservative and safe. Only the payout case makes the figure too high.

### The stale figure is the on-chain solvency guard's only input

[`CoverOrderAllocator.sol#L394-L403`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L394-L403):

```solidity
function _validateMatchingCapacity(
    CoverOrderAllocatorStorage storage $,
    uint256 period,
    uint256 matchingCapacity,
    uint256 totalAllocated
) private view {
    uint256 maxCapacity = _computeAvailableCapacity($, period);
    if (matchingCapacity > maxCapacity) revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
    if (totalAllocated > matchingCapacity) revert TotalAllocationOverflow(totalAllocated, matchingCapacity);
}
```

This is the protocol's capital adequacy check — `minCAR` is literally the minimum Capital Adequacy Ratio. It is reached from both write paths:

* `commitAllocation` at [line 348](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L348)
* `recommitAllocation` at [line 378](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L378), whose own comment says it re-validates "against fresh capacity … so recommit can capture recovered price/FLB and stays bound to the period's real collateral within tolerance." The intent to bind to *real* collateral is explicit; the staked leg does not deliver it.

The stored `commit.totalAvailableCapacity` then propagates: `_settleCoverOrder` derives the per-protocol concentration caps from it ([line 507](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L507)), so the inflation scales those caps too.

### Nothing constrains the ordering of payout and commit

* The allocator contains **no reference to incidents at all** — an open or recently-paid incident neither blocks nor forces a commit.
* `commitAllocation`'s only timing constraint is `graceExpiresAt < vault.currentPeriodEnd()` ([line 346](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L346)), which merely requires the commit to happen early enough in the period — not before any payout.
* `recommitAllocation` is available for the whole period until the first settlement, and re-derives capacity from the same stale input, so an ordinary mid-period recommit reproduces the condition even if the original commit preceded the payout.

### Failure sequence

No role misbehaves at any point. Every action is the intended operation of the system.

{% stepper %}
{% step %}

## An incident is captured

An incident is captured in period `C-1`. Its payout window is `{C-1, C}`.
{% endstep %}

{% step %}

## Period `C` starts

`totalAssetsAt(currentPeriodStart())` records the collateral — 1,000 in the PoC.
{% endstep %}

{% step %}

## A prior-period payout executes

During period `C`, the assessment is approved and `payout` transfers 400 out of the vault. Real active assets are now 600; the period-start snapshot still reads 1,000.
{% endstep %}

{% step %}

## Cover is committed against stale collateral

`ALLOCATOR_ROLE` commits (or `CONFIG_ADMIN_ROLE` recommits) period `C` cover. `_validateMatchingCapacity` computes the ceiling from 1,000, not 600, and accepts a commitment above what the vault can back. **The attached PoC reads the ceiling straight out of the contract's own `MatchingCapacityOverflow` revert: 2,000 canonical USD, when honest capacity is 1,200.**
{% endstep %}

{% step %}

## The cover becomes undercollateralised

Cover sold for period `C` is undercollateralised by the payout amount times the leverage multiplier. If an incident then hits period `C`, the shortfall passes to beneficiaries.
{% endstep %}
{% endstepper %}

### Known-issue check

Three published items are adjacent. None covers this.

* **Immunefi A5** — *"If the underlying staked asset decreases in USD value, available capital and the Capital Adequacy Ratio decrease… may cause shortfalls or undercollateralization."* That is the **price** leg, and the price leg **is** read live via `priceFeedAdapter.getPrice`. This report concerns the **quantity** leg being stale, which A5 does not describe. A5 does confirm that the program treats undercollateralisation as a real impact rather than an accepted design property.
* **Immunefi A6 / 0xMacro B1** — *"Aggregate claims are not capped on-chain against total cover sold."* That is claims-versus-cover; this is cover-versus-collateral. Opposite side of the ledger, different guard.
* **0xMacro B2 / Immunefi A4** — the first-loss buffer *"can be funded to inflate capacity at commitment and then withdrawn."* Same family (a capacity input diverging from reality) but a different input, a different mechanism, and it requires operator misbehaviour. Here the divergence is created by a **correct, expected payout** and requires no misbehaviour at all. Note that B2's input is the one read **live**; the input read stale went unexamined.
* 0xMacro **M-1** (unbounded `effectiveLeverage`) is fixed at this commit — `effectiveLeverage <= MAX_LEVERAGE_FACTOR * minCAR` with `MAX_LEVERAGE_FACTOR = 5` — and is unrelated. Macro's report quotes the `strictCapacity` / `divergenceToleranceBps` lines of this exact function while analysing M-1, but contains no reference anywhere to `totalAssetsAt` or the period-start snapshot.
* Nothing in the OpenZeppelin Phase 1 audit applies — `CoverOrderAllocator` is a new Phase 2 contract.

### Recommended fix

Read the collateral leg live and cap it by the snapshot:

```solidity
uint256 stakedNow     = $.vault.totalAssets();
uint256 stakedAtStart = $.vault.totalAssetsAt($.vault.currentPeriodStart());
uint256 staked        = Math.min(stakedNow, stakedAtStart);   // never count what has left
```

`min` preserves the intent of the period-start snapshot — mid-period deposits must not inflate the current period's capacity — while ensuring assets already paid out cannot back new cover.

Note that a naive switch to a fully live `totalAssets()` would be wrong in the other direction: withdrawal requests reduce it while that capital is still slashable, so a live-only read would understate capacity. `min` avoids both errors.

Alternatively, have `payout` maintain a per-period `paidOutThisPeriod` accumulator on the vault and subtract it in `_computeAvailableCapacity`.

## Impact Details

**Selected in-scope impact: Protocol insolvency.**

The protocol's on-chain capital adequacy check can be satisfied while actual capital adequacy is below the configured `minCAR` floor. Cover in force for the period exceeds the collateral backing it, which is an asset/liability mismatch created by the contract itself rather than by market movement.

**Magnitude.** The capacity overstatement is:

```
overstatement  =  paidFromActive  ×  assetPriceUSD  ×  (effectiveLeverage / minCAR)  ×  (1 + divergenceToleranceBps / BPS)
```

* `paidFromActive` is the portion of the payout drawn from active assets ([`FirelightVault.sol#L952`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L952)). The remainder of a payout is drawn from withdrawal buckets and does not move `totalAssets()`, so the overstatement tracks `paidFromActive`, not the full `paidAmount`. In the PoC there are no pending withdrawals, so the two coincide at 400.
* `effectiveLeverage / minCAR` is bounded at **5×** by `MAX_LEVERAGE_FACTOR` ([`CoverOrderAllocator.sol#L82`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L82), enforced at [line 665](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L665)), and is widened further by the divergence tolerance.

In the attached end-to-end PoC, a 400-asset payout at the fixture's 2.0× multiplier lifts the contract's own accepted ceiling from an honest 1,200 to 2,000, and a commitment of 1,800 against 600 of real assets is accepted. At the 5× ceiling the same 400 payout would overstate capacity by 2,000.

**Realised loss.** The shortfall materialises when an incident hits period `C`. Cover was sold on the promise of the inflated capacity, but `payout` for a period-`C` incident is limited by `activePayableAmount`, which uses the **live** `currentActiveAssets` ([`FirelightVault.sol#L926-L927`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927)). The vault therefore cannot deliver what the cover implies, and beneficiaries receive a reduced payout. The same stale snapshot also inflates the period-`C` exposure cap, since `payout` reads `totalAssetsAt(capturePeriodStart)` at [line 911](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911) — the cap says 1,000 while the vault holds 600, so the overstatement is visible on both sides of the ledger.

**Duration.** One period per occurrence: the snapshot corrects itself at the next period boundary. It recurs whenever a payout lands in the following period, which is an explicitly supported and expected flow rather than an edge case.

**Preconditions, stated honestly.** An incident whose payout executes in the period after capture — permitted by design — plus a commit or recommit after that payout in the same period. Nothing enforces or forbids that ordering. No role exceeds its authority, no attacker is required, and no market movement is involved; this is the in-scope contract defeating its own on-chain solvency guard while every dependency behaves correctly. That distinguishes it from the program's trusted-operator carve-out, which excludes reports of roles acting within their intended authority but preserves reports demonstrating an intended protocol constraint being bypassed.

**Severity calibration.** "Protocol insolvency" is the only entry in the program's in-scope impact list that describes this harm; the Low entry ("contract fails to deliver promised returns, but doesn't lose value") is inaccurate, because value *is* lost by beneficiaries. That said, I assess the practical severity as **Medium** rather than Critical and would not dispute that rating: the off-chain matching engine computes its own capacity and may catch the divergence independently, the window is bounded to one period, the multiplier is capped at 5×, and the resulting shortfall reaches beneficiaries as a reduced payout rather than as direct theft. The report is filed against the impact entry that fits; the severity is the program's to set.

## References

**Vulnerable code (commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, branch `v1_audit_ready`):**

* [`CoverOrderAllocator.sol#L696-L725`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L696-L725) — `_computeAvailableCapacity`, the mixed-clock read (root cause)
* [`CoverOrderAllocator.sol#L710-L715`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L710-L715) — the stale `totalAssetsAt(currentPeriodStart())` leg specifically
* [`CoverOrderAllocator.sol#L394-L403`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L394-L403) — `_validateMatchingCapacity`, the defeated solvency guard
* [`CoverOrderAllocator.sol#L348`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L348) and [`#L378`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L378) — the two write paths that reach it
* [`CoverOrderAllocator.sol#L82`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol#L82) — `MAX_LEVERAGE_FACTOR = 5`, bounding the amplification
* [`FirelightVault.sol#L952`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L952) — `payout` pushing its checkpoint at `Time.timestamp()`, after the period start
* [`FirelightVault.sol#L1130-L1132`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1130-L1132) — the two-period payout window that makes a prior-period payout legal mid-period
* [`FirelightVault.sol#L911`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L911) and [`#L926-L927`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L926-L927) — the inflated exposure cap versus the live payable amount
* [`FirelightVault.sol#L456-L458`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L456-L458) — `totalAssetsAt` resolving via `upperLookupRecent`
* [`FirelightVault.sol#L427-L429`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L427-L429) — `totalAssets()` net of `pendingWithdrawAssets`

**Prior audits referenced:**

* 0xMacro, *Firelight A-1*, 2026-07-29 — M-1 (unbounded leverage, fixed at this commit) and accepted risk B2 (first-loss buffer). Neither addresses the staked-collateral snapshot.
* Immunefi published known issues A4, A5 and A6.

**Protocol documentation:**

* Core Concepts — Periods and Coverage Cycles: capacity aligns to period boundaries; periods are currently 1 day and extend to 30-day cover periods at the feature-complete launch.
* Protocol Architecture — Claims and Liquidation: the payout waterfall and the two-period claim window.

## Proof of Concept

A runnable Hardhat test is provided below. It runs the **real `CoverOrderAllocator` against the real `FirelightVault`** at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (branch `v1_audit_ready`), using the repository's own `test/setup/fixtures.js`. There is no mock vault and no hand-set vault state: the capacity divergence is produced entirely by an ordinary `payout` for a prior period's incident, executed inside the contract's own payout window.

The only mocks are the peripherals the allocator needs in any test — `MockERC20` for the premium/buffer tokens and `MockAggregatorV3` for the price feed, both taken from the repository's existing fixture. The price feed is pinned at $1.00 so the price leg is neutral and capacity is driven purely by the staked-asset quantity leg, and the first-loss buffer wallet is left empty for the same reason.

### How to run

Save the file as `test/zz-audit-poc-stale-capacity-e2e.js` in the `audit-comp-firelight` repository and run:

```bash
npx hardhat test test/zz-audit-poc-stale-capacity-e2e.js
```

### Actual output

```
  AUDIT POC (E2E): allocator sizes capacity on pre-payout vault assets
      live totalAssets: 600.0   period-start snapshot still: 1000.0
      allocator ceiling: 2000.0 (honest ceiling would be 1200.0)
      COMMITTED 1800.0 of cover against 600.0 of real assets (honest capacity 1200.0)
    ✔ accepts cover far above real backing after a prior-period payout (16693ms)
    ✔ CONTROL: with no payout, the allocator ceiling matches real backing (337ms)


  2 passing (17s)
```

Test 1 demonstrates the vulnerability. Test 2 is a control: with no payout, the allocator's ceiling is exactly the honest 2,000 against 1,000 of real assets, proving the inflated ceiling in test 1 is caused by the payout and not by the fixture.

### Step-by-step explanation

Vault-asset amounts use 6 decimals; capacity figures are in canonical 18-decimal USD. The fixture's capacity multiplier is `effectiveLeverage / minCAR = 24000 / 12000 = 2.0×`, with `divergenceToleranceBps = 0` so the stale-input effect is isolated.

{% stepper %}
{% step %}

## Capital arrives

Alice deposits 1,000 assets into the real vault, which checkpoints `_traceTotalAssets`.
{% endstep %}

{% step %}

## Period `C-1` begins and an incident is captured in it

The capture timestamp is recorded.

*Assertion:* `totalAssetsAt(currentPeriodStart()) == 1000` — the capture period is properly collateralised, so the later payout is not capped to zero.
{% endstep %}

{% step %}

## Period `C` begins

This is the period whose cover will be sized against the snapshot taken here.

*Assertions:* `C == capturePeriod + 1`, confirming we are inside the contract's own payout window `{capturePeriod, capturePeriod + 1}`; and `totalAssetsAt(periodStartC) == 1000`.
{% endstep %}

{% step %}

## The real `CoverOrderAllocator` is deployed against the real vault

It uses `firstLossBuffer` pointing at an empty wallet.

*Assertion:* `usdc.balanceOf(firstLossBufferWallet) == 0`, so capacity is `stakedAssets × 2.0` and nothing else.
{% endstep %}

{% step %}

## A payout for the prior period's incident executes

The payout remains legally inside its window.

`payout(payout_receiver, 400, captureTs)` transfers 400 assets out of the vault.

*Assertions:* `totalAssets() == 600` — the real assets dropped — while `totalAssetsAt(periodStartC) == 1000` — the capacity input did not move.
{% endstep %}

{% step %}

## The allocator's own capacity ceiling is read directly out of the contract

Rather than inferring it, the test commits a deliberately excessive value and reads the ceiling from the revert, since `MatchingCapacityOverflow(matchingCapacity, maxCapacity)` carries it as an argument.

*Assertion:* the revert args are `(2000e18 + 1, 2000e18)` — the contract states its own ceiling is **2,000 canonical USD**, computed from the stale 1,000, when honest capacity against the live 600 would be **1,200**.
{% endstep %}

{% step %}

## A commitment between the honest and the stale ceiling is accepted

`commitAllocation(C, root, 1800e18, 1800e18)` succeeds — 1,800 of cover committed against 600 of real assets.

*Assertion:* `getAllocationCommitment(C).totalDeclaredAllocated == 1800e18`.
{% endstep %}
{% endstepper %}

**Result: the on-chain capital adequacy guard accepted cover 3.0× the real collateral, where the configured design intent was 2.0×.**

Step 6 is the load-bearing one: it is the contract itself reporting the inflated ceiling, so the finding does not depend on the test's own arithmetic being correct.

### Test source

```javascript
// AUDIT POC (END-TO-END) — cover capacity is sized on stale (pre-payout) vault assets.
//
// The REAL CoverOrderAllocator is pointed at the REAL FirelightVault. No mock vault, no
// hand-set state: the divergence is produced by an ordinary payout for a prior period's
// incident, executed inside the payout window {capturePeriod, capturePeriod + 1}.
//
// _computeAvailableCapacity mixes two clocks:
//   firstLossBuffer -> balanceOf(...)                            LIVE
//   staked assets   -> vault.totalAssetsAt(currentPeriodStart())  PERIOD-START SNAPSHOT
const { expect } = require('chai')
const { ethers, upgrades } = require('hardhat')
const { time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')

const DECIMALS = 6
const unit = (n) => ethers.parseUnits(String(n), DECIMALS)
const usd = (n) => ethers.parseUnits(String(n), 18) // canonical USD decimals

const randomSigner = async () => {
  const wallet = ethers.Wallet.createRandom().connect(ethers.provider)
  await ethers.provider.send('hardhat_setBalance', [wallet.address, '0x3635C9ADC5DEA00000'])
  return wallet
}

describe('AUDIT POC (E2E): allocator sizes capacity on pre-payout vault assets', function () {
  this.timeout(600000)

  it('accepts cover far above real backing after a prior-period payout', async () => {
    const {
      firelight_vault: vault, utils, users, payout_signer, payout_receiver
    } = await deployVault()

    const alice = users[0]
    await utils.mintAndApprove(unit(20000), alice)

    // === 1. Capital arrives and is checkpointed. ===
    await vault.connect(alice).deposit(unit(1000), alice.address)

    // === 2. Period C-1 begins. An incident is captured in it. ===
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    const capturePeriod = await vault.currentPeriod()
    const captureTs = BigInt(await time.latest())
    expect(await vault.totalAssetsAt(await vault.currentPeriodStart())).to.equal(unit(1000))

    // === 3. Period C begins. This period's cover is sized against the snapshot taken here. ===
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    const C = await vault.currentPeriod()
    expect(C, 'payout window is {capturePeriod, capturePeriod + 1}').to.equal(capturePeriod + 1n)

    const periodStartC = await vault.currentPeriodStart()
    expect(await vault.totalAssetsAt(periodStartC), 'period-C snapshot').to.equal(unit(1000))

    // === 4. Deploy the REAL allocator against the REAL vault. ===
    const deployer = (await ethers.getSigners())[0]
    const admin = await randomSigner()
    const curator = await randomSigner()
    const allocatorRole = await randomSigner()
    const configAdmin = await randomSigner()
    const adminRole = await randomSigner()
    const premiumCollector = await randomSigner()
    const firstLossBufferWallet = await randomSigner()

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)
    const usdt = await MockERC20.connect(deployer).deploy('MockUSDT', 'mUSDT', 6)

    // Price feed pinned at $1.00 so the price leg is neutral and capacity is driven
    // purely by the staked-asset quantity leg (the one being read stale).
    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.connect(deployer).deploy(18, 10n ** 18n)

    const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFTFactory,
      ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' }
    )

    const Factory = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Factory,
      [{
        vault: await vault.getAddress(),                 // <-- the REAL FirelightVault
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 3600,
        premiumTokens: [await usdc.getAddress(), await usdt.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          { protocol: 'Morpho', chainId: 1, maxProtocolConcentrationBps: 6000 }
        ],
        newMarkets: [
          { chainId: 1, protocol: 'Morpho', market: ethers.encodeBytes32String('market-A') }
        ],
        capacityConfig: {
          minCAR: 12000,               // 1.2x floor
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: firstLossBufferWallet.address,  // deliberately empty
          effectiveLeverage: 24000,    // capacity multiplier = 24000/12000 = 2.0x
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0    // no tolerance: isolates the stale-input effect
        }
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
    )

    // Sanity: the buffer leg contributes nothing, so capacity == stakedAssets * 2.0.
    expect(await usdc.balanceOf(firstLossBufferWallet.address)).to.equal(0n)

    // === 5. A payout for the PRIOR period's incident executes now — still inside its window. ===
    await vault.connect(payout_signer).payout(payout_receiver.address, unit(400), captureTs)

    const liveAssets = await vault.totalAssets()
    const snapshot = await vault.totalAssetsAt(periodStartC)

    expect(liveAssets, 'real assets dropped').to.equal(unit(600))
    expect(snapshot, 'capacity input is unchanged').to.equal(unit(1000))

    console.log(`      live totalAssets: ${ethers.formatUnits(liveAssets, DECIMALS)}   ` +
                `period-start snapshot still: ${ethers.formatUnits(snapshot, DECIMALS)}`)

    // === 6. Read the allocator's own capacity ceiling out of its revert. ===
    // MatchingCapacityOverflow(matchingCapacity, maxCapacity) carries maxCapacity.
    const honestCapacity = usd(1200)   // 600 live  * 2.0
    const staleCapacity  = usd(2000)   // 1000 stale * 2.0

    await expect(
      allocator.connect(allocatorRole).commitAllocation(C, ethers.id('probe'), usd(1), staleCapacity + 1n)
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')
      .withArgs(staleCapacity + 1n, staleCapacity)

    console.log(`      allocator ceiling: ${ethers.formatUnits(staleCapacity, 18)} ` +
                `(honest ceiling would be ${ethers.formatUnits(honestCapacity, 18)})`)

    // === 7. Commit cover between the honest and the stale ceiling. It is accepted. ===
    const oversold = usd(1800)
    expect(oversold).to.be.gt(honestCapacity)
    expect(oversold).to.be.lt(staleCapacity)

    await allocator.connect(allocatorRole).commitAllocation(C, ethers.id('root'), oversold, oversold)

    const commit = await allocator.getAllocationCommitment(C)
    expect(commit.totalDeclaredAllocated, 'cover committed above real backing').to.equal(oversold)

    console.log(`      COMMITTED ${ethers.formatUnits(oversold, 18)} of cover against ` +
                `${ethers.formatUnits(liveAssets, DECIMALS)} of real assets ` +
                `(honest capacity ${ethers.formatUnits(honestCapacity, 18)})`)
  })

  it('CONTROL: with no payout, the allocator ceiling matches real backing', async () => {
    // Proves the inflated ceiling is caused by the payout, not by the fixture.
    const { firelight_vault: vault, utils, users } = await deployVault()
    const alice = users[0]
    await utils.mintAndApprove(unit(20000), alice)

    await vault.connect(alice).deposit(unit(1000), alice.address)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    await time.increaseTo((await vault.currentPeriodEnd()) + 1n)
    const C = await vault.currentPeriod()

    const deployer = (await ethers.getSigners())[0]
    const admin = await randomSigner()
    const curator = await randomSigner()
    const allocatorRole = await randomSigner()
    const configAdmin = await randomSigner()
    const adminRole = await randomSigner()
    const premiumCollector = await randomSigner()
    const firstLossBufferWallet = await randomSigner()

    const MockERC20 = await ethers.getContractFactory('MockERC20')
    const usdc = await MockERC20.connect(deployer).deploy('MockUSDC', 'mUSDC', 6)
    const MockAggregator = await ethers.getContractFactory('MockAggregatorV3')
    const priceFeed = await MockAggregator.connect(deployer).deploy(18, 10n ** 18n)
    const CoverNFTFactory = await ethers.getContractFactory('CoverNFT')
    const coverNFT = await upgrades.deployProxy(
      CoverNFTFactory,
      ['Firelight Cover', 'FLCOVER', '', admin.address, deployer.address, ethers.ZeroAddress, ethers.ZeroAddress],
      { kind: 'transparent' }
    )

    const Factory = await ethers.getContractFactory('CoverOrderAllocator')
    const allocator = await upgrades.deployProxy(
      Factory,
      [{
        vault: await vault.getAddress(),
        premiumCollector: premiumCollector.address,
        coverNFT: await coverNFT.getAddress(),
        priceFeedAdapter: await priceFeed.getAddress(),
        maxPriceAge: 7 * 24 * 3600,
        premiumTokens: [await usdc.getAddress()],
        admin: admin.address,
        adminRole: adminRole.address,
        curatorRole: curator.address,
        allocatorRole: allocatorRole.address,
        configAdminRole: configAdmin.address,
        initialProtocolConcentrations: [
          { protocol: 'Morpho', chainId: 1, maxProtocolConcentrationBps: 6000 }
        ],
        newMarkets: [
          { chainId: 1, protocol: 'Morpho', market: ethers.encodeBytes32String('market-A') }
        ],
        capacityConfig: {
          minCAR: 12000,
          firstLossBufferToken: await usdc.getAddress(),
          firstLossBuffer: firstLossBufferWallet.address,
          effectiveLeverage: 24000,
          minOrderMarketCoverAmount: 1,
          divergenceToleranceBps: 0
        }
      }],
      { kind: 'transparent', unsafeAllow: ['missing-initializer-call'] }
    )

    // No payout. Live assets == snapshot == 1000, so the ceiling is the honest 2000.
    expect(await vault.totalAssets()).to.equal(unit(1000))
    await expect(
      allocator.connect(allocatorRole).commitAllocation(C, ethers.id('probe'), usd(1), usd(2000) + 1n)
    ).to.be.revertedWithCustomError(allocator, 'MatchingCapacityOverflow')
      .withArgs(usd(2000) + 1n, usd(2000))
  })
})
```
