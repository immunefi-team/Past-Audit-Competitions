# \#99154 [SC-Insight] HousePool.cancelPoolConfigProposal does not emit an event, breaking the consistent governance-event pattern used by all other admin cancel functions

**Submitted on Nov 12th 2026 at 06:57:15 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99154
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

`HousePool.cancelPoolConfigProposal` deletes the pending pool config and zeros the activation time without emitting an event, while every other governance cancel function in the protocol emits a dedicated `*Cancelled()` event. Governance monitoring that tracks the propose-finalize-cancel lifecycle therefore has a blind spot on the pool config, the most LP-relevant config surface.

The function body (`HousePool.sol:370-373`) is `delete pendingPoolConfig; poolConfigActivationTime = 0;` and nothing else. `IHousePool` declares `PoolConfigProposed` and `PoolConfigFinalized` but no `PoolConfigCancelled`. The five sibling cancel functions all emit: `RiskConfigCancelled` (`CfdEngineAdmin.sol:68`, emitted at `:143`), `CalendarConfigCancelled` (`:77`, emitted at `:186`), `FreshnessConfigCancelled` (`:86`, emitted at `:231`), `RouterConfigCancelled` (`OrderRouterAdmin.sol:91`, emitted at `:174`), and `OracleConfigCancelled` (`:100`, emitted at `:208`).

Pool config controls `seniorRateBps`, `maxSeniorExposureAssets`, `maxSeniorShareBps`, `seniorFrozenLpFeeBps`, `juniorFrozenLpFeeBps` and `markStalenessLimit`. An off-chain monitor can observe proposals and finalizations but cannot observe cancellations of this specific config family, so it cannot distinguish a still-pending proposal from a silently withdrawn one without polling storage. This is a monitoring asymmetry on the config surface that matters most to LPs, and it is the only inconsistency in an otherwise uniform event pattern.

Recommendation: add `PoolConfigCancelled()` to `IHousePool` and emit it in `cancelPoolConfigProposal`, matching the five existing cancel events.

## Proof of Concept

This is an event-coverage insight, so the reproduction is a comparison of observable behaviour rather than an exploit.

Step 1. Propose a pool config through the normal path; PoolConfigProposed is emitted and poolConfigActivationTime is set.

Step 2. Call cancelPoolConfigProposal() as the owner. The call succeeds: pendingPoolConfig is deleted and poolConfigActivationTime is zeroed. No log is emitted from the transaction. This can be confirmed by inspecting the receipt logs, which contain no PoolConfigCancelled topic, and by the absence of such an event in IHousePool (a search for PoolConfig returns only PoolConfigProposed and PoolConfigFinalized).

Step 3. Repeat the same sequence against any sibling admin contract, for example CfdEngineAdmin.cancelRiskConfigProposal(). The receipt contains RiskConfigCancelled(). The same holds for cancelCalendarConfigProposal (CalendarConfigCancelled), cancelFreshnessConfigProposal (FreshnessConfigCancelled), cancelRouterConfigProposal (RouterConfigCancelled) and cancelOracleConfigProposal (OracleConfigCancelled).

Step 4. The observable consequence: a monitor subscribed to governance events sees five cancel families and not the sixth, so a cancelled pool-config proposal is indistinguishable from a still-pending one unless the monitor polls storage on every block.

Suggested fix verification: after adding and emitting PoolConfigCancelled(), the step 2 receipt should contain exactly one such log, matching the sibling pattern.

Minimal reproducer sketch: deploy HousePool, call proposePoolConfig with any valid config, then cancelPoolConfigProposal, and assert that no log with the PoolConfigCancelled signature exists in the receipt. In Foundry this is vm.recordLogs() before the cancel and a check that no entry has the expected topic0.
