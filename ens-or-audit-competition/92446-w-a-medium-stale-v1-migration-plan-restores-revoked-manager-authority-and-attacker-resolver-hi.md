# 92446 w a medium stale v1 migration plan restores revoked manager authority and attacker resolver hijacking an ens name

**Submitted on Sep 9th 2026 at 21:19:53 UTC by @vsmarkall for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92446
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager app builds ENSv1-to-ENSv2 migration calldata from indexed V1 Registry `owner` and `resolver` fields, then reuses those classified values at submission without reading the current V1 Registry manager or resolver. If a former manager's indexed state remains cached after the registrant reclaims the name and installs a safe resolver, the normal migration restores the attacker's custom resolver and grants the former manager V2 `ROLE_SET_RESOLVER`. The migrated name then resolves its ETH address to the attacker, and the former manager retains on-chain authority to repoint the resolver after the migration workflow revokes its temporary HCA approval.

**Asset:** Manager app Files

**Severity:** Critical

**Primary eligible impact:** Hijacking or bricking an ENS name during a Manager flow—specifically, a V1-to-V2 migration assigns the name's resolver and continuing resolver-management authority to an attacker.

**Secondary demonstrated impact:** The application causes the migrated ENS name to resolve to an attacker-controlled ETH address.

## Vulnerability Details

The V1 subgraph response contains the registrant plus the mutable Registry `owner` and `resolver` fields. Manager preserves the latter fields as `managerAddress` and `v1ResolverAddress` during classification. A migration plan generated from that response is retained through review and submission.

Before submission, Manager refreshes approval keys, deterministic resolver readiness, and direct migration routes, but it never rebinds the classified manager and resolver to current V1 Registry reads. The live rebuild still receives the earlier classified objects.

This does not require a broken or attacker-controlled indexer. An already-mounted `useV1Names` query has no polling or live Registry rebind, so its previously returned domain object remains available to the migration page after the underlying V1 state changes. Although the hook declares a five-minute `staleTime`, the shared `resultQueryOptions` wrapper currently removes that option. This report therefore does not depend on a guaranteed five-minute cache duration: its demonstrated lifecycle is a migration page kept open across the victim's legitimate reclaim.

The gas-estimate plan is also keyed by selected domain IDs rather than the mutable manager and resolver fields. The normal Upgrade action passes the reviewed plan to `executeMigration`, whose submission-time rebuild uses the old `remaining` classified objects while refreshing only resolver readiness and direct migration routes.

The vulnerable data flow is:

```
indexed V1 owner/resolver
  -> classified.managerAddress / classified.v1ResolverAddress
  -> reviewed migration plan
  -> buildNextAtomicBatch(classified: remaining)
  -> stale resolver installation + ROLE_SET_RESOLVER grant
  -> verifier checks expectations derived from the same stale values
```

The security-sensitive construction is equivalent to:

```ts
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
// createMigrationData({ owner: wallet, resolver: expectedResolver })

account: classified.managerAddress,
roleBitmap: ROLE_SET_RESOLVER,

// Submission rebuild still receives the previously classified objects.
classified: remaining,
```

Receipt verification derives its expected resolver and role state from the same stale plan, so it confirms the authority restoration instead of detecting it.

### Why the V2 role grant succeeds

The role is newly issued during V2 migration. It is not residual V1 authority, and V1 itself does not grant a V2 role:

1. The stale `managerAddress` makes Manager classify the plan as requiring manager restoration. Manager prompts the victim to approve the victim-owned HCA as an operator of the V2 Registry when necessary.
2. The migration call executes first and registers the V2 name to the victim. Registration gives the owner `ROLE_SET_RESOLVER` and the paired admin role.
3. ENSv2 `PermissionedRegistry` makes an approved operator inherit the owner's effective roles. The victim-owned HCA can consequently grant the ordinary resolver role on the victim's name.
4. Manager intentionally orders `migrate` before `manager-role-grant`, so the name and owner-admin bitmap exist when the HCA calls `grantRoles(resource, ROLE_SET_RESOLVER, formerManager)`.
5. Manager's cleanup revokes only the HCA operator approval. It does not revoke the newly stored role from the former manager.

The PoC proves that the attacker role remains active after cleanup and that the attacker can use it in a separate transaction.

## Impact

The PoC begins with the victim as both the current V1 registrant and Registry manager, using a safe resolver that returns the victim address. The former manager has no current V1 authority and no V2 resolver role.

After the victim follows the normal Manager migration flow:

* the victim is still displayed as the V2 owner;
* the attacker's custom resolver is installed on the victim's name;
* Universal Resolver returns the attacker-controlled ETH address;
* an `EACRolesChanged` event proves that the former manager received `ROLE_SET_RESOLVER` during the migration;
* production receipt verification accepts the compromised state as a successful migration;
* revoking the migration HCA's operator approval does not remove the attacker's independently stored role; and
* the attacker successfully uses that role later to repoint the resolver.

This matches two explicit Critical impacts in the competition taxonomy: a Manager migration assigns a name's resolver to an attacker, and the application resolves or renders an ENS name to an attacker-controlled address. Any payment sent using that resolution can consequently be diverted. The PoC intentionally transfers no real funds; Critical classification rests on the explicit name-hijack and wrong-resolution impacts, not on a speculative dollar amount.

The owner can recover after detecting the compromise by revoking the attacker role and restoring the safe resolver, so this report does not claim irreversible ownership loss. The security impact is the successful migration-time hijack and the newly granted, durable attacker authority.

## Attack Preconditions and Feasibility

1. The attacker previously controlled both the V1 Registry manager position and a custom resolver for a victim-held, unwrapped V1 `.eth` registration.
2. Manager obtains or retains that manager and resolver state for migration.
3. The victim legitimately reclaims current V1 Registry manager authority and installs a safe resolver.
4. Before the already-mounted client state refreshes, the victim completes the normal Manager migration.

The attacker has no current authority when the vulnerable migration executes. The victim follows the prompted Manager approvals and normal Upgrade workflow. No privileged project role, leaked key, browser defect, social engineering, indexer compromise, third-party testing, or public-network state change is required. Timing depends on the page lifecycle and ordinary index propagation; focus/reconnect or explicit invalidation may refresh the query.

## Network and Deployment Relevance

The competition identifies Sepolia as the primary network, and ENSv2 is deployed on Sepolia for testing. The stateful PoC was successfully rerun against a local fork of Sepolia block `11652562` (timestamp `2026-09-07T06:57:00Z`, hash `0x972a464b81baac77a63de8dd59832a39931f450d959d54326abcf6b61a9b251d`). It uses the scoped Manager code and deployed ENS contracts rather than mocked V2 authorization logic.

This report does not assert that ENSv2 is live on Ethereum mainnet or that a mainnet V1 contract can issue `ROLE_SET_RESOLVER`. The finding applies to the competition's Sepolia ENSv2 beta and the audited pre-mainnet Manager code. That is the deployment and application explicitly covered by the competition.

## Scope and Novelty

The vulnerable source and sink are within the scoped Manager app:

* `apps/manager/src/features/migration/hooks/useV1Names.ts:12-23` keys indexed V1 domain objects only by address and configures no polling.
* `packages/utils/src/tanstack-query/neverthrow/query.ts:120-147` removes the declared `staleTime` from the returned options.
* `apps/manager/src/features/migration/pages/MigrationPage.tsx:194-200,259-282` builds the reviewed plan and passes it through the normal Upgrade action.
* `apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47` fetches indexed V1 Registry resolver and owner fields.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:481-491` places the classified resolver in migration data.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624` grants the classified manager `ROLE_SET_RESOLVER` and records it as an expected result.
* `apps/manager/src/features/migration/service/migrationApprovals.ts:186-193,320-366` builds the required V2 HCA approval and later revokes that approval.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:737-780` orders migration before the manager-role grant.
* `apps/manager/src/features/migration/service/migrationService.ts:886-925` rebuilds with the old classified objects while refreshing only resolver readiness and direct routes.
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:263-275,337-353` verifies the stale resolver and role as successful receipt-block state.

The tested competition snapshot is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. Immunefi identifies upstream revision `63772fd872af472ced58b009499355f3430c2a86` as the audited source revision. The competition repository's later metadata and Hono update does not change the migration files above.

No published known issue covers trusting a former V1 Registry manager and resolver during successful migration. The closest disclosed items concern records not displayed after migration, omitted subnames, child-registry replacement, and role-transfer errors. None freezes a revoked identity into successful migration calldata. The known-issues document expressly identifies an unintended authority grant as a new security consequence beyond known functional breakage.

ENS contracts are contextual execution infrastructure only; the vulnerable planning, construction, and verification decisions are all made by the scoped Manager application. All state-changing PoC calls run only on a local Anvil fork.

## Recommended Fix

Immediately before final call construction, read the current V1 Registry owner and resolver at one consistent block for every migrating name. Reclassify and rebuild the plan from those values, and require renewed user review if either differs from the reviewed snapshot. Derive all manager-role and resolver expectations only from that fresh state. Verification should retain or commit to the freshness block so that a result cannot be accepted merely because it matches stale calldata.

Add a regression test that changes the V1 Registry manager and resolver between discovery/planning and submission. Assert that submission either aborts for renewed review or uses the current values, grants the former manager no role, preserves safe resolution, and rejects attacker `setResolver`.

## References

* [ENS competition information](https://immunefi.com/audit-competition/audit-competition-ens/information/)
* [ENS competition scope and impact taxonomy](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [ENS competition build and test resources](https://immunefi.com/audit-competition/audit-competition-ens/resources/)
* [ENSv2 overview](https://docs.ens.domains/ensv2/overview/)
* [ENSv2 Permissioned Registry](https://docs.ens.domains/ensv2/permissioned-registry/)
* [ENSv2 contracts role model](https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md)

## Link to Proof of Concept

<https://gist.github.com/vstorchak/1cc51fbfd74e4b72aee613ffefaa0c71>

## Proof of Concept

## Proof of Concept

### Safety boundary

The PoC performs every state-changing action only against a local Anvil fork of Sepolia. It does not submit transactions to Sepolia, Ethereum mainnet, or any other public network.

### Prerequisites

* Competition repository checked out at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* Node.js v22 and pnpm v10 (`corepack enable`).
* Dependencies installed with `pnpm install --frozen-lockfile`.
* Foundry tools `anvil` and `forge` available on `PATH`.
* `curl` and outbound HTTPS access for read-only Sepolia fork data.

Optional environment variables:

* `SEPOLIA_FORK_URL`: overrides the runner's Sepolia RPC source.
* `CAND_005_RPC_URL`: uses an already-running local fork instead of starting Anvil.
* `CAND_005_FORK_BLOCK`: overrides the default pinned block `11641625`.
* `ANVIL_BIN`: supplies an explicit Anvil binary path.

### One-command reproduction

Place the seven Gist files at `full-review-docs/pocs/CAND-005/` in the competition repository, preserving `src/AttackerResolver.sol`, then run from the repository root:

```bash
bash full-review-docs/pocs/CAND-005/run.sh
```

The runner starts and stops its own Anvil instance, pins the fork to block `11641625`, and exits nonzero if any assertion fails.

### What the command proves

The command performs three independent proof methods:

1. `cacheReachability.test.tsx` exercises the production `useV1Names` hook and proves that an already-mounted migration page retains the former manager and resolver after the modeled indexed response changes.
2. `probe.test.ts` classifies a stale indexed domain and ABI-decodes the exact production migration, role-grant, and verification-expectation fields.
3. `stateful.test.ts` executes the production-built HCA call on a pinned local fork. It checks current V1 pre-state and V2 receipt-block post-state, resolves the name, verifies the role event and cleanup persistence, exercises the attacker role, and compares a fresh-plan control.

### Step-by-step exploit walkthrough

1. While legitimately managing a V1 name, the attacker sets a custom resolver whose ETH record returns the attacker address.
2. Manager obtains or retains that manager and resolver state for migration.
3. The victim reclaims the V1 Registry manager, installs a safe resolver, and writes the victim ETH address.
4. The victim completes the still-stale normal migration plan.
5. Manager migrates ownership to the victim but installs the attacker's resolver and grants the former manager `ROLE_SET_RESOLVER`.
6. Receipt verification accepts those stale values as success, while Universal Resolver returns the attacker payment address.
7. The workflow revokes the HCA operator approval, but the new former-manager role survives. A separate attacker transaction then uses that role to repoint the resolver.

### Fixture privilege separation

Local-fork impersonation is used only to construct eligible synthetic fixture names:

* The V1 BaseRegistrar owner is impersonated once to authorize the test victim as a local registrar controller, after which the victim mints the two V1 fixture registrations.
* The V2 ETHRegistrar is impersonated only to create the `RESERVED` labels expected by the deployed testnet migration controller. Real premigrated eligible names already have this reservation state.
* No impersonation is used for the victim's reclaim, safe-resolver update, HCA approval, migration execution, attacker role grant, approval cleanup, Universal Resolver read, or attacker follow-up transaction.

The vulnerable transaction is the exact production-built HCA outer call signed by the victim fixture. Setup privileges neither authorize the attacker role nor bypass the contract checks being tested.

### Expected secure result

At submission, Manager should bind the migration to the current V1 Registry manager and resolver or abort for renewed review. The final V2 resolver should remain safe, the former manager should have no resolver-management role, resolution should return the victim address, and attacker `setResolver` should revert.

### Actual vulnerable result

Clean pinned runs emit the following security-relevant state:

```
preSubmitBlock=11641643
staleReceiptBlock=11641644
freshReceiptBlock=11641645
cleanupReceiptBlock=11641646
staleFinalResolver=0xeAc0a96ab8702B22731D8f5fB8e92A5Ab84f78A2
staleResolvedAddress=0x1f56e73362f9ceda7a6c190e374361b9b0edc2ec
hcaApprovedAfterCleanup=false
attackerRoleAfterCleanup=true
freshFinalResolver=0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0
freshResolvedAddress=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

Before submission, assertions prove that the victim owns both V1 registrations and Registry nodes, both use the safe resolver, the attacker cannot change the V1 resolver, and the attacker has no V2 resolver role.

At the stale receipt block, assertions prove that the victim is V2 owner, the attacker resolver is active, and an `EACRolesChanged` event changed the attacker role from absent to present. Production receipt verification passes. After the victim revokes the HCA approval, the attacker role remains and successfully changes the stale name's resolver. The independently rebuilt fresh-plan control has no attacker role, resolves to the victim, and rejects attacker `setResolver`.

### Verification record

* Clean pinned runner: exit 0; five Manager test files/79 tests, production cache-reachability test 1/1, ABI probe 1/1, and stateful fork test 1/1.
* Independent rerun: a reviewer who did not author the harness ran the unchanged command from an isolated archive of competition commit `1c9b47f`; all tests passed with the same security-relevant state.
* Current-state rerun: the same command passed against local Sepolia fork block `11652562`, timestamp `2026-09-07T06:57:00Z`, hash `0x972a464b81baac77a63de8dd59832a39931f450d959d54326abcf6b61a9b251d`.
* Limitation: the stateful PoC deterministically supplies the prior indexed object to the production builder rather than waiting for a public subgraph to lag. The separate hook test proves that the production page retains its prior result while mounted after the modeled source response changes. Practical timing still depends on the ordinary page lifecycle and index propagation.

### PoC file manifest

The secret Gist contains all seven required files:

* `README.md`
* `run.sh`
* `cacheReachability.test.tsx`
* `stateful.test.ts`
* `probe.test.ts`
* `foundry.toml`
* `AttackerResolver.sol` (place at `src/AttackerResolver.sol`)

Secret Gist: <https://gist.github.com/vstorchak/1cc51fbfd74e4b72aee613ffefaa0c71>
