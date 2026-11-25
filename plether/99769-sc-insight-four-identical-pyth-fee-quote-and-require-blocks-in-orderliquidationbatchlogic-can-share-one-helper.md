# \#99769 [SC-Insight] Four identical Pyth-fee quote-and-require blocks in OrderLiquidationBatchLogic can share one helper

**Submitted on Nov 15th 2026 at 07:37:12 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99769
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

**Category: Insight - Code Optimizations and Enhancements.** No vulnerability, loss or exploit is claimed; this is a
pure bytecode-size reduction from removing already-present, verbatim code duplication.

## The duplication

`OrderLiquidationBatchLogic.sol` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol`, frozen commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`) contains the identical three-line Pyth-fee quote-and-require block,
verbatim, in four separate functions:

```solidity
uint256 pythFee = oracle.getUpdateFee(pythUpdateData);
if (msg.value < pythFee) {
    revert IPletherOracle.PletherOracle__InsufficientFee(msg.value, pythFee);
}
```

This exact block appears, byte-identical, at the start of each of these four functions (confirmed by direct read
of the pinned source): the function that settles an LP epoch and reconciles the pool, the function that refreshes
the mark price, the function that executes a single account liquidation, and the function that executes a batch
of liquidations. Each of the four compiles its own independent copy of this logic into the deployed contract.

## Why this is pure duplication, not four independent design choices

All four blocks: (a) read the same `IPletherOracle oracle` interface's `getUpdateFee` with the same argument
pattern (`pythUpdateData`), (b) compare the result against the same `msg.value`, and (c) revert with the same
`PletherOracle__InsufficientFee` error constructed from the same two values, in the same order. There is no
functional variation between the four call sites — each one is checking "did the caller pay enough native currency
to cover the Pyth update fee" before proceeding, and none of the four needs anything different from any other.

## Measured bytecode cost

Extracting all four occurrences into one shared internal function and calling it from each of the four original
call sites reduces the compiled `OrderRouterLiquidationBatchSidecar` contract's deployed runtime bytecode by
**339 bytes**, verified directly: baseline 24,546 bytes, with only this extraction applied (and no other change)
24,207 bytes, confirmed via `forge inspect OrderRouterLiquidationBatchSidecar deployedBytecode` against a fresh
build of the pinned source with solely this one refactor applied. No other contract or function's behavior is
changed - every one of the four call sites still performs the identical check and still reverts with the identical
error under the identical condition.

## Why this is worth fixing regardless of current deployment headroom

Every in-scope contract in this programme is not upgradeable once deployed, so any avoidable bytecode is a
permanent cost: it is paid once at deployment (higher deployment gas) and then persists as dead weight for the
contract's entire lifetime, contributing nothing to its behavior. Four copies of an identical nine-opcode-class
check is the textbook case Category 2 exists for - efficiency improvement that has no direct security effect but
supports lower deployment cost and smaller, more maintainable bytecode.

## Possible Remediation

Extract the four identical blocks into one shared `private view` helper (shown in full in `04_DEMONSTRATION.md`)
that takes the oracle reference and the update-data array, returns the quoted fee, and reverts internally if
`msg.value` is insufficient. Replace all four call sites with a single call to the helper. This is a pure
code-motion refactor: no control flow, revert condition, or external call changes - the helper performs exactly
the same two operations (`getUpdateFee`, then the `msg.value` comparison and revert) that each of the four sites
already performed inline. Measured deployed-bytecode reduction: 339 bytes, with the full `spec/lp` and
`spec/trader` regression suites passing identically before and after.

## References

- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` - the four call sites (LP-epoch settlement/pool
  reconciliation, mark-price refresh, single-account liquidation execution, liquidation-batch execution), frozen
  commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- `packages/perps/src/interfaces/IPletherOracle.sol` (or wherever `IPletherOracle` is declared in the pinned
  source) - `getUpdateFee`, `PletherOracle__InsufficientFee`.
- Plether Audit Competition Information tab, Insight Reporting rules and Category 2 (Code Optimizations and
  Enhancements) definition; Scope tab PoC-requirement carve-out for Insights - both re-captured 2026-11-10, within
  the 14-day re-capture gate as of this packet's build date.


## Proof of Concept

# Proof of Concept

**Conditions under which this Insight is valuable.** This is a pure source-code observation, reachable by reading
the pinned checkout; no runtime trigger, account state, or actor is involved. The value is highest now, before
deployment, since the in-scope code is not upgradeable once deployed and any reducible bytecode becomes a
permanent cost at that point.

**Source-level reproduction at the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`** (path relative to
`packages/perps/src`):

`router/OrderLiquidationBatchLogic.sol` contains this exact three-line block, verbatim, at four separate call
sites (confirmed by direct read of the pinned source, line numbers as checked out at commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`):

```solidity
uint256 pythFee = oracle.getUpdateFee(pythUpdateData);
if (msg.value < pythFee) {
    revert IPletherOracle.PletherOracle__InsufficientFee(msg.value, pythFee);
}
```

The four call sites are the function that settles an LP epoch and reconciles the pool, the function that
refreshes the mark price, the function that executes a single account liquidation, and the function that
executes a batch of liquidations - each immediately after obtaining its own `IPletherOracle oracle` reference.

**Executed confirmation.** Extracting this block into one shared internal helper,

```solidity
function _payPythFee(
    IPletherOracle oracle,
    bytes[] calldata pythUpdateData
) private view returns (uint256 pythFee) {
    pythFee = oracle.getUpdateFee(pythUpdateData);
    if (msg.value < pythFee) {
        revert IPletherOracle.PletherOracle__InsufficientFee(msg.value, pythFee);
    }
}
```

and replacing all four original call sites with `uint256 pythFee = _payPythFee(oracle, pythUpdateData);`, then
rebuilding:

```
forge build --skip test --skip script
forge inspect OrderRouterLiquidationBatchSidecar deployedBytecode
```

Baseline (unmodified pinned source): deployed runtime bytecode is **24,546 bytes**.
With only this one extraction applied (no other change to the file): deployed runtime bytecode is **24,207
bytes** - a reduction of exactly **339 bytes**, measured directly via `forge inspect`, not estimated.

**Behavior-preservation check.** The full `packages/perps/test/perps/spec/lp` (238 tests) and
`packages/perps/test/perps/spec/trader` (455 tests) regression suites were run against the extracted version:

```
forge test --match-path "test/perps/spec/lp/**/*.sol"
forge test --match-path "test/perps/spec/trader/**/*.sol"
```

Result: `238 tests passed, 0 failed, 0 skipped` and `455 tests passed, 0 failed, 0 skipped`, identical to the
unmodified baseline - confirming the extraction changes no observable behavior at any of the four call sites.



