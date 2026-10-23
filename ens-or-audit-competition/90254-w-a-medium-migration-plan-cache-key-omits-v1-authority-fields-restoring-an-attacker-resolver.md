# 90254 w a medium migration plan cache key omits v1 authority fields restoring an attacker resolver

**Submitted on Aug 24th 2026 at 02:19:53 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90254
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager caches each migration gas plan by wallet, HCA, and selected domain IDs, but not by the V1 rows' registry owners or resolvers. When a refreshed row for the same name shows that a victim revoked an old manager and replaced an attacker-controlled resolver, the cache key does not change and Manager keeps the pre-revocation plan. For a wallet admitted to the migration rollout, the approved transaction can therefore restore the attacker resolver, grant the revoked address V2 `ROLE_SET_RESOLVER`, and accept both values in post-state verification.

## Vulnerability Details

### Root cause: the migration-plan cache key omits V1 authority fields

[`GET_NAMES_QUERY`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L48) obtains the V1 registry owner, resolver, and registrant from the subgraph:

```graphql
resolver { address }
owner { id }
registrant { id }
```

For the normal return-to-tab reproduction, the source query is refetched too. [`useV1Names` nominally supplies a five-minute `staleTime`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23), but [`resultQueryOptions` destructures `rawStaleTime` and never includes it in the returned options](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/utils/src/tanstack-query/neverthrow/query.ts#L120-L147). Its effective stale time is therefore [Manager's explicitly configured query default of `0`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/router.tsx#L32-L36), so focus starts the V1-source and gas-plan refetches together. This wrapper-option loss explains the production focus transition; this report's security root remains the Manager plan key and missing authority validation, because even the newly current row does not schedule a follow-on plan build.

The response can be correct when fetched. The in-scope cache bug occurs afterward. [`useMigrationGasEstimate`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109) builds and stores the complete `MigrationPlan`, but its query identity includes only the selected domain IDs:

```ts
const domains = selectDomainsFromNames(v1Names, selectedNames)
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')

if (selectionRevisionRef.current.domainIds !== domainIds) {
  selectionRevisionRef.current = {
    domainIds,
    revision: selectionRevisionRef.current.revision + 1,
  }
}

const query = useQuery({
  queryKey: [
    'migration-gas-estimate',
    ownerAddress?.toLowerCase() ?? '',
    hcaAddress?.toLowerCase() ?? '',
    domainIds,
    selectionRevisionRef.current.revision,
  ],
  staleTime: 0,
  queryFn: async () => {
    // ...
    const plan = await buildMigrationPlan({ domains, /* ... */ })
    // ...
    return { estimate, plan }
  },
})
```

For a given name, changing `domain.owner.id` from the former manager to the victim and `domain.resolver.address` from the attacker resolver to a safe resolver changes neither `domainIds` nor `selectionRevisionRef`. A `v1Names` refresh therefore rerenders the hook with current authority but addresses the same React Query entry. Changing the query-function closure without changing the key does not create a new query or rebuild the plan from the new row; `staleTime: 0` does not make React Query refetch on every render. The first inline PoC also models a normal focus return: both source and plan queries refetch, the plan refetch has already captured the old row, and the later current-row rerender starts no third build. The source cache finishes current while the ready plan finishes old.

[`useMigrationGasEstimate` returns the prior `query.data.plan`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L128-L140). It neither fingerprints `owner`/`resolver` in the key nor performs an equality check before exposing that plan as `ready`. It checks `query.isPending`, not `query.isFetching`, so old data also remains actionable while a same-key background refetch is running.

The locally integrated migration package then converts these indexed values into transaction authority. [`classifyName`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L135-L185) copies `domain.resolver.address`. For an unknown/custom resolver, [`resolverStrategyFor`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L65-L89) chooses `keep-v1`. For an unwrapped `.eth`, it also treats an indexed registry owner different from the registrant as the manager to restore:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null
// ...
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

There is no current `ENSRegistry.owner(node)` or `ENSRegistry.resolver(node)` read in this classification. Manager's in-scope [`buildMigrationPlan`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380) passes the fetched domains through `classifyNames` and directly into the atomic builder. `packages/migration` supplies dependency output; the scoped defect is Manager's failure to live-validate that output before building and submitting security-critical calldata.

Preserving a custom resolver and legacy manager is intentional; restoring a value the owner has already replaced or revoked is not. The [release notes](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md#L58-L76) promise verification, custom-resolver preservation, and manager restoration. Those promises require the current authoritative V1 values, not an older cached version.

### On-chain eligibility covers the token owner, but not current V1 authority

Manager's in-scope [`computeMigrationPreflight`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L260) performs approval, HCA, route, resolver-deployment, record-inventory, and bytecode checks, but never revalidates the V1 registry owner or resolver. The separate on-chain eligibility routine checks BaseRegistrar/NameWrapper token ownership, frozen wrapper approvals, and V2 reservation status, but [also omits both registry fields](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/preflightChecks.ts#L47-L179). That eligibility result is itself [keyed only by owner and domain IDs and cached for 30 minutes](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationEligibility.ts#L8-L28). In this exploit the victim keeps the BaseRegistrar token and the V2 reservation does not change, so the ordinary eligibility check remains valid while the plan's V1 manager and resolver have become obsolete.

The execution-time rebuild in [`buildNextAtomicBatch`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L963) refreshes HCA resolver readiness and direct migration routes, but reuses `remaining`, the classified names from the preview plan. A simulation checks executability, not equality with current V1 authority; none of the built calls asserts that equality.

### The stale resolver and manager are embedded in the exact wallet calldata

[`resolverFor`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50) returns the indexed custom resolver for `keep-v1`. [`buildNameExecution`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L453-L491) puts it directly in `Migration.Data`:

```ts
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

[`buildMigrationHelperCall`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationHelperCall.ts#L60-L86) ABI-encodes this in `MigrationHelper.migrate(...)`, nested inside `StandaloneHCA.executeByOwner(...)`.

For the indexed former manager, [`buildNameExecution`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L625) appends another inner execution. [`buildRoleGrantCall`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) encodes:

```ts
ETHRegistry.grantRoles(
  labelToCanonicalId(name.label),
  1n << 24n, // ROLE_SET_RESOLVER
  name.managerAddress, // stale former manager
)
```

Manager asks the wallet to make the HCA an ETHRegistry operator when manager restoration is required. The confirmation UI only says [“Restore the existing managers for your names”](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L90); it does not show the restored address or compare it with live V1 state. After migration, [`buildMigrationOperatorApprovalRevocationCall`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L354-L366) only encodes `setApprovalForAll(HCA, false)`. It does not revoke the role that the HCA just granted to the former manager.

### The deployed migration chain consumes the stale resolver and authorizes the role grant

The ENSv2 contracts are consequence context rather than the report target. The frozen app's [contract manifest documents the exact provenance](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/manifest.ts#L1-L10): the core ENSv2 addresses match the remediated contracts-v2 PR #388 deployment at `8d1c89350729f87b3968cf41ef3d683951444a44`, while the HCA-aware `MigrationHelper` was deployed separately from PR #402. The [pinned Sepolia entries repeat that split and identify the helper address](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/manifest.ts#L83-L98).

At PR #402 commit `0717b0b6f1867f43078f19115dd5dfa4f35426ab`, the deployed [`MigrationHelper.migrate`](https://github.com/ensdomains/contracts-v2/blob/0717b0b6f1867f43078f19115dd5dfa4f35426ab/contracts/src/migration/MigrationHelper.sol#L101-L124) resolves the certified HCA caller to its EOA owner for the approval check and forwards `abi.encode(md)` unchanged in `BaseRegistrar.safeTransferFrom`. At PR #388 commit `8d1c89350729f87b3968cf41ef3d683951444a44`, [`UnlockedMigrationController.onERC721Received`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/migration/UnlockedMigrationController.sol#L92-L120) decodes that same `md`, clears the V1 record, and calls `_inject(md)`. [`_inject`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/migration/UnlockedMigrationController.sol#L153-L165) registers `md.resolver` verbatim:

```solidity
ETH_REGISTRY.register(
    md.label,
    md.owner,
    md.subregistry,
    md.resolver,
    REGISTRATION_ROLE_BITMAP,
    0
);
```

There is no contract-side comparison with the resolver authoritative when the victim initiated the flow. The report and PoC focus on this unwrapped `.eth` route.

The following HCA role grant is executable, not merely valid ABI bytes. [`REGISTRATION_ROLE_BITMAP`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registrar/ETHRegistrar.sol#L17-L23) gives the newly registered victim both `ROLE_SET_RESOLVER` and `ROLE_SET_RESOLVER_ADMIN`. Manager first asks the victim to approve the HCA as an ETHRegistry operator. [`PermissionedRegistry._getRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol#L571-L590) then combines an approved operator's roles with the owner's roles, satisfying the [admin-role check on `grantRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L117-L130). Because the migration execution precedes the role-grant execution in the same HCA batch, the HCA can grant the old manager `1 << 24` after registration.

[`RegistryRolesLib`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/libraries/RegistryRolesLib.sol#L33-L41) defines `ROLE_SET_RESOLVER = 1 << 24`. [`PermissionedRegistry.setResolver`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol#L141-L155) accepts that role and stores the caller's resolver. Manager's later HCA-operator cleanup does not revoke the former manager's per-name role. It therefore gives the attacker continuing control over the active registration until the owner separately revokes the role or the registration lifecycle ends; this report does not claim irreversibility.

### Post-state verification blesses both attacker values

[`buildNameExecution`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L544-L625) creates a `name-resolver` expectation for the same stale `expectedResolver` and a `manager-role` expectation for the same stale manager. [`verifyAtomicMigrationBatch`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L249-L275) reads the installed resolver and compares it with that stale expectation. Its [`manager-role` branch](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353) confirms that the stale attacker received the role.

The checks therefore prove that the bad values were installed rather than compare them with current V1 authority.

### End-to-end attack path

**Rollout and ordinary-flow prerequisites.** [`/migration` admits only wallets for which the server-side PostHog `migration` flag returns `true`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/migration.tsx#L16-L25); a false flag redirects to the dashboard. This report does not claim the attacker can enable that flag. It applies to users admitted to the intended rollout, which the [release notes describe as feature-gated](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md#L67-L77) and the competition's selected Critical impact expressly covers. The name must also meet the normal route's eligibility: an unwrapped, unexpired `.eth`, premigrated to a `RESERVED` V2 entry, whose BaseRegistrar token remains owned by `VICTIM`.

The state transition is:

| Moment                           | BaseRegistrar owner | V1 Registry owner                    | Authoritative V1 resolver | Manager plan / V2 result                                                                 |
| -------------------------------- | ------------------- | ------------------------------------ | ------------------------- | ---------------------------------------------------------------------------------------- |
| Initial plan built               | `VICTIM`            | `REVOKED_ATTACKER` (then authorized) | `OLD_ATTACKER_RESOLVER`   | Ready plan contains the then-current values                                              |
| After reclaim and V1-row refresh | `VICTIM`            | `VICTIM`                             | `CURRENT_SAFE_RESOLVER`   | Same domain ID keeps the cached pre-revocation plan                                      |
| Decoded Manager calldata         | `VICTIM`            | `VICTIM`                             | `CURRENT_SAFE_RESOLVER`   | `md.resolver = OLD_ATTACKER_RESOLVER`; `grantRoles(..., REVOKED_ATTACKER)`               |
| Expected V2 post-state           | —                   | —                                    | —                         | Owner is `VICTIM`; resolver is `OLD_ATTACKER_RESOLVER`; attacker has `ROLE_SET_RESOLVER` |

1. The victim is the BaseRegistrar registrant of `alice.eth`. The attacker previously had legitimate per-name V1 manager authority and used it to configure a compliant attacker-controlled custom resolver.
2. Manager receives a then-correct row with `owner = REVOKED_ATTACKER` and `resolver = OLD_ATTACKER_RESOLVER`. `useMigrationGasEstimate` builds a ready plan for that name.
3. For the simplest deterministic mounted-route sequence, the victim uses a second device or interface while the original Manager route stays mounted. They call `reclaim`, making themselves the authoritative V1 registry owner, and set `CURRENT_SAFE_RESOLVER`. The former manager now has no V1 authority.
4. The exploit does not require the V1 row to remain old. Give Manager the stronger assumption that its `v1Names` prop has refreshed to `owner = VICTIM` and `resolver = CURRENT_SAFE_RESOLVER`. Because the domain ID is unchanged, the plan query key/revision do not change and this rerender does not start a new plan build. The first PoC proves the old ready plan remains.
5. Automatic focus/reconnect/remount or manual invalidation can eventually refetch the gas plan. The exact boundary is therefore that the victim clicks Upgrade before a gas-plan refetch using the current-row closure **completes**. During background refetch, the hook still returns old `query.data` as `ready`. [`SelectNamesStep` disables Upgrade for `isPending` or a non-ready gas status, not background `isFetching`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/SelectNamesStep.tsx#L46-L61), and [`useNameSelection` retains the selection because the selectable name is unchanged](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useNameSelection.ts#L33-L49). [`handleBeginUpgrade` then captures `gasEstimate.plan`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282); execution rechecks approval/HCA state, but not current V1 manager/resolver state.
6. After revocation, the victim approves the normal prompts, including the generic “Restore the existing managers” ETHRegistry operator approval for the HCA and the atomic upgrade transaction. The UI does not identify `REVOKED_ATTACKER`. The nested helper calldata registers the V2 name to the victim but assigns `OLD_ATTACKER_RESOLVER`; the next execution grants the revoked attacker V2 `ROLE_SET_RESOLVER`.
7. The transaction executes. The post-state verifier uses the same old plan values as its expectations, so matching attacker-controlled state is accepted. The resolver can return the attacker's address; even after repair, the restored V2 role lets the former manager replace it until the owner separately revokes the role or the registration lifecycle ends.

The prerequisites are narrow and explicit: ordinary rollout/eligibility; a formerly authorized, now-revoked per-name delegate; a ready plan built before revocation; the victim's normal approvals after revocation; and the Upgrade click/plan capture before a current-row gas-plan refetch completes. The attacker need not transact during migration or retain any current V1 name authority; they only still control their own old resolver contract. No indexer lag/compromise, stolen key, or privileged project address is required. The first PoC finishes with a refreshed current V1 source row and still observes the old plan.

### Distinction from the published known issues

I compared this root cause with the complete known-issue list published for the competition. The closest entries are:

* **WEB-806:** migrated subname records are not visible in Manager. This report uses a present, unwrapped `.eth` name and concerns authority-bearing resolver/manager calldata, not missing profile display.
* **QA-01:** some role combinations around transfers break follow-up actions. The published text expressly distinguishes a new security consequence that grants unintended authority. Here migration grants a revoked attacker `ROLE_SET_RESOLVER` and accepts that state.

No published entry describes a migration-plan cache key that omits V1 authority fields or the failure to bind the captured plan to current on-chain registry owner/resolver state.

### Recommended remediation

Before any approval or atomic migration prompt:

1. Include every authority-bearing input—or a canonical digest of the full selected `V1Domain` rows—in the migration-plan query key. An owner/resolver change must create a new query and put the UI back into a loading/review state.
2. Read current V1 `ENSRegistry.owner(node)` and `resolver(node)` for every selected name at one explicit block before the first wallet prompt.
3. Derive `managerAddress`, `resolverStrategy`, and `Migration.Data.resolver` only from those live results. If they differ from preview, invalidate the plan and require review.
4. At execution-time batch rebuild, re-read the fields and fail closed on any drift.
5. Bind the final values on-chain in the same atomic transaction. Extend an audited helper/controller, or use an audited assertion call, to revert unless current V1 owner/resolver match the snapshot used to build the migration. A client-side read alone leaves a mempool TOCTOU.
6. Bind post-state verification to that final asserted snapshot rather than to values copied from an older plan.

## Impact Details

**Selected in-scope impact: Critical — “Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.”**

The demonstrated migration flow assigns the victim's V2 registry resolver to an attacker-controlled contract. That resolver can return the attacker's ETH address, so users and integrations resolving the victim's name can send funds to the attacker. The same flow grants the revoked address an on-chain V2 `ROLE_SET_RESOLVER`, which survives Manager's HCA cleanup and lets the attacker restore malicious resolution after a repair.

This report demonstrates the resolver-hijacking example in the selected impact, not permanent bricking. The victim remains the token owner, so it does not claim irreversible ownership theft. An informed owner can repair the resolver and revoke the role. That does not remove the exact listed Critical impact: the competition explicitly classifies a migration that assigns the resolver to an attacker as Critical, and the attacker has working V2 authority that did not exist before this Manager flow.

This does not depend on a fixed subgraph-cache window. An authority-only refresh preserves the domain ID, so the input change alone does not create or refetch a migration-plan query. Default focus/reconnect/remount events or manual invalidation can refetch it; the old plan remains actionable until a refetch built from the current row completes because background fetching does not clear `query.data` or the hook's `ready` status. The PoC demonstrates the exact old-row → current-row transition and no plan rebuild from the current row, even when both queries automatically refetch on focus. Affected users are those who remove an old delegate or malicious resolver after previewing an upgrade and click Upgrade before that boundary.

## References

* [ENS competition scope, exact impacts, and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Manager asset at the frozen competition commit](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager)
* [Migration-plan query identity and ready-plan return](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L140)
* [Upgrade captures the cached plan](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282)
* [Resolver/manager classification](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L125-L185)
* [Calldata and verification-expectation construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L453-L625)
* [Post-state verifier](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L249-L353)
* [Frozen deployed-contract provenance](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/manifest.ts#L1-L10)
* [PR #402 helper forwards `md`](https://github.com/ensdomains/contracts-v2/blob/0717b0b6f1867f43078f19115dd5dfa4f35426ab/contracts/src/migration/MigrationHelper.sol#L101-L124)
* [PR #388 controller consumes `md.resolver`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/migration/UnlockedMigrationController.sol#L92-L165)
* [`ROLE_SET_RESOLVER` enforcement](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol#L141-L155)

## Proof of Concept

## Proof of Concept

### What the PoC proves

The complete one-file patch below adds two deterministic local Vitest tests; it does not alter production logic:

1. **Production focus-refresh test:** composes the real `useV1Names` and `useMigrationGasEstimate` hooks. It first builds a ready plan from the pre-revocation row, makes the mocked subgraph service return the current victim/safe-resolver row, and triggers a normal focus refetch. Both queries refetch. The test proves the V1 source finishes current, while the concurrent second `buildMigrationPlan` call captured the old row, no third/current-row build occurs, and the gas hook finishes `ready` with the old plan.
2. **Calldata-builder test:** gives that pre-revocation row to the production `buildMigrationPlan` builder with deterministic preflight and direct-route fixtures. A fail-closed `readContract` stub proves this configured builder path performs no current V1 registry manager/resolver read. The test decodes the exact wallet-facing `StandaloneHCA.executeByOwner` calldata and proves it contains:
   * `MigrationHelper.migrate` with `owner = VICTIM` but `resolver = OLD_ATTACKER_RESOLVER`; and
   * `ETHRegistry.grantRoles(resource, 1 << 24, REVOKED_ATTACKER)`.
3. The second test also proves the plan constructs post-state expectations for both attacker values, the prerequisite approval is `setApprovalForAll(HCA, true)`, and cleanup is only `setApprovalForAll(HCA, false)` with no attacker-role revocation.

The tests are report-supplied scaffolding for the service behind the disclosed rollout gate; they do not bypass or test that gate. They import production files from the frozen checkout, contact no chain, use no private keys, and require no attachment. The focus test mocks the subgraph service plus expensive plan/preflight/fee dependencies while retaining both production hooks and React Query behavior. The calldata test uses the production plan and calldata builders; it supplies deterministic preflight/route inputs rather than executing the later state machine or an ENSv2 chain.

The calldata test's production imports initialize Reown code. It therefore installs fail-closed stubs for `fetch`, `WebSocket`, `XMLHttpRequest`, and `EventSource` before dynamically importing those modules. Recognized feature/telemetry `fetch` calls receive a local response; any unexpected URL or transport fails the test. No request leaves the test process.

### Reproduction — one inline patch against the scoped repository

Prerequisites: Node.js 22 (the run below used `v22.19.0`).

1. Clone the competition repository, check out the exact target commit, and install its pinned dependencies:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
```

2. In the repository root, create `poc-stale-v1-authority.patch` and copy the complete diff from the end of this report into it.
3. Check and apply the patch, then run only the two PoC tests:

```bash
git apply --check poc-stale-v1-authority.patch
git apply poc-stale-v1-authority.patch
pnpm --filter manager exec vitest run \
  src/features/migration/hooks/staleAuthorityPlan.poc.test.tsx \
  src/features/migration/service/staleAuthority.poc.test.ts \
  --reporter verbose
```

### Expected result

```
✓ ...staleAuthorityPlan.poc.test.tsx > PoC: a normal focus refresh preserves the stale authority plan > ends with a current V1 row but an old plan after both queries refetch
✓ ...staleAuthority.poc.test.ts > PoC: stale V1 authority is restored during v1 -> v2 migration > embeds the attacker resolver and re-grants the revoked manager in exact wallet calldata

Test Files  2 passed (2)
Tests       2 passed (2)
```

Together with the cited production capture/execution source, the tests provide compositional evidence for the complete data flow: the normal refresh ends with a current V1 source but an old ready plan; the downstream production builders turn that plan into the attacker resolver/role calldata and matching verification/cleanup state.

### Complete single-file patch — `poc-stale-v1-authority.patch`

Copy this file exactly as shown:

```diff
diff --git a/apps/manager/src/features/migration/hooks/staleAuthorityPlan.poc.test.tsx b/apps/manager/src/features/migration/hooks/staleAuthorityPlan.poc.test.tsx
new file mode 100644
index 0000000..a5776ba
--- /dev/null
+++ b/apps/manager/src/features/migration/hooks/staleAuthorityPlan.poc.test.tsx
@@ -0,0 +1,158 @@
+import {
+  focusManager,
+  QueryClient,
+  QueryClientProvider,
+} from '@tanstack/react-query'
+import { act, renderHook, waitFor } from '@testing-library/react'
+import { ok } from 'neverthrow'
+import type { ReactNode } from 'react'
+import { getAddress, labelhash, namehash } from 'viem'
+import { afterEach, describe, expect, it, vi } from 'vitest'
+
+import type { MigrationPlan } from '../service/buildMigrationPlan'
+import type { V1Domain } from '../service/v1SubgraphClient'
+
+const mocks = vi.hoisted(() => ({
+  buildMigrationPlan: vi.fn(),
+  ensurePreflight: vi.fn(),
+  estimateMigrationGasCost: vi.fn(),
+  getV1NamesForAddress: vi.fn(),
+  publicClient: {},
+  ownerAddress: '0x00000000000000000000000000000000000000a1',
+}))
+
+vi.mock('wagmi', () => ({
+  useConnection: () => ({ address: mocks.ownerAddress }),
+  usePublicClient: () => mocks.publicClient,
+}))
+vi.mock('@/lib/smart-account', () => ({
+  useSmartAccountContext: () => ({ ownerAddress: mocks.ownerAddress }),
+}))
+vi.mock('./useMigrationPreflight', () => ({
+  useMigrationPreflight: () => ({ ensure: mocks.ensurePreflight }),
+}))
+vi.mock('@/features/migration/service/buildMigrationPlan', () => ({
+  buildMigrationPlan: mocks.buildMigrationPlan,
+}))
+vi.mock('@/features/migration/service/estimateMigrationGasCost', () => ({
+  estimateMigrationGasCost: mocks.estimateMigrationGasCost,
+}))
+vi.mock('@/features/migration/service/v1SubgraphClient', () => ({
+  getV1NamesForAddress: mocks.getV1NamesForAddress,
+}))
+
+import { useMigrationGasEstimate } from './useMigrationGasEstimate'
+import { useV1Names } from './useV1Names'
+
+const VICTIM = getAddress('0x00000000000000000000000000000000000000a1')
+const REVOKED_ATTACKER = getAddress(
+  '0x00000000000000000000000000000000000000b1',
+)
+const HCA = getAddress('0x00000000000000000000000000000000000000c1')
+const OLD_ATTACKER_RESOLVER = getAddress(
+  '0x0000000000000000000000000000000000000bad',
+)
+const CURRENT_SAFE_RESOLVER = getAddress(
+  '0x0000000000000000000000000000000000000afe',
+)
+const NAME = 'stale-authority.eth'
+const LABEL = 'stale-authority'
+
+const row = (owner: string, resolver: string): V1Domain => ({
+  id: namehash(NAME),
+  labelName: LABEL,
+  labelhash: labelhash(LABEL),
+  name: NAME,
+  resolver: { address: resolver },
+  owner: { id: owner },
+  registrant: { id: VICTIM },
+  wrappedOwner: null,
+  parent: { name: 'eth', wrappedDomain: null },
+  registration: { expiryDate: '9999999999' },
+  wrappedDomain: null,
+})
+
+const STALE_ROW = row(REVOKED_ATTACKER, OLD_ATTACKER_RESOLVER)
+const CURRENT_ROW = row(VICTIM, CURRENT_SAFE_RESOLVER)
+
+afterEach(() => {
+  focusManager.setFocused(undefined)
+  vi.clearAllMocks()
+})
+
+describe('PoC: a normal focus refresh preserves the stale authority plan', () => {
+  it('ends with a current V1 row but an old plan after both queries refetch', async () => {
+    const queryClient = new QueryClient({
+      defaultOptions: { queries: { retry: false, staleTime: 0 } },
+    })
+    const wrapper = ({ children }: { children: ReactNode }) => (
+      <QueryClientProvider client={queryClient}>
+        {children}
+      </QueryClientProvider>
+    )
+
+    mocks.getV1NamesForAddress.mockReturnValue(ok([STALE_ROW]))
+    mocks.ensurePreflight.mockResolvedValue({})
+    mocks.buildMigrationPlan.mockImplementation(
+      async ({ domains }: { domains: readonly V1Domain[] }) =>
+        ({ sourceDomain: domains[0] }) as unknown as MigrationPlan,
+    )
+    mocks.estimateMigrationGasCost.mockResolvedValue({
+      status: 'ready',
+      gasUnits: 1n,
+      feeWei: 1n,
+      feePerGasWei: 1n,
+      transactionCount: 1,
+    })
+
+    const { result } = renderHook(() => {
+      const { data: v1Names = [] } = useV1Names()
+      const gasEstimate = useMigrationGasEstimate({
+        ownerAddress: VICTIM,
+        hcaAddress: HCA,
+        selectedNames: [NAME],
+        v1Names,
+      })
+      return { gasEstimate, v1Names }
+    }, { wrapper })
+
+    await waitFor(() => expect(result.current.gasEstimate.status).toBe('ready'))
+    expect(mocks.buildMigrationPlan).toHaveBeenCalledTimes(1)
+
+    // The victim's reclaim and resolver update are now indexed. A focus event
+    // starts the stale V1-names and gas-plan queries together. The gas query's
+    // already-started closure still contains STALE_ROW. When CURRENT_ROW causes
+    // a rerender, its unchanged gas query key does not schedule a third build.
+    mocks.getV1NamesForAddress.mockReturnValue(ok([CURRENT_ROW]))
+    await act(async () => {
+      focusManager.setFocused(false)
+      focusManager.setFocused(true)
+    })
+
+    await waitFor(() => {
+      expect(mocks.getV1NamesForAddress).toHaveBeenCalledTimes(2)
+      expect(mocks.buildMigrationPlan).toHaveBeenCalledTimes(2)
+      expect(queryClient.isFetching()).toBe(0)
+    })
+
+    expect(result.current.v1Names[0]).toMatchObject({
+      owner: { id: VICTIM },
+      resolver: { address: CURRENT_SAFE_RESOLVER },
+    })
+    const secondBuildDomains = mocks.buildMigrationPlan.mock.calls[1]?.[0]
+      .domains as readonly V1Domain[]
+    expect(secondBuildDomains[0]).toBe(STALE_ROW)
+
+    expect(result.current.gasEstimate.status).toBe('ready')
+    if (result.current.gasEstimate.status !== 'ready') {
+      throw new Error('plan is not ready')
+    }
+    const captured = result.current.gasEstimate.plan as MigrationPlan & {
+      sourceDomain: V1Domain
+    }
+    expect(captured.sourceDomain.owner.id).toBe(REVOKED_ATTACKER)
+    expect(captured.sourceDomain.resolver?.address).toBe(
+      OLD_ATTACKER_RESOLVER,
+    )
+  })
+})
diff --git a/apps/manager/src/features/migration/service/staleAuthority.poc.test.ts b/apps/manager/src/features/migration/service/staleAuthority.poc.test.ts
new file mode 100644
index 0000000..0b6d216
--- /dev/null
+++ b/apps/manager/src/features/migration/service/staleAuthority.poc.test.ts
@@ -0,0 +1,275 @@
+import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
+import {
+  type Address,
+  decodeFunctionData,
+  getAddress,
+  labelhash,
+  namehash,
+  parseAbi,
+  zeroAddress,
+} from 'viem'
+import { sepolia } from 'viem/chains'
+import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
+
+import type { MigrationPreflight } from './computeMigrationPreflight'
+import type { DirectMigrationRoute } from './directMigrationRoutes'
+import type { V1Domain } from './v1SubgraphClient'
+
+let ETH_REGISTRY_V2_ABI: (typeof import('../contracts/abis'))['ETH_REGISTRY_V2_ABI']
+let MIGRATION_HELPER_ABI: (typeof import('../contracts/abis'))['MIGRATION_HELPER_ABI']
+let OPERATOR_APPROVAL_ABI: (typeof import('../contracts/abis'))['OPERATOR_APPROVAL_ABI']
+let V2_CONTRACTS: (typeof import('../contracts/addresses'))['V2_CONTRACTS']
+let buildMigrationPlan: (typeof import('./buildMigrationPlan'))['buildMigrationPlan']
+let buildMigrationApprovalCall: (typeof import('./migrationApprovals'))['buildMigrationApprovalCall']
+let buildMigrationOperatorApprovalRevocationCall: (typeof import('./migrationApprovals'))['buildMigrationOperatorApprovalRevocationCall']
+let migrationApprovalForId: (typeof import('./migrationApprovals'))['migrationApprovalForId']
+let fetchStub: ReturnType<typeof vi.fn>
+let websocketStub: ReturnType<typeof vi.fn>
+let xhrStub: ReturnType<typeof vi.fn>
+let eventSourceStub: ReturnType<typeof vi.fn>
+
+const requestUrl = (input: string | URL | Request): string =>
+  typeof input === 'string' || input instanceof URL ? String(input) : input.url
+
+const isExpectedReownUrl = (url: string): boolean =>
+  url.startsWith('https://api.web3modal.org/') ||
+  url.startsWith('https://pulse.walletconnect.org/')
+
+beforeAll(async () => {
+  // Production migration imports initialize Reown. Install fail-closed stubs
+  // before importing them so the PoC cannot contact any external service.
+  const blockedTransport = (kind: string) =>
+    vi.fn(function blockNetworkTransport(input?: unknown): never {
+      throw new Error(`PoC blocked unexpected ${kind} request: ${String(input)}`)
+    })
+  websocketStub = blockedTransport('WebSocket')
+  xhrStub = blockedTransport('XMLHttpRequest')
+  eventSourceStub = blockedTransport('EventSource')
+  vi.stubGlobal('WebSocket', websocketStub)
+  vi.stubGlobal('XMLHttpRequest', xhrStub)
+  vi.stubGlobal('EventSource', eventSourceStub)
+  fetchStub = vi.fn(async (input: string | URL | Request) => {
+    const url = requestUrl(input)
+    if (!isExpectedReownUrl(url)) {
+      throw new Error(`PoC blocked unexpected fetch: ${url}`)
+    }
+    return new Response(JSON.stringify({}), {
+      status: 200,
+      headers: { 'Content-Type': 'application/json' },
+    })
+  })
+  vi.stubGlobal('fetch', fetchStub)
+
+  const [abis, addresses, planModule, approvalModule] = await Promise.all([
+    import('../contracts/abis'),
+    import('../contracts/addresses'),
+    import('./buildMigrationPlan'),
+    import('./migrationApprovals'),
+  ])
+  ETH_REGISTRY_V2_ABI = abis.ETH_REGISTRY_V2_ABI
+  MIGRATION_HELPER_ABI = abis.MIGRATION_HELPER_ABI
+  OPERATOR_APPROVAL_ABI = abis.OPERATOR_APPROVAL_ABI
+  V2_CONTRACTS = addresses.V2_CONTRACTS
+  buildMigrationPlan = planModule.buildMigrationPlan
+  buildMigrationApprovalCall = approvalModule.buildMigrationApprovalCall
+  buildMigrationOperatorApprovalRevocationCall =
+    approvalModule.buildMigrationOperatorApprovalRevocationCall
+  migrationApprovalForId = approvalModule.migrationApprovalForId
+
+  await vi.waitFor(() => expect(fetchStub).toHaveBeenCalled(), { timeout: 2_000 })
+  expect(
+    fetchStub.mock.calls
+      .map(([input]) => requestUrl(input as string | URL | Request))
+      .every(isExpectedReownUrl),
+  ).toBe(true)
+})
+
+afterAll(() => {
+  expect(websocketStub).not.toHaveBeenCalled()
+  expect(xhrStub).not.toHaveBeenCalled()
+  expect(eventSourceStub).not.toHaveBeenCalled()
+})
+
+const VICTIM: Address = getAddress(
+  '0x00000000000000000000000000000000000000a1',
+)
+const REVOKED_ATTACKER: Address = getAddress(
+  '0x00000000000000000000000000000000000000b1',
+)
+const HCA: Address = getAddress(
+  '0x00000000000000000000000000000000000000c1',
+)
+const OLD_ATTACKER_RESOLVER: Address = getAddress(
+  '0x0000000000000000000000000000000000000bad',
+)
+const CURRENT_SAFE_RESOLVER: Address = getAddress(
+  '0x0000000000000000000000000000000000000afe',
+)
+const NAME = 'stale-authority.eth'
+const LABEL = 'stale-authority'
+const ROLE_SET_RESOLVER = 1n << 24n
+const CURRENT_V1_MANAGER = VICTIM
+
+// This honest-but-stale subgraph/cache row predates the victim's reclaim and
+// resolver update. The current live V1 manager and token owner are both VICTIM,
+// and the current live V1 resolver is CURRENT_SAFE_RESOLVER.
+const staleDomain: V1Domain = {
+  id: namehash(NAME),
+  labelName: LABEL,
+  labelhash: labelhash(LABEL),
+  name: NAME,
+  resolver: { address: OLD_ATTACKER_RESOLVER },
+  owner: { id: REVOKED_ATTACKER },
+  registrant: { id: VICTIM },
+  wrappedOwner: null,
+  parent: { name: 'eth', wrappedDomain: null },
+  registration: { expiryDate: '9999999999' },
+  wrappedDomain: null,
+}
+
+const executeByOwnerAbi = parseAbi([
+  'function executeByOwner((address target,uint256 value,bytes callData)[] executions) payable',
+])
+
+describe('PoC: stale V1 authority is restored during v1 -> v2 migration', () => {
+  it('embeds the attacker resolver and re-grants the revoked manager in exact wallet calldata', async () => {
+    const route: DirectMigrationRoute = {
+      name: NAME,
+      receiver: V2_CONTRACTS.UnlockedMigrationController,
+      parentDependency: null,
+      expectedWrapperRegistry: null,
+      receiverReadiness: 'migration-controller',
+    }
+    const managerApproval = migrationApprovalForId({
+      id: 'eth-registry:hca',
+      hcaAddress: HCA,
+    })
+    if (managerApproval.kind !== 'operator') {
+      throw new Error('expected an operator approval')
+    }
+    const tokenApproval = migrationApprovalForId({
+      id: 'base-registrar:hca-token',
+      hcaAddress: HCA,
+      tokenId: BigInt(staleDomain.labelhash),
+    })
+    const preflight: MigrationPreflight = {
+      preExistingOwnedPermRes: null,
+      skipApprovalPhase: false,
+      skipFetchProfilesPhase: true,
+      baseRegistrarApproved: false,
+      nameWrapperApproved: true,
+      requiresManagerRestoration: true,
+      migrationApprovals: [tokenApproval, managerApproval],
+      directMigrationRoutes: new Map([[NAME, route]]),
+    }
+    const readContract = vi.fn(() => {
+      throw new Error('A live V1 manager/resolver read was attempted')
+    })
+    const publicClient = {
+      chain: sepolia,
+      readContract,
+    } as unknown as Parameters<typeof buildMigrationPlan>[0]['publicClient']
+
+    const plan = await buildMigrationPlan({
+      domains: [staleDomain],
+      hcaAddress: HCA,
+      migrationOwner: VICTIM,
+      publicClient,
+      preflight,
+    })
+
+    // The production plan never asks the live Registry for the current manager
+    // or resolver. It classifies both security-critical fields from staleDomain.
+    expect(readContract).not.toHaveBeenCalled()
+    expect(plan.classified).toHaveLength(1)
+    expect(plan.classified[0]).toMatchObject({
+      tokenHolder: VICTIM,
+      v1ResolverAddress: OLD_ATTACKER_RESOLVER,
+      resolverStrategy: 'keep-v1',
+      managerAddress: REVOKED_ATTACKER,
+    })
+    expect(plan.classified[0]?.v1ResolverAddress).not.toBe(CURRENT_SAFE_RESOLVER)
+    expect(plan.classified[0]?.managerAddress).not.toBe(CURRENT_V1_MANAGER)
+
+    // Decode the exact outer call presented to the victim's wallet/HCA.
+    const batch = plan.atomicBatches[0]
+    expect(batch).toBeDefined()
+    const decodedOuter = decodeFunctionData({
+      abi: executeByOwnerAbi,
+      data: batch!.outerCall.data,
+    })
+    expect(decodedOuter.functionName).toBe('executeByOwner')
+    const executions = decodedOuter.args[0]
+    expect(executions).toHaveLength(2)
+
+    // First inner call: MigrationHelper.migrate registers the victim as owner
+    // but restores the old attacker resolver instead of the safe live resolver.
+    expect(executions[0]?.target).toBe(V2_CONTRACTS.MigrationHelper)
+    const decodedMigrate = decodeFunctionData({
+      abi: MIGRATION_HELPER_ABI,
+      data: executions[0]!.callData,
+    })
+    expect(decodedMigrate.functionName).toBe('migrate')
+    expect(decodedMigrate.args[0]).toEqual([
+      {
+        label: LABEL,
+        owner: VICTIM,
+        subregistry: zeroAddress,
+        resolver: OLD_ATTACKER_RESOLVER,
+      },
+    ])
+
+    // Second inner call: the victim-approved HCA grants the revoked attacker
+    // V2 ROLE_SET_RESOLVER for the migrated name.
+    expect(executions[1]?.target).toBe(V2_CONTRACTS.ETHRegistry)
+    const decodedGrant = decodeFunctionData({
+      abi: ETH_REGISTRY_V2_ABI,
+      data: executions[1]!.callData,
+    })
+    expect(decodedGrant).toMatchObject({
+      functionName: 'grantRoles',
+      args: [
+        labelToCanonicalId(LABEL),
+        ROLE_SET_RESOLVER,
+        REVOKED_ATTACKER,
+      ],
+    })
+
+    // Verification is self-fulfilling: it expects both attacker values.
+    expect(batch?.verificationExpectations).toEqual(
+      expect.arrayContaining([
+        expect.objectContaining({
+          type: 'name-resolver',
+          expectedResolver: OLD_ATTACKER_RESOLVER,
+        }),
+        expect.objectContaining({
+          type: 'manager-role',
+          account: REVOKED_ATTACKER,
+          roleBitmap: ROLE_SET_RESOLVER,
+        }),
+      ]),
+    )
+
+    // Manager first approves the HCA, then cleanup only revokes that operator.
+    // It never revokes the role just granted to REVOKED_ATTACKER.
+    expect(
+      decodeFunctionData({
+        abi: OPERATOR_APPROVAL_ABI,
+        data: buildMigrationApprovalCall(managerApproval).data,
+      }),
+    ).toMatchObject({
+      functionName: 'setApprovalForAll',
+      args: [HCA, true],
+    })
+    const cleanup = buildMigrationOperatorApprovalRevocationCall(managerApproval)
+    expect(
+      decodeFunctionData({ abi: OPERATOR_APPROVAL_ABI, data: cleanup.data }),
+    ).toMatchObject({
+      functionName: 'setApprovalForAll',
+      args: [HCA, false],
+    })
+    expect(cleanup.data.toLowerCase()).not.toContain(
+      REVOKED_ATTACKER.slice(2).toLowerCase(),
+    )
+  })
+})
```
