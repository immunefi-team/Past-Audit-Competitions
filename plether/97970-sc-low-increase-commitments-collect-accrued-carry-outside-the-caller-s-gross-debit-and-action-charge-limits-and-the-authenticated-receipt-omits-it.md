# \#97970 [SC-Low] Increase commitments collect accrued carry outside the caller's gross-debit and action-charge limits, and the authenticated receipt omits it

**Submitted on Nov 4th 2026 at 05:39:49 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97970
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a trader commits an increase to an existing position through `OrderRouter.commitOrder`, the Router locks the opening execution bounty with `MarginClearinghouse.lockReservedSettlement`. That call first checkpoints and collects all the carry the position has accrued. The open-order reservation path then reports an all-zero carry effect to `OrderLifecycleBook.recordCommitment`. As a result, the request's inclusive `maxGrossAccountDebitAssets` and `maxActionChargeAssets` are never applied to this debit, and the order's authenticated terminal receipt never records it.

In the PoC (release risk parameters, public entrypoints only), the request allows at most 100 USDG of action charge and 100.2 USDG of gross debit:

- `commitOrder` debits **301.753575 USDG** of carry.
- The order executes.
- Its verified receipt reports a gross account debit of **34.616454 USDG**, while the account actually lost **336.370029 USDG**.

The Router's own policy evaluator rejects the same request before commitment, because it counts that carry. A close with the same accrued carry and the same two limits is correctly rejected at commitment.

The carry is owed and goes to LPs, so no value is stolen. What fails is the protocol's promise that every V3 order is held to its caller's financial limits and recorded in a correct receipt. This affects every increase on a position with accrued carry, whether or not the increase is ever executed: an increase that expires unexecuted keeps the commitment-time debit, and its failure receipt also records zero. Severity: **Low**.

## Vulnerability Details

### Why this is not intended behaviour

The protocol already treats this carry as a charge of the increase. It bounds it everywhere except the one place where it is actually collected:

1. **The in-scope lens attributes it to the increase.** `CfdEngineLens.previewOpen` returns it as the increase's `pendingCarryAssets` (`ICfdEngineTypes.sol` L364). In the PoC that is 301.753575 USDG, exactly the amount `commitOrder` then debits.
2. **The Router's policy evaluator bounds it for opens and increases.** `_buildOpenAssessment` adds an existing position's pending carry to `grossAccountDebitAssets` and `actionChargeAssessedAssets` (`CfdOrderPolicyEvaluatorBase.sol` L342-L348). If you run the Router's configured `CfdOrderPolicyEvaluator.assessOrder` on the exact request before commitment, it reverts with `ConstraintViolation(GrossAccountDebit, 336369905, 100200000)`. The account then really loses 336.370029 USDG across commit and execution. That is the evaluator's figure plus 124 atoms of carry for the seconds between the two transactions.
3. **The same commitment-time collection is bounded and recorded for closes.** ACCOUNTING_SPEC L939-L945: "Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds. The Book records custody before and after commitment and any carry still outstanding." The PoC control shows a close with the same carry and limits reverting with `OrderLifecycleBook__CommitmentBoundExceeded(GrossAccountDebit, 301753575, 100200000)`.
4. **The normative text covers every V3 order, not only closes:**
   - `OrderV3Types.sol` L131-L134: "Mandatory financial authority supplied with every V3 order. Maximum values are inclusive and zero is a real zero allowance … `maxGrossAccountDebitAssets` covers settlement debit, trader-claim consumption, and the reserved bounty".
   - `SECURITY.md` L756-L765: "The external V3 request sets inclusive maxima for … gross account debit, assessed action charge … Gross account debit includes settlement value lost or charged …".
   - `OrderRouter.sol` L77-L82: a fresh request "pins … financial limits in the immutable lifecycle book".
   - `ACCOUNTING_SPEC.md` L696-L697 lists "open/modify" among the paths that collect pending carry. L723-L724 adds: "realized carry is booked as LP trading revenue once. Receipts and financial bounds include the direct collection once".
   - `IOrderLifecycleBook.sol` L17-L19 defines the recorded effects as "Carry and bounty effects returned by the Engine or open-order reservation path".
5. No document, NatSpec comment or known issue says that an increase's commitment-time carry is exempt from the limits or the receipt.

The open evaluator's carry term (point 2) is never applied to carry accrued before commitment. The minimum open bounty can never be zero: the defaults are 0.01 to 0.20 USDG (`OrderRouterBase.sol` L73-L74), and `OrderRouterAdmin` rejects a zero minimum (L381-L389). So every open or increase takes the `lockReservedSettlement` branch, and that branch collects all pre-commitment carry before any limit is checked. By execution time, only the seconds since commitment remain to be bounded.

### Root cause

**1. The open branch collects carry but returns zero effects.** `packages/perps/src/router/OrderReservationAccounting.sol` L108-L128:

```solidity
function _reserveExecutionBounty(address account, uint256 sizeDelta, uint256 bounty, bool isClose)
    internal returns (CfdEnginePlanTypes.CloseCommitment memory effects)
{
    if (isClose) {
        ...
        return _reserveCloseExecutionBounty(account, sizeDelta); // Engine returns real carry effects
    }
    effects.bountyFromFreeAssets = bounty;                       // only field set for opens
    if (bounty == 0) {
        return effects;
    }
    if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
        revert OrderRouter__InsufficientFreeEquity();
    }
    clearinghouse.lockReservedSettlement(account, bounty);       // collects all accrued carry
}
```

**2. The bounty lock checkpoints carry first.** `packages/perps/src/MarginClearinghouse.sol` L818-L824:

```solidity
function lockReservedSettlement(address account, uint256 amountAssets) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);   // -> CfdEngine.realizeCarryBeforeMarginChange
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

`CfdEngine.realizeCarryBeforeMarginChange` (L535-L550) calls `_realizeCarryFromSettlement` (L1407-L1449). That function consumes position margin first and then free settlement, transfers the carry to the HousePool, and records it as LP revenue. This is a real debit: in the PoC, pool assets rise by exactly 301.753575 USDG during `commitOrder`.

**3. The Book checks the limits only against the reported carry.** `packages/perps/src/router/OrderCommitHandler.sol` L27-L32 passes the returned effects to `OrderLifecycleBook.recordCommitment` (`OrderLifecycleBook.sol` L327-L361):

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;   // 0 + 0.2 USDG
if (gross > pending.bounds.maxGrossAccountDebitAssets) revert ...CommitmentBoundExceeded(...);
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) revert ...; // 0 > 100: false
...
pending.commitment = StoredCommitment({ carryCollectedAssets: effects.carryCollectedAssets, ... }); // stores 0
```

The consistency check `settlementBeforeAssets == settlementAfterAssets + carryCollectedAssets` also passes trivially (0 == 0 + 0), because the open path never takes a custody snapshot.

**4. Execution adds commitment effects only for closes.** `packages/perps/src/CfdOrderPolicyEvaluator.sol` L64-L67:

```solidity
assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
if (order.isClose) {
    _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
}
```

`_includeCommitment` (`CfdOrderPolicyEvaluatorBase.sol` L474-L487) adds commitment carry to gross debit, assessed and collected action charge, and carry, then re-enforces the bounds. For an increase, the execution snapshot is taken after the commitment debit, so the evaluator sees only the carry accrued after commitment.

**5. The receipt copies both incomplete sources.** `OrderRouterExecutionSidecar.sol` L717 copies the stored zero commitment into `receipt.commitment`. `_assessmentEconomics` (L781-L799) copies the execution-only assessment and the post-commitment `preSettlementBalanceAssets`. So the authenticated receipt cannot reconstruct the debit, even from its own pre/post balances.

### No other caller limit can stop it

For an increase, the two bypassed fields are the only debit ceilings available. A positive `minPostSettlementBalanceAssets` is rejected for any account that already has a position (`OrderLiquidationBatchLogic.sol` L307-L312; `WORKING_WITH_AI_AGENTS.md` L114-L116: "It is rejected for increases, closes, and overlapping orders"). The PoC confirms that an increase with a positive settlement floor reverts with `OrderRouter__SettlementFloorUnsupported`. The equity and leverage bounds measure risk, not cost. Fresh opens from a flat account have no carry, so increases are the affected class.

### Sequence (PoC numbers)

1. A trader holds a 300,000 USDG LONG with 20,000 USDG margin (about 15x, against a 500x maximum) and does not touch its margin for four weeks. At release `baseCarryBps = 500` with a 1,000,000 USDG Junior pool, 301.753575 USDG of carry accrues, about 10.8 USDG per day.
2. The trader commits a 10,000 USDG increase with `maxActionChargeAssets = 100 USDG` and `maxGrossAccountDebitAssets = 100.2 USDG` (100 plus the 0.20 USDG bounty).
3. `commitOrder` collects 301.753575 USDG. The Book records commitment carry 0, carry outstanding 0, `settlementAfterAssets` 0, and bounty 0.2 USDG.
4. A public keeper executes the order. It succeeds and the position becomes 310,000.
5. The verified `OrderFinalized` receipt shows `commitment.carryCollectedAssets = 0`, `economics.carryAssets = 0.000124`, `actionChargeAssessedAssets = 34.416454` and `grossAccountDebitAssets = 34.616454`. All appear to be within the limits. The real debit was 336.370029 USDG.

After only one week of carry (75.438487 USDG), a 50 USDG action limit on the same increase is bypassed in the same way. The increase's own trade cost is 34.423196 USDG, well within the limit.

The debit does not depend on execution. In `test_UnexecutedIncreaseKeepsCommitmentCarryAndItsFailureReceiptOmitsIt` the same bounded increase is never executed and is expired by any caller after its deadline, leaving the position unchanged. The 301.753575 USDG taken at commitment stays with LPs, and the authenticated failure receipt reports `commitment.carryCollectedAssets = 0` and `grossAccountDebitAssets = 0`. A trader whose increase never happens still pays a charge above its limits that no receipt records.

## Impact Details

- **The caller's financial limits are not enforced.** On every increase of a position with accrued carry, `maxGrossAccountDebitAssets` and `maxActionChargeAssets` do not apply to that carry, even though the protocol's lens and evaluator count it as part of the increase. The debit is unbounded and grows linearly with the time since the position was last checkpointed. An increase that its caller authorised only under a ceiling executes even though the actual charge exceeds that ceiling several times over (more than 3x in the PoC: 336.370029 against 100.2 USDG). The settlement floor cannot be used as a substitute, because it is unavailable to increases.
- **The authenticated receipt is wrong.** The integration docs tell integrators to authenticate the receipt and treat it as the order's on-chain history (`CLOSE_PREVIEW.md` L55: "verified lifecycle receipts for history"; L109: "terminal receipts establish what happened on-chain"; `WORKING_WITH_AI_AGENTS.md` L298-L312). For every increase, the receipt reports zero commitment carry and a gross debit that leaves out the commitment-time collection (34.616454 vs 336.370029 USDG in the PoC). Its `preSettlementBalanceAssets` is already net of that collection. Accounting, fee reporting or limit monitoring built on receipts under-reports trader debits.
- **What is not claimed.** The carry was owed, and it reaches LPs in full (pool assets rise by exactly the collected amount). Health projections already deduct pending carry (`ACCOUNTING_SPEC.md` L698-L699, L706-L707, L751-L752), so the early collection does not move the position toward liquidation, and this report claims no liquidation impact. No third party can trigger it on someone else's account.

**Severity: Low. Impact: "Contract fails to deliver promised returns, but doesn't lose value".** The documented financial-authority and receipt guarantees fail, without loss of value.

### Recommendation

1. In `OrderReservationAccounting._reserveExecutionBounty`, return the real carry and custody effects for the open/increase branch. Either snapshot `balanceAssets`, `freeSettlementAssets` and `unsettledCarryAssets` around `lockReservedSettlement`, or let the Engine checkpoint carry and return a `CloseCommitment`, as the close path already does. Populate `carryCollectedAssets`, `carryOutstandingAssets`, `settlementBeforeAssets`, `settlementAfterAssets` and `freeSettlementAfterAssets`. `recordCommitment` will then reject an excessive commitment debit atomically, exactly as it does for closes.
2. In `CfdOrderPolicyEvaluator.assessCommittedOrder`, apply `_includeCommitment` to opens as well as closes (remove the `if (order.isClose)` guard). Commitment-time and execution-time carry are then enforced against one lifecycle-wide limit and reported once in the receipt.
3. Add regression tests:
   - commitment carry alone exceeds each limit;
   - commitment carry and execution charges are each below a limit but exceed it together;
   - an executed increase's receipt reconciles exactly to the account and pool balance changes.

Skipping the carry checkpoint in `lockReservedSettlement` is not a good fix. It would change the documented carry-before-margin-mutation order.

## References

- Open-branch reservation returns zero carry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128
- Commit handler records the returned effects: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L36
- `lockReservedSettlement` checkpoints carry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L812-L824
- Clearinghouse to Engine carry hook: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1336-L1356
- Engine carry realization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L535-L550 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449
- Close-path commitment returns real effects: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L592-L625
- Book commitment bound checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L361
- Commitment effects included only for closes: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L64-L67
- Open assessment counts pending carry against the limits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L340-L348
- `_includeCommitment`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L487
- Receipt copies the stored commitment and the execution assessment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L703-L718 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L781-L799
- `ExecutionBounds` NatSpec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L150
- `commitOrder` NatSpec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L77-L82
- `recordCommitment` interface NatSpec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IOrderLifecycleBook.sol#L17-L23
- Settlement floor rejected for increases: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L307-L312
- Minimum open bounty is always nonzero: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L70-L75 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L381-L389
- `previewOpen` and `OpenPreview.pendingCarryAssets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L92-L100 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineTypes.sol#L350-L380
- ACCOUNTING_SPEC carry rules ("open/modify", "Receipts and financial bounds include the direct collection once"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L690-L726
- ACCOUNTING_SPEC close commitment carry included in the bounds: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L939-L945
- SECURITY pinned financial policy: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L754-L765
- Settlement floor restrictions for integrators: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L114-L119
- Closing positions, commitment carry included in totals: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L98-L104


## Proof of Concept

## Proof of Concept

A single Foundry test file runs against the unmodified audit commit (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`) using the repository's own `BasePerpTest` harness.

### Setup

- **Release risk parameters**, asserted in `setUp`: VPI 0.01e18, max skew 0.4e18, maintenance/initial/FAD margin 10/20/300 bps, `baseCarryBps` 500, execution fee 4 bps, frozen spread 50 bps, confidence ratio 10 bps with a 2,500 multiplier.
- **Public paths only.** A 1,000,000 USDG Junior pool is funded through the public asynchronous ERC-7540 deposit flow (`requestDeposit` -> `settleLpEpoch` -> `claimDeposit`). The trader deposits through `MarginClearinghouse.deposit`. All orders use the production V3 `OrderRouter.commitOrder(OrderRequest)` entrypoint and are executed by an unrelated keeper EOA through `executeOrder`. The mark is installed through the public `updateMarkPrice`.
- **Mocks.** MockPyth only supplies signed-price fixture data; Pyth is out of scope. No call is pranked as a protocol contract. The harness Router is `LegacyOrderRouterHarness is OrderRouter`; that subclass only adds a legacy scalar `commitOrder` overload, which this PoC does not use.
- **Position.** The trader opens a 300,000 USDG LONG with 20,000 USDG margin and then does not touch its margin. Time advances while keeping the same weekday and hour, so the market stays live.

### Tests

| Test | What it shows |
| --- | --- |
| `test_Control_CloseWithSameLimitsIsRejectedAtCommitment` | **Control.** After 28 days, a full close with the same limits (100 USDG action charge, 100.2 USDG gross debit) reverts with `OrderLifecycleBook__CommitmentBoundExceeded(GrossAccountDebit, 301753575, 100200000)`. Trader and pool balances roll back. |
| `test_IncreaseCommitmentCollectsCarryAboveLimitsAndReceiptOmitsIt` | **The bug.** The steps are below. |
| `test_SettlementFloorIsUnavailableToIncreases` | A positive `minPostSettlementBalanceAssets` on an increase reverts with `OrderRouter__SettlementFloorUnsupported`, so no other caller limit can cap the debit. |
| `test_OneWeekOfCarryExceedsAFiftyUsdgLimit` | **Realism.** After only 7 days, 75.438487 USDG is collected at commitment despite a 50 USDG action limit. The increase's own quoted trade cost is 34.423196 USDG. The order executes and the receipt reports 34.423319 USDG. |
| `test_UnexecutedIncreaseKeepsCommitmentCarryAndItsFailureReceiptOmitsIt` | **Breadth.** The same 10,000 USDG increase is never executed; any caller expires it after its deadline and the position is unchanged. The 301.753575 USDG of carry taken at commitment is not returned (pool assets keep the full amount), and the authenticated failure receipt reports commitment carry 0 and gross account debit 0. |

Steps in `test_IncreaseCommitmentCollectsCarryAboveLimitsAndReceiptOmitsIt` (same account and time as the control, a 10,000 USDG increase):

1. `CfdEngineLens.previewOpen` attributes 301.753575 USDG of pending carry to this increase.
2. The Router's configured `CfdOrderPolicyEvaluator.assessOrder`, run on the same request before commitment, rejects it: `ConstraintViolation(GrossAccountDebit, 336369905, 100200000)`.
3. `commitOrder` succeeds anyway and debits exactly 301.753575 USDG. HousePool assets rise by the same amount. The Book stores commitment carry 0, carry outstanding 0, `settlementAfterAssets` 0, and bounty 0.2 USDG.
4. The keeper executes the order, and the position becomes 310,000.
5. The verified `OrderFinalized` receipt reports:
   - `commitment.carryCollectedAssets = 0`;
   - `economics.carryAssets = 124` atoms;
   - `actionChargeAssessedAssets = 34.416454 USDG`;
   - `grossAccountDebitAssets = 34.616454 USDG`;
   - a `preSettlementBalanceAssets` already net of the commitment debit.

   Across commit and execution, the trader actually lost 336.370029 USDG. That is the evaluator's 336.369905 USDG plus 124 atoms of carry accrued between the two transactions.

### How to run

Save the file below as `packages/perps/test/poc/IncreaseCommitmentCarryBypassesFinancialBounds.t.sol` in the audit repository, then run from the repository root:

```bash
forge test --root packages/perps --match-path test/poc/IncreaseCommitmentCarryBypassesFinancialBounds.t.sol -vv
```

### Output

The test was run twice. Logs and gas figures were identical; only the wall-clock timings differ. Compiler warnings printed before the results come from pre-existing repository and forge-std code and are omitted.

Run 1:

```text
Ran 5 tests for test/poc/IncreaseCommitmentCarryBypassesFinancialBounds.t.sol:IncreaseCommitmentCarryBypassesFinancialBoundsTest
[PASS] test_Control_CloseWithSameLimitsIsRejectedAtCommitment() (gas: 1506322)
Logs:
  close: accrued carry (actual reported by Book) 301753575
  close: caller maxGrossDebit 100200000

[PASS] test_IncreaseCommitmentCollectsCarryAboveLimitsAndReceiptOmitsIt() (gas: 3778983)
Logs:
  pre-commit lens pendingCarryAssets 301753575
  pre-commit policy gross debit (rejected) 336369905
  caller maxActionCharge 100000000
  caller maxGrossDebit 100200000
  carry debited at commitment 301753575
  Book commitment carryCollectedAssets 0
  Book commitment settlementAfterAssets 0
  Book commitment bountyFromFreeAssets 200000
  receipt commitment.carryCollectedAssets 0
  receipt economics.carryAssets 124
  receipt economics.actionChargeAssessedAssets 34416454
  receipt economics.grossAccountDebitAssets 34616454
  receipt economics.preSettlementBalanceAssets 99128944629
  settlement balance before commit 99430698204
  actual total settlement debit, commit + execution 336370029

[PASS] test_OneWeekOfCarryExceedsAFiftyUsdgLimit() (gas: 3609440)
Logs:
  one week: quoted trade cost (fee + VPI) 34423196
  one week: caller maxActionCharge 50000000
  one week: carry debited at commitment 75438487
  one week: receipt actionChargeAssessedAssets 34423319

[PASS] test_SettlementFloorIsUnavailableToIncreases() (gas: 386620)
[PASS] test_UnexecutedIncreaseKeepsCommitmentCarryAndItsFailureReceiptOmitsIt() (gas: 2484584)
Logs:
  expired: caller maxActionCharge 100000000
  expired: carry debited at commitment 301753575
  expired: trader settlement debit after expiry 301953575
  expired: LP pool asset increase 301753575
  expired: receipt commitment.carryCollectedAssets 0
  expired: receipt economics.grossAccountDebitAssets 0

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 10.29ms (8.48ms CPU time)

Ran 1 test suite in 11.01ms (10.29ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

Run 2:

```text
Ran 5 tests for test/poc/IncreaseCommitmentCarryBypassesFinancialBounds.t.sol:IncreaseCommitmentCarryBypassesFinancialBoundsTest
[PASS] test_Control_CloseWithSameLimitsIsRejectedAtCommitment() (gas: 1506322)
Logs:
  close: accrued carry (actual reported by Book) 301753575
  close: caller maxGrossDebit 100200000

[PASS] test_IncreaseCommitmentCollectsCarryAboveLimitsAndReceiptOmitsIt() (gas: 3778983)
Logs:
  pre-commit lens pendingCarryAssets 301753575
  pre-commit policy gross debit (rejected) 336369905
  caller maxActionCharge 100000000
  caller maxGrossDebit 100200000
  carry debited at commitment 301753575
  Book commitment carryCollectedAssets 0
  Book commitment settlementAfterAssets 0
  Book commitment bountyFromFreeAssets 200000
  receipt commitment.carryCollectedAssets 0
  receipt economics.carryAssets 124
  receipt economics.actionChargeAssessedAssets 34416454
  receipt economics.grossAccountDebitAssets 34616454
  receipt economics.preSettlementBalanceAssets 99128944629
  settlement balance before commit 99430698204
  actual total settlement debit, commit + execution 336370029

[PASS] test_OneWeekOfCarryExceedsAFiftyUsdgLimit() (gas: 3609440)
Logs:
  one week: quoted trade cost (fee + VPI) 34423196
  one week: caller maxActionCharge 50000000
  one week: carry debited at commitment 75438487
  one week: receipt actionChargeAssessedAssets 34423319

[PASS] test_SettlementFloorIsUnavailableToIncreases() (gas: 386620)
[PASS] test_UnexecutedIncreaseKeepsCommitmentCarryAndItsFailureReceiptOmitsIt() (gas: 2484584)
Logs:
  expired: caller maxActionCharge 100000000
  expired: carry debited at commitment 301753575
  expired: trader settlement debit after expiry 301953575
  expired: LP pool asset increase 301753575
  expired: receipt commitment.carryCollectedAssets 0
  expired: receipt economics.grossAccountDebitAssets 0

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 10.00ms (8.99ms CPU time)

Ran 1 test suite in 10.54ms (10.00ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {console2} from "forge-std/console2.sol";

/// @notice An open/increase commitment locks the execution bounty through
///         `MarginClearinghouse.lockReservedSettlement`, which checkpoints and collects the position's accrued carry.
///         `OrderReservationAccounting._reserveExecutionBounty` then hands the lifecycle Book an all-zero carry effect,
///         so the request's inclusive `maxGrossAccountDebitAssets` / `maxActionChargeAssets` are never applied to that
///         debit, and the authenticated terminal receipt never reports it. A close with the same accrued carry and the
///         same limits is correctly rejected at commitment.
/// @dev Release risk parameters. Every protocol action goes through a public entrypoint from an EOA; MockPyth only
///      supplies signed-price fixture data. The Router is the production `OrderRouter` (the test subclass only adds an
///      unused legacy overload); the V3 `commitOrder(OrderRequest)` entrypoint is called directly.
contract IncreaseCommitmentCarryBypassesFinancialBoundsTest is BasePerpTest {
    address internal constant LP = address(0x1A11CE);
    address internal constant TRADER = address(0x7A0E);
    address internal constant KEEPER = address(0x4EE9E7);

    uint256 internal constant MARK = 1e8;
    uint256 internal constant INITIAL_SIZE = 300_000e18;
    uint256 internal constant INITIAL_MARGIN = 20_000e6;
    uint256 internal constant INCREASE_SIZE = 10_000e18;
    uint256 internal constant INCREASE_MARGIN = 1_000e6;
    uint256 internal constant ACTION_LIMIT = 100e6;

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
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.basketMaxConfidenceRatioBps = 10;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(routerConfig);
        // Monday 2024-03-11 10:00 UTC: live market, after both 72-hour activations.
        vm.warp(1_710_151_200);
        vm.roll(vm.getBlockNumber() + 1);

        // Public asynchronous LP funding; with no positions, cached-price settlement is permissionless.
        settlementToken.mint(LP, 1_000_000e6);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), 1_000_000e6);
        uint256 epoch = juniorVault.requestDeposit(1_000_000e6, LP, LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(epoch));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(epoch, LP);
        vm.prank(LP);
        juniorVault.claimDeposit(epoch, claimable, LP, LP);

        _installPublicMark();
        _fundTrader(TRADER, 100_000e6);
        _openInitialPosition();

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release confidence multiplier");
        (uint256 vpi, uint256 skew, uint256 mm, uint256 im, uint256 fad, uint256 carry,,,,) = engine.riskParams();
        assertEq(vpi, 0.01e18, "release VPI");
        assertEq(skew, 0.4e18, "release max skew");
        assertEq(mm, 10, "release maintenance margin");
        assertEq(im, 20, "release initial margin");
        assertEq(fad, 300, "release FAD margin");
        assertEq(carry, 500, "release carry");
    }

    /// @notice Control: a close carrying the same accrued carry under the same two limits is rejected at commitment,
    ///         with the exact carry as the reported actual, and every balance rolls back.
    function test_Control_CloseWithSameLimitsIsRejectedAtCommitment() public {
        vm.warp(vm.getBlockTimestamp() + 28 days);
        _installPublicMark();
        uint256 accruedCarry = _quotedPendingCarry();
        uint256 bounty = router.maxOpenOrderExecutionBountyAssets();
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();

        OrderV3Types.OrderRequest memory close =
            _request(keccak256("BOUNDED_CLOSE"), INITIAL_SIZE, 0, true, ACTION_LIMIT + bounty, ACTION_LIMIT);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector,
                OrderV3Types.ConstraintKind.GrossAccountDebit,
                accruedCarry,
                ACTION_LIMIT + bounty
            )
        );
        vm.prank(TRADER);
        router.commitOrder(close);

        console2.log("close: accrued carry (actual reported by Book)", accruedCarry);
        console2.log("close: caller maxGrossDebit", ACTION_LIMIT + bounty);
        assertEq(clearinghouse.balanceAssets(TRADER), traderBefore, "close: carry debit rolled back");
        assertEq(pool.totalAssets(), poolBefore, "close: LP carry inflow rolled back");
    }

    /// @notice The increase. The protocol's own policy evaluator, run on the same request before commitment, counts
    ///         the accrued carry against these limits and rejects it. `commitOrder` nevertheless collects that carry,
    ///         records zero, and the order executes with a receipt that appears compliant.
    function test_IncreaseCommitmentCollectsCarryAboveLimitsAndReceiptOmitsIt() public {
        vm.warp(vm.getBlockTimestamp() + 28 days);
        _installPublicMark();
        uint256 bounty = router.maxOpenOrderExecutionBountyAssets();
        OrderV3Types.OrderRequest memory increase = _request(
            keccak256("BOUNDED_INCREASE"), INCREASE_SIZE, INCREASE_MARGIN, false, ACTION_LIMIT + bounty, ACTION_LIMIT
        );

        // 1. Pre-commit: the in-scope lens and the Router's own policy evaluator both treat the accrued carry as
        //    part of this increase. The evaluator rejects the request against its gross-debit limit.
        uint256 quotedCarry = _quotedPendingCarry();
        uint256 policyActual = _expectPolicyRejection(increase, bounty);
        console2.log("pre-commit lens pendingCarryAssets", quotedCarry);
        console2.log("pre-commit policy gross debit (rejected)", policyActual);
        assertGt(policyActual, increase.bounds.maxGrossAccountDebitAssets, "policy evaluator rejects the request");

        // 2. Commit: succeeds and debits the same carry.
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();
        vm.prank(TRADER);
        uint64 id = router.commitOrder(increase);
        uint256 commitDebit = traderBefore - clearinghouse.balanceAssets(TRADER);
        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(id);

        console2.log("caller maxActionCharge", increase.bounds.maxActionChargeAssets);
        console2.log("caller maxGrossDebit", increase.bounds.maxGrossAccountDebitAssets);
        console2.log("carry debited at commitment", commitDebit);
        console2.log("Book commitment carryCollectedAssets", pending.commitment.carryCollectedAssets);
        console2.log("Book commitment settlementAfterAssets", pending.commitment.settlementAfterAssets);
        console2.log("Book commitment bountyFromFreeAssets", pending.commitment.bountyFromFreeAssets);

        assertEq(commitDebit, quotedCarry, "commit debits exactly the carry the lens quoted for this increase");
        assertEq(pool.totalAssets() - poolBefore, commitDebit, "the debit is real LP carry revenue");
        assertGt(commitDebit, increase.bounds.maxActionChargeAssets, "debit exceeds the action-charge limit");
        assertGt(commitDebit + bounty, increase.bounds.maxGrossAccountDebitAssets, "debit exceeds the gross limit");
        assertEq(pending.commitment.carryCollectedAssets, 0, "Book records zero carry");
        assertEq(pending.commitment.carryOutstandingAssets, 0, "Book records no carry remainder");
        assertEq(pending.commitment.settlementAfterAssets, 0, "Book records no custody snapshot");
        assertEq(pending.commitment.bountyFromFreeAssets, bounty, "only the bounty is recorded");

        // 3. Execute promptly through a public keeper. The order executes and the authenticated receipt reports only
        //    the few seconds of carry accrued after commitment.
        _startRecordingLogs();
        bytes[] memory updateData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "bounded increase executes");
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        uint256 totalDebit = traderBefore - clearinghouse.balanceAssets(TRADER);

        console2.log("receipt commitment.carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        console2.log("receipt economics.carryAssets", uint256(receipt.economics.carryAssets));
        console2.log("receipt economics.actionChargeAssessedAssets", receipt.economics.actionChargeAssessedAssets);
        console2.log("receipt economics.grossAccountDebitAssets", receipt.economics.grossAccountDebitAssets);
        console2.log("receipt economics.preSettlementBalanceAssets", receipt.economics.preSettlementBalanceAssets);
        console2.log("settlement balance before commit", traderBefore);
        console2.log("actual total settlement debit, commit + execution", totalDebit);

        assertEq(size, INITIAL_SIZE + INCREASE_SIZE, "position was increased");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "receipt commitment omits the carry");
        assertLt(uint256(receipt.economics.carryAssets), 1e6, "receipt carry is only post-commit dust");
        assertLe(
            receipt.economics.actionChargeAssessedAssets,
            increase.bounds.maxActionChargeAssets,
            "receipt action charge appears within the limit"
        );
        assertLe(
            receipt.economics.grossAccountDebitAssets,
            increase.bounds.maxGrossAccountDebitAssets,
            "receipt gross debit appears within the limit"
        );
        assertEq(
            receipt.economics.preSettlementBalanceAssets,
            traderBefore - commitDebit,
            "receipt pre-balance already excludes the commit debit"
        );
        assertGt(totalDebit, receipt.economics.grossAccountDebitAssets + commitDebit - 1e6, "receipt misses the debit");
    }

    /// @notice No other caller limit can cap this debit for an increase. A positive settlement floor is rejected for
    ///         any account that already has a position, so `maxGrossAccountDebitAssets` and `maxActionChargeAssets`
    ///         are the only debit ceilings an increase can carry, and those are the two limits bypassed above.
    function test_SettlementFloorIsUnavailableToIncreases() public {
        vm.warp(vm.getBlockTimestamp() + 28 days);
        _installPublicMark();
        uint256 bounty = router.maxOpenOrderExecutionBountyAssets();
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        OrderV3Types.OrderRequest memory increase = _request(
            keccak256("FLOORED_INCREASE"), INCREASE_SIZE, INCREASE_MARGIN, false, ACTION_LIMIT + bounty, ACTION_LIMIT
        );
        increase.bounds.minPostSettlementBalanceAssets = traderBefore - (ACTION_LIMIT + bounty);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__SettlementFloorUnsupported.selector);
        vm.prank(TRADER);
        router.commitOrder(increase);
        assertEq(clearinghouse.balanceAssets(TRADER), traderBefore, "rejected commit leaves custody unchanged");
    }

    /// @notice Realism: one week of carry on the same position already exceeds a 50 USDG action limit on a
    ///         10,000 USDG increase whose own trade cost is well inside that limit.
    function test_OneWeekOfCarryExceedsAFiftyUsdgLimit() public {
        vm.warp(vm.getBlockTimestamp() + 7 days);
        _installPublicMark();
        uint256 limit = 50e6;
        uint256 bounty = router.maxOpenOrderExecutionBountyAssets();
        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(TRADER, CfdTypes.Side.LONG, INCREASE_SIZE, INCREASE_MARGIN, MARK);
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);

        OrderV3Types.OrderRequest memory increase =
            _request(keccak256("WEEK_INCREASE"), INCREASE_SIZE, INCREASE_MARGIN, false, limit + bounty, limit);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(increase);
        uint256 commitDebit = traderBefore - clearinghouse.balanceAssets(TRADER);

        _startRecordingLogs();
        bytes[] memory updateData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);

        console2.log("one week: quoted trade cost (fee + VPI)", uint256(preview.tradeCostAssets));
        console2.log("one week: caller maxActionCharge", limit);
        console2.log("one week: carry debited at commitment", commitDebit);
        console2.log("one week: receipt actionChargeAssessedAssets", receipt.economics.actionChargeAssessedAssets);

        assertGt(preview.tradeCostAssets, 0, "trade cost is a charge");
        assertLt(uint256(preview.tradeCostAssets), limit, "the increase's own trade cost fits the limit");
        assertGt(commitDebit, limit, "one week of carry alone exceeds the limit");
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "increase still executes");
        assertLe(receipt.economics.actionChargeAssessedAssets, limit, "receipt appears within the limit");
    }

    /// @notice Breadth: the debit does not depend on the increase executing. Nobody executes the bounded increase, any
    ///         caller expires it after its deadline, and the position is unchanged. The carry taken at commitment
    ///         stays with LPs, and the authenticated failure receipt still records zero commitment carry.
    function test_UnexecutedIncreaseKeepsCommitmentCarryAndItsFailureReceiptOmitsIt() public {
        vm.warp(vm.getBlockTimestamp() + 28 days);
        _installPublicMark();
        uint256 bounty = router.maxOpenOrderExecutionBountyAssets();
        OrderV3Types.OrderRequest memory increase = _request(
            keccak256("EXPIRED_INCREASE"), INCREASE_SIZE, INCREASE_MARGIN, false, ACTION_LIMIT + bounty, ACTION_LIMIT
        );

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();
        vm.prank(TRADER);
        uint64 id = router.commitOrder(increase);
        uint256 commitDebit = traderBefore - clearinghouse.balanceAssets(TRADER);

        vm.warp(uint256(router.lifecycleBook().orderTiming(id).executionDeadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _startRecordingLogs();
        vm.prank(KEEPER);
        router.expireOrder(id);
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        uint256 totalDebit = traderBefore - clearinghouse.balanceAssets(TRADER);

        console2.log("expired: caller maxActionCharge", increase.bounds.maxActionChargeAssets);
        console2.log("expired: carry debited at commitment", commitDebit);
        console2.log("expired: trader settlement debit after expiry", totalDebit);
        console2.log("expired: LP pool asset increase", pool.totalAssets() - poolBefore);
        console2.log("expired: receipt commitment.carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        console2.log("expired: receipt economics.grossAccountDebitAssets", receipt.economics.grossAccountDebitAssets);

        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Failed), "increase never executes");
        assertEq(size, INITIAL_SIZE, "position unchanged");
        assertGt(commitDebit, increase.bounds.maxActionChargeAssets, "commitment debit exceeds the action limit");
        assertGe(totalDebit, commitDebit, "the commitment carry is not returned on expiry");
        assertGe(pool.totalAssets() - poolBefore, commitDebit, "the carry stays with LPs");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "failure receipt records zero commitment carry");
        assertLt(receipt.economics.grossAccountDebitAssets, commitDebit, "failure receipt misses the debit");
    }

    // ---------------------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------------------

    /// @dev Carry the in-scope `CfdEngineLens.previewOpen` attributes to this increase at the current timestamp.
    function _quotedPendingCarry() internal view returns (uint256) {
        return engineLens.previewOpen(TRADER, CfdTypes.Side.LONG, INCREASE_SIZE, INCREASE_MARGIN, MARK)
            .pendingCarryAssets;
    }

    /// @dev Runs the Router's configured policy evaluator on the not-yet-committed increase and returns the
    ///      gross-debit actual it reports when it rejects the request.
    function _expectPolicyRejection(
        OrderV3Types.OrderRequest memory request,
        uint256 bounty
    ) internal view returns (uint256 actual) {
        CfdTypes.Order memory order = CfdTypes.Order({
            account: TRADER,
            sizeDelta: request.sizeDelta,
            marginDelta: request.marginDelta,
            targetPrice: request.targetPrice,
            commitTime: uint64(vm.getBlockTimestamp()),
            commitBlock: uint64(vm.getBlockNumber()),
            orderId: router.nextCommitId(),
            side: request.side,
            isClose: false
        });
        try ICfdOrderPolicyEvaluator(router.policyEvaluator())
            .assessOrder(address(engine), order, KEEPER, MARK, pool.totalAssets(), request.bounds, bounty) returns (
            OrderV3Types.ExecutionAssessment memory
        ) {
            revert("policy evaluator unexpectedly accepted the request");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                "typed constraint violation"
            );
            (OrderV3Types.ConstraintKind kind, uint256 reported, uint256 limit) =
                abi.decode(_stripSelector(reason), (OrderV3Types.ConstraintKind, uint256, uint256));
            assertEq(uint8(kind), uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "gross-debit limit");
            assertEq(limit, request.bounds.maxGrossAccountDebitAssets, "caller's gross-debit limit");
            actual = reported;
        }
    }

    function _stripSelector(
        bytes memory data
    ) internal pure returns (bytes memory out) {
        out = new bytes(data.length - 4);
        for (uint256 i; i < out.length; ++i) {
            out[i] = data[i + 4];
        }
    }

    function _openInitialPosition() internal {
        OrderV3Types.OrderRequest memory request = _request(
            keccak256("INITIAL_OPEN"), INITIAL_SIZE, INITIAL_MARGIN, false, type(uint256).max, type(uint256).max
        );
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "initial open executes");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, INITIAL_SIZE, "initial position size");
    }

    function _installPublicMark() internal {
        bytes[] memory updateData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        router.updateMarkPrice(updateData);
        assertEq(engine.lastMarkPrice(), MARK, "public mark update");
    }

    function _request(
        bytes32 clientId,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxGrossDebit,
        uint256 maxActionCharge
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = MARK;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossDebit,
            maxActionChargeAssets: maxActionCharge,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }
}
```

