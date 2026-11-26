# \#98054 [SC-Low]  Trader can cancel a committed skew-reducing open after its fill price is known

**Submitted on Nov 4th 2026 at 16:31:53 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98054
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description



## Summary

A delayed order's fill price is fixed by the first Pyth tick after its commit, inside the 15-second
`orderSettlementWindow` (`PletherOracle.sol:87`), but the order stays executable until a deadline the trader sets,
up to 60 seconds by default. Commit reserves only `marginDelta` and a bounty capped at $0.20; the execution fee and
any new reserves are funded from whatever free settlement the account holds when a keeper executes.

When the open reduces skew it earns a negative-VPI rebate that must be reserved, and the planner funds that reserve
from free settlement before it touches the order's own margin. The withdrawal guard never checks whether a pending
order is still funded. So once the fill price is known the trader chooses: let the order execute, or withdraw the
free settlement so execution fails terminally. Self-execution refunds the bounty, so abandoning the order costs only
gas.

That defeats a property the protocol states explicitly — "binding order semantics and the lack of a user
cancellation path prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`, repeated
at `:719`). It is distinct from the acknowledged clock-alignment limitation at `SECURITY.md:701-711`: that one
concerns a fill tick the trader may already have seen, after which the order still fills. Here the trader decides
whether to fill at all, after seeing the price.

## Description

**1. Commit pins the price but reserves only part of what execution needs.**
`commitOrder` reserves `marginDelta` as committed-order margin and locks the execution bounty
(`router/OrderReservationAccounting.sol:127`, `:145`), capped at $0.20 by default (`router/OrderRouterBase.sol:74`).
The execution fee, the liquidation-reserve increase and any new VPI reserve are not reserved. The fill price is the
first tick strictly after `commitTime` within the settlement window (`PletherOracle.sol:610-612`), while the order
lives until `commitTime + executionWindowSeconds` — 60 seconds by default
(`router/OrderExecutionOrchestrator.sol:11`), one hour at the admin ceiling (`OrderRouterAdmin.sol:31`).

**2. A skew-reducing open earns a rebate that must be reserved.**
VPI is the difference between post- and pre-trade skew cost (`CfdMath.sol:118-140`, `:164-180`), so an open on the
lighter side has negative VPI: a rebate. A negative lifetime VPI balance must be fully backed, and the open plan
sets the target at `abs(vpiAccrued + vpiAssets)` (`libraries/CfdEnginePlanLib.sol:343-349`).

**3. The reserve is funded from free settlement before the order's own margin.**
`_planOpenReserveFunding` (`libraries/CfdEnginePlanLib.sol:546-580`):

```solidity
uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
if (delta.vpiRebateReserveFromPledgeAssets > newPledgeAvailableAssets) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.VPI_REBATE_RESERVE_UNFUNDED;
    return;
}
```

The sidecar releases the order's committed margin back to free settlement before planning
(`OrderRouterExecutionSidecar.sol:498`), and the rebate is credited to free settlement as a negative trade cost
(`libraries/MarginClearinghouseAccountingLib.sol:148-151`), short of the reserve target by exactly the fee. Netting
those:

```
fromPledge = max(0, marginDelta + fee - freeSettlementAtPlanTime)
postMargin = marginDelta - fromPledge - liquidationReserveIncrease
```

So an order sized at initial margin plus the liquidation reserve, with a fee-sized buffer of free settlement, passes
the commit preflight (`router/OrderLiquidationBatchLogic.sol:432-445`). Remove the buffer and the same order can no
longer fund its reserve.

**4. Nothing stops the buffer from leaving after commit.**
`validateWithdraw` inspects a pending order only through the optional settlement floor, and only for a flat account:
with `minPostSettlementBalanceAssets == 0` it returns immediately (`CfdEngineSettlementSidecar.sol:261-273`). An
account holding a position runs the health path, which tests pledge plus same-account claim and never asks whether a
pending order is funded (`:287-305`).

**5. The failure is terminal and the bounty comes back.**
`VPI_REBATE_RESERVE_UNFUNDED`, `LIQUIDATION_RESERVE_UNFUNDED` and `INSUFFICIENT_INITIAL_MARGIN` are execution-time
user-invalid (`libraries/CfdEnginePlanLib.sol:112-129`), so the order is finalized `Failed` and its margin released
in full. If the trader executes its own order, the bounty is returned to the account rather than paid to a keeper
(`OrderRouter.sol:336-337`).

## Impact

LP returns in this protocol are the pool's accrued trading revenue: trader losses, carry and fees raise HousePool
NAV, and trader gains are paid out of that NAV before any of it is distributed or redeemed. This option draws value
straight out of that undistributed revenue. LPs underwrite the other side of orders the protocol declares binding;
an order that can be abandoned after its price is known is a different instrument, and nothing in the pool's
accounting charges for it.

It is an option rather than a trade because the decision is made after the outcome is observed: the trader sees the
realized fill, watches the rest of the window, and only then keeps or discards it. The premium is zero.

Measured at launch risk parameters (`script/DeployPerpsArbitrumSepolia.s.sol:435-446`: `vpiFactor` 0.01e18,
`maxSkewRatio` 0.4e18, `initMarginBps` 20, `maintMarginBps` 10, `bountyBps` 10; fee 4 bps at `CfdEngine.sol:147`):

| | $10,000 notional | $5,000,000 notional |
| --- | --- | --- |
| Pool depth / pre-existing skew | $1,002,000 / $300,000 | $200,002,000 / $60,000,000 |
| VPI rebate earned | $29.43 | $14,368.39 |
| Initial margin + liquidation reserve + fee | $20 + $10 + $4 | $10,000 + $5,000 + $2,000 |
| Account funded | $40 | $19,500 |
| Withdrawal that kills the order | $6 | $3,000 |

About $17,000 of capital therefore controls a look-back option on $5,000,000 of notional. The payoff is measured in
`test_Z03`, which commits at a $1.00 tick, lets the market move 30 bp over the next 45 s — still inside the 60 s
deadline — and then has a keeper execute. Because settlement is pinned to the first tick after commit, the position
opens at the stale $1.00 while the market is already elsewhere (`entryPrice == $1.00` is asserted):

| Branch at the decision point | Trader | HousePool |
| --- | --- | --- |
| Market 30 bp favourable, fill at the stale tick | **+$10,906.42** | **−$14,912.62** |
| Market 30 bp adverse, forced to fill | −$14,094.20 | +$10,100.00 |
| Market 30 bp adverse, order abandoned | $0.00 | $0.00 |

The pool's loss on the favourable branch is accounted for exactly: $10,906.42 to the trader, $4,006.00 in protocol
fees and $0.20 of keeper bounty, asserted to equal the pool's decrease to the unit. The third row is asserted to
leave both the trader's balance and `pool.totalAssets()` exactly unchanged.

The trader keeps row one and takes row three instead of row two, so the swing per strike is $25,000.62 on $19,500 of
posted capital. It is repeatable, and needs no privileged access, keeper cooperation or oracle manipulation.

Two conditions bound frequency, not the mechanism: the open must reduce an existing imbalance to earn a rebate, and
in the adverse branch the withdrawal must land before a keeper executes — which the $0.20 bounty cap makes unlikely
to be a real race. Severity is High rather than Critical because the gain is contingent on a price move and arrives
through an otherwise legitimate fill, not an unauthorized transfer of funds at rest.

**Recommendation.** Reserve at commit everything execution will draw from free settlement — the fee, the projected
VPI reserve increase and the liquidation-reserve increase — not just `marginDelta`. Alternatively, have the
withdrawal guard account for the funding needs of an account's pending orders. A narrower fix that removes the
zero-cost property but not the option is to forfeit the bounty when an account executes its own order into a
user-invalid failure.



## Proof of Concept

## Proof of Concept

## PoC

`packages/perps/test/poc/Z_VpiFreeOption.t.sol` extends the repository's own `BasePerpTest` with the launch risk
parameters; nothing is stubbed beyond the harness's existing `MockPyth`.

```bash
cd packages/perps
forge test --match-path test/poc/Z_VpiFreeOption.t.sol -vv
```

```
[PASS] test_Z01_Control_SkewReducingOpenExecutes()
[PASS] test_Z01_WithdrawalAfterCommitCancelsSkewReducingOpenForFree()
[PASS] test_Z01_OneCommitTwoBranchesIsALookBackOption()
[PASS] test_Z02_FiveMillionNotionalCancelCostsSeventeenThousandOfMargin()
[PASS] test_Z03_OptionPayoffComesOutOfPoolAssets()
5 passed; 0 failed; 0 skipped
```

```
A: trader gain (6dp)               10906422288
A: pool loss (6dp)                 14912622288
A: protocol fees collected (6dp)    4006000000
B: trader loss if forced to fill    14094200000
B: pool gain it never collects      10100000000
option value per strike (6dp)      25000622288
```

- **Control** — left alone, the committed open executes and the rebate is reserved, so the setup is a legitimate
  order rather than one that was always going to fail.
- **Attack** — withdrawing $6 after commit makes execution fail with `INSUFFICIENT_INITIAL_MARGIN` (code 6): no
  position opens, the account balance is unchanged, the bounty returns to the trader and the intent is gone.
- **Option** — from one snapshotted committed state, one branch takes the fill and the other walks away at no cost.
- **Scale** — the same two branches on $5,000,000 of notional against a $200M pool for a $19,500 funded account.
- **Impact** — the three branches above, run from one committed state with the market moved 30 bp, measuring the
  trader's balance, `pool.totalAssets()`, the treasury and the keeper at each step.

```solidity
function test_Z01_WithdrawalAfterCommitCancelsSkewReducingOpenForFree() public {
    _buildLongSkew();                                   // another account creates the imbalance
    (uint64 id, uint256 bounty) = _commitSkewReducingOpen();

    vm.prank(ATTACKER);
    clearinghouse.withdraw(ATTACKER, WITHDRAW);         // flat account, no floor: the guard is silent
    uint256 balanceBefore = clearinghouse.balanceAssets(ATTACKER);

    vm.prank(ATTACKER);                                 // self-execution returns the bounty
    OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _mockPythUpdateData(PRICE));
    assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));

    OrderV3Types.OrderReceipt memory receipt = _receipt(id);
    assertEq(receipt.failure.code, uint8(CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN));

    (uint256 size,,,,,,) = engine.positions(ATTACKER);
    assertEq(size, 0, "no position was opened");
    assertEq(clearinghouse.balanceAssets(ATTACKER), balanceBefore, "no value left the account");
    assertEq(receipt.bountyRecipient, ATTACKER, "the bounty came back to the trader");
    assertEq(router.pendingOrderCounts(ATTACKER), 0, "the binding intent is gone");
}
```


