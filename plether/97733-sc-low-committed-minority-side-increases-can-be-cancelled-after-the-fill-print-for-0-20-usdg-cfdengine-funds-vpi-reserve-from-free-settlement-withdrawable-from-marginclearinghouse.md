# \#97733 [SC-Low] Committed minority-side increases can be cancelled after the fill print for 0.20 USDG: CfdEngine funds VPI reserve from free settlement withdrawable from MarginClearinghouse

**Submitted on Nov 2nd 2026 at 21:57:08 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97733
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

Plether orders are binding: "Users cannot cancel queued orders once committed" (`packages/perps/SECURITY.md:366`), so that traders cannot turn "queued intents into free options" (`SECURITY.md:658-659`). On an increase that grows the account's VPI rebate reserve, execution funds that growth from the account's free settlement before the order's margin. Commit admission counts on that free settlement, but the commit reserves only the order's margin and bounty. After the fill print is public, the owner withdraws the free settlement, or reserves it with a second order, and the committed increase fails terminally. Its margin is refunded; the owner pays the 0.20 USDG opening bounty.

- Impact: Contract fails to deliver promised returns, but doesn't lose value
- Severity: Low
- Attacker needs: no role; a minority-side position whose lifetime VPI is negative (a minority-side open that stays the minority starts that way), free settlement, and a keeper that executes after a later print is public
- Measured, 2,000,000-token increase against a 20,000,000-token opposite position, with a later print 30 bps against the order: the attacker cancels for 0.20 USDG. Bound by the same order and closed at the later print, the attacker loses 7,802.49, the pool gains 6,199.89 and the treasury 1,602.40
- Stated rule: `SECURITY.md:366` ("Users cannot cancel queued orders once committed") and `SECURITY.md:658-659` ("binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options")
- Reproduce: `forge test --match-path test/perps/CommittedOrderFreeCancel.t.sol -vv` (from `packages/perps`, audit repository commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)

## Affected assets

| Component (all in `packages/perps/src`) | Role | Address (Arbitrum Sepolia v2.0.0) |
|---|---|---|
| `libraries/CfdEnginePlanLib.sol` in CfdEnginePlanner | plans the reserve funding, for the commit preview and at execution | 0x802a0755413fb966b397155b5c5cc0198e6bed3f |
| CfdEngineLens | runs the commit preview through CfdEnginePlanner (`CfdEngineLens.sol:348`) | 0xfca322c6868cef53bd5c344628f9874712e48108 |
| OrderRouter | admits the order, reserves margin and bounty | 0x918663d6da8cb33b325a04c4a5e8c51d42f3b444 |
| MarginClearinghouse | `withdraw` | 0xb5b851f627d6df3cf3f096e23e20a58884d7b89a |
| CfdEngineSettlementSidecar | withdrawal guard | 0xd3ede337e042d5aa6ba68e03d0c885b3d4a062dc |
| HousePool | LP counterparty to the order | 0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e |

Scope rows: MarginClearinghouse.sol, HousePool.sol, and "All other contracts, libraries, router contracts and interfaces in packages/perps/src". The program states that `packages/perps/src` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is identical to the v2.0.0 release source.

## Preconditions

- Attacker: no role, one account, holding a minority-side position whose lifetime VPI is negative. A minority-side open that stays the minority is paid a rebate and starts that way, and a further such increase grows the VPI rebate reserve. At the Arbitrum Sepolia deployment's open interest (block 325,366,627: LONG 1,261,800, SHORT 201,400 tokens) SHORT is the minority side, and `test_attack_deployedOpenInterest` uses those sizes.
- The committed increase carries the smallest margin the Router admits. By the code, enough margin to fund the whole reserve keeps the order binding.
- External: a print that moves against the order after its fill print and before execution. The fill is the first print after commit (`SECURITY.md:647`); the PoC executes 3 s after commit. On the Arbitrum Sepolia deployment, all 19 orders executed since v2.0.0, up to block 325,363,991, took 3 to 9 s from commit to execution (median 7 s, from public event block timestamps). The program states "This deployment is for testing only", so production keeper timing may differ. Any caller may execute earlier, and the attacker's transaction must land first.
- Known issues a reader may apply: SC-M4 (pre-audit, marked fixed), HIGH-1 and the scan finding "State-driven VPI increases are classified as user-invalid margin failures". The Severity section answers each.

## Root cause

1. Execution funds new VPI rebate reserve from the account's free settlement first and from the order's new pledge only for the rest. The liquidation reserve must then fit in what is left of the pledge. `packages/perps/src/libraries/CfdEnginePlanLib.sol:564-577`:

```solidity
        uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
            ? vpiReserveIncreaseAssets
            : openCostPlan.resultingFreeSettlementAssets;
        delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
        if (delta.vpiRebateReserveFromPledgeAssets > newPledgeAvailableAssets) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.VPI_REBATE_RESERVE_UNFUNDED;
            return;
        }

        newPledgeAvailableAssets -= delta.vpiRebateReserveFromPledgeAssets;
        if (reserveIncreaseAssets > newPledgeAvailableAssets) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.LIQUIDATION_RESERVE_UNFUNDED;
            return;
        }
```

The order's own rebate is credited as free settlement net of the fee (`libraries/MarginClearinghouseAccountingLib.sol:148-151`). In the PoC the rebate is 3,399.38 and the fee 800.00. Once the other free settlement is gone, the 800.00 comes from the 2,000.00 pledge, and the 2,000.00 liquidation-reserve increase no longer fits.

2. Commit admission runs the same plan on the account's current free settlement and rejects predictable failures (`router/OrderLiquidationBatchLogic.sol:430-444`). The commit then reserves only the order margin and the opening bounty (`router/OrderReservationAccounting.sol:120-127`, `:136-146`).

3. The withdrawal guard checks the open position's carry, VPI reserve and health, not pending orders (`CfdEngineSettlementSidecar.sol:283-304`). Its only pending-order check covers a flat account's first open with a settlement minimum (`CfdEngineSettlementSidecar.sol:261-272`).

4. A typed planner rejection is terminal: the order is `Failed`, its margin is refunded and the keeper is paid the opening bounty (`SECURITY.md:716-728`).

The broken rule: a committed open's execution depends on settlement that its owner controls until execution, so the owner decides after the fill print whether the binding order executes.

The correct sibling is the liquidation reserve, which only the order's own pledge funds (`CfdEnginePlanLib.sol:574-577`). In `test_negative_selfFundedIncreaseStaysBinding`, an increase with no VPI reserve growth, the smallest admitted margin covers fee and reserve, and the same withdrawal does not fail it.

## Attack path

Lever 1, withdrawal (`test_attack_withdrawalCancelsCommittedIncrease`):

1. Another trader holds a 20,000,000-token SHORT. The attacker holds a 2,000,000-token LONG on the minority side, with negative lifetime VPI.
2. The attacker commits a 2,000,000-token increase with 2,000.00 USDG margin, the smallest the Router admits.
3. The fill print, one second after commit, is 1.00. A later print is 1.003, 30 bps against the order.
4. Before execution, the attacker withdraws all free settlement, 4,947,199.60 USDG. The withdrawal guard passes.
5. The keeper executes with the fill print. The order fails with `PlannerRejected`, code 9 `LIQUIDATION_RESERVE_UNFUNDED`. The margin is refunded and the keeper is paid 0.20 USDG.

When the later print favours the order, the attacker skips step 4 and the order executes.

Lever 2, no withdrawal (`test_attack_blockerOrderCancelsWithoutWithdrawal`): at step 4 the attacker commits a second 2,000-token LONG whose margin and bounty leave 0.80 USDG of free settlement and whose target price can never be met. The first order fails with code 9, the second on slippage, and both margins are refunded. The two bounties cost 0.40 USDG.

## Impact

USDG, from just before the fill print. 2,000,000-token increase, later print 30 bps against the order, unless stated.

| Test | Order | Attacker | HousePool | Treasury | Keeper |
|---|---|---|---|---|---|
| `test_attack_withdrawalCancelsCommittedIncrease` | Failed, code 9 | -0.20 | +0.00 | 0 | +0.20 |
| `test_attack_blockerOrderCancelsWithoutWithdrawal` | Failed, code 9 | -0.40 | +0.00 | 0 | +0.40 |
| `test_attack_optionAgainstBindingOrder`, binding control, closed at the later print | Executed | -7,802.49 | +6,199.89 | +1,602.40 | +0.20 |
| `test_attack_optionAgainstBindingOrder`, favourable later print, kept and closed | Executed | +4,202.31 | -5,800.11 | +1,597.60 | +0.20 |
| `test_attack_deployedOpenInterest`, 800,000-token SHORT increase, margin 1,067.17, print 30 bps lower | Failed, code 9 | -0.20 | +0.00 | 0 | +0.20 |

The pool's +0.00 is under 0.001 USDG of carry for the elapsed seconds, and each cancelled position is unchanged. Without the withdrawal the same orders execute (`test_control_noWithdrawalOrderExecutes`, and the control inside `test_attack_deployedOpenInterest`). If the PoC's two later prints, 30 bps either way, are equally likely, the attacker's mean is +2,101.05 with the withdrawal and -1,800.09 when bound; the pool's is -2,900.05 and +199.89.

## Severity

The PoC shows the program's row "Contract fails to deliver promised returns, but doesn't lose value". The protocol states that committed orders cannot be cancelled: "Users cannot cancel queued orders once committed" (`SECURITY.md:366`), "Users cannot cancel a binding ordinary order." (`ACCOUNTING_SPEC.md:1266`) and "Queued binding orders cannot be cancelled by users." (`WHITEPAPER.md:1816`). That makes LPs the counterparty to the order's outcome at its fill print.

In the adverse run the pool receives 0.000120 USDG of carry and the treasury 0. Bound and closed at the later print, the same order gives the pool 6,199.89 and the treasury 1,602.40. When the later print favours the order, the pool pays what it pays on a binding order. Per run, the PoC shows a return not delivered, not value taken.

The withdrawal guard already protects one pending-order case. `ACCOUNTING_SPEC.md:929-931` reads: "Zero-minimum orders retain ordinary availability. This protection prevents owner withdrawals from manufacturing a settlement-minimum failure; it does not freeze market-dependent financial bounds." That guard covers a flat account's first open with a positive minimum (`test_control_settlementFloorGuardRefusesWithdrawal`). The recommended fix keeps ordinary availability: with it, a full withdrawal of free settlement still succeeds and the order still executes (`test_negative_fixKeepsCommittedIncreaseBinding`).

The failure here is not market-dependent: at the same fill print the order executes when the owner does nothing. `SECURITY.md:1367` makes "known typed planner/policy failures" terminal; here the owner selects that failure after the fill print, and lever 2 uses no withdrawal at all.

SC-M4 ("Risk Free Option in 60Sec Oracle Window", marked "Fixed in Plether-Fi/plether-core#24") reached the same outcome through LP depth. Its component 3 can "trigger a full margin refund on the losing order" after "determining the profitable direction".

At commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 fills use the first print after commit and orders are binding. Under the program's rule for fixed findings, the PoC shows that a committed losing order can still be failed after its fill print and refunded at the commit in scope, here through its reserve funding.

HIGH-1 covers a commit placed after a price is visible; here the order is committed before its fill print. The scan finding covers failures caused by other accounts' skew or pool-depth changes; here the owner causes the failure, with code 9.

## Recommended fix

Fund new VPI reserve only from what the order brings, its own rebate credit and then its pledge, never from the account's other free settlement:

```diff
--- a/packages/perps/src/libraries/CfdEnginePlanLib.sol
+++ b/packages/perps/src/libraries/CfdEnginePlanLib.sol
@@ -561,7 +561,12 @@ library CfdEnginePlanLib {
         uint256 vpiReserveIncreaseAssets = delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets
             ? delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets
             : 0;
-        uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
-            ? vpiReserveIncreaseAssets
+        // Fund new VPI reserve only from cash this order brings: its own rebate credit, then its new pledge.
+        // The account's other free settlement stays withdrawable and reservable until execution, so admission
+        // at commit must not count on it.
+        uint256 ownCreditAssets = openCostPlan.settlementCreditAssets < openCostPlan.resultingFreeSettlementAssets
+            ? openCostPlan.settlementCreditAssets
             : openCostPlan.resultingFreeSettlementAssets;
+        uint256 vpiReserveFromFreeAssets =
+            vpiReserveIncreaseAssets < ownCreditAssets ? vpiReserveIncreaseAssets : ownCreditAssets;
         delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
```

The patch in the PoC link makes the same change in the open quoter (`CfdEngineOpenQuoter.sol:330-332`; out of scope, it only keeps maximum-size quotes consistent). Settlement already moves exactly the planned amount (`CfdEngineSettlementSidecar.sol:466-477`).

Why it is enough: the commit preview runs the same plan, so the smallest admitted margin rises to cover the fee-sized part (2,800.00 USDG instead of 2,000.00 in the PoC). Execution of the reported increase then needs only the order's reserved margin and its own rebate, so neither a withdrawal nor a second order can fail it. A guard on withdrawals alone would leave lever 2 open.

With the patch, the 2,000.00 margin is refused at commit, the increase at 2,800.00 executes against both levers, and 267 of 271 tests in 24 related repository test files (57 suites) pass unchanged. The four that change assume the current funding: two admission properties for healing opens and increases, and two reference-ledger checks; the README lists them. If zero-margin healing increases must stay, reserve the free-funded part at commit, as the opening bounty is, instead of moving it to pledge. That part should cover everything execution would draw from the account's free settlement, the new VPI reserve and any fee.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L545-L580
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L139-L187
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L404-L455
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L116-L146
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L254-L305
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L640-L660
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1343-L1371
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L917-L931
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf


## Link to Proof of Concept

https://gist.github.com/Xylzlol/946ab429a029e5eaf55f031a47cc6c39

## Proof of Concept

Files in the proof-of-concept link: `CommittedOrderFreeCancel.t.sol` (the test), `free-cancel-fix.patch` (the recommended fix, for the optional check) and `README.md` (setup and full output).

Requirements: Foundry (checked with forge 1.8.5 and 1.5.1); the audit repository at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 with its submodules. The test uses the repository's `BasePerpTest` deployment with the release parameters from `packages/perps/DEPLOYMENT.md`; only Pyth prices (MockPyth) and USDG (MockSettlementToken) are mocks. `BasePerpTest` deploys the repository's `LegacyOrderRouterHarness`, a test subclass of OrderRouter that adds only a test marker and a legacy scalar `commitOrder` overload and overrides nothing; this test commits through the production `commitOrder`.

```
cp CommittedOrderFreeCancel.t.sol <repo>/packages/perps/test/perps/
cd <repo>/packages/perps
forge test --match-path test/perps/CommittedOrderFreeCancel.t.sol -vv
```

Expected output (abridged; the full output is in the README). Order status 2 is Executed and 3 Failed; terminal reason 7 is PlannerRejected and 3 Slippage; planner code 9 is `LIQUIDATION_RESERVE_UNFUNDED`. Amounts are USDG atoms (6 decimals).

```
[PASS] test_attack_withdrawalCancelsCommittedIncrease()
  order margin (smallest admitted) 2000000000
  == attack: withdrawal after the fill print, later print 30 bps adverse
    order status, terminal reason, planner code 3 7 9
    attacker position size before, after (tokens) 2000000 2000000
    opening bounty reserved, free settlement withdrawn 200000 4947199600000
    change: attacker, keeper, pool, treasury
  -200120
  200000
  120
  0
[PASS] test_attack_blockerOrderCancelsWithoutWithdrawal()
  free settlement before and after the blocker commit 4947199600000 800000
    committed increase: status, terminal reason, planner code 3 7 9
    blocker: status, terminal reason 3 3
    change: attacker, keeper, pool, treasury
  -400120
  400000
  120
  0
[PASS] test_attack_optionAgainstBindingOrder()
  == option: change per party (USDG atoms): attacker, pool, treasury, keeper
  favourable print, order kept
  4202307874
  -5800107874
  1597600000
  200000
  adverse print, order cancelled by withdrawal
  -200120
  120
  0
  200000
  adverse print, order binding (control)
  -7802492126
  6199892126
  1602400000
  200000
[PASS] test_attack_deployedOpenInterest()
[PASS] test_control_noWithdrawalOrderExecutes()
[PASS] test_control_settlementFloorGuardRefusesWithdrawal()
[SKIP: set FREE_CANCEL_FIX=true after applying the fix] test_negative_fixKeepsCommittedIncreaseBinding()
[PASS] test_negative_selfFundedIncreaseStaysBinding()
Suite result: ok. 7 passed; 0 failed; 1 skipped
```

Core of the attack, copied from `CommittedOrderFreeCancel.t.sol` (setup, price and order helpers are in the file):

```solidity
    function test_attack_withdrawalCancelsCommittedIncrease() public {
        (uint64 id, uint256 margin) = _commitMinorityIncrease();
        console2.log("order margin (smallest admitted)", margin);
        Outcome memory w = _runQueued(id, ONE, ONE + MOVE, true);
        _log("attack: withdrawal after the fill print, later print 30 bps adverse", w);
        _assertCancelledForBounty(w);
    }

    function _runQueued(
        uint64 id,
        uint256 fillPrice,
        uint256 laterPrice,
        bool withdraw
    ) internal returns (Outcome memory w) {
        uint64 commitTime = _commitTime(id);
        w.bounty = clearinghouse.totalBountyReservationsAssets(attacker);
        w.sizeBefore = _size(attacker);
        w.before = _balances();
        _step(1); // the fill print (commit + 1 s) is public
        _step(1);
        router.updateMarkPrice(_live(laterPrice, _now()));
        if (withdraw) {
            w.withdrawn = clearinghouse.getAccountAssetBuckets(attacker).freeSettlementAssets;
            vm.prank(attacker);
            clearinghouse.withdraw(attacker, w.withdrawn);
        }
        _step(1);
        (OrderV3Types.ExecutionResult memory r, OrderV3Types.OrderReceipt memory rc) =
            _execRecorded(id, _hist(fillPrice, commitTime));
        w.status = r.status;
        w.reason = r.terminalReason;
        w.code = rc.failure.code;
        w.sizeAfter = _size(attacker);
        w.afterExec = _balances();
    }

    function _assertCancelledForBounty(
        Outcome memory w
    ) internal pure {
        assertGt(w.withdrawn, 0, "free settlement withdrawn");
        assertEq(uint8(w.status), uint8(OrderV3Types.LifecycleStatus.Failed), "committed order fails terminally");
        assertEq(uint8(w.reason), uint8(OrderV3Types.TerminalReason.PlannerRejected), "planner rejection");
        assertTrue(
            w.code == LIQUIDATION_RESERVE_UNFUNDED || w.code == VPI_REBATE_RESERVE_UNFUNDED, "reserve funding code"
        );
        assertEq(w.sizeAfter, w.sizeBefore, "position unchanged");
        // The withdrawal checkpoints the open position's carry for the seconds elapsed; it is owed in every run.
        int256 carry = _i(w.before.attacker) - _i(w.afterExec.attacker) - int256(w.bounty);
        assertGe(carry, 0, "carry nonnegative");
        assertLe(carry, 1000, "attacker pays the opening bounty plus < 0.001 USDG carry");
        assertEq(_i(w.afterExec.pool) - _i(w.before.pool), carry, "pool receives only the elapsed carry");
        assertEq(_i(w.afterExec.keeper) - _i(w.before.keeper), int256(w.bounty), "keeper receives the bounty");
        assertEq(w.afterExec.treasury, w.before.treasury, "treasury unchanged");
    }
```

