# \#98820 [SC-Insight] Document that frozen, degraded, and terminal-deficit states reject LP entry requests

**Submitted on Nov 10th 2026 at 11:57:04 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98820
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Summary

The public `IHousePool` interface does not document a decisive distinction
between an existing queued deposit and a new deposit request. Its frozen-fee
NatSpec says that entry activation is deferred while the oracle is frozen. That
is true for a request queued earlier, but new requests are not accepted during
the freeze. The `canAcceptTrancheDeposits` NatSpec also lists lifecycle, pause,
freshness, unassigned-asset, impairment, and Senior-capacity conditions while
omitting the shared status gate that rejects both tranches during oracle-frozen
mode, degraded mode, or a nonzero current terminal deficit.

These are two expressions of one incomplete public entry-admission contract:
the shared gate decides whether a new request may enter the queue, while the
frozen-fee comment describes what happens to work that was already queued.

At the shared-status stage, the implementation returns `false` when any one of
the three blockers holds, before the downstream tranche-state, projected-entry,
and capacity checks. `TrancheVault.maxRequestDeposit` then returns zero. With a
nonzero controller and owner, an owner or approved-operator caller, a Junior
tranche that is not terminally wiped, an active ordinary lifecycle, and an
amount at or above the request minimum, a matching positive `requestDeposit`
reaches `TrancheVault__DepositsUnavailable`. If the tranche is terminally wiped,
the earlier `TrancheVault__TerminallyWiped` branch controls instead.

This is a Documentation Improvements Insight. The admission behavior is treated
as intended and correct; no vulnerability impact is alleged.

### Exact documentation gap

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

- `IHousePool.sol:512-520` describes `canAcceptTrancheDeposits` and enumerates
  several requirements, but does not state that frozen, degraded, or
  current-terminal-deficit status closes request admission.
- `IHousePool.sol:543-551` says only that “entry activation is deferred while
  frozen.” It does not distinguish an already-queued request, whose activation
  is deferred, from a new request, which cannot be queued.
- `HousePool.sol:480-503` calls `_entryStatusBlocked` after a separate
  Senior-only gate and before the tranche-state, projected-entry, and capacity
  checks reported here.
- `HousePool.sol:1139-1145` defines that shared blocker as `oracleFrozen`,
  `degradedMode`, or a nonzero deficit from the current terminal-NAV snapshot.
- `TrancheVault.sol:1313-1325` turns the closed pool gate into zero
  `maxRequestDeposit` capacity, and `TrancheVault.sol:1574-1600` turns that zero
  into `TrancheVault__DepositsUnavailable` on request preflight.
- `HousePool.sol:1100-1118` applies the same status blocker to activation. In
  the oracle-frozen control, settlement reaches the entry phase and
  `HousePool.sol:903-911` leaves previously queued deposit work deferred.

The frozen-oracle behavior is already stated precisely in the concrete
implementation NatSpec: `HousePool.sol:1416-1418` says both entry requests and
activations are unavailable. The requested improvement is limited to aligning
the public `IHousePool` interface contract with that implemented behavior.

### Why the improvement is valuable

The distinction changes the integration state machine:

- an already-queued deposit remains escrowed and waits for a later eligible
  activation; while
- a new request has zero advertised capacity and reverts instead of joining
  that queue.

An SDK, wallet, frontend, or operations runbook generated from `IHousePool`
NatSpec can otherwise describe frozen mode as “queue now, activate later,” retry
a request that cannot presently be admitted, or classify
`DepositsUnavailable` as an unexpected failure. The same distinction matters
for degraded mode and a current terminal deficit, where the admission predicate
also closes before the documented downstream checks.

`PerpsPublicLens` exposes the live `depositEnabled` boolean next to
`frozenLpFeeBps` and `oracleFrozen`. An integration that reads and honors that
boolean will avoid the transaction. This does not remove the documentation
value: the interface contract should explain why the boolean is false and
distinguish admission refusal from activation deferral.

The correction is comment-only and requires no ABI, runtime, storage, caller,
or migration change.

### Existing documentation and PR #106

The concrete `HousePool` NatSpec and `WHITEPAPER.md` already state the frozen
new-request-versus-queued-activation distinction. `ACCOUNTING_SPEC.md` and
`SECURITY.md` describe the related activation blockers, and the public lens
exposes current request availability. Those sources confirm the intended
behavior; they do not complete the canonical `IHousePool` interface contract
used by integrations and documentation generators.

Public [PR #106](https://github.com/Plether-Fi/plether-core/pull/106),
`docs(perps): align documentation with current implementation`, merged as
commit `023412d0d16a0a950b9d751ca29f67a186a5f230`, changed the exact
`frozenLpFeeBps` interface sentence to describe frozen redemption funding and
deferred activation. It did not add the separate new-request refusal or the
three shared `canAcceptTrancheDeposits` blockers. This report identifies that
remaining interface gap after the documentation-alignment change; it does not
claim that the runtime behavior itself is new.

### Recommended correction

1. Add the three shared status blockers to
   `IHousePool.canAcceptTrancheDeposits` NatSpec: oracle-frozen mode, degraded
   mode, and a nonzero current terminal deficit.
2. Change the `IHousePool.frozenLpFeeBps` note to state that new entry requests
   and deposit activation are unavailable while frozen, while claims already
   funded do not reprice.

This recommendation is intentionally limited to the shared
frozen/degraded/deficit gate. Separate tranche-specific admission rules are
outside this report.

### Limitations and non-claims

- “Entry activation is deferred” is literally true for deposits queued before
  the blocking state. The gap is that the interface does not also state the
  separate new-request result or the three shared admission blockers.
- No live exploit, loss, freeze impact, gas saving, byte saving, or
  vulnerability severity is claimed.
- No external integration failure or production occurrence was observed.


## Proof of Concept

## Proof of Concept

1. Check out commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Read `packages/perps/src/interfaces/IHousePool.sol:512-520` and
   `:543-551`. Record the documented admission conditions and the statement
   that activation is deferred while frozen.
3. Use the Junior tranche (`isSenior == false`) to isolate the shared predicate
   from separate Senior-only admission rules. Read
   `packages/perps/src/HousePool.sol:480-503` and `:1139-1145`. Observe that
   admission returns false before the downstream gates when `oracleFrozen`,
   `degradedMode`, or the current terminal deficit is nonzero.
4. Read `packages/perps/src/TrancheVault.sol:1313-1325`, `:1574-1600`, and
   `:1639-1641`. Use nonzero controller and owner addresses, call as the owner
   or an approved operator, keep `_isTerminallyWiped()` false, keep the ordinary
   lifecycle active, and use an amount at or above the request minimum. Observe
   that any one of the three shared status blockers makes `maxRequestDeposit`
   return zero and makes the positive request revert with
   `TrancheVault__DepositsUnavailable`; if `_isTerminallyWiped()` is true,
   `TrancheVault__TerminallyWiped` occurs earlier.
5. For the oracle-frozen control, read
   `packages/perps/src/HousePool.sol:794-833`, `:903-911`, and `:1100-1118`.
   With the other settlement preconditions open, observe the different
   treatment of previously queued work: the frozen settlement path reaches the
   entry gate, and activation remains deferred.
6. Read `packages/perps/src/HousePool.sol:1416-1418`. The concrete
   implementation NatSpec already states the frozen rule precisely.

### Conditions under which the Insight is valuable

The correction is valuable whenever an integrator, maintainer, SDK or
documentation generator, test author, or operations team uses the public
`IHousePool` comments to understand, implement, test, or explain the LP-entry
state machine, request retry policy, or `DepositsUnavailable` result. No shared
blocking state must be active at filing and no integration failure must already
have occurred: the implemented branches show that the interface currently
omits admission outcomes a correct public contract should describe.

For an isolated runtime demonstration, hold the step 4 prerequisites and the
other downstream admission conditions open, then make any one—not all—of
`oracleFrozen`, `degradedMode`, or current terminal deficit true. A new request
is refused rather than queued for later activation. A request queued before an
oracle freeze remains the contrasting deferred-activation case. Observing a
successful settlement return, advancing a matured redemption, or avoiding
`HousePool__NoLpEpochProgress` is not necessary to establish the documentation
gap or its value.

