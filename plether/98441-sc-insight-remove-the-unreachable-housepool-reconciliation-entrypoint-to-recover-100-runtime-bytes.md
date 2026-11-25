# \#98441 [SC-Insight] Remove the unreachable HousePool reconciliation entrypoint to recover 100 runtime bytes

**Submitted on Nov 8th 2026 at 03:40:56 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98441
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

## Description

### Summary

`HousePool.reconcile()` is an external `onlyVault` wrapper around the live internal `_reconcile(...)` mechanism, but neither configured vault implementation nor any other contract under `packages/perps/src` calls that external entrypoint. Removing the wrapper and its `IHousePool` declaration preserves the production source call graph while reducing HousePool runtime from 24,571 to 24,471 bytes: exactly 100 bytes.

This is a measurable resource/deployability optimization, not a vulnerability claim. HousePool starts only 5 bytes below the 24,576-byte EIP-170 runtime limit. The removal expands headroom to 105 bytes; the 100 recovered bytes are twenty times the contract's existing headroom.

### Frozen and release identity

The competition is frozen at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Its `packages/perps/src` tree is byte-identical to the v2.0.0 release source commit `1209ca8a5b94a4fb6323c7c4585846ed93db3932` (both resolve to tree `fff196b1b2a5bf6e9bfdc94fdb123c7854fe1252`). The release manifest identifies HousePool at `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` and the release build as Forge 1.5.1 / Solidity 0.8.35 / optimizer 200 / via-IR / Prague.

A retained Prague compilation of `ArbitrumSepoliaReleaseHousePool` from the frozen/release-identical source is 24,571 runtime bytes. Rebuilding after only the following two removals produces a 24,471-byte HousePool runtime under the same Solidity version, optimizer settings, via-IR setting, and Prague EVM target:

1. `HousePool.reconcile()` and its NatSpec; and
2. `IHousePool.reconcile()` and its NatSpec.

The 57-source compiler inputs have the same 55 unaffected source hashes; only `HousePool.sol` and `IHousePool.sol` differ.

### Why the external wrapper is unused

At frozen `HousePool.sol:1471-1473`, the entire wrapper is:

```solidity
function reconcile() external onlyVault {
    _reconcile(_getHousePoolInputSnapshot());
}
```

`onlyVault` accepts only the configured Senior or Junior vault. A complete Solidity call-expression search under `packages/perps/src` finds zero `.reconcile(...)` calls. `TrancheVault.sol` contains no call to it and exposes no generic execute/delegatecall route that could originate one.

The internal mechanism is not dead. Other state-changing HousePool paths call `_reconcile(...)` directly, including pool-config finalization, ownerless-asset assignment, seed initialization, and LP-epoch settlement. Preview functions project the reconcile result without invoking the external wrapper. The recommended change removes only an unused external ABI/dispatcher path, not the reconciliation accounting.

The test suite's use of the entrypoint does not establish production reachability: the frozen tests contain 160 direct `pool.reconcile()` calls across 29 files, each preceded by `vm.prank(...)`, across both positive and negative authorization cases. Those tests deliberately construct caller contexts that the production vault implementation does not originate.

### Conditions under which this Insight is valuable

1. **A HousePool fix or feature needs more than 5 runtime bytes.** The current runtime has only 5 bytes of EIP-170 headroom. Removing this unused surface supplies 100 bytes and raises the margin to 105 bytes without removing a production caller.
2. **Deployment resource use matters.** The runtime delta is measurable rather than cosmetic: 100 fewer deposited runtime bytes also removes 20,000 gas of EVM code-deposit cost at 200 gas per byte.
3. **Maintainers or integrators reason from the exposed ABI.** `SECURITY.md` presents `HousePool.reconcile()` as a configured-vault integration hook and the interface describes its accounting semantics, but the production vault implementation has no caller. Removing the entrypoint makes the ABI and integration documentation match the implemented architecture.
4. **Audit surface matters.** Keeping an authorized external wrapper requires reviewers to reason about a selector and privilege boundary that the production system never uses.

This is not the invalid Insight example of dead-code cleanup with no gas effect: the retained compile changes deployed runtime by exactly 100 bytes in a contract only 5 bytes below the hard limit.

### Knownness and nearest prior items

The listed SC Audit Studio report contains `SC-I1: Remove Dead Code from HousePool`, but it identifies two different private helpers (`_distributeRevenue` and `_absorbLoss`) plus the `Reconciled` event. Fix commit `8022a4429cfcc8807d74176f030770007585940a` removes those exact objects; the external `reconcile()` wrapper exists both before and after that fix. M-10 therefore has a different code path and measured runtime contribution rather than re-reporting SC-I1.

Commit `4fd80d4711e4249ea921ec4c9ea9e4bba668202e` is also adjacent: it changed this wrapper from public to `onlyVault` as defense-in-depth. It did not disclose that neither configured vault implementation calls it, did not recommend removal, and did not measure its 100-byte contribution.

### Recommendation

Remove `HousePool.reconcile()` and the corresponding `IHousePool.reconcile()` declaration, update `SECURITY.md:318`, and update tests to exercise the live state-changing HousePool paths or an explicit internal harness rather than preserving a production-dead selector.

## Proof of Concept

## Proof of Concept / Text Demonstration

### Conditions of value

This Insight is valuable when HousePool needs any runtime addition larger than its current 5-byte EIP-170 margin, when deployment code-deposit cost matters, or when maintainers/integrators rely on the documented external ABI. It does not claim loss of funds or an exploitable actor path.

### Reproduction from the frozen source

1. Check out `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Search `packages/perps/src` for `.reconcile(`. There is no production call expression. Inspect `TrancheVault.sol`; neither configured vault calls the wrapper.
3. Inspect `HousePool.sol:1465-1473`: the external function only invokes `_reconcile(_getHousePoolInputSnapshot())` and is restricted by `onlyVault`.
4. Confirm internal `_reconcile(...)` remains used by state-changing HousePool paths, including `finalizePoolConfig`, `assignUnassignedAssets`, seed initialization, and `_settleLpEpoch`.
5. Compile the unmodified release-compatible source with Solidity 0.8.35, optimizer 200, via-IR, and Prague. The retained baseline runtime is 24,571 bytes.
6. Remove only the external wrapper/NatSpec from `HousePool.sol` and the matching declaration/NatSpec from `IHousePool.sol`; then rebuild source-only with the same settings. HousePool runtime is 24,471 bytes.
7. Recompute the bounds:

```text
EIP-170 limit                  24,576 bytes
baseline runtime              24,571 bytes
baseline headroom                  5 bytes
successor runtime             24,471 bytes
successor headroom               105 bytes
runtime bytes recovered          100 bytes
recovered / old headroom      100 / 5 = 20x
code-deposit gas avoided      100 * 200 = 20,000 gas
```

The source-only build is sufficient for this Insight because the claim is the compiler-measured resource effect of deleting one unused ABI surface, not a runtime exploit. No public-chain transaction, simulation, unit-test PoC, or target interaction is needed or relied upon.

