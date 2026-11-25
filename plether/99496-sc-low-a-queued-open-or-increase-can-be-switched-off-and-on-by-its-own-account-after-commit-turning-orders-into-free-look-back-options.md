# \#99496 [SC-Low] A queued open or increase can be switched off and on by its own account after commit, turning orders into free look-back options

**Submitted on Nov 13th 2026 at 21:55:39 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99496
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
Plether's delayed-order model relies on orders being binding: SECURITY.md states that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". This does not hold, because two execution-time checks read account state that the order's owner can still change after commit. A skew-reducing open funds part of its VPI rebate reserve from the account's pre-existing free settlement, which the owner can withdraw and deposit again, and an increase's own `minPostPositionEquityAssets` and `maxPostLeverageBps` bounds read the existing position pledge, which the owner can raise at any time with `addMargin`. Either lever lets the owner decide after the fill tick whether its own order succeeds. A failed order is terminal, its committed margin is released, and a self-executed failure also returns the execution bounty, so dropping the order costs nothing. In the PoC, which uses release parameters, the production router and only Pyth and USDG mocks, a 1,000,000 USDG SHORT open that the market moved 20 bps against is cancelled at a cost of exactly 0 USDG, while the same order kept after a favourable 20 bps move nets the attacker **+1,192.97 USDG** and costs LPs **1,993.77 USDG**.


## Vulnerability Details

Plether executes orders in two steps. A trader commits an order and a keeper executes it later at the first Pyth tick after the commit time. The fill price is therefore fixed within a second of commit, but the order may be executed at any time inside its execution window, 60 seconds at release, and it still fills at that first tick even after newer marks have been installed. What stops this from being an option is that the order is binding: users cannot cancel, and any keeper can fill it. SECURITY.md states this property directly in the delayed-order model (L658-L659). The order is not binding, because two execution-time checks read account state that the owner can still change after commit.

The first check is the VPI rebate reserve. An open that reduces skew receives a negative VPI. `MarginClearinghouseAccountingLib.planOpenCostApplication` (L148-L151) credits the rebate net of the execution fee to settlement, and `CfdEnginePlanLib._planOpenReserveFunding` must then lock a reserve equal to the full rebate. It takes the reserve from free settlement first and from the new position pledge for the remainder:

```solidity
// CfdEnginePlanLib.sol L564-L567 (_planOpenReserveFunding)
uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
```

A fee-sized slice of the reserve is thus funded from the trader's pre-existing free settlement, which is not part of the order and is not reserved at commit. For a flat account, `CfdEngineSettlementSidecar.validateWithdraw` (L261-L272) lets that balance be withdrawn unless the trader opted into a settlement floor. With the balance gone, the slice comes out of the pledge. An order at minimum margin then fails with `INSUFFICIENT_INITIAL_MARGIN`, and an order at any margin level fails its own `minPostPositionEquityAssets` bound if the trader placed that bound between the two outcomes. Depositing the amount again makes the order valid again.

The second check needs no rebate. The policy evaluator checks the order's `minPostPositionEquityAssets` and `maxPostLeverageBps` against post-execution position equity, which includes the pledge of the existing position:

```solidity
// CfdOrderPolicyEvaluatorBase.sol L530-L539
uint256 normalizedEquityAssets =
    assessment.postPositionEquityAssets > 0 ? uint256(assessment.postPositionEquityAssets) : 0;
if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
    revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
        OrderV3Types.ConstraintKind.PostPositionEquity,
        normalizedEquityAssets,
        bounds.minPostPositionEquityAssets
    );
}
_requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

Commit does not require these bounds to hold, so an increase can be committed with a bound that currently fails. `CfdEngine.addMargin` (L501) raises the pledge at any time, so the owner can make the order valid whenever it wants. An account holding a minimum-size position can use this for an increase of any size, and with two accounts both directions are available.

Failing costs nothing. Both failures are terminal: the order is finalized as `Failed`, with reason `PlannerRejected` or `ConstraintViolation`, and its committed margin is released. `OrderRouter._settleTerminalBounty` pays the execution bounty to the executor, and when the executor is the order's own account it releases the bounty back:

```solidity
// OrderRouter.sol L333-L343 (_settleTerminalBounty)
if (bountyAssets != 0) {
    settlement.bountyRecipient = bountyRecipient;
    settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
    if (bountyRecipient == order.account) {
        clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
    } else {
        engine.creditBounty(
            order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime
        );
    }
}
```

If a keeper executes a dead order first, the trader loses only the bounty, which is capped at 0.20 USDG. The protocol already protects one bound of this kind: a declared settlement floor blocks withdrawals while the order is pending in `validateWithdraw`. The rebate reserve and the equity and leverage bounds have no such protection. An ordinary fresh open that pays VPI has no lever at all, since withdrawing every free atom does not change its outcome.

This is the class of pre-audit finding SC-M4, "Risk Free Option in 60Sec Oracle Window", whose third component was a trader-induced execution failure that returned the margin of the losing order. SC-M4 is marked fixed. The cancellation path it used is gone, but the two paths above do the same job.

The concrete attack runs as follows. The pool holds 10,000,000 USDG and the book is 2,000,000 USDG long, so a SHORT open reduces skew and earns a rebate. The attacker commits a 1,000,000 USDG SHORT open at the minimum margin the commit admits and leaves the settlement floor at zero. The fill price is the first tick after commit. Nobody executes for five seconds while the attacker watches the market. If the basket has moved 20 bps in the attacker's favour, it executes its own order with the old tick and closes at the market, ending 1,192.97 USDG ahead while LPs end 1,993.77 USDG behind. If the basket has moved 20 bps against it, the attacker withdraws its free settlement and executes its own order, which ends as `Failed` with `PlannerRejected`. The attacker's balance is unchanged to the atom and LPs receive nothing, whereas a binding order filled by a keeper at the tick would have cost the attacker 2,799.40 USDG and given LPs 2,000.00 USDG.

The order can also be dead by default. The attacker withdraws right after commit, so a keeper that executes the order can only terminalize it, and the attacker loses the 0.20 USDG bounty. If the market moves 20 bps in its favour first, the attacker deposits the amount again, fills at the old tick and ends 1,192.97 USDG ahead after closing. The increase path works the same way without any rebate: an account holding a 1,000 USDG LONG on the heavy side commits a 1,000,000 USDG LONG increase with an equity bound 50 USDG above what the increase would produce, and after a favourable 20 bps move it calls `addMargin` with 100 USDG, fills at the old tick and closes 1,193.78 USDG ahead.


## Impact Details
The protocol promises that queued orders are binding and cannot be cancelled by their owner, and it relies on that promise to rule out free options in its delayed-order model. The contracts do not deliver it. An account can decide after the fill tick whether its own open or increase succeeds, and dropping the order costs nothing, or at most the 0.20 USDG bounty when a keeper terminalizes a dead order first. This also contradicts the listed statement that users cannot manually cancel queued intents.

The value of the option depends on how long the order stays unexecuted and how far the price moves in that time. A prompt keeper ends the option within a second or two, because it fills a live order at the tick or terminalizes a dead one. A stand-alone open and close also pays about 8 bps in fees, so the move must exceed that before the attacker profits, although a trader who wants the position anyway gains from any favourable move. In ordinary FX conditions a basket move of several basis points within a few seconds is rare, but it does happen at scheduled releases, and the dead-by-default form costs at most 0.20 USDG per attempt, so it can be repeated at every release. When the option is exercised, LPs are the counterparty: in the PoC a 20 bps favourable move costs LPs 1,993.77 USDG on a 1,000,000 USDG order, while the opposite move, which a binding order would have turned into a 2,000.00 USDG LP gain, brings them nothing.

## Recommendation

Make an order's success independent of account state that the owner can change after commit. Fund the fee-sized part of the VPI rebate reserve from the order's own margin, or reserve at commit the free settlement the planned open will draw, as the settlement floor already does, and either reject `addMargin` while the account has a pending increase or evaluate the equity and leverage bounds against the pledge recorded at commit plus the order's own margin. As defence in depth, do not return the bounty to a self-executing account when its open or increase fails for a reason the account could have caused.

## References

SECURITY.md states that binding orders prevent free options: [SECURITY.md#L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L658-L659)

The rebate reserve is funded from pre-existing free settlement first: [CfdEnginePlanLib.sol#L546-L580](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L546-L580)

The rebate is credited net of the execution fee: [MarginClearinghouseAccountingLib.sol#L148-L151](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol?plain=1#L148-L151)

A flat account can withdraw unless it declared a settlement floor: [CfdEngineSettlementSidecar.sol#L261-L272](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol?plain=1#L261-L272)

The equity and leverage bounds read post-execution position equity: [CfdOrderPolicyEvaluatorBase.sol#L530-L539](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol?plain=1#L530-L539)

The owner can raise the pledge at any time: [CfdEngine.sol#L501](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol?plain=1#L501)

A self-executor gets the bounty back on failure: [OrderRouter.sol#L333-L343](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol?plain=1#L333-L343)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. `BasePerpTest` deploys the repository's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` unchanged and only adds a test-only scalar commit overload that this test never calls. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and `router.executeOrder`, with explicit Pyth ticks published after the commit. Orders are executed either by an arbitrary keeper address or by the order's own account. The LP deposit uses the public request → `router.settleLpEpoch` → claim flow, mark refreshes go through the permissionless `router.updateMarkPrice`, and no protocol contract is impersonated.

The test applies the release parameters from `DEPLOYMENT.md`: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4, 4 bps execution fee, 20/10 bps margins, 10 bps maximum confidence, 2,500 bps adverse multiplier, 60-second execution window, Senior cap 40M/80% and 1 USDG seeds. The pool holds 10,000,000 USDG of Junior, and an honest trader holds a 2,000,000 USDG LONG, so SHORT opens reduce skew and LONG increases pay VPI.

### Step-by-step (`test_RebateOpen_FillWhenFavourable_CancelForFreeWhenNot`)

**Commit.** The attacker commits a 1,000,000 USDG SHORT open at the smallest margin the production commit admits, with no settlement floor. The order's fill price is the first tick after commit, at 1.00.

**Favourable move.** Nobody executes for five seconds, and a newer mark 20 bps in the attacker's favour is installed. The attacker executes its own order with the old tick and closes at the new price. It ends +1,192.97 USDG, and LPs end −1,993.77 USDG. With a 10 bps move the attacker ends +196.37 USDG, and with a 20 bps move the same fill still works 55 seconds after the tick.

**Adverse move.** From the same snapshot, the market moves 20 bps against the attacker. The attacker withdraws its free settlement and executes its own order. The order ends as `Failed` with `PlannerRejected`, the attacker's balance change is exactly 0 and the LPs' change is exactly 0.

**Binding control.** From the same snapshot, a keeper fills the order at the tick, as a binding order would be filled. After the same adverse 20 bps move the attacker ends −2,799.40 USDG and LPs gain 2,000.00 USDG. With no move at all, the round trip costs the attacker 800.43 USDG in fees.

### Other scenarios covered by the test

`test_RebateOpen_DeadByDefault_RevivedByDeposit` withdraws the free settlement right after commit. A keeper that executes the dead order gets a terminal failure, and the attacker loses only the 0.20 USDG bounty. If the market moves 20 bps in the attacker's favour first, the attacker deposits the amount again, fills at the old tick and ends +1,192.97 USDG after closing.

`test_RebateOpen_AnyMargin_EquityBoundMakesWithdrawalASwitch` removes the minimum-margin condition. With 5,000 USDG of margin above the minimum, the withdrawal changes post-open equity from 7,000.00 to 6,600.00 USDG. An order carrying an equity bound between those two values fills when the free settlement is in place, and fails with `ConstraintViolation` at no cost after the withdrawal.

`test_IncreasePayingVpi_DeadByOwnBound_RevivedByAddMargin` removes the rebate. The attacker holds a 1,000 USDG LONG and commits a 1,000,000 USDG LONG increase, which pays VPI, with an equity bound 50 USDG above what the increase would produce, and commit accepts it. A keeper that executes it gets `ConstraintViolation` and the attacker loses 0.20 USDG. If the market moves against the LONG, the attacker executes its own dead order and loses exactly 0. If the basket moves 20 bps in its favour, it calls `addMargin` with 100 USDG, fills at the old tick and closes +1,193.78 USDG, with LPs −1,593.78 USDG.

`test_Control_FreshOpenPayingVpi_UnaffectedByWithdrawal` is the control: a fresh 1,000,000 USDG LONG open that pays VPI still fills after the account withdraws all of its free settlement, so only the two paths above give the owner a switch.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/QueuedOrderFreeOption.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/QueuedOrderFreeOption.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 5 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A queued open or increase can be switched off and on by its own account after commit
/// @notice An order fills at the first Pyth tick after its commit, whenever it is executed inside its window. Two
///         execution-time checks read account state that the owner can still change after commit:
///           (1) a skew-reducing (rebate-earning) open needs a fee-sized amount of pre-existing free settlement to
///               fund its VPI rebate reserve, which the owner can withdraw and deposit again;
///           (2) the order's own `minPostPositionEquityAssets` bound reads the existing position pledge, which the
///               owner can raise with `addMargin`.
///         A terminal failure returns the committed margin, and a self-executed failure also returns the bounty.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Every trade goes through the production V3
///      `IPerpsTraderActions.commitOrder(OrderRequest)` and `router.executeOrder` with explicit Pyth ticks. The LP
///      deposit uses the public request -> `router.settleLpEpoch` -> claim flow. Pyth and USDG are the only mocks.
///      No contract under src/ is modified or impersonated.
///      Pool: 10M Junior. Book: OTHER is LONG 2M, so SHORT opens reduce skew (rebate) and LONG increases pay VPI.
contract QueuedOrderFreeOptionTest is BasePerpTest {

    address internal constant ATT = address(0xA77AC); // attacker trading account
    address internal constant OTHER = address(0x07E12); // honest trader holding the heavy LONG side
    address internal constant KEEPER = address(0xBEEF); // untrusted keeper
    address internal constant LP = address(0x1B); // Junior LP

    uint256 internal constant PRICE = 1e8; // basket 1.00
    uint256 internal constant SIZE = 1_000_000e18; // 1,000,000 synthetic tokens = 1M USDG notional at 1.00
    uint256 internal constant TUE_NOV10_1000 = 1_794_304_800; // Tue 2026-11-10 10:00:00 UTC (regular trading)
    uint8 internal constant EXECUTED = uint8(OrderV3Types.LifecycleStatus.Executed);
    uint8 internal constant FAILED = uint8(OrderV3Types.LifecycleStatus.Failed);

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
        _publicJuniorDeposit(LP, 10_000_000e6);

        // Tuesday 10:00 UTC, far from any FAD / frozen window.
        _goto(TUE_NOV10_1000);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertFalse(engine.isFadWindow(), "not FAD");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence cap");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release execution window");
        _markNow(PRICE);

        // OTHER holds the heavy LONG side: 2M notional.
        _fundTrader(OTHER, 400_000e6);
        _fundTrader(ATT, 100_000e6);
        _openAt(OTHER, CfdTypes.Side.LONG, 2_000_000e18, 200_000e6, PRICE);
        _goto(vm.getBlockTimestamp() + 1);
        _markNow(PRICE);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Oracle, LP and trader helpers (production entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Pyth payload: every basket feed at `price`, published at `publishTime` (previous publish = publishTime - 1).
    function _tick(
        uint256 price,
        uint256 conf,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(
                    SafeCast.toInt64(SafeCast.toInt256(price)),
                    SafeCast.toUint64(conf),
                    -8,
                    publishTime,
                    publishTime - 1
                )
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Permissionless mark refresh with a tick published now.
    function _markNow(
        uint256 price
    ) internal {
        vm.prank(KEEPER);
        router.updateMarkPrice(_tick(price, 0, vm.getBlockTimestamp()));
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
        vm.prank(KEEPER);
        router.settleLpEpoch(_tick(PRICE, 0, vm.getBlockTimestamp()));
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _request(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        // LONG profits when the basket falls: an open needs price >= target, a close price <= target (SHORT mirrored).
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
            })
        });
    }

    function _commitNow(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(account, side, size, margin, isClose);
        vm.prank(account);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    function _tryCommit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (bool ok) {
        OrderV3Types.OrderRequest memory request = _request(account, side, size, margin, false);
        vm.prank(account);
        try IPerpsTraderActions(address(router)).commitOrder(request) {
            ok = true;
        } catch {}
    }

    function _execWith(
        address executor,
        uint64 orderId,
        bytes[] memory data
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.prank(executor);
        result = router.executeOrder(orderId, data);
    }

    /// @dev Commit, then a keeper executes one second later with a tick published after the commit.
    function _openAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        uint64 id = _commitNow(account, side, size, margin, false);
        uint256 commitTime = vm.getBlockTimestamp();
        _goto(commitTime + 1);
        OrderV3Types.ExecutionResult memory r = _execWith(KEEPER, id, _tick(price, 0, commitTime + 1));
        assertEq(uint8(r.status), EXECUTED, "open executed");
    }

    function _closeAll(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        uint64 id = _commitNow(account, side, _size(account), 0, true);
        uint256 c = vm.getBlockTimestamp();
        _goto(c + 1);
        OrderV3Types.ExecutionResult memory r = _execWith(KEEPER, id, _tick(price, 0, c + 1));
        assertEq(uint8(r.status), EXECUTED, "close executed");
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _free(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.getPnlIsolationBuckets(account).freeSettlementAssets;
    }

    function _withdrawAllFree(
        address account
    ) internal returns (uint256 amount) {
        amount = _free(account);
        vm.prank(account);
        clearinghouse.withdraw(account, amount);
    }

    /// @dev Clearinghouse balance + wallet USDG + trader claim.
    function _equity(
        address account
    ) internal view returns (int256) {
        return int256(
            clearinghouse.balanceAssets(account) + settlementToken.balanceOf(account)
                + engine.traderClaimBalanceAssets(account)
        );
    }

    /// @dev LP value: pool USDG net of trader claims.
    function _lpValue() internal view returns (int256) {
        return int256(settlementToken.balanceOf(address(pool))) - int256(engine.totalTraderClaimBalanceAssets());
    }

    function _priceAfter(
        int256 moveBps
    ) internal pure returns (uint256) {
        return uint256(int256(PRICE) + int256(PRICE) * moveBps / 10_000);
    }

    /// @dev Smallest marginDelta the production commit admits for this open at the current fresh mark.
    function _minAdmittedMargin(
        CfdTypes.Side side,
        uint256 size
    ) internal returns (uint256 lo) {
        uint256 hi = 50_000e6;
        lo = 1;
        while (lo < hi) {
            uint256 mid = (lo + hi) / 2;
            uint256 snap = vm.snapshotState();
            bool ok = _tryCommit(ATT, side, size, mid);
            vm.revertToState(snap);
            if (ok) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
    }

    function _usd(
        string memory label,
        int256 v
    ) internal pure {
        uint256 abs = v < 0 ? uint256(-v) : uint256(v);
        string memory frac = vm.toString(abs % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", v < 0 ? "-" : "", vm.toString(abs / 1e6), ".", frac));
    }

    struct Run {
        int256 attacker;
        int256 lp;
        uint8 status;
        uint8 reason;
    }

    // ------------------------------------------------------------------------------------------------------------
    // 1. Rebate-earning fresh open: fill at the old tick if the market moved in favour, cancel for free if not
    // ------------------------------------------------------------------------------------------------------------

    /// @dev `keeperPrompt`: an honest keeper executes at the first tick (the order behaves as binding).
    ///      Otherwise nobody executes for `wait` seconds, the market moves by `moveBps`, and the account either
    ///      withdraws its free settlement (`cancel`) or not, then executes its own order with the old tick.
    function _rebateOpenLife(
        int256 moveBps,
        uint256 wait,
        bool cancel,
        bool keeperPrompt
    ) internal returns (Run memory r) {
        int256 a0 = _equity(ATT);
        int256 lp0 = _lpValue();
        uint256 margin = _minAdmittedMargin(CfdTypes.Side.SHORT, SIZE);
        uint64 id = _commitNow(ATT, CfdTypes.Side.SHORT, SIZE, margin, false);
        uint256 t0 = vm.getBlockTimestamp();
        uint256 p1 = _priceAfter(moveBps);

        OrderV3Types.ExecutionResult memory res;
        if (keeperPrompt) {
            _goto(t0 + 1);
            res = _execWith(KEEPER, id, _tick(PRICE, 0, t0 + 1));
            _goto(t0 + 1 + wait);
            _markNow(p1);
        } else {
            _goto(t0 + 1 + wait);
            _markNow(p1); // a newer mark is installed; the order still fills at its own first tick
            if (cancel) {
                _withdrawAllFree(ATT);
            }
            res = _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        }
        r.status = uint8(res.status);
        r.reason = uint8(res.terminalReason);
        if (_size(ATT) != 0) {
            _closeAll(ATT, CfdTypes.Side.SHORT, p1);
        }
        r.attacker = _equity(ATT) - a0;
        r.lp = _lpValue() - lp0;
    }

    function test_RebateOpen_FillWhenFavourable_CancelForFreeWhenNot() public {
        uint256 snap = vm.snapshotState();
        Run memory fav20 = _rebateOpenLife(20, 5, false, false);
        vm.revertToState(snap);
        Run memory fav10 = _rebateOpenLife(10, 5, false, false);
        vm.revertToState(snap);
        Run memory late = _rebateOpenLife(20, 55, false, false);
        vm.revertToState(snap);
        Run memory cancelled = _rebateOpenLife(-20, 5, true, false);
        vm.revertToState(snap);
        Run memory binding = _rebateOpenLife(-20, 5, false, true);
        vm.revertToState(snap);
        Run memory flat = _rebateOpenLife(0, 5, false, true);

        _usd("+20 bps after the tick, self-fill 5 s later, close: attacker", fav20.attacker);
        _usd("+20 bps after the tick, self-fill 5 s later, close: LPs", fav20.lp);
        _usd("+10 bps after the tick, self-fill 5 s later, close: attacker", fav10.attacker);
        _usd("+20 bps, self-fill 55 s after the tick, close: attacker", late.attacker);
        _usd("-20 bps after the tick, withdraw and self-execute: attacker", cancelled.attacker);
        _usd("-20 bps after the tick, withdraw and self-execute: LPs", cancelled.lp);
        _usd("control, binding order, -20 bps, close: attacker", binding.attacker);
        _usd("control, binding order, -20 bps, close: LPs", binding.lp);
        _usd("control, binding order, no move, close: attacker", flat.attacker);

        assertEq(fav20.status, EXECUTED, "fills at the old tick");
        assertGt(fav20.attacker, 1000e6, "favourable branch is kept");
        assertEq(late.status, EXECUTED, "still fills 55 s later");
        assertEq(cancelled.status, FAILED, "terminal failure");
        assertEq(cancelled.reason, uint8(OrderV3Types.TerminalReason.PlannerRejected), "planner rejected");
        assertEq(cancelled.attacker, 0, "cancel costs nothing");
        assertEq(cancelled.lp, 0, "LPs receive nothing for the option");
        assertLt(binding.attacker, -2000e6, "a binding order would have lost");
    }

    /// @notice The order can also be dead by default: withdraw right after commit. A keeper that executes it only
    ///         takes the bounty; the account deposits again and fills at the old tick only if the market moved in
    ///         its favour.
    function test_RebateOpen_DeadByDefault_RevivedByDeposit() public {
        uint256 margin = _minAdmittedMargin(CfdTypes.Side.SHORT, SIZE);
        int256 a0 = _equity(ATT);
        uint64 id = _commitNow(ATT, CfdTypes.Side.SHORT, SIZE, margin, false);
        uint256 t0 = vm.getBlockTimestamp();
        uint256 taken = _withdrawAllFree(ATT);
        uint256 snap = vm.snapshotState();

        _goto(t0 + 1);
        OrderV3Types.ExecutionResult memory res = _execWith(KEEPER, id, _tick(PRICE, 0, t0 + 1));
        assertEq(uint8(res.status), FAILED, "keeper can only terminalize the dead order");
        assertEq(_size(ATT), 0, "no fill");
        _usd("dead order executed by a keeper: attacker (bounty only)", _equity(ATT) - a0);
        assertEq(_equity(ATT) - a0, -200_000, "0.20 USDG bounty");

        vm.revertToState(snap);
        uint256 p1 = _priceAfter(20);
        _goto(t0 + 6);
        _markNow(p1);
        vm.startPrank(ATT);
        settlementToken.approve(address(clearinghouse), taken);
        clearinghouse.deposit(ATT, taken);
        vm.stopPrank();
        res = _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        assertEq(uint8(res.status), EXECUTED, "revived");
        assertEq(_size(ATT), SIZE, "filled at the old tick");
        _closeAll(ATT, CfdTypes.Side.SHORT, p1);
        _usd("dead order revived after a +20 bps move, then closed: attacker", _equity(ATT) - a0);
    }

    /// @notice The fee-wide margin window is not a real limit. With margin 5,000 USDG above the minimum, the same
    ///         withdrawal lowers the post-open pledge by the fee-sized slice, so an equity bound placed between the two
    ///         outcomes turns the withdrawal into a switch at any margin level.
    function test_RebateOpen_AnyMargin_EquityBoundMakesWithdrawalASwitch() public {
        uint256 margin = _minAdmittedMargin(CfdTypes.Side.SHORT, SIZE) + 5000e6;

        uint256 snap = vm.snapshotState();
        uint64 id = _commitNow(ATT, CfdTypes.Side.SHORT, SIZE, margin, false);
        uint256 t0 = vm.getBlockTimestamp();
        _goto(t0 + 1);
        _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        int256 funded = engineLens.simulateLiquidation(ATT, PRICE, pool.totalAssets()).equityAssets;
        vm.revertToState(snap);
        id = _commitNow(ATT, CfdTypes.Side.SHORT, SIZE, margin, false);
        t0 = vm.getBlockTimestamp();
        _goto(t0 + 1);
        _withdrawAllFree(ATT);
        _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        assertEq(_size(ATT), SIZE, "well-margined open fills even after the withdrawal");
        int256 starved = engineLens.simulateLiquidation(ATT, PRICE, pool.totalAssets()).equityAssets;
        vm.revertToState(snap);
        _usd("post-open equity with free settlement left in place", funded);
        _usd("post-open equity after withdrawing it", starved);
        assertGt(funded, starved, "the withdrawal lowers post-open equity");

        OrderV3Types.OrderRequest memory request = _request(ATT, CfdTypes.Side.SHORT, SIZE, margin, false);
        request.bounds.minPostPositionEquityAssets = uint256((funded + starved) / 2);
        vm.prank(ATT);
        id = IPerpsTraderActions(address(router)).commitOrder(request);
        t0 = vm.getBlockTimestamp();
        _goto(t0 + 1);
        snap = vm.snapshotState();
        OrderV3Types.ExecutionResult memory res = _execWith(KEEPER, id, _tick(PRICE, 0, t0 + 1));
        assertEq(uint8(res.status), EXECUTED, "fills when the free settlement is there");
        vm.revertToState(snap);
        int256 a0 = _equity(ATT);
        _withdrawAllFree(ATT);
        res = _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        assertEq(uint8(res.status), FAILED, "fails after the withdrawal");
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "own bound");
        assertEq(_equity(ATT) - a0, 0, "free cancel");
    }

    // ------------------------------------------------------------------------------------------------------------
    // 2. No rebate needed: an increase that PAYS VPI, dead by its own equity bound, revived with addMargin
    // ------------------------------------------------------------------------------------------------------------

    struct IncreaseSetup {
        uint64 id;
        uint256 t0;
        int256 a0;
        int256 lp0;
    }

    /// @dev ATT holds a 1,000-USDG LONG (heavy side). It commits a 1M LONG increase whose
    ///      `minPostPositionEquityAssets` is 50 USDG above what the increase would produce.
    function _commitDeadIncrease() internal returns (IncreaseSetup memory s) {
        _openAt(ATT, CfdTypes.Side.LONG, 1000e18, 100e6, PRICE);
        _goto(vm.getBlockTimestamp() + 1);
        _markNow(PRICE);
        s.a0 = _equity(ATT);
        s.lp0 = _lpValue();

        uint256 snap = vm.snapshotState();
        uint64 probe = _commitNow(ATT, CfdTypes.Side.LONG, SIZE, 6000e6, false);
        uint256 c = vm.getBlockTimestamp();
        _goto(c + 1);
        _execWith(KEEPER, probe, _tick(PRICE, 0, c + 1));
        assertEq(_size(ATT), SIZE + 1000e18, "the increase fills when unbounded");
        int256 projected = engineLens.simulateLiquidation(ATT, PRICE, pool.totalAssets()).equityAssets;
        vm.revertToState(snap);

        OrderV3Types.OrderRequest memory request = _request(ATT, CfdTypes.Side.LONG, SIZE, 6000e6, false);
        request.bounds.minPostPositionEquityAssets = uint256(projected) + 50e6;
        vm.prank(ATT);
        s.id = IPerpsTraderActions(address(router)).commitOrder(request); // accepted although the bound fails now
        s.t0 = vm.getBlockTimestamp();
    }

    function test_IncreasePayingVpi_DeadByOwnBound_RevivedByAddMargin() public {
        IncreaseSetup memory s = _commitDeadIncrease();
        uint256 snap = vm.snapshotState();

        // (a) a keeper executes it: terminal ConstraintViolation, only the bounty moves
        _goto(s.t0 + 1);
        OrderV3Types.ExecutionResult memory res = _execWith(KEEPER, s.id, _tick(PRICE, 0, s.t0 + 1));
        assertEq(uint8(res.status), FAILED, "dead");
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "own bound");
        assertEq(_size(ATT), 1000e18, "not increased");
        _usd("dead increase executed by a keeper: attacker", _equity(ATT) - s.a0);

        // (b) the market moved against the LONG: the account terminalizes its own dead order, bounty returned
        vm.revertToState(snap);
        _goto(s.t0 + 6);
        _markNow(_priceAfter(20));
        res = _execWith(ATT, s.id, _tick(PRICE, 0, s.t0 + 1));
        assertEq(uint8(res.status), FAILED, "dead");
        assertEq(_size(ATT), 1000e18, "not increased");
        _usd("market against the LONG, own dead order self-executed: attacker", _equity(ATT) - s.a0);
        assertEq(_equity(ATT) - s.a0, 0, "bounty and committed margin returned in full");

        // (c) the market moved in favour of the LONG (basket down 20 bps): add 100 USDG of margin and fill
        vm.revertToState(snap);
        uint256 p1 = _priceAfter(-20);
        _goto(s.t0 + 6);
        _markNow(p1);
        vm.prank(ATT);
        engine.addMargin(ATT, 100e6);
        res = _execWith(ATT, s.id, _tick(PRICE, 0, s.t0 + 1));
        assertEq(uint8(res.status), EXECUTED, "revived");
        assertEq(_size(ATT), SIZE + 1000e18, "increase filled at the old tick");
        _closeAll(ATT, CfdTypes.Side.LONG, p1);
        _usd(
            "market in favour of the LONG (-20 bps), revived with addMargin, then closed: attacker", _equity(ATT) - s.a0
        );
        _usd("market in favour of the LONG (-20 bps), revived with addMargin, then closed: LPs", _lpValue() - s.lp0);
        assertGt(_equity(ATT) - s.a0, 0, "favourable branch is kept");
    }

    // ------------------------------------------------------------------------------------------------------------
    // 3. Control: an ordinary fresh open that pays VPI has no such lever
    // ------------------------------------------------------------------------------------------------------------

    function test_Control_FreshOpenPayingVpi_UnaffectedByWithdrawal() public {
        uint256 margin = _minAdmittedMargin(CfdTypes.Side.LONG, SIZE);
        uint64 id = _commitNow(ATT, CfdTypes.Side.LONG, SIZE, margin, false);
        uint256 t0 = vm.getBlockTimestamp();
        _goto(t0 + 1);
        _withdrawAllFree(ATT);
        assertEq(_free(ATT), 0, "nothing free");
        OrderV3Types.ExecutionResult memory res = _execWith(ATT, id, _tick(PRICE, 0, t0 + 1));
        assertEq(uint8(res.status), EXECUTED, "still fills");
        assertEq(_size(ATT), SIZE, "filled");
    }

}
```

