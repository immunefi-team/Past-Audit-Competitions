# 90165 w a critical resolver role removal merges name scoped permissions and leaves a delegate authorized

## #90165 \[W\&A-Critical] Resolver role removal merges name-scoped permissions and leaves a delegate authorized

**Submitted on Aug 23rd 2026 at 12:04:42 UTC by @v\_c0d35 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90165
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## Resolver role removal merges name-scoped permissions and leaves a delegate authorized

### Brief/Intro

At public competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the Explorer groups resolver permissions by delegate account without preserving which ENS name owns each permission. When one delegate has roles on two names that share a supported PermissionedResolver, Explorer displays one combined row and tells the owner that **Remove user** will remove the delegate from all roles. The transaction instead applies the combined role bitmap to only one name. A permission on the other name remains active after the successful removal transaction, allowing the delegate to continue changing that name's resolver records.

### Vulnerability Details

The vulnerable state arises through ordinary role management:

* Alice controls `decoy.eth` and `victim.eth`, both using the same PermissionedResolver.
* Alice grants Mallory `ROLE_SET_TEXT` on `decoy.eth` and `ROLE_SET_ADDR` on `victim.eth`.
* Alice later selects Mallory's combined row and confirms **Remove user**.

Resolver permissions are name-scoped resources, so the UI must retain the association between each name and its role bitmap. `groupRolesByAccount()` in `apps/portal/src/lib/roles/resolverRoles.ts:179-220` instead keys its map only by the lower-cased account. When it encounters another resource for that account, it appends the role and decoded bitmap to the existing group:

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  } else {
    const names = new Set<string>()
    if (resolvedName) names.add(resolvedName)
    grouped.set(account, {
      account,
      resolvedNames: names,
      roles: [role],
      decodedRoles: [...decoded],
    })
  }
}
```

The resulting object contains a set of names and one flattened array of roles, but no mapping from a name to its own roles. The sidebar consumes those independent arrays at `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-173` and chooses only the first non-root name:

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
const originalPermissions = useMemo(
  () => roleToPermissions(decodedRoles),
  [decodedRoles],
)
```

For the example above, `decodedRoles` contains both `ROLE_SET_TEXT` and `ROLE_SET_ADDR`, while `roleName` contains only `decoy.eth`. The interface still presents the action as applying to the whole combined row. Its confirmation at `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:441-448` says:

```tsx
<DialogHeader>
  <DialogTitle>Remove user</DialogTitle>
  <DialogDescription>
    Are you sure you want to remove this user from all roles? This
    action cannot be undone.
  </DialogDescription>
</DialogHeader>
```

`handleRemoveUser()` then stores the single selected name together with every flattened role at `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-322`:

```ts
removeUserMutation.reset()
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
openModal()
```

The transaction helper cannot reconstruct the lost name-to-role relationship. At `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`, a name-scoped removal prepares one call containing exactly that `name` and the flattened `roles` array:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles: roles as ResolverRole[],
      }
    : {
        resolverAddress,
        targetAccount: account,
        scope: 'name',
        name,
        roles: roles as ResolverRole[],
      },
)
```

The resulting call for the tested ordering is equivalent to:

```
authorizeNameRoles(decoy.eth, ROLE_SET_TEXT | ROLE_SET_ADDR, mallory, false)
```

It revokes permissions only from the `decoy.eth` resource. `ROLE_SET_ADDR` belongs to the separate `victim.eth` resource, so it remains active. The removal transaction succeeds because applying an extra unset bit to `decoy.eth` is valid; the receipt therefore gives Alice no indication that the second resource was omitted.

The accompanying end-to-end reproduction renders the production role table and sidebar with both resource hashes, confirms the all-roles removal text, prepares and decodes the production transaction, and applies its one-resource state transition. It then uses the surviving `ROLE_SET_ADDR` permission to prepare the production record-update calldata for `victim.eth`. Its single-resource control gives Mallory both roles on `victim.eth`; in that case the same flow encodes `victim.eth` and removes the complete bitmap, isolating the failure to cross-resource grouping.

The published QA-01 issue concerns errors after transfers with certain role combinations. This path requires no transfer and produces a successful removal transaction that leaves authority active. The public known-issue text specifically treats a role combination that grants unintended authority as a new security consequence.

### Impact Details

Alice reviews a combined row that names both resources and approves a transaction described as removing Mallory from all roles. The signed transaction contains different security semantics: one name plus a bitmap assembled from multiple names. Alice therefore receives a successful receipt while Mallory remains authorized on `victim.eth`.

In the demonstrated configuration, the surviving permission is `ROLE_SET_ADDR`, described by Explorer as **Set Address — Can set address records** in `apps/portal/src/lib/roles/resolverRoles.ts:57-66`. Mallory can use that permission immediately after the apparent revocation to replace `victim.eth` address records. Applications and users resolving the name during this period can then receive the address selected by Mallory, creating a direct path to misdirected transfers and transactions.

The retained authority remains effective until Alice identifies the omitted resource and submits another resource-specific revocation. The security exposure therefore covers every record change Mallory can authorize between the successful all-roles removal and that additional corrective transaction. The transaction mismatch itself is deterministic whenever one account row combines roles from multiple name resources. For this reason, albeit strictly this finding could fit the critical impact "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user submits a transaction to an unintended recipient" I'm picking a low severity for it.

### References

* [Assessed public repository commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae)
* [`groupRolesByAccount()`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [Resolver role definitions and descriptions](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L108)
* [Sidebar selects one name and the flattened role array](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173)
* [`handleRemoveUser()`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L322)
* [All-roles confirmation text](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L448)
* [`prepareRevokeResolverRolesTransaction()`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L28-L75)
* [Supported existing-resolver selection](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L267-L300)
* [Supported per-name role selection](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L248-L289)
* [Record transaction preparation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L97-L150)

### Proof of Concept

### Proof of Concept

### What this proves

This test renders the production resolver-role table and sidebar with one delegate holding roles on two different names through the same resolver. It follows the visible **Remove user** flow, captures the production transaction intent, and decodes its calldata. It then demonstrates that the second name's address role remains and that the delegate can prepare a production `setAddr` transaction for that name. A single-resource negative control proves that the normal one-name flow revokes the complete bitmap.

The test is local and does not broadcast a transaction.

### 1. Check out the assessed source

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

Use Node 22 and pnpm 10, as specified by the competition resources.

### 2. Add the reproduction test

Save the following file as `apps/portal/resolver-cross-resource-role-revocation.test.ts`:

```ts
import { permissionedResolverAuthorizeNameRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { publicResolverSetAddrSnippet } from '@ensdomains/ensjs-abi/v1/publicResolver'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import { createElement, type PropsWithChildren, type ReactNode } from 'react'
import type { Address } from 'viem'
import {
  decodeFunctionData,
  namehash,
  toFunctionSelector,
  toHex,
} from 'viem'
import { sepolia } from 'viem/chains'
import { packetToBytes } from 'viem/ens'
import { beforeEach, expect, test, vi } from 'vitest'

const ADMIN = '0x1111111111111111111111111111111111111111' as Address
const ATTACKER = '0x2222222222222222222222222222222222222222' as Address
const RESOLVER = '0x3333333333333333333333333333333333333333' as Address
const FIRST_NAME = 'decoy.eth'
const SECOND_NAME = 'victim.eth'

const capture = vi.hoisted(() => ({
  prepared: [] as unknown[],
  revokeResolverRoles: vi.fn(async () => `0x${'01'.repeat(32)}` as const),
}))

vi.mock('wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('wagmi')>()
  return {
    ...actual,
    usePublicClient: () => ({}),
    useWalletClient: () => ({
      data: {
        account: { address: '0x1111111111111111111111111111111111111111' },
        chain: sepolia,
      },
    }),
  }
})

vi.mock('@/components/EntityBadge', () => ({
  EntityBadge: ({ children }: { readonly children: ReactNode }) =>
    createElement('span', null, children),
}))

vi.mock('@/features/registry/utils/signer.helpers', () => ({
  createEOASigner: () => ({ type: 'eoa', walletClient: {} }),
}))

vi.mock(
  '@/features/resolver/helpers/revokeResolverRoles',
  async (importOriginal) => {
    const actual =
      await importOriginal<
        typeof import('./src/features/resolver/helpers/revokeResolverRoles')
      >()
    return {
      ...actual,
      revokeResolverRoles: capture.revokeResolverRoles,
    }
  },
)

vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal: vi.fn(),
    closeModal: vi.fn(),
    clearTransaction: vi.fn(),
  }),
}))

vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({
    transactions,
  }: {
    readonly transactions: readonly {
      readonly intent?: {
        readonly prepare?: (context: {
          readonly walletClient: unknown
          readonly chainId: number
        }) => unknown
      }
      readonly onStart?: () => void
    }[]
  }) =>
    createElement(
      'button',
      {
        type: 'button',
        onClick: () => {
          const transaction = transactions[0]
          const prepared = transaction?.intent?.prepare?.({
            walletClient: {
              account: {
                address: '0x1111111111111111111111111111111111111111',
              },
              chain: sepolia,
            },
            chainId: sepolia.id,
          })
          if (prepared) capture.prepared.push(prepared)
          transaction?.onStart?.()
        },
      },
      'confirm transaction',
    ),
}))

vi.mock('@/features/roles/hooks/useResetMutationsOnAccountChange', () => ({
  useResetMutationsOnAccountChange: vi.fn(),
}))

vi.mock('@/utils/query/pollForIndexerSync', () => ({
  pollForIndexerSync: vi.fn(async () => undefined),
}))

vi.mock('@/hooks/use-mobile', () => ({ useIsMobile: () => false }))

vi.mock('@/components/ui/sheet', () => {
  const Wrapper = ({ children }: PropsWithChildren) =>
    createElement('div', null, children)
  return {
    Sheet: Wrapper,
    SheetContent: Wrapper,
    SheetHeader: Wrapper,
    SheetTitle: Wrapper,
  }
})

vi.mock('@/components/ui/dialog', () => {
  const Wrapper = ({ children }: PropsWithChildren) =>
    createElement('div', null, children)
  return {
    Dialog: Wrapper,
    DialogClose: Wrapper,
    DialogContent: Wrapper,
    DialogDescription: Wrapper,
    DialogFooter: Wrapper,
    DialogHeader: Wrapper,
    DialogTitle: Wrapper,
  }
})

import { ResolverRolesTable } from './src/features/resolver/components/ResolverRolesTable'
import { prepareSaveRecordsTransaction } from './src/features/records/helpers/saveRecords'
import type {
  ResolverNode,
  ResolverRole,
} from './src/features/resolver/hooks/useResolverOverview'
import type { CustomTransactionIntent } from '@ens-apps/transaction-manager'
import {
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from './src/lib/roles/resolverRoles'

const makeRole = (
  name: string,
  roleBitmap: bigint,
): ResolverRole => ({
  account: ATTACKER,
  resource: computeNameResource(name),
  roleBitmap: roleBitmap.toString(),
  blockNumber: 1,
  transactionHash: null,
  timestamp: null,
  name,
})

const makeNode = (name: string): ResolverNode => ({
  id: namehash(name),
  name,
  owner: { id: ADMIN },
  resolver: { id: RESOLVER, address: RESOLVER },
})

const renderRoles = (
  roles: readonly ResolverRole[],
  nodes: readonly ResolverNode[],
) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  })

  render(
    createElement(
      QueryClientProvider,
      { client: queryClient },
      createElement(ResolverRolesTable, {
        roles,
        nodes,
        resolverAddress: RESOLVER,
        canManageRoles: true,
      }),
    ),
  )
}

const openAndConfirmRemove = () => {
  fireEvent.click(screen.getByRole('button', { name: 'Edit user roles' }))
  fireEvent.click(screen.getByRole('button', { name: 'Remove user' }))
  fireEvent.click(screen.getByRole('button', { name: 'Remove' }))
  fireEvent.click(screen.getByRole('button', { name: 'confirm transaction' }))
}

beforeEach(() => {
  capture.prepared.length = 0
  capture.revokeResolverRoles.mockClear()
})

test('a merged remove confirms two names but submits one name and leaves address authority', async () => {
  const roles = [
    makeRole(FIRST_NAME, resolverRoles.ROLE_SET_TEXT),
    makeRole(SECOND_NAME, resolverRoles.ROLE_SET_ADDR),
  ]
  const nodes = [makeNode(FIRST_NAME), makeNode(SECOND_NAME)]

  renderRoles(roles, nodes)
  openAndConfirmRemove()

  expect(
    screen.getByText(`Roles scoped to ${FIRST_NAME}, ${SECOND_NAME}`),
  ).toBeInTheDocument()
  expect(
    screen.getByText(/remove this user from all roles/i),
  ).toBeInTheDocument()

  await waitFor(() =>
    expect(capture.revokeResolverRoles).toHaveBeenCalledOnce(),
  )
  expect(capture.prepared).toHaveLength(1)

  const intent = capture.prepared[0] as CustomTransactionIntent
  expect(intent.request.type).toBe('eoa')
  if (intent.request.type !== 'eoa' || !intent.request.data) {
    throw new Error('Expected an EOA transaction with calldata')
  }

  expect(intent.request).toMatchObject({
    chainId: sepolia.id,
    from: ADMIN,
    to: RESOLVER,
    value: 0n,
  })
  expect(intent.request).not.toHaveProperty('token')
  expect(intent.request.data.slice(0, 10)).toBe(
    toFunctionSelector('authorizeNameRoles(bytes,uint256,address,bool)'),
  )

  const decodedRevoke = decodeFunctionData({
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    data: intent.request.data,
  })
  expect(decodedRevoke.functionName).toBe('authorizeNameRoles')
  expect(decodedRevoke.args).toEqual([
    toHex(packetToBytes(FIRST_NAME)),
    resolverRoles.ROLE_SET_TEXT | resolverRoles.ROLE_SET_ADDR,
    ATTACKER,
    false,
  ])
  expect(decodedRevoke.args[0]).not.toBe(toHex(packetToBytes(SECOND_NAME)))

  const submitted = capture.revokeResolverRoles.mock.calls[0]?.[0]
  expect(submitted).toMatchObject({
    resolverAddress: RESOLVER,
    name: FIRST_NAME,
    account: ATTACKER,
    roles: ['ROLE_SET_TEXT', 'ROLE_SET_ADDR'],
    chainId: sepolia.id,
  })

  const rolesAfterReceipt = new Map([
    [computeNameResource(FIRST_NAME), resolverRoles.ROLE_SET_TEXT],
    [computeNameResource(SECOND_NAME), resolverRoles.ROLE_SET_ADDR],
  ])
  rolesAfterReceipt.set(
    computeNameResource(FIRST_NAME),
    (rolesAfterReceipt.get(computeNameResource(FIRST_NAME)) ?? 0n) &
      ~decodedRevoke.args[1],
  )
  expect(rolesAfterReceipt.get(computeNameResource(FIRST_NAME))).toBe(0n)
  expect(rolesAfterReceipt.get(computeNameResource(SECOND_NAME))).toBe(
    resolverRoles.ROLE_SET_ADDR,
  )

  const [refreshedGroup] = groupRolesByAccount([
    makeRole(SECOND_NAME, resolverRoles.ROLE_SET_ADDR),
  ])
  expect(refreshedGroup.roles).toHaveLength(1)
  expect(refreshedGroup.decodedRoles).toEqual(['ROLE_SET_ADDR'])

  const attackerIntent = await prepareSaveRecordsTransaction({
    name: SECOND_NAME,
    resolverAddress: RESOLVER,
    originalRecords: [],
    pendingChanges: {
      newRecords: [
        {
          type: 'address',
          id: 60,
          key: 'ETH',
          value: ATTACKER,
        },
      ],
      editedValues: new Map(),
      deletedIds: new Set(),
    },
    walletClient: {
      account: { address: ATTACKER },
      chain: sepolia,
    } as Parameters<typeof prepareSaveRecordsTransaction>[0]['walletClient'],
    chainId: sepolia.id,
  })
  expect(attackerIntent.request.type).toBe('eoa')
  if (
    attackerIntent.request.type !== 'eoa' ||
    !attackerIntent.request.data
  ) {
    throw new Error('Expected attacker EOA calldata')
  }

  expect(attackerIntent.request).toMatchObject({
    chainId: sepolia.id,
    from: ATTACKER,
    to: RESOLVER,
    value: 0n,
  })
  const decodedSetAddr = decodeFunctionData({
    abi: publicResolverSetAddrSnippet,
    data: attackerIntent.request.data,
  })
  expect(decodedSetAddr.functionName).toBe('setAddr')
  expect(decodedSetAddr.args).toEqual([
    namehash(SECOND_NAME),
    60n,
    ATTACKER,
  ])
})

test('a single-resource row revokes the complete confirmed resource', async () => {
  const roles = [
    makeRole(
      SECOND_NAME,
      resolverRoles.ROLE_SET_TEXT | resolverRoles.ROLE_SET_ADDR,
    ),
  ]

  renderRoles(roles, [makeNode(SECOND_NAME)])
  openAndConfirmRemove()

  await waitFor(() =>
    expect(capture.revokeResolverRoles).toHaveBeenCalledOnce(),
  )
  const intent = capture.prepared[0] as CustomTransactionIntent
  if (intent.request.type !== 'eoa' || !intent.request.data) {
    throw new Error('Expected an EOA transaction with calldata')
  }
  const decodedRevoke = decodeFunctionData({
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    data: intent.request.data,
  })

  expect(screen.getByText(`Roles scoped to ${SECOND_NAME}`)).toBeInTheDocument()
  expect(decodedRevoke.args).toEqual([
    toHex(packetToBytes(SECOND_NAME)),
    resolverRoles.ROLE_SET_TEXT | resolverRoles.ROLE_SET_ADDR,
    ATTACKER,
    false,
  ])

  const roleAfterReceipt =
    (resolverRoles.ROLE_SET_TEXT | resolverRoles.ROLE_SET_ADDR) &
    ~decodedRevoke.args[1]
  expect(roleAfterReceipt).toBe(0n)
})
```

### 3. Run the test

From the repository root:

```sh
pnpm --dir apps/portal exec vitest run resolver-cross-resource-role-revocation.test.ts
```

Observed result against the assessed commit:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

### 4. Read the decisive assertions

The first test confirms all of the following in one flow:

1. The UI shows one account row scoped to both `decoy.eth` and `victim.eth`.
2. The confirmation says the user will be removed from all roles.
3. The prepared transaction is a single `authorizeNameRoles` call for `decoy.eth`.
4. Its bitmap combines `ROLE_SET_TEXT` from `decoy.eth` with `ROLE_SET_ADDR` from `victim.eth`.
5. Applying that one-resource revocation leaves `ROLE_SET_ADDR` on `victim.eth`.
6. The retained delegate can prepare `setAddr(namehash("victim.eth"), 60, mallory)` through the production record helper.

The second test uses one name and confirms that the same UI and helper revoke the complete bitmap when no cross-resource grouping occurs.
