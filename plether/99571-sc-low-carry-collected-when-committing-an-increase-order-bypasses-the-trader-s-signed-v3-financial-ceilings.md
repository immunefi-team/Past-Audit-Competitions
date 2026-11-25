# \#99571 [SC-Low] Carry collected when committing an increase order bypasses the trader's signed V3 financial ceilings

**Submitted on Nov 14th 2026 at 09:39:18 UTC by @songthuy307 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99571
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every public V3 order carries user-signed `ExecutionBounds`. The protocol documents them as mandatory financial authority, and zero means zero:

- `OrderV3Types` NatSpec: "Mandatory financial authority supplied with every V3 order … zero is a real zero allowance" ([`src/OrderV3Types.sol:131-151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L151)).
- `SECURITY.md` security-property table, "Bounded authority": "zero maxima are real zero allowances rather than unbounded sentinels" ([`SECURITY.md:368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L368), [`SECURITY.md:756-758`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L756-L758)).
- The trading-integration guide defines `maxActionChargeAssets` as the "Maximum assessed action charge after carry, signed virtual price impact (VPI), fees, and frozen spread are combined", and `maxGrossAccountDebitAssets` as the "Maximum aggregate settlement debit, consumed trader claims, and reserved bounty" ([`WORKING_WITH_AI_AGENTS.md:102-103`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L102-L103)).
- `ACCOUNTING_SPEC.md` applies carry collection to "guard, open/modify, withdrawal, close, and liquidation paths" and states "Receipts and financial bounds include the direct collection once" ([`ACCOUNTING_SPEC.md:696-697`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L696-L697), [`ACCOUNTING_SPEC.md:723-724`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L723-L724)).

When an account commits a **non-close order on an existing position** (an increase), `MarginClearinghouse` collects the account's accrued carry during the commit, before the order is recorded. The commitment effects handed to `OrderLifecycleBook`, however, always report `carryCollectedAssets = 0`, and the committed-order evaluator adds the stored commitment only for closes. The carry collected at commit is therefore never checked against `maxActionChargeAssets` / `maxGrossAccountDebitAssets` and never appears in the receipt. An increase whose signed ceilings should reject it is accepted and executed, and the trader is debited more than the ceilings they signed. A partial close in the identical state with the identical bound is correctly rejected for exactly that carry. Fresh opens from a flat account are not affected because they have no accrued carry.

Code reviewed at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (in-scope freeze).

## Vulnerability Details

Commit flow for a non-close order ([`src/router/OrderCommitHandler.sol:21-36`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L36)):

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
clearinghouse.recordBountyReservation(...);
_onOrderCommitted(order, effects);                 // -> OrderLifecycleBook.recordCommitment(orderId, effects)
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
```

1. For `isClose == false`, `_reserveExecutionBounty` returns an `effects` struct whose only non-zero field is `bountyFromFreeAssets` ([`src/router/OrderReservationAccounting.sol:108-128`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128)). With a non-zero bounty it calls `clearinghouse.lockReservedSettlement(account, bounty)`. That function first runs `_checkpointCarryBeforeMarginChange(account)` ([`src/MarginClearinghouse.sol:818-822`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L822)), which calls `CfdEngine.realizeCarryBeforeMarginChange`: the account's pending carry is collected from margin/free settlement and paid to the HousePool.
2. With a zero bounty the same collection happens in `reserveCommittedOrderMargin` ([`src/MarginClearinghouse.sol:620-632`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L620-L632), called from [`src/router/OrderReservationAccounting.sol:136-146`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L136-L146)). That call runs after `recordCommitment`, so it can never be reported either.
3. `OrderLifecycleBook.recordCommitment` ([`src/OrderLifecycleBook.sol:327-361`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L361)) enforces the bounds only against the reported effects:

   ```solidity
   uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;   // carry reported as 0
   if (gross > pending.bounds.maxGrossAccountDebitAssets) revert ...;
   if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) revert ...;  // 0 > limit is never true
   if (effects.settlementBeforeAssets != effects.settlementAfterAssets + effects.carryCollectedAssets ...) // 0 == 0 + 0
   pending.commitment = StoredCommitment({carryCollectedAssets: effects.carryCollectedAssets, ...});      // stores 0
   ```

4. At execution, `CfdOrderPolicyEvaluator.assessCommittedOrder` adds the stored commitment only for closes ([`src/CfdOrderPolicyEvaluator.sol:65-67`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L65-L67); `_includeCommitment` at [`src/CfdOrderPolicyEvaluatorBase.sol:474-486`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L486)):

   ```solidity
   if (order.isClose) {
       _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
   }
   ```

   The execution-time assessment does count carry accrued **after** the commit against the bounds (31 atoms in PoC 3). The carry collected **at** the commit is in neither place.

The close path measures the same carry in `_reserveCloseExecutionBounty`, reports it in `effects.carryCollectedAssets`, rejects it in `recordCommitment` when it exceeds the bounds, and adds it again at execution. This is what the documentation requires for every path: "Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds" ([`ACCOUNTING_SPEC.md:942-944`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L942-L944); also [`CANONICAL_ENTRYPOINTS.md:100-104`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L100-L104)). No documented exception exists for increases.

## Impact Details

- **Signed ceilings are not enforced on increases.** A trader signs `maxActionChargeAssets = X` and `maxGrossAccountDebitAssets = Y`. The increase is accepted and executed while the account is debited more than `X` and `Y` for that action. This includes a signed zero allowance, which the protocol defines as "a real zero allowance". The protocol's documented "Bounded authority" property fails to hold for this public flow.
- **Measured at a realistic scale with the release parameters from `DEPLOYMENT.md`** (`vpiFactor 0.01e18`, maintenance 10 bps, initial 20 bps, carry 500 bps, execution fee 4 bps): a $160k long held for 28 days in the default $1M test pool, followed by a $16k increase. The trader signs a gross-debit ceiling of **33.420942 USDG**. The account is debited **108.787906 USDG** for the action. Of that, **75.366964 USDG** is commit-time carry that no bound and no receipt field reports. With the test-default parameters the same flow gives a 6.600031 USDG ceiling and an 81.844613 USDG debit. The excess grows with borrow base, utilization and time since the account's last carry checkpoint.
- **Receipts under-report the action's debit.** The authenticated receipt reports `grossAccountDebitAssets = 33.420942 USDG` and `commitment.carryCollectedAssets = 0`. The trading-integration guide tells agents to rely on these bounds and receipts rather than their own accounting ([`WORKING_WITH_AI_AGENTS.md:110-111`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L110-L111)).
- **No value is lost.** The collected amount is carry the trader already owed, and it is credited to LPs. The issue is therefore filed as **Low ("Contract fails to deliver promised returns, but doesn't lose value")**: the protocol fails to deliver its promised bound protection, without loss of funds. No third party can trigger it for another account, because the commit is account-originated. This report does not claim theft, freezing or griefing.

### Why this is not Cecuro MED-9 (Addressed)

MED-9 concerned Router-generated protection close requests created with unlimited financial maxima. The fix snapshots the trader's ceilings for trigger and retry attempts, and that fix is present at the audited commit. This report concerns the ordinary public `commitOrder` path for increases, where the ceilings are present and correct but the carry collected during commitment is not reported to the Book and not included by the evaluator. The root cause, code location and required fix all differ, and fixing MED-9 did not fix this path: the PoC passes at the audited commit, which already contains the MED-9 fix. Cecuro LOW-34 is about the assessed-versus-collected interpretation of close carry in analytics, not about bound enforcement. No listed known issue, `SECURITY.md` known limitation, previous-audit item or public repository PR/issue (checked up to PR #138, 2026-11-13) describes this path.

## References

- Commit entry point: [`src/OrderRouter.sol:85-90`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L85-L90). Commit flow: [`src/router/OrderCommitHandler.sol:21-36`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L36).
- Non-close effects and reservations: [`src/router/OrderReservationAccounting.sol:108-146`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L146).
- Carry checkpoints: [`src/MarginClearinghouse.sol:620-632`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L620-L632), [`src/MarginClearinghouse.sol:818-822`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L822).
- Bound checks on reported effects: [`src/OrderLifecycleBook.sol:327-361`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L361).
- Close-only commitment inclusion: [`src/CfdOrderPolicyEvaluator.sol:65-67`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L65-L67), [`src/CfdOrderPolicyEvaluatorBase.sol:474-486`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L486).
- Intent:
  - [`src/OrderV3Types.sol:131-151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L151)
  - [`SECURITY.md:368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L368)
  - [`SECURITY.md:756-758`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L756-L758)
  - [`WORKING_WITH_AI_AGENTS.md:102-103`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L102-L103)
  - [`ACCOUNTING_SPEC.md:696-697`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L696-L697)
  - [`ACCOUNTING_SPEC.md:723-724`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L723-L724)
  - [`ACCOUNTING_SPEC.md:942-944`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L942-L944)
  - [`CANONICAL_ENTRYPOINTS.md:100-104`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L100-L104)
- Existing tests: `test/perps/matrices/OrderExecutionBoundsMatrix.t.sol` disables carry (`baseCarryBps = 0`) for its open/increase rows. Its only positive-carry bound case is a partial close (`test_CommitmentAndDelayedCarry_GrossAndActionAtomBounds_BothSides`), which passes at the audited commit and shows the close path rejects commitment carry above the bound.

## Recommendation

Account for commit-time carry on the non-close path exactly as on the close path:

- In `_reserveExecutionBounty` for `isClose == false`, checkpoint carry explicitly before locking the bounty and reserving margin. Measure settlement before and after, and fill `carryCollectedAssets`, `carryOutstandingAssets`, `settlementBeforeAssets`, `settlementAfterAssets` and `freeSettlementAfterAssets`. The later same-timestamp checkpoints in `lockReservedSettlement` / `reserveCommittedOrderMargin` then collect nothing. Alternatively, call `recordCommitment` after `_reserveCommittedMargin`, using effects measured across both reservations.
- In `CfdOrderPolicyEvaluator.assessCommittedOrder`, include the stored commitment for every order, not only `order.isClose`, so that execution-time bounds and receipts contain the commit-time carry exactly once.


## Proof of Concept

Runs in the repository's own Foundry suite at the in-scope commit. `BasePerpTest` deploys the production contracts with mocked Pyth updates and mock USDG only, which the rules allow because both are out-of-scope parts. Its `LegacyOrderRouterHarness` is a subclass of the production `OrderRouter` that only adds legacy convenience overloads and overrides nothing. The PoC calls only the production V3 `commitOrder(OrderRequest)` and `executeOrder`. The actors are a trader `0xB0A1` (20k–60k USDG) and a keeper `0xB0A2`. Mark and execution price are 0.80.

Save the file below as `packages/perps/test/perps/matrices/Surf292CommitCarryBoundPoc.t.sol` and run:

```bash
cd packages/perps
forge test --match-path test/perps/matrices/Surf292CommitCarryBoundPoc.t.sol -vv
```

The file contains two contracts: one with the test-default risk parameters, and one with the release parameters from `DEPLOYMENT.md`. Each runs the same three tests:

1. `test_IncreaseCommitChargesCarryThroughZeroActionChargeBound`: from the same state, a partial close with `maxActionChargeAssets = 0` reverts `CommitmentBoundExceeded(ActionCharge, 272, 0)`. An increase with `maxActionChargeAssets = 0` commits, the same 272 atoms are debited and paid to the pool, and the Book stores `carryCollectedAssets = 0`.
2. `test_IncreaseExecutesWithTotalChargeAboveSignedLimit`: the trader signs ceilings equal to the execution-time charges. The increase executes and the total action charge exceeds the signed `maxActionChargeAssets`.
3. `test_RealisticScale_IncreaseChargedAboveSignedCeilings`: reproduces the realistic-scale numbers given in Impact Details (Description). It asserts that post-commit carry is counted (`receipt.economics.carryAssets = 31`) while the 75 USDG commit-time collection is not, and that the settlement debit exceeds the signed `maxGrossAccountDebitAssets`.

Output (Forge 1.5.1, solc 0.8.35, offline):

```
Ran 3 tests for test/perps/matrices/Surf292CommitCarryBoundPoc.t.sol:Surf292CommitCarryBoundPocTest
[PASS] test_IncreaseCommitChargesCarryThroughZeroActionChargeBound() (gas: 4789920)
Logs:
  carry rejected on close commit (atoms) 272
  carry collected on increase commit (atoms) 272
  signed maxActionChargeAssets 0
  recorded commitment.carryCollectedAssets 0
[PASS] test_IncreaseExecutesWithTotalChargeAboveSignedLimit() (gas: 6509868)
Logs:
  commit-time carry (atoms) 272
  execution actionChargeAssessed (atoms) 3200000
  signed maxActionChargeAssets 3200000
  total action charge paid (atoms) 3200272
  receipt.commitment.carryCollectedAssets 0
  receipt.economics.carryAssets 0
  bounty 200000
  total settlement debit 3400272
[PASS] test_RealisticScale_IncreaseChargedAboveSignedCeilings() (gas: 6714405)
Logs:
  commit-time carry (atoms, 1e6 = 1 USDG) 75244582
  signed maxActionChargeAssets 6400031
  signed maxGrossAccountDebitAssets 6600031
  receipt actionChargeAssessedAssets 6400031
  receipt grossAccountDebitAssets 6600031
  receipt commitment.carryCollectedAssets 0
  actual settlement debit for the action 81844613
  receipt executionFeeAssets 6400000
  receipt carryAssets (post-commit accrual only) 31
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 34.67ms (55.86ms CPU time)
Ran 3 tests for test/perps/matrices/Surf292CommitCarryBoundPoc.t.sol:Surf292CommitCarryBoundReleaseParamsPocTest
[PASS] test_IncreaseCommitChargesCarryThroughZeroActionChargeBound() (gas: 4788900)
Logs:
  carry rejected on close commit (atoms) 272
  carry collected on increase commit (atoms) 272
  signed maxActionChargeAssets 0
  recorded commitment.carryCollectedAssets 0
[PASS] test_IncreaseExecutesWithTotalChargeAboveSignedLimit() (gas: 6600547)
Logs:
  commit-time carry (atoms) 272
  execution actionChargeAssessed (atoms) 4796804
  signed maxActionChargeAssets 4796804
  total action charge paid (atoms) 4797076
  receipt.commitment.carryCollectedAssets 0
  receipt.economics.carryAssets 0
  bounty 200000
  total settlement debit 4997076
[PASS] test_RealisticScale_IncreaseChargedAboveSignedCeilings() (gas: 6839444)
Logs:
  commit-time carry (atoms, 1e6 = 1 USDG) 75366964
  signed maxActionChargeAssets 33220942
  signed maxGrossAccountDebitAssets 33420942
  receipt actionChargeAssessedAssets 33220942
  receipt grossAccountDebitAssets 33420942
  receipt commitment.carryCollectedAssets 0
  actual settlement debit for the action 108787906
  receipt executionFeeAssets 6400000
  receipt carryAssets (post-commit accrual only) 31
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 35.64ms (50.76ms CPU time)
Ran 2 test suites in 46.53ms (70.31ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: accrued carry collected while committing a public V3 increase order is never checked against the
///         user-signed `maxActionChargeAssets` / `maxGrossAccountDebitAssets`, while the same carry on a close
///         commitment is rejected by those bounds.
contract Surf292CommitCarryBoundPocTest is BasePerpTest {

    address private constant ACCOUNT = address(0xB0A1);
    address private constant KEEPER = address(0xB0A2);
    uint256 private constant MARK = 80_000_000;
    uint256 private constant SIZE = 10_000e18;
    uint256 private constant MARGIN = 2000e6;
    uint256 private constant FUNDING = 20_000e6;
    uint256 private constant BEFORE_FAD = 1_729_283_399;
    uint256 private constant ACCRUAL = 1200;

    function setUp() public override {
        super.setUp();
        (,,,,, uint256 baseCarryBps,,,,) = engine.riskParams();
        assertEq(baseCarryBps, 500, "default deployment charges carry");
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 1 hours;
        _setRouterConfig(config);
    }

    /// Increase commit with a signed zero action-charge allowance succeeds and still debits carry; a partial close
    /// from the identical state with the identical bound is rejected for exactly that carry.
    function test_IncreaseCommitChargesCarryThroughZeroActionChargeBound() public {
        _seedPositionAndAccrueCarry();
        uint256 seeded = vm.snapshotState();

        // Control: partial close, maxActionChargeAssets = 0 -> CommitmentBoundExceeded(ActionCharge, carry, 0).
        OrderV3Types.OrderRequest memory closeRequest = _request(true);
        closeRequest.bounds.maxActionChargeAssets = 0;
        uint256 closeCarry;
        vm.prank(ACCOUNT);
        try router.commitOrder(closeRequest) returns (uint64) {
            revert("close commit with zero action-charge bound must revert");
        } catch (bytes memory err) {
            assertEq(bytes4(err), IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
            (OrderV3Types.ConstraintKind kind, uint256 actual, uint256 limit) =
                abi.decode(_args(err), (OrderV3Types.ConstraintKind, uint256, uint256));
            assertEq(uint8(kind), uint8(OrderV3Types.ConstraintKind.ActionCharge));
            assertEq(limit, 0);
            closeCarry = actual;
        }
        assertGt(closeCarry, 0, "close commitment observed positive carry");
        assertTrue(vm.revertToState(seeded));

        // Bug: increase, same state, same maxActionChargeAssets = 0 -> commit succeeds and carry is debited.
        OrderV3Types.OrderRequest memory increase = _request(false);
        increase.bounds.maxActionChargeAssets = 0;
        uint256 settlementBefore = clearinghouse.balanceAssets(ACCOUNT);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(increase);
        uint256 carryPaid = settlementToken.balanceOf(address(pool)) - poolBefore;
        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(id);

        console2.log("carry rejected on close commit (atoms)", closeCarry);
        console2.log("carry collected on increase commit (atoms)", carryPaid);
        console2.log("signed maxActionChargeAssets", pending.bounds.maxActionChargeAssets);
        console2.log("recorded commitment.carryCollectedAssets", pending.commitment.carryCollectedAssets);

        assertEq(carryPaid, closeCarry, "increase commit collected the same carry the close bound rejected");
        assertEq(settlementBefore - clearinghouse.balanceAssets(ACCOUNT), carryPaid, "carry debited from trader");
        assertEq(engine.unsettledCarryAssets(ACCOUNT), 0);
        assertEq(pending.bounds.maxActionChargeAssets, 0, "signed zero allowance");
        assertEq(pending.commitment.carryCollectedAssets, 0, "collected carry is not recorded");
    }

    /// End to end: the increase executes with action charge exactly at the signed limit, but the account paid the
    /// commit-time carry on top of it. The terminal receipt omits that carry.
    function test_IncreaseExecutesWithTotalChargeAboveSignedLimit() public {
        _seedPositionAndAccrueCarry();
        uint256 seeded = vm.snapshotState();

        // Twin run with permissive bounds measures commit-time carry and execution-time charges.
        OrderV3Types.OrderRequest memory twinRequest = _request(false);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(ACCOUNT);
        uint64 twinId = router.commitOrder(twinRequest);
        uint256 commitCarry = settlementToken.balanceOf(address(pool)) - poolBefore;
        OrderV3Types.OrderReceipt memory twin = _executeAndReceipt(twinId);
        assertEq(uint8(twin.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 execActionCharge = twin.economics.actionChargeAssessedAssets;
        uint256 execGross = twin.economics.grossAccountDebitAssets;
        assertTrue(vm.revertToState(seeded));

        // Real run: the user signs ceilings equal to the execution-time charges only.
        OrderV3Types.OrderRequest memory request = _request(false);
        request.bounds.maxActionChargeAssets = execActionCharge;
        request.bounds.maxGrossAccountDebitAssets = execGross;
        uint256 settlementBefore = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(request);
        OrderV3Types.OrderReceipt memory receipt = _executeAndReceipt(id);

        uint256 bounty = receipt.bountyAssets;
        uint256 totalDebit = settlementBefore - clearinghouse.balanceAssets(ACCOUNT);
        uint256 totalActionCharge = commitCarry + receipt.economics.actionChargeAssessedAssets;

        console2.log("commit-time carry (atoms)", commitCarry);
        console2.log("execution actionChargeAssessed (atoms)", receipt.economics.actionChargeAssessedAssets);
        console2.log("signed maxActionChargeAssets", request.bounds.maxActionChargeAssets);
        console2.log("total action charge paid (atoms)", totalActionCharge);
        console2.log("receipt.commitment.carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        console2.log("receipt.economics.carryAssets", receipt.economics.carryAssets);
        console2.log("bounty", bounty);
        console2.log("total settlement debit", totalDebit);

        assertGt(commitCarry, 0);
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        assertEq(receipt.economics.actionChargeAssessedAssets, request.bounds.maxActionChargeAssets);
        assertGt(totalActionCharge, request.bounds.maxActionChargeAssets, "signed action-charge ceiling exceeded");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "receipt omits commit-time carry");
    }

    /// Same end-to-end flow at a realistic scale: $160k long held 28 days, then a $16k increase signed with
    /// ceilings equal to its execution-time charges. The commit-time carry is charged on top of the ceiling.
    function test_RealisticScale_IncreaseChargedAboveSignedCeilings() public {
        _seedPositionAndAccrueCarry(20 * SIZE, 20_000e6, 60_000e6, 28 days);
        uint256 seeded = vm.snapshotState();

        OrderV3Types.OrderRequest memory twinRequest = _request(false);
        twinRequest.sizeDelta = 2 * SIZE;
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(ACCOUNT);
        uint64 twinId = router.commitOrder(twinRequest);
        uint256 commitCarry = settlementToken.balanceOf(address(pool)) - poolBefore;
        OrderV3Types.OrderReceipt memory twin = _executeAndReceipt(twinId);
        assertEq(uint8(twin.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 execActionCharge = twin.economics.actionChargeAssessedAssets;
        uint256 execGross = twin.economics.grossAccountDebitAssets;
        assertTrue(vm.revertToState(seeded));

        OrderV3Types.OrderRequest memory request = _request(false);
        request.sizeDelta = 2 * SIZE;
        request.bounds.maxActionChargeAssets = execActionCharge;
        request.bounds.maxGrossAccountDebitAssets = execGross;
        uint256 settlementBefore = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(request);
        OrderV3Types.OrderReceipt memory receipt = _executeAndReceipt(id);
        uint256 totalDebit = settlementBefore - clearinghouse.balanceAssets(ACCOUNT);

        console2.log("commit-time carry (atoms, 1e6 = 1 USDG)", commitCarry);
        console2.log("signed maxActionChargeAssets", request.bounds.maxActionChargeAssets);
        console2.log("signed maxGrossAccountDebitAssets", request.bounds.maxGrossAccountDebitAssets);
        console2.log("receipt actionChargeAssessedAssets", receipt.economics.actionChargeAssessedAssets);
        console2.log("receipt grossAccountDebitAssets", receipt.economics.grossAccountDebitAssets);
        console2.log("receipt commitment.carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        console2.log("actual settlement debit for the action", totalDebit);
        console2.log("receipt executionFeeAssets", receipt.economics.executionFeeAssets);
        console2.log("receipt carryAssets (post-commit accrual only)", receipt.economics.carryAssets);

        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        assertGt(commitCarry, 1e6, "more than 1 USDG of carry escapes the ceilings");
        assertEq(receipt.commitment.carryCollectedAssets, 0);
        // Carry accrued after the commit IS assessed against the bound; only the commit-time collection escapes it.
        assertGt(receipt.economics.carryAssets, 0, "execution-time carry is counted");
        assertLt(uint256(receipt.economics.carryAssets), commitCarry);
        assertGt(commitCarry + receipt.economics.actionChargeAssessedAssets, request.bounds.maxActionChargeAssets);
        assertGt(totalDebit, request.bounds.maxGrossAccountDebitAssets, "debit above signed gross ceiling");
    }

    function _seedPositionAndAccrueCarry() private {
        _seedPositionAndAccrueCarry(2 * SIZE, 2 * MARGIN, FUNDING, ACCRUAL);
    }

    function _seedPositionAndAccrueCarry(
        uint256 seedSize,
        uint256 seedMargin,
        uint256 funding,
        uint256 accrual
    ) private {
        _fundTrader(ACCOUNT, funding);
        // Long accruals start the same weekday/time-of-day earlier so the commit still happens in the Live regime.
        vm.warp(BEFORE_FAD - 1 hours - (accrual > ACCRUAL ? accrual : 0));
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, uint64(vm.getBlockTimestamp()));
        OrderV3Types.OrderRequest memory seed = _request(false);
        seed.sizeDelta = seedSize;
        seed.marginDelta = seedMargin;
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(seed);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "seed must execute");
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, uint64(vm.getBlockTimestamp()));
        vm.warp(vm.getBlockTimestamp() + accrual);
        if (accrual > ACCRUAL) {
            vm.prank(address(router));
            engine.updateMarkPrice(MARK, uint64(vm.getBlockTimestamp()));
        }
    }

    function _request(
        bool isClose
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + 0xB000);
        request.side = CfdTypes.Side.LONG;
        request.targetPrice = MARK;
        request.sizeDelta = SIZE;
        request.marginDelta = isClose ? 0 : MARGIN;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: 3600,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _executeAndReceipt(
        uint64 id
    ) private returns (OrderV3Types.OrderReceipt memory receipt) {
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeOrder(id, data);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        address book = address(router.lifecycleBook());
        for (uint256 i; i < logs.length; i++) {
            if (
                logs[i].emitter != book || logs[i].topics.length != 4
                    || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
            ) {
                continue;
            }
            (,,, OrderV3Types.OrderReceipt memory found) =
                abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (found.orderId == id) {
                return found;
            }
        }
        revert("missing receipt");
    }

    function _args(
        bytes memory err
    ) private pure returns (bytes memory out) {
        out = new bytes(err.length - 4);
        for (uint256 i; i < out.length; i++) {
            out[i] = err[i + 4];
        }
    }

}

/// @notice Same three scenarios with the release risk parameters from packages/perps/DEPLOYMENT.md
///         (vpiFactor 0.01e18, maint 10 bps, init 20 bps, FAD 300 bps, carry 500 bps, execution fee 4 bps).
contract Surf292CommitCarryBoundReleaseParamsPocTest is Surf292CommitCarryBoundPocTest {

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

}
```

