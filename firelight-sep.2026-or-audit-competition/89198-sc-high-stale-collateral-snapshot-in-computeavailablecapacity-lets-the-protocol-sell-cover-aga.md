# 89198 sc high stale collateral snapshot in computeavailablecapacity lets the protocol sell cover against funds it has already paid out driving car to 0 17 against the enforced 1 2 floor

**Submitted on Aug 17th 2026 at 19:20:52 UTC by @webrainsec for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89198
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator._computeAvailableCapacity` values staked collateral at a period-start snapshot, while `FirelightVault.payout` writes its post-payout checkpoint at `Time.timestamp()`. A payout executed inside a period therefore never moves that period's snapshot, and the payout window is `{capturePeriod, capturePeriod + 1}`, so payouts landing mid-period are routine flow rather than an edge case. For the rest of that period the on-chain solvency check computes capacity from collateral the protocol has already paid out, and sells new cover against it. On a Flare mainnet fork of the live vault, $90,000,000 of cover settles against $7,687,143 of remaining collateral, a Capital Adequacy Ratio of 0.17 against the 1.2 floor the contract enforces.

## Vulnerability Details

Capacity is the protocol's on-chain solvency constraint (`CoverOrderAllocator.sol:719-724`):

```solidity
uint256 strictCapacity = availableCollateralCanonical.mulDiv(config.effectiveLeverage, config.minCAR);
totalAvailableCapacity = strictCapacity.mulDiv(BPS_DENOMINATOR + config.divergenceToleranceBps, BPS_DENOMINATOR);
```

`config.minCAR` is the Capital Adequacy Ratio floor itself: `MIN_CAR_BPS = 12_000` is the documentation's 1.2 floor, and `_setCapacityConfig` reverts below it (`:662`). With `CAR = Available Capital / Solvency Capital Requirement` and the SCR's *"coverage-over-leverage component"*, the ceiling above is exactly `CAR >= minCAR / (1 + divergenceTolerance)`.

The two collateral legs of that ceiling are read at different times (`:704-717`):

```solidity
config.firstLossBufferToken.balanceOf(config.firstLossBuffer)   // :705  live
$.vault.totalAssetsAt($.vault.currentPeriodStart())             // :711  period-start snapshot
```

`FirelightVault.payout` checkpoints the post-payout balance at `:952`:

```solidity
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

At `Time.timestamp()`, never at the period start, so `upperLookupRecent(currentPeriodStart())` returns the pre-payout value for the remainder of the period. One waterfall spends both legs (`IncidentManager.sol:637`, `:658`, `:684`), so the ceiling falls by the buffer's contribution and does not fall by the vault's.

The sale happens inside the period, not at the boundary. `commitAllocation` requires `commitmentPeriod == currentPeriod` (`:343`) and rejects a commit whose settlement grace would run past the period end (`:346`). The ceiling is therefore computed at an arbitrary block inside the period while the staked leg stays frozen at that period's start.

Nothing corrects it before the period ends. `_computeAvailableCapacity` is reached only from `_validateMatchingCapacity` (`:400`), via `commitAllocation` (`:348`) and `recommitAllocation` (`:378`). Settlement never re-checks capacity, and `recommitAllocation` is available only *"if no orders have been settled yet"* (`ICoverOrderAllocator.sol:564`). The divergence tolerance does not absorb it either: `:75-78` scopes the tolerance to *"the price/FLB drift between the off-chain matcher snapshot and commit inclusion"* and caps it at 10%, while a payout can remove 100% of staked assets in one transaction.

**Remediation.** Have `payout` accumulate the `paidFromActive` it already computes at `:937`/`:941` into a per-period total, and value the staked leg at `totalAssetsAt(currentPeriodStart())` minus that total, saturating at zero. That keeps the period-start basis for staker flows and corrects only the movement the protocol itself executes.

## Impact Details

Measured on a fork of the live vault proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`, upgraded to the in-scope V2 implementation, at 2.0x effective leverage with the divergence tolerance at zero:

```
vault before the payout           : 56,687,143 FXRP
vault after  the payout           :  7,687,143 FXRP
buffer after the payout           :          0 USDC
totalAssetsAt(P+1 start)          : 56,687,143 FXRP   unchanged by the payout

capacity live collateral supports :    $12,811,906
capacity the chain accepts        :    $94,478,573
cover actually settled            :    $90,000,000
real CAR                          : 0.1708   (enforced floor 1.2)
```

The protocol writes $90,000,000 of cover obligations against $7,687,143 of collateral. Stakers carry exposure the solvency check reported as covered, and cover buyers pay premium upfront for cover the remaining collateral cannot meet. Neither side can exit: cover *"is enabled for a fixed 30-day period and cannot be cancelled mid-period"*. The live vault holds nine figures of FXRP, and any covered incident in that period pays out pro-rata against collateral the ceiling said was there.

This is not the protocol operating as documented. "Claims Liquidation" defines payout capacity as the same two legs and states *"Capacity is measured at settlement. ... After each incident, capacity is reduced by the amount paid."* "Capital Adequacy" states the guard is automatic: *"capacity that would push CAR below the 1.2 floor cannot be allocated. Manual intervention is possible for edge cases but the baseline protection is automatic."*

No role acts outside its authority and no attacker is involved. In PoC test 3 the matcher does exactly what step 4 of the documented matching flow prescribes (`ICoverOrderAllocator.sol:40-41`) and reads live `vault.totalAssets()` before the payout lands, which is when a matcher runs. The chain accepts its honest number afterwards: $2,000,000 of capacity computed against $1,000,000 of live collateral that is $200,000 by the time the commit is mined. The trigger is the protocol paying a legitimate assessed claim, on-chain, inside the period the capacity validator is measuring.

## References

* `contracts/core/CoverOrderAllocator.sol:704-724` (the two legs and the ceiling), `:705` live buffer, `:711` snapshotted vault
* `contracts/core/CoverOrderAllocator.sol:75-78`, `:343`, `:346`, `:348`, `:378`, `:400`, `:662`
* `contracts/core/FirelightVault.sol:952` (the checkpoint at `Time.timestamp()`, the root cause), `:937`/`:941` (`paidFromActive`)
* `contracts/core/IncidentManager.sol:637`, `:658`, `:684` (one waterfall spends both legs)
* `contracts/core/interfaces/ICoverOrderAllocator.sol:40-41` (published matcher formula, live `totalAssets()`), `:564`
* <https://docs.firelight.finance/risk-and-pricing-model/capital-adequacy> (CAR definition, the 1.2 floor, "the baseline protection is automatic")
* <https://docs.firelight.finance/protocol-architecture/claims-liquidation> ("After each incident, capacity is reduced by the amount paid")

## Link to Proof of Concept

<https://gist.github.com/webrainsec/8ceb37689a5748f48f522b30e8c4f574>

## Proof of Concept

Two runnable Hardhat tests against the real `FirelightVault`, `CoverOrderAllocator`, `IncidentManager` and `CoverNFT`. Only the price feed is mocked, standing in for the Flare FTSO adapter. Gist: <https://gist.github.com/webrainsec/8ceb37689a5748f48f522b30e8c4f574>

`_computeAvailableCapacity` has no public getter, so both tests bracket it: committing `c` succeeds and committing `c + 1` must revert `MatchingCapacityOverflow`. That pins the ceiling to exactly `c`, so each figure below is a measurement rather than an inference.

### Mainnet fork, `FORK_SUBMISSION3_poc.js`

```bash
FORK=1 NODE_RPC_URL=https://flare-api.flare.network/ext/C/rpc BN=67610359 \
  npx hardhat test test/FORK_SUBMISSION3_poc.js
```

{% stepper %}
{% step %}

### Fork Flare and read the live vault

Fork Flare at block 67610359 and read the live vault proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`: `totalAssets` 56,687,143 FXRP, `contractVersion` 1.
{% endstep %}

{% step %}

### Upgrade the proxy

Upgrade that proxy to the in-scope V2 `FirelightVault` through the real `ProxyAdmin` `0x15f89292cef63405a53ab806966681ecb430df85` using its owner `0xF811E83b45d8De67858eFA3aC1202A8B46B8d0b3`. Assert `contractVersion` is 2 and `totalAssets` is unchanged, so the real balance sheet and checkpoint history carry into the test.
{% endstep %}

{% step %}

### Deploy the Phase 2 stack

Deploy the Phase 2 stack against that vault at `effectiveLeverage` 20000 (2.0x), `minCAR` 12000, `divergenceToleranceBps` 0, and fund the first-loss buffer with 1,000,000 USDC.
{% endstep %}

{% step %}

### Settle cover and write a new cover order

In period P, settle $50,000,000 of cover, then write a cover order for period P+1.
{% endstep %}

{% step %}

### File, confirm, and assess an incident

One hour into P, CURATOR files, confirms and assesses an incident for the full $50,000,000 and submits it.
{% endstep %}

{% step %}

### Approve the assessment

Roll into period P+1. One hour in, ASSESSMENT\_APPROVER calls `approveCurrentAssessment`, inside the payout window and inside that role's authority.
{% endstep %}

{% step %}

### Observe the waterfall

Observe the waterfall spent both legs: the buffer goes to 0 USDC, the vault from 56,687,143 to 7,687,143 FXRP.
{% endstep %}

{% step %}

### Observe the stale snapshot

Observe `totalAssetsAt(P+1 start)` is still 56,687,143 FXRP, unchanged by the payout.
{% endstep %}

{% step %}

### Measure the capacity ceiling

Measure the ceiling: the chain accepts $94,478,573 where live collateral supports $12,811,906.
{% endstep %}

{% step %}

### Commit and settle new cover

ALLOCATOR commits period P+1 and settles $90,000,000 of cover. Both succeed. Real CAR is 0.1708 against the enforced 1.2 floor.
{% endstep %}
{% endstepper %}

The test also asserts the sequence happens inside one period with no boundary crossed: period start, payout at +3,661s, cover sold at +3,663s, period end at +86,400s, `currentPeriod()` unchanged throughout.

### Local, `SUBMISSION3_poc.js`

```bash
npx hardhat test test/SUBMISSION3_poc.js
```

* **Test 1** runs the same flow on a fresh stack at the contract's ceilings (`effectiveLeverage` 60000, `divergenceToleranceBps` 1000). Capacity measures $6,600,000 before the payout and $5,500,000 after. The drop is exactly the buffer leg's $1,100,000; the vault's spent $800,000 contributes nothing to it. $5,000,000 of cover then settles against $200,000 of live collateral.
* **Test 2** is the control. The identical payout, same amounts and roles, executed one period earlier so it precedes the boundary. The snapshot carries it, capacity measures $1,100,000, and the same $5,000,000 sale is refused on both routes: claiming $5,500,000 reverts `MatchingCapacityOverflow`, claiming the honest $1,100,000 reverts `TotalAllocationOverflow`. Payout timing is the only variable.
* **Test 3** shows no role deviates. The matcher reads live `vault.totalAssets()` per step 4 of the published flow and computes $2,000,000 before the payout lands. The payout executes, live collateral drops to $200,000, and the chain still accepts the matcher's $2,000,000. The buffer is empty throughout, so the buffer leg cannot account for any of it.
* **Test 4** repeats at 2.4x leverage with the divergence tolerance at zero, inside the documented 2.0x to 2.5x launch band. Real CAR again lands at 0.24.
