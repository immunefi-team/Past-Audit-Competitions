# 90498 w a critical resolver role table merges different names into one row but signs calldata for only one name

**Submitted on Aug 25th 2026 at 10:04:51 UTC by @pks271 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90498
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:

    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

    (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

Resolver-role authority is scoped by `(account, resource/name)`, not only by account. The Portal loses that boundary in the table model.

### Root cause: grouping key is only `account`

`groupRolesByAccount` creates the table row map with `account` as the key. `resource` is read, but it is not part of the key:

```typescript
// apps/portal/src/lib/roles/resolverRoles.ts:193-204
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
  }
```

So these two distinct permissions:

```
(0xUser, a.eth) -> ROLE_SET_TEXT
(0xUser, b.eth) -> ROLE_SET_ADDR
```

become one UI row:

```
0xUser -> resolvedNames [a.eth, b.eth], decodedRoles [ROLE_SET_TEXT, ROLE_SET_ADDR]
```

### Root cause: the sidebar then chooses one name

The sidebar does not pass an array of names/resources to the write path. It derives exactly one `roleName` from the row:

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Then Remove user stores only that single name together with the union of roles:

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:316-321
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

This is why merging two names into one row does **not** revoke both names. The write payload has one `name` field, not `names[]` or `(resource, roleBitmap)[]`. Once `roleName` is `a.eth`, the builder can only construct a transaction for `a.eth`.

The result is:

```
UI row:
  names = [a.eth, b.eth]
  roles = [ROLE_SET_TEXT, ROLE_SET_ADDR]

Remove user payload:
  name = a.eth
  roles = [ROLE_SET_TEXT, ROLE_SET_ADDR]

Signed calldata:
  authorizeNameRoles(dnsEncode("a.eth"), bitmap(SET_TEXT | SET_ADDR), 0xUser, false)

Missing calldata:
  authorizeNameRoles(dnsEncode("b.eth"), bitmap(SET_ADDR), 0xUser, false)
```

Therefore `b.eth` is not touched.

### Root cause: empty name is mapped to root scope

If no resource resolves to a name, the same sidebar expression returns `''`. The grant helper treats that empty string as resolver root scope:

```typescript
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-53
const writeParams = grantResolverRolesWriteParameters(
  walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles,
      }
```

That produces `grantRootRoles(bitmap, account)`. In the ensjs type comment, root scope means globally for the PermissionedResolver: any name and any record type covered by the granted role bitmap. It is not limited to the name the admin thought they were editing.

## Impact

A resolver admin can sign a transaction whose calldata does not match the role row shown by the official Portal.

In the revoke branch, “Remove user” leaves authority behind on another name because the app only signs for one derived name. In the grant branch, an unresolved row can be converted into resolver-wide root role calldata. If the role bitmap includes `ROLE_SET_ADDR`, the recipient can receive address-record authority across names served by that resolver, subject to the signer having the required admin authority.

This maps to the competition impact for incorrect transaction details causing a user to submit an unintended transaction. It also crosses the known-issue carve-out for role combinations that grant or preserve authority they should not.

## Recommendation

Do not use account-only grouping for write actions. Render and edit resolver roles by `(account, resource/name)`, or make the write model preserve `roles: [{ resource, roleBitmap }]` and construct one revoke per resource. Fail closed when a resource cannot be resolved to a concrete name. Never convert an empty UI-derived name into root scope without an explicit root-scope control and confirmation.

## Proof of Concept

Run:

```bash
pnpm --filter portal exec vitest run src/features/resolver/helpers/resolver-role-cross-resource-scope.test.ts
```

Result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

PoC file:

```
repo/apps/portal/src/features/resolver/helpers/resolver-role-cross-resource-scope.test.ts
```

PoC source:

```typescript
/**
 * PoC: the resolver-role table collapses per-name permissions into one account
 * row, but the transaction builders later operate on a single derived name.
 *
 * No network is touched. The component-level cases render ResolverRolesSidebar,
 * click the real Remove/Save controls, capture the TransactionModal intent, and
 * decode the generated calldata.
 */
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { createElement } from 'react'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createWalletClient, decodeFunctionData, http, parseAbi, toHex } from 'viem'
import { packetToBytes } from 'viem/ens'

const capturedModal = vi.hoisted(() => ({
  transactions: [] as unknown[],
}))

const modalControls = vi.hoisted(() => ({
  openModal: vi.fn(),
  closeModal: vi.fn(),
  clearTransaction: vi.fn(),
}))

vi.mock('wagmi', () => ({
  useWalletClient: () => ({
    data: {
      account: {
        address: '0x9999999999999999999999999999999999999999',
      },
    },
  }),
  usePublicClient: () => ({}),
}))

vi.mock('@/hooks/use-mobile', () => ({
  useIsMobile: () => false,
}))

vi.mock('@/lib/wagmi', () => ({
  sepoliaWithEns: { id: 11155111 },
}))

vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    isOpen: false,
    openModal: modalControls.openModal,
    closeModal: modalControls.closeModal,
    clearTransaction: modalControls.clearTransaction,
  }),
}))

vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({ transactions }: { readonly transactions: unknown[] }) => {
    capturedModal.transactions = transactions
    return null
  },
}))

// Mirrors ensjs's own dnsEncode (wallet/v2/revokeResolverRoles.ts:93).
const dnsEncode = (name: string) => toHex(packetToBytes(name))
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { ResolverRolesSidebar } from '../components/ResolverRolesSidebar'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
} from '@/lib/roles/resolverRoles'

const ATTACKER = '0x2222222222222222222222222222222222222222'
const RESOLVER = '0x3333333333333333333333333333333333333333'
const SET_ADDR = 1n << 0n
const SET_TEXT = 1n << 4n

const walletClient = createWalletClient({
  account: privateKeyToAccount(
    '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
  ),
  chain: sepolia,
  transport: http(),
})

const permissionedResolverAbi = parseAbi([
  'function grantRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
])

const renderSidebar = (original: {
  readonly account: string
  readonly resolvedNames: readonly string[]
  readonly decodedRoles: readonly string[]
  readonly roles: readonly unknown[]
}) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  })

  return render(
    createElement(
      QueryClientProvider,
      { client: queryClient },
      createElement(ResolverRolesSidebar, {
        row: {
          id: 'role-row',
          original,
        } as never,
        open: true,
        setOpen: vi.fn(),
        resolverAddress: RESOLVER,
        canManageRoles: true,
      }),
    ),
  )
}

const preparedIntentFromCapturedModal = () => {
  const transaction = capturedModal.transactions[0] as {
    readonly intent?: {
      readonly prepare?: (ctx: {
        readonly walletClient: typeof walletClient
        readonly chainId: number
      }) => { readonly request: { readonly data: `0x${string}` } } | undefined
    }
  }

  expect(transaction.intent?.prepare).toBeTypeOf('function')
  const intent = transaction.intent?.prepare?.({
    walletClient,
    chainId: sepolia.id,
  })
  expect(intent).toBeDefined()
  return intent
}

describe('resolver-role rows are not isolated by name/resource', () => {
  beforeEach(() => {
    capturedModal.transactions = []
    modalControls.openModal.mockClear()
    modalControls.closeModal.mockClear()
    modalControls.clearTransaction.mockClear()
  })

  it('component Remove user captures only one name from a multi-name row', async () => {
    const user = userEvent.setup()
    const rows = groupRolesByAccount(
      [
        {
          account: ATTACKER,
          resource: computeNameResource('a.eth'),
          roleBitmap: String(SET_TEXT),
        },
        {
          account: ATTACKER,
          resource: computeNameResource('b.eth'),
          roleBitmap: String(SET_ADDR),
        },
      ],
      buildResourceToNameMap([{ name: 'a.eth' }, { name: 'b.eth' }]),
    )

    renderSidebar(rows[0])

    expect(screen.getByText('Roles scoped to a.eth, b.eth')).toBeInTheDocument()
    await user.click(screen.getByRole('button', { name: /Remove user/i }))
    await user.click(screen.getByRole('button', { name: /^Remove$/i }))

    await waitFor(() => expect(modalControls.openModal).toHaveBeenCalledTimes(1))

    const intent = preparedIntentFromCapturedModal()
    const decoded = decodeFunctionData({
      abi: permissionedResolverAbi,
      data: intent!.request.data,
    })

    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args[0]).toEqual(dnsEncode('a.eth'))
    expect(decoded.args[0]).not.toEqual(dnsEncode('b.eth'))
    expect(decoded.args[3]).toBe(false)
  })

  it('component Save encodes an unresolved row as a resolver-wide root grant', async () => {
    const user = userEvent.setup()
    const rows = groupRolesByAccount(
      [
        {
          account: ATTACKER,
          resource: computeNameResource('a.eth'),
          roleBitmap: String(SET_TEXT),
        },
      ],
      buildResourceToNameMap([{ name: 'other.eth' }]),
    )

    renderSidebar(rows[0])

    expect(screen.queryByText(/Roles scoped to/)).not.toBeInTheDocument()
    await user.click(screen.getAllByRole('checkbox', { name: 'Manager' })[0])
    await user.click(screen.getByRole('button', { name: /^Save$/i }))

    await waitFor(() => expect(modalControls.openModal).toHaveBeenCalledTimes(1))

    const intent = preparedIntentFromCapturedModal()
    const decoded = decodeFunctionData({
      abi: permissionedResolverAbi,
      data: intent!.request.data,
    })

    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args[0]).toBe(SET_ADDR)
    expect(decoded.args[1]).toBe(ATTACKER)
  })
})
```
