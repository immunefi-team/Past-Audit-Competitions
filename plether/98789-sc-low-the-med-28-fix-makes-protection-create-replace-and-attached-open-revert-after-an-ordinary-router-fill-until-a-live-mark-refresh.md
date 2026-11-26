# \#98789 [SC-Low] The MED-28 fix makes protection create, replace and attached open revert after an ordinary Router fill until a live mark refresh

**Submitted on Nov 10th 2026 at 07:45:06 UTC by @humbleservant for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98789
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

A Router-executed order writes its historical fill tick to the Engine as the new mark when that tick is newer than the current mark, and leaves the oracle's accepted basket where it was, which ORACLE_SYNCHRONIZATION.md describes as expected. `PositionProtectionBook` was written to arm in that state: its guard `_acceptedMarkInstalled()` returns true when the accepted basket is older than the Engine mark, and the attached-open path arms on exactly that. The fix for Cecuro MED-28 (PR #124) put a call to `oracle.getLatestPrice(MarkRefresh)` in front of the guard, and that view reverts `PletherOracle__PriceOutOfOrder` whenever, outside a frozen window, the accepted basket is older than the Engine mark. After such a fill, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert for every caller until a signed live payload is accepted again. A trader who opens through the Router and then asks for a stop-loss is refused because of their own fill. No funds move, and a trader who sends a live mark refresh in the same transaction gets through for one Pyth update fee, so I rate this Low. I report it under the program's rule for fixed findings, because the MED-28 fix introduces the behaviour. I found it by running the protection entry points after a Router fill in place of the fixture's direct Engine open.

## Vulnerability Details

### 1. A Router fill moves the Engine mark and leaves the accepted basket behind

```solidity
// OrderRouterExecutionSidecar.sol L1053-1061
function _updateEngineMark(
    IOrderRouterExecutionHost host,
    OracleResult memory result
) private {
    ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
    if (result.publishTime > engine_.lastMarkTime()) {
        engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
    }
}
```

The oracle keeps a second record, the accepted basket behind `getAcceptedBasketMark()`. Only `_resolveCurrentBasket` writes it (PletherOracle.sol L795-799). That function has two callers, `_updateAndGetSnapshot` (L526) and `_updateLiquidationSnapshot` (L539), both on the live update paths, and it reaches the write only when it is handed a nonempty live payload outside a freeze; L754-760 return early otherwise. The historical order-execution entry points, `updateOrderExecutionPrice` (L221) and `updateBatchOrderExecutionPrice` (L245), never reach it. ORACLE_SYNCHRONIZATION.md L150-151 says the same: historical execution may advance the Engine without advancing the accepted live basket.

A keeper fills an order at the first Pyth tick after its commit. Unless a live payload carrying a tick at least that new was accepted between the commit and the fill, the fill leaves the accepted basket older than the Engine mark. In my PoC the accepted basket's publish time is 1709535601 and the Engine mark time after the fill is 1709535602.

### 2. The Book's own guard accepts that state

```solidity
// PositionProtectionBook.sol L888-895
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) {
        return true;
    }
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

`acceptedTime < markTime` returns true. The guard rejects only an accepted basket that is ahead of the Engine mark, or equal in time with a different price. The attached path depends on the first branch: `afterOrderTerminal` (L598-602) arms a `PendingOpen` protection with `ENGINE.lastMarkTime()` right after its parent fills, and a parent fill is what puts the Engine mark ahead.

### 3. PR #124 put an ordering check in front of the guard

```solidity
// PositionProtectionBook.sol L862-886
function _freshProtectionObservation()
    private
    view
    returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
{
    IPletherOracle oracle = _activeOracle();
    IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
    if (policy.oracleFrozen) {
        revert OrderRouter__ConditionalTriggerFrozen();
    }
    uint64 markTime = ENGINE.lastMarkTime();
    cachedMarkPrice = ENGINE.lastMarkPrice();
    if (
        cachedMarkPrice == 0 || markTime == 0
            || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
    ) {
        revert OrderRouter__ProtectionMarkTooStale();
    }
    IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
    currentMarkPrice = current.markPrice;
    currentPublishTime = current.publishTime;
    if (!_acceptedMarkInstalled()) {
        revert PositionProtectionBook__OracleMarkAhead();
    }
}
```

Before PR #124 (commit `eadc5c58`) this helper was `_freshCachedProtectionMark`: it read the Engine mark and called `_acceptedMarkInstalled()`, so the Engine-ahead state passed. The guard came earlier, in PR #120 (commit `81919218`), which the audit response also links under MED-28 as its Oracle/Engine synchronization change. The MED-28 fix added the `getLatestPrice` call at L880 so that trigger prices are validated against the accepted observation. `getLatestPrice` builds its snapshot with `enforcePublishOrder = true` (PletherOracle.sol L578):

```solidity
// PletherOracle.sol L892-896, inside _snapshotFromBasket
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

The oracle interface documents this publish-order check for the view (IPletherOracle.sol L338-341), and the view does what it says. The conflict is in the Book, which now calls it on a path where its own guard allows the state: with the accepted basket older than the Engine mark, the view reverts before the Book reaches that guard. The `acceptedTime < markTime` branch of `_acceptedMarkInstalled()` can no longer be taken from `_freshProtectionObservation`. Three entry points call the helper: `createPositionProtection` (L229), `replacePositionProtection` (L295) and `_stageAttached` (L463), which `commitOpenOrderWithProtection` runs after it commits the parent order (L432-441).

### 4. What a trader sees

All four cases below run through the production Router code (deployed by the fixture inside a harness that adds one overload my PoC never calls), with an arbitrary keeper address executing the fills.

- A trader opens a position through the Router and calls `createPositionProtection`. The call reverts `PletherOracle__PriceOutOfOrder(1709535601, 1709535602)`. An `updateMarkPrice` call with an empty payload reverts with the same error, so the state cannot be repaired without a signed payload. After one live payload is accepted, the identical call arms (`test_CreateRevertsAfterOwnOpenFillsUntilLiveRefresh`).
- ALICE holds an armed protection. BOB's unrelated open fills. ALICE's `replacePositionProtection` reverts and her trigger prices stay as they were (`test_ReplaceRevertsAfterAnotherTradersFillUntilLiveRefresh`).
- BOB's open fills. ALICE's `commitOpenOrderWithProtection` reverts whole: no parent order is committed and nothing is staged (`test_AttachedOpenRevertsAfterAnotherTradersFillUntilLiveRefresh`).
- ALICE's attached protection arms while the Engine mark is ahead of the accepted basket, through `afterOrderTerminal`. In the same block and the same state, `replacePositionProtection` on that protection reverts (`test_AttachedProtectionArmsWhileEngineMarkIsAhead_ReplaceRevertsInSameState`). The Book applies two different rules to one state.

The state lasts from the fill until the next accepted live payload, and the next fill with a newer tick brings it back: the second, third and sixth tests each start from a synchronized state and lose it to BOB's fill.

### 5. Freshness is satisfied when the calls revert

SECURITY.md L563-564 says freshness checks still apply to protection creation, and the interface comment at `IPositionProtectionActions.sol` L12 says creation validates against the accepted fresh basket. In the PoC both observations are fresh by that rule when the calls revert. At the refused call the block time is 1709535606, so the accepted basket is 5 seconds old and the Engine mark 4 seconds old, against a 60-second limit for this policy (`orderExecutionStalenessLimit`, PletherOracle.sol L76, selected at OracleFreshnessPolicyLib.sol L65-68). The staleness test sits in `_acceptedLiveBasket` (PletherOracle.sol L706-727) and runs before `_snapshotFromBasket`; it passes. The error returned is the ordering error, not `PletherOracle__StalePrice` and not `OrderRouter__ProtectionMarkTooStale`.

The Cecuro report's response to MED-28 says that creation, replacement and attached-open staging validate trigger prices against the current accepted oracle observation, and that the synchronization guard rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark. An accepted basket behind the Engine mark is neither of those two cases. `_acceptedMarkInstalled()` passes it, and WORKING_WITH_AI_AGENTS.md L270-272 gives the failing cases for the attached path as an accepted basket that is newer than the Engine mark or disagrees with it at the same timestamp. The response also says a qualifying mark has to be installed before arming. In the state I describe, the installed Engine mark is the newer of the two observations, and the oracle verified the Pyth payload it came from when the fill ran.

I searched the twelve documents in `packages/perps` and the comments in the Book and its interfaces for a statement that a live refresh has to precede these three calls, and found none. WORKING_WITH_AI_AGENTS.md L249-251 states what creation and replacement require of the oracle: thresholds on the untriggered side of a fresh mark, and an unfrozen oracle.

### 6. Limits

- A trader who sends a live `updateMarkPrice` and the protection call in one transaction is not blocked. `test_RefreshAndCreateInOneTransactionSucceeds` does this from a smart account: the account pays exactly one Pyth update fee, and the two calls use 644,452 gas in the fixture.
- The three calls already need an accepted basket whose components are at most 60 seconds old (PletherOracle.sol L717-722), and only a live Router update writes that basket. Without a live update whose ticks are that recent they revert on freshness, which is the documented requirement: `PletherOracle__StalePrice` from the view when a fill has kept the Engine mark fresh, and `OrderRouter__ProtectionMarkTooStale` (PositionProtectionBook.sol L874-879) when the Engine mark is stale as well. My PoC does not exercise either case. What I report is the refusal inside that window, when both observations are fresh.
- Protections that are already armed keep working. `test_ArmedProtectionStillCancelsAndTriggersWhileEngineMarkIsAhead` cancels one and triggers another in the Engine-ahead state, and the trigger's live payload resynchronizes the accepted basket.
- The three failing calls revert whole, so no trigger bounty or margin stays locked.
- The protocol's own client package, `packages/perps-aa-client`, is outside the competition scope and I cite it only for the intended call shape. It builds every trader protection action (create, replace, cancel, cancel-and-close, amend and protected open) as one call to the Book with zero native value (`src/protection.ts` L41-45, README L205-206), and its `src` has no builder for `updateMarkPrice`. The only protection builder that carries native value is the keeper's trigger (`src/keeperExecution.ts` L52-55). A trader on that client sends the bare call and depends on someone else's live update landing after the last fill.

### 7. Suggested fix

Handle the Engine-ahead case in `_freshProtectionObservation` before calling the view. When `acceptedTime < ENGINE.lastMarkTime()`, validate the trigger prices against the Engine mark and use the Engine mark time as the arming baseline, the baseline `afterOrderTerminal` already uses (L602). Keep the `getLatestPrice` path when the accepted basket is at or ahead of the Engine mark. MED-28 was about an Engine mark older than the accepted observation, so I believe this keeps that fix intact.

## Impact Details

Impact: "Contract fails to deliver promised returns, but doesn't lose value". After a Router fill whose tick is newer than the accepted basket, three trader entry points revert for every caller until a live payload is accepted: `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection`. A position opened through the Router cannot receive a stop-loss or take-profit through the single call that the interface and the client package expose, an armed stop-loss cannot be tightened, and an open with an attached protection cannot be committed.

No funds are lost or frozen. The calls revert whole, existing protections still trigger and cancel, and a trader who bundles a live refresh pays one Pyth update fee and gets through. Those limits are why I selected Low and not the Griefing line: a third party who keeps the state alive with their own fills causes reverts that each affected trader can route around.

## References

- Frozen competition commit (the code is not deployed): https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Observation helper with the added view call: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L886
- The Book's guard: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L888-L895
- Attached arming in `afterOrderTerminal`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L581-L604
- Call sites: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L229 , https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L295 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L463
- Ordering check in the view: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L571-L580 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L901
- Accepted basket's only writer: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L801
- Engine mark write on a fill: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061
- Freshness limit for the policy: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L65-L68
- PR #124, "fix(perps): harden trader protection, liquidation pricing, and manual exits", commit `eadc5c587e2f2f1ec26c51132472ab876d6edc15`, the change to `PositionProtectionBook.sol`. Its commit message lists "fix(perps): validate protection against current oracle publication".
- Cecuro AI audit report (2026-10-26), MED-28 and the project's response to it: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf . The response links two plether-core commits. `81919218` is PR #120 in this repository and added `_acceptedMarkInstalled()`. `8f6cfd3d` is not in the history of the competition commit, where its change is part of the squashed PR #124 commit above. GitHub still serves it under this repository, titled "fix(perps): validate protection against current oracle publication", the title the PR #124 commit message lists: https://github.com/immunefi-team/audit-comp-plether/commit/8f6cfd3d84d1289aa1209e3b08a06e9bfdf38a0d
- `packages/perps/ORACLE_SYNCHRONIZATION.md` L149-153, `packages/perps/SECURITY.md` L563-565 and L1328-1341, `packages/perps/WORKING_WITH_AI_AGENTS.md` L249-251 and L268-272, `packages/perps/src/interfaces/IPositionProtectionActions.sol` L10-14, `packages/perps/src/interfaces/IPletherOracle.sol` L338-341
- Known items checked: Cecuro MED-28 (arming from a stale Engine mark, the inverse state), MED-17 (oracle lag blocking triggers), MED-30 (frozen-market triggers), LOW-73 (creation in degraded mode) and the October scan finding "Later mark refresh can make live historical-price orders permanently out of order" (a pending order's tick against the Engine mark, a different pair). SECURITY.md L1341 lists dependence on keeper infrastructure and oracle publication as a known limitation; in the PoC the keeper executed the fill and the accepted basket is 5 seconds old, so neither is missing.

## Proof of Concept

## Proof of Concept

Save the file below as `packages/perps/test/poc/ProtectionArmingAfterOrderFill.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It extends the repo's `BasePerpTest`, so the oracle, execution sidecars, Engine, clearinghouse and `PositionProtectionBook` are the real contracts. The Router is the fixture's `LegacyOrderRouterHarness`, which is the production `OrderRouter` plus a test-only scalar commit overload that my PoC never calls. The token and Pyth are the fixture's mocks, and the fixture's default risk parameters and two-feed basket stay as they are. Standalone orders are committed with the production `commitOrder(OrderRequest)`, attached opens with the Book's `commitOpenOrderWithProtection`, and every fill is executed by an arbitrary keeper address through `executeOrder`. Historical payloads carry `publishTime = commitTime + 1` and `prevPublishTime = commitTime` for each feed, with zero confidence. Every payload carries the price 1e8 except the triggering payload in the sixth test, which carries the take-profit price. The mock Pyth fee is zero except in the fifth test, which sets it to 1e9 wei, an arbitrary mock value, to show that the bundled refresh costs exactly one update fee.

The six tests:

1. `test_CreateRevertsAfterOwnOpenFillsUntilLiveRefresh`: a trader's own Router-filled open makes `createPositionProtection` revert; an empty-payload refresh cannot repair it; after a live payload the same call arms.
2. `test_ReplaceRevertsAfterAnotherTradersFillUntilLiveRefresh`: another trader's fill makes `replacePositionProtection` revert on an armed protection; after a live payload it applies.
3. `test_AttachedOpenRevertsAfterAnotherTradersFillUntilLiveRefresh`: another trader's fill makes `commitOpenOrderWithProtection` revert with nothing committed; after a live payload it commits and stages.
4. `test_AttachedProtectionArmsWhileEngineMarkIsAhead_ReplaceRevertsInSameState`: the attached path arms in the Engine-ahead state and the standalone replacement reverts in that same state.
5. `test_RefreshAndCreateInOneTransactionSucceeds`: limit. A smart account that sends a live refresh and the create call together arms and pays one Pyth update fee.
6. `test_ArmedProtectionStillCancelsAndTriggersWhileEngineMarkIsAhead`: limit. Armed protections cancel and trigger in the Engine-ahead state.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IRouterCommit {
    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64 orderId);
}

interface IRouterMarkRefresh {
    function updateMarkPrice(
        bytes[] calldata pythUpdateData
    ) external payable;
}

/// @dev Minimal smart-account trader with a single-call entry (`exec`) and a batch entry (`execBatch`). Used to show
///      a live refresh and a protection call inside one transaction.
contract BatchingTraderAccount {

    address internal immutable OWNER;

    constructor() {
        OWNER = msg.sender;
    }

    receive() external payable {}

    function exec(
        address target,
        uint256 value,
        bytes calldata data
    ) external payable returns (bytes memory ret) {
        require(msg.sender == OWNER, "owner");
        bool ok;
        (ok, ret) = target.call{value: value}(data);
        if (!ok) {
            assembly ("memory-safe") {
                revert(add(ret, 32), mload(ret))
            }
        }
    }

    function execBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata datas
    ) external payable {
        require(msg.sender == OWNER, "owner");
        for (uint256 i; i < targets.length; ++i) {
            (bool ok, bytes memory ret) = targets[i].call{value: values[i]}(datas[i]);
            if (!ok) {
                assembly ("memory-safe") {
                    revert(add(ret, 32), mload(ret))
                }
            }
        }
    }

}

/// @dev Behaviour under test. A Router-executed order installs the Engine mark from its historical post-commit tick
///      (`OrderRouterExecutionSidecar._updateEngineMark`) and never writes the oracle's accepted basket.
///      `PositionProtectionBook._freshProtectionObservation` reads `getLatestPrice(MarkRefresh)`, which rejects an
///      accepted basket older than the Engine mark, so `createPositionProtection`, `replacePositionProtection` and
///      `commitOpenOrderWithProtection` revert until a live payload is accepted.
/// @dev Every order here is committed and executed through the production Router by an arbitrary keeper address.
///      Historical fills carry explicit per-feed updates with publishTime = commitTime + 1 and prevPublishTime =
///      commitTime, the shape Pyth's unique parse accepts. Clock reads use `vm.getBlockTimestamp()`.
contract ProtectionArmingAfterOrderFillTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant MARK = 1e8;
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant MARGIN = 2000e6;
    uint256 internal constant TAKE_PROFIT = 90_000_000;
    uint256 internal constant STOP_LOSS = 110_000_000;
    uint256 internal constant PYTH_FEE = 1e9;

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;

    function setUp() public override {
        super.setUp();
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        _fundTrader(BOB, 20_000e6);
        _fundTrader(CAROL, 20_000e6);
        vm.deal(KEEPER, 10 ether);
        _acceptLiveMark(MARK);
    }

    // ------------------------------------------------------------------
    // 1. A trader opens through the Router, then tries to add a protection
    // ------------------------------------------------------------------

    function test_CreateRevertsAfterOwnOpenFillsUntilLiveRefresh() public {
        (uint64 acceptedBefore,) = _accepted();
        assertEq(engine.lastMarkTime(), acceptedBefore, "start: accepted basket and Engine mark agree");

        (uint64 orderId, uint64 commitTime) = _commitOpen(ALICE, CfdTypes.Side.LONG);
        _advance(5, 2);
        OrderV3Types.ExecutionResult memory fill = _executeAtFirstPostCommitTick(orderId, commitTime, MARK);
        assertEq(uint8(fill.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "open filled");

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, SIZE, "position exists");
        uint64 engineMarkTime = engine.lastMarkTime();
        (uint64 acceptedAfter,) = _accepted();
        assertEq(engineMarkTime, commitTime + 1, "fill installed its historical tick as the Engine mark");
        assertEq(acceptedAfter, acceptedBefore, "fill did not write the accepted basket");
        emit log_named_uint("accepted basket publish time", acceptedAfter);
        emit log_named_uint("Engine mark time after the fill", engineMarkTime);
        emit log_named_uint("block time at the refused create", vm.getBlockTimestamp());

        // The view the Book reads reverts in this state, and so does creation.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedAfter, engineMarkTime)
        );
        pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(TAKE_PROFIT, STOP_LOSS);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedAfter, engineMarkTime)
        );
        actions.createPositionProtection(params);
        assertEq(views.activePositionProtectionId(ALICE), 0, "no protection armed");

        // A refresh without a signed payload cannot repair it.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedAfter, engineMarkTime)
        );
        router.updateMarkPrice(new bytes[](0));

        // Control: once a live payload is accepted, the same call arms.
        _acceptLiveMark(MARK);
        (uint64 acceptedRepaired,) = _accepted();
        assertGe(acceptedRepaired, engine.lastMarkTime(), "live refresh re-synchronizes the accepted basket");
        vm.prank(ALICE);
        uint64 protectionId = actions.createPositionProtection(params);
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "armed after the live refresh"
        );
    }

    // ------------------------------------------------------------------
    // 2. Another trader's fill blocks replacement of an armed protection
    // ------------------------------------------------------------------

    function test_ReplaceRevertsAfterAnotherTradersFillUntilLiveRefresh() public {
        _openThroughRouter(ALICE, CfdTypes.Side.LONG);
        _advance(1, 1);
        _acceptLiveMark(MARK);
        vm.prank(ALICE);
        uint64 protectionId = actions.createPositionProtection(_params(TAKE_PROFIT, STOP_LOSS));
        (uint64 acceptedBefore,) = _accepted();

        (uint64 bobOrderId, uint64 bobCommitTime) = _commitOpen(BOB, CfdTypes.Side.SHORT);
        _advance(5, 2);
        _executeAtFirstPostCommitTick(bobOrderId, bobCommitTime, MARK);
        uint64 engineMarkTime = engine.lastMarkTime();
        (uint64 acceptedAfter,) = _accepted();
        assertEq(engineMarkTime, bobCommitTime + 1, "BOB's fill moved the Engine mark");
        assertEq(acceptedAfter, acceptedBefore, "accepted basket unchanged");

        PositionProtectionTypes.PositionProtectionParams memory tighter = _params(95_000_000, 105_000_000);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedAfter, engineMarkTime)
        );
        actions.replacePositionProtection(protectionId, tighter);
        PositionProtectionTypes.PositionProtectionView memory unchanged = views.getPositionProtection(protectionId);
        assertEq(unchanged.takeProfitTriggerPrice, TAKE_PROFIT, "take-profit unchanged");
        assertEq(unchanged.stopLossTriggerPrice, STOP_LOSS, "stop-loss unchanged");

        // Control: after a live refresh the replacement applies.
        _acceptLiveMark(MARK);
        vm.prank(ALICE);
        actions.replacePositionProtection(protectionId, tighter);
        PositionProtectionTypes.PositionProtectionView memory replaced = views.getPositionProtection(protectionId);
        assertEq(replaced.takeProfitTriggerPrice, 95_000_000, "take-profit replaced");
        assertEq(replaced.stopLossTriggerPrice, 105_000_000, "stop-loss replaced");
    }

    // ------------------------------------------------------------------
    // 3. Another trader's fill blocks an open with an attached protection
    // ------------------------------------------------------------------

    function test_AttachedOpenRevertsAfterAnotherTradersFillUntilLiveRefresh() public {
        (uint64 acceptedBefore,) = _accepted();
        (uint64 bobOrderId, uint64 bobCommitTime) = _commitOpen(BOB, CfdTypes.Side.SHORT);
        _advance(5, 2);
        _executeAtFirstPostCommitTick(bobOrderId, bobCommitTime, MARK);
        uint64 engineMarkTime = engine.lastMarkTime();
        (uint64 acceptedAfter,) = _accepted();
        assertEq(engineMarkTime, bobCommitTime + 1, "BOB's fill moved the Engine mark");
        assertEq(acceptedAfter, acceptedBefore, "accepted basket unchanged");

        OrderV3Types.OrderRequest memory aliceOpen = _request(CfdTypes.Side.LONG, false);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(TAKE_PROFIT, STOP_LOSS);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedAfter, engineMarkTime)
        );
        actions.commitOpenOrderWithProtection(aliceOpen, params);
        assertEq(router.pendingOrderCounts(ALICE), 0, "no parent order committed");
        assertEq(views.activePositionProtectionId(ALICE), 0, "nothing staged");

        // Control: after a live refresh the same request commits the parent and stages the protection.
        _acceptLiveMark(MARK);
        aliceOpen = _request(CfdTypes.Side.LONG, false);
        vm.prank(ALICE);
        (uint64 parentOrderId, uint64 protectionId) = actions.commitOpenOrderWithProtection(aliceOpen, params);
        assertGt(parentOrderId, 0, "parent committed");
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.PendingOpen),
            "protection staged after the live refresh"
        );
    }

    // ------------------------------------------------------------------
    // 4. The attached path arms in the very state the standalone paths refuse
    // ------------------------------------------------------------------

    function test_AttachedProtectionArmsWhileEngineMarkIsAhead_ReplaceRevertsInSameState() public {
        OrderV3Types.OrderRequest memory aliceOpen = _request(CfdTypes.Side.LONG, false);
        vm.prank(ALICE);
        (uint64 parentOrderId, uint64 protectionId) =
            actions.commitOpenOrderWithProtection(aliceOpen, _params(TAKE_PROFIT, STOP_LOSS));
        uint64 parentCommitTime = uint64(vm.getBlockTimestamp());

        // The parent's own fill puts the Engine mark ahead of the accepted basket.
        _advance(5, 2);
        OrderV3Types.ExecutionResult memory fill = _executeAtFirstPostCommitTick(parentOrderId, parentCommitTime, MARK);
        assertEq(uint8(fill.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "parent filled");
        (uint64 acceptedTime,) = _accepted();
        uint64 engineMarkTime = engine.lastMarkTime();
        assertLt(acceptedTime, engineMarkTime, "Engine mark ahead of the accepted basket at arming time");
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "attached protection armed while the Engine mark is ahead"
        );

        // Same block, same state: the standalone replacement of that protection reverts.
        PositionProtectionTypes.PositionProtectionParams memory tighter = _params(95_000_000, 105_000_000);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineMarkTime)
        );
        actions.replacePositionProtection(protectionId, tighter);
    }

    // ------------------------------------------------------------------
    // 5. Limit: a trader who refreshes in the same transaction is not blocked
    // ------------------------------------------------------------------

    function test_RefreshAndCreateInOneTransactionSucceeds() public {
        BatchingTraderAccount account = _newFundedAccount(20_000e6);
        OrderV3Types.OrderRequest memory open = _request(CfdTypes.Side.LONG, false);
        uint64 orderId =
            abi.decode(account.exec(address(router), 0, abi.encodeCall(IRouterCommit.commitOrder, (open))), (uint64));
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        _advance(5, 2);
        _executeAtFirstPostCommitTick(orderId, commitTime, MARK);
        (uint64 acceptedTime,) = _accepted();
        uint64 engineMarkTime = engine.lastMarkTime();
        assertLt(acceptedTime, engineMarkTime, "Engine mark ahead after the account's own fill");

        bytes memory createCall =
            abi.encodeCall(IPositionProtectionActions.createPositionProtection, (_params(TAKE_PROFIT, STOP_LOSS)));
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineMarkTime)
        );
        account.exec(address(actions), 0, createCall);

        baseMockPyth.setFee(PYTH_FEE);
        bytes[] memory live = _livePayload(MARK);
        uint256 refreshFee = pletherOracle.getUpdateFee(live);
        assertEq(refreshFee, PYTH_FEE, "refresh costs one Pyth update fee");
        emit log_named_uint("refresh fee paid by the trader (wei)", refreshFee);
        vm.deal(address(account), 1 ether);

        address[] memory targets = new address[](2);
        uint256[] memory values = new uint256[](2);
        bytes[] memory datas = new bytes[](2);
        targets[0] = address(router);
        values[0] = refreshFee;
        datas[0] = abi.encodeCall(IRouterMarkRefresh.updateMarkPrice, (live));
        targets[1] = address(actions);
        datas[1] = createCall;
        uint256 gasBefore = gasleft();
        account.execBatch(targets, values, datas);
        emit log_named_uint("gas: refresh + create in one transaction", gasBefore - gasleft());

        uint64 protectionId = views.activePositionProtectionId(address(account));
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "armed in the same transaction as the refresh"
        );
        assertEq(address(account).balance, 1 ether - refreshFee, "the trader paid exactly one Pyth update fee");
    }

    // ------------------------------------------------------------------
    // 6. Limit: protections that are already armed keep cancelling and triggering
    // ------------------------------------------------------------------

    function test_ArmedProtectionStillCancelsAndTriggersWhileEngineMarkIsAhead() public {
        _openThroughRouter(ALICE, CfdTypes.Side.LONG);
        _openThroughRouter(CAROL, CfdTypes.Side.LONG);
        _advance(1, 1);
        _acceptLiveMark(MARK);
        vm.prank(ALICE);
        uint64 aliceProtectionId = actions.createPositionProtection(_params(TAKE_PROFIT, STOP_LOSS));
        vm.prank(CAROL);
        uint64 carolProtectionId = actions.createPositionProtection(_params(TAKE_PROFIT, STOP_LOSS));

        (uint64 bobOrderId, uint64 bobCommitTime) = _commitOpen(BOB, CfdTypes.Side.SHORT);
        _advance(3, 2);
        _executeAtFirstPostCommitTick(bobOrderId, bobCommitTime, MARK);
        (uint64 acceptedTime,) = _accepted();
        assertLt(acceptedTime, engine.lastMarkTime(), "Engine mark ahead after BOB's fill");

        vm.prank(CAROL);
        actions.cancelPositionProtection(carolProtectionId);
        assertEq(
            uint8(views.getPositionProtection(carolProtectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Cancelled),
            "cancel works while the Engine mark is ahead"
        );

        _advance(2, 2);
        bytes[] memory crossing = _livePayload(TAKE_PROFIT);
        uint256 fee = pletherOracle.getUpdateFee(crossing);
        vm.prank(KEEPER);
        actions.triggerPositionProtection{value: fee}(aliceProtectionId, crossing);
        assertEq(
            uint8(views.getPositionProtection(aliceProtectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "an armed protection still triggers"
        );
        (acceptedTime,) = _accepted();
        assertGe(acceptedTime, engine.lastMarkTime(), "the trigger's live payload re-synchronizes the accepted basket");
    }

    // ------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------

    function _openThroughRouter(
        address trader,
        CfdTypes.Side side
    ) internal {
        (uint64 orderId, uint64 commitTime) = _commitOpen(trader, side);
        _advance(2, 2);
        OrderV3Types.ExecutionResult memory fill = _executeAtFirstPostCommitTick(orderId, commitTime, MARK);
        assertEq(uint8(fill.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "setup open filled");
    }

    function _newFundedAccount(
        uint256 amount
    ) internal returns (BatchingTraderAccount account) {
        account = new BatchingTraderAccount();
        settlementToken.mint(address(account), amount);
        account.exec(address(settlementToken), 0, abi.encodeCall(IERC20.approve, (address(clearinghouse), amount)));
        account.exec(
            address(clearinghouse), 0, abi.encodeWithSignature("deposit(address,uint256)", address(account), amount)
        );
    }

    function _commitOpen(
        address trader,
        CfdTypes.Side side
    ) internal returns (uint64 orderId, uint64 commitTime) {
        OrderV3Types.OrderRequest memory request = _request(side, false);
        vm.prank(trader);
        orderId = router.commitOrder(request);
        commitTime = uint64(vm.getBlockTimestamp());
    }

    function _executeAtFirstPostCommitTick(
        uint64 orderId,
        uint64 commitTime,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _historicalPayload(price, commitTime);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.prank(KEEPER);
        result = router.executeOrder{value: fee}(orderId, data);
    }

    function _advance(
        uint256 secondsForward,
        uint256 blocksForward
    ) internal {
        vm.warp(vm.getBlockTimestamp() + secondsForward);
        vm.roll(vm.getBlockNumber() + blocksForward);
    }

    function _accepted() internal view returns (uint64 publishTime, uint256 price) {
        (, price, publishTime) = pletherOracle.getAcceptedBasketMark();
    }

    function _acceptLiveMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes[] memory live = _livePayload(price);
        uint256 fee = pletherOracle.getUpdateFee(live);
        router.updateMarkPrice{value: fee}(live);
    }

    function _livePayload(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 publishTime = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _historicalPayload(
        uint256 price,
        uint64 commitTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, uint256(commitTime) + 1, uint256(commitTime))
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _request(
        CfdTypes.Side side,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = SIZE;
        request.marginDelta = isClose ? 0 : MARGIN;
        if (isClose) {
            request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        } else {
            request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        }
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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

}
```

Run from `packages/perps` (the repo compiles with via_ir, so the first run takes a few minutes):

```bash
FOUNDRY_PROFILE=quick forge test --match-path test/poc/ProtectionArmingAfterOrderFill.t.sol -vv
```

Expected output:

```
Ran 6 tests for test/poc/ProtectionArmingAfterOrderFill.t.sol:ProtectionArmingAfterOrderFillTest
[PASS] test_ArmedProtectionStillCancelsAndTriggersWhileEngineMarkIsAhead() (gas: 7910646)
[PASS] test_AttachedOpenRevertsAfterAnotherTradersFillUntilLiveRefresh() (gas: 5439356)
[PASS] test_AttachedProtectionArmsWhileEngineMarkIsAhead_ReplaceRevertsInSameState() (gas: 2946029)
[PASS] test_CreateRevertsAfterOwnOpenFillsUntilLiveRefresh() (gas: 3252030)
Logs:
  accepted basket publish time: 1709535601
  Engine mark time after the fill: 1709535602
  block time at the refused create: 1709535606

[PASS] test_RefreshAndCreateInOneTransactionSucceeds() (gas: 3491856)
Logs:
  refresh fee paid by the trader (wei): 1000000000
  gas: refresh + create in one transaction: 644452

[PASS] test_ReplaceRevertsAfterAnotherTradersFillUntilLiveRefresh() (gas: 5154226)
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 23.17ms (42.99ms CPU time)

Ran 1 test suite in 27.09ms (23.17ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```
