# 92605 w a critical merged resolver role removal revokes one scope while leaving root record write authority

**Submitted on Sep 10th 2026 at 21:36:55 UTC by @Paulius for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92605
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

## Description

## Brief/Intro

The resolver roles UI groups all grants for the same account into one table row, even when those grants belong to different resolver scopes. The edit and remove sidebar then chooses a single name scope from that merged row and submits one scoped revoke.

As a result, an admin can confirm a removal that the UI describes as removing the user from all roles, while root or other name-scoped resolver permissions for that same delegate remain active on-chain.

## Vulnerability Details

`groupRolesByAccount` builds the table data with `role.account.toLowerCase()` as the only grouping key. During that grouping it merges all decoded permissions into one `decodedRoles` array and all resolved scope names into one `resolvedNames` set. The returned `roles` array does retain the original `(resource, roleBitmap)` entries, but the sidebar does not use those pairs when constructing an edit or removal action.

The table does display all resolved scope labels in the row. The sidebar also acknowledges a row containing `(root)` by showing `Global roles (all names)`. That label correctly signals that at least one global role exists, but it does not fix the action that follows because the sidebar still derives a single editable scope from the merged row with:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? '';
```

This means a row that contains both `(root)` and `alice.eth` will be edited as `alice.eth`. A row that contains several name scopes will be edited as the first non-root name returned by the indexed role data. The permissions shown in the sidebar still come from the merged `decodedRoles`; although the row retains the original pairs, the action path ignores them and cannot revoke the same permissions across every represented scope.

The remove flow has the same problem. Even after the sidebar has shown `Global roles (all names)`, the confirmation modal separately says it will "remove this user from all roles". That confirmation is misleading because `removeUserMutation` calls `revokeResolverRoles` once with one `name`, one `account`, and the merged list of roles. `revokeResolverRoles` encodes either a root-scope revoke when `name === ''`, or a single name-scope revoke otherwise:

```ts
name === ''
  ? { scope: 'root', roles }
  : { scope: 'name', name, roles }
```

It cannot remove all scopes represented by the merged row. If the delegate has a root role plus a name role, the selected name causes the root revoke to be skipped. If the delegate has roles on multiple names, the other names are skipped.

For example:

1. An ordinary owner or authorized administrator of a permissioned resolver has the permissions required to access the role-management UI and revoke `ROLE_SET_TEXT`.
2. A delegate has `ROLE_SET_TEXT` on the root resolver scope.
3. The same delegate also has `ROLE_SET_TEXT` on `alice.eth`.
4. The roles table groups both grants into a single row for that delegate.
5. The admin opens the row and confirms removal.
6. The sidebar selects `alice.eth` as the action scope and submits a revoke for that name only.
7. The root-scoped `ROLE_SET_TEXT` grant remains active.

Because root roles apply to every node on that resolver, the delegate can still modify non-financial records for any name using the resolver immediately after the successful removal transaction, even though the admin explicitly used the UI path that promised to remove the user from all roles.

The owner can remove the remaining root grant with a separate root-scoped revoke. The indexer's exact post-revocation presentation is not needed for the finding and was not relied on. The defect is therefore a falsely confirmed, incomplete revocation, rather than permanent concealment of the remaining authority.

## Impact Details

This creates retained delegated authority after an explicit permission removal. In the clearest three-party case, resolver administrator A removes delegate D through the UI, while another name owner B has a registered name using that resolver. D retains the root permission and can continue changing non-sensitive records for B's name. B need not connect a wallet or interact with the removal flow.

The clearest Medium case is root `ROLE_SET_TEXT` or similar non-financial record authority, since a root-scoped resolver role can apply broadly across names using that resolver. The delegate may continue changing profile text, content, or other non-sensitive metadata records after the owner or manager believed access was revoked.

This is not account takeover and does not directly transfer ENS ownership. The supplied isolated local-EVM PoC demonstrates the complete three-party condition: it registers `bob.eth` to B in a fresh `RootRegistry`/`ETHRegistry` hierarchy, assigns that name the tested `PermissionedResolver`, performs the UI-produced name-only revoke, and shows D writing B's `description` record through the retained root role. B does not connect a wallet or interact with A's removal workflow.

## References

* [Competition-branch Explorer asset snapshot](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal): immutable competition-branch snapshot corresponding to the selected program asset.
* [apps/portal/src/lib/roles/resolverRoles.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts): `groupRolesByAccount` groups only by account and merges roles from all resources.
* [apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx): an empty selected name is the supported root resource path where roles apply to all names.
* [apps/portal/src/features/resolver/helpers/grantResolverRoles.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts): an empty name is encoded as `scope: 'root'` when granting resolver roles.
* [apps/portal/src/features/resolver/components/ResolverRolesTable.tsx](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx): the grouped account rows are passed into the roles table and then into the sidebar.
* [apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx): the sidebar labels rows containing `(root)` as `Global roles (all names)`, then picks one non-root name from `resolvedNames` and uses merged roles for save and remove actions. The remove path starts in `handleRemoveUser`.
* [apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts): the helper encodes a revoke for only one root or name scope through `revokeResolverRolesWriteParameters`.
* [apps/portal/src/utils/ens/eventSignatures.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/ens/eventSignatures.ts): resolver role changes are emitted as `EACRolesChanged`.
* [apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts): the project's PermissionedResolver read ABI uses `hasRootRoles` for root-scoped role checks.

## Link to Proof of Concept

<https://gist.github.com/pauliax/567c8c754f1b613018bc0edea935fad0>

## Proof of Concept

## Proof of Concept

This proof of concept uses the regression test for the sidebar and calldata path. In its isolated local EVM, it deploys a fresh ENSv2-style registry hierarchy, registers `bob.eth` to B, assigns it to the same permissioned resolver, and then demonstrates the retained delegate authority changing B's `description` record after A completes the misleading removal flow.

Preconditions:

* A resolver delegate has at least two grants on the same resolver, using the same account but different scopes.
* One grant is root-scoped and another is name-scoped, or the grants are on two different names.
* An ordinary owner or authorized administrator uses the Explorer roles UI to manage the delegate and has the permissions required to revoke `ROLE_SET_TEXT`.
* A registered name owned by B uses that resolver, so the retained root role can affect B's non-sensitive records. The local EVM test establishes this with `bob.eth`; it is not a claim about any live name or live resolver deployment.

Steps:

1. Load the resolver roles page with role data containing the same delegate account on root and `alice.eth`.
2. The table calls `groupRolesByAccount`.
3. Because the grouping key is only the lowercased account address, the root grant and the `alice.eth` grant appear as one row.
4. The admin opens that row in the sidebar.
5. The sidebar computes `roleName` as the first non-root value in `resolvedNames`, so the pending action is scoped to `alice.eth`.
6. The admin clicks remove and confirms the modal that says the user will be removed from all roles.
7. The transaction is prepared through `revokeResolverRoles` with `name = 'alice.eth'`.
8. `revokeResolverRoles` encodes a name-scoped revoke for `alice.eth`; it does not encode a root-scoped revoke.
9. After the transaction succeeds, the delegate's root-scoped resolver role is still active.
10. The isolated EVM registers `bob.eth` to B with the same resolver and demonstrates a permitted `description` write by D after the removal. B takes no action. A subsequent root-scoped cleanup removes the permission and the same write is rejected.

Expected behavior:

The UI should either show separate rows for each `(account, resource)` pair, or the remove flow should submit revokes for every `(resource, roles)` pair represented by the row. The confirmation text should also match the exact scopes that will be revoked.

Actual behavior:

The UI merges all scopes for an account, chooses one scope for the transaction, and can leave the other scopes active after a full-user removal confirmation.

Supporting local PoC/regression test:

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.review.test.tsx`

In one terminal, start an isolated local chain:

```bash
anvil --host 127.0.0.1 --port 18547 --chain-id 11155111 --silent
```

In another terminal, run:

```bash
ENS_REVIEW_LOCAL_EVM=1 pnpm --filter=portal test src/features/resolver/components/ResolverRolesSidebar.review.test.tsx
```

The artifact patch must place `PermissionedResolverImpl.review.json`, `PermissionedRegistry.review.json`, and `LabelStore.review.json` under `audits/evidence/` at the repository root. The test reads those exact paths relative to the Portal package and writes `local-evm-result.json` there after execution.

The test establishes the vulnerable pre-transaction state, drives the actual sidebar's all-roles confirmation, captures and decodes the prepared application/ENSJS calldata, and confirms it is a single `authorizeNameRoles` revoke for `alice.eth`. With `ENS_REVIEW_LOCAL_EVM=1`, it executes that calldata against fresh contracts from the pinned ENS contracts-v2 revision, including a `RootRegistry`, an `ETHRegistry`, and a `PermissionedResolver` proxy. It verifies B owns `bob.eth`, that `bob.eth` uses the tested resolver, D's successful post-removal text-record write, and rejection of that write after an explicit root cleanup.

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.review.test.tsx

// Independent review: actual sidebar -> actual application/ENSJS calldata.
// ENS_REVIEW_LOCAL_EVM=1 also executes it against the pinned resolver and
// registry bytecode on localhost:18547. No public network, fork, or live
// wallet is used.
import { readFileSync, writeFileSync } from 'node:fs'
import { resolve } from 'node:path'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { Row } from '@tanstack/react-table'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import {
  type Abi, type Address, type Hex, concat, createPublicClient,
  createWalletClient, decodeFunctionData, encodeAbiParameters, encodeFunctionData,
  http, isAddressEqual, parseAbi, toHex,
} from 'viem'
import { namehash, packetToBytes } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { beforeAll, describe, expect, it, vi } from 'vitest'
import {
  type AccountRoleGroup, buildResourceToNameMap, computeNameResource,
  groupRolesByAccount, resolverRoles,
} from '@/lib/roles/resolverRoles'

const localEvm = process.env.ENS_REVIEW_LOCAL_EVM === '1'
const rpc = 'http://127.0.0.1:18547'
const publicClient = createPublicClient({ chain: sepolia, transport: http(rpc) })
let owner: Address = '0x1111111111111111111111111111111111111111'
let delegate: Address = '0x2222222222222222222222222222222222222222'
let victim: Address = '0x3333333333333333333333333333333333333333'
let resolverAddress: Address = '0x3333333333333333333333333333333333333333'
let rootRegistryAddress: Address = '0x4444444444444444444444444444444444444444'
let ethRegistryAddress: Address = '0x5555555555555555555555555555555555555555'
let walletClient = createWalletClient({ account: owner, chain: sepolia, transport: http(rpc) })
const textRole = resolverRoles.ROLE_SET_TEXT
const zeroResource = `0x${'00'.repeat(32)}`
const name = 'alice.eth'
const anotherName = 'bob.eth'
const maxExpiry = (1n << 64n) - 1n
const registryRootRoles = 1n | (1n << 8n)
const abi = parseAbi([
  'function initialize(address owner, uint256 bitmap, bytes[] setters)',
  'function grantRootRoles(uint256 bitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 bitmap, address account) returns (bool)',
  'function authorizeNameRoles(bytes name, uint256 bitmap, address account, bool grant) returns (bool)',
  'function hasRootRoles(uint256 bitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 bitmap, address account) view returns (bool)',
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function setText(bytes32 node, string key, string value)',
  'function text(bytes32 node, string key) view returns (string)',
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
  'function setParent(address parent, string label)',
  'function findOwner(string label) view returns (address)',
  'function getResolver(string label) view returns (address)',
])

const startTransaction = vi.fn((_intent: unknown, ..._rest: unknown[]) => 'review-transaction')
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction },
  waitForTransaction: vi.fn(async () => ({ hash: `0x${'ab'.repeat(32)}` })),
}))
vi.mock('wagmi', () => ({
  useWalletClient: () => ({ data: walletClient }),
  usePublicClient: () => publicClient,
}))
vi.mock('@/lib/wagmi', () => ({ sepoliaWithEns: sepolia }))
vi.mock('@/hooks/use-mobile', () => ({ useIsMobile: () => false }))
vi.mock('@/utils/query/pollForIndexerSync', () => ({ pollForIndexerSync: vi.fn(async () => {}) }))
vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({ openModal: vi.fn(), closeModal: vi.fn(), clearTransaction: vi.fn() }),
}))
// Replace only the downstream signing modal, leaving the remove confirmation,
// pending action, mutation, transaction helper and ENSJS encoder real.
vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({ transactions }: { transactions: { onStart: () => void }[] }) => (
    <button onClick={() => transactions[0].onStart()}>Execute review transaction</button>
  ),
}))
const { ResolverRolesSidebar } = await import('./ResolverRolesSidebar')
const { prepareRevokeResolverRolesTransaction } = await import('../helpers/revokeResolverRoles')
const { getResolverInitCalldata } = await import('../utils/permissionedResolver')

async function send(data: Hex, sender = owner, to: Address | null = resolverAddress) {
  const hash = await walletClient.sendTransaction({ account: sender, ...(to ? { to } : {}), data, gas: 8_000_000n })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return receipt
}

async function deployArtifact(
  artifact: { bytecode: { object: Hex } },
  args: Hex = '0x',
) {
  const receipt = await send(concat([artifact.bytecode.object, args]), owner, null)
  return receipt.contractAddress!
}

describe('independent resolver-role submission review', () => {
  beforeAll(async () => {
    if (!localEvm) return
    const accounts = await walletClient.getAddresses()
    ;[owner, delegate, victim] = accounts
    walletClient = createWalletClient({ account: owner, chain: sepolia, transport: http(rpc) })
    const resolverArtifact = JSON.parse(readFileSync(resolve(
      process.cwd(), '../../audits/evidence/PermissionedResolverImpl.review.json',
    ), 'utf8')) as { abi: Abi; bytecode: Hex }
    const labelStoreArtifact = JSON.parse(readFileSync(resolve(
      process.cwd(), '../../audits/evidence/LabelStore.review.json',
    ), 'utf8')) as { bytecode: { object: Hex } }
    const registryArtifact = JSON.parse(readFileSync(resolve(
      process.cwd(), '../../audits/evidence/PermissionedRegistry.review.json',
    ), 'utf8')) as { bytecode: { object: Hex } }
    const labelStoreAddress = await deployArtifact(
      labelStoreArtifact,
      encodeAbiParameters([{ type: 'address' }], ['0x0000000000000000000000000000000000000000']),
    )
    rootRegistryAddress = await deployArtifact(
      registryArtifact,
      encodeAbiParameters(
        [{ type: 'address' }, { type: 'address' }, { type: 'uint256' }],
        [labelStoreAddress, owner, registryRootRoles],
      ),
    )
    ethRegistryAddress = await deployArtifact(
      registryArtifact,
      encodeAbiParameters(
        [{ type: 'address' }, { type: 'address' }, { type: 'uint256' }],
        [labelStoreAddress, owner, registryRootRoles],
      ),
    )
    await send(encodeFunctionData({
      abi,
      functionName: 'register',
      args: ['eth', owner, ethRegistryAddress, '0x0000000000000000000000000000000000000000', 0n, maxExpiry],
    }), owner, rootRegistryAddress)
    await send(encodeFunctionData({
      abi,
      functionName: 'setParent',
      args: [rootRegistryAddress, 'eth'],
    }), owner, ethRegistryAddress)
    const deployment = await send(concat([
      resolverArtifact.bytecode, encodeAbiParameters([{ type: 'address' }], [owner]),
    ]), owner, null)
    const implementation = deployment.contractAddress!
    // Ordinary ERC-1167 proxy delegates to the unmodified deployed implementation.
    const proxy = await send(concat([
      '0x3d602d80600a3d3981f3363d3d373d3d3d363d73', implementation,
      '0x5af43d82803e903d91602b57fd5bf3',
    ]), owner, null)
    resolverAddress = proxy.contractAddress!
    await send(getResolverInitCalldata(owner))
    await send(encodeFunctionData({
      abi,
      functionName: 'register',
      args: ['bob', victim, '0x0000000000000000000000000000000000000000', resolverAddress, 0n, maxExpiry],
    }), owner, ethRegistryAddress)
    expect(await publicClient.readContract({
      address: ethRegistryAddress, abi, functionName: 'findOwner', args: ['bob'],
    })).toBe(victim)
    expect(isAddressEqual(await publicClient.readContract({
      address: ethRegistryAddress, abi, functionName: 'getResolver', args: ['bob'],
    }) as Address, resolverAddress)).toBe(true)
    await send(encodeFunctionData({ abi, functionName: 'grantRootRoles', args: [textRole, delegate] }))
    await send(encodeFunctionData({
      abi, functionName: 'authorizeNameRoles',
      args: [toHex(packetToBytes(name)), textRole, delegate, true],
    }))
    expect(await publicClient.readContract({
      address: resolverAddress, abi, functionName: 'hasRootRoles', args: [1n, owner],
    })).toBe(true) // actual route's ROLE_SET_ADDR access gate
  })

  it('the actual all-roles confirmation submits only one name revoke; root-only cleanup is different', async () => {
    const row = groupRolesByAccount([
      { account: delegate, resource: zeroResource, roleBitmap: String(textRole) },
      { account: delegate, resource: computeNameResource(name), roleBitmap: String(textRole) },
    ], buildResourceToNameMap([{ name }]))[0]
    render(
      <QueryClientProvider client={new QueryClient()}>
        <ResolverRolesSidebar row={{ id: '0', original: row } as Row<AccountRoleGroup>}
          open setOpen={vi.fn()} resolverAddress={resolverAddress} canManageRoles />
      </QueryClientProvider>,
    )
    expect(screen.getByText('Global roles (all names)')).toBeInTheDocument()
    fireEvent.click(screen.getByRole('button', { name: 'Remove user', exact: true }))
    expect(screen.getByText(/remove this user from all roles/)).toBeInTheDocument()
    fireEvent.click(screen.getByRole('button', { name: 'Remove', exact: true }))
    fireEvent.click(screen.getByText('Execute review transaction'))
    await waitFor(() => expect(startTransaction).toHaveBeenCalledTimes(1))
    const intent = startTransaction.mock.calls[0][0] as unknown as { request: { data: Hex; to: Address } }
    const decoded = decodeFunctionData({ abi, data: intent.request.data })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args).toEqual([toHex(packetToBytes(name)), textRole, delegate, false])

    const cleanup = prepareRevokeResolverRolesTransaction({
      resolverAddress, name: '', account: delegate, roles: ['ROLE_SET_TEXT'], walletClient, chainId: sepolia.id,
    })
    expect(decodeFunctionData({ abi, data: cleanup.request.data! }).functionName).toBe('revokeRootRoles')
    if (!localEvm) return

    const receipt = await send(intent.request.data)
    expect(receipt.logs.length).toBeGreaterThan(0)
    expect(await publicClient.readContract({
      address: resolverAddress, abi, functionName: 'roles',
      args: [BigInt(computeNameResource(name)), delegate],
    })).toBe(0n)
    expect(await publicClient.readContract({
      address: resolverAddress, abi, functionName: 'hasRootRoles', args: [textRole, delegate],
    })).toBe(true)
    // Another name on the same resolver remains writable via the retained root grant.
    await send(encodeFunctionData({
      abi, functionName: 'setText', args: [namehash(anotherName), 'description', 'retained delegate write'],
    }), delegate)
    expect(await publicClient.readContract({
      address: resolverAddress, abi, functionName: 'text', args: [namehash(anotherName), 'description'],
    })).toBe('retained delegate write')
    await send(cleanup.request.data!)
    expect(await publicClient.readContract({
      address: resolverAddress, abi, functionName: 'hasRootRoles', args: [textRole, delegate],
    })).toBe(false)
    await expect(publicClient.simulateContract({
      address: resolverAddress, abi, functionName: 'setText', account: delegate,
      args: [namehash(anotherName), 'description', 'must now fail'],
    })).rejects.toThrow()
    writeFileSync(resolve(process.cwd(), '../../audits/evidence/local-evm-result.json'), JSON.stringify({
      localOnly: true, chainId: sepolia.id, rpc, owner, delegate, victim,
      rootRegistryAddress, ethRegistryAddress, resolverAddress,
      registeredName: anotherName,
      registeredNameOwner: victim,
      registeredNameResolver: resolverAddress,
      removalHash: receipt.transactionHash, removalStatus: receipt.status,
      decodedFunction: decoded.functionName, nameScope: name,
      nameScopeRolesAfterRemoval: '0', rootTextRoleAfterRemoval: true,
      anotherNode: anotherName, anotherNodeWriteSucceeded: true,
      rootCleanupRemovedPermission: true, postCleanupWriteRejected: true,
      contractsV2Commit: '97a57293f3b4279d94b571e678edb53ce62638f4',
      limitations: 'Fresh local RootRegistry, ETHRegistry, and ERC-1167 PermissionedResolver proxy built from the pinned contracts-v2 revision; no indexer, live deployment, or full wallet modal tested.',
    }, null, 2) + '\n')
  })
})
```
