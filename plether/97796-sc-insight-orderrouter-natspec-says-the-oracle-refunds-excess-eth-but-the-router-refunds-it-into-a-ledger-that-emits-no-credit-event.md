# \#97796 [SC-Insight] OrderRouter NatSpec says the oracle refunds excess ETH, but the Router refunds it into a ledger that emits no credit event

**Submitted on Nov 3rd 2026 at 08:43:42 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97796
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description


**Target:** `packages/perps/src` tree at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (OrderRouter.sol, OrderRouterAdmin.sol, router/OrderLiquidationBatchLogic.sol)
**Impact:** Insight — Documentation improvements / Security best practices

## Description

### 1. The NatSpec names the wrong refunder

`OrderRouter.updateMarkPrice` says: "The oracle handles the Pyth fee and normally refunds unused ETH to the caller" (OrderRouter.sol:424-425). `executeLiquidation` says: "The oracle handles Pyth fees and ETH refunds" (OrderRouter.sol:490).

In the code, the Router sends the oracle exactly the Pyth fee, so the oracle never sees the excess:

- The call is `oracle.updatePrice{value: pythFee}(...)` (router/OrderLiquidationBatchLogic.sol:603-608).
- The Router refunds `msg.value - pythFee` itself via `_sendEth` (router/OrderLiquidationBatchLogic.sol:621, :846-858).
- If that 30,000-gas call fails, `_sendEth` credits the amount to `OrderRouterAdmin.claimableEth` (also router/OrderExecutionHandler.sol:30-43).

### 2. The Router's deferral ledger is silent

- `OrderRouterAdmin.creditClaimableEth` increases `claimableEth[beneficiary]` and emits nothing (OrderRouterAdmin.sol:264-275).
- `claimBalance()` emits nothing either (OrderRouterAdmin.sol:279-281, :293-306).
- Only `claimBalanceTo` emits an event, `EthRefundClaimedTo` (OrderRouterAdmin.sol:290).
- The oracle's ledger, by contrast, emits `EthRefundDeferred` when it credits and `EthRefundClaimed` / `EthRefundClaimedTo` when it pays out (interfaces/IPletherOracle.sol:231-240).

## Impact

Consider a keeper whose fallback reverts or uses more than 30,000 gas. It follows the NatSpec and looks for its deferred refund in `PletherOracle.claimableEth`, or watches for `EthRefundDeferred`. Neither ever shows the refund. The ETH sits in `OrderRouterAdmin.claimableEth` with no event to reveal it.

Nothing is lost, because the beneficiary can still call `claimBalance()`. But the funds are hard to discover, and claims made through `claimBalance()` cannot be audited from logs.

## Recommendation

- Update OrderRouter.sol:424-425 and :490 to say that the Router pays the oracle exactly the Pyth fee and refunds the rest, crediting `OrderRouterAdmin.claimableEth` if the refund call fails.
- Emit an event in `creditClaimableEth` (for example `EthRefundDeferred(beneficiary, amount)`) and in `claimBalance()` (for example `EthRefundClaimed(msg.sender, amount)`), mirroring the oracle's ledger.

## Proof of Concept

## Proof of Concept
