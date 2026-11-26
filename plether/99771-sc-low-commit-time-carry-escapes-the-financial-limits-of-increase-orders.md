# \#99771 [SC-Low] Commit-time carry escapes the financial limits of increase orders

**Submitted on Nov 15th 2026 at 07:42:53 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99771
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Intro

A trader can attach `maxActionChargeAssets` and `maxGrossAccountDebitAssets` to an increase order to cap what it may cost, but the increase can still execute above both limits. Committing the increase collects the carry already accrued on the existing position, yet the open reservation path reports zero collected carry to the order's lifecycle record, so the commit-time check sees zero, and the execution check, which adds commitment carry back only for closes, sees only the new trade's charge. The carry taken at commit therefore never counts against either limit. In the PoC, which uses release parameters, the production router and only Pyth and USDG mocks, Alice's 10,000-token LONG is doubled to 20,000 tokens by an arbitrary keeper although her combined action charge (**23.037959 USDG**) exceeds her `maxActionChargeAssets` (21.018152) and her actual account debit (**23.237959 USDG**) exceeds her `maxGrossAccountDebitAssets` (21.218152), while the lifecycle book records 0 commitment carry.

## Vulnerability Details

The canonical bounds are meant to cap an order's total financial effect. The protocol's integration guide (WORKING_WITH_AI_AGENTS.md L102-L103) defines `maxGrossAccountDebitAssets` as the "maximum aggregate settlement debit, consumed trader claims, and reserved bounty" and `maxActionChargeAssets` as the "maximum assessed action charge after carry, signed virtual price impact (VPI), fees, and frozen spread are combined".

An increase follows the open-order branch of `OrderReservationAccounting._reserveExecutionBounty`, which sets only `bountyFromFreeAssets` and leaves every other commitment field, including `carryCollectedAssets`, at zero:

```solidity
// OrderReservationAccounting.sol L108-L128 (_reserveExecutionBounty)
if (isClose) {
    ...
    return _reserveCloseExecutionBounty(account, sizeDelta);
}
effects.bountyFromFreeAssets = bounty;
if (bounty == 0) {
    return effects;
}
if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
    revert OrderRouter__InsufficientFreeEquity();
}
clearinghouse.lockReservedSettlement(account, bounty);
```

Reserving the bounty is not neutral for an account with an open position. `MarginClearinghouse.lockReservedSettlement` checkpoints carry first, which collects the carry accrued on the existing position from the trader's settlement balance:

```solidity
// MarginClearinghouse.sol L818-L824
function lockReservedSettlement(
    address account,
    uint256 amountAssets
) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

`OrderCommitHandler._createPendingOrder` (L21-L35) then passes those incomplete effects to `_onOrderCommitted`, which forwards them to `OrderLifecycleBook.recordCommitment`. The book checks the commitment against the order's limits using the reported carry, which is zero for an increase, and stores that zero:

```solidity
// OrderLifecycleBook.sol L335-L347 (recordCommitment)
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.GrossAccountDebit, gross, pending.bounds.maxGrossAccountDebitAssets
    );
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.ActionCharge,
        effects.carryCollectedAssets,
        pending.bounds.maxActionChargeAssets
    );
}
```

At execution, `CfdOrderPolicyEvaluator.assessCommittedOrder` assesses the order against the account state after the commit, in which the existing position's carry has already been collected, and it adds the recorded commitment back only for closes:

```solidity
// CfdOrderPolicyEvaluator.sol L64-L67 (assessCommittedOrder)
assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
if (order.isClose) {
    _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
}
```

The limits are therefore checked twice and pass both times: once at commit with zero carry, and once at execution with only the new trade's VPI, fees and bounty. A close does not have this gap, because its reservation path returns the collected carry in its effects and the evaluator includes it. `MarginClearinghouse.reserveCommittedOrderMargin` (L620-L633), which runs after the effects are recorded, also checkpoints carry, so it is outside the bounds for the same reason.

The concrete path runs as follows. Alice opens a 10,000-token LONG with 1,000 USDG in her clearinghouse account. Thirty days later she commits a 10,000-token increase through `commitOrder`, with `maxActionChargeAssets` and `maxGrossAccountDebitAssets` set between what the increase alone costs and what it costs including the carry taken at commit. The commit collects 4.039614 USDG of carry from her settlement balance, and the lifecycle book records 0. Bob, an arbitrary keeper, executes the order, and her position doubles to 20,000 tokens.

## Impact Details

The contract promises that an order whose total financial effect exceeds the trader's own limits will not execute, and it does not deliver that for increases. Alice's increase executes although its combined action charge of 23.037959 USDG exceeds her 21.018152 USDG limit and her actual account debit of 23.237959 USDG exceeds her 21.218152 USDG limit, each by 2.019807 USDG. With the commit carry counted, the bounded order would have failed instead of charging the new trade cost and doubling her exposure. The carry itself was already owed on her original position, so no value is lost; the harm is that the protocol executes an additional trade the trader capped, and that the lifecycle book understates the order's debit to any client that reads its commitment record.

## Recommendation

Report the carry collected by `lockReservedSettlement` (and by the later committed-margin reservation) in the open path's commitment effects, as the close path already does, and include the recorded commitment carry in the execution assessment for opens and increases as well as for closes, so that both limits always see the order's full action charge and account debit.

## References

The open reservation path leaves commitment carry at zero: [OrderReservationAccounting.sol#L108-L128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol?plain=1#L108-L128)

Reserving the bounty checkpoints and collects carry: [MarginClearinghouse.sol#L818-L824](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol?plain=1#L818-L824)

The lifecycle book checks and stores the reported carry: [OrderLifecycleBook.sol#L327-L360](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol?plain=1#L327-L360)

The execution assessment adds commitment carry only for closes: [CfdOrderPolicyEvaluator.sol#L23-L67](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol?plain=1#L23-L67)

## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every order goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed through the permissionless `router.executeOrder` with a signed basket published one second after the commit. The LP deposit uses the public request → `router.settleLpEpoch` → claim flow, and mark refreshes go through the permissionless `router.updateMarkPrice`. No protocol contract is impersonated. The test applies the release parameters from `DEPLOYMENT.md`.

### Step-by-step (`test_IncreaseExecutesAboveActionChargeAndGrossDebitLimits`)

**Setup.** An LP deposits 100,000 USDG of Junior. On Monday, Alice deposits 1,000 USDG and opens a 10,000-token LONG at 1.00. The clock moves 30 days forward to a Wednesday in regular trading hours.

**Measurement.** On a snapshot, an unbounded copy of the increase is committed and assessed through the production policy evaluator. The commit collects 4.039614 USDG of carry, and the execution assessment for the increase alone is an action charge of 18.998345 USDG and a gross debit of 19.198345 USDG. The snapshot is restored.

**Bounded commit.** Alice commits the 10,000-token increase with `maxActionChargeAssets` = 21.018152 USDG and `maxGrossAccountDebitAssets` = 21.218152 USDG, halfway between the execution-only amounts and the combined amounts. The commit takes 4.039614 USDG of carry from her settlement balance, and `pendingIntent(orderId).commitment.carryCollectedAssets` in the lifecycle book is 0.

**Execution.** Bob executes the order as an ordinary keeper. It executes, and Alice's position is 20,000 tokens. Her combined action charge is 23.037959 USDG and her clearinghouse balance fell by 23.237959 USDG from before the commit to after execution, so each limit is exceeded by 2.019807 USDG.

### Other scenarios covered by the test

`test_Control_LimitBelowExecutionChargeFails` commits the same increase with `maxActionChargeAssets` one atom below the execution-only charge. The order fails and the position stays at 10,000 tokens, which shows that the limit is enforced, but only against the execution part of the charge.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/IncreaseCommitCarryBounds.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/IncreaseCommitCarryBounds.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 2 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title An increase executes above the trader's action-charge and gross-debit limits
/// @notice Committing an increase reserves its execution bounty through `lockReservedSettlement`, which checkpoints
///         and collects the carry accrued on the existing position. The open reservation path never reports that carry
///         in its commitment effects, so the lifecycle book checks and stores zero commitment carry, and the execution
///         assessment adds commitment carry back only for closes. The order's `maxActionChargeAssets` and
///         `maxGrossAccountDebitAssets` are therefore checked without the carry already taken at commit.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Production V3 `commitOrder` and permissionless
///      `executeOrder` / `updateMarkPrice` / `settleLpEpoch`. Pyth and USDG are the only mocks. No contract under src/
///      is modified or impersonated.
contract IncreaseCommitCarryBoundsTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // trader with an open LONG
    address internal constant BOB = address(0xB0B); // any keeper
    address internal constant KEEPER = address(0xBEEF);
    address internal constant LP = address(0x1B);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SIZE = 10_000e18; // 10,000 tokens

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        _publicJuniorDeposit(LP, 100_000e6);
        _fundTrader(ALICE, 1000e6);

        // Alice opens a 10,000-token LONG.
        _goto(vm.getBlockTimestamp() + 1);
        _markNow();
        _commitAndExecute(_request(0, 0), KEEPER);
        assertEq(_size(), SIZE, "LONG opened");

        // 30 days later (a Wednesday, regular trading hours) carry has accrued on the position.
        _goto(vm.getBlockTimestamp() + 30 days);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertFalse(engine.isFadWindow(), "not FAD");
        _markNow();
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers (public entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(PRICE)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _markNow() internal {
        bytes[] memory data = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _goto(juniorVault.depositEpochStart(requestId));
        bytes[] memory data = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    /// @dev A 10,000-token LONG open or increase with 100 USDG of margin. Zero limits mean unlimited.
    function _request(
        uint256 maxActionCharge,
        uint256 maxGrossDebit
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(ALICE, router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: 100e6,
            targetPrice: 1,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: maxGrossDebit == 0 ? type(uint256).max : maxGrossDebit,
                maxActionChargeAssets: maxActionCharge == 0 ? type(uint256).max : maxActionCharge,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commit(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId, uint256 commitTime) {
        commitTime = vm.getBlockTimestamp();
        vm.prank(ALICE);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    function _execute(
        uint64 orderId,
        uint256 commitTime,
        address executor
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory data = _payload(commitTime + 1);
        vm.prank(executor);
        r = router.executeOrder(orderId, data);
    }

    function _commitAndExecute(
        OrderV3Types.OrderRequest memory request,
        address executor
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        (uint64 id, uint256 c) = _commit(request);
        _goto(c + 1);
        r = _execute(id, c, executor);
    }

    function _size() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(ALICE);
    }

    function _balance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ALICE);
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        string memory frac = vm.toString(v % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", vm.toString(v / 1e6), ".", frac));
    }

    struct Measured {
        uint256 carryAtCommit; // Alice's settlement balance drop caused by the commit itself
        uint256 actionChargeExecution; // execution assessment, increase alone
        uint256 grossDebitExecution;
    }

    /// @dev Commits an unbounded increase on a snapshot and measures the commit-time carry and the execution
    ///      assessment the Router will apply, then restores the snapshot.
    function _measure() internal returns (Measured memory m) {
        uint256 snap = vm.snapshotState();
        uint256 before = _balance();
        (uint64 id,) = _commit(_request(0, 0));
        m.carryAtCommit = before - _balance();
        _goto(vm.getBlockTimestamp() + 1);
        OrderV3Types.ExecutionAssessment memory a =
            ICfdOrderPolicyEvaluator(router.policyEvaluator()).assessCommittedOrder(address(engine), id, BOB, PRICE);
        m.actionChargeExecution = a.actionChargeAssessedAssets;
        m.grossDebitExecution = a.grossAccountDebitAssets;
        vm.revertToState(snap);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Alice sets both limits between the execution-only amount and the combined amount. The commit collects
    ///         carry, the book records zero, and Bob executes the increase although both limits are exceeded.
    function test_IncreaseExecutesAboveActionChargeAndGrossDebitLimits() public {
        Measured memory m = _measure();
        uint256 maxActionCharge = m.actionChargeExecution + m.carryAtCommit / 2;
        uint256 maxGrossDebit = m.grossDebitExecution + m.carryAtCommit / 2;

        uint256 balanceBefore = _balance();
        (uint64 id, uint256 c) = _commit(_request(maxActionCharge, maxGrossDebit));
        uint256 carryCollected = balanceBefore - _balance();
        uint256 carryRecorded = router.lifecycleBook().pendingIntent(id).commitment.carryCollectedAssets;

        _goto(c + 1);
        OrderV3Types.ExecutionResult memory r = _execute(id, c, BOB);
        uint256 totalDebit = balanceBefore - _balance();
        uint256 combinedActionCharge = carryCollected + m.actionChargeExecution;

        _usd("carry collected from Alice at commitment", carryCollected);
        _usd("carry recorded in the lifecycle book", carryRecorded);
        _usd("action charge assessed for execution alone", m.actionChargeExecution);
        _usd("Alice's maxActionChargeAssets", maxActionCharge);
        _usd("COMBINED ACTION CHARGE", combinedActionCharge);
        _usd("gross debit assessed for execution alone", m.grossDebitExecution);
        _usd("Alice's maxGrossAccountDebitAssets", maxGrossDebit);
        _usd("ACTUAL CLEARINGHOUSE DEBIT, commit through execution", totalDebit);
        console2.log("position size after execution (tokens)", _size() / 1e18);

        assertGt(carryCollected, 0, "commit collected carry");
        assertEq(carryRecorded, 0, "lifecycle book records zero commitment carry");
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "increase executed");
        assertEq(_size(), 2 * SIZE, "position doubled");
        assertGt(combinedActionCharge, maxActionCharge, "combined action charge exceeds the limit");
        assertGt(totalDebit, maxGrossDebit, "actual account debit exceeds the limit");
    }

    /// @notice Control: a limit just below the execution-only charge makes the same increase fail, so the limits are
    ///         enforced, but only against the execution part of the charge.
    function test_Control_LimitBelowExecutionChargeFails() public {
        Measured memory m = _measure();
        (uint64 id, uint256 c) = _commit(_request(m.actionChargeExecution - 1, 0));
        _goto(c + 1);
        OrderV3Types.ExecutionResult memory r = _execute(id, c, BOB);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "rejected by the action-charge limit");
        assertEq(_size(), SIZE, "position unchanged");
    }

}
```

