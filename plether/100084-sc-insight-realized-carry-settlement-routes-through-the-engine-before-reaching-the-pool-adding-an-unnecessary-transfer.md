# \#100084 [SC-Insight] Realized carry settlement routes through the engine before reaching the pool, adding an unnecessary transfer

**Submitted on Nov 16th 2026 at 08:00:16 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100084
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

## Summary

`CfdEngine._realizeCarryFromSettlement` (`CfdEngine.sol:1407-1448`) routes every nonzero realized-carry payment
through the engine itself before forwarding it on to the pool, instead of paying the pool directly in one transfer.

At `CfdEngine.sol:1421-1422`, the engine asks the clearinghouse to settle due carry with the engine named as the
recipient: `clearinghouse.consumeSettlementLoss(account, carryDueAssets, address(this))`. `consumeSettlementLoss`
(`MarginClearinghouse.sol:1115-1150`) takes `recipient` as a plain parameter with no restriction on its value (only
the caller is gated, by `onlyOperator`, at `:1121`); it transfers the collected settlement token straight to
whichever `recipient` it is given (`:1148`) and emits `AssetSeized(account, settlementAsset, totalConsumedAssets,
recipient)` (`:1149`). Passing `address(this)` lands the tokens in the engine's own balance.

Two lines later, at `CfdEngine.sol:1432`, the engine forwards that same amount on to the pool:
`settlementToken.safeTransfer(address(pool), realizedCarryAssets)`, then calls `pool.recordClaimantInflow(
realizedCarryAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived)`
(`:1433-1437`). Nothing between the two transfers reads or otherwise uses the engine's token balance: it goes from
zero to `realizedCarryAssets` and back to zero inside this one call, on every realization where carry is actually
due.

The indirection is unnecessary. `consumeSettlementLoss`'s `recipient` parameter already accepts any address, and
the same engine's own settlement sidecar already calls the clearinghouse with the pool passed directly as the
recipient for the analogous pledge- and rebate-reserve collections: `clearinghouse.consumePnlPledgeLoss(
delta.account, delta.pricePnlPledgeConsumedAssets, host.pool())` (`CfdEngineSettlementSidecar.sol:580`) and
`clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets)` (`:692`).
Passing `address(pool)` at `CfdEngine.sol:1421-1422` instead of `address(this)`, and deleting the now-redundant
`safeTransfer` at `:1432`, removes one ERC-20 external call, the engine's two balance-slot writes (one to receive
the tokens, one to empty back out), and one `Transfer` log, on every nonzero carry realization, while leaving
`pool.recordClaimantInflow` untouched.

`recordClaimantInflow` (`HousePool.sol:624-650`) is accounting-safe under this change: called with
`ClaimantInflowCashMode.CashArrived`, it increments `accountedAssets` by the `amount` parameter it is given
(`:640-641`); it never reads the caller's or its own token balance to decide how much arrived. Redirecting where the
tokens physically land does not change what this call records.

`_realizeCarryFromSettlement` sits on every state-changing path that can owe carry: `addMargin`,
`realizeCarryBeforeMarginChange`, `settleTraderClaim`, `settlementRealizeCarry`, and the open/close/liquidation
settlement appliers (`CfdEngine.sol:215, 518, 548, 569, 1162, 1293, 1313, 1330`), so the saving recurs on ordinary
trading activity, not on a rare path.

This exact recipient argument has not changed since it was introduced in the monorepo's original package
restructuring and is unchanged on the current public default branch; none of the four pinned audit reports, the
pinned native scan, or any merged/open pull request on the public repository (including the two merged pull
requests that otherwise reworked how and when this same function collects carry) names this recipient or the
redundant transfer.

## References

All paths are under `packages/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `src/CfdEngine.sol`: `:85` (`pool` state variable); `:215, 518, 548, 569, 1162, 1293, 1313, 1330` (call sites of
  `_realizeCarryFromSettlement`); `:1407-1448` (`_realizeCarryFromSettlement`), specifically `:1421-1422`
  (`consumeSettlementLoss(account, carryDueAssets, address(this))`), `:1432`
  (`settlementToken.safeTransfer(address(pool), realizedCarryAssets)`), `:1433-1437`
  (`pool.recordClaimantInflow(...)`)
- `src/MarginClearinghouse.sol`: `:217` (`AssetSeized` event declaration); `:1115-1150`
  (`consumeSettlementLoss`), specifically `:1121` (`onlyOperator` caller gate, no recipient restriction),
  `:1148-1149` (`safeTransfer(recipient, totalConsumedAssets)` and the `AssetSeized` emit)
- `src/CfdEngineSettlementSidecar.sol`: `:580` (`consumePnlPledgeLoss(delta.account,
  delta.pricePnlPledgeConsumedAssets, host.pool())`); `:692` (`consumeVpiRebateReserve(delta.account, host.pool(),
  delta.vpiRebateReserveConsumedAssets)`) — the codebase's own precedent for sending the clearinghouse-collected
  amount directly to the pool
- `src/HousePool.sol`: `:624-650` (`recordClaimantInflow`), specifically `:629-631` (caller gated to the engine or
  its settlement sidecar), `:640-641` (`accountedAssets += amount`, driven by the parameter, not a balance read)


## Proof of Concept

# Proof of Concept

Conditions under which this Insight is valuable: every call that realizes nonzero carry, which is the ordinary case
for any open position once any time has passed since its last carry checkpoint. The same two-line change applies
uniformly at every call site listed above; no caller-supplied input needs to be large, adversarial, or unusual for
the saving to occur, and the saving is realized by whichever account happens to trigger the realization (the
account itself, in the trader-initiated paths).

Measured, matched comparison: built the pinned contracts themselves (`CfdEngine.sol`, `MarginClearinghouse.sol`,
`HousePool.sol`, unmodified, commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`) with the project's own compiler
settings (Solidity 0.8.35, optimizer on, 200 runs, via-IR) using Foundry (forge 1.8.3), once before
and once after making exactly the two-line change this report recommends, with nothing else different between the
two builds:

- Deployed bytecode size of `CfdEngine.sol`: 23,137 bytes before, 22,992 bytes after (145 bytes smaller; headroom
  against the 24,576-byte EIP-170 limit grows from 1,439 to 1,584 bytes).
- Gas cost of one representative call that realizes nonzero carry (`CfdEngine.addMargin` on an already-open
  position, after enough time has passed for carry to accrue -- the same operation the pinned project's own
  gas-profiling test suite already times for the pre-existing, zero-carry case), measured the same way that test
  suite measures it (a `gasleft()` delta around the call, before any end-of-transaction SSTORE-clear refund is
  netted in): 517,540 gas before, 492,512 gas after -- 25,028 gas saved per realization, about 4.8% of that one
  call. The realized carry amount itself was identical in both builds (9,072,379 settlement-token units), so the
  two runs are comparing the same underlying realization, not two different amounts.
- The observable shape of that same call, read from the emitted logs rather than estimated: before the change, the
  call emits two settlement-token `Transfer` events (clearinghouse to engine, then engine to pool) and the
  clearinghouse's `AssetSeized.recipient` for the collection is the engine's own address; after the change, the
  call emits one `Transfer` event (clearinghouse straight to pool) and `AssetSeized.recipient` is the pool's
  address. The pool's own settlement-token balance after the call -- the observable result of
  `pool.recordClaimantInflow` plus the physical transfer -- was identical in both builds, down to the unit,
  consistent with that call being computed from the `amount` argument rather than from a balance read.

This comparison ran end to end against the actual pinned `MarginClearinghouse.consumeSettlementLoss` and
`HousePool.recordClaimantInflow` functions -- not mocks or stand-ins for them -- with only the recipient argument
and the one redundant transfer line in `CfdEngine.sol` differing between the two builds. The saving recurs on every
nonzero carry realization, which includes ordinary `addMargin`, claim settlement, and close/liquidation settlement
calls, not only the sampled one.

