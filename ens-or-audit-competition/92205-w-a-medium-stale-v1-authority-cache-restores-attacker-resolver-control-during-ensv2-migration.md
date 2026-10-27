# 92205 w a medium stale v1 authority cache restores attacker resolver control during ensv2 migration

**Submitted on Sep 8th 2026 at 08:37:54 UTC by @muhobaba for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92205
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

**Affected asset:** `apps/manager`

**Selected impact:** Hijacking or bricking an ENS name during a Manager V1-to-V2 migration by assigning its resolver to an attacker.

Mallory previously managed Alice's `victim.eth` name in the V1 ENS Registry or controlled its custom resolver. Alice revokes Mallory and replaces that resolver, but then approves a normal V1-to-V2 migration while the Manager app still holds the earlier V1 response in its five-minute client cache. The Manager confirms that Alice still owns the `.eth` registration token, but it never re-reads the current V1 Registry `owner(node)` or `resolver(node)`. It consequently constructs a migration that preserves Mallory's former resolver and/or grants Mallory V2 `ROLE_SET_RESOLVER`. Mallory does not steal Alice's wallet, session, or keys, and this report does not depend on compromising the subgraph or finding a bug in ENS contracts.

The narrow demonstrated primitive is wrong security-sensitive transaction construction: with current on-chain authority represented as Alice plus a safe resolver, the local PoC shows that the Manager makes no live authority read and encodes Mallory's cached resolver and role grant into the atomic migration batch. Either branch is independently security-relevant. Preserving Mallory's resolver immediately directs the migrated name to a resolver she controls; restoring `ROLE_SET_RESOLVER` lets her replace the resolver after migration. Alice remains the V2 owner and can recover if she notices, so this report does not claim permanent NFT theft. Until recovery, Mallory can control what the name resolves to.

This matches the competition's explicit Critical migration-hijack impact, including assigning a name's resolver to an attacker: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>. It also constructs transaction details that conflict with Alice's current authority choices. The likelihood is conditional on prior authority and a stale response at migration time, but the application itself creates a deterministic five-minute freshness window; an indexer fault is unnecessary.

I inspected the frozen source path from V1 data loading through classification, live eligibility checks, transaction construction, execution, and post-state verification. The offline PoC passes all five source-level tests. I then executed the production Manager planning path on a disposable Anvil fork pinned to Sepolia block 11,659,516 against the deployed Beta V1 and V2 contracts. The real eligibility check, preflight, approval builder, HCA batch, MigrationHelper, V2 Registry, and production post-state verifier all completed. From one identical pre-migration snapshot, current V1 data preserved Alice's safe resolver and rejected Mallory's resolver change; stale data assigned Mallory's resolver, granted Mallory the live V2 resolver role, and allowed Mallory to change the resolver again. The fork PoC passes 1/1. The complete migration test run passed 446 tests with the opt-in fork test skipped, and the Manager typecheck passed. I also retrieved the deployed public ENSv2 Beta bundle on 2026-09-08 and confirmed that it contains the same vulnerable logic. I sent no transaction to Sepolia or any public network.

The competition repository flattens the affected files into a single snapshot commit and supplies no release tags. Its snapshot message identifies upstream commit `63772fd872af472ced58b009499355f3430c2a86`; the upstream commit object and its earlier history were not available from the competition remote. The earliest version I could verify is therefore this frozen competition snapshot. No fix is present in the assessed source, and no fixed public release was available to verify. The validation checkout at `1c9b47f` changes `apps/manager/package.json`, not the affected migration logic.

## Vulnerability Details

### Background

V1 `.eth` registration authority has three values that must not be conflated:

* The Base Registrar ERC-721 owner, called the `registrant` by the V1 subgraph, holds the registration token. Alice retains this role throughout the attack.
* The V1 ENS Registry `owner(node)` is the name's current manager and may differ from the registrant. Mallory once held this role, but Alice has revoked her before migrating.
* The V1 ENS Registry `resolver(node)` determines which resolver serves the name. Alice has replaced Mallory's former custom resolver with a safe resolver before migrating.

The V2 migration intentionally keeps Alice as owner, preserves custom resolvers, and restores a legacy manager with only `ROLE_SET_RESOLVER`. The release notes in `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md` describe manager restoration and custom-resolver preservation as intended behavior. This makes the freshness of the source manager and resolver an authorization decision, rather than display metadata.

The complete attack prerequisites are:

1. Alice is the current Base Registrar owner of an unwrapped `.eth` name.
2. Mallory was previously either the V1 Registry owner-manager, the controller of a custom resolver, or both.
3. Alice removes that authority through another V1 client, tab, or direct transaction after the Manager has fetched the name, then returns to migration before the cached response becomes stale. A short indexer lag can produce the same input, but is not required.
4. The migration feature flag is enabled for Alice and she approves the normal migration transactions.
5. For the resolver-preservation variant, the former resolver is a non-public custom resolver that Mallory controls. For the manager-restoration variant, no resolver precondition is needed because the restored role itself authorizes resolver replacement.

The release notes say the rollout remains behind the migration feature flag and targets Sepolia. ENS has also publicly invited eligible users to test the ENSv2 upgrade flow: <https://ens.domains/blog/post/ensv2-beta-public-testing>. The affected logic is present in the deployed public Beta JavaScript bundle, while an anonymous migration-route request redirects because feature evaluation is wallet-gated. I did not measure which connected wallets receive the flag, so eligible-user prevalence remains unknown. The competition scope expressly includes the Manager and V1-to-V2 migration flow: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>.

The root cause is in the Manager app: it treats cached indexer fields as authoritative when deciding who receives V2 authority and which resolver goes into the transaction. The V1 subgraph itself is out of scope, and no incorrect subgraph response is required.

First, `apps/manager/src/features/migration/service/v1SubgraphClient.ts`, `GET_NAMES_QUERY`, fetches all three distinct values:

```graphql
resolver { address }
owner { id }
registrant { id }
```

`apps/manager/src/features/migration/hooks/useV1Names.ts`, `v1NamesQueryOptions` (lines 17-22), keys that response only by the connected owner and considers it fresh for five minutes:

```ts
queryKey: qk('migration', 'v1_names', {
  address: address?.toLowerCase(),
}),
queryFn:
  enabled && address ? () => getV1NamesForAddress(address) : skipToken,
staleTime: 5 * 60 * 1000,
```

The cache alone is enough for the exploit. For example, the subgraph may correctly report Mallory as manager at time T0. Alice changes the V1 Registry at T1 using another client. At T2, less than five minutes after T0, the Manager can legitimately reuse the T0 response without asking the subgraph again.

Second, `packages/migration/src/service/classifyNames.ts`, `classifyName` (lines 137 and 148-184), copies those cached fields into security-sensitive planning state. For an unwrapped name it requires the cached registrant to equal Alice, but derives `managerAddress` from the cached registry owner and derives resolver strategy from the cached resolver:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null
```

Later in the same function, the cached registry owner becomes the manager to restore:

```ts
if (!effectiveWrappedDomain) {
  const registrant = domain.registrant
  if (registrant?.id.toLowerCase() !== addr) return null
  if (parentName !== 'eth') return null
  if (hasExpiredDotEthRegistration(domain, parentName, nowSeconds)) {
    return {
      type: 'ineligible',
      name: { domain, reason: 'expired-registration' },
    }
  }

  const tokenHolder = toAddress(registrant.id)
  if (!tokenHolder) return null

  const registryOwnerAddress = toAddress(domain.owner.id)
  const managerAddress: Address | null =
    registryOwnerAddress &&
    registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
      ? registryOwnerAddress
      : null
```

A non-public custom `v1ResolverAddress` selects the `keep-v1` strategy. Thus, the same cached object establishes that Alice may migrate the registration and separately decides that Mallory should retain resolver authority.

Third, the on-chain eligibility check validates the wrong authority dimension for this decision. `packages/migration/src/service/preflightChecks.ts`, `checkOwnership` (lines 47-101), reads only Base Registrar `ownerOf(labelhash)` for an unwrapped name:

```ts
name.tokenType === 'unwrapped'
  ? {
      address: BASE_REGISTRAR,
      abi: BASE_REGISTRAR_ABI,
      functionName: 'ownerOf' as const,
      args: [BigInt(name.domain.labelhash)] as const,
    }
```

This correctly proves that Alice still holds the registration token. It says nothing about the current V1 Registry `owner(node)` or `resolver(node)`, and neither `runEligibilityChecks` nor `computeMigrationPreflight` reads those values. Forcing migration preflight to refresh therefore does not refresh the cached authorization fields.

Fourth, the cached fields reach two concrete transaction sinks:

* `apps/manager/src/features/migration/service/encodeMigration.ts`, `resolverFor` (lines 24-50), returns `name.v1ResolverAddress` for a non-public custom resolver under `keep-v1`.
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`, `buildRoleGrantCall` (lines 8-27), encodes `ETHRegistry.grantRoles(resource, 1 << 24, name.managerAddress)`, where that role bitmap is `ROLE_SET_RESOLVER`.

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`, `buildNameExecution` (lines 479-490 and 608-624), places both values into the atomic batch:

```ts
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

The same function separately appends the cached manager's role grant:

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

`buildAtomicMigrationInnerExecutions` then orders the MigrationHelper call before the manager-role grant in the same outer HCA batch. The resulting state is:

| Authority                            | Current V1 state before migration | Cached state used by Manager     | V2 transaction outcome               |
| ------------------------------------ | --------------------------------- | -------------------------------- | ------------------------------------ |
| Registration/name owner              | Alice                             | Alice                            | Alice                                |
| Manager / resolver-setting authority | Alice                             | Mallory                          | Mallory receives `ROLE_SET_RESOLVER` |
| Resolver                             | Alice's safe resolver             | Mallory's former custom resolver | Mallory's resolver is assigned       |

The execution path does not repair the mismatch. `apps/manager/src/features/migration/service/migrationService.ts`, `buildNextAtomicBatch` (lines 886-960), rebuilds each batch from `plan.classified` and `plan.profiles`, which already contain the stale values. Immediately before the wallet prompt, `executeMigration` (lines 1169-1204) refreshes migration approval state only. It does not refresh V1 manager or resolver state.

Post-state verification reinforces the bad plan rather than detecting it. `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts`, `verifyExpectation` (lines 263-275 and 337-353), checks that the V2 resolver equals `expectedResolver` and that `managerAddress` has the expected role. Since both expectations came from the stale snapshot, successful restoration of Mallory is reported as a successful migration.

The normal review flow does not identify either stale address. `apps/manager/src/features/migration/hooks/useNameSelection.ts` (lines 33-39) initially selects every eligible name. `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx` (lines 81-108) then says only “Approve manager restoration,” “Restore the existing managers for your names,” and “Migrate selected names and restore their records.” A wallet may show raw calldata, but the Manager does not tell Alice that the “existing” manager is the already-revoked Mallory or that the migration will preserve Mallory's former resolver. Alice still has to approve the wallet prompts; this is not an interactionless attack.

The public known-issues list, rechecked on 2026-09-08, does not disclose this root cause. `WEB-806` concerns migrated subname records not appearing in a profile. `QA-01` concerns errors after transfers with certain role combinations and expressly says that a demonstrated role combination granting unintended authority would be a new finding. This report instead proves stale authority restoration during an unwrapped-name migration and does not rely on either known defect.

## Impact Details

The local evidence establishes a boundary crossing from historical V1 authority to current V2 authority. Mallory needs no present permission when Alice starts migration. Alice has deliberately removed her, but the transaction constructed by the Manager reintroduces her.

A concrete attack proceeds as follows:

1. Alice owns `victim.eth`. Mallory is its V1 Registry manager and controls its custom resolver as part of a previous arrangement.
2. Alice opens the migration page. The Manager fetches a response containing Alice as registrant, Mallory as registry owner, and Mallory's resolver.
3. Alice removes Mallory as manager and changes to a safe resolver through another V1 client or transaction.
4. Alice returns to the still-open migration page within the five-minute cache freshness period, selects `victim.eth`, and approves the ordinary migration flow.
5. Base Registrar `ownerOf` still returns Alice, so the live eligibility guard succeeds. No guard asks the V1 Registry for the current manager or resolver.
6. The atomic batch migrates the name to V2 with Mallory's resolver and grants Mallory `ROLE_SET_RESOLVER`.
7. Mallory can serve attacker-chosen resolution data through the restored resolver or use the role to call the V2 registry's resolver setter. The Explorer path in `apps/portal/src/routes/$name/change-resolver.tsx` checks precisely `ROLE_SET_RESOLVER`, and `apps/portal/src/features/resolver/helpers/changeResolver.ts` encodes `setResolver` for an account with that permission.

The two sinks are independent. If the manager field is current but the resolver field is stale, the migration still assigns Mallory's resolver. If the resolver field is current but the manager field is stale, Mallory still receives `ROLE_SET_RESOLVER` and can redirect resolution afterward. This makes the finding resilient to partial cache updates or differences in the prior V1 setup.

The timing requirement is realistic but bounded. The deterministic trigger is a user changing V1 authority after the Manager's fetch and then continuing the normal migration within five minutes. A lagging indexer can extend or recreate the stale input, but the report does not require or accuse the indexer. The feature flag and the need for Mallory to have held prior authority reduce general prevalence. They do not make the transaction construction probabilistic once the stale object reaches the planner.

Read-only deployment checks support reachability without exercising the vulnerability. The public Beta asset `useEligibleV1Names-CHGkL6IA.js` (<https://app.ens.dev/assets/useEligibleV1Names-CHGkL6IA.js>), fetched on 2026-09-08 with SHA-256 `372e07f6499771fd899f928e1e251783d6458f8b74b6f910a6638564099eb7ae`, contains the same `v1_names` five-minute cache, unwrapped `ownerOf` eligibility read, manager derivation from `domain.owner`, custom-resolver `keep-v1` branch, and `grantRoles(..., 1n << 24n, managerAddress)` encoding. This establishes deployment of the vulnerable application logic, although connected-wallet feature-flag eligibility remains unknown.

At Sepolia block 11,659,273, `eth_getCode` returned non-empty bytecode for the Manager-configured ETHRegistry, MigrationHelper, UnlockedMigrationController, permissioned resolver implementation, and VerifiableFactory. A read-only V1 subgraph sample on 2026-09-08 found five unwrapped names among the first 1,000 returned registrations whose registrant and registry owner differed, and 46 unwrapped names with a resolver. This non-random sample shows that the restoration and resolver branches have real inputs; it is not a prevalence estimate and does not prove that any sampled user is vulnerable at this moment.

The stronger fork PoC pins Sepolia block 11,659,516 and uses the addresses selected by the production Manager build: V1 BaseRegistrar `0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85`, V1 legacy Registry `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`, V2 ETHRegistry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, MigrationHelper `0xddC597d937618849348E18Db5D631Ce747bCDeEF`, and HCA implementation `0xAA761541620fC1a42bb701a26a9f107A9DF1E904`. This distinction matters: the older Sepolia V1 controller at `0xF42d...` writes to a different historical registrar and is not the source deployment configured by this Beta Manager.

The configured V1 registrar's public controllers were revoked at the migration cutover. The test therefore creates a disposable pre-existing V1 token in the same way as the repository's `dev-migration-tool` Anvil seeder: it impersonates the registrar owner only on the local fork, authorizes Alice as a controller, and registers a test token. It similarly impersonates the deployed V2 registrar to establish the required RESERVED premigration record. It stops both impersonations before the security scenario, establishes and then revokes Mallory's V1 manager and resolver authority with ordinary V1 calls, and takes the comparison snapshot. The privileged local setup models the source token and premigration state; it is not an attacker capability or a step in the exploit.

The vulnerable branch's trace then shows the deployed HCA calling `MigrationHelper.migrate` with Mallory's stale resolver, the deployed V2 Registry registering the name with that resolver, and a second call to `ETHRegistry.grantRoles(canonicalId, 16777216, Mallory)`. `16777216` is `ROLE_SET_RESOLVER`. The migration transaction kept Alice as V2 owner. A subsequent transaction sent by Mallory directly called the deployed `ETHRegistry.setResolver` and succeeded. Direct reads after that call returned Alice as owner, `true` for Mallory's resolver role, and Mallory's replacement resolver as the active resolver.

The negative control starts from the exact same Anvil snapshot and changes only the domain object passed to the production planner: it supplies the current V1 owner and safe resolver rather than the cached values. The same deployed migration then retains the safe resolver, does not grant Mallory a role, and Mallory's otherwise identical `setResolver` call reverts. Two further branches from that snapshot isolate the sinks: stale resolver data alone assigns Mallory's resolver without granting a role, while stale manager data alone preserves Alice's safe resolver but grants Mallory the role and lets her replace it. The combined branch produces both effects. This isolates each stale authority field as an independent cause instead of merely showing that migration can grant a role. Alice can still replace the resolver and revoke Mallory if she detects the vulnerable state, so the demonstrated effect is a recoverable name-resolution hijack rather than permanent loss. The competition nevertheless classifies a migration that assigns a name's resolver to an attacker as Critical without requiring permanent ownership loss.

## Remediation

Treat the subgraph response as discovery data, not authorization data. Before constructing a migration preview, and again immediately before constructing each submitted atomic batch, read the current V1 ENS Registry `owner(node)` and `resolver(node)` for every selected name at a consistent block. Recompute `managerAddress`, `v1ResolverAddress`, and `resolverStrategy` from those on-chain values. If they differ from the values Alice reviewed, invalidate the plan and require a refreshed preview; a read failure must fail closed.

The strongest fix is to make the migration transaction itself derive these values from the V1 Registry or revert unless supplied expected values still match current state. That removes the remaining time-of-check/time-of-use gap between the Manager's final RPC read and transaction inclusion. If this requires a contract change outside the competition scope, the Manager-side reread and explicit mismatch rejection should still be implemented immediately.

The preview should show each non-owner manager address and custom resolver address that will be retained. This does not replace the authorization check, but it lets Alice assess intended restoration rather than approving a generic “existing managers” label.

Add regression coverage for:

* a manager revoked after the initial V1 query;
* a resolver replaced after the initial V1 query;
* independent stale-manager and stale-resolver cases;
* both fields changing between preview and final submission;
* failed or partial V1 Registry reads;
* the current-state negative control, where an unchanged legitimate manager and custom resolver are preserved;
* a contract-level check, if added, proving that a post-preview authority change reverts atomically.

No verified upstream fix or fixed release was available during this review.

## References

* ENS audit competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Manager app asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Deployed ENSv2 Beta bundle: <https://app.ens.dev/assets/useEligibleV1Names-CHGkL6IA.js>

## Link to Proof of Concept

<https://gist.github.com/muhammetyurdan/2fdc54f75e24d2d8e3c8bc5ad6ef8a33>

## Proof of Concept

## Proof of Concept

**Runnable test files (secret Gist):** <https://gist.github.com/muhammetyurdan/2fdc54f75e24d2d8e3c8bc5ad6ef8a33>

The source-level PoC is `apps/manager/src/features/migration/service/staleV1Authorization.poc.test.ts`. It is an offline Vitest test: it does not contact a chain, change public state, or require attacker credentials. Its first test runs the real `runEligibilityChecks` path and records the requested function names, proving that the live gate reads only Base Registrar `ownerOf` and V2 `getStatus`, with no V1 Registry `owner` or `resolver` call. The remaining tests supply a preflight representing Alice legitimately passing that ownership and approval stage, then inspect the exact transaction-building output.

From the repository root with the workspace dependencies installed, run:

```sh
corepack pnpm@10.27.0 --filter manager exec vitest run src/features/migration/service/staleV1Authorization.poc.test.ts --reporter=verbose
```

The five tests cover:

1. the actual live eligibility gate: only `ownerOf` and `getStatus` are read, while the cached Mallory authority remains eligible;
2. stale resolver only: the current manager is Alice, yet Mallory's former resolver is encoded;
3. stale manager only: the current resolver is safe, yet Mallory receives `ROLE_SET_RESOLVER`;
4. both stale: both capabilities are restored and `readContract` is never called;
5. current-state negative control: neither Mallory's role grant nor resolver is present.

The observed pass summary was:

```
Test Files  1 passed (1)
     Tests  5 passed (5)
```

The process exited successfully. The test environment emitted unrelated Reown remote-configuration and happy-dom teardown warnings after the pass summary; they do not change the assertions or exit status.

The deployed-contract PoC is `apps/manager/src/features/migration/service/staleV1Authorization.fork.poc.test.ts`. Start a disposable Anvil fork from the repository root:

```sh
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
  --fork-block-number 11659516 --chain-id 11155111 \
  --port 8545 --host 0.0.0.0 --steps-tracing
```

Then run the gated fork test. In PowerShell:

```powershell
$env:ENS_FORK_POC='1'
$env:ANVIL_RPC_URL='http://127.0.0.1:8545'
corepack pnpm@10.27.0 --filter manager exec vitest run `
  src/features/migration/service/staleV1Authorization.fork.poc.test.ts `
  --reporter=verbose
```

The fork test uses production code to run eligibility, compute the preflight, build and submit every required approval, build the atomic HCA batch, execute it through deployed contracts, and verify the receipt with `verifyAtomicMigrationBatch`. It then makes the former manager's resolver change through the production `buildSetResolverCall`. It asserts all four branches below:

| Identical fork state           | Resolver immediately after migration | Former manager has `ROLE_SET_RESOLVER` | Former manager can replace resolver                    |
| ------------------------------ | ------------------------------------ | -------------------------------------- | ------------------------------------------------------ |
| Current V1 object              | Alice's safe resolver                | No                                     | No; call reverts                                       |
| Only resolver is stale         | Mallory's former resolver            | No                                     | No role needed; Mallory controls the assigned resolver |
| Only manager is stale          | Alice's safe resolver                | Yes                                    | Yes                                                    |
| Manager and resolver are stale | Mallory's former resolver            | Yes                                    | Yes                                                    |

The clean pinned-fork run reported:

```
Test Files  1 passed (1)
     Tests  1 passed (1)
```

The test prints the fork block, all participating addresses, every branch's post-state, and run-specific local transaction hashes. On one retained vulnerable branch, `cast run` decoded the following decisive calls:

```
MigrationHelper.migrate([(..., owner=Alice, resolver=MalloryResolver)], [], [], [])
ETHRegistry.register(..., owner=Alice, ..., resolver=MalloryResolver, ...)
ETHRegistry.grantRoles(canonicalId, 16777216, Mallory)
ETHRegistry.setResolver(canonicalId, MalloryReplacementResolver)  // sent by Mallory
```

These are transactions on the disposable local fork. The test sends nothing to a public network.

Regression checks on the same checkout also passed:

```sh
corepack pnpm@10.27.0 --filter manager exec vitest run src/features/migration --reporter=dot
corepack pnpm@10.27.0 --filter manager typecheck
```

The complete migration run reported 46 passing files, 446 passing tests, and the opt-in fork PoC as one skipped file/test. Typecheck exited successfully. The offline PoC uses only in-memory fixtures and mocks. The fork PoC is skipped unless `ENS_FORK_POC=1` is explicitly set and only modifies the disposable Anvil state.
