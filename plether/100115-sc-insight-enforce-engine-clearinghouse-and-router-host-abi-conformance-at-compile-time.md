# \#100115 [SC-Insight] Enforce Engine, Clearinghouse, and Router host-ABI conformance at compile time

**Submitted on Nov 16th 2026 at 08:29:42 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100115
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Summary

Several of the protocol's most security-sensitive cross-contract and sidecar boundaries are expressed as Solidity
interfaces at call sites, but the contracts serving those calls do not inherit the corresponding interfaces. The
current frozen implementation is ABI-compatible; this report does **not** claim a present vulnerability, loss, or
exploit. The recommendation is to make that compatibility a compiler-enforced invariant so a future one-sided API
change fails during the build instead of on an order, settlement, liquidation, or protection path at runtime.

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

- `CfdEngine.sol:46` inherits `ICfdEngineTypes`, `IWithdrawGuard`, `ICfdEngineAdminHost`, `CfdTreasury`, and
  `ReentrancyGuardTransient`, but not `ICfdEngineCore`, `ICfdEngineSettlementHost`, or the two-getter
  `ICfdEnginePlanningHost` declared in `CfdEnginePlanningReader.sol:9-14`.
- `MarginClearinghouse.sol:30` inherits the slim public `IMarginAccount`, but not the 47-function
  `IMarginClearinghouse` operator/integration surface used by the engine, router, settlement sidecar, protection book,
  and lenses.
- `OrderRouter.sol:32` directly inherits `IPerpsKeeper` and `IPerpsTraderActions`; its base graph also already enforces
  `IOrderRouterAccounting` and `IOrderRouterAdminHost`. It does not inherit `IOrderRouterExecutionHost`, the actual
  19-function `IOrderLiquidationBatchHost` declared in `OrderLiquidationBatchLogic.sol:27-81`, or the three router-host
  interfaces declared in `PositionProtectionBook.sol:68-120`.

These are concrete protocol boundaries rather than unused interface files. `CfdEngineSettlementSidecar` repeatedly
casts the engine to `ICfdEngineSettlementHost`; `CfdEnginePlanningReader` binds its deploying engine as
`ICfdEnginePlanningHost`; `HousePool`, the router stack, and public lens use `ICfdEngineCore`. The execution and recovery
sidecars repeatedly bind `address(this)` as `IOrderRouterExecutionHost`, delegated liquidation logic uses
`IOrderLiquidationBatchHost`, and the protection book casts its immutable router to its three host interfaces.

Without inheritance, changing only an implementation's parameter list can remove the selector while all consumers
still compile, producing a runtime revert when that route is reached. Changing only a return schema can preserve the
selector while making a caller decode stale data incorrectly or revert. Explicit implementation-interface
conformance turns both classes of drift into a compile-time failure.

The legacy aggregate `ICfdEngine` is intentionally **not** the proposed interface: its own NatSpec says no live
contract must implement the complete surface because two preview selectors belong to the diagnostic lens. The narrow
`ICfdEngineCore`, settlement-host, and planning-host interfaces are the appropriate implementation-facing contracts.

## Current measured state

An isolated Solidity 0.8.35 build from the exact frozen tree succeeded. A read-only ABI check compared canonical input
signatures/selectors, canonical return schemas (including tuple components), and state mutability across ten relevant
interface-to-implementation pairs. All comparisons pass: zero missing functions and zero return/mutability mismatches.
This clean baseline is the right point to add enforcement without changing behavior.


## Proof of Concept

## Proof of Concept

## Conditions under which this Insight is valuable

This control is valuable when any implementation or host interface is edited, split, or rebased. Plether deploys these
components separately and binds immutable sidecars/books to them; many host callbacks are reached only in particular
order, recovery, liquidation, or protection states. A one-sided signature or return-schema edit can therefore compile
successfully and remain latent until that state is exercised. Local history shows the relevant interfaces have been
edited repeatedly, including `ICfdEngineCore` in PR #126 and `IMarginClearinghouse` /
`IOrderRouterExecutionHost` in PR #128.

The value is lower where existing integration tests happen to cover a route, and the SDK compatibility suite already
pins selected public/client ABIs. Those are useful mitigations, but they do not require each implementation to satisfy
the internal host interfaces its counterparties compile against.

## Read-only demonstration at the frozen commit

1. Confirm the pin and inspect the three implementation headers:

```bash
git rev-parse HEAD
sed -n '46p' packages/perps/src/CfdEngine.sol
sed -n '30p' packages/perps/src/MarginClearinghouse.sol
sed -n '32p' packages/perps/src/OrderRouter.sol
```

2. Inspect representative typed boundaries:

```bash
sed -n '9,26p' packages/perps/src/CfdEnginePlanningReader.sol
sed -n '7,110p' packages/perps/src/interfaces/ICfdEngineSettlementHost.sol
sed -n '8,139p' packages/perps/src/interfaces/IOrderRouterExecutionHost.sol
sed -n '27,81p' packages/perps/src/router/OrderLiquidationBatchLogic.sol
sed -n '68,120p' packages/perps/src/PositionProtectionBook.sol
```

3. Build the named production targets with Solidity 0.8.35 and compare each interface ABI with its implementation ABI,
including function inputs, outputs, tuple components, and mutability. The supplied read-only checker produced:

```text
ICfdEngineCore -> CfdEngine: functions=33 missing=0 return_or_mutability_mismatches=0
ICfdEngineSettlementHost -> CfdEngine: functions=13 missing=0 return_or_mutability_mismatches=0
ICfdEnginePlanningHost -> CfdEngine: functions=2 missing=0 return_or_mutability_mismatches=0
IMarginClearinghouse -> MarginClearinghouse: functions=47 missing=0 return_or_mutability_mismatches=0
IOrderRouterExecutionHost -> OrderRouter: functions=16 missing=0 return_or_mutability_mismatches=0
IOrderLiquidationBatchHost -> OrderRouter: functions=19 missing=0 return_or_mutability_mismatches=0
IPositionProtectionRouterHost -> OrderRouter: functions=7 missing=0 return_or_mutability_mismatches=0
IPositionProtectionOrderCommitHost -> OrderRouter: functions=2 missing=0 return_or_mutability_mismatches=0
IPositionProtectionCloseAttemptHost -> OrderRouter: functions=1 missing=0 return_or_mutability_mismatches=0
IOrderRouter -> OrderRouter: functions=19 missing=0 return_or_mutability_mismatches=0
result=PASS failures=0
```

The PASS is intentional: it proves this is a preventive compile-time control recommendation, not a disguised claim of
a current mismatch.

