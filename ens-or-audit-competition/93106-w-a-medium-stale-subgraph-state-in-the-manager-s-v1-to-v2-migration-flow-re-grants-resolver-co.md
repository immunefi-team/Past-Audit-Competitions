# 93106 w a medium stale subgraph state in the manager s v1 to v2 migration flow re grants resolver control authority to an ensv1 manager the registrant already revoked

**Submitted on Sep 14th 2026 at 09:25:29 UTC by @Kodekloner43926 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93106
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager's v1-to-v2 migration restores the previous ENSv1 registry manager as an ENSv2 role holder, but it derives that manager from a possibly-stale subgraph snapshot instead of reading the current on-chain registry owner at submission time. If the registrant revokes a delegated manager with ENSv1 `reclaim` after the snapshot was taken (or while the Manager's cached domain data or immutable migration preview is still active), the normal migration transaction still grants the revoked account ENSv2 `ROLE_SET_RESOLVER` for the migrated name. The former manager can then point the name at an attacker-controlled resolver and redirect its resolution. The submitter self-assesses this as Medium severity: the impact is serious (resolver substitution for the migrated name), but it requires a narrow sequence — a mid-flow reclaim combined with stale indexed state — and the grant is revocable by the owner once discovered.

## Vulnerability Details

Affected asset: **Manager app Files** (`apps/manager`), frozen competition revision. The vulnerable flow is the v1-to-v2 migration in `apps/manager/src/features/migration/`, which consumes classification helpers from `packages/migration` as part of that flow.

**1. The manager to restore is taken from indexed state, not from the chain.** For an unwrapped `.eth` name, the classifier invoked by the Manager's migration flow treats the subgraph's `domain.owner` as the manager whenever it differs from the BaseRegistrar registrant (`packages/migration/src/service/classifyNames.ts`, lines 149–185):

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The current ENSv1 registry owner is never read from chain for this decision.

**2. The only ownership preflight proves token ownership, not manager authority.** `checkOwnership` (`packages/migration/src/service/preflightChecks.ts`, lines 47–91) verifies that the migrating wallet still holds the registration NFT:

```typescript
functionName: 'ownerOf',
args: [BigInt(name.domain.labelhash)],
```

A registrant who has reclaimed the registry node still passes this check — it says nothing about who currently manages the ENS registry node.

**3. The stale address is written into the migration transaction unconditionally.** The batch builder (`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`, lines 608–624) pushes a manager-restoration call whenever the classified object carries a `managerAddress`:

```typescript
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
  /* ...verification expectation that the manager received the role... */
}
```

And the call builder (`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`, lines 11–27) encodes the grant for whatever address the classification carried:

```typescript
functionName: 'grantRoles',
args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

**4. Execution does not refresh the manager.** `buildNextAtomicBatch` (`apps/manager/src/features/migration/service/migrationService.ts`, lines 886–925) re-reads resolver readiness and direct migration routes from chain immediately before submission, but takes its `remaining` names from `plan.classified` — the immutable preview's classification, including the stale `managerAddress`. No step between preview and submission reads the current ENSv1 registry owner for an unwrapped name.

**How the stale state arises (all ordinary conditions):**

* The Manager caches the V1 domain query for five minutes (`apps/manager/src/features/migration/hooks/useV1Names.ts`, `staleTime: 5 * 60 * 1000`).
* The migration preview plan is immutable once built; the wallet-confirmation and execution path reuse its classified names.
* Independently of the client cache, the hosted subgraph itself lags the chain, so even a freshly-issued query can return the pre-reclaim owner.

**Preconditions:**

1. The victim owns an unwrapped `.eth` BaseRegistrar token (the normal migration population).
2. The ENSv1 registry node previously had a different owner/manager (a delegated manager — a standard ENSv1 arrangement).
3. The Manager has cached or previewed the domain while that prior manager is present, and manager restoration is included in the migration plan.
4. The victim revokes that manager by reclaiming the ENSv1 registry node (in another tab or client) before submitting the migration, then completes the normal migration flow.

The attack needs no compromised key, browser defect, local-network access, or control of the ENS subgraph. Ordinary indexer delay or the application's own immutable preview is sufficient.

**Root cause in one sentence:** the migration flow treats indexed, possibly-stale `domain.owner` state as an authorization fact and restores it as ENSv2 role authority without an on-chain freshness check, while its only ownership preflight validates the registration NFT rather than the registry node's current manager.

**Recommended fix.** Immediately before constructing the manager-restoration call, read the current ENSv1 registry owner for each unwrapped name (at the same block used for other live checks) and restore a role only when that authoritative address still differs from the registrant; bind the manager address into the plan-freshness check and abort with a plan-changed error if it differs from the preview. Display every manager address and the exact role in the wallet-confirmation UI. Add regression tests covering both directions: preview with manager A → registry reclaimed to registrant (no grant to A), and preview with manager A → manager changed to B (new preview required before granting B).

## Impact Details

Selected impact category: **Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer) — a migration that assigns the name's resolver control to an attacker.**

**Submitter's severity self-assessment: Medium.** The program's impact list places this category at Critical; the self-assessment below explains why the submitter evaluates the realistic severity as Medium and leaves the final classification to the judges.

**What the vulnerability grants:** the migration transaction grants the revoked account ENSv2 `ROLE_SET_RESOLVER` for the migrated name. That role directly authorizes `PermissionedRegistry.setResolver`, so the former manager can replace the name's resolver with an attacker-controlled contract and take over all of its resolution — address records, text records, and any record a consumer reads through the name. Until the owner notices and revokes the hidden role, applications resolving the name can be served attacker-controlled addresses.

**Attack path:**

1. The victim owns `victim.eth` at the BaseRegistrar; an attacker was previously the ENSv1 registry manager of the node.
2. The Manager fetches the domain; the snapshot contains `registrant = victim` and `owner = attacker`, so the classification stores the attacker as `managerAddress`.
3. The victim revokes that manager by reclaiming the ENSv1 registry node in another tab or client.
4. The token-ownership preflight still passes (`BaseRegistrar.ownerOf` still returns the victim). No code reads the current ENSv1 registry owner.
5. The victim submits the displayed migration; the live batch builder reuses the preview's stale `managerAddress`.
6. The atomic migration batch migrates the name and grants `ROLE_SET_RESOLVER` to the attacker; the post-migration verification confirms the attacker received the role, so the flow reports success.
7. The attacker sets an attacker-controlled resolver and changes address resolution for the migrated name.

**Why Medium rather than Critical (the self-assessment):**

* **Narrow precondition window.** The victim must reclaim the registry node in the interval between the subgraph snapshot (or the five-minute client cache / immutable preview) and migration submission, and must have had a distinct delegated manager beforehand. This is an unusual mid-flow action sequence, not a state a normal migration passes through.
* **Single, revocable role.** The grant is limited to `ROLE_SET_RESOLVER` on the migrated name's resource. It confers no ownership or transfer authority, and the owner can revoke it once discovered — the harm is bounded hijacked resolution in the interim, not permanent loss of the name.
* **No fund movement in the transaction itself.** The migration transaction moves no user funds; theft requires the attacker to exercise the redirected resolution against a payer, a second-order consequence.

**Why not lower than Medium:** until discovered, resolver substitution gives the attacker complete control over what the name resolves to — including the ETH address payers rely on — so the consequence is materially above a display or availability issue.

**Risk breakdown:**

* **Victim:** a registrant migrating an unwrapped `.eth` name that has (or had) a delegated ENSv1 manager.
* **Attacker prerequisite:** being the previously-delegated manager whose authority the victim revoked mid-flow.
* **Required victim interaction:** the victim performs the reclaim and then completes the normal migration flow.
* **Complexity:** low once the precondition state exists; the stale-address propagation is deterministic.
* **Security boundary crossed:** an account whose ENSv1 authority was revoked receives fresh ENSv2 authority during migration.
* **Maximum consequence:** resolver substitution for the migrated name and redirection of its resolution until the owner intervenes.

**Relation to published known issues.** No published entry covers stale manager restoration. QA-01 concerns functional follow-up errors after certain transfers; this finding produces a *successful* migration that silently grants a revoked account effective resolver authority — which the known-issues notice explicitly treats as a new security consequence rather than a functional error. The ENSv1/ENSv2 contracts are used only to demonstrate the authority the app reads and grants; the vulnerable query, preflight, transaction construction, and execution logic are in the in-scope Manager app code (with the classifier it invokes quoted for context).

## References

* ENS Audit Competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS Audit Competition published known issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* Subgraph client consumed by the Manager (cached domain fetch): <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L40>
* Manager-side classification consumption: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L149-L185>
* Ownership preflight (token ownership only): <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L47-L91>
* Grant-call builder: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L27>
* Unconditional manager-grant push in the atomic batch: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624>
* Execution rebuild from the immutable plan: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886-L925>

## Proof of Concept

## Proof of Concept

### Part A — Step-by-step (application flow)

1. A victim owns an unwrapped `.eth` name and has a delegated ENSv1 manager (the registry node owner differs from the registrant).
2. The victim opens the Manager's migration page. The V1-names query (cached for five minutes) returns the domain with `registrant = victim`, `owner = manager`.
3. The victim builds the migration preview. The classification stores the manager as `managerAddress`; the preview plan is immutable from this point.
4. The victim revokes the manager by reclaiming the ENSv1 registry node to their own address in another tab or ENS client.
5. The victim returns to the Manager and submits the migration. The ownership preflight passes (the victim still holds the registration NFT); nothing reads the current registry owner.
6. Observed: the atomic migration batch executes, and the post-migration verification confirms the revoked manager received `ROLE_SET_RESOLVER` on the migrated name's V2 resource. The Manager reports a successful migration.
7. The revoked manager calls `setResolver` with an attacker-controlled resolver and controls the name's resolution.

### Part B — Deterministic PoC (production code paths)

Prerequisites:

* Check out the frozen competition revision.
* Use Node.js 22 and pnpm 10, as specified by the competition.
* Install dependencies with `pnpm install --frozen-lockfile`.

Create the file `audits/Immunefi/ENS-Sep-2026/poc/migration-stale-manager-restoration.poc.test.ts` in the scoped repository with the following content:

```ts
import assert from 'node:assert/strict'
import { it } from 'vitest'
import { decodeFunctionData } from '../../../../apps/manager/node_modules/viem/_esm/index.js'
import { classifyName } from '../../../../packages/migration/src/service/classifyNames.ts'
import type { V1Domain } from '../../../../packages/migration/src/service/v1SubgraphClient.ts'
import { ETH_REGISTRY_V2_ABI } from '../../../../apps/manager/src/features/migration/contracts/abis.ts'
import { V2_CONTRACTS } from '../../../../apps/manager/src/features/migration/contracts/addresses.ts'
import { buildRoleGrantCall } from '../../../../apps/manager/src/features/migration/service/buildRoleGrantCalls.ts'

const victim = '0x1111111111111111111111111111111111111111'
const revokedManager = '0x2222222222222222222222222222222222222222'

// The subgraph snapshot consumed by the migration UI. The victim is the
// BaseRegistrar registrant (they own the registration NFT), but the cached
// registry owner still names the old manager — the pre-reclaim state.
const cachedDomain: V1Domain = {
  id: `0x${'a'.repeat(64)}`,
  labelName: 'victim',
  labelhash: `0x${'b'.repeat(64)}`,
  name: 'victim.eth',
  resolver: null,
  owner: { id: revokedManager },
  registrant: { id: victim },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: '4102444800' },
  wrappedDomain: null,
}

// Authoritative state at submission time: the victim has reclaimed the ENSv1
// registry node, revoking the old manager. The application never reads this
// value — checkOwnership only calls BaseRegistrar.ownerOf, which the victim
// still satisfies.
const liveV1RegistryOwner = victim

it('encodes the revoked cached manager instead of the authoritative current manager', () => {
  // Stage 1 — production classifier over the stale snapshot.
  const classified = classifyName(cachedDomain, victim)
  assert(classified && classified.type === 'classified')
  assert.equal(classified.name.tokenType, 'unwrapped')
  assert.equal(classified.name.tokenHolder, victim)
  assert.equal(
    classified.name.managerAddress?.toLowerCase(),
    revokedManager.toLowerCase(),
    'the stale subgraph owner is treated as the manager to restore',
  )

  // Stage 2 — production transaction builder for the manager-role-grant phase.
  const call = buildRoleGrantCall(classified.name)
  assert.equal(call.to, V2_CONTRACTS.ETHRegistry)
  assert.equal(call.value, 0n)

  // Stage 3 — decode the calldata the wallet would sign.
  const decoded = decodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    data: call.data,
  })
  assert.equal(decoded.functionName, 'grantRoles')

  const [resource, roles, account] = decoded.args as readonly [
    bigint,
    bigint,
    `0x${string}`,
  ]
  // The remediated V2 registry packs roles in nybbles; ROLE_SET_RESOLVER is
  // bit 24 (see buildRoleGrantCalls.ts).
  assert.equal(roles, 1n << 24n, 'ROLE_SET_RESOLVER is granted')
  // The grantee is the revoked manager, not the current registry owner.
  assert.equal(account.toLowerCase(), revokedManager.toLowerCase())
  assert.notEqual(
    account.toLowerCase(),
    liveV1RegistryOwner.toLowerCase(),
    'the grant goes to the account the victim already revoked',
  )

  console.log(
    JSON.stringify(
      {
        name: cachedDomain.name,
        authoritativeV1ManagerAtSubmission: liveV1RegistryOwner,
        staleManagerEncodedByMigration: account,
        v2Function: decoded.functionName,
        v2TargetContract: call.to,
        resource: resource.toString(),
        role: 'ROLE_SET_RESOLVER',
        consequence:
          'the revoked manager can replace the migrated name resolver and redirect all records',
      },
      null,
      2,
    ),
  )
})

it('produces no manager grant when the snapshot reflects the reclaim (contrast case)', () => {
  // Same domain, but the (fresh) subgraph now shows the post-reclaim state:
  // registry owner == registrant. Classification then yields no manager, and
  // the grant phase is skipped entirely — proving the vulnerable grant in the
  // first test exists only because the stale address was trusted.
  const reclaimedDomain: V1Domain = {
    ...cachedDomain,
    owner: { id: victim },
  }
  const fresh = classifyName(reclaimedDomain, victim)
  assert(fresh && fresh.type === 'classified')
  assert.equal(fresh.name.managerAddress, null)
  assert.throws(
    () => buildRoleGrantCall(fresh.name),
    /No manager address/,
    'with a current snapshot there is no manager grant to build',
  )
})
```

The PoC drives the production classifier (`classifyName`) and the production grant-call builder (`buildRoleGrantCall`) that the atomic migration batch invokes, then decodes the resulting calldata. The first test proves the revoked manager receives `ROLE_SET_RESOLVER`; the contrast test proves that with a post-reclaim snapshot no grant is produced at all — isolating the stale-address trust as the cause.

From the repository root, enter the Manager workspace and run:

```bash
cd apps/manager
./node_modules/.bin/vitest --root ../.. --config apps/manager/vitest.config.ts run audits/Immunefi/ENS-Sep-2026/poc/migration-stale-manager-restoration.poc.test.ts
```

On Windows PowerShell, the equivalent verified command is:

```powershell
cd apps\manager
& .\node_modules\.bin\vitest.CMD --root ..\.. --config apps\manager\vitest.config.ts run audits\Immunefi\ENS-Sep-2026\poc\migration-stale-manager-restoration.poc.test.ts
```

Observed result:

```
 Test Files  1 passed (1)
      Tests  2 passed (2)
```

Verified against the frozen revision's dependencies with Vitest `v4.1.5` on Node 22. (On setups where the repository is mirrored by pnpm as a linked project, the same run may report the file twice — both executions pass.)

**Expected result:** immediately before constructing the manager-restoration call, the Manager reads the current ENSv1 registry owner for each unwrapped name and restores a role only when that authoritative address still differs from the registrant; a change after preview rebuilds the preview or omits the grant.

**Observed result:** the Manager restores `domain.owner.id` from a cached subgraph/preview object without any current on-chain registry-owner check. Token-ownership checks do not cover manager authority, so a successfully revoked manager is granted new ENSv2 resolver control for the migrated name.
