# \#97794 [SC-Insight] `IPletherOracle` NatSpec contradicts the implementation: snapshot `publishTime` is the newest component time (since the MED-23 fix), and empty payloads are valid replays

**Submitted on Nov 3rd 2026 at 08:40:23 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97794
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description


**Target:** `packages/perps/src` tree at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (interfaces/IPletherOracle.sol, PletherOracle.sol)
**Impact:** Insight — Documentation improvements

## Description

Keepers and integrators code against `IPletherOracle`. Two parts of its NatSpec no longer describe what `PletherOracle` does.

### 1. `publishTime` is documented as the earliest component time; the code returns the newest

- **Interface:** `PriceSnapshot.publishTime` and `LiquidationBatchSnapshot.publishTime` are documented as the "Earliest component publish time" (interfaces/IPletherOracle.sol:31 and :53).
- **Implementation:**
  - The basket builder sets `basket.publishTime = uint64(maxPublishTime)` and stores the minimum separately as `basket.oldestPublishTime` (PletherOracle.sol:874-875).
  - `_snapshotFromBasket` copies the newest time into `snapshot.publishTime` (PletherOracle.sol:889). The liquidation snapshots do the same (PletherOracle.sol:294, :319).
- **Why it changed:** this is the behaviour the Cecuro MED-23 fix introduced ("The live basket's progress marker uses the newest component timestamp"). ORACLE_SYNCHRONIZATION.md:26 agrees ("Its publish time is the newest component time").
- **The interface was not updated:**
  - The same interface still says "Earliest" in both structs.
  - Elsewhere it describes the liquidation return value as the "newest publish time" (interfaces/IPletherOracle.sol:320).
  - So the interface contradicts itself.
- **Consequence:**
  - The router installs this value as the Engine mark time (`engine.updateMarkPrice(..., snapshot.publishTime)`, router/OrderLiquidationBatchLogic.sol:610; OrderRouterExecutionSidecar.sol:1058-1059).
  - An integrator who follows the struct NatSpec will assume every component is at least as new as `publishTime`.
  - In fact a component can be older by up to the configured divergence limit.

### 2. Payloads are documented as "Nonempty", but empty data is a valid zero-fee replay

- **Interface:** `updateLiquidationBatchPrice`, `updateLiquidationPrice`, `updatePrice(address,bytes[])` and `getUpdateFee` document `pythUpdateData` as "Nonempty Pyth update payloads" (interfaces/IPletherOracle.sol:306, :318, :331, :400).
- **Implementation:**
  - `_resolveCurrentBasket` treats `pythUpdateData.length == 0` as a replay of the accepted live basket (PletherOracle.sol:758-760).
  - `getUpdateFee` returns 0 for empty data (PletherOracle.sol:465-466).
  - The implementation's own NatSpec says "Signed Pyth payloads, or empty for replay" (PletherOracle.sol:195), and ORACLE_SYNCHRONIZATION.md:29-30 documents the empty replay.
- **Self-contradiction:** the `getUpdateFee` interface docblock contradicts itself. Its `@dev` says "Returns zero for an empty replay" (interfaces/IPletherOracle.sol:399), and its `@param` says "Nonempty".

## Impact

- No funds are at risk.
- Item 1 misdescribes a value the Engine uses as its mark time, so freshness reasoning built on the interface is wrong.
- Item 2 hides a supported zero-fee path (empty-payload replay) from keepers. It also implies that calls with empty data are invalid.

## Recommendation

- Change both `publishTime` descriptions (interfaces/IPletherOracle.sol:31, :53) to "Newest component publish time". Mention that the oldest component may lag it by up to the configured divergence limit.
- Change the four `pythUpdateData` descriptions to "Signed Pyth update payloads, or empty to replay the accepted live basket at zero fee". This matches PletherOracle.sol:195.

## Proof of Concept

## Proof of Concept
