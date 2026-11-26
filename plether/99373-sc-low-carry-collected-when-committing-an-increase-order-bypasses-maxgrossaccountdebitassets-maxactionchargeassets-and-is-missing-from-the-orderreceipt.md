# \#99373 [SC-Low] Carry collected when committing an increase order bypasses `maxGrossAccountDebitAssets` / `maxActionChargeAssets` and is missing from the `OrderReceipt`

**Submitted on Nov 13th 2026 at 12:05:27 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99373
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

A trader signs two limits on every order: `maxGrossAccountDebitAssets` (the most that may leave the account) and `maxActionChargeAssets` (carry + fees). When the trader commits an order that **increases** an existing position, the bounty lock silently collects the account's accrued carry. The Router then reports `carryCollectedAssets = 0`, so neither limit sees that debit at commit. Execution counts only carry accrued after the commit, so it does not see it either. As a result, an increase order that the trader's own limits should reject is committed and executed, the account is debited more than `maxGrossAccountDebitAssets`, and the receipt shows zero carry. The close path counts exactly the same commit-time carry and reverts correctly.

### Vulnerability Details

**1. The open commit reports no carry, but the lock collects it.**

```solidity
// packages/perps/src/router/OrderReservationAccounting.sol:114-127
if (isClose) {
    ...
    return _reserveCloseExecutionBounty(account, sizeDelta);   // close: carry reported
}
@> effects.bountyFromFreeAssets = bounty;                     // open: carryCollectedAssets stays 0
...
@> clearinghouse.lockReservedSettlement(account, bounty);
```

```solidity
// packages/perps/src/MarginClearinghouse.sol:818-824
function lockReservedSettlement(address account, uint256 amountAssets) external onlyReservedSettlementLocker {
@>  _checkpointCarryBeforeMarginChange(account);   // realizes accrued carry -> pool
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

`_checkpointCarryBeforeMarginChange` calls `CfdEngine.realizeCarryBeforeMarginChange` (`CfdEngine.sol:535-550`). That returns early only when `pos.lots == 0`, so for a live position it runs `_realizeCarryFromSettlement` (`CfdEngine.sol:1407-1440`), and the trader's custody drops by the accrued carry.

**2. The commitment bounds check uses the reported `0`.**

```solidity
// packages/perps/src/OrderLifecycleBook.sol:335-352
@> uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;   // 0 + bounty
   if (gross > pending.bounds.maxGrossAccountDebitAssets) revert ...;
@> if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) revert ...; // 0
   if (effects.settlementBeforeAssets != effects.settlementAfterAssets + effects.carryCollectedAssets ...) // 0 == 0 + 0
```

`OrderCommitHandler.sol:27-32` passes these `effects` through unchanged.

**3. Execution only sees carry accrued after the commit.**

```solidity
// packages/perps/src/policy/CfdOrderPolicyEvaluatorBase.sol:342-347
uint256 realizedCarryAssets = snapshot.position.size == 0 ? 0 : delta.pendingCarryAssets; // ~0 after commit checkpoint
assessment.grossAccountDebitAssets = realizedCarryAssets + positiveTradeCostAssets + executionBountyAssets;
assessment.actionChargeAssessedAssets = _netOpenActionCharge(realizedCarryAssets, delta.tradeCostAssets);
```

So the carry collected at commit is checked by neither limit, and it is not recorded in the receipt.

**What the protocol promises:**
- `OrderV3Types.sol:133`: "`maxGrossAccountDebitAssets` covers settlement debit, trader-claim consumption, and the reserved bounty".
- `ACCOUNTING_SPEC.md:723`: "Receipts and financial bounds include the direct collection once".
- `WORKING_WITH_AI_AGENTS.md:102-103`: `maxActionChargeAssets` is the "Maximum assessed action charge after carry".

No document excludes carry collected at open/increase commit from these limits.

### Impact Details

The broken invariant is: **every debit caused by an order stays within the trader's signed `maxGrossAccountDebitAssets` / `maxActionChargeAssets` and appears in the `OrderReceipt`.**

Numbers from the PoC (LONG position, 5% `baseCarryBps`, 30 days without a checkpoint):

| | Signed limit | Actual |
|---|---|---|
| Gross account debit | 4.200000 USDG | **4.589222 USDG** |
| Action charge (carry + fee) | 4.000000 USDG | **4.389222 USDG** |
| Carry in `OrderReceipt` | — | **0** (real: 0.389222) |

- An increase order executes even though the trader's limits should have rejected it.
- The amount over the limit equals the full accrued carry at commit, with no cap. It grows linearly with time since the last checkpoint and with position size.
- Integrations and AI agents that reconcile preview against receipt will not match custody. The only on-chain evidence is the `CarryRealized` event.

Severity context: the carry is debt the trader already owed, so no value is stolen or misallocated and net equity is unchanged. Only the trader's own transaction triggers the issue. Post-trade health checks still run against post-carry state.

### Mitigation

Account for open commits the same way as closes. Snapshot `balanceAssets(account)` before `lockReservedSettlement`, then fill `settlementBeforeAssets`, `settlementAfterAssets` and `carryCollectedAssets` in `effects`, so that `recordCommitment` bounds the carry and the receipt records it. Alternatively, checkpoint carry through the engine for increases, as `reserveCloseOrderExecutionBounty` does, and return those effects.

### References

- `packages/perps/src/router/OrderReservationAccounting.sol:114-127`: open path leaves `carryCollectedAssets = 0`
- `packages/perps/src/MarginClearinghouse.sol:818-824`: `lockReservedSettlement` checkpoints carry (`:822`)
- `packages/perps/src/CfdEngine.sol:535-550`, `:1407-1440`: carry realization
- `packages/perps/src/router/OrderCommitHandler.sol:27-33`: effects passed to `recordCommitment`
- `packages/perps/src/OrderLifecycleBook.sol:335-352`: commitment bounds check
- `packages/perps/src/policy/CfdOrderPolicyEvaluatorBase.sol:342-347`: execution counts only post-commit carry
- `packages/perps/src/OrderV3Types.sol:133`, `ACCOUNTING_SPEC.md:723`: documented guarantees

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/poc/OpenCommitCarryOutsideBounds.t.sol`. It uses the repo's existing `CfdClosePreviewTestBase` fixture. Run:

```
cd packages/perps && forge test --match-path test/perps/poc/OpenCommitCarryOutsideBounds.t.sol -vv
```

- `test_openCommitCarryOutsideBounds`: control case, where a close commit with a tight bound reverts with `CommitmentBoundExceeded`. In the same state, an open commit with `maxGross = bounty` succeeds while custody drops by the carry. The recorded `carryCollectedAssets` is 0.
- `test_openLifecycleTotalDebitExceedsGrossBound`: full lifecycle with `maxGross = bounty + fee`. The order executes, total custody debit is `maxGross + carry`, and both receipt carry fields are 0.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdClosePreviewTestBase} from "../CfdClosePreviewTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";

contract OpenCommitCarryOutsideBoundsPoC is CfdClosePreviewTestBase {
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.baseCarryBps = 500;
    }

    function _req(bool isClose, uint256 size, uint256 margin, uint256 maxGross)
        internal view returns (OrderV3Types.OrderRequest memory r)
    {
        r.clientOrderId = keccak256(abi.encode("oc", router.nextCommitId(), isClose));
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = isClose ? 1 : PRICE;
        r.isClose = isClose;
        r.bounds = _bounds();
        r.bounds.maxGrossAccountDebitAssets = maxGross;
        r.bounds.maxActionChargeAssets = isClose ? maxGross : 0;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
    }

    function test_openCommitCarryOutsideBounds() public {
        _openNormally(CfdTypes.Side.LONG, 700e6);
        vm.warp(vm.getBlockTimestamp() + 30 days);
        _mockPythUpdateData(PRICE);

        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 custodyBefore = clearinghouse.balanceAssets(ACCOUNT);
        console2.log("bounty", bounty);
        console2.log("expected pending carry", _expectedIndexedCarryAssets(ACCOUNT));

        // Control: close commit with a tight bound reverts (carry is counted)
        OrderV3Types.OrderRequest memory c = _req(true, SIZE, 0, 1);
        vm.prank(ACCOUNT);
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        router.commitOrder(c);

        // Increase commit: maxGross = bounty, maxActionCharge = 0
        OrderV3Types.OrderRequest memory o = _req(false, SIZE, 100e6, bounty);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(o);
        uint256 debited = custodyBefore - clearinghouse.balanceAssets(ACCOUNT);
        OrderV3Types.PendingIntent memory pi = router.lifecycleBook().pendingIntent(id);
        console2.log("custody debited by commit", debited);
        console2.log("recorded carryCollected", pi.commitment.carryCollectedAssets);
        console2.log("maxGross", o.bounds.maxGrossAccountDebitAssets);
        assertGt(debited, o.bounds.maxGrossAccountDebitAssets);
        assertEq(pi.commitment.carryCollectedAssets, 0);
    }

    function test_openLifecycleTotalDebitExceedsGrossBound() public {
        _openNormally(CfdTypes.Side.LONG, 700e6);
        vm.warp(vm.getBlockTimestamp() + 30 days);
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 fee = _engineExecutionFeeAssets(SIZE, PRICE);
        uint256 maxGross = bounty + fee;           // "at most bounty + fee may leave my account"
        OrderV3Types.OrderRequest memory o = _req(false, SIZE, 100e6, maxGross);
        o.bounds.maxActionChargeAssets = fee;      // carry + fee <= fee  =>  carry must be 0

        uint256 pendingCarry = _expectedIndexedCarryAssets(ACCOUNT);
        console2.log("pending carry before commit", pendingCarry);
        assertGt(pendingCarry, 0);

        uint256 custody0 = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(o);
        OrderV3Types.ExecutionAssessment memory a =
            policyEvaluator.assessCommittedOrder(address(engine), id, KEEPER, PRICE);
        console2.log("exec assessed carry", a.carryAssets);
        console2.log("exec gross", a.grossAccountDebitAssets);
        console2.log("exec action charge", a.actionChargeAssessedAssets);

        vm.recordLogs();
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(id, update);
        Vm.Log[] memory logs = vm.getRecordedLogs();

        (uint256 size,,,,,,) = engine.positions(ACCOUNT);
        assertEq(size, 2 * SIZE, "increase executed");
        uint256 debit = custody0 - clearinghouse.balanceAssets(ACCOUNT);
        console2.log("maxGrossAccountDebitAssets", maxGross);
        console2.log("maxActionChargeAssets", o.bounds.maxActionChargeAssets);
        console2.log("total custody debit for this order", debit);
        assertEq(debit, maxGross + pendingCarry, "carry collected at commit escaped both bounds");

        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint64(uint256(logs[i].topics[1])) == id
            ) {
                (,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                console2.log("receipt commitment carry", receipt.commitment.carryCollectedAssets);
                console2.logInt(receipt.economics.carryAssets);
                assertEq(receipt.commitment.carryCollectedAssets, 0);
                assertEq(receipt.economics.carryAssets, 0);
                return;
            }
        }
        fail("no receipt");
    }
}
```

Result (actually run):

```
[PASS] test_openCommitCarryOutsideBounds()
  bounty 200000
  expected pending carry 389222
  custody debited by commit 389222
  recorded carryCollected 0
  maxGross 200000
[PASS] test_openLifecycleTotalDebitExceedsGrossBound()
  pending carry before commit 389222
  exec assessed carry 0
  exec gross 4200000
  exec action charge 4000000
  maxGrossAccountDebitAssets 4200000
  maxActionChargeAssets 4000000
  total custody debit for this order 4589222
  receipt commitment carry 0
  0
```

