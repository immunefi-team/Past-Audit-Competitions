# \#98767 [SC-Medium] Plether: close-leg execution fees are kept by the LP pool instead of the protocol treasury whenever the close earns a VPI rebate

**Submitted on Nov 10th 2026 at 03:30:00 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98767
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Close-leg execution fees are kept by the LP pool instead of the protocol treasury whenever the close earns a VPI rebate

**Severity:** Low
**Impact:** Contract fails to deliver promised returns, but doesn't lose value
**Asset:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src (`libraries/CfdEnginePlanLib.sol`, with `CfdEngineSettlementSidecar.sol`)
**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## What is and is not claimed

Claimed: on a close that earns a VPI rebate `V`, the trader bears the full 4 bps execution fee `F`, but the treasury is credited only `max(0, F - V)`. The remaining `min(F, V)` stays in HousePool as LP value. Once the book's net skew is above roughly 4% of pool depth (the release skew cap is 40%), every close on the heavy side credits the treasury nothing, whatever the trade size. The open leg of the same trade, with the same rebate, credits the full fee.

Not claimed: any loss to traders or LPs, insolvency, or a freeze. Traders pay what they are quoted. The only party short is the protocol treasury, and the amount it is short is held by LPs.

## Summary

The close planner adds the signed VPI, the execution fee and other action charges into one net amount, and then caps the treasury's fee at the cash that net amount actually collects. A VPI rebate therefore cancels the fee before the treasury is credited. The rebate paid to the trader is reduced by the fee, so the trader still pays it; the pool simply pays out `V - F` in place of `V` and keeps the difference.

The protocol has a mechanism for this case: when a fee was not collected from trader cash, it is topped up from free pool cash. The open path uses it and credits the whole fee. The close path computes the top-up budget from the already-capped fee, so the budget is zero and the top-up never runs, even with ample free cash.

## Root cause

All references are at the pin.

1. `_planCloseActionSettlement` nets VPI against the fee (`libraries/CfdEnginePlanLib.sol:1041-1044`):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
```

2. It then overwrites the fee with the part covered by collected or withheld cash (`:1066-1068`). If `V >= F`, nothing is collected and the fee becomes 0. If `V < F`, it becomes `F - V`.

```solidity
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

3. The pool top-up that exists to fund an uncollected fee is budgeted from that overwritten value (`:984`), so it cannot restore what step 2 removed:

```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
```

4. The open path nets the same rebate against the fee (`libraries/OpenAccountingLib.sol:135`) but passes the nominal fee as the top-up budget (`CfdEngineSettlementSidecar.sol:498`), so the whole fee is credited:

```solidity
_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets);
```

The close path's call (`CfdEngineSettlementSidecar.sol:626-628`) receives the capped budget from step 3.

## When it happens

No attacker and no privileged role are needed; ordinary trading triggers it.

- A close earns a rebate when it reduces net skew. That is every close by a trader on the heavier side of the book.
- For a small trade of size `S` on a book with net skew `K` and depth `D`, the rebate is about `vpiFactor × K × S / D` and the fee is `feeBps × S`. The rebate reaches the fee when `K / D ≥ feeBps / vpiFactor = 0.0004 / 0.01 = 4%`. Trade size cancels out.
- The release configuration has `vpiFactor = 0.01e18`, `executionFeeBps = 4`, `maxSkewRatio = 0.4e18` (`packages/perps/DEPLOYMENT.md`, `script/DeployPerpsArbitrumSepolia.s.sol:434-445`). Skew of 4% of depth is one tenth of the permitted cap.
- A close that increases skew pays VPI and credits the full fee. A position that was itself rebated at open has that rebate clawed back at close, so its fee is also credited in full. Carry or frozen spread collected in the same close adds cash and can cover part of the gap.

## Measurements

Local Foundry tests at the pin with the release risk parameters, on the repository's `BasePerpTest` harness and mock price feed. All closes are at an unchanged mark. Pool sizes are test values; the treasury is read as `clearinghouse.balanceAssets(engine.protocolTreasury())`.

**A 100,000 USDG trader on the heavy side of a 102,206,000 USDG pool.** The open leg credited the full 40 USDG in every row.

| Book net skew (USDG) | Skew / depth | Close fee `F` | Close rebate `V` | Treasury credit on close | Fee not credited |
|---|---|---|---|---|---|
| 0 | 0% | 40.00 | 0.49 | 39.51 | 1% |
| 1,000,000 | 0.97% | 40.00 | 10.27 | 29.73 | 26% |
| 2,000,000 | 1.95% | 40.00 | 20.06 | 19.94 | 50% |
| 3,000,000 | 2.93% | 40.00 | 29.84 | 10.16 | 75% |
| 5,000,000 | 4.89% | 40.00 | 49.41 | 0 | 100% |
| 10,000,000 | 9.78% | 40.00 | 98.32 | 0 | 100% |
| 20,000,000 | 19.56% | 40.00 | 196.13 | 0 | 100% |

**Ten such traders round-tripping on the 5,000,000-skew book:**

| Item | USDG |
|---|---|
| Execution fees assessed (20 legs × 40) | 800.00 |
| Traders' total net cost | 824.46 |
| Treasury credited on opens | 400.00 |
| Treasury credited on closes | 0 |
| Pool token balance gain | 424.46 |
| Free pool cash after the last close | 97,197,147.46 |

The treasury received half of the fees the traders paid. The pool held more than 97 million USDG of free cash, so the missing 400 USDG was not withheld by any cash-priority rule.

**Open versus close with the same rebate (1,000,000 USDG pool):** a rebated 100,000 USDG open credits 40 USDG; a rebated 100,000 USDG close credits 0.

**Controls:** a skew-raising close credits its full fee (20 USDG on 50,000 USDG). With `vpiFactor = 0` a round trip credits both fees (80 USDG).

## Impact

The treasury does not receive execution-fee revenue that traders pay and that the documentation assigns to it. In a book skewed beyond about 4% of depth, the treasury collects no close-leg fee from the heavy side, which by definition holds most of the open interest. The uncredited amount becomes LP value.

Total revenue lost over a period was not measured; it depends on trade flow and how skewed the book runs.

## Why this is a defect and not the documented cash-credit rule

The documentation limits the treasury's fee in one situation only: when cash is short.

- Trading costs page: "Only a fee collected from trader collateral or funded from available pool cash is credited to treasury margin. Any amount that cannot be cash-credited under trader-payout and claim seniority is not recorded as a protocol receivable or as LP bad debt."
- `ACCOUNTING_SPEC.md:782`: "protocol fee top-ups are subordinate to trader claims and immediate trader payouts; any fee amount that cannot be cash-credited under this priority is not recorded as a protocol fee receivable."

That is not what happens here. Free pool cash is available (97 million USDG in the aggregate test), no trader payout or claim is competing for it, and the fee is still not credited.

The documentation also says where an uncredited fee must not go:

- Trading costs page: "A cash-credited protocol execution fee belongs to the protocol treasury. An uncredited amount is not a treasury receivable or LP revenue."
- Market model page: "Protocol execution fees belong to the treasury rather than LPs."
- `WHITEPAPER.md:412-413`: "a VPI rebate reduces them [pool assets]; the protocol execution fee belongs to treasury rather than LP assets."
- `WHITEPAPER.md:1455`: "voluntary closes pay the 4-basis-point treasury execution fee".

In the tests the uncredited fee is LP revenue: the pool's token balance rises by it.

Nothing in the public docs says a VPI rebate reduces the treasury's fee. The trading costs page lists the fee and signed VPI as separate terms of the close result, says negative VPI "is funded by the liquidity pool", and describes VPI offsetting only earlier VPI.

The repository's own accounting test states the intent. `test_ExecutionFeesAreProtocolRevenue` (`test/perps/accounting/cash-priority/TreasuryFeeCustody.t.sol:109-134`) asserts "Open and close execution fees should both accrue as protocol revenue" and that LP equity moves by at most 1,000 atoms. It passes because `BasePerpTest` sets `vpiFactor = 0`. With only `vpiFactor` changed to the release value it fails:

```
[FAIL: LP equity delta should be limited to incidental one-second senior yield: 40002497 > 1000]
```

The 40,002,497 atoms are the close-leg fee. To reproduce, add this file beside the PoC and run `forge test --match-path test/perps/VendorFeeTestAtReleaseVpi.t.sol --match-test test_ExecutionFeesAreProtocolRevenue`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;

import {MarginAndReservationAdmissionFeesTest} from "./accounting/cash-priority/TreasuryFeeCustody.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

contract VendorFeeTestAtReleaseVpi is MarginAndReservationAdmissionFeesTest {
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
    }
}
```

### An objection we expect

`test/perps/invariant/properties/PerpVpiFrozenAccountingInvariant.t.sol:118-119` carries the comment "A net VPI rebate can offset all or part of the assessed fee; nominal fee arithmetic alone is not treasury revenue", and its model credits `max(0, F - V)`. We read that as the invariant mirroring the implementation's arithmetic, for three reasons:

- it contradicts the accounting test above, in the same repository, whose assertion names the intent directly;
- it contradicts the open path, which nets the same rebate and still credits the full fee;
- it has no counterpart in the public documentation or `ACCOUNTING_SPEC.md`, both of which condition the credit on cash availability only.

If the team's position is that the close-leg behaviour is intended, then the open path and the documentation are the parts that are wrong, and the finding stands as an inconsistency between them.

## Why this is not a known issue

- **Cecuro MED-6** (treasury spending LP withdrawal reserves on fee top-ups) concerns which cash a top-up may use. It was fixed by capping top-ups at free settlement cash. This report is about the size of the close-leg budget, which is zero while free cash is plentiful.
- **Cecuro LOW-26** (underfunded protocol top-ups blocking opens) concerns open-path admission when pool cash is short. Its resolution describes the open-path policy this report asks the close path to match: the fee credited is "the portion funded directly by trade cost plus a top-up capped by post-open settlement-free pool cash".
- **AI scan "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"** concerns the size of the rebate under depth manipulation, not where the fee goes.
- Not in `packages/perps/SECURITY.md` Known Limitations.

## Recommended fix

Budget the close-leg top-up from the fee the trader bore, not from the capped fee: the part collected in cash plus the part absorbed by netting against the rebate. Keep the existing cap at free pool cash, so trader payouts and claims retain priority.

Two cases must stay excluded, and the project's tests cover both:

- a close whose action charge was waived because nothing could be collected has no borne fee to credit (`test_OracleFrozenFullClose_ClaimRecoveryPaysFeeThenBaseThenSpread`);
- a full-loss close that collects only the price cap (`test_PreviewClose_FullLossWriteoffMatchesLiveSettlement`).

Simply budgeting from the nominal fee, as the open path does, breaks the first of these. `invariant_PersistentVpiCashAndReserveModel` encodes the current credit and would need its model updated with any fix. The harnesses should also run at a nonzero `vpiFactor`, since the release value is never exercised by the accounting suite today.




## Link to Proof of Concept

https://gist.github.com/dexloom/8eea36de757c6a2ac0e9357177925679

## Proof of Concept

## Proof of concept

PoC: https://gist.github.com/dexloom/8eea36de757c6a2ac0e9357177925679

At the pin, initialise submodules, add the file to `packages/perps/test/perps/`, then:

    git submodule update --init --recursive
    cd packages/perps
    forge test --match-path test/perps/CloseFeeRoutingPoC.t.sol -vv

Expected: 12 tests pass. No file under `src` is changed and no RPC is needed.

- `CloseFeeBookSkewTest.test_BookSkew_*`: the table above; each asserts `credit == max(0, F - V)` and that the open leg credited the full fee.
- `CloseFeeBookSkewTest.test_Aggregate_TenTradersOnSkewedBook`: the aggregate figures.
- `CloseFeeSmallPoolTest.test_RoundTrip_TreasuryReceivesOneFee_PoolKeepsTheOther`: trader pays 80.002497 USDG, treasury receives 40, pool gains 40.002497.
- `CloseFeeSmallPoolTest.test_SameRebate_OpenCreditsFullFee_CloseCreditsZero`: the open/close asymmetry.
- `CloseFeeSmallPoolTest.test_Control_SkewRaisingCloseCreditsFullFee` and `CloseFeeZeroVpiControlTest.test_Control_ZeroVpiCreditsBothFees`: controls.

Recorded output (trimmed):

```
CloseFeeBookSkewTest
  book_skew_usdg: 0         close_rebate_V: 489198     treasury_close_credit: 39510802
  book_skew_usdg: 1000000   close_rebate_V: 10273162   treasury_close_credit: 29726838
  book_skew_usdg: 2000000   close_rebate_V: 20057096   treasury_close_credit: 19942904
  book_skew_usdg: 3000000   close_rebate_V: 29840971   treasury_close_credit: 10159029
  book_skew_usdg: 5000000   close_rebate_V: 49408433   treasury_close_credit: 0
  book_skew_usdg: 10000000  close_rebate_V: 98324097   treasury_close_credit: 0
  book_skew_usdg: 20000000  close_rebate_V: 196130671  treasury_close_credit: 0
[PASS] test_Aggregate_TenTradersOnSkewedBook()
  fees_assessed_total: 800000000
  traders_net_cost_total: 824460150
  treasury_open_credits: 400000000
  treasury_close_credits: 0
  pool_token_gain: 424460150
  pool_free_cash_after: 97197147456392
CloseFeeSmallPoolTest
  open_leg_credit: 40000000   close_leg_credit: 0   pool_token_gain: 40002497   trader_cost: 80002497
  rebated_open_credit: 40000000   rebated_close_credit: 0
  skew_raising_close_credit: 20000000
CloseFeeZeroVpiControlTest
  zero_vpi_round_trip_credit: 80000000
12 tests passed, 0 failed
```
