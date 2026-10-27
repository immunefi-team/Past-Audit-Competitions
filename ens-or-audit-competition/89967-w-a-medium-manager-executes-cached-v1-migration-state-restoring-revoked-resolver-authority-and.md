# 89967 w a medium manager executes cached v1 migration state restoring revoked resolver authority and redirecting ens payments

## #89967 \[W\&A-Medium] Manager executes cached V1 migration state, restoring revoked resolver authority and redirecting ENS payments

**Submitted on Aug 22nd 2026 at 07:26:31 UTC by @nikhil6675 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89967
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## Brief/Intro

`apps/manager` loads V1 registry ownership through a shared `v1_names` query cached for five minutes and stores that value with preview-time V1 profile records in an executable `MigrationPlan`. The submit path compares only migration approval keys before passing the cached plan into HCA batch construction, without re-reading the V1 registry owner, resolver, or record values. A former manager revoked before the preview can regain `ROLE_SET_RESOLVER` and redirect the name through a resolver deployed after migration. Separately, the migration transaction can replay a corrected V1 `addr(60)` into the V2 resolver.

## Vulnerability Details

#### 1. V1 discovery retains the registry owner in a shared five-minute cache

* `apps/manager/src/features/migration/hooks/useV1Names.ts:15-22`: the address-scoped `migration/v1_names` query has `staleTime: 5 * 60 * 1000`.
* `apps/manager/src/features/migration/service/v1SubgraphClient.ts:23-37`: V1 discovery returns `owner { id }`, the registry owner used as manager state.
* `apps/manager/src/features/dashboard/useDashboardV1Names.ts:18-24`: Dashboard consumes the same `useV1Names()` query later consumed by `/migration` during client-side navigation.

```typescript
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

#### 2. Classification binds cached discovery state into the executable plan

* `packages/migration/src/service/classifyNames.ts:160-184`: an unwrapped name whose cached `domain.owner.id` differs from the registrant receives that owner as `managerAddress`.
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:320-406`: `classifyNames()` output is supplied to batch construction and returned as `plan.classified`.
* `apps/manager/src/features/migration/pages/MigrationPage.tsx:271-282`: starting migration passes `gasEstimate.plan` directly to the migration state machine.

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

#### 3. Preview-time profile values remain attached to the plan and retries

* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:349-406`: profile values fetched during preview are passed to atomic batch construction and stored as `plan.profiles`.
* `apps/manager/src/features/migration/service/fetchV1Profiles.ts:88-132`: exact record values are read from V1 resolver contracts through `publicClient.multicall`.
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:413-476`: `adjustPlanForRetry()` filters migrated names while retaining the original plan profile map and classified entries for remaining names.

```typescript
const profiles = await fetchProfilesForNames({
  namesToOwnedPermRes,
  preflight,
  publicClient,
})

return {
  ...,
  classified,
  profiles,
  atomicBatches: atomicPlan.batches,
}
```

```typescript
const remainingClassified = plan.classified.filter(
  (name) => !migratedSet.has(name.domain.name),
)

return {
  ...plan,
  classified: remainingClassified,
  atomicBatches: remainingAtomicBatches,
  stepDescriptors,
}
```

#### 4. Pre-submit validation compares approval keys, not V1 authority or records

* `apps/manager/src/features/migration/service/migrationService.ts:367-390`: `assertMigrationApprovalPlanCurrent()` recomputes missing approvals and compares only sorted approval keys.
* `apps/manager/src/features/migration/service/migrationService.ts:1170-1194`: the approval comparison runs before the first wallet prompt, but no V1 `owner(node)`, resolver, or record comparison follows it.
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:78-96`: confirmation copy says “Restore the existing managers” and does not display the manager address or a live-state diff.

```typescript
const plannedApprovalKeys = sortedApprovalKeys(
  params.plan.preflight.migrationApprovals ?? [],
)
const currentApprovalKeys = sortedApprovalKeys(missing)
const matches =
  plannedApprovalKeys.length === currentApprovalKeys.length &&
  plannedApprovalKeys.every(
    (approvalKey, index) => approvalKey === currentApprovalKeys[index],
  )
```

#### 5. Cached manager state becomes a V2 resolver-authority grant

* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-25`: `buildRoleGrantCall()` encodes `grantRoles(resource, ROLE_SET_RESOLVER, name.managerAddress)`.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-623`: every non-null cached manager adds a `manager-role-grant` execution and matching verification expectation.
* The primary PoC locates the exact stale grant calldata inside successful migration transaction `0x5402d46e06f7f109d8a6cb0971a9ddf7848ad37f0e2593086fa55cc0eef903f5`.

```typescript
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

#### 6. Cached resolver profiles become V2 record setters

* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:627-669`: cached profile records are flattened into resolver setters, wrapped as `profile-replay`, and copied into verification expectations.
* The `01B` PoC locates the exact stale `setAddr(node, 60, attackerRecipient)` calldata inside successful migration transaction `0x2fd7832fcebc62646c21d53f2ed9f26ab6a7398446bc04115c759aec30e064b4`.

```typescript
const profileCalls = flattenProfileInnerCalls(
  new Map([[profileEntry.node, profileEntry.profile]]),
)
innerExecutions.push({
  phase: 'profile-replay',
  name,
  names: [name],
  call: wrapInnerCallsAsMulticall(resolver, profileCalls),
})
```

#### 7. Post-state verification treats cached expectations as authoritative

* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-379`: manager-role verification checks the cached account/bitmap and profile-address verification checks the cached record value.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:615-669`: those expectations are created from the same `classified.managerAddress` and `profileEntry.profile` used to construct the calls.
* Successful verification performs no comparison with the current V1 registry owner, resolver, or record values.

```typescript
case 'profile-address': {
  const value = await context.publicClient.readContract({
    address: expectation.resolver,
    abi: permissionedResolverReadAbi,
    functionName: 'addr',
    args: [expectation.node, expectation.coinType],
    ...block,
  })
  return value.toLowerCase() === expectation.value.toLowerCase()
}
```

## Impact Details

* The primary PoC shows a former V1 manager already revoked before the migration preview regaining `ROLE_SET_RESOLVER`, deploying a resolver only after migration, and redirecting standard ENS resolution and a `0.001 ETH` payment to an attacker-controlled recipient.
* The independent `01B` path shows a corrected V1 `addr(60)` replayed during migration, immediately redirecting standard resolution and a second `0.001 ETH` payment; this path applies to recognized legacy PublicResolvers migrated into an owned PermissionedResolver, while the canonical resolver `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5` follows the keep-resolver path.
* Successful migration transaction inputs contain the exact stale `grantRoles(resource, ROLE_SET_RESOLVER, attacker)` and `setAddr(node, 60, attackerRecipient)` calls.
* The confirmation dialog lists “Restore the existing managers” without showing the manager address or a live-state comparison.
* The resulting role and resolver record are persistent V2 state until the victim discovers the wrong resolution and submits a separate signed correction transaction.
* No validator compromise, project credential, privileged project address, public-network mutation, or real ENS name is required.
* Reloading Manager after correcting V1 state produces `attackerHasRole: false` and the victim address, isolating stale plan state as the cause.

## References

* ENS audit competition repository: <https://github.com/immunefi-team/audit-comp-ens>
* audited revision: <https://github.com/ensdomains/apps-monorepo/tree/63772fd872af472ced58b009499355f3430c2a86>
* competition snapshot: <https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae>
* migration release notes: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/docs/HCA\\_MIGRATION\\_RELEASE\\_NOTES.md>
* ENS V2 contracts: <https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md>

### Link to Proof of Concept

<https://gist.github.com/nikhil66-str/dfc7b84c75c4ddc0ac6beaa4151f829a>

### Proof of Concept

### Proof of Concept

Runnable PoC, supporting files, and clean-checkout instructions:

`https://gist.github.com/nikhil66-str/dfc7b84c75c4ddc0ac6beaa4151f829a`

Tested competition snapshot: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

Audited upstream revision: `63772fd872af472ced58b009499355f3430c2a86`

The test runs against the competition repository's Dockerized local Anvil Sepolia environment.

### Revoked manager authority restoration

1. The victim owns an unwrapped V1 `.eth` registration and the attacker is its V1 Registry manager.
2. The Manager Dashboard fetches the name and stores the attacker manager in the shared `v1_names` query cache.
3. Before the migration preview opens, the victim reclaims the V1 Registry node.
4. On-chain reads confirm that the BaseRegistrar owner, V1 Registry manager, and active `addr(60)` are the victim.
5. The intercepted discovery response is also updated to the victim before client-side navigation to `/migration`.
6. Manager reuses the cached former-manager value when constructing the migration plan.
7. The victim submits the migration through the Manager UI.
8. The successful migration transaction contains `grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
9. After migration, `hasRoles(resource, ROLE_SET_RESOLVER, attacker)` returns `true`.
10. The attacker deploys a custom resolver after migration and sets it as the name's V2 resolver using the restored role.
11. Standard ENS resolution returns the attacker-controlled recipient `0x8fd379246834eac74B8419FfdA202CF8051F7A03`.
12. A local `0.001 ETH` payment following that resolution increases the recipient balance by exactly `1000000000000000` wei.

Local transaction evidence:

* Stale role-grant migration transaction: `0x5402d46e06f7f109d8a6cb0971a9ddf7848ad37f0e2593086fa55cc0eef903f5`
* Resolver deployment transaction: `0x04abdec263b4fdbe36cf608df7232b416098578ba28f149712acb6241f833752`
* Resolver-change transaction: `0x0209f2f16df66c6f0ab414b1c0e21c7628d177b248a403e5f0454d187f8b6c51`
* Redirected-payment transaction: `0x706380717d46bca0b997c75079a48a91ad5d8b4b09e5f0a0d589244e657fba50`

### Stale address record replay

1. A recognized legacy PublicResolver contains an attacker-controlled `addr(60)`.
2. Manager prepares the migration plan and reads that value from the V1 resolver on-chain.
3. Before submission, the victim changes the live V1 `addr(60)` to the victim address.
4. The victim submits the existing migration plan.
5. The successful migration transaction contains `setAddr(node, 60, attackerRecipient)` with the preview-time value.
6. The final V2 resolver and standard ENS resolution return the attacker-controlled recipient.
7. A second local `0.001 ETH` payment following resolution increases the attacker-controlled recipient balance by exactly `1000000000000000` wei.

Local transaction evidence:

* Stale setter migration transaction: `0x2fd7832fcebc62646c21d53f2ed9f26ab6a7398446bc04115c759aec30e064b4`
* Redirected-payment transaction: `0x77058a7cdbfc73f7853966803ec1a58fad21815e29ca9c24e93d3128e3e6b17e`

This record-replay path applies to recognized legacy PublicResolvers migrated into an owned PermissionedResolver. The cached-manager path does not depend on that resolver class.

### Negative control

The test repeats the initial setup but reloads Manager after correcting the V1 manager and address.

Final state:

* V2 owner: victim
* `attackerHasRole`: `false`
* V2 `addr(60)`: victim

### Execution

Complete the setup instructions in the Gist README, then run:

```bash
export CI=1
export ANVIL_RPC_URL=http://127.0.0.1:18545
export MANAGER_APP_URL=http://127.0.0.1:3100
pnpm --dir e2e exec playwright test --config=projects/manager/playwright.poc.config.ts --retries=0 --reporter=list
```

Expected result:

```
4 passed (2.2m)
```
