# \#99696 [SC-Low] Creating, replacing or attaching TP/SL protection reverts after any order execution until a separate live mark refresh lands (redundant accepted-basket read added by the MED-28 fix)

**Submitted on Nov 14th 2026 at 20:35:46 UTC by @EntropyResearch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99696
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity:** Low — *Contract fails to deliver promised returns, but doesn't lose value*. Precedent: Cecuro LOW-73 ("Trader cannot add new stop-loss protection while degraded mode is active") was rated Low and fixed in the same PR (#124) that introduced this issue.
**Target:** `packages/perps/src/PositionProtectionBook.sol` (in scope), interacting with `PletherOracle.sol` and `OrderRouterExecutionSidecar.sol`
**Commit:** a82d4fefb02768e5b3bddcc97dee371eee24fea8 (introduced by eadc5c5, "fix(perps): harden trader protection, liquidation pricing, and manual exits (#124)")

## Brief / Intro

The fix for MED-28 / MED-17 added an `oracle.getLatestPrice(PriceMode.MarkRefresh)` read to `PositionProtectionBook._freshProtectionObservation`, which `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` all call. That view reads the oracle's **accepted live basket** and reverts with `PletherOracle__PriceOutOfOrder` whenever the basket is older than `engine.lastMarkTime()`.

Every `executeOrder` / `executeOrderBatch` call that resolves a post-commit Pyth tick newer than the last accepted live basket moves the Engine mark forward without touching the accepted basket. This happens whether the order fills, fails terminally or stays pending, because the Router installs the mark in the outer frame before the per-order rollback frame. From that moment the three protection functions revert until someone sends `router.updateMarkPrice` with freshly signed Pyth data.

Consequences: a trader who opens a position through the Router cannot add a take-profit / stop-loss in the next transaction; any unrelated trader's execution between a victim's refresh and its create re-creates the state; opens with protection attached revert after any execution. Before eadc5c5 these calls required only an Engine mark at most 60 s old, so a mark freshened by an order execution was sufficient. The regression is exactly that delta, and the added read provides no extra protection (see "Why the added read protects nothing").

## Vulnerability Details

`_freshProtectionObservation` is called by `_create` (L229), `_replace` (L295) and `_stageAttached` (L463):

```solidity
// PositionProtectionBook.sol:862-886
uint64 markTime = ENGINE.lastMarkTime();
cachedMarkPrice = ENGINE.lastMarkPrice();
if (cachedMarkPrice == 0 || markTime == 0 || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)) {
    revert OrderRouter__ProtectionMarkTooStale();          // pre-existing requirement (Engine mark <= 60 s old)
}
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // added in eadc5c5
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
```

`getLatestPrice(MarkRefresh)` → `_getLatestPriceSnapshotWithConfidence` (`PletherOracle.sol:571-580`) → `_acceptedLiveBasket` (L706-727, reads `_acceptedPrices`) → `_snapshotFromBasket(basket, policy, true)`, which enforces ordering against the Engine mark:

```solidity
// PletherOracle.sol:892-896
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

Order execution advances the Engine mark but never the accepted basket:

- Historical resolution (`_resolveHistoricalOrderBasket`, `PletherOracle.sol:596-643`) parses a unique post-commit tick; `_acceptedPrices` / `_acceptedBasket` are written only in `_resolveCurrentBasket` (L795-799), i.e. by live refresh, liquidation, protection trigger and atomic LP settlement.
- `OrderRouterExecutionSidecar._updateEngineMark` (L1053-1061) installs the newer tick as the Engine mark (`if (result.publishTime > engine_.lastMarkTime())`). It is called from `_prepareSingleOracle` (L979) and `_prepareBatchOracle` (L1036) in the outer frame, before the per-item call (L209-231, L292-322), so terminal failures (slippage, planner rejection) and pending results leave the mark advanced as well.

After such an execution `acceptedTime < engine.lastMarkTime()` and the view reverts. The same state also surfaces earlier, as `PletherOracle__StalePrice` from `_acceptedLiveBasket` (L719-722), once the accepted basket is older than the 60 s MarkRefresh limit while executions keep the Engine mark itself fresh. (After a governance oracle rotation the new oracle has no accepted basket and the read reverts with `NoAcceptedBasket` until the first live update; this is a trusted-role event repaired by one refresh and is mentioned only for completeness.)

### Why the added read protects nothing

The accepted live basket is written only by `PletherOracle.updatePrice` / `updateLiquidationPrice` / `updateLiquidationBatchPrice`, which revert unless `msg.sender == engine.orderRouter()` (`_updateAndGetSnapshot`, `_updateLiquidationSnapshot`). Every Router call site installs that snapshot into the Engine in the same transaction (`router/OrderLiquidationBatchLogic.sol` L561-563 LP settlement, L607-610 mark refresh / protection trigger, L718-723 single liquidation, L755-757 batch liquidation), and order executions only move the Engine mark forward. Therefore `acceptedTime <= engine.lastMarkTime()` always holds, and equal times imply equal prices (`_snapshotFromBasket` L897-901). It follows that `getLatestPrice(MarkRefresh)` either reverts or returns exactly the Engine mark, and that the "accepted mark ahead of the Engine" state MED-28 guards against cannot arise in production (Router-only oracle writers arrived in 8191921 / #120, before this read was added in eadc5c5 / #124). The project's own MED-28 tests have to `vm.prank(address(router))` on the oracle to construct that state (`test/perps/spec/trader/PositionProtection.t.sol:281-286, 2472-2476, 3425-3428`).

The Book's own consistency check treats the reverting state as valid:

```solidity
// PositionProtectionBook.sol:888-895
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) return true;
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

The `acceptedTime < markTime` branch is unreachable from create / replace / attach because `getLatestPrice` has already reverted, while `afterOrderTerminal` (L596-604) arms attached protection from exactly that state.

### What is and is not documented

- The dependency on a valid accepted basket is mentioned: `IPositionProtectionActions.sol:12` says creation "validates trigger geometry against the accepted fresh basket"; the team's MED-28 response says the guard "rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark. A qualifying mark must be installed before arming can proceed"; `ORACLE_SYNCHRONIZATION.md:150-151` notes that "historical execution may advance the Engine without advancing the accepted live basket".
- None of these says that every order execution invalidates the basket for protection management, and they describe the "fresh Engine mark" and "current accepted basket" requirements as jointly satisfiable. In PoC tests 1-3 the accepted basket is only seconds old and the calls still revert. The MED-28 response addresses the accepted mark being *ahead of* the Engine, the opposite of the state reported here.
- Trader- and integrator-facing documentation lists only "a fresh mark" as the precondition (`CANONICAL_ENTRYPOINTS.md:116-131`, `WORKING_WITH_AI_AGENTS.md:248-262`, `SECURITY.md:563-565`, `DEPLOYMENT.md:406-410`).
- The repository's own client builds create / replace / protected-open as single-call, zero-value plans targeting the Book (`packages/perps-aa-client/src/protection.ts:41-56, 109-124`; its README: "Every builder targets the configured PositionProtectionBook with zero native value"). `updateMarkPrice` lives on the Router and needs `msg.value >= pythFee` (`OrderLiquidationBatchLogic.sol:603-606`); neither contract offers a multicall.
- Tests: most protection tests arm through `_createProtectionFor` (`PositionProtection.t.sol:3898-3909`), which pranks the Router at the timestamp of the last live refresh. The eadc5c5 tests that execute through the Router insert a live refresh before creating protection (L1845-1849, L1933-1941, L2311; at L1933-1941 the executed order fails with Slippage and the refresh re-posts the same price). No test asserts the revert.

## Impact Details

- Documented risk-management functions stop working after each qualifying execution until a separate live refresh lands:
  - `createPositionProtection` — a trader who just opened through the Router cannot add a TP/SL.
  - `replacePositionProtection` — a trader cannot tighten an existing stop-loss after anyone else's execution.
  - `commitOpenOrderWithProtection` — opens with TP/SL attached revert.
- Workaround and its limits: the trader must first send `router.updateMarkPrice` with freshly signed Pyth data (the Pyth fee is dust; the cost is a second transaction that an EOA cannot make atomic with the Book call). `updateMarkPrice` with an empty payload replays the same accepted basket and reverts the same way. An execution with a newer tick landing between the two transactions breaks it again.
- Traders who cannot set or tighten a stop-loss stay exposed at up to 500x leverage. No funds are lost or frozen and exits (`cancelPositionProtection`, `cancelPositionProtectionAndCommitClose`, ordinary closes) remain available, hence Low.

Eligibility: this is a new bug introduced by the fix for previously reported issues (MED-28 / MED-17), which the program states is valid ("a report on the same root cause is valid only if it shows that the fix ... is incomplete or introduces a new bug"). It is not in the known-issues list, the previous reports or the `SECURITY.md` known limitations.

## Risk Breakdown

- **Likelihood: high.** No attacker is required. Every `executeOrder` / `executeOrderBatch` that resolves a post-commit tick newer than the last accepted live basket creates the state, including an order that fails. The ordinary flow "open through the Router, then add a stop-loss" hits it.
- **Difficulty to trigger deliberately: trivial.** Any account can get any order executed through the permissionless keeper path, which re-creates the state between a victim's refresh and its protection call.
- **Impact: availability of risk-management functions only.** No funds are lost or frozen; exits stay available. Funds at risk: none directly. Traders who cannot set or tighten a stop-loss carry unprotected exposure at up to 500x leverage until they find a way to refresh and call again.

## Recommendation

Use the Engine mark and its publish time as the current observation in `_freshProtectionObservation`. The Engine mark is already freshness-checked, and by the invariant above it is never older than the accepted basket, so the `getLatestPrice(MarkRefresh)` read adds no information while adding the revert paths described here. Keep `_acceptedMarkInstalled()` as the guard against an accepted basket that is ahead of, or inconsistent with, the Engine mark.

```solidity
uint64 markTime = ENGINE.lastMarkTime();
cachedMarkPrice = ENGINE.lastMarkPrice();
if (cachedMarkPrice == 0 || markTime == 0 || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)) {
    revert OrderRouter__ProtectionMarkTooStale();
}
currentMarkPrice = cachedMarkPrice;
currentPublishTime = markTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
```

Add a regression test that executes an order through the real Router (historical tick) before creating, replacing and attaching protection, plus an invariant that the accepted basket time never exceeds the Engine mark time.

## References

- `packages/perps/src/PositionProtectionBook.sol:229, 295, 463, 596-604, 862-895`
- `packages/perps/src/PletherOracle.sol:571-580` (`_getLatestPriceSnapshotWithConfidence`), `596-643` (`_resolveHistoricalOrderBasket`), `706-727` (`_acceptedLiveBasket`), `749-799` (`_resolveCurrentBasket`), `878-903` (`_snapshotFromBasket`)
- `packages/perps/src/OrderRouterExecutionSidecar.sol:209-231, 292-322, 979, 1036, 1053-1061`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:561-563, 603-610, 718-723, 755-757`
- `packages/perps/src/interfaces/IPositionProtectionActions.sol:12`; `packages/perps/ORACLE_SYNCHRONIZATION.md:150-151`
- Introducing commit: eadc5c5 (#124)

## Proof of Concept

## Proof of Concept

### How to run

No RPC, fork or environment variables are needed; the PoC runs in the repository's own Foundry test suite at the commit in scope, as this program's PoC rules allow.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# copy the files below into packages/perps/test/perps/poc/
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/perps/poc/ProtectionBlockedAfterFillPoC.t.sol' -vv
```

Tested with Foundry `forge 1.9.0` (solc 0.8.35 is selected by `foundry.toml`). The package compiles with via-IR, so the first run takes roughly 10 minutes. `FOUNDRY_SPARSE_MODE=true` only limits compilation to the matched test files; the results are the same without it.

**Conformity with the program's PoC rules.** The tests use the repository's own `BasePerpTest` fixture, which deploys the real `CfdEngine`, `MarginClearinghouse`, `HousePool`, `TrancheVault`s, `PletherOracle` and `OrderRouter` stack. Only out-of-scope components are mocked: Pyth (`MockPyth`) and USDG (`MockSettlementToken`). No in-scope contract is modified. The fixture deploys the router as `LegacyOrderRouterHarness`, a repository test utility that inherits the production `OrderRouter` without overriding any function; every call in these tests uses production selectors (`commitOrder(OrderRequest)`, `executeOrder`, `executeLiquidation`, `updateMarkPrice`, and the `PositionProtectionBook`, `CfdEngine` and `MarginClearinghouse` entrypoints). Risk parameters are the release values from `packages/perps/DEPLOYMENT.md`. Each test prints every step and the values that prove the impact (`-vv`).

### What the tests show

- `test_poc_ownFillBlocksCreatePositionProtection` — open through the Router, then `createPositionProtection` reverts with `PriceOutOfOrder(acceptedTime, markTime)`; the identical call succeeds after a separate live refresh.
- `test_poc_thirdPartyFillBlocksReplacePositionProtection` — the victim has an armed protection; an unrelated trader's order is executed; the victim's `replacePositionProtection` reverts; it succeeds after a refresh.
- `test_poc_anyFillBlocksCommitOpenOrderWithProtection` — after any execution, `commitOpenOrderWithProtection` reverts; the identical request succeeds after a refresh.
- `test_poc_failedExecutionAlsoBlocksCreatePositionProtection` — a third party's open with an unreachable target price fails (no fill), yet the Engine mark advances and the victim's `createPositionProtection` reverts.
- `test_poc_staleAcceptedBasketBlocksCreateDespiteFreshEngineMark` — the same state surfacing as `StalePrice`: executions keep the Engine mark 6 s old while the accepted basket is 62 s old.
- `test_poc_emptyReplayCannotRepairTheState` — `router.updateMarkPrice` with an empty payload reverts with `PriceOutOfOrder`, so only freshly signed Pyth data in a separate transaction repairs the state.

### Test output (`-vv`)

```
[PASS] test_poc_anyFillBlocksCommitOpenOrderWithProtection() (gas: 7365207)
Logs:
  STEP 1: an unrelated trader's order is executed through the Router
    oracle accepted live basket publish time: 1709535601
    Engine mark time                       : 1709535602
  STEP 2: victim calls commitOpenOrderWithProtection -> expected PriceOutOfOrder
    -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)
  STEP 3 (control): a separate live refresh lands, the identical request now succeeds
    -> parent order id: 2
    -> attached protection id: 1
[PASS] test_poc_emptyReplayCannotRepairTheState() (gas: 3770753)
Logs:
  STEP 1: victim opens a LONG through the Router
    oracle accepted live basket publish time: 1709535601
    Engine mark time                       : 1709535602
  STEP 2: anyone calls router.updateMarkPrice with an empty payload (replay) -> expected PriceOutOfOrder
    -> reverted: PletherOracle__PriceOutOfOrder
[PASS] test_poc_failedExecutionAlsoBlocksCreatePositionProtection() (gas: 6758678)
Logs:
  STEP 1: victim opens a LONG and a live refresh synchronizes the oracle and the Engine
    oracle accepted live basket publish time: 1709535603
    Engine mark time                       : 1709535603
  STEP 2: a third party commits a LONG open with target 2.00 (unfillable at 1.00); a keeper executes it
    -> third party position size after execution (0 = did not fill): 0
    oracle accepted live basket publish time: 1709535603
    Engine mark time                       : 1709535604
  STEP 3: victim calls createPositionProtection -> expected PriceOutOfOrder
    -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)
[PASS] test_poc_ownFillBlocksCreatePositionProtection() (gas: 5158645)
Logs:
  STEP 1: victim opens a $100k LONG through the Router (commit, then keeper execute)
    oracle accepted live basket publish time: 1709535601
    Engine mark time                       : 1709535602
  STEP 2: victim calls createPositionProtection (TP 0.90 / SL 1.10) -> expected PriceOutOfOrder
    -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)
  STEP 3 (control): a separate live refresh lands, the identical call now succeeds
    oracle accepted live basket publish time: 1709535603
    Engine mark time                       : 1709535603
    -> protection created, id: 1
[PASS] test_poc_staleAcceptedBasketBlocksCreateDespiteFreshEngineMark() (gas: 7354318)
Logs:
  STEP 1: victim opens a LONG; 55 s later an unrelated order is executed (Engine mark stays fresh)
    Engine mark age (s): 6
    accepted live basket age (s): 62
  STEP 2: victim calls createPositionProtection -> expected StalePrice
    -> reverted: PletherOracle__StalePrice(MarkRefresh, feed, acceptedTime, 60, now)
[PASS] test_poc_thirdPartyFillBlocksReplacePositionProtection() (gas: 8667788)
Logs:
  STEP 1: victim opens a LONG, a live refresh lands, victim arms TP/SL protection
    -> protection armed, id: 1
  STEP 2: an unrelated trader's order is executed through the permissionless keeper path
    oracle accepted live basket publish time: 1709535603
    Engine mark time                       : 1709535604
  STEP 3: victim tries to tighten the stop-loss from 1.10 to 1.02 -> expected PriceOutOfOrder
    -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)
  STEP 4 (control): a separate live refresh lands, the identical replace now succeeds
    -> stop-loss replaced
Suite result: ok. 6 passed; 0 failed; 0 skipped;
```

### `packages/perps/test/perps/poc/PocBase.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

/// @notice Shared PoC base for the Immunefi Plether audit competition.
/// @dev Built on the repository's own `BasePerpTest` fixture (real CfdEngine, MarginClearinghouse, HousePool,
///      TrancheVaults, PletherOracle, OrderRouter stack). Only out-of-scope components are mocked: Pyth (`MockPyth`)
///      and USDG (`MockSettlementToken`). The fixture deploys the router as `LegacyOrderRouterHarness`, a test utility
///      that inherits the production `OrderRouter` unchanged; every call in these PoCs uses production selectors
///      (`commitOrder(OrderRequest)`, `executeOrder`, `executeLiquidation`, `updateMarkPrice`).
///      Risk parameters are the release values from packages/perps/DEPLOYMENT.md.
abstract contract PocBase is BasePerpTest {

    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal keeper = makeAddr("keeper");

    uint256 private _pocClientOrderNonce;

    /// @dev Release parameters (DEPLOYMENT.md "Initial release parameters").
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

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with a market target price.
    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        // Market order: closes of LONG and opens of SHORT accept any price at or below the target, so use the maximum;
        // the other two cases accept any price at or above the target, so use 1 (OrderValidationLib.checkSlippage).
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        orderId = _commitWithTarget(trader, side, size, margin, target, isClose);
    }

    /// @dev Commits through the production `OrderRouter.commitOrder(OrderRequest)` with an explicit target price.
    function _commitWithTarget(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _orderRequest(side, size, margin, targetPrice, isClose);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @dev A fully bounded V3 request with permissive financial limits, as an integrator would build it.
    function _orderRequest(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("plether-poc", ++_pocClientOrderNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
                executionWindowSeconds: uint32(window),
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
            })
        });
    }

    /// @dev A keeper executes `orderId` through the production Router with the first post-commit signed Pyth tick.
    function _execute(
        uint64 orderId,
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.executeOrder(orderId, updateData);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}

/// @notice Sanity check that the base drives a full open/close through the production Router.
contract PocSanityTest is PocBase {

    function test_poc_sanity_open_close_via_router() public {
        _fundTrader(attacker, 10_000e6);

        uint64 openId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 1_000e6, false);
        _execute(openId, 1e8);
        assertEq(_positionSize(attacker), 100_000e18, "open should fill");

        uint64 closeId = _commit(attacker, CfdTypes.Side.LONG, 100_000e18, 0, true);
        _execute(closeId, 1e8);
        assertEq(_positionSize(attacker), 0, "close should fill");
    }

}
```

### `packages/perps/test/perps/poc/ProtectionBlockedAfterFillPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC: after an order execution, TP/SL protection cannot be created, replaced or attached until a separate
///         live mark refresh lands.
/// @dev Root cause (introduced in eadc5c5 / #124): PositionProtectionBook._freshProtectionObservation() calls
///      PletherOracle.getLatestPrice(PriceMode.MarkRefresh). That view reads the oracle's *accepted live* basket and
///      reverts with PriceOutOfOrder when it is older than Engine.lastMarkTime(). Order execution installs a newer
///      historical Pyth tick as the Engine mark (OrderRouterExecutionSidecar._updateEngineMark) but never writes the
///      accepted live basket. The Book's own _acceptedMarkInstalled() treats `acceptedTime < markTime` as valid.
///      Every test logs the accepted-basket time, the Engine mark time and the outcome of each step.
contract ProtectionBlockedAfterFillPoC is PocBase {

    IPositionProtectionActions internal protection;

    uint256 internal constant MARK = 1e8;
    uint256 internal constant SIZE = 100_000e18; // $100k notional at 1.00
    uint256 internal constant MARGIN = 1_000e6;
    // LONG (long USD) profits when the basket falls: take-profit below the mark, stop-loss above it.
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;

    function setUp() public override {
        super.setUp();
        protection = IPositionProtectionActions(address(router.positionProtectionBook()));
        _fundTrader(victim, 20_000e6);
        _fundTrader(attacker, 20_000e6);
        _liveRefresh(MARK);
    }

    /// Scenario 1: the most common user flow. Open through the Router, then add a stop-loss. It reverts.
    function test_poc_ownFillBlocksCreatePositionProtection() public {
        emit log("STEP 1: victim opens a $100k LONG through the Router (commit, then keeper execute)");
        _routerOpen(victim);
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        _logSync();
        assertLt(acceptedTime, markTime, "fill moved the Engine mark past the accepted live basket");

        emit log("STEP 2: victim calls createPositionProtection (TP 0.90 / SL 1.10) -> expected PriceOutOfOrder");
        vm.prank(victim);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log("  -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)");

        emit log("STEP 3 (control): a separate live refresh lands, the identical call now succeeds");
        _liveRefresh(MARK);
        _logSync();
        vm.prank(victim);
        uint64 protectionId = protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log_named_uint("  -> protection created, id", protectionId);
        assertGt(protectionId, 0, "same request succeeds once the accepted basket catches up");
    }

    /// Scenario 2: any third party's execution blocks the victim from tightening an existing stop-loss.
    function test_poc_thirdPartyFillBlocksReplacePositionProtection() public {
        emit log("STEP 1: victim opens a LONG, a live refresh lands, victim arms TP/SL protection");
        _routerOpen(victim);
        _liveRefresh(MARK);
        vm.prank(victim);
        uint64 protectionId = protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log_named_uint("  -> protection armed, id", protectionId);

        emit log("STEP 2: an unrelated trader's order is executed through the permissionless keeper path");
        _routerOpen(attacker);
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        _logSync();
        assertLt(acceptedTime, markTime, "third-party fill moved the Engine mark past the accepted basket");

        emit log("STEP 3: victim tries to tighten the stop-loss from 1.10 to 1.02 -> expected PriceOutOfOrder");
        vm.prank(victim);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protection.replacePositionProtection(protectionId, _params(LONG_TAKE_PROFIT, 102_000_000));
        emit log("  -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)");

        emit log("STEP 4 (control): a separate live refresh lands, the identical replace now succeeds");
        _liveRefresh(MARK);
        vm.prank(victim);
        protection.replacePositionProtection(protectionId, _params(LONG_TAKE_PROFIT, 102_000_000));
        emit log("  -> stop-loss replaced");
    }

    /// Scenario 3: after any execution, a trader cannot open with TP/SL attached.
    function test_poc_anyFillBlocksCommitOpenOrderWithProtection() public {
        emit log("STEP 1: an unrelated trader's order is executed through the Router");
        _routerOpen(attacker);
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        _logSync();
        assertLt(acceptedTime, markTime, "fill moved the Engine mark past the accepted live basket");

        emit log("STEP 2: victim calls commitOpenOrderWithProtection -> expected PriceOutOfOrder");
        OrderV3Types.OrderRequest memory request = _orderRequest(CfdTypes.Side.LONG, SIZE, MARGIN, 1, false);
        vm.prank(victim);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protection.commitOpenOrderWithProtection(request, _params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log("  -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)");

        emit log("STEP 3 (control): a separate live refresh lands, the identical request now succeeds");
        _liveRefresh(MARK);
        vm.prank(victim);
        (uint64 parentOrderId, uint64 protectionId) =
            protection.commitOpenOrderWithProtection(request, _params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log_named_uint("  -> parent order id", parentOrderId);
        emit log_named_uint("  -> attached protection id", protectionId);
        assertGt(parentOrderId, 0, "parent open queued after the refresh");
        assertGt(protectionId, 0, "attached protection staged after the refresh");
    }

    /// Scenario 4: the state does not require a fill. An execution that FAILS terminally (a third party's open whose
    /// target price cannot be met) still installs the newer historical tick as the Engine mark, because the Router
    /// updates the mark before the per-order rollback frame.
    function test_poc_failedExecutionAlsoBlocksCreatePositionProtection() public {
        emit log("STEP 1: victim opens a LONG and a live refresh synchronizes the oracle and the Engine");
        _routerOpen(victim);
        _liveRefresh(MARK);
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        _logSync();
        assertEq(acceptedTime, engine.lastMarkTime(), "synchronized after the live refresh");

        emit log("STEP 2: a third party commits a LONG open with target 2.00 (unfillable at 1.00); a keeper executes it");
        uint64 orderId = _commitWithTarget(attacker, CfdTypes.Side.LONG, SIZE, MARGIN, 200_000_000, false);
        _execute(orderId, MARK);
        emit log_named_uint("  -> third party position size after execution (0 = did not fill)", _positionSize(attacker));
        assertEq(_positionSize(attacker), 0, "the order did not fill");
        uint64 markTime = engine.lastMarkTime();
        _logSync();
        assertLt(acceptedTime, markTime, "the failed execution still advanced the Engine mark");

        emit log("STEP 3: victim calls createPositionProtection -> expected PriceOutOfOrder");
        vm.prank(victim);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log("  -> reverted: PletherOracle__PriceOutOfOrder(acceptedTime, markTime)");
    }

    /// Scenario 5: the same state surfacing as StalePrice. Executions keep the Engine mark fresh, but the accepted live
    /// basket is older than the 60 s MarkRefresh limit. Before eadc5c5 only Engine-mark freshness was required.
    function test_poc_staleAcceptedBasketBlocksCreateDespiteFreshEngineMark() public {
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        emit log("STEP 1: victim opens a LONG; 55 s later an unrelated order is executed (Engine mark stays fresh)");
        _routerOpen(victim);
        vm.warp(uint256(acceptedTime) + 55);
        _routerOpen(attacker);
        vm.warp(uint256(acceptedTime) + 62);
        emit log_named_uint("  Engine mark age (s)", block.timestamp - uint256(engine.lastMarkTime()));
        emit log_named_uint("  accepted live basket age (s)", block.timestamp - uint256(acceptedTime));
        assertLe(block.timestamp - uint256(engine.lastMarkTime()), 60, "Engine mark is fresh");
        assertGt(block.timestamp - uint256(acceptedTime), 60, "accepted live basket is older than 60 s");

        emit log("STEP 2: victim calls createPositionProtection -> expected StalePrice");
        // Build the expected revert before pranking: external view calls would otherwise consume the prank.
        bytes memory expectedRevert = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StalePrice.selector,
            IPletherOracle.PriceMode.MarkRefresh,
            _basePythFeedIds()[0],
            uint256(acceptedTime),
            pletherOracle.orderExecutionStalenessLimit(),
            block.timestamp
        );
        vm.prank(victim);
        vm.expectRevert(expectedRevert);
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        emit log("  -> reverted: PletherOracle__StalePrice(MarkRefresh, feed, acceptedTime, 60, now)");
    }

    /// Scenario 6: the cheap workaround does not exist. Replaying the accepted basket with an empty payload reverts
    /// the same way, so the trader needs freshly signed Pyth data in a separate transaction.
    function test_poc_emptyReplayCannotRepairTheState() public {
        emit log("STEP 1: victim opens a LONG through the Router");
        _routerOpen(victim);
        _logSync();
        emit log("STEP 2: anyone calls router.updateMarkPrice with an empty payload (replay) -> expected PriceOutOfOrder");
        try router.updateMarkPrice(new bytes[](0)) {
            fail("empty replay unexpectedly succeeded");
        } catch (bytes memory err) {
            assertEq(bytes4(err), IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "replay is out of order");
            emit log("  -> reverted: PletherOracle__PriceOutOfOrder");
        }
    }

    // --- helpers ---

    function _logSync() internal {
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        emit log_named_uint("  oracle accepted live basket publish time", acceptedTime);
        emit log_named_uint("  Engine mark time                       ", engine.lastMarkTime());
    }

    function _routerOpen(
        address trader
    ) internal {
        uint64 orderId = _commit(trader, CfdTypes.Side.LONG, SIZE, MARGIN, false);
        _execute(orderId, MARK);
        assertEq(_positionSize(trader), SIZE, "open filled through the real Router");
    }

    /// @dev Live signed basket refresh through the Router (same encoding as PositionProtection.t.sol).
    function _liveRefresh(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
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

}
```
