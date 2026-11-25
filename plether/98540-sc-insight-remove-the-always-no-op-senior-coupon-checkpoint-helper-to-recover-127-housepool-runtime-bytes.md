# \#98540 [SC-Insight] Remove the always-no-op senior-coupon checkpoint helper to recover 127 HousePool runtime bytes

**Submitted on Nov 8th 2026 at 14:57:56 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98540
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**

## Description

## Description

### Summary

`HousePool._checkpointSeniorCouponBeforePrincipalMutation()` has one frozen-source caller,
`initializeSeedPosition()`. The call is reached, but the caller always runs `_reconcile(...)`
first. A successful `_reconcile(...)` leaves `lastSeniorCouponCheckpointTime` at or ahead of
the current transaction timestamp. The later helper therefore always computes zero elapsed
time and returns at its first branch; only its state-changing branches are unreachable.

Removing only the helper call and the helper reduced the retained HousePool runtime from
24,571 to 24,444 bytes under the pinned compiler settings: an isolated 127-byte reduction.
HousePool starts only 5 bytes below the 24,576-byte EIP-170 limit; the change raises runtime
headroom to 132 bytes.

This is a measurable deployability and resource-use optimization. It is not a vulnerability,
funds-at-risk, runtime-gas, or production-incident claim.

### Why the only call is always a no-op

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, seed initialization performs:

```solidity
_reconcile(_getHousePoolInputSnapshot());

uint256 shares = ITrancheVaultBootstrap(targetVault).quoteBootstrapDeposit(amount);
if (shares == 0) {
    revert HousePool__BootstrapSharesZero();
}

_checkpointEngineCarryIndexes();
_checkpointSeniorCouponBeforePrincipalMutation();
```

`_reconcile(...)` computes:

```solidity
uint256 couponElapsed =
    block.timestamp > lastSeniorCouponCheckpointTime
        ? block.timestamp - lastSeniorCouponCheckpointTime
        : 0;
```

If that value is positive, `_reconcile(...)` assigns
`lastSeniorCouponCheckpointTime = block.timestamp`. If it is zero, the checkpoint was already
greater than or equal to that same timestamp. A revert cannot reach the later helper.

Between `_reconcile(...)` and the subject helper, `quoteBootstrapDeposit(amount)` is declared
`external view`, so Solidity executes it as a static call: neither the vault nor a callback can
write `lastSeniorCouponCheckpointTime` or any other state. Its zero-share branch reverts instead
of reaching the subject helper. The subsequent `_checkpointEngineCarryIndexes()` advances
engine carry state and reads the pool through a view call; it does not write the HousePool
coupon checkpoint. `block.timestamp` cannot change within the transaction. The later helper
consequently recomputes zero and returns here:

```solidity
if (couponElapsed == 0) {
    return;
}
```

Its principal, junior-supply, coupon-payment, waterfall, and checkpoint writes are therefore
unreachable at its only frozen caller. The live coupon accounting in `_reconcile(...)` remains.

### Isolated compiler measurement

The measurement copy restored the separately filed M-10 external `HousePool.reconcile()`
wrapper and its `IHousePool` declaration to exact frozen bytes before removing only:

1. the I-17 helper call; and
2. `_checkpointSeniorCouponBeforePrincipalMutation()`.

The complete tracked change was `HousePool.sol | 23 deletions`. One retained source-only
build used Forge 1.5.1, Solidity 0.8.35, optimizer 200, via-IR, and Prague. It produced:

```text
EIP-170 runtime limit           24,576 bytes
frozen HousePool runtime        24,571 bytes
frozen headroom                      5 bytes
I-17-only runtime               24,444 bytes
I-17-only headroom                 132 bytes
runtime bytes recovered            127 bytes
code-deposit gas component      25,400 gas
```

The final line is derived as `127 bytes * 200 gas/byte`. It is the EVM deployment
code-deposit component, not a runtime-gas or dollar-savings claim. The report does not use
the 227-byte cumulative M-10-plus-M-60 figure as I-17's contribution.

### Conditions under which this Insight is valuable

1. **A HousePool feature or fix needs more than the present 5-byte margin.** The isolated
   removal raises EIP-170 headroom to 132 bytes without removing a live accounting path.
2. **Deployment resource use matters.** The retained compiler delta removes 127 deposited
   runtime bytes and their derived 25,400-gas code-deposit component.
3. **Review and maintenance should follow the live architecture.** Removing a second coupon
   implementation whose meaningful branches cannot run at its only caller narrows the code
   that future accounting changes must preserve and audit. This supporting benefit is not the
   eligibility basis; the measured resource effect is.

### Knownness and nearest public items

The nearest retained public items are similar in theme but do not have the same root,
current path, no-op condition, minimal correction, and measured resource effect:

- Filed Insight `#98441` removes the different external `HousePool.reconcile()` ABI wrapper
  and its `IHousePool` declaration because no production source caller uses that selector; it
  measures 100 bytes. I-17 removes an internal helper and its sole reached-but-always-no-op
  call, with an independently isolated 127-byte measurement. The two corrections and byte
  deltas are not bundled.
- The 4 May 2026 SC Audit Studio pre-audit item `SC-I1` identifies two different private
  helpers, `_distributeRevenue` and `_absorbLoss`, plus the `Reconciled` event. Its response
  says those objects were removed in Plether-Fi/plether-core#19. It does not identify this
  helper, this caller condition, or this measured correction.
- `SC-Q4` is the strongest overlap. That acknowledged May pre-audit proposal would replace
  the entire HousePool/tranche architecture. Its deletion list includes the historical
  `_checkpointSeniorYieldBeforePrincipalMutation`, `initializeSeedPosition`, and both public
  and internal reconcile functions. It does not preserve the current architecture and identify
  a redundant helper within it. The present no-op condition arose later: commit
  `e711621f8795909faa63f1abe4845550b88c4f1f` made the preceding `_reconcile(...)`
  unconditional for both seed types while leaving the later helper call.
- `LOW-65` is the ownerless-asset issue repaired by that reconcile ordering. It concerns
  preventing pre-existing ownerless value from being routed to a junior seed receiver, not the
  residual helper's no-op state or its 127-byte resource effect.
- Cecuro `MED-1` and `MED-27` concern stale-mark and senior-rate checkpoint semantics.
  Native `SCAN-1` concerns checkpoint-frequency compounding. None concerns this seed-path
  no-op call or this minimal bytecode removal.

No exact collision was found in the retained public corpus. That statement is bounded: a
private or concurrently filed first Insight is unknowable from local evidence and requires an
authenticated filing-day duplicate refresh.

### Recommendation

Remove the call from `initializeSeedPosition()` and delete
`_checkpointSeniorCouponBeforePrincipalMutation()`. Keep coupon checkpointing in the
surviving `_reconcile(...)` path. If call ordering is changed later, preserve the intended
coupon invariant in that canonical path and its tests rather than restoring the redundant
implementation.

## Proof of Concept

## Proof of Concept / Text Demonstration

### Control-flow demonstration

```text
initializeSeedPosition()
  -> _reconcile(...)
       elapsed > 0: checkpoint := current block.timestamp
       elapsed = 0: checkpoint was already >= current block.timestamp
  -> quoteBootstrapDeposit(amount) [external view / static call]
       shares = 0: revert; otherwise no state write
  -> _checkpointEngineCarryIndexes()
       no HousePool coupon-checkpoint write; timestamp unchanged
  -> _checkpointSeniorCouponBeforePrincipalMutation()
       recomputed elapsed = 0
       immediate return
```

### Measurement reproduction

1. Check out `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Confirm the helper has one call expression, after `_reconcile(...)`, in
   `initializeSeedPosition()`.
3. Remove only that call and the helper definition.
4. Build with Forge 1.5.1, Solidity 0.8.35, optimizer 200, via-IR, and Prague.
5. Compare the retained isolated baseline of 24,571 runtime bytes with the 24,444-byte
   successor: `24,571 - 24,444 = 127` bytes.
6. Recompute headroom: `24,576 - 24,571 = 5` and
   `24,576 - 24,444 = 132` bytes.

No attachment, Gist, public-chain transaction, simulation, or vulnerability PoC is relied on.
The conditions of value, mechanism, measurement, limitations, and knownness distinctions are
all stated inline.

