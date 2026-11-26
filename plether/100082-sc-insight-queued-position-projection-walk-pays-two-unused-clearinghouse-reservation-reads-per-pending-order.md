# \#100082 [SC-Insight] Queued-position projection walk pays two unused clearinghouse reservation reads per pending order

**Submitted on Nov 16th 2026 at 07:44:24 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100082
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

**Category: Insight - Code Optimizations and Enhancements.** No vulnerability, loss, or exploit
is claimed; this is a per-pending-order gas-efficiency observation in an existing, frequently
called internal view.

## The redundant reads

`OrderLiquidationBatchLogic._queuedPosition` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402`,
frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`) projects where an account's live
position will sit once its own pending Router orders are accounted for. For every pending order
in its walk it calls `host.getPendingOrderView(orderId)` (`:381-382`) and then reads exactly four
values out of the result: `pending.isClose` (`:383-384`), `pending.side` (`:385,394,396`),
`pending.sizeDelta` (`:387,395,397`), and the second return value, `nextOrderId` (`:400`), which
advances the linked-list walk.

`getPendingOrderView` (`OrderRouter.sol:185-188`) delegates to
`OrderCommitHandler._getPendingOrderView` (`:68-86`), which builds the full ten-field
`IOrderRouterAccounting.PendingOrderView` struct (`interfaces/IOrderRouterAccounting.sol:48-59`).
Two of its ten fields are populated by external calls into `MarginClearinghouse`, not from the
Router's own storage:

`committedMarginAssets: clearinghouse.getOrderReservation(orderId).remainingAmountAssets` and
`executionBountyAssets: clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, orderId).amountAssets`
(`OrderCommitHandler.sol:82-84`).

`getOrderReservation` (`MarginClearinghouse.sol:1807-1811`) and `getBountyReservation` (`:1634-1639`)
each return a distinct two-storage-slot struct from the clearinghouse's own state
(`IMarginClearinghouse.sol:206-214` and `:93-98` respectively). So every call `_queuedPosition`
makes into `getPendingOrderView` pays for two external calls into a separate deployed contract and
four of that contract's storage reads, purely to populate `committedMarginAssets` and
`executionBountyAssets`. `_queuedPosition` never reads either field, and never reads
`pending.marginDelta`, `pending.targetPrice`, `pending.commitTime`, `pending.commitBlock`, or
`pending.orderId` either - confirmed by reading the complete function body, not by assumption.

## Why the two clearinghouse calls are not a design requirement here

`getPendingOrderView` is a genuinely shared, multi-consumer view, and at least two of its other
current callers do read the clearinghouse-sourced fields: `CfdEngineLens.sol:674` accumulates
both `committedMarginAssets` and `executionBountyAssets` while previewing a liquidation-batch
execution's released reservations, and `CfdOrderPolicyEvaluator.sol:34` reads
`committedMarginAssets` to adjust a locked-margin snapshot when assessing a committed order's
execution. The two clearinghouse calls are the right design for those callers. `_queuedPosition`
is different: it is a `private` helper used only to project where an account's position will sit
once its own queued orders are considered, so that a caller can validate a new close size against
it or detect an opposing live direction. Nothing in that purpose touches committed margin or
bounty custody, and the function already demonstrates this by never reading either field.

## Why this is worth fixing regardless of current deployment headroom

`_queuedPosition` is not a rarely used helper. It runs on every single order commit: its two call
sites, `_validateCloseAdmission` (`:351`) and `_validatedOpenBounty` (`:424`), are called
unconditionally from `_commitOrder` for a close and an open commit respectively, and `_commitOrder`
backs every public `commitOrder`, `commitProtectedOpen`, and `commitProtectedClose` entry point. The
cost repeats once per pending order in the walk, and an account's live pending-order count is
exactly the Router's own bounded queue: `maxPendingOrders` defaults to 5 (`OrderCommitHandler.sol:15`)
and governance can raise it up to `MAX_PENDING_ORDERS_LIMIT = 32` (`OrderRouterAdmin.sol:21,397`). So
every ordinary trader who queues more than one order before any of them execute - the Router's own
stated delayed-FIFO "MEV shield" design - pays this avoidable per-order clearinghouse-call cost on
every later commit, scaling with how many orders they currently have in flight.

## Measured result

A local Foundry harness built against the unmodified pinned source, then rebuilt against a
behavior-preserving narrow accessor that removes the two clearinghouse calls, measured full
`commitOrder` gas at account pending-queue sizes of 1, 5, and 32 (full methodology and numbers in
the Insight Demonstration field below). At queue size 32 - the governance-configurable ceiling -
the narrow accessor saves 687,170 gas out of 2,054,174, about 33.5% of that commit's total gas,
with no change to `_queuedPosition`'s observable behavior.

## References

All source paths are under `packages/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `src/router/OrderLiquidationBatchLogic.sol:367-402` - `_queuedPosition`, the queue-projection
  walk; `:351` and `:424` - its two call sites.
- `src/OrderRouter.sol:185-188` - `getPendingOrderView`, the external wrapper.
- `src/router/OrderCommitHandler.sol:68-86` - `_getPendingOrderView`, including the two
  clearinghouse calls at `:82-84`; `:15` - the default `maxPendingOrders = 5`.
- `src/OrderRouterAdmin.sol:21,397` - `MAX_PENDING_ORDERS_LIMIT = 32`, the governance ceiling.
- `src/interfaces/IOrderRouterAccounting.sol:48-59` - the `PendingOrderView` struct.
- `src/MarginClearinghouse.sol:1634-1639,1807-1811` - `getBountyReservation`,
  `getOrderReservation`.
- `src/interfaces/IMarginClearinghouse.sol:93-98,206-214` - the `BountyReservation` and
  `OrderReservation` structs.
- `src/CfdEngineLens.sol:674` and `src/CfdOrderPolicyEvaluator.sol:34` - two other current callers
  of `getPendingOrderView` confirmed to read its clearinghouse-sourced fields
  (`committedMarginAssets`/`executionBountyAssets` and `committedMarginAssets` respectively).

Plether Audit Competition Information tab, Insight Reporting rules and Category 2 (Code
Optimizations and Enhancements) definition; Scope tab PoC-requirement carve-out for Insights 


## Proof of Concept

# Proof of Concept

Conditions under which this Insight is valuable: any account that has more than one order
pending at a time, which is the ordinary case the Router's delayed-FIFO ("MEV shield") design
exists for, not an edge case. Every close or open commit after the first then pays this cost once
for every order currently pending on that account, from the default cap of 5 up to the
governance-configurable maximum of 32.

Read-only source demonstration:

1. Read `OrderLiquidationBatchLogic._queuedPosition`
   (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:367-402`); its loop body (`:380-401`)
   reads only `pending.isClose`, `pending.side`, `pending.sizeDelta`, and the returned
   `nextOrderId` from each `host.getPendingOrderView(orderId)` call.
2. Read `OrderCommitHandler._getPendingOrderView` (`:68-86`); its last two struct fields,
   `committedMarginAssets` and `executionBountyAssets` (`:82-84`), are populated by two separate
   external calls into `MarginClearinghouse` (`getOrderReservation`, `:1807-1811`;
   `getBountyReservation`, `:1634-1639`), each reading a distinct two-storage-slot struct in that
   contract (`IMarginClearinghouse.sol:206-214`, `:93-98`).
3. Read `_validateCloseAdmission` (`:351`) and `_validatedOpenBounty` (`:424`); both call
   `_queuedPosition` unconditionally, and both are themselves called unconditionally from
   `_commitOrder`, which backs every public `commitOrder`/`commitProtectedOpen`/`commitProtectedClose`
   entry point.
4. Read `OrderCommitHandler.sol:15` (`maxPendingOrders = 5`, the default) and
   `OrderRouterAdmin.sol:21,397` (`MAX_PENDING_ORDERS_LIMIT = 32`, the governance ceiling).

Measured result: a local Foundry harness built against the unmodified pinned source measured
full `commitOrder` gas for an ordinary open commit at account pending-queue sizes of 1, 5, and 32
orders - that is, 0, 4, and 31 pre-existing pending orders that `_queuedPosition` walks during the
measured commit's own validation, immediately before that commit is itself queued. A second,
behavior-preserving build replaced `_queuedPosition`'s call to `getPendingOrderView` with a narrow
accessor returning only `isClose`, `side`, `sizeDelta`, and the account-queue successor pointer,
read directly from the Router's own existing order record through a storage pointer and making no
call into `MarginClearinghouse`. The accessor preserves the exact same linked-list traversal
(advancing through the returned `nextAccountOrderId` exactly as before); a line-by-line comparison
shows every conditional branch and the loop structure of `_queuedPosition` itself are unchanged,
with only the data source for the same four values replaced.

Matched, full-commit gas measured against the pinned source, before vs. after:

- Queue size 1 (0 pre-existing orders walked): 1,166,805 before, 1,166,890 after. The loop body
  never executes at this depth, so no saving is expected; the 85-gas difference is noise from
  adding a new external function to the contract's dispatch table, not from any change inside
  `_queuedPosition`'s walk.
- Queue size 5 (4 pre-existing orders walked): 1,214,311 before, 1,125,786 after - 88,525 gas
  saved, about 7.3% of that commit's total gas.
- Queue size 32 (31 pre-existing orders walked, the governance ceiling): 2,054,174 before,
  1,367,004 after - 687,170 gas saved, about 33.5% of that commit's total gas.

The savings per additional walked pending order are consistent across both measured intervals -
about 22,131 gas per order between queue sizes 1 and 5, and about 22,171 gas per order between
queue sizes 5 and 32 - which is what a fixed per-iteration cost removed from the walk should
produce. All three measured commits in both the before and after builds completed successfully
with the account's pending-order count and queue linkage exactly as expected at each depth,
confirming the walk still terminates and still finds every entry after the change.

Together, the source reading and the matched measurement establish that every commit after the
first pays an avoidable, queue-depth-scaling cost for two clearinghouse struct reads whose results
`_queuedPosition` discards, and that a narrow accessor removes essentially all of that cost while
leaving the function's observable behavior identical.

