# 89675 w a medium v1 v2 migration re grants role set resolver to a revoked manager from a stale snapshot

**Submitted on Aug 20th 2026 at 13:24:32 UTC by @GaelicThunder for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89675
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

During the v1→v2 migration the Manager app grants the name's *v1 manager* the permanent v2 role `ROLE_SET_RESOLVER`. The manager address is taken from a **cached v1-subgraph snapshot** and is **never revalidated on-chain** before the grant is executed. A manager the owner has already revoked (via `BaseRegistrar.reclaim`) is therefore silently re-armed on v2 with permanent authority to change the name's resolver — i.e. to redirect where the name resolves.

The app's own post-migration verifier reads the *same* stale field, so the migration is reported as **verified** while the role sits on the wrong address. The confirmation UI never displays which address receives the role, so the owner has no way to notice.

## Root cause

**1. The manager is derived only from the subgraph payload** — `packages/migration/src/service/classifyNames.ts:162-167`:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)   // v1 registry owner = "manager"
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

**2. That value becomes a permanent on-chain role grant** — `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9-26`:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
...
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]   // ETHRegistry.grantRoles
```

**3. Nothing re-reads the live v1 registry owner.** The only on-chain reads across the whole migration surface are `ownerOf` (the registration-NFT holder) — `apps/manager/.../migrationService.ts:721` and `packages/migration/src/service/preflightChecks.ts:62`. `migrationInvariants.ts` verifies *contract* invariants (bytecode, certification) only, never the manager. There is no `ENS.owner(node)` read anywhere in the flow.

**4. The snapshot is stale by design, and stays stale.**

* `apps/manager/src/features/migration/hooks/useV1Names.ts:22` → `staleTime: 5 * 60 * 1000`. The subgraph name list — and therefore `domain.owner.id` — is served from a **5-minute client cache**, on top of the subgraph's own indexing lag.
* `apps/manager/src/features/migration/pages/MigrationPage.tsx:175` — the page that *executes* the migration consumes that cached query directly: `const { data: v1Names = [] } = useV1Names()`.
* `buildMigrationPlan.ts:327` classifies once. `adjustPlanForRetry` (`buildMigrationPlan.ts:413-478`) filters already-migrated names out of `plan.classified` but **never re-classifies**, so the snapshot also survives every failure/retry for the entire migration session.

So the exposure window is **at least five minutes and extends across all retries** — this is not a single-block race.

**5. Builder and verifier read the same stale field** — `buildAtomicMigrationBatches.ts:607-623`:

```ts
if (classified.managerAddress) {
  innerExecutions.push({ phase: 'manager-role-grant', call: buildRoleGrantCall(classified) })
  verificationExpectations.push({
    type: 'manager-role',
    account: classified.managerAddress,      // <-- same stale value the call used
    roleBitmap: ROLE_SET_RESOLVER, ...
  })
}
```

The post-migration check compares the chain against the stale expectation, so it passes. The app cannot detect its own mistake.

**6. No disclosure.** The only user-facing strings are `WalletConfirmationStepsDialog.tsx:84,87` — "Approve manager restoration" / "Restore the existing managers for your names" — and `GameStep.tsx:137`. The recipient address is never rendered, before or after signing.

## Attack scenario

No attacker privilege and no attacker transaction is required up to step 4. The "attacker" is simply an address the owner previously trusted as manager and has since revoked.

```
T0  alice.eth — registrant = Victim, v1 registry owner (manager) = Attacker
    (ordinary delegation: a team member, an ex-collaborator, an old multisig)

T1  Victim opens Manager → migration plan classified
    snapshot: managerAddress = Attacker

T2  Victim revokes the manager on-chain:
    BaseRegistrar.reclaim(tokenId, Victim)
    ("tidy up permissions before migrating" — the natural sequence)

T3  Victim executes / resumes the migration.
    Reached whenever step 3 happens within the 5-minute useV1Names cache,
    within subgraph indexing lag, or on ANY retry of a plan built before T2.

T4  ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, Attacker)   <-- PERMANENT on v2
    App reports "migration verified" (verifier reads the same stale value).
    UI never showed the Victim which address was granted.

T5  Attacker calls setResolver(alice.eth, attackerResolver)
    → addr(alice.eth) = Attacker. Every payment sent to alice.eth goes to the Attacker.
```

T5 uses `ROLE_SET_RESOLVER` for its documented purpose on the (out-of-scope) v2 registry. The in-scope defect is entirely in T1→T4: the application decides, without revalidation, to grant on-chain authority to an address whose authority no longer exists.

## Impact

Permanent, unnoticed resolver-control over the migrated name for an address the owner explicitly revoked. Because the resolver determines `addr()`, this is a direct route to redirecting funds sent to the name — the competition's Critical class *"a Manager flow (registration, v1→v2 migration, transfer) that assigns the name, its ownership, registry, or resolver to an attacker"*, and it also lands in *"resolving an ENS name to an attacker-controlled address."*

## Link to Proof of Concept

<https://gist.github.com/GaelicThunder/30b37bee34f2c3cbfd412084e1b15732>

## Proof of Concept

## Proof of concept

`poc/staleManagerRoleGrant.poc.test.ts` — drop into `apps/manager/src/features/migration/service/` and run:

```bash
cd apps/manager
./node_modules/.bin/vitest run src/features/migration/service/staleManagerRoleGrant.poc.test.ts
```

Result: **4/4 pass** (`poc/output.txt`). It uses only shipped functions — `classifyNames`, `buildRoleGrantCall`, and a real `decodeFunctionData` against the app's own `ETH_REGISTRY_V2_ABI` — and it is deliberately **bidirectional**:

| Test                                           | Assertion                                                                                    |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| manager derived from subgraph `owner`          | `managerAddress === ATTACKER`                                                                |
| **CONTROL** — fresh classify after `reclaim()` | `managerAddress === null`; `buildRoleGrantCall` throws                                       |
| **BUG** — stale snapshot                       | decodes to `grantRoles(canonicalId('alice'), 1<<24, ATTACKER)` on `V2_CONTRACTS.ETHRegistry` |
| isolating pair                                 | same function, inputs differ only in freshness                                               |

The control matters: it shows the safe path behaves *differently*, isolating the root cause to **stale migration state** rather than to any ENSv2 contract behaviour.

## Recommendation

1. Read `ENS.owner(node)` on-chain immediately before emitting the role grant; if it differs from `classified.managerAddress`, drop the grant and re-plan. This is the actual fix — the grant must be authorised by live state, not by a snapshot.
2. Re-classify inside `adjustPlanForRetry` instead of reusing `plan.classified`.
3. Derive the verification expectation from the freshly-read on-chain value, so the verifier can disagree with the builder.
4. Render the recipient address in the "Approve manager restoration" confirmation step
