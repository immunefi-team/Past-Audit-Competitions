# \#100063 [SC-Insight] `MarginClearinghouse.deposit(address,uint256)` is the only custody-changing user entry point without `nonReentrant` (`MarginClearinghouse.sol`)

**Submitted on Nov 16th 2026 at 07:07:07 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100063
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**

## Description

## Brief/Intro

`MarginClearinghouse` guards every custody-changing user entry point with `ReentrancyGuardTransient` except `deposit(address,uint256)`. That function reaches the same `_deposit` internals as the guarded `depositMargin` and `depositFor`, so a callback during a guarded `withdrawMargin` can re-enter custody through `deposit()` and credit a balance mid-withdrawal, while the same re-entry through `depositMargin()` is rejected. There is no harm with the real, hook-free USDG; this is a hardening Insight.

## Vulnerability Details

```solidity
// packages/perps/src/MarginClearinghouse.sol#L348-L353  (unguarded)
    function deposit(
        address account,
        uint256 amount
    ) external {
        _deposit(account, msg.sender, amount);
    }

// packages/perps/src/MarginClearinghouse.sol#L359-L363  (guarded sibling, same internals)
    function depositMargin(
        uint256 amount
    ) external nonReentrant {
        _deposit(msg.sender, msg.sender, amount);
    }
```

`depositFor` (`:374`), `withdraw` (`:406`) and `withdrawMargin` (`:416`) are all `nonReentrant`. `_deposit` makes an external token call and, when the engine is wired, an engine call (`_realizeOrCheckpointCarryBeforeMarginChange`). The unguarded selector is also the ERC-style `deposit(account, amount)` form that a contract integrator is most likely to call.

## Impact Details

With the trusted, hook-free USDG nothing can be exploited. The PoC needs a test-only USDG mock with a transfer hook (`MaliciousSettlementToken.sol`) to create the callback; the scope allows USDG to be mocked and assumes the real token has no hooks. The risk is that the guarded invariant fails silently on this one path if the settlement token ever gains hooks, or if `_deposit` gains further external calls. Submitted as a Security Best practices Insight.

## Recommendation

Add `nonReentrant` to `deposit(address,uint256)`.

## References

- `packages/perps/src/MarginClearinghouse.sol#L348-L353`, `#L359-L363`, `#L374`, `#L406`, `#L416` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at the in-scope commit. No in-scope contract is modified; only Pyth (`MockPyth`) and USDG are mocked. `PocBase.sol` loads the DEPLOYMENT.md release parameters. The full source of every file needed is included below.

**Setup:**
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the 3 files below into packages/perps/test/perps/audit/
forge test --root packages/perps --match-path "test/perps/audit/INS4_ClearinghouseDepositNoReentrancyGuard.t.sol" -vv
```

**Output:**
```
[PASS] test_Control_DepositMarginReentryBlocked() (gas: 184574)
[PASS] test_DepositReentersDuringGuardedWithdraw() (gas: 223024)
Logs:
  nested deposit credited 5000000
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

- `test_DepositReentersDuringGuardedWithdraw`: a fresh clearinghouse with the test-only hook token (engine unset). The token's transfer hook calls `deposit()` during a guarded `withdrawMargin`, and the nested deposit credits 5 USDG.
- `test_Control_DepositMarginReentryBlocked`: the same hook calling the guarded `depositMargin()` is rejected by the reentrancy guard, so the whole `withdrawMargin` reverts.

### `packages/perps/test/perps/audit/INS4_ClearinghouseDepositNoReentrancyGuard.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.sol";
import {MaliciousSettlementToken} from "./MaliciousSettlementToken.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {console2} from "forge-std/console2.sol";

/// @notice MarginClearinghouse.deposit(address,uint256) is the only custody-mutating user entry point without
///         `nonReentrant` (depositMargin, depositFor, withdraw, withdrawMargin all have it). A call-back during a
///         guarded withdraw can therefore re-enter custody through deposit(), but not through depositMargin().
///         Not exploitable with the standard USDG, which has no transfer hooks.
contract INS4_ClearinghouseDepositNoReentrancyGuardTest is PocBase {

    MaliciousSettlementToken internal hookToken;
    MarginClearinghouse internal ch;
    address internal user = makeAddr("user");

    function setUp() public override {
        super.setUp();
        hookToken = new MaliciousSettlementToken();
        ch = new MarginClearinghouse(address(hookToken)); // engine unset: pure custody behaviour
        hookToken.mint(user, 100e6);
        vm.startPrank(user);
        hookToken.approve(address(ch), type(uint256).max);
        ch.depositMargin(100e6);
        vm.stopPrank();
        // the hook re-enters as the token contract itself, depositing its own balance into its own account
        hookToken.mint(address(hookToken), 5e6);
        vm.prank(address(hookToken));
        hookToken.approve(address(ch), type(uint256).max);
    }

    function test_DepositReentersDuringGuardedWithdraw() public {
        hookToken.armHook(address(ch), abi.encodeWithSignature("deposit(address,uint256)", address(hookToken), 5e6));
        vm.prank(user);
        ch.withdrawMargin(40e6); // nonReentrant; the outbound transfer fires the hook
        console2.log("nested deposit credited", ch.balanceAssets(address(hookToken)));
        assertEq(ch.balanceAssets(address(hookToken)), 5e6, "deposit() executed inside withdraw's guard");
        assertEq(ch.balanceAssets(user), 60e6);
    }

    function test_Control_DepositMarginReentryBlocked() public {
        hookToken.armHook(address(ch), abi.encodeWithSignature("depositMargin(uint256)", 5e6));
        vm.prank(user);
        vm.expectRevert(bytes("malicious hook failed")); // inner ReentrancyGuardReentrantCall
        ch.withdrawMargin(40e6);
        assertEq(ch.balanceAssets(address(hookToken)), 0);
    }

}
```

### `packages/perps/test/perps/audit/MaliciousSettlementToken.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;

import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";

/// @dev Settlement-token fixture that violates the "standard, passive ERC20" assumption the perps
///      stack relies on. It can (a) take a bps fee on every ordinary transfer (fee-on-transfer: the
///      sender pays `amount`, the recipient receives `amount - fee`, the fee is burned), (b) rebase a
///      single address by delta, and (c) re-enter an arbitrary target contract while a transfer is in
///      flight. All knobs default to inert so it behaves exactly like `MockSettlementToken` unless the
///      test arms them. Subclasses `MockSettlementToken` so it is assignable to the inherited fixture.
contract MaliciousSettlementToken is MockSettlementToken {

    address public hookTarget;
    bytes public hookData;
    bool public hookArmed;
    uint256 public transferFeeBps;
    /// @dev When non-zero, the fee applies only to transfers whose `from` is this address. This lets a
    ///      test fee a single attacker deposit while leaving all other flows (seed, LP, outbound
    ///      settlement moves) at standard 0-fee behavior.
    address public feeSender;

    constructor() MockSettlementToken() {}

    function setFeeSender(address sender) external {
        feeSender = sender;
    }

    function armHook(address target, bytes calldata data) external {
        hookTarget = target;
        hookData = data;
        hookArmed = true;
    }

    function disarmHook() external {
        hookArmed = false;
    }

    /// @dev `to` receives `amount * (1e4 - feeBps) / 1e4`; the fee is burned off the sender.
    function setTransferFeeBps(uint256 bps) external {
        require(bps <= 1_000, "fee too large");
        transferFeeBps = bps;
    }

    /// @dev Rebases a single address by `delta` (positive mint / negative burn from that address).
    function rebase(address to, int256 delta) external {
        if (delta >= 0) {
            _mint(to, uint256(delta));
        } else {
            _burn(to, uint256(-delta));
        }
    }

    /// @dev OZ v5 customization point. Applies the fee-on-transfer and the re-entrancy hook.
    function _update(address from, address to, uint256 value) internal override {
        uint256 fee = 0;
        uint256 delivered = value;
        if (transferFeeBps > 0 && from != address(0) && to != address(0) && (feeSender == address(0) || from == feeSender)) {
            fee = (value * transferFeeBps) / 10_000;
            delivered = value - fee;
        }
        super._update(from, to, delivered);
        if (fee > 0) {
            super._update(from, address(0), fee); // burn the fee off the sender
        }
        if (hookArmed) {
            hookArmed = false;
            (bool ok,) = hookTarget.call(hookData);
            require(ok, "malicious hook failed");
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
